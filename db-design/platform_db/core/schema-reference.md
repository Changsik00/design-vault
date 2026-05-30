---
type: core
aliases:
  - 스키마 레퍼런스
  - schema reference
  - DDL
tags:
  - platform-db
  - core
  - schema
  - ddl
  - erd
---

# platform_db — 스키마 레퍼런스

> 작성일: 2026-05-28 
> 진입점: [[architecture]] (개요·결정·운영) · 검증: [[requirements]] (요구사항·BDD)
>
> **이 문서**: `platform_db`의 기술 레퍼런스 — DB 토폴로지·ERD·스키마 DDL·3-gate·billing 흐름·멀티테넌시·보안·consent 모델.  
> "왜/운영"은 architecture.md, "요구/검증"은 requirements.md.

---

# A. DB 토폴로지 & 접근 전략

## A.1 DB 구조

```
MySQL 8 (단일 인스턴스)
├── platform_db          ← 모든 서비스 공유 (이 문서 범위)
│     [identity]   organization, identity_user, user_profile
│                  membership, membership_invite, delegation_grant
│                  org_relation, audit_log
│     [billing]    plan_definition, product, product_feature, product_sku
│                  org_entitlement, org_subscription, subscription_item
│                  payment_ledger, pg_webhook_event, outbox_event
│                  billing_event
│     [product]    product, product_feature, product_sku
│     [consent]    user_consent_event  (설계 확정, 미구현 — §I)
│     [api_key]    api_key             (설계 확정, 미구현 — §J)
│
├── academy_db           ← academy-api 전용. 모든 테이블에 org_pk NOT NULL
├── agent_db             ← 설계 확정 (§L 참조)
├── market_db            ← 설계 확정 (§L 참조)
└── store_db             ← 설계 확정 (§L 참조)
```

**cross-DB 방향 규칙**:
- `서비스DB → platform_db` 읽기: **허용**
- `platform_db → 서비스DB`: **금지**
- `academy_db → store_db` (peer): **금지**
- cross-schema FK: **전면 금지** (독립 백업/복원 보장)

> **🛢️ MySQL 사용 배경**: MySQL 8은 회사 표준 인프라 규약에 따른 선택으로 기술적 선호의 결과가 아니다. DB를 자유롭게 선택할 수 있다면 **PostgreSQL을 권장**한다.
>
> | 이유 | 상세 |
> |---|---|
> | **RLS 네이티브 지원** | `CREATE POLICY org_isolation ON table USING (org_pk = ...)` — §G "CI 린트 보강" 우회로가 처음부터 불필요 |
> | **JSONB GIN 인덱스** | `feature_limits`, `meta_json`, `payload_json`에 `@>` 연산자 인덱스 → MySQL JSON 대비 대폭 빠른 쿼리 |
> | **파티셔닝 완전 지원** | `audit_log.created_at`에 TIMESTAMP 그대로 사용 가능. MySQL 8.0의 `PARTITION BY RANGE COLUMNS` + TIMESTAMP 미지원 버그로 인해 §D.8의 DATETIME 우회가 필요 없어짐 |
> | **`pgcrypto` 내장** | 감사 해시 체이닝(`prev_hash`/`row_hash`)을 DB 네이티브 SHA-256으로 처리 — 앱 레이어 불필요 |
> | **`FOR UPDATE SKIP LOCKED`** | outbox_event 워커 lock 경합 없는 네이티브 지원 |
> | **Drizzle PG 방언 풍부** | GENERATED ALWAYS 컬럼, CTE with writes, 더 많은 타입 지원 |
>
> 현재는 MySQL + Drizzle 스택이 확립되어 있어 전환보다 최적화를 선택한다. **"MySQL은 RLS 없음"은 PostgreSQL을 사용했다면 발생하지 않았을 구조적 제약임을 명시한다.**

## A.2 접근 계층

```
소비 앱 (academy-api, agent-api, …)
  ↓ 유일한 경로
@aiagent/db-platform 패키지
  ├── identity/index.ts   (getOrganization, getMembership, …)
  ├── billing/index.ts    (getEntitlement, getFeatureLimit, …)
  └── gates.ts            (getPermissionContext, checkGateA/B)
  ↓
Drizzle ORM → platform_db (MySQL)
```

Drizzle 스키마 직접 참조 금지. 패키지 함수만 호출.

---

# B. 식별자 체계

| 용도 | 타입 | 규칙 |
|---|---|---|
| 내부 PK / JOIN | `BIGINT UNSIGNED AUTO_INCREMENT` | DB 외부 노출 금지 |
| 외부 노출 | `CHAR(26)` ULID (`public_id`) | API 응답, URL에 사용 |
| Firebase UID | `VARCHAR(128)` | 인증 연결 키. PK·FK 아님 |
| API Key | `VARCHAR(255)` prefix+hash | B2B 머신 인증 (§J) |

---

# C. ERD (텍스트)

## C.1 Identity 영역

```
identity_user (1)────(1) user_profile
      │
      │ N
      │
    membership ────── (N) organization (1)
      │                        │
      │                        │
    membership_invite           │ N
                                │
                            org_relation (자기참조, self-ref CHECK)
                                │
                            delegation_grant (grantor_pk, grantee_pk, org_pk)
```

## C.2 Billing 영역

```
organization (1) ────(N) org_entitlement ────(1) product
      │
      │ (1)
      │
org_subscription ────(N) subscription_item ────(1) product
      │
      │ (1)
      │
payment_ledger (append-only)

platform_db 이벤트 버스:
  outbox_event  ← billing 상태 변경 시 INSERT
  billing_event ← 구독 lifecycle 이벤트
  pg_webhook_event ← PG 수신 webhook (멱등 처리)
```

## C.3 관계 요약

| 테이블 | 참조 | FK |
|---|---|---|
| `user_profile.user_pk` | `identity_user.pk` | `fk_up_user` |
| `membership.user_pk` | `identity_user.pk` | `fk_mbr_user` |
| `membership.org_pk` | `organization.pk` | `fk_mbr_org` |
| `delegation_grant.org_pk` | `organization.pk` | `fk_grant_org` |
| `delegation_grant.grantor_pk` | `identity_user.pk` | `fk_grant_grantor` |
| `delegation_grant.grantee_pk` | `identity_user.pk` | `fk_grant_grantee` |
| `org_entitlement.org_pk` | `organization.pk` | `fk_ent_org` |
| `product_feature.product_pk` | `product.pk` | `fk_pf_product` |

---

# D. 스키마 상세

## D.1 identity_user

```sql
CREATE TABLE identity_user (
  pk                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  public_id         CHAR(26)       NOT NULL,              -- ULID (외부 노출 전용)
  firebase_uid      VARCHAR(128),                          -- 인증 연결 키 (PK/FK 아님)
  email             VARCHAR(255),
  email_verified    BOOLEAN NOT NULL DEFAULT FALSE,        -- Firebase JWT email_verified 동기화
  email_verified_at TIMESTAMP,
  phone_e164        VARCHAR(20),                           -- E.164 형식 (+821012345678)
  phone_verified    BOOLEAN NOT NULL DEFAULT FALSE,        -- SMS OTP 또는 Firebase Phone Auth
  phone_verified_at TIMESTAMP,
  type              ENUM('HUMAN','SERVICE','SYSTEM') NOT NULL DEFAULT 'HUMAN',
  status            ENUM('ACTIVE','SUSPENDED','DELETED') NOT NULL DEFAULT 'ACTIVE',
  -- DORMANT: 열린결정(architecture.md §15.5) — 법 의무 아님, 제품 정책 결정 후 ENUM 추가
  perm_version      BIGINT UNSIGNED NOT NULL DEFAULT 1,
  created_at        TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
  deleted_at        TIMESTAMP,
  UNIQUE KEY uq_identity_user_public_id (public_id),
  UNIQUE KEY uq_identity_user_firebase_uid (firebase_uid),
  INDEX idx_identity_user_email (email),
  INDEX idx_identity_user_email_verified (email_verified),
  INDEX idx_identity_user_phone (phone_e164)
);
```

