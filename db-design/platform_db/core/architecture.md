---
type: core
aliases:
  - 아키텍처 핸드북
  - platform_db architecture
  - 규율 문서
  - governance
tags:
  - platform-db
  - core
  - architecture
  - governance
---

# platform_db — 아키텍처 & 규율

> 이 문서 = **시스템 아키텍처 + 강제 규율(governance)**. *"이렇게 되어 있고, 이렇게 해야 한다"*를 적는다.  
> 분업: **왜** → `decisions/` · **운영** → [[operability]] · **무엇** → [[requirements]] · **행위 검증** → [[bdd-scenarios]] · **개념 Q&A** → `explainers/`  
> 작성일: 2026-05-28 

---

## 1. 시스템 아키텍처

### 1.1 토폴로지

회사의 여러 SaaS(academy / agent / market / store / fitness)를 받치는 **공통 플랫폼 코어 DB**.

```
                  ┌──────────────── platform_db (공통 코어, 단일 InnoDB) ──────────────┐
                  │  identity · authZ(membership/role/capability) · entitlement        │
   academy-api ──▶│  product · billing · consent · audit · api_key                    │◀── @aiagent/db-platform 패키지로만 접근
   agent-api   ──▶│  = "누구 / 어디 소속 / 무엇을 할 수 있나 / 무엇을 구독 / 무엇에 동의"의 SSOT │
   market-api  ──▶└────────────────────────────────────────────────────────────────────┘
   store-api   ──▶   academy_db / agent_db / market_db / store_db  (서비스 도메인, org_pk 격리)
   fitness-api ──▶   + Qdrant(벡터) · Neo4j(그래프) · Redis(캐시) · S3(자산)
```

**한 줄 철학**: *공통은 묶고(strong consistency), 도메인은 뗀다(독립 확장)* — **비대칭 분리**([[design-asymmetry]]).

> **🛢️ MySQL 제약(배경)**: MySQL 8은 회사 표준 인프라 규약에 따른 선택이며 기술적 선호가 아니다. 자유 선택이라면 **PostgreSQL 권장**(RLS·JSONB GIN·파티셔닝·pgcrypto·`FOR UPDATE SKIP LOCKED`). "MySQL은 RLS 없음"은 PostgreSQL이라면 없었을 **구조적 제약**임을 명시한다. 현재는 전환보다 최적화를 선택.

### 1.2 권한 모델 — 3-gate

```
Layer 1 인증  : Firebase Auth (누구냐) — JWT, firebase_uid
Layer 2 소속  : db-platform (어디 소속이냐) — membership tuple
Layer 3 정책  : 각 서비스 CASL ability (무엇을 할 수 있냐)
```

```
0) principal resolve   HUMAN: Firebase→user.pk / MACHINE: api_key→principal_pk
1) Gate A 소속   membership(principal, org).status = ACTIVE  (+ platform_role)
2) Gate B 이용권 org_entitlement(org, service).status ∈ {ACTIVE, GRACE} + feature_limit
3) Gate C 정책   RBAC : service_membership.role_code → ROLE_PERMISSION[service][role] (코드)
                 ReBAC: delegation_grant(grantee, capability, ACTIVE, !expired)
                 ABAC : resource.owner_pk == principal.pk  +  환경(client_ip 등)
→ A ∧ B ∧ C → ALLOW, audit_log 기록
```

| 모델 | 의미 | 저장 |
|---|---|---|
| RBAC | 직책/소속 | `service_membership.role_code` + 코드 `ROLE_PERMISSION` ([[role-as-code]]) |
| ABAC | 속성(소유·테넌트·한도·환경) | 리소스 컬럼 + `org_entitlement.feature_limits` + client_ip |
| ReBAC | 관계/위임 | `delegation_grant.capability`(임퍼소네이션 금지) |
| 계층 | 조직 위계 | `org_relation`(HQ_BRANCH/HOLDING) — **권한 근거 아님, 명시적 membership만** |

