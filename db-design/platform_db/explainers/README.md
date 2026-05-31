---
type: index
aliases: [explainers 인덱스, 학습 가이드]
tags: [platform-db, explainer, index]
---

# platform_db explainers — 학습 인덱스

> `platform_db` 설계를 *공부용*으로 풀어쓴 교육 문서 모음. 각 문서는 **Q&A + 용어 정리 + 테스트 방법** 구조.
> **분류**: 디렉토리 = 주제, `difficulty` 속성 = 난이도(🟢 초 → 🟡 중 → 🔴 고).
> 강의 진도는 각 주제 안에서 초 → 고 순으로 따라가면 됩니다.

**주제별 인덱스**: [🔐 auth](./auth/README.md) · [💳 billing](./billing/README.md) · [🗃️ data-modeling](./data-modeling/README.md) · [🛡️ security](./security/README.md) · [📋 compliance](./compliance/README.md) · [🛠️ operations](./operations/README.md) · [🧭 decisions](../decisions/README.md)

## 🔐 인가·인증 (Auth)  ·  10편

| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[gate-abc-flow]] | Gate A/B/C 전체 흐름 |
| 🟢 초 | [[gate-b-entitlement]] | Gate B와 엔타이틀먼트 개념 |
| 🟡 중 | [[bola-object-authz]] | "로그인은 됐는데 남의 데이터가 보이는" 취약점 |
| 🟡 중 | [[casl-ability]] | RBAC·ABAC·ReBAC을 하나의 can()으로 합치기 (Gate C 내부) |
| 🟡 중 | [[fail-closed]] | 의심스러우면 막는다 |
| 🟡 중 | [[explainers/auth/gate-b-billing-grace|gate-b-billing-grace]] | Gate B 유예 기간 설계 결정 (status + validUntil 복합 체크) |
| 🟡 중 | [[role-capability]] | role 2단 분리와 capability |
| 🔴 고 | [[machine-identity-apikey]] | 사람만 사용자가 아니다 |
| 🔴 고 | [[perm-version-propagation]] | perm_version 전파 |
| 🔴 고 | [[rebac-delegation]] | 관계 기반 권한 |

## 💳 결제·구독 (Billing)  ·  7편

| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[feature-limits]] | feature_limits 3중 정의 우선순위 |
| 🟢 초 | [[subscription-lifecycle]] | 구독 상태 머신 (TRIALING → ACTIVE → CANCELED → EXPIRED) |
| 🟡 중 | [[idempotency-key]] | 멱등성 키 (payment_ledger) |
| 🟡 중 | [[outbox-pattern]] | Outbox 패턴 |
| 🟡 중 | [[usage-metering]] | usage_snapshot |
| 🟡 중 | [[webhook-processing]] | PG 웹훅 수신·처리 흐름 |
| 🔴 고 | [[consistency-model]] | 강한 일관성 vs 결과적 일관성 (왜 단일 트랜잭션 + outbox) |

## 🗄️ 데이터 모델링 (Data Modeling)  ·  8편

| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[delete-patterns]] | 삭제·이력 보존 패턴 (status / deleted_at / append-only) |
| 🟢 초 | [[enum-vs-varchar-check]] | ENUM vs VARCHAR+CHECK (D6 원칙) |
| 🟢 초 | [[json-column]] | JSON 컬럼 (언제 쓰고 언제 정규화하나) |
| 🟢 초 | [[pk-ulid-strategy]] | BIGINT pk + ULID public_id 전략 |
| 🟡 중 | [[fk-strategy]] | FK 전략 (언제 걸고, cross-schema에서 왜 끊나) |
| 🟡 중 | [[index-design]] | 인덱스 설계 원리 |
| 🟡 중 | [[partitioning]] | DB 파티셔닝 (audit_log 파티션 예시) |
| 🔴 고 | [[online-ddl-migration]] | 온라인 DDL과 대형 테이블 마이그레이션 위험 |

## 🛡️ 보안·격리 (Security)  ·  3편

| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[multitenancy-rls]] | Pool 모델 멀티테넌시와 RLS 개념 |
| 🟡 중 | [[least-privilege-grant]] | "UPDATE 금지"를 코드 규약이 아니라 DB 권한으로 |
| 🟡 중 | [[secret-encryption]] | 무엇을 어디서 어떻게 보호하나 |

## ⚖️ 컴플라이언스 (Compliance)  ·  3편

| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟡 중 | [[data-lifecycle-retention]] | 데이터 생명주기와 보존·파기 (retention) |
| 🟡 중 | [[pipa-consent]] | PIPA 개인정보 동의 요건 |
| 🔴 고 | [[audit-hash-chain]] | audit_log 해시 체인 무결성 |

## 🛠️ 운영 (Operations)  ·  6편

| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟡 중 | [[break-glass]] | Break-glass 긴급 접근 |
| 🟡 중 | [[observability-slo]] | 관측성과 신뢰성 계약 (SLO·SLI·모니터링) |
| 🟡 중 | [[orm-testing-drizzle]] | "타입은 통과하는데 진짜 DB에선 터지는" 갭을 메우는 법 |
| 🟡 중 | [[permission-debugger]] | "왜 막혔나"를 1초에 답하는 계약 (DBG-1) |
| 🟡 중 | [[rate-limiting]] | 무엇을 어디서 막나 |
| 🔴 고 | [[testing-strategy]] | 실제 DB에서, 그리고 운영 DB에서 안전하게 테스트하기 |

---
> 총 37편. 새 문서는 해당 주제 디렉토리에 두고 frontmatter에 `difficulty: 초|중|고`를 넣으면 이 인덱스 분류에 들어갑니다.
