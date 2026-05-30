# platform_db — 문서 허브

> 진입점. 어디서 무엇을 읽을지 이 페이지에서 결정한다.

---

## 폴더 구조

```
platform_db/
  core/        핵심 레퍼런스 — 아키텍처·스키마·요구사항·운영
  decisions/   설계 결정 — 왜 이렇게 만들었나 (gate-b 스타일)
  adr/         ADR 원문 — 의사결정 기록 (ADR-025 ~ ADR-044)
  explainers/  온보딩 Q&A — DB 지식이 적은 개발자를 위한 설명 문서 21편
```

---

## 목적별 진입점

| 목적 | 시작점 |
|---|---|
| 전체 그림이 보고 싶다 | [[architecture]] |
| 스키마·DDL이 궁금하다 | [[schema-reference]] |
| "왜 이렇게 설계했나?" | `decisions/` (아래 표) |
| ADR 원문이 필요하다 | `adr/` (아래 표) |
| 처음 합류해 Q&A로 배우고 싶다 | `explainers/` (아래 목록 — P0부터) |

---

## core/ — 핵심 레퍼런스

| 파일 | 내용 |
|---|---|
| [[architecture]] | 아키텍처 핸드북 (진입점) — 8라운드 설계 여정, D1~D12 결정, 운영 플레이북, ADR 부록 |
| [[schema-reference]] | ERD · DDL · 3-gate · billing 흐름 · 멀티테넌시 · 보안 · consent 모델 |
| [[requirements]] | 요구사항 추적표 104건 + BDD 시나리오 Domain 1~10 |
| [[review-checklist]] | 설계 검토 체크리스트 P0~P5 (42건, 23 완료) |
| [[migration-ops]] | Drizzle 마이그레이션 운영 가이드 (0000~0006) |

---

## decisions/ — 설계 결정

> **배경 → 대안 → 우리 결정(이탈 근거) → 트레이드오프 → 향후 조건 → 관련 파일** 형식.

| 파일 | 한 줄 요약 |
|---|---|
| [[decisions/gate-b-billing-grace\|gate-b-billing-grace]] | Gate B validUntil 복합체크 — status-only 권고를 이탈한 이유 |
| [[design-asymmetry]] | 비대칭 분리 — MSA도 모놀리스도 아닌 platform_db + 서비스 DB |
| [[auth-projection]] | billing truth → auth projection — org_entitlement 분리 이유 |
| [[payment-atomicity]] | 결제↔권한 단일 트랜잭션 — Kafka를 쓰지 않은 이유 |
| [[multitenancy-pool]] | Pool 모델 — schema-per-tenant를 거부한 이유 |
| [[firebase-boundary]] | Firebase = 인증만 — 인가를 우리 DB에 둔 이유 |
| [[role-as-code]] | role→action 코드 상수 — DB 레지스트리를 거부한 이유 |

---

## adr/ — ADR 원문

| 파일 | 결정 |
|---|---|
| [[ADR-044-platform-db-boundary\|ADR-044]] | platform_db 경계: identity + billing + product 통합 |
| [[ADR-041-multitenancy-db-strategy\|ADR-041]] | 멀티테넌시: 단일 DB + 행 격리 + 분리 트리거 |
| [[ADR-042-cross-tenant-query-separation\|ADR-042]] | cross-tenant 조회: Admin Role 거부 + 아키텍처 분리 |
| [[ADR-032-platform-identity-billing-access-strategy\|ADR-032]] | identity/billing 접근: Option A(공유 패키지) + Option B 전환 준비 |
| [[ADR-035-rag-shared-collection-multitenancy\|ADR-035]] | Qdrant: shared collection + payload filter |
| [[ADR-036-qdrant-neo4j-dual-index\|ADR-036]] | RAG 이중 색인: Qdrant(dense) + Neo4j(graph) |
| [[ADR-025-domain-bounded-context\|ADR-025]] | 도메인 바운디드 컨텍스트 / YAGNI |