- 평가기 단일(CASL). 캐싱은 *입력 블록*(membership/grants/entitlement)만 TTL 60s, **최종 can() 캐싱 금지**.
- `perm_version` self-healing: 권한 변경→bump→`X-Perm-Version` 불일치→snapshot 재요청. 개인=user_pv / org=org_pv 분리(thundering herd 회피).
- **현황**: DB 함수(`checkGateA/B`)는 ✅. 단 `GateAGuard`(실시간 재검증)는 🧊 Icebox — 현재 Firebase custom claims로 간접 커버(취소 시 ~1h stale, 민감 쓰기는 `@VerifyOnDb`로 메움). `checkGateB`는 `GateBGuard`로 래핑(기본값 `"ACADEMY"` — [[service-extensibility]] EXT-5).

### 1.3 데이터 일관성 — billing → projection → auth

```
PG Webhook (Stripe/Toss 이벤트 스트림)
    ↓
org_subscription  ← Billing Canonical Truth (provider, invoice, retry, grace, refund, trial, ...)
    ↓ [이벤트 투영 — 단일 트랜잭션]
org_entitlement   ← Authorization Projection (status, valid_until, grace_until, feature_limits)
    ↓
Gate B            ← can_access? 만 판단. subscription 직접 조회 금지(불변식 #4)
```

- **왜 분리하나**: billing 복잡도(provider/invoice/retry)를 auth에서 격리 → auth 단일 테이블 조회, billing 장애 격리, 캐시 단순. ([[auth-projection]])
- **결제 단일 트랜잭션**: `payment_ledger INSERT(CHARGE) + org_subscription UPDATE + org_entitlement UPSERT + perm_version bump + outbox INSERT`를 한 InnoDB 트랜잭션으로 → **"결제됐는데 권한 미반영" 창 0**(2PC·Kafka 불필요). ([[payment-atomicity]])
- 멱등 2중: webhook `event_id` UNIQUE + payment `idempotency_key` UNIQUE([[idempotency-key]]). 환불은 append-only. async 부수효과만 outbox([[outbox-pattern]]).
- eventual consistency 수용: webhook→투영 수 초 지연 허용(SaaS 표준). 배치 실패 안전망은 Gate B `validUntil` 복합체크([[decisions/gate-b-billing-grace|gate-b-billing-grace]]).

### 1.4 멀티테넌시 — Pool 모델 + 분리 트리거

**전략: 공유 DB + `org_pk` 행 격리**([[multitenancy-pool]]).

| 저장소 | 격리 방법 | 상태 |
|---|---|---|
| MySQL | `org_pk NOT NULL` + 모든 조회 `WHERE org_pk` 필수 | ✅ 스키마 강제 · 🟡 CI 린트 미완 |
| Qdrant | `org_id` payload 필터 강제 | ✅ · 🟡 `is_tenant` 마커 미추가 ([[rag-multitenancy]]) |
| Neo4j | `orgId` 노드 속성 + Cypher 강제 | ✅ · 🟡 APOC 쓰기측 차단 P1 |
| Redis | key prefix `org:{org_pk}:...` | ✅ |

**분리 트리거** — 하나라도 충족 시 착수 (단계: Read replica → 고속 테이블 분리 → DB-per-large-tenant):

| 트리거 | 조건 | 행동 |
|---|---|---|
| T1 | 단일 org가 고속 테이블 행의 20%+ | 해당 테넌트 전용 DB |
| T2 | 대화/로그 P99 > 500ms | 해당 테이블 별 인스턴스 |
| T3 | usage_log 월 500만+/50GB+ | OLAP(ClickHouse/BigQuery) 이관 |
| T4 | ISMS-P/GDPR 계약 | 물리 격리 DB·리전 + 감사 외부 WORM |

cross-tenant 집계는 **아키텍처 분리**(`internal/`·`*-admin`) — Admin role 금지([[cross-tenant-separation]]). 운영자 cross-tenant는 [[operator-plane]] 경유.

