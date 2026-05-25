# 데이터 모델 (v0.1 MVP)

> 작성일: 2026-05-23
> 갱신: 2026-05-25 (장기 운영 재검토 — `user_profile`, `student`, `usage_log` 추가, `membership.status` 정리, 파티셔닝 반영)
>
> **변경 이력**:
> - 2026-05-23: `user` → `identity_user` + `membership` 분리.
> - 2026-05-25 (v1): `organization` SSOT 채택(Option A), `academy_pk` → `org_pk`, `academy_config` 분리.
> - 2026-05-25 (v2): 장기 운영 관점 재검토. `user_profile`(display_name SSOT), `student`(알림 대상 + PIPA), `usage_log`(비용 cap) 신규. `membership.status` INVITED 제거. `audit_log`/`usage_log` 파티셔닝. `public_id` 범위 확정. 상세 결정 근거는 [`db-design-decisions.md`](db-design-decisions.md).

---

## 구조 개요

```
identity_db (SSOT — platform-data-design.md 정의)
  identity_user, user_profile,
  organization, membership, membership_invite,
  delegation_grant, org_entitlement, audit_log (partitioned)

academy_db (도메인 — 본 문서 정의)
  academy_config, youtube_channel,
  lecture, lecture_chunk, video_asset, youtube_video,
  student, notification, trust_relationship,
  usage_log (partitioned)
```

- identity_db 테이블은 본 문서에서 **참조**만 한다. 스키마 정의는 [`platform-data-design.md`](platform-data-design.md) §2.
- 모든 academy_db 테이블은 `org_pk NOT NULL`을 보유한다 (테넌트 경계 불변식).
- cross-db FK(academy_db → identity_db)는 하드로 걸지 않는다. 앱 레벨 정합 + identity_db 상시 존재 전제.

---

## 1. identity_db 참조 스키마

> 아래는 academy-api 코드에서 JOIN/참조할 때 컬럼을 명시하기 위한 요약이다. 원본은 [`platform-data-design.md`](platform-data-design.md) §2.

### 1.1 identity_user

```sql
-- platform-data-design.md §2 정의 (요약)
CREATE TABLE identity_user (
  pk            BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,  -- 내부 FK/JOIN 전용
  public_id     CHAR(26)     NOT NULL UNIQUE,                -- ULID, 외부 노출용
  firebase_uid  VARCHAR(128) NULL UNIQUE,                    -- SERVICE 계정은 NULL
  email         VARCHAR(255) NULL,
  phone_e164    VARCHAR(20)  NULL,
  type          ENUM('HUMAN','SERVICE','SYSTEM') NOT NULL DEFAULT 'HUMAN',
  status        ENUM('ACTIVE','SUSPENDED','DELETED') NOT NULL DEFAULT 'ACTIVE',
  perm_version  BIGINT UNSIGNED NOT NULL DEFAULT 1,
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at    TIMESTAMP NULL
);
-- ★ display_name/학년/학원 이름 등 프로필은 여기 없음 → academy_db.*_profile 테이블
-- ★ user_type='agent'(구) → type='SERVICE'(신). 하위 호환 불필요(production 미진입)
```

- `type='HUMAN'`: 일반 사람 사용자
- `type='SERVICE'`: AI agent / 자동화 봇 / machine-to-machine 계정
- `type='SYSTEM'`: 시스템 내부 actor (감사 로그 출처 표시용)

### 1.1-B user_profile (플랫폼 공통 프로필 — identity_db)

```sql
-- identity_db (platform-data-design.md §2 신규 추가)
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
-- ★ identity_user와 1:1. identity_user INSERT 시 즉시 함께 INSERT.
-- ★ display_name이 유일한 SSOT. 모든 서비스가 여기서 읽는다.
-- ★ 서비스별 추가 프로필(강사 소개글, 판매자 상호명)은 서비스 DB의 *_profile 테이블에.
```

### 1.2 organization

```sql
-- platform-data-design.md §2 정의 (요약)
CREATE TABLE organization (
  pk         BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id  CHAR(26)    NOT NULL UNIQUE,                    -- ULID
  type       ENUM('ACADEMY','COMPANY','TEAM','PERSONAL') NOT NULL,
  slug       VARCHAR(50) NOT NULL UNIQUE,
  name       VARCHAR(100) NOT NULL,
  status     ENUM('ACTIVE','SUSPENDED','CLOSED') NOT NULL DEFAULT 'ACTIVE',
  perm_version BIGINT UNSIGNED NOT NULL DEFAULT 1,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP NULL
);
-- ★ 기존 academy 테이블의 테넌트 루트 역할을 이어받음.
-- ★ type='ACADEMY'인 organization.pk가 academy_db 전체의 org_pk 기준.
```

