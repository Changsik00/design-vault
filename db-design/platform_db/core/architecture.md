# platform_db — 아키텍처 & 의사결정 핸드북

> 작성일: 2026-05-28 · 유지: dennis
>
> **이 문서가 무엇인가**: `platform_db` 설계의 "무엇을·왜·무엇을 일부러 안 했나·운영 중 무슨 일이 생기면 어떻게 하나"를 한 곳에 담은 핸드북. ADR + 설계 결정 + 운영 플레이북을 통합한다.
>
> **누구를 위한 것인가**: 이 코드베이스를 새로 맡는 엔지니어 / 보안·컴플라이언스 검토자 / 다음 서비스(agent·market·store·fitness)를 붙일 개발자
>
> **문서 지도 (총 3개, 이 문서가 진입점)**:
> - **이 문서 `architecture.md`** — 개요·설계 여정(R0~R7)·결정(D1~D12)·자문처리·운영 플레이북·부록 ADR
> - [[schema-reference]] — ERD·스키마·3-gate·billing 흐름·consent 모델
> - [[requirements]] — 요구사항 추적표·BDD 시나리오

---

## 1. 한눈에 — 이 시스템은 무엇인가

회사의 여러 SaaS(academy / agent / market / store / fitness)를 받치는 **공통 플랫폼 코어 DB**다.

```
                  ┌──────────────── platform_db (공통 코어, 단일 InnoDB) ──────────────┐
                  │  identity · authZ(membership/role/capability) · entitlement        │
   academy-api ──▶│  product · billing · consent · audit · api_key                    │◀── @aiagent/db-platform 패키지로만 접근
   agent-api   ──▶│  = "누구 / 어디 소속 / 무엇을 할 수 있나 / 무엇을 구독 / 무엇에 동의"의 SSOT │
   market-api  ──▶└────────────────────────────────────────────────────────────────────┘
   store-api   ──▶   academy_db / agent_db / market_db / store_db  (서비스 도메인, org_pk 격리)
   fitness-api ──▶   + Qdrant(벡터) · Neo4j(그래프) · Redis(캐시) · S3(자산)
```

**한 줄 철학**: *공통은 묶고(strong consistency), 도메인은 뗀다(독립 확장)* — **비대칭 분리**.

> **🛢️ MySQL 사용 배경**: MySQL 8은 회사 표준 인프라 규약에 따른 선택으로 기술적 선호의 결과가 아니다. DB를 자유롭게 선택할 수 있다면 **PostgreSQL을 권장**한다.
>
> | 이유 | 상세 |
> |---|---|
> | **RLS 네이티브 지원** | `CREATE POLICY org_isolation ON table USING (org_pk = ...)` — D10 "MySQL RLS 없음 → CI 린트 보강" 우회로가 처음부터 불필요 |
> | **JSONB GIN 인덱스** | `feature_limits`, `meta_json`, `payload_json`에 `@>` 연산자 인덱스 → MySQL JSON 대비 대폭 빠른 쿼리 |
> | **파티셔닝 완전 지원** | `audit_log.created_at`에 TIMESTAMP 그대로 사용 가능. MySQL 8.0의 `PARTITION BY RANGE COLUMNS` + TIMESTAMP 미지원 버그로 인해 DATETIME 우회가 필요 없어짐 |
> | **`pgcrypto` 내장** | 감사 해시 체이닝(D11 `prev_hash`/`row_hash`)을 DB 네이티브 SHA-256으로 처리. 앱 레이어 불필요 |
> | **`FOR UPDATE SKIP LOCKED`** | outbox_event 워커 lock 경합 없는 네이티브 지원 |
> | **Drizzle PG 방언 풍부** | GENERATED ALWAYS 컬럼, CTE with writes, 더 많은 타입 지원 |
>
> 현재는 MySQL + Drizzle 스택이 확립되어 있어 전환보다 최적화를 선택한다. **D10 "MySQL은 RLS 없음"은 PostgreSQL을 사용했다면 발생하지 않았을 구조적 제약임을 명시한다.**

---

## 2. 설계 여정 — 왜 이렇게 됐나

이 설계는 8라운드의 검토·자문을 거쳐 수렴했다.

| 라운드 | 무엇을 물었나 | 무엇이 바뀌었나 |
|---|---|---|
| **R0** 기존 설계 | identity/billing/권한 골격 ([[design-asymmetry]]·[[multitenancy-pool]]·[[cross-tenant-separation]]·[[identity-billing-access]]) | platform_db(공통)+서비스 DB, entitlement=런타임 권위, 3-gate, 결제 단일 트랜잭션 |
| **R1** 멀티테넌시 | RDB/Qdrant/Neo4j 격리 충분? | org_pk 행 격리 + 분리 트리거 T1~T4. RLS 부재→CI 린트, Qdrant `is_tenant`, Neo4j 멀티홉 강제 |
| **R2** BDD/사용성 | 요구사항을 다 수용하나? | 골격 수용. **최대 갭 발견: 권한 어휘가 academy에 묶임** |
| **R3** cross-service 권한 | role/capability를 어떻게 일반화? | **role 2단 분리** + capability 네임스페이스 + role→action 코드 상수 |
| **R4** 보안 표준 | NIST/OWASP/ISMS-P 충족? | BOLA 프레임워크화, NIST 환경속성, 감사 불변성(해시→WORM). **트리거·JSON블롭 거부** |
| **R5** 정책·동의 | PIPA/정보통신망법은? | `user_consent_event` append-only, 14세 미만, 제3자 제공 4요건, 철회권 |
| **R6** 자체 비교 분석 A | 누락은? | 15건 보완(rate-limit·webhook·fan-out·meta schema·perm 전파·SLA·delegation 경계·JWT·키 cadence). **휴면 법폐지 정정** |
| **R7** 자체 비교 분석 B | 장기 리스크는? | **platform_db 비대화** 경고 → 논리 소유권. break-glass·데이터분류·암호화·Admin SDK 키 |
| **R8** 자문 | awesome-database-design 분류 기준 전체 항목 점검 | membership PK 선언 정정 · Gate B valid_until 인덱스 확정 · pg_provider ENUM 불일치 식별 · external_sub_id·pg_webhook status 인덱스 추가 · billing FK 비대칭 의도 명시 · user_consent_event 파티셔닝 검토 |