### 1.5 정책·동의 모델

- **`user_consent_event` append-only**: `user × consent_type × terms_version × action(GRANTED/REVOKED) × meta_json × prev_hash/row_hash`. 반복 이력 전부 보존([[pipa-consent]]).
- **consent_type 네임스페이스**: `platform.*`(계정) / `pg.*`(결제 제3자) / 서비스 `*`(도메인 DB).
- **전자서명법 §3**: 약관 체크박스 동의 = 서면 서명과 동일 효력. `user_consent_event` row가 법적 증거.
- **데이터 이전 경계**: 본인 정보(identity·profile·멤버십)는 이전 가능, 학생·수업기록·결제이력은 org 소유로 이전 불가 — `org_pk NOT NULL` 격리가 강제. (동의: `platform.content_ownership`·`platform.data_transfer`)
- **법 요건**: 14세 미만 법정대리인 동의(§22, 법적 필수) · 제3자 제공 4요건(§17, meta_json + JSON Schema + RFC 8785 canonical) · 철회권(§37, REVOKED + perm_version) · 마케팅 옵트아웃(정보통신망법 §50).
- 보존 5년 + 해시 사슬(tamper-evident). 약관 본문은 S3/CMS, DB엔 `terms_version`만.

---

## 2. 규율 (Governance) — 위반 시 PR 반려

### 2.1 불변식 — 코드 검증됨, 위반 PR 반려

1. **Firebase = 인증, 인가 = 우리 DB.** `firebase_uid`는 조회 키일 뿐 PK/FK 아님.
2. **내부 PK는 BIGINT, 외부 노출은 ULID(`public_id`).** 시퀀셜 PK를 URL·API에 노출 금지.
3. **모든 도메인 테이블은 `org_pk NOT NULL`.** 예외 없음 — 테넌트 경계는 불변식.
4. **`canXXX()`는 `org_entitlement`만 읽는다.** `payment_ledger` 직접 조회 금지. entitlement는 billing의 authorization projection이며 Gate B의 유일 진실 원천.
5. **`service`는 `VARCHAR(50)` + `CHECK`.** ENUM 아님. 새 서비스 = CHECK 1줄 추가([[service-extensibility]]). 현재: `ACADEMY`·`MARKET`·`AGENT`·`YOUTUBE`·`STORE`.
6. **cross-DB는 아래로만.** `service_db → platform_db` 읽기 OK. 옆으로(`academy_db → store_db`) 금지. cross-schema FK 금지([[fk-strategy]]).
7. **strong consistency = 단일 서버 트랜잭션. async = outbox.**
8. **`checkGateB(orgPk, service)`는 서비스를 명시.** 기본값 `"ACADEMY"`. 신규 서비스는 명시 전달.
9. **Gate B는 `status + valid_until` 복합 체크.** `status IN ('ACTIVE','GRACE') AND (valid_until IS NULL OR valid_until > NOW())`. status만 보면 배치 실패 시 영구 무료 위험.
10. **`org_entitlement.feature_limits`가 런타임 한도의 최종 권위.** product_feature·plan_definition은 초기값 복사용. 런타임 직접 조회 금지.
11. **구독은 N:M.** `subscription_item(subscription_pk, sku_pk)`이 진실 원천. `org_subscription`은 헤더만, sku_pk 단일 FK 금지.

파생 표준: `public_id`는 외부 노출 테이블만 · soft-delete 3종(status/deleted_at/append-only, [[delete-patterns]]) · 금액은 정수 minor unit(float 금지).

### 2.2 보안 규율