### 1.3 membership

```sql
-- platform-data-design.md §2 정의 (요약)
CREATE TABLE membership (
  user_pk    BIGINT UNSIGNED NOT NULL,
  org_pk     BIGINT UNSIGNED NOT NULL,
  role       ENUM('OWNER','DIRECTOR','TEACHER','MEMBER','STUDENT','PARENT') NOT NULL,
  status     ENUM('ACTIVE','SUSPENDED') NOT NULL DEFAULT 'ACTIVE',  -- INVITED 없음
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_pk, org_pk),
  INDEX idx_org_role (org_pk, role)
);
-- ★ 기존 (user_pk, product, workspace_pk, role) → (user_pk, org_pk).
-- ★ service 구분은 org.type + org_entitlement로. membership에 product 컬럼 없음.
-- ★ 기존 actions JSON → delegation_grant 테이블로 분리.
-- ★ INVITED 상태 없음: 초대는 membership_invite만으로 관리. 수락 시에만 membership INSERT.
```

**academy product 사용 예:**
```
(user_pk=42, org_pk=1, role='DIRECTOR')  -- 한울학원 원장
(user_pk=43, org_pk=1, role='TEACHER')   -- 한울학원 강사
(user_pk=43, org_pk=2, role='TEACHER')   -- 다른 학원에서도 강사
```

### 1.4 org_entitlement

```sql
-- platform-data-design.md §2 정의 (요약)
CREATE TABLE org_entitlement (
  pk             BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  org_pk         BIGINT UNSIGNED NOT NULL,
  product_code   VARCHAR(50) NOT NULL,
  service        ENUM('ACADEMY','MARKET','AGENT') NOT NULL,
  status         ENUM('ACTIVE','GRACE','SUSPENDED','EXPIRED') NOT NULL DEFAULT 'ACTIVE',
  source         ENUM('SUBSCRIPTION','PROMO','MANUAL','FREE') NOT NULL,
  feature_limits JSON NULL,
  -- 예: {"max_teachers":10,"daily_uploads":6,"rag_enabled":true,"youtube_enabled":true}
  valid_until    TIMESTAMP NULL,
  grace_until    TIMESTAMP NULL,
  updated_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_org_product (org_pk, product_code)
);
-- ★ P0: source=MANUAL/FREE로 수동 INSERT. 기존 academy.plan/rag_enabled/daily_video_limit 역할 흡수.
-- ★ canXXX() Gate B의 유일한 진실원천. payment_ledger 직접 조회 금지.
```

**기존 `academy` 컬럼 → `org_entitlement.feature_limits` 매핑:**

| 기존 `academy` 컬럼 | 신규 위치 |
|---|---|
| `plan ENUM('basic','pro','enterprise')` | `org_entitlement.product_code` |
| `rag_enabled BOOLEAN` | `feature_limits.rag_enabled` |
| `youtube_enabled BOOLEAN` | `feature_limits.youtube_enabled` |
| `daily_video_limit INT` | `feature_limits.daily_uploads` |

---

## 2. academy_db 테이블

### 2.1 academy_config (학원 도메인 설정)

> 기존 `academy` 테이블에서 plan/limit 컬럼은 `org_entitlement`로 이동. 순수 academy 도메인 설정만 남김.

```sql
CREATE TABLE academy_config (
  pk                  BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  org_pk              BIGINT UNSIGNED NOT NULL UNIQUE,              -- cross-db → identity_db.organization
  youtube_auto_mode   ENUM('auto','review_required','per_teacher') NOT NULL DEFAULT 'review_required',
  default_visibility  ENUM('public','unlisted','private') NOT NULL DEFAULT 'unlisted',
  evaluation_weights  JSON NULL,                                    -- v1.0 강사 평가 가중치
  created_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at          TIMESTAMP NULL,
  INDEX idx_org (org_pk)
);
-- ★ 하드 FK(→ identity_db.organization) 없음. 앱 레벨 정합.
-- ★ org_pk UNIQUE: organization 1개 = academy_config 1개.
```