> **핵심 전환점은 R2→R3**: "데이터 구조는 멀티서비스 준비됐는데 권한 *언어*가 academy MVP에 고정돼 있다"를 발견하고 role을 2단으로 끌어올린 것이 이 설계의 가장 큰 통찰이다.

---

## 3. 흔들리지 않는 원칙 (설계 불변식)

### 3.1 현재 불변식 — 코드 검증됨, 위반 PR 반려

1. **Firebase = 인증, 인가 = 우리 DB.** `firebase_uid`는 조회 키일 뿐 PK/FK 아님.
2. **내부 PK는 BIGINT, 외부 노출은 ULID(`public_id`).** 시퀀셜 PK를 URL·API에 노출 금지.
3. **모든 도메인 테이블은 `org_pk NOT NULL`.** 예외 없음 — 테넌트 경계는 불변식.
4. **`canXXX()`는 `org_entitlement`만 읽는다.** `payment_ledger` 직접 조회 금지. `org_entitlement`는 billing의 authorization projection이며 Gate B의 유일한 진실 원천.
5. **`service`는 `VARCHAR(50)` + `CHECK`.** ENUM 아님. 새 서비스 추가 = CHECK 목록 1줄 추가. 현재 허용: `ACADEMY`, `MARKET`, `AGENT`, `YOUTUBE`, `STORE`.
6. **cross-DB는 아래로만.** `service_db → platform_db` 읽기 OK. 옆으로(`academy_db → store_db`) 금지.
7. **strong consistency = 단일 서버 트랜잭션. async = outbox.**
8. **`checkGateB(orgPk, service)`는 서비스를 명시한다.** 기본값 `"ACADEMY"`. 신규 서비스는 `service` 파라미터를 명시적으로 전달.
9. **Gate B는 `status + valid_until` 복합 체크.** `status IN ('ACTIVE','GRACE') AND (valid_until IS NULL OR valid_until > NOW())`. status만 보면 배치 실패 시 영구 무료 위험.
10. **`org_entitlement.feature_limits`가 런타임 한도의 최종 권위.** `product_feature`·`plan_definition.default_limits`는 entitlement 생성 시 초기값 복사용. 런타임 한도 판단 시 이 두 테이블 직접 조회 금지.
11. **구독은 N:M.** `subscription_item(subscription_pk, sku_pk)`이 진실 원천. `org_subscription`은 구독 헤더(billing canonical truth)만 담고 sku_pk 단일 FK 보유 금지.

파생 표준: `public_id`는 외부 노출 테이블만 · soft-delete 3종(status / deleted_at / append-only) · 금액은 정수 minor unit(float 금지).

### 3.2 설계 목표 — 단계적 마이그레이션 진행 중

> 이 항목들은 R3 자체 분석 결과 채택된 장기 설계 방향이다.
> **현재 코드와 다르므로** 코드 리뷰 체크리스트가 아닌 *로드맵 체크리스트*로 사용한다.

**G1. role 2단 분리** _(D1 · 현재: 단일 `membership.role` ENUM, 목표: `membership.platform_role` + `service_membership.role_code` 분리)_

> 현재 `membership.role = OWNER | DIRECTOR | TEACHER | MEMBER | STUDENT | PARENT`는 academy 서비스에 특화된 역할 체계다. 멀티서비스 확장 시 새 서비스마다 role이 폭증·충돌한다. 목표 구조: `platform_role`(OWNER/ADMIN/MEMBER — 테넌트 소속 권위)과 `service_membership.role_code`(academy.TEACHER / market.SELLER 등 서비스 도메인 역할)로 분리.

**G2. 권한 어휘 서비스 네임스페이스** _(D2 · 현재: `capability = PUBLISH_VIDEO` 등 평문, 목표: `<service>.<action>`)_

> 현재 `delegation_grant.capability` 값은 academy-only 평문(`PUBLISH_VIDEO`, `APPROVE_VIDEO` 등). 멀티서비스에서 어느 서비스의 권한인지 파싱 불가. 목표: `academy.publish_video`, `market.approve_listing` 등 `<service>.<action>` 네임스페이스. role→action 매핑은 코드 상수(`ROLE_PERMISSION[service][role]`) — DB 레지스트리 저장 금지.

**G3. `organization.type` → service-agnostic `org_kind`** _(D5 · 현재: `type ENUM(ACADEMY, COMPANY, TEAM, PERSONAL)`, 목표: generic kind + `org_entitlement.service` 조합)_

> 현재 `organization.type`에 `ACADEMY`가 포함되어 org 자체가 서비스에 종속됨. 목표: org는 순수 tenant(kind = INDIVIDUAL / TEAM / ENTERPRISE 등), 서비스 접근은 `org_entitlement.service`만으로 판단.

---

## 4. 핵심 결정 12건 (D1~D12)

자문 8라운드(R0~R7)를 통해 수렴한 결정들. 각 결정은 *왜 했고, 무엇을 기각했나*를 함께 적는다.

