# ADR-032: Platform identity/billing 접근 전략 — Option A (공유 패키지) 채택, Option B (분리 서비스) 전환 준비

| 항목 | 값 |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-05-25 |
| **Author** | changsik |
| **Replaces** | (없음 — 신규) |
| **Related** | `packages/db-platform/docs/ARCHITECTURE.md`, `apps/platform-api/README.md`, `apps/platform-api/docs/migration-plan.md`, `apps/academy-mvp/docs/ARCHITECTURE.md` |

---

## Context

### 배경

academy-api 가 identity_db(organization, membership, delegation_grant 등)와 billing_db(org_entitlement, usage_log 등)에 접근해야 한다. 이 두 DB는 academy 전용이 아니다 — agent-api, marketplace-api 등 모노레포 내 다른 앱도 동일한 identity/billing 인프라를 공유해야 한다.

따라서 **"어떤 앱이 identity/billing DB에 어떻게 접근하는가"** 를 초기에 결정해야 한다. 잘못 결정하면 앱이 늘어날수록 마이그레이션 비용이 선형이 아닌 지수로 증가한다.

### 검토한 옵션

#### Option C — 경계 없음 (기각)

각 앱이 identity_db / billing_db 를 Drizzle 로 직접 참조.

```
academy-api  ──── identity_db (Drizzle 직접)
agent-api    ──── identity_db (Drizzle 직접)
marketplace  ──── identity_db (Drizzle 직접)
```

**문제**:
- identity 스키마 컬럼 하나를 바꾸면 모든 앱의 Drizzle 쿼리를 동시에 수정/배포해야 함
- "어떤 앱이 membership 테이블을 쓰는가" 추적 불가능
- Gate 로직(membership.status = ACTIVE 체크)이 각 앱에 중복 작성 → 불일치 버그 발생

**기각** — 앱이 2개를 넘는 시점에 통제 불가 상태가 된다.

---

#### Option A — 공유 패키지 `@aiagent/db-platform`

모든 앱이 동일한 패키지를 통해 identity/billing 에 접근. 패키지 내부에서 Drizzle 쿼리.

```
academy-api  ──┐
agent-api    ──┼──  @aiagent/db-platform  ──  identity_db
marketplace  ──┘         (패키지)          └─  billing_db
```

**장점**:
- 스키마 변경 시 패키지 하나만 수정 → 소비 앱은 패키지 버전 업그레이드만
- Gate 로직(`checkGateA`, `checkGateB`, `getPermissionContext`) 단일 출처
- 네트워크 홉 없음 → 매 요청당 Gate 체크가 μs (로컬 DB 쿼리)
- 로컬 개발 환경 단순 — 앱 하나만 실행

**단점**:
- 모든 소비 앱이 identity_db / billing_db 에 직접 DB 커넥션 보유
- 컴플라이언스(SOC2, PCI-DSS)가 auth/billing DB 격리를 요구할 경우 대응 필요
- 스키마 변경 시 모든 소비 앱이 같은 배포 윈도우에 업그레이드해야 함 (패키지 monorepo 이므로 일반적으로 동시 배포)

---

#### Option B — 분리 서비스 `apps/platform-api`

identity/billing 을 독립 NestJS 서비스로 운영. 소비 앱은 HTTP 로 접근.

```
academy-api  ──┐
agent-api    ──┼──  platform-api (HTTP)  ──  identity_db
marketplace  ──┘     (NestJS 서비스)      └─  billing_db
```

