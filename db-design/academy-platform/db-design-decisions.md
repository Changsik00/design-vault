# DB 설계 결정 문서 — 장기 운영 관점 재검토

> 작성일: 2026-05-25
> 목적: MVP만 보고 결정하면 나중에 스키마 변경이 리스키한 항목을 선별해 장기 운영 기준으로 결정한다.
> BDD F1~F27, 페르소나(학원장/강사/학생/학부모), 플랫폼 로드맵(v0.1~v3.0)을 교차 검토.

---

## 0. 검토 기준

스키마 변경 리스크는 두 가지로 나뉜다.

| 리스크 | 예시 | 장기 비용 |
|---|---|---|
| **구조 변경** | PK 타입 교체, 컬럼 삭제, FK 재배선 | ALTER TABLE + 전체 앱 수정 + 데이터 마이그레이션 |
| **의미 변경** | 컬럼 용도 재해석, NULL 의미 추가 | 버그 숨김, 감사 추적 혼란 |

MVP만 생각하고 결정했다가 나중에 바꿔야 하는 항목 = **지금 결정해야 하는 항목**.

---

## 1. `display_name` — `identity_db.user_profile` (신규 테이블)

### 배경

- BDD 전 시나리오(F1-01, F3-01, F6-01, F20-03, F26-02 등)에서 사용자 이름 사용
- Option A 이후 `identity_user`에서 `display_name` 제거됨
- "어디에 저장할지" 결정이 없는 상태

### 장기 시나리오 검토

| Phase | 상황 |
|---|---|
| v0.1 | 학원장·강사 이름이 알림톡·검수화면·감사로그에 출력 |
| v1.0 | 학생·학부모 이름도 등록됨 |
| v2.0 | market 판매자, agent 이름 등 서비스 확장 |
| v3.0 | 글로벌: locale, timezone 필요 |

### 결정: `identity_db.user_profile` 신규 생성

**이유**: 사람의 표시 이름·아바타는 학원·마켓·에이전트 어디서나 동일한 플랫폼 공통 정보다. 서비스별로 다른 이름이 필요한 경우(예: 판매자 상호명)는 서비스별 extension 테이블로 처리한다.

```sql
-- identity_db
CREATE TABLE user_profile (
  user_pk      BIGINT UNSIGNED NOT NULL,
  display_name VARCHAR(120)  NOT NULL,
  avatar_url   VARCHAR(500)  NULL,
  locale       VARCHAR(10)   NOT NULL DEFAULT 'ko',
  timezone     VARCHAR(50)   NOT NULL DEFAULT 'Asia/Seoul',
  updated_at   TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (user_pk),
  CONSTRAINT fk_up_user FOREIGN KEY (user_pk) REFERENCES identity_user(pk)
);
-- ★ identity_user와 1:1. identity_user INSERT 즉시 함께 INSERT.
-- ★ 서비스별 추가 프로필(예: academy 강사 소개글)은 academy_db.teacher_profile 등 별도 테이블.
-- ★ SERVICE/SYSTEM type 계정도 row 생성 (시스템 이름 표시 필요).
```

**서비스별 extension 패턴 (나중에 필요할 때):**
```sql
-- academy_db
CREATE TABLE teacher_profile (
  user_pk    BIGINT UNSIGNED NOT NULL,
  org_pk     BIGINT UNSIGNED NOT NULL,
  subjects   JSON,          -- ["수학","영어"]
  bio        VARCHAR(500),
  PRIMARY KEY (user_pk, org_pk)
);
```

---

## 2. `membership.status` — INVITED 제거

### 배경

현재 `membership.status ENUM('ACTIVE','INVITED','SUSPENDED')`.

### 장기 문제

'INVITED' 멤버가 membership 테이블에 있으면:
- "학원 멤버 몇 명?" 쿼리가 INVITED 포함 여부에 따라 다름 → 항상 WHERE status='ACTIVE' 강제
- 초대 거절 시 membership row를 지우는지, INVITED 상태로 두는지 모호
- `membership_invite`와 상태 이중 관리

### 결정: `membership.status` = ACTIVE / SUSPENDED 만