### 2.2 youtube_channel (학원당 1개)

```sql
CREATE TABLE youtube_channel (
  pk                       BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  org_pk                   BIGINT UNSIGNED NOT NULL,
  youtube_channel_id       VARCHAR(64) NOT NULL,                   -- YouTube UCxxxx ID
  channel_name             VARCHAR(200),
  oauth_refresh_token_kms  VARCHAR(500) NOT NULL,                  -- KMS envelope encrypted
  oauth_scope              VARCHAR(500) NOT NULL,
  last_synced_at           TIMESTAMP NULL,
  created_at               TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at               TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at               TIMESTAMP NULL,
  UNIQUE KEY uk_channel_org (org_pk)
);
```

### 2.3 lecture (강의 1건)

```sql
CREATE TABLE lecture (
  pk               BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id        CHAR(26)     NOT NULL UNIQUE,                   -- ULID
  org_pk           BIGINT UNSIGNED NOT NULL,                       -- 테넌트 경계
  teacher_pk       BIGINT UNSIGNED NOT NULL,                       -- cross-db → identity_db.identity_user
  type             ENUM('lecture','material') NOT NULL DEFAULT 'lecture',
                   -- 'lecture'  : 전체 파이프라인 (STT → ... → YouTube)
                   -- 'material' : RAG 인덱싱만 (강사 사전 자료)
  title            VARCHAR(200),
  subject          VARCHAR(60),
  grade_level      VARCHAR(20),
  class_group      VARCHAR(120),
  recorded_at      TIMESTAMP NULL,
  duration_sec     INT NULL,
  status           ENUM(
                     'queued','transcribing','chunking','rendering',
                     'pending_review','uploading','published','failed','rejected'
                   ) NOT NULL DEFAULT 'queued',
  source_audio_s3  VARCHAR(500),
  source_material_s3 VARCHAR(500),
  failure_reason   VARCHAR(500),
  created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at       TIMESTAMP NULL,
  INDEX idx_org_status (org_pk, status, created_at DESC),
  INDEX idx_teacher (teacher_pk, recorded_at DESC),
  INDEX idx_org_type (org_pk, type, status)
);
```

### 2.4 lecture_chunk (RAG 메타 포함)

```sql
CREATE TABLE lecture_chunk (
  pk               BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id        CHAR(26)     NOT NULL UNIQUE,               -- ULID, API 응답용
  lecture_pk       BIGINT UNSIGNED NOT NULL,
  org_pk           BIGINT UNSIGNED NOT NULL,                       -- denormalized (격리 쿼리 최적화)
  teacher_pk       BIGINT UNSIGNED NOT NULL,                       -- denormalized
  seq              SMALLINT NOT NULL,
  title            VARCHAR(200),
  start_sec        INT NOT NULL,
  end_sec          INT NOT NULL,
  srt_content      TEXT,
  qdrant_point_ids JSON,
  embedding_model  VARCHAR(64),
  rag_indexed_at   TIMESTAMP NULL,
  created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at       TIMESTAMP NULL,
  UNIQUE KEY uk_chunk_lecture_seq (lecture_pk, seq),
  INDEX idx_org (org_pk, created_at DESC),
  INDEX idx_indexing (rag_indexed_at),
  CONSTRAINT fk_chunk_lecture FOREIGN KEY (lecture_pk) REFERENCES lecture(pk)
);
```

### 2.5 video_asset (청크별 파생 자산)

```sql
CREATE TABLE video_asset (
  pk            BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  chunk_pk      BIGINT UNSIGNED NOT NULL,
  lecture_pk    BIGINT UNSIGNED NOT NULL,                          -- denormalized
  type          ENUM(
                  'transcript_json','script_md','srt_subtitle',
                  'audio_mp3','hyperframes_html','video_mp4','thumbnail_png'
                ) NOT NULL,
  s3_url        VARCHAR(500) NOT NULL,
  mime_type     VARCHAR(60),
  size_bytes    BIGINT,
  source_engine VARCHAR(60),
  status        ENUM('pending','ready','failed') NOT NULL DEFAULT 'pending',
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at    TIMESTAMP NULL,
  INDEX idx_chunk_type (chunk_pk, type, status),
  INDEX idx_lecture (lecture_pk, type),
  CONSTRAINT fk_asset_chunk FOREIGN KEY (chunk_pk) REFERENCES lecture_chunk(pk)
);
```