| ID | 결정 | 구현 상태 | 왜 | 기각/대안 |
|---|---|---|---|---|
| **D1** | role 2단: `platform_role` + `service_membership.role_code` | 🟡 **목표** — 현재: `membership.role` ENUM 단일. phase-17 대상 | 서비스마다 role 어휘·hierarchy·matrix가 달라 단일 글로벌 ENUM은 폭증·충돌 | 단일 `membership.role` ENUM(academy 종속) |
| **D2** | capability 네임스페이스 `<service>.<action>`(`delegation_grant.capability_code`) | 🟡 **목표** — 현재: `capability` 6종 CHECK 고정. phase-17 대상 | academy 어휘(PUBLISH_VIDEO)가 platform에 고착되는 것 방지 | academy-only ENUM |
| **D3** | role→action 매핑은 **코드 상수**, DB 레지스트리는 P2 | ✅ 구현됨 | DB에 rule 넣으면 디버깅 지옥·traceability 붕괴(Stripe/Linear도 코드) | DB `role_capability` 레지스트리 보류(테넌트 커스텀롤 트리거 시만) |
| **D4** | 서비스 계정 = `platform_role='SERVICE'` + `api_key` | 🟡 type='SERVICE' ✅, api_key 테이블 phase-17 | 사람·머신 동일 3-gate, 감사 통합 | 글로벌 `AGENT_*` role / 별도 agent 테이블 |
| **D5** | `organization.org_kind`(generic) + 서비스는 entitlement | 🟡 **목표** — 현재: `type ENUM('ACADEMY','COMPANY','TEAM','PERSONAL')`. phase-17 대상 | org은 tenant, service는 capability — 분리 | `type ENUM(ACADEMY…)` 서비스 종속 |
| **D6** | service 식별자 `VARCHAR(50)+CHECK` | ✅ 구현됨 | 서비스 추가가 온라인 DDL(CHECK 1줄 추가, 테이블 락 없음) | ENUM(변경 시 테이블 락) |
| **D7** | `user_consent_event` **append-only 이벤트** | ⚠️ 설계 확정, phase-17 구현 | mutable boolean이면 반복 on/off 이력 소실→PIPA 분쟁 입증 불가 | `granted boolean + revoked_at` |
| **D8** | `api_key` 하드닝(`allowed_ip_cidr`·`rotated_at`·`revoked_reason`) | ⚠️ 설계 확정, phase-17 구현 | B2B 키 유출 대비(Confused Deputy 방지) | — |
| **D9** | `permission_snapshot`은 프론트 read-model만 | 🟡 설계 확정, 미구현 P1 | 백엔드 can() 캐싱은 ABAC 조건 오답 캐싱 함정 | 백엔드 권한 오라클 캐싱 |
| **D10** | 멀티테넌시: RLS 없음→CI 린트, Neo4j 멀티홉 강제, Qdrant `is_tenant` | 🟡 RLS/CI린트 미완, 나머지 ✅ | MySQL은 RLS 없음(정직한 자인) | per-tenant VIEW(수백 테넌트 비현실) |
| **D11** | 표준보안: BOLA 프레임워크화 / NIST 환경속성 / 감사 해시체이닝→WORM | 🟡 BOLA ✅, 해시체이닝·WORM P1 | 위협모델 기반(OWASP API Top10, ISMS-P) | audit 트리거(root가 DROP 가능→무력), JSON 권한블롭, 복합 UNIQUE 불요 |
| **D12** | 운영보강: 논리 소유권·secret cadence·break-glass·데이터분류·perm 전파 | 🟡 정책 명시 ✅, break-glass·암호화 P1 | platform_db 비대화·운영자 통제·키 cadence | phone/email 전면암호화, 휴면 법의무(2023 폐지) |

> **D6 미적용 사례(R8 자문)**: `service` VARCHAR+CHECK 원칙을 `pg_provider`(org_subscription·payment_ledger·billing_event·pg_webhook_event 4곳)와 `billing_event.event_type`에 미일관 적용. 특히 `pg_provider`는 대형 테이블에 중복 — PG 추가 시 `ALTER MODIFY COLUMN` 잠금이 D6가 피하려던 바로 그것. → phase-17+ ENUM→VARCHAR(50)+CHECK 마이그레이션 대상 추가. (스키마 상세: [[schema-reference]])

---

## 5. 권한 모델 (3-gate)

### 5.1 3-Layer 구조

```
Layer 1 인증  : Firebase Auth (누구냐) — JWT, firebase_uid
Layer 2 소속  : @aiagent/db-platform (어디 소속이냐) — membership tuple
Layer 3 정책  : 각 서비스 CASL ability (무엇을 할 수 있냐)
```

배포는 Option A(공유 패키지) → 트리거 도달 시 Option B(`platform-api` HTTP 서비스). 인터페이스를 미리 깨끗이 둬서 전환은 *구현 교체*.

### 5.2 3-gate `canXXX` (사람·머신 공통)

```
0) principal resolve   HUMAN: Firebase→user.pk / MACHINE: api_key→principal_pk
1) Gate A 소속   membership(principal, org).status = ACTIVE  (+ platform_role)
2) Gate B 이용권 org_entitlement(org, service).status ∈ {ACTIVE, GRACE} + feature_limit
3) Gate C 정책   RBAC : service_membership.role_code → ROLE_PERMISSION[service][role] (코드)
                 ReBAC: delegation_grant(grantee, capability_code, ACTIVE, !expired)
                 ABAC : resource.owner_pk == principal.pk  +  환경(client_ip 등)
→ A ∧ B ∧ C → ALLOW, audit_log 기록
```

- 평가기 단일(CASL). 캐싱은 *입력 블록*(membership/grants/entitlement)만 TTL 60s, **최종 can() 캐싱 금지**.
- `perm_version` self-healing: 권한 변경→bump→`X-Perm-Version` 불일치→snapshot 재요청. **개인=user_pv / org 전체=org_pv 분리**(thundering herd 회피).

### 5.3 RBAC / ABAC / ReBAC 정의

| 모델 | 의미 | 저장 |
|---|---|---|
| RBAC | 직책/소속 | `service_membership.role_code` + 코드 `ROLE_PERMISSION` |
| ABAC | 속성(소유·테넌트·한도·환경) | 리소스 컬럼 + `org_entitlement.feature_limits` + client_ip |
| ReBAC | 관계/위임 | `delegation_grant.capability_code`(임퍼소네이션 금지) |
| 계층 | 조직 위계 | `org_relation`(HQ_BRANCH/HOLDING) — **권한 근거 아님, 명시적 membership만** |

### 5.4 구현 현황

