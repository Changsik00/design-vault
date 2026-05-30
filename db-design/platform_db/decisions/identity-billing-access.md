# identity/billing 접근 전략 — 공유 패키지(Option A) 채택, 분리 서비스(B) 전환 준비

> 상태: 채택 · 영역: platform_db 접근 경계 · 형식: 비교 → 결정

## 결정

소비 앱(academy-api, agent-api, marketplace-api 등)은 identity/billing에 **`@aiagent/db-platform` 공유 패키지로만** 접근한다(Drizzle 직접 참조 금지). 단, **인터페이스를 분리 서비스(Option B)처럼 설계**해 전환 비용을 최소화한다.

---

## 맥락 — 왜 결정이 필요했나

identity/billing DB는 academy 전용이 아니다 — 여러 앱이 같은 인프라를 공유한다. "어떤 앱이 어떻게 접근하나"를 초기에 잘못 정하면, 앱이 늘수록 마이그레이션 비용이 선형이 아니라 **지수로** 증가한다.

---

## 비교한 선택지

### 선택지 C — 경계 없음 (각 앱이 Drizzle 직접 참조)

- **장점**: 가장 단순, 중간 계층 없음
- **단점**: identity 컬럼 하나 바꾸면 모든 앱 쿼리 동시 수정·배포 / "누가 membership 테이블을 쓰는가" 추적 불가 / Gate 로직이 앱마다 중복돼 불일치 버그
- **기각** — 앱이 2개를 넘는 순간 통제 불가.

### 선택지 A — 공유 패키지 `@aiagent/db-platform` (채택)

모든 앱이 같은 패키지를 통해 접근, Drizzle 쿼리는 패키지 내부에 캡슐화.

- **장점**:
  - 스키마 변경 시 패키지 하나만 수정 → 소비 앱은 버전 업그레이드만
  - Gate 로직(`checkGateA/B`, `getPermissionContext`) 단일 출처
  - 네트워크 홉 없음 → Gate 체크가 μs(로컬 DB 쿼리)
  - 로컬 개발 단순(앱 하나만 실행)
- **단점**:
  - 모든 소비 앱이 identity/billing DB에 직접 커넥션 보유
  - SOC2·PCI-DSS가 auth/billing DB 격리를 요구하면 대응 필요
  - 스키마 변경 시 소비 앱이 같은 배포 윈도우에 업그레이드

### 선택지 B — 분리 서비스 `platform-api` (HTTP)

identity/billing을 독립 서비스로 운영, 소비 앱은 HTTP로 접근.

- **장점**: DB 단일 소유자(스키마 변경이 소비 앱 배포와 분리) / 컴플라이언스(DB 격리) 충족 / 팀 분리 시 독립 배포(Conway's Law 정합)
- **단점**: 매 요청 Gate 체크가 네트워크 홉 → 캐싱 없이는 레이턴시↑ / Redis 캐시·Internal JWT 설계 필요 / platform-api 장애 시 전 소비 앱 인가 불가(단일 장애점) / 로컬 개발 복잡

| 기준 | C: 직접 | A: 공유 패키지 (우리) | B: HTTP 서비스 |
|---|---|---|---|
| 초기 구현 비용 | 최저 | 낮음 | 높음 |
| Gate 체크 레이턴시 | μs | μs(로컬) | ms(HTTP+캐시) |
| 스키마 변경 독립성 | 없음 | 낮음(동시 배포) | 높음 |
| 팀 경계 정합 | 없음 | 낮음 | 높음 |
| 컴플라이언스(DB 격리) | 없음 | 보통 | 높음 |
| 장애 전파 | — | 낮음(DB 직접) | 높음(cascading) |

---

## 왜 A인가

마이그레이션 비용의 실체는 "A 구조 vs B 구조"가 아니라 **"경계가 있는가 없는가"**다. 패키지가 인터페이스를 노출하고 내부 구현(Drizzle)을 캡슐화하면, A→B 전환은 외과수술이 아니라 **구현 교체**다 — 소비 앱 코드 0 변경.

```typescript
// 소비 앱 코드 — A/B 전환 불문 동일
import { getPermissionContext } from '@aiagent/db-platform'
const ctx = await getPermissionContext(userPk, orgPk)
```

현 단계 판단 기준 두 가지:
- **Conway's Law**: 지금은 소규모 팀이 identity/billing/academy를 모두 소유 → 지금 B를 강제하면 기술 구조가 팀 구조를 앞서감(불필요한 운영 부담).
- **소비 앱 수**: 1개=A 충분 / 2~3개=A + 추출 계획 / 3개+ 팀 분리=B 전환. 현재 실운영은 academy-api 1개.

---

## 트레이드오프 — 우리가 받아들인 것

- 모든 소비 앱이 DB 커넥션을 직접 들고 있어, 컴플라이언스 요구가 오면 전환 작업이 필요하다(그 비용을 "경계 설계"로 미리 낮춰둠).
- 스키마 변경 시 monorepo 소비 앱이 함께 배포돼야 한다.

## 전환 조건 — B(HTTP 서비스)로 갈 때

세 가지 중 하나 충족 시:
1. 소비 앱 3개 이상 동시 운영 시작
2. platform 전담 팀 분리
3. SOC2 / PCI-DSS 컴플라이언스 요구

> 장기 위험: 패키지 인터페이스가 오염되면(Drizzle 타입 노출 등) 전환 비용이 커진다 → 패키지 export 목록을 PR에서 검수한다.

---

## 관련 문서

- [[design-asymmetry]] — platform_db 경계(무엇을 공유 패키지가 감싸는가)
- [[auth-projection]] · [[role-as-code]] — 패키지가 노출하는 Gate 로직의 내용
> 소스 문서
- [[architecture]] — §2 R0, §12.1 논리 소유권·전환 단계