```sql
status ENUM('ACTIVE','SUSPENDED') NOT NULL DEFAULT 'ACTIVE'
```

**초대 흐름 표준화:**
1. 초대 발송 → `membership_invite` INSERT (status='PENDING', token 포함)
2. 수락 → `membership_invite.status='ACCEPTED'` + `membership` INSERT (status='ACTIVE')
3. 거절/만료 → `membership_invite.status='EXPIRED'/'REVOKED'` (membership에 row 없음)
4. 탈퇴 → `membership.status='SUSPENDED'` (identity_user는 유지)

---

## 3. `student` — 지금 스키마 정의, MVP는 부분 구현

### 배경

BDD에서 학생 데이터가 이미 v0.1부터 필요한 것이 확인됨:
- **F20-01**: `lecture.class_group → 학생 목록 → 학부모 phone` (알림톡 발송 대상)
- **F24-01**: 학원 일 업로드 cap 체크 (학원 단위 설정)
- **F25**: 자녀 등장 동의 (`consent_video_appearance`)
- **F26-02**: 학원 폐업 시 학생 데이터 처리

현재 `notification` 테이블에 `parent_phone`, `parent_email`이 denormalized되어 있음.
이것으로는 "학생 김민서의 학부모 연락처" 조회가 불가 — MVP에서도 알림 발송이 필요하므로 **학생 테이블이 v0.1부터 필요하다.**

### 결정: `academy_db.student` 신규 생성 (MVP 포함)

```sql
-- academy_db
CREATE TABLE student (
  pk                       BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id                CHAR(26)     NOT NULL UNIQUE,          -- ULID
  org_pk                   BIGINT UNSIGNED NOT NULL,
  user_pk                  BIGINT UNSIGNED NULL,                   -- v1.0+ 학생 계정 연결 시
  display_name             VARCHAR(120) NOT NULL,
  grade_level              VARCHAR(20)  NULL,
  class_group              VARCHAR(120) NULL,                      -- 반 이름
  parent_name              VARCHAR(120) NULL,
  parent_phone             VARCHAR(20)  NULL,
  parent_email             VARCHAR(254) NULL,
  consent_video_appearance TINYINT(1)   NOT NULL DEFAULT 0,        -- 자녀 등장 동의
  consent_pdf_s3_url       VARCHAR(500) NULL,                      -- 14세 미만 법정대리인 서명본
  status                   ENUM('ACTIVE','INACTIVE') NOT NULL DEFAULT 'ACTIVE',
  created_at               TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at               TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at               TIMESTAMP NULL,
  INDEX idx_org (org_pk),
  INDEX idx_org_class (org_pk, class_group),
  INDEX idx_user (user_pk)
);
-- ★ MVP: user_pk=NULL (학생 계정 없음). v1.0에서 연결.
-- ★ parent_phone/email: notification의 denormalized 컬럼을 여기서 정규화.
-- ★ consent_video_appearance: PIPA 의무. 초기값 0(미동의) — 명시 동의 시만 1.
-- ★ class_group은 lecture.class_group과 매핑 키. 변경 시 lecture 과거 데이터 영향 없음.
```

**알림 발송 흐름 변경:**
- 기존: `notification.parent_phone` denormalized
- 신규: `lecture.class_group` → `student(class_group)` → `student.parent_phone` → `notification`
- `notification.parent_phone`/`parent_email`은 발송 시점 스냅샷으로 유지 (기록 목적)

---

## 4. `usage_log` — API 비용 추적 (신규)

### 배경

BDD F24-03: "A학원의 월 Claude 비용 누적 = $148" → 학원별 비용 추적이 명시적으로 요구됨.
현재 이 데이터를 저장할 테이블이 없음.

### 장기 시나리오

| Phase | 필요 이유 |
|---|---|
| v0.1 | 일/월 비용 cap 체크 (F24-03) |
| v0.5 | 학원장 대시보드에 "이번 달 사용 비용" 표시 |
| v1.0 | 강사별 비용 attribution |
| v2.0 | 학원별 청구 자동화 (billing_db 연동) |