| 함수 | 영역 | 상태 |
|---|---|---|
| `getOrganization`, `getUserProfile`, `getUserByFirebaseUid` | identity | ✅ |
| `getActiveMembership`, `getMembership` | identity | ✅ |
| `getMembershipInviteByToken`, `acceptMembershipInvite` | identity | ✅ |
| `getDelegationGrants`, `bumpPermVersion` | identity | ✅ |
| `getEntitlement`, `getEntitlementByService` | billing | ✅ |
| `getFeatureLimit`, `getFeatureLimitByProduct` | billing | ✅ |
| `getPermissionContext`, `checkGateA` | gates | ✅ |
| `checkGateB(orgPk, service)` | gates | ✅ service 파라미터 지원. 기본값 `"ACADEMY"`. 신규 서비스는 service 명시 필요 |

---

## 6. 데이터 일관성 — 결제·권한이 어긋나지 않는다

### 6.1 Billing Truth → Projection → Authorization

이 설계의 핵심 구조적 결정:

```
PG Webhook (Stripe/Toss 이벤트 스트림)
    ↓
org_subscription  ← Billing Canonical Truth
  (provider, invoice, retry_count, grace_until,
   refund, chargeback, trial, cancel_at_period_end, ...)
    ↓ [이벤트 투영 — 단일 트랜잭션]
org_entitlement   ← Authorization Projection
  (status, valid_until, grace_until, feature_limits)
    ↓
Gate B            ← can_access? 만 판단. subscription 직접 조회 금지
```

**왜 분리하나**:
- `org_subscription`은 billing semantics(provider, invoice, retry 등)가 복잡. Gate B는 "이 org가 지금 서비스를 쓸 수 있나?"만 궁금.
- 권한 체크마다 billing을 직접 읽으면: auth 느려짐 + billing 장애 전파 + 캐싱 복잡.
- `org_entitlement`를 projection으로 분리하면: auth는 단일 테이블 조회, billing 장애 격리, JWT/Redis 캐시 단순화.

**eventual consistency 수용**: PG webhook → entitlement 투영 사이 수 초 지연 허용. 취소 직후 짧은 창에 권한이 살아있을 수 있음 — SaaS 표준 베스트 프랙티스.

**영수증 검증 역할**: 최초 구매 직후 point-in-time 검증, webhook 누락 시 fallback, 사용자 수동 복구("구매 복원") 용도. 구독 lifecycle의 continuous synchronization 역할은 webhook.

### 6.2 결제 단일 트랜잭션

```sql
-- 결제 성공 시 platform_db 단일 트랜잭션:
BEGIN;
  INSERT payment_ledger (CHARGE, idempotency_key);
  UPDATE org_subscription SET status = ...;
  INSERT org_entitlement ... ON DUPLICATE KEY UPDATE status, feature_limits, valid_until;
  UPDATE organization SET perm_version = perm_version + 1;
  INSERT outbox_event (event_type='subscription.activated', ...);
COMMIT;
```

- 단일 InnoDB라 **"결제됐는데 권한 미반영" 창이 0** (2PC·Kafka 불필요). SaaS 최악 UX를 구조적으로 차단.
- 멱등 2중: webhook `event_id` UNIQUE + payment `idempotency_key` UNIQUE.
- 환불은 append-only(REFUND row, 기존 UPDATE 금지). PG는 adapter 추상화(Toss/Stripe/PayPal/Manual).
- async 부수효과(영수증·알림·검색 인덱싱)만 outbox → eventual consistency.

---

## 7. 정책 & 동의

- **`user_consent_event` append-only**: `user × consent_type × terms_version × action(GRANTED/REVOKED) × meta_json × prev_hash/row_hash`. 반복 on/off 이력 전부 보존.
- **consent_type 네임스페이스**: `platform.*`(계정) / `pg.*`(결제 제3자) / 서비스 `*`(도메인 DB).
- **전자서명법 §3**: 약관 체크박스 동의 = 전자서명으로서 서면 서명과 동일한 법적 효력. `user_consent_event` row가 그 기록이며 법적 증거로 사용 가능.
- **`platform.content_ownership`**: TEACHER role 취득 시 표시. "내가 플랫폼에서 만든 콘텐츠의 소유권은 나에게 있으며 이전할 수 있다"에 동의. 미동의 시 admin 데이터 이전 처리 불가.
- **`platform.data_transfer`**: 강사 학원 이전 시 admin 처리 전 강사 본인 동의. 동의 없는 이전 = 법적 근거 없는 데이터 변경.
- **데이터 이전 범위**: 본인 정보(identity·profile·멤버십)는 항상 이전 가능. 학생·수업기록·결제이력은 org 소유로 이전 불가. `org_pk NOT NULL` 격리 구조가 이 경계를 강제함.
- **만 14세 미만(PIPA §22)**: 가입 차단 + 법정대리인 동의(`platform.under14_guardian` + guardian/서명 meta). **법적 필수.**
- **제3자 제공(PIPA §17)**: 4요건(recipient/purpose/items/retention) `meta_json` 정형화 + JSON Schema 검증 + canonical(RFC 8785).
- **철회권(§37)**: REVOKED 이벤트 + 즉시 perm_version 동기화.
- **마케팅(정보통신망법 §50)**: 채널 세분화 옵트인/아웃.
- **보존 5년 + 해시 사슬**(tamper-evident). 약관 본문은 S3/CMS, DB엔 `terms_version`만.
- 4분류 위치 규칙: 계정→platform / 도메인→서비스 DB / 제3자→컨텍스트 분할 / 약관본문→S3.

---

## 8. 멀티테넌시 & 격리

**전략: Pool 모델 (공유 DB + `org_pk` 행 격리)**

| 저장소 | 격리 방법 | 구현 상태 |
|---|---|---|
| MySQL | `org_pk NOT NULL` + 모든 조회 `WHERE org_pk` 필수 | ✅ 스키마 레벨 강제. 모든 gate 함수가 orgPk 필수 파라미터. 🟡 CI 린트 미완 |
| Qdrant | `org_id` payload 필터 강제 | ✅ 색인/검색 모두 `org_id` must 필터. 🟡 `is_tenant` 최적화 마커 미추가 |
| Neo4j | `orgId` 노드 속성 + Cypher 강제 | ✅ 노드 생성·조회·멀티홉 모두 `orgId` 강제. 🟡 APOC 쓰기측 교차 차단 P1 예정 |
| Redis | key prefix `org:{org_pk}:...` | ✅ |

