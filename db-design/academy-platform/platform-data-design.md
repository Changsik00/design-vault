# 플랫폼 데이터 설계 제안 — Identity / Org / Authorization / Billing

> 대상: NEXT Platform (academy / market / agent / future services 공통 기반)
> 전제: 단일 MySQL 8 서버 + 스키마(DB) 분리, NestJS, Drizzle, Firebase Auth(인증 전용)
> 원칙: **완성형 스키마를 한 번에 그리되, "지금 만들 것"과 "미룰 것"을 §11에서 명확히 분리한다.**

---

## 0. 설계 불변식 (이걸 어기면 나중에 터진다)

1. **Firebase = 인증, 인가 = 내 DB.** Firebase UID는 *조회 키*일 뿐 PK도 FK도 아니다.
2. **내부 PK는 BIGINT, 외부 노출은 ULID(`public_id`).** 시퀀셜 PK를 URL/API에 노출하지 않는다.
3. **role은 user에 없다. `membership`에 있다.** name/학년 등 프로필도 `identity_user`에 없다.
4. **모든 도메인 테이블은 `org_pk NOT NULL`.** 테넌트 경계는 컬럼이 아니라 *불변식*이다.
5. **`canXXX()`는 `org_entitlement`(런타임 권위 상태)만 읽는다.** billing ledger 직접 조회 금지.
6. **capability는 유한값이다(VARCHAR+CHECK).** 임의 JSON 권한 금지 → ABAC 카오스 방지. ENUM 회피 이유: ALTER 없이 CHECK constraint IN() 목록 추가가 더 안전.
7. **크로스DB는 아래로만(domain → identity/shared). 옆으로(peer product DB) 금지.**
8. **strong consistency가 필요한 상태는 단일 서버 트랜잭션으로. async는 outbox로.**

---

## 1. DB 토폴로지

```
MySQL Server 1대
├─ identity_db   ← SSOT. 누구인가 / 어디 소속인가 / 무엇을 위임받았나 / 런타임 access state
│    identity_user, user_profile, organization, org_relation, membership, membership_invite,
│    delegation_grant, org_entitlement, api_key, audit_log
│
├─ billing_db    ← 돈. 결제/구독/원장. (access state의 "writer", reader 아님)
│    product, product_feature, subscription, subscription_item,
│    payment_ledger, pg_webhook_event, outbox_event
│
├─ academy_db    ← 도메인. lecture/video/chunk… (전부 org_pk 보유, identity로 cross-db 참조)
├─ market_db     ← (future) 동일 패턴
└─ agent_db      ← (future) 동일 패턴
```

**왜 복제(identity_user 사본)·Kafka가 없나:** 같은 MySQL 인스턴스라 `academy_db.lecture JOIN identity_db.identity_user`가 네이티브로 되고, 트랜잭션도 한 커넥션에서 스키마를 넘나든다. 복제는 *서로 다른 서버*일 때만 필요한 부채다. 우리는 single-server이므로 SSOT를 직접 JOIN한다.

**크로스DB 방향 규칙:**
- `academy_db → identity_db` 읽기: OK (아래로).
- `billing_db ↔ identity_db.org_entitlement` 쓰기: OK (같은 트랜잭션, 아래 §5).
- `academy_db → market_db` 직접 JOIN: **금지** (옆으로). 필요하면 API/공유 read-model로.
- DB 유저 권한을 least-privilege로: `academy-api`의 MySQL 계정 = `academy_db.*` 전체 + `identity_db.*` 읽기. `market_db`는 접근 거부. → "옆으로 JOIN" 실수가 DB 레벨에서 차단됨.

**FK 전략:**
- **스키마 *내부*: 하드 FK 사용.** (정합성 + 인덱스 의도 문서화)
- **스키마 *간*: 하드 FK 안 검.** 앱 레벨 정합 + SSOT(identity) 상시 존재 전제. → 각 DB를 독립적으로 백업/복원 가능하게 유지.

---

## 2. identity_db