**장점**:
- identity/billing DB 의 유일한 소유자 → 스키마 변경이 소비 앱 배포와 분리 가능
- 컴플라이언스 요구 충족 (DB 격리)
- 팀 분리 시 platform 팀이 독립 배포 가능 (Conway's Law 정합)

**단점**:
- Gate 체크(매 요청마다 발생)가 네트워크 홉 → 캐싱 없이는 레이턴시 증가
- Redis 캐싱 레이어 설계 필요 (membership, entitlement TTL 60s)
- 로컬 개발 환경에서 platform-api + 소비 앱 동시 실행 필요
- platform-api 장애 시 소비 앱의 인가 불가 → 단일 장애점 (최소 2 replica 운영 필요)
- 서비스 간 Internal JWT 관리 추가

---

### 트레이드오프 매트릭스

| 기준 | Option A | Option B |
|---|---|---|
| 초기 구현 비용 | 낮음 | 높음 (Redis + Internal JWT + replica 설계) |
| Gate 체크 레이턴시 | μs (로컬 쿼리) | ms (HTTP + Redis 캐시 hit) |
| 스키마 변경 독립성 | 낮음 (모든 앱 동시 배포) | 높음 (platform-api 독립 배포) |
| 팀 경계 정합성 | 낮음 (공유 패키지는 누구 소유?) | 높음 (platform 팀 명확) |
| 컴플라이언스 대응 | 보통 (DB 직접 접근 남음) | 높음 (DB 격리) |
| 장애 전파 | 낮음 (DB 직접) | 높음 (platform-api 장애 시 cascading) |
| A→B 마이그레이션 비용 | — | 높음 if 인터페이스 없을 때 / 낮음 if 인터페이스 있을 때 |

---

### 현업 결정 기준 (이 프로젝트에 적용)

현업에서 이 결정은 보통 두 기준으로 내린다:

**1. Conway's Law — 팀 구조가 서비스 경계를 결정한다**

> 시스템 설계는 그 조직의 커뮤니케이션 구조를 닮는다. — Melvin Conway, 1967

- 현재: 소규모 팀이 identity/billing/academy 모두 소유
- 팀이 분리되면 자연스럽게 Option B 로 수렴
- 지금 Option B 를 강제하면 팀 구조보다 기술 구조가 앞서감 → 불필요한 운영 부담

**2. 소비 앱 수 — 실제 재사용 시점이 분리 기준**

| 소비 앱 수 | 현업 판단 |
|---|---|
| 1개 | Option A 충분 |
| 2~3개 동시 | Option A + 추출 계획 |
| **3개 이상 + 팀 분리** | **Option B 로 전환** |

현재 계획된 소비 앱: academy-api, agent-api, marketplace-api → 3개.
MVP 단계에서는 academy-api 1개만 실제 운영.

---

## Decision

**Option A 채택. 단, 인터페이스를 Option B 처럼 설계하여 전환 비용을 최소화한다.**

### 핵심 판단

마이그레이션 비용의 실체는 "A 구조 vs B 구조" 차이가 아니라 **"경계가 있는가 없는가"** 다.

- `@aiagent/db-platform` 패키지가 인터페이스를 소비 앱에 노출
- 내부 구현(Drizzle)은 패키지 경계 안에 완전히 캡슐화
- Option B 전환 시 패키지 내부 구현만 HTTP 로 교체 → 소비 앱 0 변경

```typescript
// 소비 앱 코드 — Option A/B 전환 불문 동일
import { getPermissionContext } from '@aiagent/db-platform'
const ctx = await getPermissionContext(userPk, orgPk)
```

이 단일 원칙을 지키면 A→B 전환은 외과수술이 아닌 **구현 교체**다.

### apps/platform-api Placeholder

전환 준비를 위해 `apps/platform-api/` 를 placeholder 로 생성.

- `README.md`: 전환 트리거 3가지 (소비 앱 3개 이상 동시 운영 / platform 전담 팀 분리 / SOC2·PCI-DSS 요구)
- `docs/ARCHITECTURE.md`: Option B 구현 설계 (캐싱 전략, Internal JWT, Module 구성)
- `docs/migration-plan.md`: A→B 단계별 체크리스트
- `docs/api-contracts.md`: 미래 HTTP 엔드포인트 계약

---

## Consequences

### 즉각 적용

- 모든 소비 앱은 identity/billing DB 에 **직접 Drizzle 접근 금지**
- `@aiagent/db-platform` 만 identity/billing 접근 창구
- 패키지 내부: Drizzle schema, DB connection 은 외부 export 금지
- Gate 로직은 `getPermissionContext()` 하나로 통일 — 앱에서 직접 membership 체크 금지

### Option B 전환 트리거 (세 가지 중 하나 충족 시)

1. 소비 앱 3개 이상 동시 운영 시작
2. platform 전담 팀 분리
3. SOC2 / PCI-DSS 컴플라이언스 요구

전환 절차 → [`apps/platform-api/docs/migration-plan.md`](../../apps/platform-api/docs/migration-plan.md)

### 장기 위험

- `@aiagent/db-platform` 내부 인터페이스가 오염될 경우(Drizzle 타입 노출 등) 전환 비용 증가 → PR 리뷰에서 `src/index.ts` export 목록 검수 필수

---

## 참조

| 문서 | 내용 |
|---|---|
| `packages/db-platform/docs/ARCHITECTURE.md` | 패키지 내부 설계 + Option A/B 구현 계획 |
| `apps/platform-api/README.md` | 전환 트리거 + 현재 상태 |
| `apps/platform-api/docs/ARCHITECTURE.md` | Option B NestJS 서비스 설계 |
| `apps/platform-api/docs/migration-plan.md` | A→B 단계별 체크리스트 |
| `apps/platform-api/docs/api-contracts.md` | Option B HTTP API 계약 |
| `apps/academy-mvp/docs/ARCHITECTURE.md` | 소비 앱 적용 예시 (OrgGateGuard) |
| ADR-024 | Layered Clean within NestJS (패키지/앱 공통 구조 원칙) |