**격리 자가진단 DoD**:
- ① 개발자가 org_pk 빼먹으면 다른 테넌트 데이터 유출되나? → **차단**(Not NULL + gate 함수 필수)
- ② 클라가 tenant_id 위조하면 뚫리나? → **차단**(JWT 검증 후 server-side orgPk 조회)

**분리 트리거** — 하나라도 충족되면 분리 착수:

| 트리거 | 조건 | 행동 |
|---|---|---|
| T1 | 단일 org가 고속 테이블 행의 20% 이상 | 해당 테넌트 전용 DB |
| T2 | 대화/로그 P99 > 500ms | 해당 테이블 별 인스턴스 |
| T3 | usage_log 월 500만+/50GB+ | ClickHouse/BigQuery 이관 |
| T4 | ISMS-P/GDPR 계약 | 물리 격리 DB·리전 + 감사 외부 WORM |

분리 단계: Read replica → 고속 테이블 분리 → DB-per-large-tenant. `org_pk` 단위라 org 무중단 마이그레이션 가능.

cross-tenant 집계는 **아키텍처 분리**(`internal/` 모듈 / `*-admin` 서비스) — **Admin role 금지**([[cross-tenant-separation]], 공격 표면 0).

---

## 9. 보안

- **OWASP BOLA**: org_pk 질의 강제를 Drizzle base repo로 **프레임워크화**(복합 UNIQUE 불요 — 실체는 질의 필터).
- **NIST SP 800-162 환경속성**: api_key `allowed_ip_cidr` 컨텍스트 + Gateway/WAF IP 강제.
- **감사 불변성(ISMS-P/SOC2)**: app 계정 UPDATE/DELETE 권한 제거 + **해시 체이닝** → **외부 WORM**(S3 Object Lock). *트리거는 root가 DROP 가능해 채택 안 함.*
- **Secret/Key**: KMS(refresh_token·서명PDF), 평문/`.env` git 금지. rotation cadence(§12.3). **Firebase: Web API Key=공개(App Check+도메인제한), Admin SDK 키=회전 대상.**
- **데이터 분류**: PII / 민감(fitness health) / 미성년(guardian) / 결제연계 → 차등 보호. 선별 app-level 암호화(secret·guardian만, phone/email은 KMS+접근통제).
- **Break-glass**: §12.4 참조.

---

## 10. 설계 비교 분석 기록 — 채택 / 수정 / 거부

> 자체 비교 분석 결과를 *그대로 받아들이지 않고* 어떻게 판단했는지. 거부·정정이 독립적 판단의 증거다.

### 10.1 채택

| 분석 결과 | 출처 | 채택 형태 |
|---|---|---|
| role 2단 분리 / capability 네임스페이스 | R3 | D1·D2 |
| Qdrant `is_tenant` 인덱스 / Neo4j 멀티홉 강제 | R1 | D10 |
| api_key 하드닝 / 감사 외부 WORM | R4·R7 | D8·D11 |
| platform_db 비대화 → 논리 소유권 | R7 | D12, §12.1 |
| break-glass / 데이터 분류 / Admin SDK 키 회전 | R7 | D12, §9, §12 |
| append-only 동의 / 14세·제3자 메타 | R5 | D7 |

### 10.2 거부 / 정정

| 검토 제안 | 우리 판단 | 이유 |
|---|---|---|
| 권한 `role_capability` DB 레지스트리(P1) | **보류(P2)** | "rule을 DB에 저장 금지" 원칙 충돌. 트리거=테넌트 커스텀롤 |
| audit `BEFORE UPDATE` 트리거로 불변성 | **거부** | root가 트리거 DROP 가능 → 명시한 위협을 못 막음. 해시체이닝/WORM이 정답 |
| BOLA 복합 UNIQUE `(org_pk, public_id)` | **불요** | public_id(ULID)가 이미 전역 UNIQUE. 실체는 질의 시 org_pk 필터 강제 |
| api_key `allowed_environments JSON` 블롭 | **거부** | 불변식 #6(임의 JSON 권한 금지) 충돌. 구조화 scopes로 |
| phone/email **전면** app-level 암호화 | **거부(선별로)** | 인덱스·유니크·검색 파괴. secret/guardian만 암호화, phone/email은 KMS+접근통제 |
| 휴면계정(1년) 법적 의무 대응 | **정정** | 정보통신망법 유효기간제 **2023 폐지** → 법 의무 아님. DORMANT는 선택적 제품정책 |
| Debezium/Kafka CDC 지금 도입 | **연기(P2)** | "MQ는 outbox 위에 나중에" 방침. 컴플라이언스 트리거(T4) 시 |

> 이 표가 보여주는 것: 표준·비교 분석을 *맹종*하지 않고 **우리 원칙·위협모델·규모**에 비춰 걸렀다.

---

## 11. 의도적으로 안 한 것 (YAGNI) + 도입 트리거

| 미도입 | 이유 | 도입 트리거 |
|---|---|---|
| 완전 MSA(auth/billing/role DB 분리) | 분산 트랜잭션 지옥, 현 규모 과도 | (안 함 — 비대칭 분리 유지) |
| OpenFGA / 정책 엔진 | MVP overkill | 학원 1000+ / 정책 복잡도 폭증 |
| DB role/capability 레지스트리 | rule을 DB에 = 디버깅 지옥 | 테넌트 커스텀 롤 실수요 |
| 일반 relation_tuple(Zanzibar full) | delegation_grant로 충분 | delegation 패턴 부족 확인 시 |
| audit `BEFORE UPDATE` 트리거 | root가 DROP 가능 | (안 함 — 해시/WORM으로 대체) |
| Kafka/Debezium CDC | MQ는 outbox 위에 나중에 | ISMS-P/SOC2 감사 요건(T4) |
| websocket 권한 push | 403+폴링 충분 | 동시편집 충돌 실발생 시 |
| platform 공통 usage 카운터 | 사용량은 서비스 DB | cross-service 합산 과금 실수요 시 |
| schema-per-tenant | MySQL은 db=schema, pool 복잡 | (안 함 — T4 도달 시 DB-per-tenant) |
| 즉시 DB-per-tenant | 현 규모 운영비만 상승 | T1/T2 트리거 도달 시 |