```sql
-- 사용자 (HUMAN / SERVICE / SYSTEM 공통)
CREATE TABLE identity_user (
  pk            BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,   -- 내부 FK/JOIN 전용
  public_id     CHAR(26)     NOT NULL UNIQUE,                 -- ULID, 외부 노출용
  firebase_uid  VARCHAR(128) NULL UNIQUE,                     -- 인증 연결키(FK 아님). SERVICE는 NULL
  email         VARCHAR(255) NULL,                            -- 앱 레벨 unique(ACTIVE 한정)
  phone_e164    VARCHAR(20)  NULL,
  type          ENUM('HUMAN','SERVICE','SYSTEM') NOT NULL DEFAULT 'HUMAN',
  status        ENUM('ACTIVE','SUSPENDED','DELETED') NOT NULL DEFAULT 'ACTIVE',
  perm_version  BIGINT UNSIGNED NOT NULL DEFAULT 1,           -- 프론트 권한 동기화용(§9)
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at    TIMESTAMP NULL,
  INDEX idx_email (email)
);
-- ★ name/birth/학년/닉네임 절대 여기 X → user_profile(공통) + 서비스별 *_profile 테이블
-- ★ email 전역 unique 강제 X: Firebase가 provider 단위로 이미 보장. 재가입/탈퇴 재사용 대비
--    "ACTIVE 중 unique"는 generated column + unique index로 강제 가능(MySQL 8).

-- 플랫폼 공통 프로필 (identity_user와 1:1, 함께 INSERT)
CREATE TABLE user_profile (
  user_pk      BIGINT UNSIGNED NOT NULL,
  display_name VARCHAR(120)  NOT NULL,
  avatar_url   VARCHAR(500)  NULL,
  locale       VARCHAR(10)   NOT NULL DEFAULT 'ko',
  timezone     VARCHAR(50)   NOT NULL DEFAULT 'Asia/Seoul',
  updated_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (user_pk),
  CONSTRAINT fk_up_user FOREIGN KEY (user_pk) REFERENCES identity_user(pk)
);
-- ★ display_name의 유일한 SSOT. 모든 서비스가 JOIN해서 읽는다.
-- ★ identity_user INSERT 트랜잭션 안에서 함께 INSERT (절대 분리하지 말 것).
-- ★ 서비스별 추가 프로필(강사 소개글, 학원명)은 각 도메인 DB의 *_profile 테이블에.

-- 조직(테넌트). academy/company/team…
CREATE TABLE organization (
  pk            BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id     CHAR(26)     NOT NULL UNIQUE,
  type          ENUM('ACADEMY','COMPANY','TEAM','PERSONAL') NOT NULL,
  slug          VARCHAR(50)  NOT NULL UNIQUE,
  name          VARCHAR(100) NOT NULL,
  status        ENUM('ACTIVE','SUSPENDED','CLOSED') NOT NULL DEFAULT 'ACTIVE',
  perm_version  BIGINT UNSIGNED NOT NULL DEFAULT 1,
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  deleted_at    TIMESTAMP NULL
);

-- 조직 계층 (본사→지점, 프랜차이즈 그룹→단위 학원)
-- ★ 스키마는 v0.1 부터 생성. 코드(계층 조회·정책 상속)는 P1에서 연결.
-- ★ 독립 학원: org_relation row 없음 (membership만으로 충분)
-- ★ 프랜차이즈: parent=그룹org, child=단위학원. 두 구조가 같은 DB에 공존.
CREATE TABLE org_relation (
  pk             BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  parent_org_pk  BIGINT UNSIGNED NOT NULL,
  child_org_pk   BIGINT UNSIGNED NOT NULL,
  relation_type  ENUM('HQ_BRANCH','HOLDING') NOT NULL DEFAULT 'HQ_BRANCH',
  -- HQ_BRANCH: 본사→지점 (프랜차이즈, 학원 체인)
  -- HOLDING:   지주→자회사 (법인 분리된 투자 구조, v2+)
  created_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_parent_child (parent_org_pk, child_org_pk),
  CONSTRAINT chk_no_self_ref CHECK (parent_org_pk != child_org_pk),  -- 자기 참조 방지
  CONSTRAINT fk_orgrel_parent FOREIGN KEY (parent_org_pk) REFERENCES organization(pk),
  CONSTRAINT fk_orgrel_child  FOREIGN KEY (child_org_pk)  REFERENCES organization(pk)
);
-- ★ 순환 참조(A→B→A) 방지는 앱 레벨 INSERT 시 getAncestors() 탐지로 추가.
-- ★ MVP는 depth 1 (그룹→단위학원)만. depth 2+ (지역본부→지점)는 P2.
-- ★ 스키마 P0 생성 / 계층 탐색 코드 P1 연결 — db-design-decisions.md §11 참조.
-- 프랜차이즈 예시:
--   organization(pk=1, type='COMPANY', name='한울학원 그룹')  ← 부모
--   organization(pk=2, type='ACADEMY', name='한울학원 강남')  ← 자식
--   org_relation(parent=1, child=2, relation_type='HQ_BRANCH')
--   membership(user_pk=오너, org_pk=1, role='OWNER')  ← 그룹 수준 권한
--   membership(user_pk=오너, org_pk=2, role='DIRECTOR') ← 지점 직접 접근 (명시적)
-- 독립 학원 예시:
--   organization(pk=10, type='ACADEMY', name='별빛학원')  ← org_relation 없음
--   membership(user_pk=원장, org_pk=10, role='DIRECTOR')

-- 소속 + role (RBAC 베이스). user × org = role
CREATE TABLE membership (
  user_pk     BIGINT UNSIGNED NOT NULL,
  org_pk      BIGINT UNSIGNED NOT NULL,
  role        ENUM('OWNER','DIRECTOR','TEACHER','MEMBER','STUDENT','PARENT') NOT NULL,
  status      ENUM('ACTIVE','SUSPENDED') NOT NULL DEFAULT 'ACTIVE',  -- INVITED 없음
  created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_pk, org_pk),
  INDEX idx_org_role (org_pk, role),
  CONSTRAINT fk_mbr_user FOREIGN KEY (user_pk) REFERENCES identity_user(pk),
  CONSTRAINT fk_mbr_org  FOREIGN KEY (org_pk)  REFERENCES organization(pk)
);
-- ★ INVITED 상태 없음: 초대는 membership_invite만으로 관리. 수락 시에만 membership INSERT.
-- ★ service(academy/market)는 여기 안 둔다. org.type + org_entitlement로 결정.
--   한 org가 여러 service를 쓰고 service별 role이 *달라야* 하면 그때
--   membership_service(user_pk, org_pk, service, role)를 추가. MVP엔 불필요.

-- 초대(아직 계정 없는 사람 포함). "팀원 요청 페이지"
CREATE TABLE membership_invite (
  pk          BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  org_pk      BIGINT UNSIGNED NOT NULL,
  email       VARCHAR(255) NOT NULL,
  role        ENUM('OWNER','DIRECTOR','TEACHER','MEMBER','STUDENT','PARENT') NOT NULL,
  token       CHAR(43)     NOT NULL UNIQUE,                   -- 초대 링크 토큰
  status      ENUM('PENDING','ACCEPTED','EXPIRED','REVOKED') NOT NULL DEFAULT 'PENDING',
  invited_by  BIGINT UNSIGNED NOT NULL,
  expires_at  TIMESTAMP NOT NULL,
  created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_org_status (org_pk, status),
  INDEX idx_email (email)
);
-- 가입 완료 시 membership(status=ACTIVE)로 승격.

-- 위임/대행 (ReBAC). "원장이 강사에게 업로드 권한 부여"
CREATE TABLE delegation_grant (
  pk          BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  org_pk      BIGINT UNSIGNED NOT NULL,
  grantor_pk  BIGINT UNSIGNED NOT NULL,                       -- 준 사람(원장)
  grantee_pk  BIGINT UNSIGNED NOT NULL,                       -- 받은 사람(강사)
  capability  VARCHAR(50) NOT NULL,                                  -- 'PUBLISH_VIDEO','APPROVE_VIDEO' 등
                                                                     -- CHECK constraint로 유한값 강제
  CONSTRAINT chk_capability CHECK (capability IN (
    'PUBLISH_VIDEO','APPROVE_VIDEO','VIEW_ALL_LECTURES',
    'MANAGE_SCHEDULE','MANAGE_MEMBERS','VIEW_BILLING'
  )),
  scope_json  JSON NULL,                                      -- 선택 제한 {"subject":"MATH"} 등
  status      ENUM('ACTIVE','REVOKED') NOT NULL DEFAULT 'ACTIVE',
  expires_at  TIMESTAMP NULL,
  created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_grantee (org_pk, grantee_pk, status),
  CONSTRAINT fk_grant_org     FOREIGN KEY (org_pk)     REFERENCES organization(pk),
  CONSTRAINT fk_grant_grantor FOREIGN KEY (grantor_pk) REFERENCES identity_user(pk),
  CONSTRAINT fk_grant_grantee FOREIGN KEY (grantee_pk) REFERENCES identity_user(pk)
);
-- ★ "DIRECTOR로 등록"의 대안. 임퍼소네이션(신분 변신) 쓰지 말 것 — 감사 추적이 흐려진다.
--   항상 "강사 본인이 위임받은 capability를 행사"로 로그가 남아야 함.
-- ★ capability = VARCHAR(50) + CHECK constraint. ENUM이 아닌 이유: MySQL ENUM 변경은 ALTER TABLE
--   (InnoDB copy + 락). CHECK constraint는 INFORMATION_SCHEMA 수정만으로 무중단 추가 가능.
--   새 capability 추가 = CHECK constraint IN() 목록에 값 추가. 임의 JSON 속성 금지는 동일.

-- 런타임 권위 access 상태. ★ canXXX()의 유일한 진실원천 ★
CREATE TABLE org_entitlement (
  pk             BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  org_pk         BIGINT UNSIGNED NOT NULL,
  product_code   VARCHAR(50) NOT NULL,                        -- billing_db.product.code 미러
  service        ENUM('ACADEMY','MARKET','AGENT') NOT NULL,
  status         ENUM('ACTIVE','GRACE','SUSPENDED','EXPIRED') NOT NULL DEFAULT 'ACTIVE',
  source         ENUM('SUBSCRIPTION','PROMO','MANUAL','FREE') NOT NULL,
  feature_limits JSON NULL,                                   -- {"max_teachers":10,"daily_uploads":6}
  valid_until    TIMESTAMP NULL,
  grace_until    TIMESTAMP NULL,
  updated_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_org_product (org_pk, product_code),
  INDEX idx_org_service_status (org_pk, service, status),
  CONSTRAINT fk_ent_org FOREIGN KEY (org_pk) REFERENCES organization(pk)
);
-- ★ payment_ledger 직접 조회 금지. billing은 이 테이블의 writer만 바꾼다(§5).
-- ★ source=MANUAL/PROMO/FREE → "결제 없이 access 부여" 자연 지원.
-- ★ 만료 즉시 차단 = status를 EXPIRED/SUSPENDED로 (캐시 무효화 + perm_version bump).

-- 머신 신분/B2B (§10) — DEFER
CREATE TABLE api_key (
  pk               BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  org_pk           BIGINT UNSIGNED NOT NULL,
  principal_pk     BIGINT UNSIGNED NOT NULL,                  -- identity_user.pk (type=SERVICE 권장)
  key_prefix       CHAR(8)  NOT NULL,                         -- 식별/로그용(비밀 아님)
  secret_hash      CHAR(64) NOT NULL,                         -- sha256(secret). 평문 저장 금지
  scopes           JSON     NOT NULL,                         -- ["read","write","admin","webhook","automation"]
  allowed_services JSON     NOT NULL,                         -- ["ACADEMY","MARKET"]
  status           ENUM('ACTIVE','REVOKED') NOT NULL DEFAULT 'ACTIVE',
  expires_at       TIMESTAMP NULL,
  last_used_at     TIMESTAMP NULL,
  created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_org (org_pk, status),
  INDEX idx_prefix (key_prefix)
);
-- 다중 키 = rotation. 회수 = status REVOKED(즉시 반영). 키별 scope/service 제한.

-- 감사 로그 (append-only, 월별 파티셔닝, 3년 보존)
CREATE TABLE audit_log (
  pk            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_pk        BIGINT UNSIGNED NULL,
  actor_type    ENUM('HUMAN','API_KEY','SYSTEM') NOT NULL,
  actor_pk      BIGINT UNSIGNED NULL,                         -- identity_user.pk
  api_key_pk    BIGINT UNSIGNED NULL,                         -- 머신 호출 시
  action        VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50)  NULL,
  resource_pk   BIGINT UNSIGNED NULL,
  result        ENUM('ALLOW','DENY','ERROR') NOT NULL,
  ip            VARBINARY(16) NULL,
  meta_json     JSON NULL,
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (pk, created_at),                               -- 파티션 키를 PK에 포함 (MySQL 필수)
  INDEX idx_org_created (org_pk, created_at),
  INDEX idx_actor (actor_pk, created_at)
) PARTITION BY RANGE COLUMNS(created_at) (
  PARTITION p202601 VALUES LESS THAN ('2026-02-01 00:00:00'),
  PARTITION p202602 VALUES LESS THAN ('2026-03-01 00:00:00'),
  PARTITION p202603 VALUES LESS THAN ('2026-04-01 00:00:00'),
  PARTITION p202604 VALUES LESS THAN ('2026-05-01 00:00:00'),
  PARTITION p202605 VALUES LESS THAN ('2026-06-01 00:00:00'),
  PARTITION p202606 VALUES LESS THAN ('2026-07-01 00:00:00'),
  PARTITION pFuture VALUES LESS THAN (MAXVALUE)
);
-- ★ append-only. 앱 DB 계정에서 UPDATE/DELETE 권한 제거.
-- ★ 3년 보존: 월 1회 배치로 36개월 초과 파티션 ALTER TABLE ... DROP PARTITION
-- ★ created_at을 PK에 포함하는 이유: MySQL RANGE 파티셔닝은 파티션 키가 PK의 일부여야 함
```