### 결정: `academy_db.usage_log` 신규 생성

```sql
-- academy_db
CREATE TABLE usage_log (
  pk          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_pk      BIGINT UNSIGNED NOT NULL,
  lecture_pk  BIGINT UNSIGNED NULL,
  chunk_pk    BIGINT UNSIGNED NULL,
  service     ENUM('GOOGLE_STT','GOOGLE_TTS','CLAUDE','HYPERFRAMES',
                   'UPSTAGE_EMBEDDING','YOUTUBE_API','KMS') NOT NULL,
  units       DECIMAL(12,4) NOT NULL,       -- 서비스별 단위 (초/토큰/청크/API call)
  cost_usd    DECIMAL(12,6) NOT NULL,       -- 실비용 USD (float 금지 → DECIMAL)
  created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (pk, created_at),             -- 파티셔닝 키 포함
  INDEX idx_org_service (org_pk, service, created_at)
)
PARTITION BY RANGE COLUMNS(created_at) (
  PARTITION p202601 VALUES LESS THAN ('2026-02-01'),
  PARTITION p202602 VALUES LESS THAN ('2026-03-01'),
  PARTITION p202603 VALUES LESS THAN ('2026-04-01'),
  PARTITION p202604 VALUES LESS THAN ('2026-05-01'),
  PARTITION p202605 VALUES LESS THAN ('2026-06-01'),
  PARTITION p202606 VALUES LESS THAN ('2026-07-01'),
  PARTITION p202607 VALUES LESS THAN ('2026-08-01'),
  PARTITION p202608 VALUES LESS THAN ('2026-09-01'),
  PARTITION p202609 VALUES LESS THAN ('2026-10-01'),
  PARTITION p202610 VALUES LESS THAN ('2026-11-01'),
  PARTITION p202611 VALUES LESS THAN ('2026-12-01'),
  PARTITION p202612 VALUES LESS THAN ('2027-01-01'),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
-- ★ DECIMAL(12,6): float 금지. 마이크로달러 단위까지 기록.
-- ★ 파티셔닝: 월별 삭제/보관이 DROP PARTITION 한 줄로 가능.
-- ★ cap 체크: SELECT SUM(cost_usd) WHERE org_pk=? AND service='CLAUDE' AND MONTH=? → 파티션 pruning.
```

---

## 5. `audit_log` — 파티셔닝 + 보존 정책

### 배경

BDD F26-02: "permission_audit_log 는 보존 (의무 기간 3년)".
규모 추정: 학원 100개 × 일 200 이벤트 = 연 730만 row. 3년 = 2,190만 row.

### 결정: 월별 파티셔닝 + 보존 정책 명시