---

## 12. 운영 플레이북

> "운영 중 X가 생기면 어떻게 하나" — 인수자가 가장 필요로 할 부분.

### 12.1 platform_db 비대화 대응 (가장 큰 장기 리스크)

**증상**: 단일 DB가 identity+billing+entitlement+product+consent+audit를 다 안아 "second monolith"화.

**예방 (지금부터)**:
- `@aiagent/db-platform`을 **컨텍스트 모듈**로 분리: `identity/`·`access/`·`billing/`·`product/`·`consent/`·`audit/`. 각 모듈만 자기 테이블 쓰기.
- **마이그레이션 소유권**: 컨텍스트별 디렉토리 + 리뷰 오너. 한 PR이 여러 컨텍스트 스키마 동시 변경 → 리뷰 reject.
- 컨텍스트 간 직접 FK 최소화 → 모듈 인터페이스 경유 (= 나중 물리 분리 **절단선**).

**확장 단계**: 논리 분리(지금) → 트래픽/팀 분리 시 컨텍스트 단위 schema/DB 분리([[multitenancy-pool]] 패턴 재사용) → Option B(`platform-api` HTTP 서비스).

### 12.2 멀티테넌시 분리 트리거 운영

단계: Read replica → 고속 테이블 분리(`chat_db`/`analytics_db`) → DB-per-large-tenant.

```
분리 대상 기준:
  T1: 단일 org가 특정 테이블 행수의 20% 이상 → 해당 테넌트 전용 DB
  T2: 고속 테이블 P99 > 500ms → 해당 테이블 별 인스턴스
  T3: usage_log 월 500만 건 또는 50GB+ → OLAP(ClickHouse/BigQuery) 이관
  T4: ISMS-P/GDPR 계약 체결 → 물리 격리 + 감사 외부 WORM
```

`org_pk` 단위 분리라 org 무중단 마이그레이션 가능.

### 12.3 Secret / Key Rotation

| 대상 | Cadence | 누출 시 |
|---|---|---|
| `api_key`(머신) | 권고 90일 / 강제 365일, `rotated_at` 추적, 다중키 무중단 | `status='REVOKED'` 즉시 + 신규 발급 |
| OAuth refresh_token | provider 정책, 만료 시 재인증 | KMS 무효화 + 재연결 |
| KMS DEK | AWS/GCP 자동 연 1회 | 키 폐기 → 재암호화 |
| Firebase Admin SDK 키 | 권고 180일(CI/secret store) | 즉시 회전 + GCP IAM 점검 |
| Firebase Web API Key | 회전 불요(공개) | App Check + authorized domains로 방어 |
| DB 계정 비밀번호 | 180일 rotation. least-privilege per service | 즉시 변경 + 접속 이력 감사 |

### 12.4 Break-glass (긴급 운영 접근)

임퍼소네이션(위임)과 **다름** — 승인된 예외 경로:
1. 운영자가 사유(reason) + 대상 + 짧은 만료로 요청 → **승인자(approver)** 결재.
2. `delegation_grant(capability='platform.break_glass')` 또는 `break_glass_session` 발급.
3. 모든 행위 `audit_log(break_glass=true)` 기록. **절대 silent 금지.**
4. 만료 시 자동 회수 + **사후 리뷰**(SOC2/ISMS-P 단골 점검).

### 12.5 감사 무결성 운영

- 월 1회 배치: `audit_log`·`user_consent_event` 해시 사슬 처음부터 재계산 → 불일치 시 보안 알림.
  > ⚠️ **P1 미구현**: `prev_hash`/`row_hash` 컬럼이 아직 DDL에 없음. 해시 체이닝은 phase-17에서 컬럼 추가 후 활성화. 현재는 append-only + DB 계정 권한으로만 무결성 보장.
- 앱 DB 계정에서 UPDATE/DELETE 권한 제거(DDL 단계 GRANT 점검).
- T4 도달 시 외부 WORM(S3 Object Lock) 적재로 root 위협까지 차단.

### 12.6 정보주체 권리 / 동의 운영

- **열람청구**: platform `user_consent_event` + 각 서비스 consent_event **fan-out 조회**(`internal/`).
- **철회**: REVOKED 이벤트 → perm_version bump → 발송 큐 즉시 차단.
- **탈퇴/삭제**: `identity_user.status='DELETED'` → 30일 후 hard delete. platform이 outbox `user.deleted` 발행 → 각 서비스가 dangling fan-out anonymize. consent는 5년 보존 후 파기, user_pk anonymize.

### 12.7 장애 대응 — 권한이 갑자기 안 먹힐 때

```
1. identity_user.status, membership.status 확인
2. org_entitlement.status / valid_until 확인 → GRACE/EXPIRED 전환 여부
3. org.perm_version vs 클라이언트 X-Perm-Version 비교
4. Redis 캐시 flush: DEL "perm:{user_pk}:{org_pk}"
5. audit_log.result='DENY' 로 최근 30분 조회
```

#### 결제는 됐는데 권한이 안 풀릴 때

```
1. payment_ledger에서 idempotency_key로 SUCCEEDED 확인
2. outbox_event.status='PENDING'인 subscription.activated 이벤트 조회
3. outbox 워커 재시도 또는 수동 org_entitlement UPDATE + perm_version 갱신
```

#### PG webhook 중복 처리 방지