---

## 3. billing_db

```sql
CREATE TABLE product (
  pk      BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  code    VARCHAR(50) NOT NULL UNIQUE,                        -- 'ACADEMY_PRO','YOUTUBE_AUTOMATION'
  service ENUM('ACADEMY','MARKET','AGENT') NOT NULL,
  name    VARCHAR(100) NOT NULL,
  status  ENUM('ACTIVE','RETIRED') NOT NULL DEFAULT 'ACTIVE'
);

CREATE TABLE product_feature (
  pk          BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  product_pk  BIGINT UNSIGNED NOT NULL,
  feature_key VARCHAR(50) NOT NULL,                           -- 'max_teachers','daily_uploads','rag_chat'
  limit_value BIGINT NULL,                                    -- NULL=무제한, boolean은 0/1
  UNIQUE KEY uk_product_feature (product_pk, feature_key),
  CONSTRAINT fk_pf_product FOREIGN KEY (product_pk) REFERENCES product(pk)
);

CREATE TABLE subscription (
  pk                 BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  org_pk             BIGINT UNSIGNED NOT NULL,                -- cross-db → identity_db.organization
  payer_user_pk      BIGINT UNSIGNED NOT NULL,
  status             ENUM('TRIALING','ACTIVE','PAST_DUE','CANCELED','EXPIRED') NOT NULL,
  pg_provider        ENUM('TOSS','STRIPE','PAYPAL','MANUAL') NOT NULL,
  external_sub_id    VARCHAR(255) NULL,
  current_period_end TIMESTAMP NULL,
  grace_until        TIMESTAMP NULL,
  created_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_org (org_pk),
  INDEX idx_status_period (status, current_period_end)
);

-- 하나의 subscription이 여러 product 포함
CREATE TABLE subscription_item (
  pk              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  subscription_pk BIGINT UNSIGNED NOT NULL,
  product_pk      BIGINT UNSIGNED NOT NULL,
  quantity        INT NOT NULL DEFAULT 1,                     -- 좌석 수 등
  status          ENUM('ACTIVE','CANCELED') NOT NULL DEFAULT 'ACTIVE',
  UNIQUE KEY uk_sub_product (subscription_pk, product_pk),
  CONSTRAINT fk_si_sub     FOREIGN KEY (subscription_pk) REFERENCES subscription(pk),
  CONSTRAINT fk_si_product FOREIGN KEY (product_pk)      REFERENCES product(pk)
);

-- 돈 원장 (append-only)
CREATE TABLE payment_ledger (
  pk              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  org_pk          BIGINT UNSIGNED NOT NULL,
  subscription_pk BIGINT UNSIGNED NULL,
  type            ENUM('CHARGE','REFUND','CHARGEBACK','CREDIT') NOT NULL,
  amount_minor    BIGINT NOT NULL,                            -- 최소단위(원/센트) 정수 저장
  currency        CHAR(3) NOT NULL,
  pg_provider     ENUM('TOSS','STRIPE','PAYPAL','MANUAL') NOT NULL,
  pg_payment_id   VARCHAR(255) NULL,
  idempotency_key VARCHAR(255) NOT NULL UNIQUE,               -- 중복 결제/재시도 차단
  status          ENUM('PENDING','SUCCEEDED','FAILED') NOT NULL,
  receipt_url     VARCHAR(512) NULL,
  created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_org_created (org_pk, created_at)
);
-- ★ 환불도 새 row(REFUND). 기존 row UPDATE로 금액 수정 금지. float 금지 → 정수 minor unit.

-- PG webhook 멱등
CREATE TABLE pg_webhook_event (
  pk           BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  pg_provider  ENUM('TOSS','STRIPE','PAYPAL') NOT NULL,
  event_id     VARCHAR(255) NOT NULL,                         -- PG 고유 이벤트 ID
  signature_ok TINYINT(1) NOT NULL DEFAULT 0,
  payload_json JSON NOT NULL,
  status       ENUM('RECEIVED','PROCESSED','SKIPPED','FAILED') NOT NULL DEFAULT 'RECEIVED',
  processed_at TIMESTAMP NULL,
  created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_provider_event (pg_provider, event_id)        -- 중복 webhook 멱등
);

-- Transactional Outbox (async 발행원). MQ는 이 위에 나중에 얹음
CREATE TABLE outbox_event (
  pk             BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  aggregate_type VARCHAR(50) NOT NULL,
  aggregate_pk   BIGINT UNSIGNED NOT NULL,
  event_type     VARCHAR(80) NOT NULL,                        -- 'subscription.activated'
  payload_json   JSON NOT NULL,
  status         ENUM('PENDING','SENT','FAILED') NOT NULL DEFAULT 'PENDING',
  created_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  sent_at        TIMESTAMP NULL,
  INDEX idx_status_created (status, created_at)
);
```