---

## explainers/ — 온보딩 Q&A

> 대상: DB 지식이 많지 않은 개발자 · 형식: Q&A(왜 그렇게 설계했는지) · 상태: 전편 완료(21편)

### P0 — 코드 짜기 전에 반드시 읽어야 하는 것

| # | 파일 | 핵심 질문 |
|---|---|---|
| 1 | [[gate-abc-flow]] | API 요청 하나가 DB에서 어떤 3단계를 거치나요? |
| 2 | [[gate-b-entitlement]] | 결제 상태랑 서비스 접근이 왜 다른 테이블인가요? |
| 3 | [[role-capability]] | platform_role이 뭐고 service_role이 뭔가요? 왜 둘인가요? |
| 4 | [[pk-ulid-strategy]] | 테이블마다 pk랑 public_id가 두 개인데 뭘 쓰나요? |
| 5 | [[multitenancy-rls]] | MySQL은 RLS가 없는데 다른 org 데이터를 어떻게 막나요? |

### P1 — 기능 구현 전에 읽으면 좋은 것

| # | 파일 | 핵심 질문 |
|---|---|---|
| 6 | [[explainers/gate-b-billing-grace\|gate-b-billing-grace]] | status만 보면 안 되고 validUntil도 같이 봐야 하는 이유는? |
| 7 | [[enum-vs-varchar-check]] | ENUM 쓰면 편한데 왜 VARCHAR+CHECK를 쓰나요? |
| 8 | [[subscription-lifecycle]] | TRIALING→ACTIVE→CANCELED→EXPIRED 각 상태에서 무슨 일이? |
| 9 | [[feature-limits]] | product_feature, plan_definition, org_entitlement 중 뭘 읽나요? |
| 10 | [[idempotency-key]] | 결제 요청을 두 번 보내면 어떻게 되나요? |
| 11 | [[outbox-pattern]] | 이벤트를 그냥 Kafka/메시지큐로 바로 보내면 안 되나요? |
| 12 | [[webhook-processing]] | Toss/Stripe 결제 결과가 DB에 어떻게 반영되나요? |
| 13 | [[index-design]] | 인덱스가 없으면 뭐가 문제고, 복합 인덱스는 어떻게 작동하나요? |
| 14 | [[pipa-consent]] | 개인정보 동의가 왜 테이블이 따로 있고, 철회를 즉시 반영하나요? |
| 15 | [[delete-patterns]] | 삭제할 때 status / deleted_at / append-only 중 뭘 쓰나요? |
| 16 | [[fk-strategy]] | FK를 언제 걸고, cross-schema에서는 왜 끊나요? |
| 17 | [[json-column]] | JSON 컬럼은 언제 쓰고 언제 정규화하나요? |

### P2 — 운영·확장할 때 필요한 것

| # | 파일 | 핵심 질문 |
|---|---|---|
| 18 | [[break-glass]] | 긴급 상황에 운영팀이 DB를 직접 건드릴 때 왜 특별한 컬럼이 필요한가요? |
| 19 | [[partitioning]] | 파티션이 뭔가요? 왜 audit_log만 파티션을 쓰나요? |
| 20 | [[online-ddl-migration]] | 운영 중 대형 테이블에 컬럼 추가하면 왜 서비스가 멈출 수 있나요? |
| 21 | [[audit-hash-chain]] | 로그에 해시값을 왜 저장하나요? 그냥 로그 남기면 안 되나요? |

### 읽는 순서 추천

```
처음 합류:  1 → 2 → 4 → 5 → 3      (Gate 흐름 + 식별자 + 멀티테넌시)
            ↓
빌링 흐름:  6 → 8 → 9 → 10 → 11 → 12
            ↓
스키마 설계: 13 → 15 → 16 → 17 → 7  (인덱스·삭제·FK·JSON·타입)
            ↓
나머지(14·18~21)는 해당 기능 구현·운영할 때
```