```
pg_webhook_event UNIQUE(pg_provider, event_id) → 중복 webhook은 SKIPPED
status='RECEIVED' → 처리 성공 후 'PROCESSED'
실패 시 'FAILED' + 재큐 (단, idempotency_key로 결제 중복 방지)
```

### 12.8 권한 변경 운영

- 역할 *배정* 변경 = DB UPDATE(즉시, perm_version bump).
- 역할 *룰*(ROLE_PERMISSION) 변경 = **코드 배포**.
- ⚠️ **열린 결정**: ROLE_PERMISSION 변경 배포 SLA(hot-fix vs weekly) — 명문화 필요.

### 12.9 인시던트 대응

- 계정 침해: `identity_user.status='SUSPENDED'` 1회 → **전 서비스 즉시 차단**.
- entitlement 정지: `org_entitlement.status='SUSPENDED'` + org_pv bump → 소속 전원 차단.
- PG/알림 vendor 장애: fallback chain(알림톡→SMS→이메일), 배너 표시.
- YouTube OAuth 만료: `youtube_channel.last_error` + 재연결 배너(서비스 도메인 영역).

### 12.10 마이그레이션 운영

- Drizzle, 컨텍스트별 마이그레이션 디렉토리 + 오너.
- ENUM→VARCHAR+CHECK 같은 안전한 온라인 DDL 우선; 비싼 변경(`org_relation`, `service_membership` 신규)은 스키마 먼저.
- migration SQL에 `IF NOT EXISTS` / `COLUMN EXISTS` 방어 코드 포함.
- Option A(`@aiagent/db-platform`)→Option B(`platform-api` HTTP) 전환 트리거: 소비 앱 3개 동시 / platform 전담 팀 / SOC2·PCI. 인터페이스(`getPermissionContext()` 등) 동일 유지 → 구현만 HTTP로 교체.

### 12.11 정기 운영

| 작업 | 주기 | 방법 |
|---|---|---|
| TRIALING→EXPIRED 전환 | 일 1회 | `UPDATE org_entitlement SET status='EXPIRED' WHERE valid_until < NOW() AND status='TRIALING'` |
| 만료 구독 정리 | 일 1회 | `UPDATE org_entitlement SET status='EXPIRED' WHERE valid_until < NOW() AND status='ACTIVE'` |
| GRACE 구독 전환 | 일 1회 | `valid_until < NOW() AND grace_until > NOW() → status='GRACE'` |
| audit_log 파티션 추가 | 분기 1회 | `ALTER TABLE audit_log ADD PARTITION (PARTITION pYYYYMM VALUES LESS THAN (...))` |
| 만료 초대 토큰 정리 | 일 1회 | `UPDATE membership_invite SET status='EXPIRED' WHERE expires_at < NOW() AND status='PENDING'` |
| 해시 사슬 검증 | 월 1회 | audit_log + user_consent_event 전체 재계산 |

---

## 13. 기능 우선순위

### P0 — 지금 필요 (서비스 최소 요건)

- platform_db + academy_db 멀티DB 연결
- identity·membership·org_relation(스키마) 완성
- `org_entitlement`(수동) + 3-gate `canXXX`
- `user_consent_event`(14세·제3자 포함) — PIPA §22 법적 필수
- `audit_log` + 파티션
- `service` VARCHAR+CHECK (academy/market/agent 3개 이상)

### P1 — 다음 단계 (운영 요건)

- `api_key` + 하드닝(`allowed_ip_cidr`, `rotated_at`, `revoked_reason`)
- 결제 연동 (payment_ledger + pg_webhook + outbox 완전 가동)
- `permission_snapshot` + perm_version 멀티인스턴스 전파
- 감사 해시체이닝
- 정보주체 열람 fan-out (`internal/`)
- 약관 재동의 인터셉터
- NIST 환경속성(Gateway/WAF IP)
- CI 린트: org_pk 누락 쿼리 감지
- Qdrant `is_tenant` 최적화 마커
- Neo4j APOC 쓰기측 교차 테넌트 관계 차단

### P2 — 장기 / 트리거 조건부

- 외부 WORM(S3 Object Lock, T4 트리거 시)
- DB role/capability 레지스트리 (테넌트 커스텀롤 실수요 시)
- usage OLAP (T3 트리거 시)
- OpenFGA / 정책 엔진 (학원 1000+ 시)
- Option B `platform-api` HTTP 서비스 분리 (소비 앱 3개+ 시)
- DB-per-tenant (T1/T2 트리거 시)
- Kafka/Debezium CDC (T4 감사 요건 시)

---

## 14. 법 / 표준 준거

> **읽기 주의**: "설계 확정" ≠ "구현 완료". ⚠️는 설계는 됐으나 아직 코드 없음(phase-17 대상). 감사자는 ⚠️ 항목을 준수로 읽으면 안 됨.

| 근거 | 설계 상태 | 구현 상태 | 비고 |
|---|---|---|---|
| PIPA §15 (보유) | ✅ 설계 | ⚠️ phase-17 | user_consent_event 5년 보존 — 미구현 |
| PIPA §17~18 (제3자 제공) | ✅ 설계 | ⚠️ phase-17 | 4요건 meta_json 정형화 — 미구현(CON-4) |
| PIPA §22 (14세 미만) | ✅ 설계 | ⚠️ **법적 필수** phase-17 | guardian 동의 로직 미구현(CON-3) |
| PIPA §35 (열람) | 🟡 설계 중 | ⚠️ phase-17 | fan-out 조회 미구현 |
| PIPA §37 (철회) | ✅ 설계 | ⚠️ phase-17 | REVOKED 이벤트 미구현(CON-6) |
| 정보통신망법 §50 (수신거부) | ✅ 설계 | ⚠️ phase-17 | 마케팅 옵트아웃 미구현(CON-5) |
| 정보통신망법 국외이전 고지 | ✅ 설계 | ⚠️ phase-17 | `platform.third_party_firebase` 동의 미구현 |
| (구)유효기간제 (휴면) | ⛔ | ⛔ | **2023 폐지 — 법 의무 아님** |
| NIST SP 800-162 (ABAC) | 🟡 설계 중 | 🟡 부분 | Subject×Object ✅. Environment(client_ip) P1 |
| OWASP API #1 (BOLA) | ✅ 설계 | ✅ 구현됨 | org_pk 질의 강제 |
| ISMS-P/SOC2 | 🟡 설계 중 | 🟡 부분 | 감사 append-only ✅. 해시 P1, WORM P2, break-glass P1 |