- **OWASP BOLA**: org_pk 질의 강제를 Drizzle base repo로 프레임워크화(복합 UNIQUE 불요 — 실체는 질의 필터).
- **NIST SP 800-162 환경속성**: api_key `allowed_ip_cidr` + Gateway/WAF IP 강제.
- **감사 불변성(ISMS-P/SOC2)**: app 계정 UPDATE/DELETE 권한 제거 + 해시 체이닝 → 외부 WORM(S3 Object Lock). *트리거는 root가 DROP 가능해 채택 안 함.* ([[audit-hash-chain]])
- **Secret/Key**: KMS(refresh_token·서명PDF), 평문/`.env` git 금지. Firebase Web API Key=공개(App Check+도메인제한), Admin SDK 키=회전 대상.
- **데이터 분류**: PII / 민감(health) / 미성년(guardian) / 결제연계 → 차등 보호. 선별 app-level 암호화(secret·guardian만, phone/email은 KMS+접근통제).
- **감사 2-lane**: 컴플라이언스(보안유의 100%·WORM) vs access 텔레메트리(샘플링) — [[audit-two-lane]].

### 2.3 마이그레이션 상태·목표 (G1~G3)

> 0008로 role 2단 분리가 구현됨. G1/G2 완료, G3 부분.

- **G1. role 2단 분리** (D1) — ✅ **0008**: `platform_role`(OWNER/MEMBER/SERVICE, ⚠️ ADMIN 미채택) + `service_membership.role_code`([[role-capability]]).
- **G2. capability 네임스페이스** (D2) — ✅ **0008**: `ACADEMY.<action>`. ⚠️ CHECK 하드코딩은 [[service-extensibility]]에서 의도적 수용.
- **G3. `organization.org_kind`** (D5) — 🟡 **0008 부분**: `type ENUM('COMPANY','TEAM','PERSONAL')`(ACADEMY 제거). `org_kind VARCHAR+CHECK` 전환은 미완(저빈도라 ENUM 유지).

### 2.4 의도적으로 안 하는 것 (YAGNI) + 도입 트리거

| 미도입 / 거부 | 이유 | 도입 트리거 |
|---|---|---|
| 완전 MSA(auth/billing/role DB 분리) | 분산 트랜잭션 지옥, 현 규모 과도 | (안 함 — 비대칭 분리 유지) |
| DB role/capability 레지스트리 | rule을 DB에 = 디버깅 지옥([[role-as-code]]) | 테넌트 커스텀 롤 실수요 |
| audit `BEFORE UPDATE` 트리거 | root가 DROP 가능 → 위협 못 막음 | (안 함 — 해시/WORM으로 대체) |
| BOLA 복합 UNIQUE `(org_pk, public_id)` | public_id(ULID) 이미 전역 UNIQUE | (불요 — 질의 필터가 실체) |
| api_key `allowed_environments` JSON 블롭 | 불변식 #6(임의 JSON 권한 금지) 충돌 | (거부 — 구조화 scopes로) |
| phone/email 전면 app-level 암호화 | 인덱스·유니크·검색 파괴 | (거부 — secret/guardian만, 나머지 KMS) |
| 휴면(1년) 법적 대응 | 유효기간제 2023 폐지 — 법 의무 아님 | (정정 — 선택적 제품정책) |
| OpenFGA / 정책 엔진 | MVP overkill | 학원 1000+ / 정책 복잡도 폭증 |
| Zanzibar full relation_tuple | delegation_grant로 충분 | delegation 패턴 부족 확인 시 |
| Kafka/Debezium CDC | MQ는 outbox 위에 나중에([[payment-atomicity]]) | ISMS-P/SOC2(T4) |
| platform 공통 usage 카운터 | 카운터는 서비스측(집계만 platform, [[operability]] O4) | cross-service 합산 과금 실수요 |
| schema-per-tenant / 즉시 DB-per-tenant | MySQL db=schema pool 복잡 / 현 규모 운영비만↑ | T1/T2/T4 트리거 |

---

## 3. 결정 색인 → decisions/

> **설계 연혁**: R0~R8 자문(멀티테넌시·cross-service 권한·보안표준·정책동의·자체비교·외부자문)을 거쳐 수렴. **라운드별 상세·기각·대안은 `decisions/`가 보유** — 여기는 색인.