**설계 포인트**:
- `firebase_uid`: 인증 SSOT. 조회 키일 뿐 FK로 참조 금지
- `perm_version`: 권한 변경 시 bump → 프론트 캐시 invalidation
- `type='SERVICE'`: agent/api_key 사용자. 사람과 동일 3-gate
- `email_verified`: Firebase JWT `email_verified` claim을 DB에 동기화 (TTL 1h stale 방어). 이메일 인증 필수 기능(결제·초대) 차단 근거
- `phone_verified`: SMS OTP 인증 완료 여부. `phone_e164`가 있어도 verified=false면 미인증 상태
- 탈퇴 처리 시: `email=NULL, phone_e164=NULL, firebase_uid=NULL, email_verified=FALSE` (hard anonymize, 30일 배치)

## D.2 user_profile

```sql
CREATE TABLE user_profile (
  user_pk      BIGINT UNSIGNED PRIMARY KEY,          -- identity_user.pk 참조 (FK)
  display_name VARCHAR(120) NOT NULL,
  avatar_url   VARCHAR(500),
  locale       VARCHAR(10)  NOT NULL DEFAULT 'ko',
  timezone     VARCHAR(50)  NOT NULL DEFAULT 'Asia/Seoul',
  updated_at   TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
  CONSTRAINT fk_up_user FOREIGN KEY (user_pk) REFERENCES identity_user(pk)
);
```

**설계 포인트**: display_name SSOT. 서비스별 추가 프로필 (`academy_db.teacher_profile` 등)은 서비스 DB에.

## D.3 organization

```sql
CREATE TABLE organization (
  pk           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  public_id    CHAR(26)       NOT NULL,              -- ULID
  type         ENUM('COMPANY','TEAM','PERSONAL') NOT NULL,
  -- 0008: ACADEMY 제거(서비스 종류는 org_entitlement.service가 결정). 향후: org_kind VARCHAR(30)+CHECK로 전환
  slug         VARCHAR(50)    NOT NULL DEFAULT '',
  name         VARCHAR(100)   NOT NULL,
  status       ENUM('ACTIVE','SUSPENDED','CLOSED') NOT NULL DEFAULT 'ACTIVE',
  perm_version BIGINT UNSIGNED NOT NULL DEFAULT 1,
  created_at   TIMESTAMP NOT NULL DEFAULT NOW(),
  deleted_at   TIMESTAMP,
  UNIQUE KEY uq_org_public_id (public_id),
  UNIQUE KEY uq_org_slug (slug),
  INDEX idx_org_status (status)
);
```

**설계 포인트**:
- **0008**: `ACADEMY` 타입 제거 — org는 순수 테넌트, 서비스 종류는 `org_entitlement.service`가 결정(D5 부분 완료).
- ⚠️ **여전히 ENUM**: `org_kind VARCHAR(30)+CHECK` 전환(완전 service-agnostic)은 미완. org.type은 저빈도 변경이라 ENUM 유지가 당장 큰 비용은 아니나, D6 원칙(VARCHAR+CHECK)과는 부분 불일치.

## D.4 membership

```sql
CREATE TABLE membership (
  user_pk       BIGINT UNSIGNED NOT NULL,
  org_pk        BIGINT UNSIGNED NOT NULL,
  platform_role ENUM('OWNER','MEMBER','SERVICE') NOT NULL DEFAULT 'MEMBER',
  -- 0008: 서비스 무관 테넌트 권위만 표현. 도메인 역할은 service_membership.role_code(D.4a)
  status        ENUM('ACTIVE','SUSPENDED') NOT NULL DEFAULT 'ACTIVE',
  created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
  PRIMARY KEY (user_pk, org_pk),
  INDEX idx_membership_org_platform_role (org_pk, platform_role),
  INDEX idx_membership_user (user_pk),
  CONSTRAINT fk_mbr_user FOREIGN KEY (user_pk) REFERENCES identity_user(pk),
  CONSTRAINT fk_mbr_org  FOREIGN KEY (org_pk)  REFERENCES organization(pk)
);
```

**설계 포인트**:
- 복합 PK `(user_pk, org_pk)` → 1 user = N org 멤버십 자연 지원
- **0008**: `role` → `platform_role`(테넌트 권위) + `service_membership`(서비스 도메인 역할) 2단 분리 완료.
- ⚠️ **`ADMIN` 미채택**: 초기 D1 명세는 `OWNER/ADMIN/MEMBER/SERVICE`였으나 구현은 `OWNER/MEMBER/SERVICE`로 단순화(owner 아닌 사람은 전부 MEMBER, 관리 권한은 service_membership 역할/delegation으로 표현). org 레벨 비-owner 관리자 수요가 생기면 `ADMIN` 재도입 재검토.

## D.4a service_membership (0008 신규)

```sql
CREATE TABLE service_membership (
  user_pk    BIGINT UNSIGNED NOT NULL,
  org_pk     BIGINT UNSIGNED NOT NULL,
  service    VARCHAR(50) NOT NULL,              -- 'ACADEMY', 'MARKET', …
  role_code  VARCHAR(50) NOT NULL,              -- 'DIRECTOR', 'TEACHER', 'STUDENT'
  status     ENUM('ACTIVE','SUSPENDED') NOT NULL DEFAULT 'ACTIVE',
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  PRIMARY KEY (user_pk, org_pk, service),
  INDEX idx_service_membership_org_service (org_pk, service),
  INDEX idx_service_membership_user_service (user_pk, service)
);
```

**설계 포인트**:
- 서비스별 도메인 역할 격리(D1) — academy/market/agent 역할 어휘 충돌 방지.
- `role_code VARCHAR` → 새 서비스 추가 시 마이그레이션 없이 확장(role 어휘는 코드 상수 `ROLE_PERMISSION[service]`가 권위).
- 조회: `WHERE user_pk=? AND org_pk=? AND service='ACADEMY'` → `role_code` 반환.
- `membership`과 1:1 대응(같은 user_pk/org_pk) + `service` 차원만 추가.

## D.5 membership_invite

```sql
CREATE TABLE membership_invite (
  pk         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk     BIGINT UNSIGNED NOT NULL,
  email      VARCHAR(255) NOT NULL,
  role_code  VARCHAR(50)  NOT NULL,                   -- 0008: ENUM → VARCHAR. 'DIRECTOR','TEACHER','STUDENT' 등(service_membership.role_code와 동일 어휘)
  token      CHAR(43)     NOT NULL,                   -- URL-safe base64(32bytes)
  status     ENUM('PENDING','ACCEPTED','EXPIRED','REVOKED') NOT NULL DEFAULT 'PENDING',
  invited_by BIGINT UNSIGNED NOT NULL,
  expires_at TIMESTAMP NOT NULL,                      -- 24시간
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  UNIQUE KEY uq_membership_invite_token (token),
  INDEX idx_invite_org_status (org_pk, status),
  INDEX idx_invite_email (email)
);
```

## D.6 delegation_grant (ReBAC)

```sql
CREATE TABLE delegation_grant (
  pk          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  grantor_pk  BIGINT UNSIGNED NOT NULL,
  grantee_pk  BIGINT UNSIGNED NOT NULL,
  org_pk      BIGINT UNSIGNED NOT NULL,
  capability  VARCHAR(100) NOT NULL,
  -- 0008: <service>.<action> 네임스페이스 적용. 향후: capability_code로 컬럼명 변경 검토
  scope_json  JSON,
  status      ENUM('ACTIVE','REVOKED') NOT NULL DEFAULT 'ACTIVE',
  expires_at  TIMESTAMP,
  created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
  CONSTRAINT chk_capability CHECK (capability IN (
    'ACADEMY.PUBLISH_VIDEO','ACADEMY.APPROVE_VIDEO','ACADEMY.VIEW_ALL_LECTURES',
    'ACADEMY.MANAGE_SCHEDULE','ACADEMY.MANAGE_MEMBERS','ACADEMY.VIEW_BILLING'
  )),
  INDEX idx_delegation_grantee_org (org_pk, grantee_pk, status),
  INDEX idx_delegation_grantor (grantor_pk),
  CONSTRAINT fk_grant_org      FOREIGN KEY (org_pk)     REFERENCES organization(pk),
  CONSTRAINT fk_grant_grantor  FOREIGN KEY (grantor_pk) REFERENCES identity_user(pk),
  CONSTRAINT fk_grant_grantee  FOREIGN KEY (grantee_pk) REFERENCES identity_user(pk)
);
```