> **PG Adapter**: Toss/Stripe/PayPal는 코드 레벨 어댑터 인터페이스로 추상화. DB는 `pg_provider` enum과 `external_*` id만 알면 됨. PG별 테이블 분리 불필요.

---

## 4. 도메인 DB (academy_db) 패턴

```sql
-- 모든 도메인 테이블의 표준 형태 (lecture 예시)
CREATE TABLE lecture (
  pk         BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id  CHAR(26) NOT NULL UNIQUE,
  org_pk     BIGINT UNSIGNED NOT NULL,                        -- 테넌트 경계 (= academy)
  owner_pk   BIGINT UNSIGNED NOT NULL,                        -- cross-db → identity_db.identity_user
  title      VARCHAR(200) NULL,
  status     ENUM('QUEUED','PROCESSING','PENDING_REVIEW','PUBLISHED','FAILED') NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP NULL,
  INDEX idx_org (org_pk),
  INDEX idx_org_owner (org_pk, owner_pk)
);
-- ★ org_pk NOT NULL은 예외 없음. cross-db FK(owner_pk→identity)는 하드로 안 검(백업 독립성).
-- ★ Qdrant payload / Neo4j property에도 동일하게 org_pk 강제 — DB뿐 아니라 RAG 경계까지.
--   (mvp-scope DoD "멀티테넌트 격리 100% E2E"의 진짜 사고 표면이 여기다.)
```