### 2.6 youtube_video (publish 결과)

```sql
CREATE TABLE youtube_video (
  pk                   BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id            CHAR(26)     NOT NULL UNIQUE,           -- ULID, 공유 링크용
  org_pk               BIGINT UNSIGNED NOT NULL,
  channel_pk           BIGINT UNSIGNED NOT NULL,
  lecture_pk           BIGINT UNSIGNED NOT NULL,
  chunk_pk             BIGINT UNSIGNED NULL,                       -- NULL = 전체 concat
  youtube_video_id     VARCHAR(64),
  title                VARCHAR(200),
  description          TEXT,
  tags                 JSON,
  thumbnail_url        VARCHAR(500),
  duration_sec         INT,
  visibility           ENUM('public','unlisted','private') NOT NULL,
  status               ENUM('pending','uploading','published','failed','rejected') NOT NULL,
  published_at         TIMESTAMP NULL,
  view_count_snapshot  INT,
  like_count_snapshot  INT,
  last_synced_at       TIMESTAMP NULL,
  created_at           TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at           TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at           TIMESTAMP NULL,
  INDEX idx_org_status (org_pk, status, published_at DESC),
  INDEX idx_lecture (lecture_pk),
  CONSTRAINT fk_video_lecture FOREIGN KEY (lecture_pk) REFERENCES lecture(pk),
  CONSTRAINT fk_video_channel FOREIGN KEY (channel_pk) REFERENCES youtube_channel(pk)
);
```

### 2.7 student (학생 — MVP 포함)

> v0.1부터 필요: 알림톡 대상 조회(F20-01)와 PIPA 자녀 등장 동의(F25) 모두 학생 테이블에 의존.
> `notification.parent_phone/email`은 발송 시점 스냅샷으로 유지하되, 정규화 SSOT는 여기.

```sql
CREATE TABLE student (
  pk                       BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id                CHAR(26)     NOT NULL UNIQUE,           -- ULID
  org_pk                   BIGINT UNSIGNED NOT NULL,
  user_pk                  BIGINT UNSIGNED NULL,                    -- v1.0+ 학생 계정 연결 시
  display_name             VARCHAR(120) NOT NULL,
  grade_level              VARCHAR(20)  NULL,
  class_group              VARCHAR(120) NULL,                       -- lecture.class_group 매핑 키
  parent_name              VARCHAR(120) NULL,
  parent_phone             VARCHAR(20)  NULL,
  parent_email             VARCHAR(254) NULL,
  consent_video_appearance TINYINT(1)   NOT NULL DEFAULT 0,         -- 자녀 등장 동의 (PIPA)
  consent_pdf_s3_url       VARCHAR(500) NULL,                       -- 만 14세 미만 법정대리인 서명본
  status                   ENUM('ACTIVE','INACTIVE') NOT NULL DEFAULT 'ACTIVE',
  created_at               TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at               TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at               TIMESTAMP NULL,
  INDEX idx_org (org_pk),
  INDEX idx_org_class (org_pk, class_group),
  INDEX idx_user (user_pk)
);
-- ★ consent_video_appearance 초기값 0 (미동의). 명시 동의 시에만 1로 변경.
-- ★ class_group: lecture.class_group과 동일 문자열로 매핑. 알림 대상 조회 키.
-- ★ user_pk: MVP에서는 NULL. v1.0에서 학생이 직접 로그인하면 identity_user와 연결.
-- ★ 알림 흐름: lecture.class_group → student(class_group) → student.parent_phone → notification.
```

### 2.8 notification (발송 로그)

```sql
CREATE TABLE notification (
  pk               BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id        CHAR(26)     NOT NULL UNIQUE,               -- ULID, 읽음 확인 API용
  org_pk           BIGINT UNSIGNED NOT NULL,
  user_pk          BIGINT UNSIGNED NULL,                           -- cross-db → identity_user.pk
  parent_phone     VARCHAR(20) NULL,                              -- 학부모 대상 (별도 계정 없음)
  parent_email     VARCHAR(254) NULL,
  channel          ENUM('alimtalk','sms','email') NOT NULL,
  template_id      VARCHAR(80),
  payload          JSON,
  status           ENUM('queued','sent','failed','read') NOT NULL DEFAULT 'queued',
  provider_msg_id  VARCHAR(120),
  failure_reason   VARCHAR(500),
  related_video_pk BIGINT UNSIGNED NULL,
  sent_at          TIMESTAMP NULL,
  read_at          TIMESTAMP NULL,
  created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_org_status (org_pk, status, created_at DESC),
  INDEX idx_video (related_video_pk)
);
```

