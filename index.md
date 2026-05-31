---
type: index
aliases:
  - index
  - 위키 카탈로그
  - platform_db index
tags:
  - platform-db
  - index
---

# platform_db — 위키 카탈로그 (index)

이 vault의 **단일 카탈로그**다. 무엇이 어디 있는지 여기서 다 찾는다.

> 🧭 사람용 소개·배포 랜딩 → [README](./README.md) · 위키 운영 규약(Ingest/Query/Lint) → [WIKI](./WIKI.md) · 변경 이력 → [log](./log.md) · 작성 규약 → [[CONVENTIONS]]
> 📄 **설계 문서(코드 없음).** 상태 표기 = *설계 성숙도*(✅ 확정 · 🟡 부분 · 🔴 미설계 · ⛔ 보류 · ❓ 미결정).
> 🐘 DDL·정책은 **PostgreSQL 1순위**, MySQL은 `🐬 MySQL이라면` 콜아웃.

## 🚪 진입 순서 (처음이면)

1. [[architecture]] — 아키텍처 & 규율 (개요·불변식·3-gate)
2. [[requirements]] — 무엇을 충족해야 하나 (+ 요구 추적 매트릭스)
3. [[schema-reference]] — 전체 DDL · ERD · billing · 보안
4. **decisions** — "왜 이렇게 했나" (아래 §decisions)
5. **explainers** — 개념 학습 문서 38편 (아래 §explainers)
6. [[bdd-scenarios]] · [[e2e-journeys]] — 행위 검증

## 🧩 핵심 한 장 요약

- **격리**: 모든 도메인 테이블 `org_pk NOT NULL`(Pool 모델) + RLS + 질의 강제 → BOLA 차단
- **인가**: 3-gate(소속·결제·정책), 역할은 코드 상수, 위임은 `delegation_grant`
- **결제**: `org_subscription`(진실) → `org_entitlement`(권한 투영, service당 1개), 단일 트랜잭션 + outbox
- **불변**: 금융·감사·동의는 append-only(GRANT로 강제), 식별자는 BIGINT pk + ULID public_id

## 📚 core — 설계 정본

| 문서 | 내용 |
|---|---|
| [[architecture]] | 아키텍처 + 규율(불변식 15개·3-gate·멀티테넌시·로드맵·법/표준 준거) |
| [[requirements]] | 요구사항(분류별 ID) + 추적 매트릭스 |
| [[schema-reference]] | 전체 DDL · ERD(mermaid) · 3-gate · billing · 보안 · consent |
| [[operability]] | 운영 가능성 O1~O6(operator-plane·support action·보존) |
| [[bdd-scenarios]] | BDD(화이트박스) + 요구 추적 매트릭스 |
| [[e2e-journeys]] | E2E(블랙박스) 가치 여정 + 예외·생명주기 |
| [[CONVENTIONS]] | 파일명·frontmatter·wikilink·ID 규약·DB 엔진·다이어그램 |

## 🧭 decisions — 왜 이렇게 했나 · 13편

| 결정 | 한 줄 |
|---|---|
| [[multitenancy-pool]] | Pool 모델 — 공유 DB + org_pk 행 격리 |
| [[payment-atomicity]] | 결제 단일 Postgres 트랜잭션 (2PC·Kafka 거부) |
| [[auth-projection]] | 권한 투영 — org_subscription → org_entitlement |
| [[audit-two-lane]] | 감사 2-lane (audit_log vs payment_ledger) |
| [[role-as-code]] | role→action을 코드 상수로 |
| [[service-extensibility]] | 서비스 확장과 결합 (VARCHAR+CHECK) |
| [[design-asymmetry]] | 통합 platform_db (identity/billing 비분리) |
| [[firebase-boundary]] | Firebase 경계 — firebase_uid는 조회 키 |
| [[identity-billing-access]] | identity·billing 접근 전략 |
| [[cross-tenant-separation]] | cross-tenant 조회는 아키텍처 분리 |
| [[operator-plane]] | 운영자 신원 평면 (org_pk 없는 operator) + override 계약 |
| [[decisions/gate-b-billing-grace\|gate-b-billing-grace]] | Gate B 유예 기간 (status + validUntil) |
| [[rag-multitenancy]] | RAG(Qdrant 벡터·Neo4j 그래프) 멀티테넌시 격리 |

## 🎓 explainers — 학습 문서 · 38편 (난이도 🟢 초 → 🟡 중 → 🔴 고)