Repo 레벨 강제 (Drizzle): 도메인 조회는 `org_pk` 없는 `findOne(pk)`를 만들지 말 것. `findOne(pk, orgPk)` 시그니처를 강제해서 누락을 타입/런타임에서 차단.

---

## 5. 핵심 — billing → access 갱신은 "단일 서버 트랜잭션"

```
[구독 활성화 / 갱신 / 만료 / 환불 발생]
  BEGIN;                                            -- 한 커넥션, 두 스키마
    INSERT billing_db.payment_ledger (...);         -- 돈 기록
    UPDATE billing_db.subscription   (...);         -- 구독 상태
    -- ★ 같은 트랜잭션에서 런타임 access 상태 갱신 ★
    INSERT ... ON DUPLICATE KEY UPDATE
      identity_db.org_entitlement (status, feature_limits, valid_until, ...);
    UPDATE identity_db.organization SET perm_version = perm_version + 1;  -- 프론트 동기화
    INSERT billing_db.outbox_event (event_type='subscription.activated', ...);
  COMMIT;
```

- 단일 InnoDB 인스턴스라 **크로스 스키마 트랜잭션이 그냥 ACID**. 2PC도 Kafka도 필요 없음.
- 따라서 "결제됐는데 권한 반영 안 됨" 같은 eventual consistency 창이 **없다** → 요구 #6/#10의 *strong consistency*를 공짜로 만족.
- async 부수효과(알림/분석/영수증 메일)는 outbox_event 워커가 따로 처리(§8). 이쪽만 eventual.