### 2.9 trust_relationship (Layer 3 — DIRECTOR가 TEACHER에게 부여)

> `membership`으로 표현하기엔 너무 세밀한 academy 전용 비즈니스 룰.
> 플랫폼 레벨 `delegation_grant`와의 관계: delegation_grant는 플랫폼 공통 capability 위임, trust_relationship은 academy 도메인 세부 scope. 두 레이어 병존.
> 상세 정책은 [`auth-and-policy.md`](auth-and-policy.md) §5.

```sql
CREATE TABLE trust_relationship (
  pk            BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id     CHAR(26)     NOT NULL UNIQUE,                  -- ULID, 위임 확인 링크용
  org_pk        BIGINT UNSIGNED NOT NULL,
  teacher_pk    BIGINT UNSIGNED NOT NULL,                          -- 수혜자 (cross-db → identity_user.pk)
  granted_by_pk BIGINT UNSIGNED NOT NULL,                         -- DIRECTOR (cross-db)
  scope         ENUM(
                  'view_all_lectures',
                  'delete_all_lectures',
                  'auto_publish_own',
                  'auto_publish_all',
                  'view_all_materials',
                  'manage_youtube_channel',                        -- BDD F38-01
                  'approve_videos'                                 -- BDD F40-01
                ) NOT NULL,
  effective_from TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  effective_to   TIMESTAMP NULL,
  reason         VARCHAR(500),
  created_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  deleted_at     TIMESTAMP NULL,
  UNIQUE KEY uk_trust (org_pk, teacher_pk, scope),
  INDEX idx_teacher (teacher_pk, effective_from, effective_to)
);
```

### 2.10 usage_log (서비스별 비용 추적 — 파티셔닝)

> BDD F24-03의 "학원 월 Claude 비용 $148" 기능 근거 테이블.
> append-only. soft delete 없음 — 파티션 DROP으로 3년 보존 후 제거.
> 상세 설계 근거: [`db-design-decisions.md`](db-design-decisions.md) §8.

```sql
CREATE TABLE usage_log (
  pk          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_pk      BIGINT UNSIGNED NOT NULL,
  lecture_pk  BIGINT UNSIGNED NULL,                            -- 어떤 강의 처리 중 발생했는지
  chunk_pk    BIGINT UNSIGNED NULL,
  service     ENUM(
                'GOOGLE_STT',
                'GOOGLE_TTS',
                'CLAUDE',
                'HYPERFRAMES',
                'UPSTAGE_EMBEDDING',
                'YOUTUBE_API',
                'KMS'
              ) NOT NULL,
  units       DECIMAL(12,4)  NOT NULL,                        -- 서비스별 단위: 초(STT/TTS), 토큰(Claude), 건(API)
  cost_usd    DECIMAL(12,6)  NOT NULL,                        -- 정산 기준 USD (DECIMAL — 부동소수점 금지)
  created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (pk, created_at),                               -- 파티션 키를 PK에 포함 (MySQL 필수)
  INDEX idx_org_service (org_pk, service, created_at),
  INDEX idx_org_month (org_pk, created_at)
) PARTITION BY RANGE COLUMNS(created_at) (
  PARTITION p202601 VALUES LESS THAN ('2026-02-01 00:00:00'),
  PARTITION p202602 VALUES LESS THAN ('2026-03-01 00:00:00'),
  PARTITION p202603 VALUES LESS THAN ('2026-04-01 00:00:00'),
  PARTITION p202604 VALUES LESS THAN ('2026-05-01 00:00:00'),
  PARTITION p202605 VALUES LESS THAN ('2026-06-01 00:00:00'),
  PARTITION p202606 VALUES LESS THAN ('2026-07-01 00:00:00'),
  PARTITION pFuture VALUES LESS THAN (MAXVALUE)
);
-- ★ 월별 집계 쿼리: SELECT SUM(cost_usd) FROM usage_log WHERE org_pk=? AND service='CLAUDE'
--   AND created_at >= '2026-05-01' AND created_at < '2026-06-01'
-- ★ 3년 보존 정책: 월 1회 배치로 36개월 초과 파티션 ALTER TABLE ... DROP PARTITION
-- ★ public_id 없음: 외부 API에서 개별 row를 직접 참조할 일 없음
```