**설계 포인트**:
- **0008**: `ACADEMY.<action>` 네임스페이스 적용 완료(6종). role→capability 매핑은 코드 상수가 권위.
- ⚠️ **CHECK가 여전히 하드코딩**: 멀티서비스(`MARKET.<action>` 등) 추가 시 CHECK 목록을 마이그레이션으로 확장해야 함 — 네임스페이스는 붙였으나 "마이그레이션 없는 개방"(D6 정신)은 아직 아님.

## D.7 org_relation

```sql
CREATE TABLE org_relation (
  pk              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  parent_org_pk   BIGINT UNSIGNED NOT NULL,
  child_org_pk    BIGINT UNSIGNED NOT NULL,
  relation_type   ENUM('HQ_BRANCH','HOLDING') NOT NULL,
  created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
  CONSTRAINT chk_no_self_ref CHECK (parent_org_pk != child_org_pk),
  UNIQUE KEY uq_org_relation_parent_child (parent_org_pk, child_org_pk),
  INDEX idx_org_relation_child (child_org_pk)
);
```

## D.8 audit_log

```sql
CREATE TABLE audit_log (
  pk            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,  -- AUTO_INCREMENT 필수
  org_pk        BIGINT UNSIGNED,
  actor_type    ENUM('HUMAN','API_KEY','SYSTEM') NOT NULL,
  actor_pk      BIGINT UNSIGNED,
  api_key_pk    BIGINT UNSIGNED,                      -- api_key 구현 후 FK 추가 예정
  action        VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50),
  resource_pk   BIGINT UNSIGNED,
  result        ENUM('ALLOW','DENY','ERROR') NOT NULL,
  ip            VARBINARY(16),                        -- IPv4(4byte) or IPv6(16byte). PIPA 감사 요건
  meta_json     JSON,
  break_glass   BOOLEAN NOT NULL DEFAULT FALSE,        -- P1: break-glass 비상접근 플래그 (ISMS-P §6.4)
  created_at    DATETIME NOT NULL DEFAULT (NOW()),     -- RANGE 파티셔닝 호환 (TIMESTAMP 불가)
  PRIMARY KEY (pk, created_at),                       -- 파티션 테이블 PK = 파티션 키 포함 필수 (MySQL 규칙)
  INDEX idx_audit_org_created (org_pk, created_at),
  INDEX idx_audit_actor (actor_pk, created_at),
  INDEX idx_audit_break_glass (break_glass, created_at) -- break_glass=TRUE 비상접근 이벤트 빠른 조회
) PARTITION BY RANGE COLUMNS(created_at) (
  PARTITION p202601 VALUES LESS THAN ('2026-02-01 00:00:00'),
  PARTITION p202602 VALUES LESS THAN ('2026-03-01 00:00:00'),
  PARTITION p202603 VALUES LESS THAN ('2026-04-01 00:00:00'),
  PARTITION p202604 VALUES LESS THAN ('2026-05-01 00:00:00'),
  PARTITION p202605 VALUES LESS THAN ('2026-06-01 00:00:00'),
  PARTITION p202606 VALUES LESS THAN ('2026-07-01 00:00:00'),
  PARTITION p_future VALUES LESS THAN (MAXVALUE)
);
```

**설계 포인트**:
- 월별 RANGE 파티셔닝 (조회 성능, 파티션 단위 아카이빙)
- `ip VARBINARY(16)`: PIPA 개인정보 처리방침 감사 요건
- `created_at DATETIME` (TIMESTAMP 아님): MySQL 8.0에서 `PARTITION BY RANGE COLUMNS`에 TIMESTAMP 미지원 버그 회피 — PostgreSQL에서는 불필요한 우회
- WORM 원칙: 이 테이블은 INSERT만. UPDATE·DELETE 금지
- ⚠️ **파티션 자동 추가 미구현**: `p_future` 단일 파티션만 존재. 월별 파티션 자동 생성 배치가 없어 약 3개월 뒤 `p_future`가 신규 데이터를 단독 흡수 → INSERT 성능 저하. phase-17에서 월초 `REORGANIZE PARTITION p_future` 배치 스케줄러 필요.

---

## D.9 product

```sql
CREATE TABLE product (
  pk        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  public_id CHAR(26)     NOT NULL,
  code      VARCHAR(50)  NOT NULL,
  service   VARCHAR(50)  NOT NULL,                   -- VARCHAR+CHECK (ENUM 아님)
  name      VARCHAR(100) NOT NULL,
  status    ENUM('ACTIVE','RETIRED') NOT NULL DEFAULT 'ACTIVE',
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
  CONSTRAINT chk_product_service CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE')),
  UNIQUE KEY uq_product_public_id (public_id),
  UNIQUE KEY uq_product_code (code),
  INDEX idx_product_status (status)
);
```

## D.10 product_feature

```sql
CREATE TABLE product_feature (
  pk          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  product_pk  BIGINT UNSIGNED NOT NULL,
  feature_key VARCHAR(50) NOT NULL,
  limit_value BIGINT,                                -- NULL = 무제한
  UNIQUE KEY uq_product_feature (product_pk, feature_key),
  CONSTRAINT fk_pf_product FOREIGN KEY (product_pk) REFERENCES product(pk)
);
```

## D.11 product_sku

```sql
CREATE TABLE product_sku (
  pk            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  product_pk    BIGINT UNSIGNED NOT NULL,
  code          VARCHAR(50) NOT NULL,
  billing_cycle ENUM('MONTHLY','ANNUAL','ONE_TIME','USAGE') NOT NULL,
  price_krw     INT UNSIGNED NOT NULL DEFAULT 0,     -- KRW 원 단위 정수 (float 금지)
  plan_code     VARCHAR(50),
  status        ENUM('ACTIVE','RETIRED') NOT NULL DEFAULT 'ACTIVE',
  created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
  UNIQUE KEY uq_product_sku_code (code),
  INDEX idx_product_sku_product (product_pk),
  INDEX idx_product_sku_plan_code (plan_code)
);
```

## D.12 org_entitlement

```sql
CREATE TABLE org_entitlement (
  pk                   BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk               BIGINT UNSIGNED NOT NULL,
  product_code         VARCHAR(50)     NOT NULL,
  service              VARCHAR(50)     NOT NULL,    -- VARCHAR+CHECK
  status               ENUM('ACTIVE','GRACE','SUSPENDED','EXPIRED') NOT NULL DEFAULT 'ACTIVE',
  source               ENUM('SUBSCRIPTION','PROMO','MANUAL','FREE')  NOT NULL DEFAULT 'FREE',
  feature_limits       JSON,                        -- {"daily_uploads":6, ...}
  valid_until          TIMESTAMP,
  grace_until          TIMESTAMP,
  plan_code            VARCHAR(50),
  current_period_start TIMESTAMP,
  current_period_end   TIMESTAMP,
  updated_at           TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
  CONSTRAINT chk_entitlement_service CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE')),
  UNIQUE KEY uq_org_product (org_pk, product_code),
  INDEX idx_org_service_status (org_pk, service, status, valid_until),  -- Gate B 핫패스 (R8 자문 반영)
  INDEX idx_entitlement_expiry (valid_until, status),  -- 만료 배치: WHERE valid_until < NOW() AND status='ACTIVE'
  CONSTRAINT fk_ent_org FOREIGN KEY (org_pk) REFERENCES organization(pk)
);
```