---

## 6. canXXX() 평가 모델 (3-gate, Human/Machine 공통)

```
canXXX(principal, org_pk, service, action, resource):

 0) principal resolve
    - HUMAN  : Firebase token → identity_user.pk
    - MACHINE: api_key → principal_pk + allowed_services + scopes

 1) [Gate A — 소속] membership(principal, org).status == ACTIVE
    - MACHINE: api_key.org_pk == org_pk

 2) [Gate B — 결제/이용권] org_entitlement(org, service).status IN ('ACTIVE','GRACE')
    - feature_limit 체크: 예) action='upload' → daily_uploads 한도 vs 현재 사용량
    - MACHINE: service ∈ api_key.allowed_services

 3) [Gate C — 정책] (단일 평가기 = CASL)
    - RBAC : action ∈ ROLE_PERMISSION[service][membership.role] ?
    - ReBAC: delegation_grant(grantee=principal, capability↦action, ACTIVE, !expired) 존재?
    - ABAC : ownership → resource.owner_pk == principal.pk ?
    - MACHINE: api_key.scopes ⊇ action이 요구하는 scope(read/write/admin)

 → A AND B AND C 모두 통과해야 ALLOW. 결과를 audit_log에 기록.
```

**정책 평가기는 하나(CASL)로 단일화.** membership/grant/entitlement는 *CASL ability를 만드는 입력*이지, 별도의 `evalCondition` JSON 인터프리터를 또 만들지 않는다(엔진 이중화 금지).