**비용 cap 조회 예시 (BDD F24-03):**
```sql
-- 이번 달 Claude 누적 비용 조회
SELECT SUM(cost_usd) AS total_usd
FROM usage_log
WHERE org_pk = :orgPk
  AND service = 'CLAUDE'
  AND created_at >= DATE_FORMAT(NOW(), '%Y-%m-01')
  AND created_at < DATE_FORMAT(NOW() + INTERVAL 1 MONTH, '%Y-%m-01');
```

**units 단위 정의:**

| service | units 의미 |
|---|---|
| `GOOGLE_STT` | 오디오 초(second) |
| `GOOGLE_TTS` | 문자 수(character) |
| `CLAUDE` | 총 토큰(input+output) |
| `HYPERFRAMES` | 생성 슬라이드 수 |
| `UPSTAGE_EMBEDDING` | 임베딩 벡터 수 |
| `YOUTUBE_API` | API 호출 unit cost |
| `KMS` | 복호화 작업 수 |

---

## 3. 멀티테넌시 격리 원칙

- 모든 academy_db 테이블은 `org_pk BIGINT UNSIGNED NOT NULL` 보유 (예외 없음)
- NestJS Interceptor가 모든 쿼리에 `org_pk = current.orgPk` 자동 주입
- Qdrant payload / Neo4j property에도 동일하게 `org_id` 필수 — DB 뿐 아니라 RAG 경계까지
- Soft delete: `deleted_at` 컬럼 통일. Hard delete는 90일 배치
- E2E 테스트: 학원 A 사용자가 학원 B 데이터 조회 절대 불가 (100% 커버)

**Drizzle 강제 패턴:**
```typescript
// org_pk 없는 findOne(pk) 만들지 말 것 — 컴파일 레벨 차단
async findOne(pk: bigint, orgPk: bigint): Promise<Lecture>
```

---

## 4. Qdrant — 단일 Collection

```
collection: lectures
  vector_size: 1024 (Upstage Solar Embedding)
  distance: cosine

  payload schema:
    org_id           (필수, filter)    ← 기존 academy_id → org_id
    teacher_id       (필수, filter)
    lecture_id       (필수)
    chunk_id         (필수)
    chunk_type       ('lecture' | 'material')
    subject          (filter)
    grade_level      (filter)
    title            (display)
    snippet_text     (display)
    visibility       ('public' | 'unlisted' | 'private')
    indexed_at       (timestamp)
    embedding_model
```

### 4.1 필터 전략

| 사용처 | 필터 |
|---|---|
| 강사 RAG (자가 분석) | `org_id == X AND teacher_id == Y` |
| 학원 RAG (학원장 Q&A) | `org_id == X` |
| 학원 RAG + 공개만 (학부모 v1.0+) | `org_id == X AND visibility != 'private'` |
| 글로벌 RAG (v2.0+) | 익명 집계 별도 collection |

### 4.2 Lifecycle

- `lecture publish` 직후 `index-rag` job → upsert
- `lecture delete` 시 `qdrant_point_ids`로 hard delete + MySQL soft delete
- `embedding_model` 변경 시 mass re-embed 배치 (Phase 2+)

---

## 5. Neo4j — 온톨로지

### 5.1 노드 타입 (MVP 8종)

| Node | 정의 | 키 속성 |
|---|---|---|
| `Academy` | 학원 (테넌트 루트) | org_id |
| `Subject` | 과목 | name, org_id |
| `GradeLevel` | 학년 | value, org_id |
| `Chapter` | 단원 | name, subject, grade, org_id |
| `Concept` | 개념 | name, subject, grade, org_id |
| `Definition` | 정의 | text, concept, org_id |
| `Example` | 예제 | text, concept, org_id |
| `Lecture` | 강의 (MySQL mirror) | lecture_id, org_id |
| `Teacher` | 강사 (MySQL mirror) | user_id, org_id |

> `Academy` 노드의 `org_id`는 `identity_db.organization.pk`와 동일한 값.

### 5.2 관계 (MVP 12종)