### 🔐 auth — 인증·인가 · 10편
| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[gate-abc-flow]] | Gate A/B/C 전체 흐름 |
| 🟢 초 | [[gate-b-entitlement]] | Gate B와 엔타이틀먼트 개념 |
| 🟡 중 | [[bola-object-authz]] | "로그인은 됐는데 남의 데이터가 보이는" 취약점 |
| 🟡 중 | [[casl-ability]] | RBAC·ABAC·ReBAC을 하나의 can()으로 (Gate C 내부) |
| 🟡 중 | [[fail-closed]] | 의심스러우면 막는다 |
| 🟡 중 | [[explainers/auth/gate-b-billing-grace\|gate-b-billing-grace]] | Gate B 유예 기간 설계 (status + validUntil) |
| 🟡 중 | [[role-capability]] | role 2단 분리와 capability |
| 🔴 고 | [[machine-identity-apikey]] | 사람만 사용자가 아니다 (api_key) |
| 🔴 고 | [[perm-version-propagation]] | perm_version 전파 |
| 🔴 고 | [[rebac-delegation]] | 관계 기반 권한 위임 |

### 💳 billing — 결제·구독 · 7편
| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[feature-limits]] | feature_limits 3중 정의 우선순위 |
| 🟢 초 | [[subscription-lifecycle]] | 구독 상태 머신 (TRIALING → ACTIVE → …) |
| 🟡 중 | [[idempotency-key]] | 멱등성 키 (payment_ledger) |
| 🟡 중 | [[outbox-pattern]] | Outbox 패턴 (+ 발행 의무 레지스트리) |
| 🟡 중 | [[usage-metering]] | usage_snapshot |
| 🟡 중 | [[webhook-processing]] | PG 웹훅 수신·처리 흐름 |
| 🔴 고 | [[consistency-model]] | 강한 일관성 vs 결과적 일관성 |

### 🗄️ data-modeling — 데이터 모델링 · 9편
| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[delete-patterns]] | 삭제·이력 보존 패턴 (status / deleted_at / append-only) |
| 🟢 초 | [[enum-vs-varchar-check]] | ENUM vs VARCHAR+CHECK (D6 원칙) |
| 🟢 초 | [[json-column]] | JSONB 컬럼 (언제 쓰고 언제 정규화) |
| 🟢 초 | [[pk-ulid-strategy]] | BIGINT pk + ULID public_id 전략 |
| 🟡 중 | [[fk-strategy]] | FK 전략 (cross-schema에서 왜 끊나) |
| 🟡 중 | [[index-design]] | 인덱스 설계 원리 |
| 🟡 중 | [[partitioning]] | 선언적 파티셔닝 (audit_log) |
| 🔴 고 | [[online-ddl-migration]] | 온라인 DDL과 대형 테이블 마이그레이션 |
| 🔴 고 | [[concurrency-control]] | 동시성 제어 — race·락·조건부 UPDATE·SKIP LOCKED |

### 🛡️ security — 보안·격리 · 3편
| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[multitenancy-rls]] | Pool 모델 멀티테넌시와 RLS |
| 🟡 중 | [[least-privilege-grant]] | "UPDATE 금지"를 코드가 아니라 DB 권한으로 |
| 🟡 중 | [[secret-encryption]] | 무엇을 어디서 어떻게 보호하나 |

### 📋 compliance — 컴플라이언스 · 3편
| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟡 중 | [[data-lifecycle-retention]] | 데이터 생명주기와 보존·파기 |
| 🟡 중 | [[pipa-consent]] | PIPA 개인정보 동의 요건 |
| 🔴 고 | [[audit-hash-chain]] | audit_log 해시 체인 무결성 |

### 🛠️ operations — 운영·테스트 · 6편
| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟡 중 | [[break-glass]] | Break-glass 긴급 접근 |
| 🟡 중 | [[observability-slo]] | 관측성과 신뢰성 계약 (SLO·SLI) |
| 🟡 중 | [[orm-testing-drizzle]] | 타입은 통과하는데 진짜 DB에선 터지는 갭 |
| 🟡 중 | [[permission-debugger]] | "왜 막혔나"를 1초에 답하는 계약 (DBG-1) |
| 🟡 중 | [[rate-limiting]] | 무엇을 어디서 막나 |
| 🔴 고 | [[testing-strategy]] | 실제 DB·운영 DB에서 안전하게 테스트 |

## 🗄️ 핵심 엔티티 (entities)

테이블 정본은 [[schema-reference]] §D. 자주 찾는 것:

- **identity**: `identity_user` · `user_profile` · `membership` · `service_membership` · `membership_invite`
- **authZ**: `delegation_grant`(ReBAC) · `org_relation` · `operator`
- **billing**: `org_subscription` · `subscription_item` · `org_entitlement`(uq `(org_pk, service)`) · `product`·`product_sku`·`product_feature` · `plan_definition`
- **원장·이벤트**: `payment_ledger`(append-only) · `billing_event` · `pg_webhook_event` · `outbox_event`
- **감사·동의·사용량**: `audit_log`(파티션·WORM) · `user_consent_event`(append-only) · `usage_snapshot` · `api_key`

## 🗃️ 아카이브

- [[archive/enterprise-saas-multitenancy|archive]] — 결정 전 탐색 로그(원본 seed). 정본 아님.