**캐싱:** Redis에 *입력 블록*(membership / grants / entitlement)만 TTL 60s + 쓰기 시 무효화. **최종 `can()` 결정은 캐싱하지 않는다** — `owner_pk == $user.pk` 같은 조건은 리소스별로 답이 달라서 (user, action) 단위 boolean 캐싱이 오답을 캐싱한다.

---

## 7. 정책 어휘 (코드 상수, DB 아님)

```ts
// 서비스 × role → 기본 action. CASL rule을 DB에 저장하지 않는다(디버깅 지옥 방지).
const ROLE_PERMISSION = {
  ACADEMY: {
    OWNER:    ['*'],
    DIRECTOR: ['lecture.*', 'video.approve', 'video.publish', 'member.manage', 'billing.view'],
    TEACHER:  ['lecture.create', 'lecture.view_own', 'video.request_publish'],
    STUDENT:  ['lecture.view_enrolled', 'rag.query'],
    PARENT:   ['notification.receive'],
  },
  MARKET: { /* ... */ },
} as const;

// capability ↦ action 매핑(위임 평가용)
const CAPABILITY_ACTION = {
  PUBLISH_VIDEO:      ['video.publish'],
  APPROVE_VIDEO:      ['video.approve'],
  VIEW_ALL_LECTURES:  ['lecture.view_all'],
  MANAGE_SCHEDULE:    ['schedule.*'],
  MANAGE_MEMBERS:     ['member.manage'],
} as const;
```

---

## 8. Sync vs Async 경계

| 경로 | 일관성 | 구현 |
|---|---|---|
| 결제 → org_entitlement(런타임 access) | **Strong** | 단일 서버 트랜잭션(§5). MQ 미사용 |
| 권한/구독 변경 → 프론트 | near-real-time | perm_version bump + 403 refresh(§9), 후에 SSE |
| 알림/분석/영수증/검색 인덱싱 | Eventual | outbox_event → 워커 polling → 핸들러(후에 MQ) |

→ **Runtime Access State만 strong, 나머지는 outbox로 async.** MQ(Kafka 등)는 outbox 위에 *나중에* 얹는 선택이지 지금 의존성이 아니다.

---

## 9. 프론트 권한 동기화

- 진입 시 `GET /me/permissions` → snapshot `{ perm_version, abilities[], entitlements[] }` (CASL rule을 그대로 직렬화).
- 클라가 `perm_version` 보관. 모든 API 응답 헤더에 `X-Perm-Version` 동봉 → 불일치 시 snapshot 재요청(self-healing).
- **403 수신 → 강제 refresh.** stale 권한 자동 치유.
- 실시간 push가 필요해지면 SSE로 "perm_version bumped" 통지만 보냄(페이로드 X). **MVP는 403 + version polling으로 충분.** websocket은 과설계.

---

## 10. B2B / Server-to-Server (요구 #13)

- **머신 신분 = `identity_user(type='SERVICE')`** + org 소속(membership 또는 api_key.org_pk). Human과 같은 테이블·같은 게이트를 타게 해서 canXXX가 양쪽을 동일하게 처리.
- **`api_key`**: prefix(식별) + secret_hash(검증), `scopes`(read/write/admin/webhook/automation), `allowed_services`(서비스 제한), `expires_at`, 다중 키(rotation), `status=REVOKED`(즉시 회수).
- **rate limit / quota**: 카운터는 Redis, 정책 한도는 `org_entitlement.feature_limits` 또는 별도 `api_quota_policy`. product별 quota는 entitlement에서 끌어옴.
- **webhook(우리가 제공)**: signature 서명 + 수신측 검증. 우리가 받는 PG webhook은 `pg_webhook_event`로 멱등.
- **외부 호출 감사**: `audit_log.actor_type='API_KEY'` + `api_key_pk` → "어떤 키가 무엇을 했나" 추적.
- **OAuth Client Credentials**: api_key 모델 위에 표준화. Internal API vs External Public API는 라우팅/게이트웨이 분리(여기서도 X-User-Pk 같은 헤더는 게이트웨이가 inbound strip + 내부망/mTLS 전제 없으면 신뢰 금지).

---

## 11. ★ Phasing — 지금 만들 것 vs 미룰 것 ★

