# platform_db — 문서 허브

> 진입점. 어디서 무엇을 읽을지 이 페이지에서 결정한다.

---

## 폴더 구조

```
platform_db/
  core/        핵심 레퍼런스 — 아키텍처·스키마·요구사항·운영
  decisions/   설계 결정 — 왜 이렇게 만들었나 (gate-b 스타일)
  adr/         ADR 원문 — 의사결정 기록 (ADR-025 ~ ADR-044)
  study/       DB 학습 — 이 프로젝트 사례로 배우는 DB 개념
```

---

## 목적별 진입점

| 목적 | 시작점 |
|---|---|
| 전체 그림이 보고 싶다 | `core/architecture.md` |
| 스키마·DDL이 궁금하다 | `core/schema-reference.md` |
| "왜 이렇게 설계했나?" | `decisions/` |
| ADR 원문이 필요하다 | `adr/` |
| DB 개념을 공부하고 싶다 | `study/` — 아래 학습 순서 참고 |

---

## core/ — 핵심 레퍼런스

| 파일 | 내용 |
|---|---|
| `architecture.md` | 아키텍처 핸드북 (진입점) — 8라운드 설계 여정, D1~D12 결정, 운영 플레이북, ADR 부록 |
| `schema-reference.md` | ERD · DDL · 3-gate · billing 흐름 · 멀티테넌시 · 보안 · consent 모델 |
| `requirements.md` | 요구사항 추적표 104건 + BDD 시나리오 Domain 1~10 |
| `review-checklist.md` | 설계 검토 체크리스트 P0~P5 (32건, 18 완료) |
| `migration-ops.md` | Drizzle 마이그레이션 운영 가이드 (0000~0006) |

---

## decisions/ — 설계 결정

> `gate-b-billing-grace.md`와 동일 형식.  
> **배경 → 대안 → 우리 결정(이탈 근거) → 트레이드오프 → 향후 조건 → 관련 파일**

| 파일 | 한 줄 요약 |
|---|---|
| `gate-b-billing-grace.md` | Gate B validUntil 복합체크 — status-only 권고를 이탈한 이유 |
| `design-asymmetry.md` | 비대칭 분리 — MSA도 모놀리스도 아닌 platform_db + 서비스 DB |
| `auth-projection.md` | billing truth → auth projection — org_entitlement 분리 이유 |
| `payment-atomicity.md` | 결제↔권한 단일 트랜잭션 — Kafka를 쓰지 않은 이유 |
| `multitenancy-pool.md` | Pool 모델 — schema-per-tenant를 거부한 이유 |
| `firebase-boundary.md` | Firebase = 인증만 — 인가를 우리 DB에 둔 이유 |
| `role-as-code.md` | role→action 코드 상수 — DB 레지스트리를 거부한 이유 |

---

## adr/ — ADR 원문

| 파일 | 결정 |
|---|---|
| `ADR-044-platform-db-boundary.md` | platform_db 경계: identity + billing + product 통합 |
| `ADR-041-multitenancy-db-strategy.md` | 멀티테넌시: 단일 DB + 행 격리 + 분리 트리거 |
| `ADR-042-cross-tenant-query-separation.md` | cross-tenant 조회: Admin Role 거부 + 아키텍처 분리 |
| `ADR-032-platform-identity-billing-access-strategy.md` | identity/billing 접근: Option A(공유 패키지) + Option B 전환 준비 |
| `ADR-035-rag-shared-collection-multitenancy.md` | Qdrant: shared collection + payload filter |
| `ADR-036-qdrant-neo4j-dual-index.md` | RAG 이중 색인: Qdrant(dense) + Neo4j(graph) |
| `ADR-025-domain-bounded-context.md` | 도메인 바운디드 컨텍스트 / YAGNI |

---

## study/ — DB 학습

> DB를 처음 공부하는 분을 위한 **이 프로젝트 사례 중심** 설명.  
> 아래 순서대로 읽으면 기초 → 심화 순서가 됩니다.

### 기초 (데이터 모델링)

| 파일 | 배울 것 |
|---|---|
| `pk-ulid.md` | PK 설계: 순번 PK 노출이 왜 위험한가, ULID가 해결하는 것 |
| `soft-delete.md` | 삭제 3종 패턴: status / deleted_at / append-only |
| `fk-strategy.md` | FK: 언제 걸고, cross-schema에서 왜 끊나 |
| `enum-vs-varchar.md` | ENUM vs VARCHAR+CHECK: 타입 선택이 DDL 비용을 어떻게 바꾸나 |

### 중급 (성능 & 인덱스)

| 파일 | 배울 것 |
|---|---|
| `index-design.md` | 복합 인덱스 설계: 순서가 왜 중요한가 |
| `partition-range.md` | RANGE 파티셔닝: audit_log를 월별로 나누는 법 |

### 심화 (패턴 & 일관성)

| 파일 | 배울 것 |
|---|---|
| `outbox-pattern.md` | Outbox 패턴: 이벤트 유실 없이 async 처리 |
| `idempotency-key.md` | 멱등키: 같은 요청 두 번 와도 안전한 법 |
| `json-column.md` | JSON 컬럼: 언제 쓰고 언제 정규화하나 |
| `append-only.md` | append-only 패턴: 왜 UPDATE를 안 하나 |