### 3.1 핵심 결정 D1~D12 (구현 상태)

| ID | 결정 | 상태 · 문서 |
|---|---|---|
| D1 | role 2단(`platform_role` + `service_membership`) | ✅ 0008 · [[role-capability]] |
| D2 | capability 네임스페이스 `<service>.<action>` | ✅ 0008 · [[service-extensibility]] |
| D3 | role→action = **코드 상수**(DB 레지스트리 거부) | ✅ · [[role-as-code]] |
| D4 | 서비스 계정 = `platform_role='SERVICE'` + api_key | 🟡 SERVICE ✅ / api_key phase-17 |
| D5 | `org_kind` generic + 서비스는 entitlement | 🟡 0008 부분 |
| D6 | service 식별자 `VARCHAR(50)+CHECK`(온라인 DDL) | ✅ · [[service-extensibility]] |
| D7 | `user_consent_event` append-only | ⚠️ phase-17 · [[pipa-consent]] |
| D8 | `api_key` 하드닝 | ⚠️ phase-17 |
| D9 | `permission_snapshot`은 프론트 read-model만 | 🟡 P1 |
| D10 | 멀티테넌시: RLS 없음→CI 린트 | 🟡 · [[multitenancy-pool]] |
| D11 | 표준보안: BOLA / NIST / 해시→WORM | 🟡 BOLA ✅, 해시·WORM P1 · [[audit-hash-chain]] |
| D12 | 운영보강: 논리 소유권·break-glass·키 cadence | 🟡 · [[operability]] |

> **D6 미적용 사례**: `service` VARCHAR+CHECK 원칙을 `pg_provider`(4곳)·`billing_event.event_type`에 미일관 적용 → phase-17+ ENUM→VARCHAR(50)+CHECK 마이그레이션 대상([[schema-reference]]).

### 3.2 결정 문서 (비교 → 결정)

**경계·구조**: [[design-asymmetry]] · [[identity-billing-access]] · [[service-extensibility]]  
**멀티테넌시·격리**: [[multitenancy-pool]] · [[cross-tenant-separation]] · [[rag-multitenancy]]  
**billing·권한**: [[auth-projection]] · [[payment-atomicity]] · [[decisions/gate-b-billing-grace|gate-b-billing-grace]] · [[firebase-boundary]] · [[role-as-code]]  
**운영 가능성**: [[operator-plane]] · [[audit-two-lane]]

> RAG 검색 품질(Qdrant+Neo4j 이중 색인)·content-api 도메인 레이어링은 academy 서비스 도메인이라 본 핸드북 범위 밖.

---

## 4. 운영 → operability/

> 운영 **모델·절차·신뢰성 계약·관측 KPI**는 [[operability]](O1~O6). 여기는 아키텍처가 강제하는 **운영 규율**만.

- **append-only 강제**: `audit_log`·`user_consent_event`·`payment_ledger`는 INSERT만 — 앱 DB 계정에서 UPDATE/DELETE GRANT 제거([[schema-reference]] §M).
- **권한 변경**: 역할 *배정* 변경 = DB UPDATE + `perm_version` bump(즉시). 역할 *룰*(`ROLE_PERMISSION`) 변경 = **코드 배포**.
- **Break-glass는 silent 금지**: 전건 audit + 만료 자동회수 + 사후 리뷰([[break-glass]] · [[operator-plane]]).
- **운영자는 tenant role이 아님**: 별도 신원 평면 + cross-tenant는 아키텍처 분리·break-glass 경유([[operator-plane]]).
- **정기 배치 필수**: TRIALING→EXPIRED·파티션 추가·sweeper·해시검증 미구현 시 사고([[operability]] O3).
- **secret rotation cadence**: api_key 90/365d · KMS DEK 연1회 · Firebase Admin SDK 180d · DB 계정 180d.

---

## 5. 로드맵 · 준거 · 열린 결정 · 용어

### 5.1 기능 우선순위