```
(Academy)─[:HAS_SUBJECT]→(Subject)
(Subject)─[:CONTAINS_CHAPTER]→(Chapter)
(Chapter)─[:CONTAINS_CONCEPT]→(Concept)
(Concept)─[:OF_GRADE]→(GradeLevel)
(Concept)─[:HAS_DEFINITION]→(Definition)
(Concept)─[:DEMONSTRATED_BY]→(Example)
(Concept)─[:PREREQUISITE_OF {weight}]→(Concept)
(Concept)─[:RELATED_TO {weight}]→(Concept)
(Concept)─[:GENERALIZES]→(Concept)
(Lecture)─[:COVERS {weight}]→(Concept)
(Lecture)─[:USES]→(Example|Definition)
(Teacher)─[:TEACHES_LECTURE]→(Lecture)
(Teacher)─[:SPECIALIZES_IN]→(Concept)
```

### 5.3 격리 규칙

- 모든 노드는 `org_id` 속성 필수
- Cypher 미들웨어가 `WHERE n.org_id = $orgId` 자동 주입
- 학원 간 교차 쿼리 금지 (글로벌 RAG는 v2.0 별도 처리)

### 5.4 LLM 추출 → Cypher MERGE

```
Claude 출력 (JSON):
{
  "nodes": [
    {"type": "Concept", "name": "이차함수", "subject": "수학", "grade": "중3"},
    {"type": "Definition", "text": "y=ax²+bx+c", "concept": "이차함수"}
  ],
  "edges": [
    {"from": "Chapter:이차함수와 그래프", "to": "Concept:이차함수", "type": "CONTAINS_CONCEPT"},
    {"from": "Concept:이차식", "to": "Concept:이차함수", "type": "PREREQUISITE_OF", "weight": 0.9}
  ]
}
→ Cypher MERGE (org_id 강제 주입)
```

---

## 6. Cross-store 동기화

### 6.1 강의 publish 흐름

```
1. lecture_chunk INSERT (academy_db)
2. video_asset INSERT (academy_db)
3. youtube_video INSERT/UPDATE (academy_db, status=published)
4. index-rag job:
   - chunk.srt_content → embedding
   - Qdrant upsert (org_id 포함)
   - lecture_chunk.qdrant_point_ids UPDATE
5. extract-neo4j-concepts job:
   - Claude 호출
   - Neo4j MERGE (org_id 포함)
   - lecture_chunk.rag_indexed_at UPDATE
```

### 6.2 강의 delete 흐름

```
1. lecture.deleted_at = NOW (soft delete)
2. youtube_video.status='deleted', YouTube API videos.delete()
3. Qdrant delete by chunk.qdrant_point_ids
4. Neo4j: Lecture 노드 detach + delete (Concept 노드 유지)
```

---

## 7. 인덱싱 / 파티셔닝 (Phase별)

| 테이블 | MVP | v1.0+ | 비고 |
|---|---|---|---|
| `lecture` | 기본 인덱스 | 월 RANGE 파티셔닝 | |
| `lecture_chunk` | 기본 인덱스 | — | |
| `video_asset` | 기본 인덱스 | — | |
| `youtube_video` | 기본 인덱스 | 월 RANGE 파티셔닝 | |
| `notification` | 기본 인덱스 | 월 RANGE 파티셔닝 | |
| `usage_log` | **월 RANGE 파티셔닝** | — | v0.1부터 파티셔닝 필수 (F24 비용 cap) |
| `audit_log` | **월 RANGE 파티셔닝** | — | identity_db. 3년 보존, PK에 created_at 포함 |

학원 1000개 + 일 6편 = 월 18만 row 수준에서 파티셔닝 시작.
`usage_log`와 `audit_log`는 설계 시점부터 파티셔닝 적용 — 나중에 추가 시 테이블 재생성 필요.

---

## 8. 차후 추가 예정 테이블

| 테이블 | 도입 시점 | 용도 |
|---|---|---|
| `student_profile` | v1.0 | 학생 프로필 (org_pk 보유, identity_user 확장) |
| `teacher_review` | v1.0 | 학생 익명 평점 |
| `teacher_evaluation` | v1.0 | 강사 5축 평가 |
| `rag_query_log` | v0.5 | 학원장 RAG Q&A 로그 |
| `teacher_schedule_request` | v1.0 | 스케줄 픽스 워크플로 |
| `class_session` | v2.0 | Android 앱 수업 세션 |
| `attendance` | v0.5 | 출결 |
| `memo` | v0.5 | 상담 메모 |

각 도입 시점에 별도 spec + 마이그레이션 + 본 문서 갱신. 모든 테이블은 `org_pk NOT NULL` 필수.