---

## 15. 열린 결정

1. **ROLE_PERMISSION 변경 배포 SLA** (hot-fix vs weekly) — 유일한 미해결. 운영팀이 "DB 한 줄이면 될 걸 왜 배포 기다리냐" 압박 예상, 정책 명문화 필요.
2. **만 14세 미만 본인확인 강도** — 서명 PDF vs PASS/카카오 본인인증(법무 확인).
3. **감사 외부 WORM 시점** — P2 유지 vs ISMS-P 일정 따라 앞당김.
4. **데이터 리전/주권** — 단일 리전(AWS Seoul) 가정, 해외 고객 시 리전 분리.
5. **휴면(DORMANT) 채택 여부** — 법 의무 아님, 제품 결정.
6. **결제 정식화 범위** — 1회성/쿠폰/proration/인보이스/세금.
7. **강사 콘텐츠 소유권 ToS 조항** — "플랫폼에서 생성된 교육 콘텐츠의 소유권은 해당 콘텐츠를 생성한 강사에게 있으며, 강사 본인 요청 및 소속 학원 동의 시 이전 가능"을 ToS에 명시해야 admin 이전이 법적 근거를 가짐. **법무 검토 후 확정 필요.**
8. **이메일 인증 필수 기능 범위** — 결제·초대·민감 쓰기에서 email_verified=false 차단 여부. 제품 결정 후 Guard 레벨에서 강제.

---

## 16. 용어 사전

| 용어 | 뜻 |
|---|---|
| `platform_db` | identity+billing+product+consent+audit 공통 코어 단일 DB |
| `membership.platform_role` | 테넌트 소속 + 플랫폼 권위(OWNER/ADMIN/MEMBER/SERVICE) |
| `service_membership.role_code` | 서비스별 도메인 역할(`academy.director`, `market.seller` 등) |
| `org_entitlement` | 런타임 권위 access 상태 — canXXX Gate B의 유일 진실 원천 |
| `delegation_grant.capability_code` | 위임 capability(`<service>.<action>`), ReBAC |
| `org_relation` | 조직 계층(HQ_BRANCH/HOLDING). **권한 근거 아님** — 명시적 membership만 |
| `user_consent_event` | append-only 동의 이벤트(PIPA/정보통신망법) |
| `outbox_event` | async 부수효과 발행원(결제 부수효과·fan-out 등) |
| `perm_version` | 권한 변경 전파 카운터(user_pv / org_pv 분리) |
| 3-gate canXXX | A 소속 · B 이용권 · C 정책(RBAC/ABAC/ReBAC) |
| Pool 모델 | 공유 DB + org_pk 행 격리. 테넌트마다 별도 DB 없음 |
| Option A→B | 공유 패키지(`db-platform`) → HTTP 서비스(`platform-api`) 전환 |
| 비대칭 분리 | 공통(platform_db)은 묶고, 도메인은 별도 DB로 분리하는 전략 |
| Break-glass | 승인된 긴급 운영 접근. 임퍼소네이션과 다름 |

---

## 17. 왜 이 설계를 신뢰할 수 있나

> **이 설계는 "예쁜 ERD"가 아니라 8라운드 검토를 거쳐 수렴한 의사결정의 기록이다.**

- 결제↔권한을 단일 트랜잭션으로 묶어 SaaS 최악 UX("결제됐는데 권한 미반영")를 구조적으로 없앴다.
- 권한 어휘를 academy 종속에서 서비스 네임스페이스(`<service>.<action>`)로 끌어올려 5개+ 서비스 확장을 열었다.
- 동의를 append-only 이벤트로 PIPA 분쟁을 법적으로 방어했다.
- **무엇보다, 표준·자문을 맹종하지 않고 우리 원칙·위협모델·규모에 비춰 걸러냈고(§10.2), 일부러 안 한 것과 그 도입 트리거를 명시했으며(§11), 비대화 같은 장기 리스크의 운영 전략을 미리 적어뒀다(§12).** 차단성 미해결은 1건(배포 SLA)뿐이다.

---

## 18. 부록 — 설계 결정 모음

> 본문의 핵심 결정은 각각 **비교 → 결정** 형식의 독립 문서로 정리돼 있다. "무엇을 비교해 이렇게 정했는가"는 아래 문서에서 본다.

**경계·구조**
- [[design-asymmetry]] — platform_db 경계: 완전 MSA vs 모놀리스 vs **비대칭 분리**
- [[identity-billing-access]] — 접근 전략: 직접 Drizzle vs **공유 패키지(A)** vs HTTP 서비스(B)

**멀티테넌시·격리**
- [[multitenancy-pool]] — 단일 DB 행 격리: schema/DB-per-tenant vs **Pool 모델**
- [[cross-tenant-separation]] — cross-tenant 조회: Admin role vs **아키텍처 분리**
- [[rag-multitenancy]] — Qdrant 격리: collection-per-tenant vs **shared + payload 필터**

**billing·권한**
- [[auth-projection]] — billing 직접 조회 vs **authorization projection 분리**
- [[payment-atomicity]] — Kafka + eventual vs **단일 InnoDB 트랜잭션**
- [[decisions/gate-b-billing-grace\|gate-b-billing-grace]] — status-only vs **status + validUntil 복합 체크**
- [[firebase-boundary]] — Firebase Custom Claims로 인가 vs **인가는 우리 DB**
- [[role-as-code]] — DB role 레지스트리 vs **코드 상수**

> RAG 검색 품질(Qdrant+Neo4j 이중 색인)과 content-api 도메인 레이어링은 academy 서비스 도메인이라 본 platform 핸드북 범위 밖이다.