**설계 포인트**:
- **런타임 권위 access 상태** — `canXXX()`는 이 테이블만 읽음 (`payment_ledger` 직접 조회 금지)
- **feature_limits 우선순위**: `org_entitlement.feature_limits`가 최종 권위(SSOT). `product_feature`·`plan_definition.default_limits`는 entitlement 최초 생성 시 초기값 복사용으로만 사용. 런타임 한도 판단 시 이 두 테이블 직접 조회 금지(불변식 #10).
- **Gate B 체크**: `status IN ('ACTIVE','GRACE') AND (valid_until IS NULL OR valid_until > NOW())`. status만 보면 배치 실패 시 영구 무료 위험(불변식 #9).
- **grace_until**: Gate B 판정 기준. `org_subscription.grace_until`은 빌링 추적용, Gate B는 이 컬럼만 읽음.
- UNIQUE(org_pk, product_code): 한 org가 같은 product를 두 번 entitlement 받을 수 없음
- `GRACE`: 결제 실패 후 유예 기간 (grace_until까지 서비스 유지)

## D.13 org_subscription

```sql
CREATE TABLE org_subscription (
  pk                   BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk               BIGINT UNSIGNED NOT NULL,
  payer_user_pk        BIGINT UNSIGNED NOT NULL,
  -- sku_pk 제거: subscription_item(N:M)이 진실 원천. 불변식 #11
  status               ENUM('TRIALING','ACTIVE','PAST_DUE','CANCELED','EXPIRED') NOT NULL DEFAULT 'TRIALING',
  pg_provider          ENUM('TOSS','STRIPE','PAYPAL','MANUAL') NOT NULL DEFAULT 'MANUAL',
  external_sub_id      VARCHAR(255),               -- PG 측 구독 ID
  trial_ends_at        TIMESTAMP,
  current_period_start TIMESTAMP NOT NULL,
  current_period_end   TIMESTAMP NOT NULL,
  grace_until          TIMESTAMP,                  -- 빌링 추적용. Gate B 판정은 org_entitlement.grace_until 사용
  cancelled_at         TIMESTAMP,
  created_at           TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at           TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
  INDEX idx_org_subscription_org_status (org_pk, status),
  INDEX idx_org_subscription_external_sub_id (external_sub_id)  -- PG webhook이 external_sub_id로 구독 조회
);
```

## D.14 subscription_item

```sql
CREATE TABLE subscription_item (
  pk              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  subscription_pk BIGINT UNSIGNED NOT NULL,
  sku_pk          BIGINT UNSIGNED NOT NULL,         -- product_sku 참조 (N:M 진실 원천)
  quantity        INT NOT NULL DEFAULT 1,
  status          ENUM('ACTIVE','CANCELED') NOT NULL DEFAULT 'ACTIVE',
  UNIQUE KEY uq_sub_sku (subscription_pk, sku_pk),
  CONSTRAINT fk_sub_item_sub FOREIGN KEY (subscription_pk) REFERENCES org_subscription(pk),
  CONSTRAINT fk_sub_item_sku FOREIGN KEY (sku_pk) REFERENCES product_sku(pk)
);
```

**설계 포인트**: `org_subscription`과 `product_sku`의 N:M 연결. 하나의 구독에 여러 SKU 포함 가능(번들). org_subscription의 단일 sku_pk FK는 제거됨 — 이 테이블이 구독 상품의 진실 원천(불변식 #11).

## D.15 plan_definition

```sql
CREATE TABLE plan_definition (
  pk           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  plan_code    VARCHAR(50) NOT NULL,
  display_name VARCHAR(100) NOT NULL,
  default_limits JSON NOT NULL,              -- {"daily_uploads":10, "members":50, ...}
  is_active    BOOLEAN NOT NULL DEFAULT TRUE,
  created_at   TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
  UNIQUE KEY uq_plan_definition_plan_code (plan_code),
  INDEX idx_plan_definition_active (is_active)
);
```

**설계 포인트**: `product_sku.plan_code`가 참조하는 플랜 정의. `default_limits`는 `org_entitlement.feature_limits` 초기값으로 사용.

## D.16 billing_event

```sql
CREATE TABLE billing_event (
  pk              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk          BIGINT UNSIGNED NOT NULL,
  subscription_pk BIGINT UNSIGNED,
  event_type      ENUM('SUBSCRIPTION_START','SUBSCRIPTION_END','PLAN_CHANGE','INVOICE_PAID','INVOICE_FAILED') NOT NULL,
  plan_code       VARCHAR(50) NOT NULL,
  metadata        JSON,
  created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
  INDEX idx_billing_event_org_created_at (org_pk, created_at),
  INDEX idx_billing_event_subscription (subscription_pk)
);
```

**설계 포인트**: 구독 lifecycle 감사 이벤트. `payment_ledger`가 금융 원장이라면 `billing_event`는 구독 상태 변화의 로그. append-only.
- `event_type ENUM(5종)`: lifecycle 이벤트는 늘어남 → D6 동일 논리로 `VARCHAR(50)+CHECK` 전환 대상 (R8 자문). 현재 ENUM 유지, phase-17+ 마이그레이션.
- **FK 없음(의도적)**: billing 도메인 고write·append-only 특성상 FK 잠금 회피 (billing append-only 의도적 설계).

## D.17 payment_ledger

```sql
CREATE TABLE payment_ledger (
  pk              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk          BIGINT UNSIGNED NOT NULL,
  subscription_pk BIGINT UNSIGNED,
  type            ENUM('CHARGE','REFUND','CHARGEBACK','CREDIT') NOT NULL,
  amount_minor    BIGINT NOT NULL,                  -- 정수 minor unit (KRW: 원, USD: cent)
  currency        CHAR(3) NOT NULL DEFAULT 'KRW',
  pg_provider     ENUM('TOSS','STRIPE','PAYPAL','MANUAL') NOT NULL DEFAULT 'MANUAL',
  pg_payment_id   VARCHAR(255),
  idempotency_key VARCHAR(255) NOT NULL,            -- 중복 결제 방지
  status          ENUM('PENDING','SUCCEEDED','FAILED') NOT NULL DEFAULT 'PENDING',
  receipt_url     VARCHAR(512),
  created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
  UNIQUE KEY uq_idempotency_key (idempotency_key),
  INDEX idx_ledger_org_created (org_pk, created_at)
);
```

**설계 포인트**: append-only 금융 원장. UPDATE·DELETE 금지.
- `pg_provider ENUM`: PG 추가 시 대형 테이블 `ALTER MODIFY COLUMN` 잠금 위험 → D6 동일 논리로 `VARCHAR(50)+CHECK` 전환 대상 (R8 자문). org_subscription·billing_event·pg_webhook_event 3곳 동시 마이그레이션 필요.
- **FK 없음(의도적)**: billing 고write·append-only 패턴상 FK 잠금 회피 (billing append-only 의도적 설계).

## D.18 pg_webhook_event

```sql
CREATE TABLE pg_webhook_event (
  pk           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  pg_provider  ENUM('TOSS','STRIPE','PAYPAL') NOT NULL,
  event_id     VARCHAR(255) NOT NULL,
  signature_ok BOOLEAN NOT NULL DEFAULT FALSE,      -- HMAC 서명 검증 결과
  payload_json JSON NOT NULL,
  status       ENUM('RECEIVED','PROCESSED','SKIPPED','FAILED') NOT NULL DEFAULT 'RECEIVED',
  processed_at TIMESTAMP,
  created_at   TIMESTAMP NOT NULL DEFAULT NOW(),
  UNIQUE KEY uq_provider_event (pg_provider, event_id),  -- 멱등 보장
  INDEX idx_pg_webhook_status (status, created_at)       -- 재처리 워커: WHERE status='FAILED' (R8 자문)
);
```

**설계 포인트**:
- `pg_provider ENUM`: `MANUAL` 제외 — 수동 결제는 webhook이 없으므로 의도적 제외 (org_subscription·payment_ledger와 달리 MANUAL 항목 없음)
- `pg_provider ENUM → VARCHAR+CHECK`: D6 동일 논리 적용 대상 (R8 자문). phase-17+ 마이그레이션.
- **FK 없음(의도적)**: billing 고write 패턴상 FK 잠금 회피 (billing append-only 의도적 설계).

## D.19 outbox_event

```sql
CREATE TABLE outbox_event (
  pk             BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  aggregate_type VARCHAR(50) NOT NULL,
  aggregate_pk   BIGINT UNSIGNED NOT NULL,
  event_type     VARCHAR(80) NOT NULL,
  payload_json   JSON NOT NULL,
  status         ENUM('PENDING','SENT','FAILED') NOT NULL DEFAULT 'PENDING',
  created_at     TIMESTAMP NOT NULL DEFAULT NOW(),
  sent_at        TIMESTAMP,
  INDEX idx_outbox_status_created (status, created_at)
);
```

---

# E. 3-gate 인가 모델

## E.1 전체 흐름

```
HTTP 요청
  ↓
FirebaseJwtGuard             → JWT verify, allMemberships custom claims 주입
  ↓                             (firebase_uid · orgPk · service · roleCode 포함)
[GateAGuard — 🧊 Icebox]     → DB 실시간 membership 재검증 (미구현)
  ↓                             현재는 JWT claims로 간접 커버 — 멤버십 취소 후 최대 ~1h stale window
GateBGuard                   → x-org-pk 헤더 → allMemberships 매칭 → checkGateB(orgPk, service)
  ↓                             헤더 없으면 memberships[0] 폴백 (billing 경로 호환)
AcademyPolicyGuard           → Gate C: CASL ability 빌드 (RBAC/ReBAC/ABAC 통합)
  ↓
AbilityGuard / @CheckAbility → can(action, resource) 평가
  ↓ (Sensitive write)
@VerifyOnDb                  → DB 재검증 (stale JWT claims 방어)
  ↓
비즈니스 로직 → audit_log INSERT
```

> ⚠️ **Gate A 현황(솔직히)**: 실시간 DB 멤버십 재검증 가드(`GateAGuard`)는 **미구현(Icebox)**. 현재는 Firebase custom claims(`allMemberships`)로 간접 커버하므로, 멤버십이 취소돼도 **토큰 만료 전(~1h)까지 통과**할 수 있다. 민감 쓰기는 `@VerifyOnDb`(E.5)로 즉시 차단해 이 창을 메운다.

## E.2 Gate별 상세

### Gate A — 소속

```typescript
// platform_role(테넌트 권위) 확인. 서비스 도메인 역할은 service_membership.role_code
membership = await getActiveMembership(userPk, orgPk);   // platform_role, status
// membership.status !== 'ACTIVE' → 403
// ⚠️ 실시간 재검증(GateAGuard)은 Icebox — 현재 JWT claims로 커버, 취소 시 ~1h stale (E.1 주석)
```

### Gate B — 결제

```typescript
entitlement = await getEntitlementByService(orgPk, service);
// status NOT IN ('ACTIVE','GRACE') → 402
// status IN ('ACTIVE','GRACE') but valid_until < NOW() → 402 (배치 실패 방어)
const now = new Date();
const pass =
  (entitlement.status === 'ACTIVE' || entitlement.status === 'GRACE') &&
  (entitlement.validUntil === null || entitlement.validUntil > now);
if (!pass) throw PaymentRequiredException();
```

**구현**: `checkGateB(orgPk, service: EntitlementService = "ACADEMY")` — service 파라미터로 다중 서비스(ACADEMY/MARKET/AGENT/YOUTUBE/STORE) 지원. 미전달 시 ACADEMY 기본값. 신규 서비스는 service 파라미터를 명시적으로 전달해야 함.

**인덱스**: `idx_org_service_status (org_pk, service, status, valid_until)` — R8 자문 반영, valid_until 포함 확정(불변식 #9). DDL §D.12 수정 완료.

### Gate C — 정책 (CASL)

```typescript
// RBAC: service_membership.role_code → ROLE_PERMISSION[service][roleCode] (코드 상수)
// ReBAC: delegation_grant.capability (0008: 'ACADEMY.<action>' 네임스페이스)
// ABAC: resource 소유권 (lecture.teacher_pk === userPk)
ability = buildAbility(serviceRole, delegationGrants, entitlement);
if (!ability.can(action, resource)) throw ForbiddenException;
```

## E.3 perm_version 동기화

역할 변경 시 `organization.perm_version`을 bump한다. Firebase custom claims 기반이라 stale window(~1h)가 존재하며, 즉시 반영이 필요하면 클라이언트가 `forceRefresh(true)`를 호출한다.

```
학원장: PATCH /members/:id/role
  → bumpPermVersion(orgPk) → UPDATE organization SET perm_version = perm_version + 1 → 200
다음 토큰 갱신(~1h) 시 custom claims 반영 / 즉각 적용은 forceRefresh(true)
```

## E.4 위임 권한 (delegation_grant)

```
학원장: POST /delegation-grants { grantee: teacherPk, capability: 'ACADEMY.MANAGE_MEMBERS' }
  → INSERT delegation_grant → bumpPermVersion(orgPk) → 201
강사가 회원 관리 API 호출
  → Gate C: ability.can('ACADEMY.MANAGE_MEMBERS', ...) → delegation에 있으면 허용
```

## E.5 Sensitive Write (DB 재검증)

```
JWT claims는 TTL 1h 동안 stale 가능 (revoke 후에도 유효)
→ publish/delete/결제 등 sensitive write는 @VerifyOnDb로 DB 최신 상태 재확인
→ perm_version 불일치 → 강제 refresh. Gate A Icebox의 stale window를 메우는 핵심 방어선.
```

## E.6 fail-closed 원칙

```typescript
// JWT claims 파싱 실패 또는 permission_version 불일치 → 무조건 401
// DB 재검증 중 오류 → 503 (허용 아님)
// 모든 권한 판단: deny-by-default, 명시적 allow만 통과
```

---

# F. Billing 흐름

## F.1 결제 → 권한 갱신 (단일 트랜잭션)

```sql
BEGIN;
  -- 1. 결제 원장 기록 (append-only)
  INSERT INTO payment_ledger (org_pk, subscription_pk, type, amount_minor, idempotency_key, status)
  VALUES (?orgPk, ?subPk, 'CHARGE', ?amount, ?idempotencyKey, 'SUCCEEDED');

  -- 2. 구독 상태 갱신
  UPDATE org_subscription
  SET status='ACTIVE', current_period_start=?, current_period_end=?
  WHERE pk=?;

  -- 3. 권한 활성화 (upsert)
  INSERT INTO org_entitlement (org_pk, product_code, service, status, source, feature_limits, valid_until)
  VALUES (?orgPk, ?productCode, ?service, 'ACTIVE', 'SUBSCRIPTION', ?limits, ?validUntil)
  ON DUPLICATE KEY UPDATE
    status='ACTIVE', feature_limits=VALUES(feature_limits), valid_until=VALUES(valid_until);

  -- 4. perm_version 갱신 → 클라이언트 캐시 invalidation
  UPDATE organization SET perm_version = perm_version + 1 WHERE pk=?;

  -- 5. async 부수효과 (알림·영수증·분석)
  INSERT INTO outbox_event (aggregate_type, aggregate_pk, event_type, payload_json)
  VALUES ('subscription', ?subPk, 'subscription.activated', ?payload);
COMMIT;
```

## F.2 구독 상태 머신

```
TRIALING ──결제 성공──▶ ACTIVE
ACTIVE   ──결제 실패──▶ PAST_DUE
PAST_DUE ──유예기간──▶ GRACE (org_entitlement)
PAST_DUE ──유예만료──▶ EXPIRED
ACTIVE   ──취소 요청──▶ CANCELED
```

## F.3 entitlement 상태별 접근 정책

| entitlement.status | Gate B 결과 | 사용자 경험 |
|---|---|---|
| ACTIVE | 통과 | 정상 |
| GRACE | 통과 | "결제 실패 — XX일 내 갱신 필요" 배너 |
| SUSPENDED | 차단 (402) | "서비스 정지" |
| EXPIRED | 차단 (402) | "구독 만료" |

## F.4 PG 웹훅 멱등 처리

```
1. pg_webhook_event에 UNIQUE(pg_provider, event_id) 로 중복 수신 차단
2. signature_ok=FALSE이면 SKIPPED (처리 건너뜀, 기록은 남김)
3. 처리 완료 후 status='PROCESSED', processed_at=NOW()
4. 같은 event_id 재수신 → INSERT 실패 → 앱에서 200 OK 반환 (PG 재전송 방지)
```

---

# G. 멀티테넌시 격리

## G.1 MySQL (platform_db) — 구현 현황

모든 테이블: `org_pk` 컬럼으로 행 격리.

| 항목 | 상태 | 비고 |
|---|---|---|
| 모든 테이블 `org_pk NOT NULL` | ✅ 구현됨 | `packages/db-platform/src/*/schema.ts` 검증 완료 |
| Gate 함수 전체 `orgPk` 파라미터 필수 | ✅ 구현됨 | `getActiveMembership(userPk, orgPk)` 등 |
| cross-tenant 쿼리 CI 린트 | 🟡 미구현 (P1) | `@aiagent/db-platform` 패키지 내 쿼리 정적 분석 미완 |

```sql
-- 올바른 패턴: 모든 쿼리에 org_pk 필터
SELECT * FROM org_entitlement
WHERE org_pk = ?orgPk AND service = ?service AND status IN ('ACTIVE','GRACE');

-- 금지 패턴 (CI 린트 P1 구현 후 빌드 차단 예정)
SELECT * FROM org_entitlement WHERE service = ?service; -- org_pk 누락
```

## G.2 Qdrant (벡터) — 구현 현황

| 항목 | 상태 | 비고 |
|---|---|---|
| 모든 point에 `org_id` payload 포함 | ✅ 구현됨 | `qdrant-index.adapter.ts` |
| 모든 search에 `org_id` must filter | ✅ 구현됨 | `qdrant-search.service.ts` |
| `is_tenant` 마커 payload | 🟡 미추가 (P1) | 컬렉션 격리 감사용 |

```typescript
// apps/academy-api/src/infrastructure/qdrant/qdrant-index.adapter.ts
// 인덱싱 시 org_id 필수 포함 (확인됨 ✅)
await this.client.upsert(collection, {
  points: [{
    id: chunk.chunkId,
    vector: embedding,
    payload: {
      org_id: chunk.orgPublicId,  // ← 테넌트 격리 키
      chunk_id: chunk.chunkId,
      lecture_id: chunk.lectureId,
      // ...
    },
  }],
});

// apps/academy-api/src/infrastructure/qdrant/qdrant-search.service.ts
// 검색 시 org_id must filter 강제 (확인됨 ✅)
const filters: QdrantCondition[] = [
  { key: "org_id", match: { value: orgPublicId } },  // ← 필수, 생략 불가
  ...scopeFilters,
];
const result = await this.client.search(collection, {
  vector: queryEmbedding,
  filter: { must: filters },
  limit,
});
```

## G.3 Neo4j (그래프) — 구현 현황

| 항목 | 상태 | 비고 |
|---|---|---|
| 노드 생성 시 `orgId` 속성 포함 | ✅ 구현됨 | `neo4j-concept.adapter.ts` |
| 모든 쿼리에 `orgId` 필터 | ✅ 구현됨 | MATCH 절에 `{orgId: $orgId}` 필수 |
| 멀티홉 traversal 양 끝 `orgId` | ✅ 구현됨 | 중간 노드까지 검증 |
| APOC write-side cross-org 차단 | 🟡 미구현 (P1) | `apoc.merge.relationship` 호출 시 |

```cypher
// apps/academy-api/src/infrastructure/neo4j/neo4j-concept.adapter.ts
// 노드 생성: MERGE 키에 orgId 포함 (확인됨 ✅)
MERGE (c:LectureChunk { chunkId: $chunkId, orgId: $orgId })
ON CREATE SET c.lectureId = $lectureId, c.createdAt = datetime()

MERGE (concept:Concept { name: $name, orgId: $orgId })
ON CREATE SET concept.createdAt = datetime()

// 단홉 읽기: orgId 필터 (확인됨 ✅)
MATCH (chunk:LectureChunk { orgId: $orgId })
      -[:HAS_CONCEPT]->(concept:Concept { orgId: $orgId })
WHERE chunk.lectureId = $lectureId
RETURN concept.name AS name

// 멀티홉: 경계 노드 양 끝 모두 orgId 체크 (확인됨 ✅)
MATCH (a:Concept { orgId: $orgId })-[:RELATED_TO*1..3]->(b:Concept { orgId: $orgId })
WHERE a.name = $conceptName
RETURN b.name AS related, count(*) AS weight
ORDER BY weight DESC LIMIT 10
```

---

# H. 보안 표준

## H.1 BOLA (OWASP API1) 방어

```typescript
// org_pk는 클라이언트가 제공하지 않음 — 토큰에서 바인딩
// 리소스 조회 시 항상 org_pk 조건 포함
const lecture = await db.query.lecture.findFirst({
  where: and(eq(lecture.pk, lectureId), eq(lecture.orgPk, orgPk))
  //          ^ 리소스 ID           ^ 테넌트 경계 필수
});
if (!lecture) throw NotFoundException; // 존재 여부 비공개 (404)
```

## H.2 감사 로그 불변성

```
audit_log: INSERT 전용. UPDATE·DELETE 금지.
파티션 DROP 금지 (아카이빙 필요 시 외부 WORM 스토리지로 EXPORT 후 DROP).
미래: prev_hash/row_hash 해시 체이닝 + 외부 WORM 연동 (ISMS-P 심사 트리거 시)
```

## H.3 암호화 경계

| 데이터 | 처리 | 비고 |
|---|---|---|
| 비밀번호 | Firebase Auth 관리 (bcrypt) | 직접 저장 안 함 |
| OAuth refresh_token | AWS KMS envelope encryption | youtube_channel.oauth_refresh_token_kms |
| api_key secret | bcrypt hash 저장 (prefix만 평문) | §J 참조 |
| IP 주소 | VARBINARY(16) raw 저장 | PIPA 처리방침 |
| 결제 카드 정보 | PG 측 보관 (PCI-DSS) | 직접 저장 안 함 |

## H.4 secret 로테이션 정책

| 대상 | 주기 | 방법 |
|---|---|---|
| DB 접속 계정 비밀번호 | 90일 | AWS Secrets Manager 자동 |
| API Key | 90일 권장, 만료 강제 없음 | `rotated_at` 필드로 추적 |
| Firebase Service Account | 연 1회 | GCP IAM 수동 |
| OAuth refresh_token | 정기적 재발급 | youtube 채널 연결 시 갱신 |

---

# I. Consent 모델

## I.1 배경 — PIPA 법적 요건

| 조항 | 요건 |
|---|---|
| §22 수집·이용 동의 | 이용 목적·항목·기간 명시, 필수/선택 분리 |
| §17 제3자 제공 | 제공 목적·대상·항목·기간 별도 동의 (Firebase Auth 국외이전 포함) |
| §37 철회권 | 동의 철회 즉시 처리, 철회 이후 처리 금지 |
| §22③ 14세 미만 | 법정대리인 동의 필수 |

mutable boolean flag로는 반복 on/off 이력 소실 → PIPA 분쟁 시 입증 불가.  
**→ append-only 이벤트 테이블** 구조 채택.

## I.2 user_consent_event 설계

**consent_type 네임스페이스** (전자서명법: 약관 체크박스 동의 = 서면 서명과 동일 효력):

| consent_type | 분류 | 시점 | 필수? |
|---|---|---|---|
| `platform.terms_of_service` | 서비스 이용약관 | 가입 | ✅ 필수 |
| `platform.privacy_policy` | 개인정보 처리방침 | 가입 | ✅ 필수 |
| `platform.third_party_firebase` | Firebase Auth 국외이전 (PIPA §17) | 가입 | ✅ 필수 |
| `platform.marketing_email` | 이메일 마케팅 수신 | 가입 | 선택 |
| `platform.marketing_sms` | SMS 마케팅 수신 | 가입 | 선택 |
| `platform.under14_guardian` | 14세 미만 법정대리인 동의 (PIPA §22③) | 가입 | 법적 필수 |
| `platform.content_ownership` | **강사 콘텐츠 소유권 약관** — "내가 만든 콘텐츠는 내 소유" | TEACHER role 취득 시 | 선택 (이전 시 필요) |
| `platform.data_transfer` | **데이터 이전 동의** — admin 강사 이전 처리 전 강사 본인 동의 | 이전 요청 시 | 이전 요청 시 필수 |
| `platform.withdrawal` | 탈퇴 최종 확인 | 탈퇴 요청 시 | 탈퇴 시 필수 |
| `pg.toss_third_party` | Toss 결제 제3자 제공 | 첫 Toss 결제 시 | 결제 시 필수 |
| `pg.stripe_third_party` | Stripe 결제 제3자 제공 | 첫 Stripe 결제 시 | 결제 시 필수 |
| `[service].*` | 서비스별 약관 (academy_db 등 서비스 DB에서 관리) | 서비스 최초 이용 | 서비스별 |

> **전자서명법 근거**: 약관에 체크박스로 동의 = 전자서명법 §3의 전자서명으로서 서면 서명과 동일한 법적 효력. `user_consent_event`가 그 기록.

```sql
CREATE TABLE user_consent_event (
  pk            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_pk       BIGINT UNSIGNED NOT NULL,
  consent_type  VARCHAR(50) NOT NULL,          -- 위 네임스페이스 참조
  action        ENUM('GRANTED','REVOKED') NOT NULL,
  version       VARCHAR(20) NOT NULL,               -- 약관 버전 (예: '2026-05-01')
  ip            VARBINARY(16),
  user_agent    VARCHAR(500),
  meta_json     JSON,                               -- P1: PIPA §17 4요건 정형화 (제3자 제공 시 제공받는자·목적·항목·보유기간)
  prev_hash     CHAR(64),                           -- P1: 직전 row SHA-256 (해시 체이닝 무결성)
  row_hash      CHAR(64),                           -- P1: 현재 row 전체 SHA-256 (위변조 감지)
  created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
  INDEX idx_consent_user_type (user_pk, consent_type, created_at)
  -- UPDATE·DELETE 금지 (append-only)
);
```

> **P1 해시 컬럼 참고**: `prev_hash`/`row_hash`는 DDL에는 있으나 phase-17 구현 전까지 애플리케이션에서 NULL로 삽입. 해시 사슬 검증 배치(§12.5)는 컬럼 활성화 이후 운영.

> **파티셔닝 검토(P2, R8 자문)**: append-only + 5년 보존 후 파기 패턴은 `audit_log`와 동일. 파티션 DROP이 5년 후 파기의 가장 깔끔한 구현 → `audit_log` 월별 RANGE 파티셔닝 동일 패턴 적용 고려.

**현재 상태 쿼리** (최신 동의 상태):
```sql
SELECT action FROM user_consent_event
WHERE user_pk=? AND consent_type=?
ORDER BY created_at DESC LIMIT 1;
-- 결과 'GRANTED' = 동의 상태, 'REVOKED' = 철회 상태, 없음 = 미동의
```

## I.3 BDD 시나리오

```gherkin
Scenario: 가입 시 PIPA 필수 동의 누락 차단
  Given 김지영이 가입 페이지에 있다
  When 필수 약관 중 하나라도 체크 없이 "가입" 버튼을 누른다
  Then 가입이 차단된다 ("필수 약관에 동의해 주세요")
  And user_consent_event에 아무 row도 INSERT되지 않는다

Scenario: 정상 가입 시 동의 이벤트 기록
  When 모든 필수 약관에 동의하고 가입한다
  Then user_consent_event에 다음 row들이 INSERT된다:
    | consent_type                  | action  | version      |
    | platform.terms_of_service     | GRANTED | 2026-05-01   |
    | platform.privacy_policy       | GRANTED | 2026-05-01   |
    | platform.third_party_firebase | GRANTED | 2026-05-01   |

Scenario: 마케팅 수신 동의 철회
  Given 김지영이 마케팅 동의를 GRANTED한 상태다
  When 김지영이 "개인정보 처리 → 마케팅 수신 거부"를 클릭한다
  Then user_consent_event에 (consent_type='platform.marketing_email', action='REVOKED') row가 INSERT된다
  And 기존 GRANTED row는 변경되지 않는다 (append-only)

Scenario: fan-out 익명화 (탈퇴 요청)
  Given 김지영이 회원 탈퇴를 요청한다
  When 탈퇴 처리가 실행된다
  Then user_consent_event에 (consent_type='ALL', action='REVOKED') row가 INSERT된다
  And identity_user.status='DELETED', deleted_at=NOW()
  And 서비스 DB의 개인식별 컬럼들이 익명화된다 (fan-out)
  And audit_log에 actor_type='SYSTEM', action='USER_DELETION' 기록된다
```

---

# J. api_key 테이블

```sql
CREATE TABLE api_key (
  pk               BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk           BIGINT UNSIGNED NOT NULL,
  user_pk          BIGINT UNSIGNED NOT NULL,     -- 키 소유자 identity_user (type='SERVICE')
  key_prefix       VARCHAR(10) NOT NULL,         -- 평문 prefix (예: 'ak_live_')
  secret_hash      VARCHAR(255) NOT NULL,        -- bcrypt hash
  scopes           JSON NOT NULL,                -- ["lecture:read", "membership:write"]
  allowed_ip_cidr  VARCHAR(50),                 -- 허용 IP 범위 (NIST 환경속성)
  allowed_services JSON,                         -- ["ACADEMY", "MARKET"]
  rotated_at       TIMESTAMP,                   -- 마지막 rotation 시각
  last_used_at     TIMESTAMP,
  expires_at       TIMESTAMP,
  revoked_at       TIMESTAMP,
  revoked_reason   VARCHAR(255),
  created_at       TIMESTAMP NOT NULL DEFAULT NOW(),
  INDEX idx_api_key_org (org_pk),
  INDEX idx_api_key_user (user_pk)
);
```

**설계 포인트**:
- 동일 org에 다중 키 허용 (rotation 중 overlap 지원)
- `allowed_ip_cidr`: Confused Deputy 방어 (NIST SP 800-162 환경속성)
- `secret_hash`: bcrypt. 평문은 발급 시 1회만 노출
- 3-gate 동일 적용: api_key로 인증해도 Gate A/B/C 동일 통과 필수

---

# K. service 확장 방법

신규 서비스(`STORE`, `FITNESS` 등) 추가 시 절차:

```sql
-- migration 파일 추가 (예: 0005_add_fitness_service.sql)
ALTER TABLE org_entitlement
  DROP CONSTRAINT chk_entitlement_service,
  ADD CONSTRAINT chk_entitlement_service
    CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE','FITNESS'));

ALTER TABLE product
  DROP CONSTRAINT chk_product_service,
  ADD CONSTRAINT chk_product_service
    CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE','FITNESS'));
```

**온라인 DDL**: CHECK constraint 추가/변경은 InnoDB에서 테이블 락 없음.  
**ENUM과의 차이**: ENUM은 `MODIFY COLUMN` 필요 → 대형 테이블 잠금 유발.

---

# L. 서비스 DB 템플릿

각 서비스 DB는 동일한 격리 원칙을 따른다: **모든 테이블에 `org_pk NOT NULL`**, cross-schema FK 금지.

## L.1 academy_db (현재 구현됨)

```sql
-- 모든 테이블 공통 패턴
CREATE TABLE lecture (
  pk         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk     BIGINT UNSIGNED NOT NULL,           -- ← 필수. 격리 기준
  public_id  CHAR(26) NOT NULL,
  teacher_pk BIGINT UNSIGNED NOT NULL,           -- platform_db.identity_user.pk 참조 (FK 없음)
  title      VARCHAR(200) NOT NULL,
  status     ENUM('DRAFT','PUBLISHED','ARCHIVED') NOT NULL DEFAULT 'DRAFT',
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  INDEX idx_lecture_org_status (org_pk, status)
  -- cross-schema FK 금지: org_pk → platform_db.organization.pk (앱 레이어 검증)
);
```

## L.2 agent_db (설계 확정)

```sql
-- agent_db: AI 에이전트 실행 기록
CREATE TABLE agent_session (
  pk         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk     BIGINT UNSIGNED NOT NULL,
  user_pk    BIGINT UNSIGNED NOT NULL,
  agent_type VARCHAR(50) NOT NULL,
  status     ENUM('RUNNING','COMPLETED','FAILED','CANCELED') NOT NULL DEFAULT 'RUNNING',
  input_json JSON,
  output_json JSON,
  tokens_used INT UNSIGNED NOT NULL DEFAULT 0,
  started_at TIMESTAMP NOT NULL DEFAULT NOW(),
  ended_at   TIMESTAMP,
  INDEX idx_agent_session_org (org_pk, started_at),
  INDEX idx_agent_session_user (user_pk, started_at)
);

CREATE TABLE agent_tool_call (
  pk          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  session_pk  BIGINT UNSIGNED NOT NULL,
  org_pk      BIGINT UNSIGNED NOT NULL,           -- 비정규화 (격리 쿼리 최적화)
  tool_name   VARCHAR(100) NOT NULL,
  input_json  JSON,
  output_json JSON,
  duration_ms INT UNSIGNED,
  created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
  INDEX idx_tool_call_session (session_pk),
  INDEX idx_tool_call_org (org_pk, created_at)
);
```

## L.3 market_db (설계 확정)

```sql
-- market_db: 마켓플레이스 상품·주문
CREATE TABLE market_item (
  pk          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk      BIGINT UNSIGNED NOT NULL,           -- 판매 org
  public_id   CHAR(26) NOT NULL,
  seller_pk   BIGINT UNSIGNED NOT NULL,
  title       VARCHAR(200) NOT NULL,
  price_krw   INT UNSIGNED NOT NULL DEFAULT 0,
  status      ENUM('DRAFT','ACTIVE','SOLD_OUT','RETIRED') NOT NULL DEFAULT 'DRAFT',
  created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
  UNIQUE KEY uq_market_item_public_id (public_id),
  INDEX idx_market_item_org_status (org_pk, status)
);

CREATE TABLE market_order (
  pk          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk      BIGINT UNSIGNED NOT NULL,           -- 구매 org
  buyer_pk    BIGINT UNSIGNED NOT NULL,
  item_pk     BIGINT UNSIGNED NOT NULL,
  ledger_pk   BIGINT UNSIGNED,                    -- platform_db.payment_ledger.pk (FK 없음)
  status      ENUM('PENDING','PAID','CANCELED','REFUNDED') NOT NULL DEFAULT 'PENDING',
  created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
  INDEX idx_market_order_org (org_pk, created_at),
  INDEX idx_market_order_buyer (buyer_pk, created_at)
);
```

## L.4 store_db (설계 확정)

```sql
-- store_db: 디지털 스토어 (템플릿·콘텐츠 판매)
CREATE TABLE store_product (
  pk          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  org_pk      BIGINT UNSIGNED NOT NULL,
  public_id   CHAR(26) NOT NULL,
  product_type ENUM('TEMPLATE','COURSE','ASSET') NOT NULL,
  title       VARCHAR(200) NOT NULL,
  price_krw   INT UNSIGNED NOT NULL DEFAULT 0,
  status      ENUM('DRAFT','PUBLISHED','RETIRED') NOT NULL DEFAULT 'DRAFT',
  created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
  UNIQUE KEY uq_store_product_public_id (public_id),
  INDEX idx_store_product_org_status (org_pk, status)
);

CREATE TABLE store_purchase (
  pk          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  buyer_org_pk BIGINT UNSIGNED NOT NULL,          -- 구매 org
  product_pk  BIGINT UNSIGNED NOT NULL,
  buyer_user_pk BIGINT UNSIGNED NOT NULL,
  ledger_pk   BIGINT UNSIGNED,
  purchased_at TIMESTAMP NOT NULL DEFAULT NOW(),
  expires_at  TIMESTAMP,
  INDEX idx_store_purchase_buyer_org (buyer_org_pk, purchased_at)
);
```

---

# M. DB 계정 최소 권한 (Least Privilege)

각 서비스는 자기 DB에만 접근하는 전용 계정을 사용한다.

| 계정 | 접근 DB | 권한 |
|---|---|---|
| `platform_rw` | `platform_db` | SELECT, INSERT, UPDATE (DELETE 금지) |
| `platform_ro` | `platform_db` | SELECT only (reporting, analytics) |
| `academy_rw` | `academy_db` | SELECT, INSERT, UPDATE |
| `academy_ro` | `academy_db` | SELECT only |
| `agent_rw` | `agent_db` | SELECT, INSERT, UPDATE |
| `market_rw` | `market_db` | SELECT, INSERT, UPDATE |
| `store_rw` | `store_db` | SELECT, INSERT, UPDATE |
| `audit_append` | `platform_db.audit_log` | INSERT only | 
| `migrator` | 전체 | DDL 권한 (migration 실행 전용, 상시 접속 금지) |

**원칙**:
- cross-DB 접근 계정 없음. academy-api는 `academy_rw` + `platform_ro`만 사용
- `DELETE` 권한: 어떤 계정에도 없음 (논리 삭제만 허용)
- `DROP`, `TRUNCATE`: `migrator` 계정에만, CI/CD 파이프라인 내에서만 실행
- 비밀번호: AWS Secrets Manager 관리, 90일 자동 로테이션

---

# N. 자체 비교 분석 — 무엇이 바뀌었나

자체 비교 분석 (R0–R7, 8라운드) 결과로 확정된 주요 설계 변경 사항.

## N.1 핵심 변경 비교표

| 항목 | 초기 설계 | 자체 비교 분석 결과 | 결정 |
|---|---|---|---|
| `organization.type` | ENUM(4종) 고정 | ENUM(3종, ACADEMY 제거). `org_kind VARCHAR+CHECK`는 향후 | 🟡 0008 부분 (D5 일부) |
| `membership.platform_role` | 도메인 역할 혼재 | platform_role 분리 + service_membership 병행 | ✅ **0008 완료** |
| `delegation_grant.capability` | 6종 고정 CHECK | `ACADEMY.<action>` 네임스페이스(6종 CHECK) | ✅ **0008 완료** |
| org 인가 판단 소스 | payment_ledger 직접 | org_entitlement SSOT | **확정 구현** |
| 감사 로그 동의 이력 | mutable boolean | append-only 이벤트 테이블 | **확정 설계** |
| 결제·권한 원자성 | 별개 트랜잭션 | 단일 트랜잭션 (§F.1) | **확정 구현** |
| 멀티테넌시 격리 | 앱 레이어 관례 | org_pk NOT NULL + CI 린트 P1 | 진행 중 |
| PG webhook 멱등 | 앱 레이어 처리 | UNIQUE(provider, event_id) + append | **확정 구현** |

## N.2 MySQL vs PostgreSQL 구조적 차이

자체 분석에서 도출된 MySQL 선택의 구조적 제약:

| 제약 | MySQL 우회법 | PostgreSQL 네이티브 |
|---|---|---|
| RLS 없음 | CI 린트 + org_pk 컬럼 강제 | `CREATE POLICY` |
| TIMESTAMP 파티셔닝 버그 | `DATETIME` 컬럼 사용 | TIMESTAMP 직접 사용 |
| JSON 인덱스 미흡 | JSON 쿼리 자제, 정규화 우선 | JSONB GIN 인덱스 |
| 감사 해시 내장 없음 | 앱 레이어 SHA-256 | `pgcrypto` |
| `FOR UPDATE SKIP LOCKED` | `SELECT ... FOR UPDATE` + 앱 retry | 네이티브 지원 |

현재 스택(MySQL + Drizzle)은 위 제약을 모두 앱/CI 레이어에서 보완 중.  
전환 비용보다 최적화 비용이 낮은 현 시점에서는 MySQL 유지를 선택한다.