```sql
-- identity_db
CREATE TABLE audit_log (
  pk            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_pk        BIGINT UNSIGNED NULL,
  actor_type    ENUM('HUMAN','API_KEY','SYSTEM') NOT NULL,
  actor_pk      BIGINT UNSIGNED NULL,
  api_key_pk    BIGINT UNSIGNED NULL,
  action        VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50)  NULL,
  resource_pk   BIGINT UNSIGNED NULL,
  result        ENUM('ALLOW','DENY','ERROR') NOT NULL,
  ip            VARBINARY(16) NULL,
  meta_json     JSON NULL,
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (pk, created_at),             -- 파티셔닝 키 포함
  INDEX idx_org_created (org_pk, created_at),
  INDEX idx_actor (actor_pk, created_at)
)
PARTITION BY RANGE COLUMNS(created_at) (
  -- usage_log와 동일 패턴, 월별 파티션
  PARTITION p202601 VALUES LESS THAN ('2026-02-01'),
  -- ...
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

**보존 정책 (코드/운영 문서에 명시):**
- 보존 기간: **3년** (개인정보보호법 의무)
- 파기 방식: `ALTER TABLE audit_log DROP PARTITION p{yyyymm}` (36개월 후 월 1회)
- 개인정보 포함 row: `ip`, `actor_pk` → 파기 전 anonymize job (ip=NULL, actor_pk 유지, meta_json에서 이메일 제거)

---

## 6. `public_id` (ULID) 적용 범위 확정

### 기준

API로 외부에 노출되거나, 클라이언트가 참조하는 ID를 가진 테이블은 모두 `public_id CHAR(26) NOT NULL UNIQUE` 보유.

| 테이블 | public_id 필요 | 이유 |
|---|---|---|
| `identity_user` | ✅ (있음) | API 응답, webhook |
| `organization` | ✅ (있음) | API 응답 |
| `lecture` | ✅ (있음) | 영상 관리 API |
| `student` | ✅ (추가) | 학생 관리 API, 동의 관리 |
| `youtube_video` | ✅ (추가) | 영상 페이지 URL |
| `lecture_chunk` | ✅ (추가) | RAG API 응답, `qdrant_point_ids` 추적 |
| `notification` | ✅ (추가) | 클릭 추적 URL `/track/click/<public_id>` |
| `trust_relationship` | ✅ (추가) | 권한 관리 API |
| `membership_invite` | ❌ — `token` 이 public ID 역할 | |
| `academy_config` | ❌ — `org.public_id` 로 접근 | |
| `youtube_channel` | ❌ — org 단위 1:1, org_pk 기반 조회 | |
| `video_asset` | ❌ — 내부 자산, API 노출 X | |
| `user_profile` | ❌ — `identity_user.public_id` 사용 | |
| `usage_log` | ❌ — 내부 집계 전용 | |
| `audit_log` | ❌ — 내부 감사 전용 | |

---

## 7. Soft Delete 패턴 표준화

### 문제

현재 두 패턴이 혼재:
- `deleted_at TIMESTAMP NULL` (content tables)
- `status ENUM + deleted_at` (일부)
- `status='DELETED'` only (identity_user)

### 결정: 테이블 종류별로 패턴 분리

**상태 머신 테이블** (비즈니스 상태가 핵심인 테이블):
```
identity_user: status ENUM('ACTIVE','SUSPENDED','DELETED') + deleted_at
organization: status ENUM('ACTIVE','SUSPENDED','CLOSED') + deleted_at
membership: status ENUM('ACTIVE','SUSPENDED') — deleted_at 없음 (상태로만 제어)
org_entitlement: status ENUM('ACTIVE','GRACE','SUSPENDED','EXPIRED') — deleted_at 없음
```

**컨텐츠 테이블** (CRUD 대상 리소스):
```
lecture, lecture_chunk, video_asset, youtube_video, youtube_channel,
student, trust_relationship, notification:
  deleted_at TIMESTAMP NULL (soft delete)
  Application WHERE deleted_at IS NULL 자동 적용
```

**이벤트/로그 테이블** (append-only):
```
audit_log, usage_log, payment_ledger: 
  삭제 없음. 보존 기간 경과 시 파티션 DROP.
```

**Hard delete 배치 (90일 규칙):**
- soft delete 후 90일 지난 컨텐츠 테이블 row를 배치로 hard delete
- 단, audit_log/usage_log는 보존 정책 기간(3년)까지 유지

---

## 8. `org_entitlement.feature_limits` — JSON 유지 + 쿼리 전략

### 배경

JSON 컬럼은 개별 키 인덱싱 불가 → 분석 쿼리에 취약.

### 결정: JSON 유지 + 필요 시 Generated Column 추가

MVP에서는 JSON 그대로 사용 (런타임 gate check는 전체 row를 읽어 앱에서 파싱).

분석 필요 시점에 STORED Generated Column 추가 (MySQL 8 online DDL 지원):
```sql
ALTER TABLE org_entitlement
  ADD COLUMN rag_enabled TINYINT(1) GENERATED ALWAYS AS (
    JSON_VALUE(feature_limits, '$.rag_enabled')
  ) STORED,
  ADD COLUMN daily_uploads INT GENERATED ALWAYS AS (
    JSON_VALUE(feature_limits, '$.daily_uploads')
  ) STORED,
  ADD INDEX idx_rag (rag_enabled),
  ADD INDEX idx_daily (daily_uploads);