**P0 (서비스 최소 요건)**: 멀티DB 연결 · identity/membership/org_relation · `org_entitlement`+3-gate · `user_consent_event`(14세·제3자, PIPA 법적 필수) · `audit_log`+파티션 · `service` VARCHAR+CHECK.  
**P1 (운영 요건)**: `api_key`+하드닝 · 결제 연동(ledger+webhook+outbox) · perm_version 멀티인스턴스 전파 · 감사 해시체이닝 · 열람 fan-out · CI 린트(org_pk 누락) · Qdrant `is_tenant` · Neo4j APOC 차단 · **운영 가능성(operator-plane·audit-2lane·permission-debug)**.  
**P2 (트리거 조건부)**: 외부 WORM(T4) · DB role 레지스트리(커스텀롤) · usage OLAP(T3) · OpenFGA(1000+) · Option B `platform-api`(소비앱 3개+) · DB-per-tenant(T1/T2) · Kafka CDC(T4).

### 5.2 법·표준 준거

> "설계 확정" ≠ "구현 완료". ⚠️는 설계는 됐으나 코드 없음(phase-17). 감사자는 ⚠️를 준수로 읽으면 안 됨.

| 근거 | 설계 | 구현 |
|---|---|---|
| PIPA §15 보유 / §17~18 제3자 / §22 14세 / §35 열람 / §37 철회 | ✅ 설계(§22·§35 일부) | ⚠️ phase-17 (§22 **법적 필수**) |
| 정보통신망법 §50 수신거부 / 국외이전 고지 | ✅ 설계 | ⚠️ phase-17 |
| (구)유효기간제 휴면 | ⛔ | ⛔ 2023 폐지 — 법 의무 아님 |
| NIST SP 800-162 ABAC | 🟡 | 🟡 Subject×Object ✅, Environment P1 |
| OWASP API #1 BOLA | ✅ | ✅ org_pk 질의 강제 |
| ISMS-P/SOC2 | 🟡 | 🟡 append-only ✅, 해시 P1, WORM P2, break-glass P1 |

### 5.3 열린 결정

1. **ROLE_PERMISSION 변경 배포 SLA**(hot-fix vs weekly) — 명문화 필요.
2. 만 14세 미만 본인확인 강도(서명 PDF vs PASS/카카오) — 법무.
3. 감사 외부 WORM 시점(P2 vs ISMS-P 일정).
4. 데이터 리전/주권(단일 Seoul vs 해외 고객 시 분리).
5. 휴면(DORMANT) 채택 여부 — 제품 결정.
6. 결제 정식화 범위(쿠폰/proration/인보이스/세금).
7. 강사 콘텐츠 소유권 ToS 조항 — 법무 확정 필요.
8. 이메일 인증 필수 기능 범위(email_verified=false 차단) — 제품 결정.

### 5.4 용어 사전

| 용어 | 뜻 |
|---|---|
| `platform_db` | identity+billing+product+consent+audit 공통 코어 단일 DB |
| `membership.platform_role` | 테넌트 권위(OWNER/MEMBER/SERVICE) |
| `service_membership.role_code` | 서비스별 도메인 역할(academy.DIRECTOR 등) |
| `org_entitlement` | 런타임 권위 access 상태 — Gate B 유일 진실 원천 |
| `delegation_grant.capability` | 위임 capability(`<service>.<action>`), ReBAC |
| `org_relation` | 조직 계층 — **권한 근거 아님** |
| `user_consent_event` | append-only 동의 이벤트(PIPA) |
| `outbox_event` | async 부수효과 발행원 |
| `perm_version` | 권한 변경 전파 카운터(user_pv/org_pv 분리) |
| 3-gate canXXX | A 소속 · B 이용권 · C 정책 |
| Pool 모델 | 공유 DB + org_pk 행 격리 |
| Option A→B | 공유 패키지 → HTTP 서비스 전환 |
| Break-glass | 승인된 긴급 운영 접근(임퍼소네이션과 다름) |