> 요구 목록 전체는 **완성형 비전**이다. academy 8주 MVP에 이걸 다 만들면 침몰한다.
> 아래 P0만 만들고, 나머지는 *스키마는 비워두되 코드로 채우지 않는다*.

| 영역 | 테이블/기능 | Phase | 비고 |
|---|---|---|---|
| Identity | identity_user, user_profile, organization, membership, membership_invite | **P0 (v0.1)** | user_profile은 identity_user와 함께 생성 |
| 위임 | delegation_grant | **P0 (v0.1)** | trust_relationship가 MVP 스코프 |
| Access | org_entitlement | **P0 (v0.1)** | **source=MANUAL/FREE로 수동 채움.** 게이트 코드만 완성 |
| 정책 | 3-gate canXXX + ROLE_PERMISSION(code) | **P0 (v0.1)** | RBAC + ownership + grant까지. 일반 ABAC 엔진은 나중 |
| 감사 | audit_log | **P0 (v0.1)** | append-only, 처음부터 |
| 프론트 | perm_version + snapshot + 403 refresh | **P0 (v0.1)** | SSE/websocket 제외 |
| 도메인 | academy_db.* (org_pk 강제) | **P0 (v0.1)** | RAG(Qdrant/Neo4j) org_pk 격리 포함 |
| Billing | product, subscription, payment_ledger, pg_webhook_event, PG adapter | **P1 (v1.0~)** | 결제 붙일 때. org_entitlement *writer*만 교체 |
| Async | outbox_event + 워커 (+ MQ) | **P1~P2** | 부수효과가 쌓일 때 |
| Org 계층 | org_relation (본사-지점) | **스키마 P0 / 코드 P1** | 독립 학원 + 프랜차이즈 동시 지원. 스키마 생성은 지금, 계층 조회·정책 상속 코드는 P1 |
| B2B | api_key, service_account, quota, OAuth CC | **P2~P3 (v2.0+)** | 외부 API 제품화 시 |
| 일반 ReBAC | 범용 relation_tuple | **필요 시** | delegation_grant로 부족해질 때만. 미리 만들지 말 것 |
| 실시간 | SSE/websocket 권한 push | **필요 시** | 동시 편집 충돌이 실제로 생길 때 |

**P0 DB는 `identity_db` + `academy_db`만 실제 생성.** `billing_db`는 스키마만 정의하고 비워둔다(파일럿 학원은 `org_entitlement`를 수동 INSERT). 결제가 들어오는 날, 바뀌는 건 entitlement를 *쓰는* 쪽뿐이고 *읽는* canXXX는 그대로다.

---

## 12. 안티패턴 (재확인)

| 하지 마 | 이유 |
|---|---|
| identity_user 복제 + Kafka sync | single-server에선 부채. 그냥 cross-db JOIN |
| canXXX가 payment_ledger 조회 | 돈과 권한을 묶으면 환불/grace/프로모가 다 권한 버그로 |
| capability를 임의 JSON으로 | ABAC 카오스. 유한 VARCHAR(50)+CHECK로 |
| role/name을 identity_user에 | 멀티테넌트/멀티프로덕트 불가, "A학원 이름 바꾸니 B도 바뀜" |
| CASL rule을 DB에 저장 | 비즈니스 로직이 데이터가 되면 디버깅 지옥 |
| 최종 can() 결정 캐싱 | 리소스별 ABAC 조건 오답 캐싱 |
| peer product DB 직접 JOIN | 배포 독립성 파괴(옆으로 금지) |
| 머신용 별도 권한 경로 | api_key도 같은 3-gate를 타야 일관성 유지 |
| 금액 float 저장 | 반올림 오차. 정수 minor unit |
| 임퍼소네이션으로 위임 구현 | 감사 추적/책임 소재 소실. scoped grant로 |

---

## 13. 한 줄 요약

> **identity_db가 "누구/어디소속/무엇위임"의 SSOT, `user_profile`이 display_name 단일 진실원천, `org_entitlement`가 런타임 권위 access 상태, `canXXX`는 membership·entitlement·policy 3-gate.** 결제는 단일 서버 트랜잭션으로 entitlement를 *동기* 갱신(2PC·Kafka 불필요)하고, 나머지는 outbox로 *async*. capability는 VARCHAR(50)+CHECK로 유한 위임 표현, ABAC 폭발 방지. **`org_relation`은 스키마 P0 생성 — 독립 학원(row 없음)과 프랜차이즈(HQ_BRANCH row)가 같은 DB에서 공존.** P0는 identity_db + academy_db만, billing/B2B/실시간은 트리거 도달 시 승격.