```

이 ALTER는 데이터 이동 없이 generated column 추가만 하므로 production에서도 안전.

---

## 9. `delegation_grant` capability 확장 전략

### 배경

현재 ENUM:
```
ENUM('PUBLISH_VIDEO','APPROVE_VIDEO','VIEW_ALL_LECTURES',
     'MANAGE_SCHEDULE','MANAGE_MEMBERS','VIEW_BILLING')
```

서비스 확장 시 새 capability 추가 필요 → MySQL ENUM ALTER 필요.

### 결정: VARCHAR(50)으로 변경 + CHECK constraint

```sql
capability VARCHAR(50) NOT NULL,
CONSTRAINT chk_capability CHECK (capability IN (
  'PUBLISH_VIDEO','APPROVE_VIDEO','VIEW_ALL_LECTURES',
  'MANAGE_SCHEDULE','MANAGE_MEMBERS','VIEW_BILLING'
  -- 추가 시 여기에 append (ALTER TABLE, 온라인 DDL)
)),
```

이유: ENUM 추가는 MySQL에서 온라인 DDL이지만 CHECK constraint 수정이 더 명시적이고 ORM과도 잘 맞는다. VARCHAR + 앱 레벨 validation으로 유연성 확보.

단, CASL의 `CAPABILITY_ACTION` 매핑 코드(코드 상수)도 함께 관리 → DB 변경 + 코드 변경 동기화 필수.

---

## 10. 전체 스키마 변경 요약

아래가 이번 재검토로 확정된 변경 사항이다.

### 신규 테이블

| 테이블 | 위치 | 이유 |
|---|---|---|
| `user_profile` | `identity_db` | `display_name` SSOT, 플랫폼 공통 프로필 |
| `student` | `academy_db` | v0.1부터 알림톡 대상 + PIPA 동의 관리 필요 |
| `usage_log` | `academy_db` | BDD F24 비용 cap 체크, 과금 기반 |

### 스키마 수정

| 테이블 | 변경 |
|---|---|
| `membership` | `status` ENUM에서 'INVITED' 제거 → ('ACTIVE','SUSPENDED') |
| `audit_log` | 월별 파티셔닝 + PK에 created_at 포함 |
| `usage_log` | 월별 파티셔닝 + PK에 created_at 포함 |
| `delegation_grant` | `capability` ENUM → VARCHAR(50) + CHECK |
| `youtube_video` | `public_id CHAR(26)` 추가 |
| `lecture_chunk` | `public_id CHAR(26)` 추가 |
| `notification` | `public_id CHAR(26)` 추가 |
| `trust_relationship` | `public_id CHAR(26)` 추가 |

### 컬럼 이동

| 이동 대상 | 기존 위치 | 신규 위치 |
|---|---|---|
| `display_name` | `identity_user` (제거됨) | `identity_db.user_profile` |
| `parent_phone`, `parent_email` | `notification` (denormalized 유지) | `academy_db.student` (정규화, notification은 스냅샷) |

---

## 11. `org_relation` — 독립 학원 + 프랜차이즈 동시 지원

### 배경

학원장이 여러 학원을 운영하는 시나리오가 두 가지로 나뉜다.

| 시나리오 | 구조 | 예시 |
|---|---|---|
| **독립 복수 학원** | 무관한 학원 여러 개를 동일 원장이 운영 | 박원장이 A학원·B학원 각각 DIRECTOR |
| **프랜차이즈/체인** | 그룹 본부 아래 단위 학원 위계 | 한울학원 그룹 → 강남점·강북점·수원점 |

MVP 설계 시 "프랜차이즈는 Enterprise 기능"으로 P2 DEFER 처리했으나, **스키마 나중 추가 비용이 크다**. `org_relation`은 단순 참조 테이블이라 지금 생성해도 비용이 없고, 없으면 나중에 데이터 마이그레이션이 필요하다.

### 결정

**`org_relation` 스키마 v0.1 포함, 코드 P1 연결.**

| 레이어 | 시점 | 내용 |
|---|---|---|
| 테이블 DDL | **P0 (v0.1)** | `org_relation` 생성. 독립 학원은 row 없이 운영 |
| 계층 탐색 코드 | **P1** | `getAncestors(orgPk)`, `getDescendants(orgPk)` 유틸 |
| 정책 상속 | **P1** | 그룹 OWNER의 자식 org 접근 범위 계산 |
| 통합 대시보드 | **P1** | 그룹 단위 집계 API |

### 두 구조의 공존 방식

```
독립 학원 (org_relation row 없음):
  organization(pk=10, type='ACADEMY', name='별빛학원')
  membership(user_pk=박원장, org_pk=10, role='DIRECTOR')

프랜차이즈 (org_relation row 있음):
  organization(pk=1,  type='COMPANY', name='한울학원 그룹')   ← 그룹 부모
  organization(pk=2,  type='ACADEMY', name='한울학원 강남')   ← 단위 학원
  organization(pk=3,  type='ACADEMY', name='한울학원 강북')
  org_relation(parent=1, child=2, relation_type='HQ_BRANCH')
  org_relation(parent=1, child=3, relation_type='HQ_BRANCH')
  membership(user_pk=김지영, org_pk=1, role='OWNER')    ← 그룹 수준
  membership(user_pk=김지영, org_pk=2, role='DIRECTOR') ← 지점 직접 (명시적)
  membership(user_pk=김지영, org_pk=3, role='DIRECTOR')
```

### 멤버십 상속 전략

P1에서 **명시적 멤버십** 방식을 채택한다 (상속 방식 배제).

- **명시적**: 그룹 OWNER도 각 지점 org_pk에 membership row가 있어야 접근 가능
- **상속**: OWNER이면 자식 org 자동 접근 (계층 트리 재귀 탐색 필요)

명시적 방식 채택 이유:
1. 현재 3-gate canXXX() 변경 없이 작동 (Gate A: membership 직접 조회)
2. 특정 지점만 위임하는 케이스 자연 표현 가능 (지점 A는 DIRECTOR, 지점 B는 MEMBER)
3. 계층 재귀 쿼리 없이 단순 JOIN으로 권한 확인

지점이 수백 개인 대형 프랜차이즈에서 상속이 필요해지면, 그때 `canXXX()`에 `getAncestors()` 경로를 추가한다.

### 리스크

- **순환 참조 방지**: `org_relation`에 parent ≠ child CHECK constraint 추가. 앱 레벨에서도 INSERT 시 순환 탐지.
- **깊이 제한**: MVP는 depth 1 (그룹 → 단위학원)만 사용. depth 2 이상(그룹 → 지역본부 → 단위학원)은 P2.

---

## 12. BDD 반영 필요 항목 (우선순위)

| 우선순위 | 항목 | 영향 BDD |
|---|---|---|
| **P0** | `academy_pk` → `org_pk` 전면 교체 | F0.4, F2-01, F4-04, F5-01 |
| **P0** | 학원 가입 → organization + academy_config + org_entitlement 3-step | F1-01 |
| **P0** | 강사 초대 → membership_invite 기반 2-step 재작성 | F3-01~03 |
| **P0** | `student` 테이블 기반 알림 대상 조회 흐름 | F20-01~03 |
| **P0** | `permission_audit_log` → `audit_log` 전체 교체 | F2-02, F3-01, F4-03, F5-01 |
| **P1** | membership 구조 (`product`/`workspace_pk` 제거) | F1-01, F3-01, F4-01, F6-01 |
| **P1** | `identity_user` ENUM 대문자 통일 | F1-01, F3-01, F6-01 |
| **P1** | `user_profile.display_name` 참조로 변경 | F1-01, F3-01 |
| **P1** | `academy.daily_video_limit` → `org_entitlement.feature_limits` | F24-02 |
| **P1** | 비용 cap → `usage_log` 집계 기반으로 변경 | F24-03 |
| **P1** | org_entitlement Gate B 시나리오 추가 | 신규 Feature |
| **P2** | `mvp-scope.md`, `features.md` `academy_id` → `org_id` | 텍스트 교체 |
| **P2** | `perm_version` 동기화 시나리오 추가 | 신규 Feature |
