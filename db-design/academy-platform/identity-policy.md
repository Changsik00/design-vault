# Identity & Membership 정책

> 작성일: 2026-05-23
> **본 문서는 회사 전체 사용자 관리 정책의 단일 진술이다.** 본 monorepo 가 통합 서비스의 표준 reference 가 되는 것을 전제로 작성.
> 모든 결정은 **채택 (Adopted)** 상태. 변경 시 본 문서 + 관련 v3 문서 동시 갱신.

---

## 1. 의도 (Why this exists)

### 1.1 현재 구조의 문제

회사 서비스가 늘어나면서 아래 문제가 누적되고 있다.

| # | 문제 | 영향 |
|---|---|---|
| 1 | **각 서비스마다 user 테이블·인증 로직 중복** | 같은 사람이 서비스마다 별도 계정. SSOT 부재 |
| 2 | **회원관리 모델이 "상점 1개" 가정에 묶임** | 멀티 워크스페이스·멀티 product 시나리오 처리 불가 |
| 3 | **agent 연동 관리가 서비스별로 분산** | 같은 agent 가 product 별로 별도 토큰·권한 체계로 등록. 표준 없음 |
| 4 | **권한 룰이 코드에 산재** | "이 사용자가 무엇을 할 수 있는가" 가 단일 위치에서 답이 안 나옴 |
| 5 | **신규 서비스 onboarding 시 인증 재구현** | 매번 0 부터 출발. 통합 SSO 없음 |
| 6 | **사용자 권한 audit 불가** | 한 사람이 회사 자산 전체에서 무엇에 접근할 수 있는지 한 화면에서 못 봄 |

### 1.2 본 monorepo 의 역할

> **흩어져서 관리되지 않는 회원·권한·agent 통합 정책을 본 monorepo + academy-mvp 서비스를 시발점으로 정착시킨다.**

- 본 monorepo 의 `packages/identity` 가 회사 표준 identity 패키지가 된다
- academy-mvp 가 첫 reference 구현이다
- 차후 신규 서비스는 본 패키지 (또는 차후 추출된 identity-api) 를 표준 의존성으로 채택한다

### 1.3 통합 가치

| 영역 | Before | After |
|---|---|---|
| 사용자 신원 | 서비스마다 별도 계정 | 1 firebase_uid = 회사 전 서비스 동일 사용자 |
| 멀티 워크스페이스 | 미지원 | 1 user = N workspace (academy / content / 미래 product) |
| 권한 관리 | 코드 산재 | 중앙 membership tuple + per-product policy |
| Agent 연동 | 서비스별 토큰 | 동일 identity 모델 (service account 형) |
| 신규 서비스 onboard | 재구현 | `packages/identity` import |
| Audit | 불가 | 한 사용자 = 한 행 + N membership |

---

## 2. 채택된 결정 사항 (Adopted Decisions)

### 2.1 아키텍처 — 3-Layer 분리

```
┌─────────────────────────────────────────────┐
│  Layer 1 — Identity (인증, "누구냐")          │
│     Firebase Authentication                  │
│     - JWT 발급, firebase_uid                 │
├─────────────────────────────────────────────┤
│  Layer 2 — Membership (관계, "어디 소속이냐") │
│     packages/identity                        │
│     - User profile (중앙 identity_user)       │
│     - Membership tuple                       │
│       (user × product × workspace × role)    │
│     - Permission resolver                    │
├─────────────────────────────────────────────┤
│  Layer 3 — Policy (룰, "무엇을 할 수 있냐")    │
│     각 product 의 auth 모듈 (academy-api 등)  │
│     - CASL ability builder                   │
│     - canCreateYoutubeChannel(...) 등        │
│     - trust_relationship 같은 product-룰     │
└─────────────────────────────────────────────┘
```

### 2.2 핵심 결정 9 항목

| # | 영역 | 채택 |
|---|---|---|
| 1 | 인증 provider (Layer 1) | **Firebase Authentication** |
| 2 | Layer 2 배포 형태 | **`packages/identity` (TypeScript 패키지)** — MVP. 차후 service 분리 |
| 3 | 권한 모델 | **CASL (RBAC + ABAC 혼합)** — ReBAC 확장 가능 |
| 4 | 권한 저장 | **Membership tuple** `(user × product × workspace × role)` — Zanzibar style 단순화 |
| 5 | 권한 전달 | **Hybrid** — read 는 Firebase Custom Claims, write/sensitive 는 DB 재검증 |
| 6 | workspace 식별자 | **정수 PK + product (composite)** — URN 은 display only |
| 7 | Agent 연동 | **Service Account 형 identity_user** — 동일 모델, role 만 다름 |
| 8 | 문서 버전 정책 | **v3 in-place 갱신** (v3 가 production 진입 전이므로) |
| 9 | Migration 경로 | **packages → 별도 DB → 별도 service** (3 단계) |

---

## 3. 데이터 모델

### 3.1 핵심 테이블

#### `identity_user` (중앙 사용자)

> 2026-05-25 갱신: Option A 적용. ULID `public_id` 추가, `firebase_uid` nullable, `email` nullable, `display_name` 제거(서비스별 profile 테이블로), ENUM 대문자 통일, `perm_version` 추가. 상세는 [`platform-data-design.md`](platform-data-design.md) §2.

```sql
CREATE TABLE identity_user (
  pk            BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,   -- 내부 FK/JOIN 전용
  public_id     CHAR(26)     NOT NULL UNIQUE,                 -- ULID, 외부 노출용
  firebase_uid  VARCHAR(128) NULL UNIQUE,                     -- SERVICE 계정은 NULL
  email         VARCHAR(255) NULL,                            -- ACTIVE 중 unique (앱 레벨 강제)
  phone_e164    VARCHAR(20)  NULL,
  type          ENUM('HUMAN','SERVICE','SYSTEM') NOT NULL DEFAULT 'HUMAN',
  status        ENUM('ACTIVE','SUSPENDED','DELETED') NOT NULL DEFAULT 'ACTIVE',
  perm_version  BIGINT UNSIGNED NOT NULL DEFAULT 1,           -- 프론트 권한 동기화용
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at    TIMESTAMP NULL,
  INDEX idx_email (email)
);
```

- `type='HUMAN'`: 일반 사람 사용자
- `type='SERVICE'`: AI agent / 자동화 봇 / machine-to-machine (구 `agent`/`service` 통합)
- `type='SYSTEM'`: 시스템 내부 actor (감사 로그 출처 표시용)
- 셋 다 동일 인증·권한 흐름. role 만 product 가 정의.
- `display_name` 제거 → `academy_db.student_profile` 등 서비스별 profile 테이블에 위치.

#### `membership` (관계 튜플)

> 2026-05-25 갱신: Option A 적용. `product` 컬럼 제거(서비스 구분은 org.type + org_entitlement), `workspace_pk` → `org_pk`(organization 직접 참조), `actions JSON` 제거(→ delegation_grant 분리), 복합 PK로 단순화.

```sql
CREATE TABLE membership (
  user_pk    BIGINT UNSIGNED NOT NULL,
  org_pk     BIGINT UNSIGNED NOT NULL,
  role       ENUM('OWNER','DIRECTOR','TEACHER','MEMBER','STUDENT','PARENT') NOT NULL,
  status     ENUM('ACTIVE','INVITED','SUSPENDED') NOT NULL DEFAULT 'ACTIVE',
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_pk, org_pk),
  INDEX idx_org_role (org_pk, role),
  CONSTRAINT fk_mbr_user FOREIGN KEY (user_pk) REFERENCES identity_user(pk),
  CONSTRAINT fk_mbr_org  FOREIGN KEY (org_pk)  REFERENCES organization(pk)
);
```

#### `audit_log` (감사 로그 — append-only)

> 2026-05-25 갱신: `permission_audit_log` → `audit_log`로 확장. 권한 변경 외 모든 중요 action 포함. Human/API_KEY/SYSTEM actor 통합.

```sql
CREATE TABLE audit_log (
  pk            BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  org_pk        BIGINT UNSIGNED NULL,
  actor_type    ENUM('HUMAN','API_KEY','SYSTEM') NOT NULL,
  actor_pk      BIGINT UNSIGNED NULL,            -- identity_user.pk
  api_key_pk    BIGINT UNSIGNED NULL,            -- 머신 호출 시
  action        VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50)  NULL,
  resource_pk   BIGINT UNSIGNED NULL,
  result        ENUM('ALLOW','DENY','ERROR') NOT NULL,
  ip            VARBINARY(16) NULL,
  meta_json     JSON NULL,
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_org_created (org_pk, created_at),
  INDEX idx_actor (actor_pk, created_at)
);
```

append-only. 앱 DB 계정에서 UPDATE/DELETE 권한 제거.

### 3.2 튜플 표현

문자열 표기 (로그/CLI/디버깅용):
```
user:public_id_01HXY... / org:slug-한울학원 / DIRECTOR
```

DB row (실제 저장):
```
(user_pk=42, org_pk=1, role='DIRECTOR', status='ACTIVE')
```

### 3.3 product-specific 정책 (Layer 3)

product 별 추가 룰은 별도 테이블 또는 코드.

**academy-api 예 (`trust_relationship`):**
```sql
CREATE TABLE trust_relationship (
  trust_pk        BIGINT PK,
  academy_pk      BIGINT,
  teacher_user_pk BIGINT,
  granted_by_pk   BIGINT,
  scope           ENUM(
    'view_all_lectures',
    'delete_all_lectures',
    'auto_publish_own',
    'auto_publish_all',
    'view_all_materials',
    'manage_youtube_channel',   -- ⭐ 학원장이 YouTube 채널 관리 위임 (BDD F38-01)
    'approve_videos'            -- ⭐ 학원장이 검수 위임 (BDD F40-01)
  ) NOT NULL,
  effective_from  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  effective_to    TIMESTAMP NULL,
  ...
);
```

**Scope 사용 패턴:**
- `auto_publish_own` / `auto_publish_all` — 검수 우회 (강사 자율성)
- `view_all_lectures` / `delete_all_lectures` / `view_all_materials` — 협업 강사 (학원 내 공유)
- `manage_youtube_channel` — 학원장이 본인 Google 계정으로 채널 admin 불가 시 위임 (현실 케이스)
- `approve_videos` — 학원장 부재 시 검수 권한 위임 (휴가·인수인계)

→ membership 으로 표현하기엔 너무 세밀한 비즈니스 룰. Layer 3 에서 처리.

---

## 4. 권한 흐름

### 4.1 로그인 → 요청 (read)

```
[Client]
   Firebase Auth signIn → ID Token (with custom claims) 획득
       │
       │ Authorization: Bearer <jwt>
       ▼
[academy-api]
   AuthGuard
     1. Firebase Admin SDK verifyIdToken
     2. JWT custom claims 에서 memberships 추출 (in-token)
     3. CASL Ability 빌드
     4. req.user / req.ability 부착
       │
       ▼
   PolicyGuard
     @CheckAbility(ability => ability.can('view', lecture))
       │
       ▼
   Handler 실행
```

→ DB hit = 0.

### 4.2 권한 변경 → claims 동기화

```
[Director PWA]
   POST /admin/trust-relationships
   { teacher_user_pk: 42, scope: 'auto_publish_own' }
       │
       ▼
[academy-api]
   1. DB 에 grant INSERT
   2. permission_audit_log INSERT
   3. Firebase Admin SDK setCustomUserClaims(target_uid, { memberships: [...] })
   4. (옵션) FCM push to target → 클라이언트가 getIdToken(true) 호출 → 즉시 refresh
       │
       ▼
[강사 Client]
   다음 요청부터 새 권한 적용 (즉시 ~ 최대 1시간 TTL)
```

### 4.3 Sensitive mutation (write 의 DB 재검증)

```
[Client]
   POST /videos/:id/publish
       │
       ▼
[academy-api]
   AuthGuard
     ... (4.1 과 동일, JWT claims 로 1차 ability)
       │
       ▼
   PolicyGuard (sensitive=true)
     1. JWT claims 의 ability 1차 체크 → 통과 시
     2. DB 에서 membership 재 fetch (최신 상태)
     3. 새 ability 빌드 + 재 체크 → 통과 시
     4. Handler 실행
       │
       ▼
   publish 실행
```

→ Stale claims 위험 차단. routine read 는 빠른 길, sensitive write 만 추가 DB hit.

### 4.4 Agent 인증 (service account)

```
[Agent (예: scheduling bot)]
   Firebase Custom Token 으로 sign-in (service account 키 보유)
   identity_user.user_type='agent' 로 등록되어 있음
   membership: (user_pk=99, product='academy', workspace_pk=42, role='AGENT_SCHEDULER')
       │
       │ Authorization: Bearer <jwt>
       ▼
[academy-api]
   AuthGuard
     ... (사람 사용자와 동일 흐름)
   PolicyGuard
     AGENT_SCHEDULER role 에 정의된 권한만 가짐
```

→ 사람·agent·service account 가 **동일 모델, 다른 role**. 신규 agent 도입 = 새 role 추가만.

---

## 5. 비교 분석 (Alternatives Considered)

### 5.1 인증 provider — Firebase vs 대안

| 항목 | **Firebase Auth ✓ 채택** | Auth0 | AWS Cognito | Keycloak (자체호스팅) | 자체 구축 |
|---|---|---|---|---|---|
| 비용 (10K MAU) | 무료 | 월 ~$240 | 거의 무료 | 인프라비 | 개발·운영비 ↑↑ |
| 한국 OAuth (Kakao/Naver) | Custom Token 으로 통합 가능 | Custom Connector | 가능 (복잡) | Custom Provider | 직접 구현 |
| Admin SDK | TS/Node 1급 | TS/Node 1급 | TS/Node OK | OIDC | 자체 |
| 운영 부담 | 0 (관리형) | 0 | 0 | 중간 (자체호스팅) | 매우 높음 |
| Vendor lock-in | 중간 | 높음 | 높음 (AWS) | 낮음 | 없음 |
| 한국 회사 신뢰성 | Google 보장 | 미국 회사 | Amazon 보장 | 자체 | — |
| 다중 device 세션 | 표준 | 표준 | 표준 | 표준 | 직접 |
| GDPR / PIPA | 표준 준수 | 표준 | 표준 | 자체 책임 | 자체 책임 |

**채택 사유:**
- 무료 한도 충분 (50K MAU 까지). 학원 plan 가격대에서 운영 비용 부담 0
- Google 생태계 (STT/Vision/TTS 이미 사용) 와 자연 통합
- 카카오/네이버 로그인 Custom Token 으로 흡수 가능
- 회사 전체 표준화 시 vendor 리스크는 monitoring (필요 시 Keycloak 으로 이행 경로 있음)

**기각 사유:**
- Auth0: 비용 누적 빠름 (3K MAU 부터 paid)
- Cognito: AWS lock-in 강함, 한국 OAuth 구현 복잡
- Keycloak: 운영 부담 (Kubernetes 위에 별도 service). 회사 규모 1000+ 학원 시점에 재검토 가능
- 자체 구축: ROI 최악. 보안 책임 100%

### 5.2 Layer 2 배포 — 패키지 vs 서비스 vs Gateway

| 항목 | **packages/identity ✓ 채택** | apps/identity-api (별도 service) | API Gateway (Kong/Envoy) |
|---|---|---|---|
| Network hop | 0 (in-process) | 1 (HTTP) | 1 (proxy) |
| 배포 단위 | 1 | 2 | 2~3 |
| 운영 부담 | 낮음 | 중간 | 높음 |
| Polyglot 지원 | TS only | 모든 언어 | 모든 언어 |
| DB 액세스 | 호출자가 주입 | 자체 DB | 별도 service 호출 |
| 개발 속도 | 빠름 | 느림 | 매우 느림 |
| 적합 단계 | MVP ~ 2nd product | 3rd+ product | 5+ products |

**채택 사유:**
- 현재 user-facing product = academy-api 1개 (content-api 는 internal worker)
- 서비스 추출은 인터페이스만 깨끗하면 v2.0 에 옮기는 비용 낮음
- 패키지로 시작해도 동일한 API 시그니처 유지

**전환 신호:**
- Android 백엔드 (Kotlin) 가 직접 identity 조회 필요 → service 로
- 2번째 user-facing product 출범 → service 로
- 신규 비TS 서비스 → service 로

### 5.3 권한 모델 — RBAC vs ReBAC vs ABAC vs 혼합

| 항목 | RBAC | ReBAC (Zanzibar) | ABAC | **CASL Hybrid ✓ 채택** |
|---|---|---|---|---|
| 표현력 | 낮음 | 매우 높음 | 높음 | 중간~높음 |
| 학습 곡선 | 낮음 | 매우 높음 | 중간 | 낮음 |
| TypeScript 친화 | ★★★ | ★ (OpenFGA SDK 있음) | ★★ | ★★★ |
| 인프라 부담 | 없음 | OpenFGA 서버 | 정책 엔진 | 없음 |
| 변경 대응 | 어려움 (룰 추가 시 코드 변경) | 강함 | 강함 | 중간 |
| 디버깅 | 쉬움 | 어려움 | 어려움 | 쉬움 |

**채택 사유:**
- CASL 은 RBAC (role-based) + ABAC (attribute-based) 모두 지원
- ReBAC 패턴 (관계 기반) 도 ability builder 에서 직접 표현 가능 — 그래프 쿼리는 별도지만 룰 표현은 충분
- 추가 인프라 없음 (npm 라이브러리)
- 확장 시 OpenFGA + CASL 병행도 가능

**기각 사유:**
- 순수 RBAC: trust_relationship 같은 attribute-based 룰 못 담음
- 순수 ReBAC (OpenFGA 단독): MVP 단계 overkill. 학습·인프라 비용
- 순수 ABAC: 룰 정의 복잡, 디버깅 어려움

### 5.4 권한 저장 — In-code / Role Table / Tuple / 정책 엔진

| 항목 | In-code (하드코딩) | Role table (RBAC) | **Membership tuple ✓ 채택** | OpenFGA / Cerbos |
|---|---|---|---|---|
| 변경 즉시성 | 배포 필요 | DB UPDATE | DB UPDATE | 정책 reload |
| Audit | 어려움 | 가능 | 가능 (audit_log) | 강함 |
| 표현력 | 코드만큼 | 낮음 | 중간 (tuple) | 매우 높음 |
| 학습 | 0 | 낮음 | 낮음 | 높음 |
| 인프라 | 없음 | DB | DB | 별도 service |
| 멀티 product | 어려움 | product 마다 별도 | 한 테이블 | 강함 |

**채택 사유:**
- Tuple 모델은 Zanzibar 단순화 버전. 표현력 충분
- 모든 product 가 같은 membership 테이블 공유 → 통합 audit
- 변경 즉시 반영 (DB UPDATE + Firebase claims 동기화)

**기각 사유:**
- 정책 엔진 (OpenFGA): 강력하지만 MVP overkill. 학원 500+ 도달 시 재검토 가능
- In-code: SSOT 없음, 배포 부담
- Role table: 멀티 product 한 곳에서 못 본다

### 5.5 권한 전달 — Custom Claims / Fetch / Hybrid

| 항목 | Custom Claims only | DB Fetch only | **Hybrid ✓ 채택** | Redis cache (60s) |
|---|---|---|---|---|
| DB hit / 요청 (read) | 0 | 1 | 0 | 0 (cache hit) ~ 1 |
| 권한 변경 반영 시간 | 1 hour (token TTL) | 즉시 | sensitive 즉시 / read TTL | 60초 |
| 구현 복잡도 | 낮음 | 낮음 | 중간 | 중간 |
| Stale 권한 위험 (write) | 높음 | 0 | 0 (재검증) | 60초 |
| Token 크기 | claims 1KB 한계 | 무관 | claims 1KB 한계 | 무관 |

**채택 사유 (Hybrid):**
- Read trafficking 99% → JWT claims 로 무료 (latency 0, cost 0)
- Sensitive write (publish, delete, grant) → DB 재검증으로 stale 0
- Token TTL 1 hour → routine read 의 stale risk 수용 가능
- Token refresh push (FCM trigger getIdToken(true)) 로 즉시 동기화 옵션

**기각 사유:**
- Claims only: write stale 위험 (1시간) → publish 같은 행위에 부적합
- DB Fetch only: read traffic 비용 (학원당 GET N00/일 → DB N00 hit)
- Redis cache only: stale 60초 수용 가능하나, hybrid 보다 운영 부담 동일 + claims 의 이점 없음

### 5.6 workspace 식별자 — 정수 PK / URN / UUID

| 항목 | **정수 PK + product ✓ 채택** | URN-style (`wks-001-한울학원`) | UUID |
|---|---|---|---|
| DB index 효율 | 최고 | 중간 | 중간 |
| ORM 친화 | ★★★ | ★★ | ★★ |
| URL 안전 | OK (숫자) | URL encode 필요 (한글) | OK |
| 사람 읽기 | 어려움 (별도 helper) | 쉬움 | 어려움 |
| 글로벌 unique | (product, pk) tuple 로 보장 | 자체 unique | unique |
| 노출 위험 (incremental) | 약간 (rate limit + auth 로 차단) | 낮음 | 낮음 |

**채택 사유:**
- DB key 표준 = 정수 PK
- `(product, workspace_pk)` composite 가 글로벌 unique
- 사람 친화 표현은 `formatWorkspace({product, pk, name})` helper 로 처리
- URL 노출은 auth 로 차단 (incremental ID enumeration 막힘)

**기각 사유:**
- URN: 한글 인코딩 / 길이 / index 효율 트레이드오프
- UUID: ORM 약간 무거움 + 사람 디버깅 어려움. v2.0 글로벌 RAG 단계에서 충돌 회피 위해 일부 도메인에 도입 가능

### 5.7 Agent 연동 — Service Account vs API Key vs 별도 모델

| 항목 | **Service Account (identity_user) ✓ 채택** | API Key | 별도 agent table |
|---|---|---|---|
| 인증 모델 | 사람과 동일 (Firebase Custom Token) | 별도 키 검증 | 별도 |
| 권한 모델 | membership 공유 | 별도 정책 | 별도 |
| 코드 path | 통합 | 분기 (사람 vs key) | 분기 |
| Audit | 통합 (identity_user 같이) | 별도 | 별도 |
| Revoke | identity_user.status='suspended' | 키 회전 | 별도 |
| 신규 agent 도입 | role 추가만 | 인증·정책 신규 구축 | 신규 구축 |

**채택 사유:**
- 사람·agent 가 같은 코드를 탄다 → 인증·권한 분기 없음
- audit 통합 (한 화면에서 사람·agent 행위 추적)
- 차후 LangGraph 기반 multi-agent 도 동일 모델로 흡수

**기각 사유:**
- API Key 만: 권한 표현 약함 (스코프 정도). audit·revoke 어려움
- 별도 table: 코드 path 분기. 신규 agent 도입 시 매번 작업 추가

### 5.8 문서 버전 — v3 in-place vs v4 신규

| 항목 | **v3 in-place ✓ 채택** | v4 신규 |
|---|---|---|
| 정책 일관성 ("큰 패러다임 변경 시 v4") | 모호한 경계 | 명확 |
| v3 lifecycle | 정상 진화 | 1일짜리 historical |
| 변경 범위 | auth layer 한정 (Vite/MySQL/Qdrant 등 영향 X) | 동일 |
| dev 혼동 | 적음 (v3 하나) | 약간 (v3/v4 매핑 필요) |
| production code 영향 | 0 (코드 진입 전) | 0 |

**채택 사유:**
- v3 가 2026-05-23 작성, production code 진입 전
- 변경이 auth layer 한정 (다른 v3 결정 영향 X)
- v4 분리하면 v3 lifecycle 1일 = 의미 없음
- commit message 에 "centralized identity 도입" 명시 + v3/README 의 v1/v2/v3 관계 표 갱신

### 5.9 Migration 경로 — Big Bang vs Phased

| 항목 | **Phased ✓ 채택** | Big Bang (한 번에 service 추출) |
|---|---|---|
| 단계 | packages → DB 분리 → service 분리 | 처음부터 service |
| 첫 번째 product 출시 시간 | 빠름 | 느림 |
| 두 번째 product 시 비용 | 패키지 import (낮음) | 0 (이미 service) |
| 인터페이스 안정성 | 단계별 검증 | 한 번에 결정 |
| Rollback 가능성 | 단계별 | 어려움 |

**채택 사유:**
- YAGNI — service 가 필요해질 때 만든다
- Phase 별 검증: 패키지로 인터페이스 안정화 → DB 분리로 데이터 안정화 → service 추출로 운영 분리
- 각 단계가 다음 단계의 입증

---

## 6. 예상 반박과 답변 (FAQ)

### Q1. "그냥 Firebase Custom Claims 만 쓰면 안 되나?"

**A.** 90% 의 트래픽 (routine read) 에는 충분. 그러나 sensitive mutation (publish, delete, grant 회수) 에서 stale claims 위험이 사업 리스크.

예시: 학원장이 강사 grant 회수 → 강사 token 1시간 동안 stale → 그 사이 강사가 회수 의도된 영상 publish 가능.

Hybrid 는 read 의 성능을 유지하면서 sensitive write 의 stale 위험만 차단.

### Q2. "왜 Firebase 인가? Auth0 / Cognito / Keycloak 다 있는데."

**A.** §5.1 비교 표 참조. 핵심:
- 비용: Firebase 50K MAU 까지 무료. Auth0 는 3K 부터 paid
- Google 생태계 통합 (STT/Vision/TTS 이미 사용)
- 한국 OAuth (Kakao/Naver) Custom Token 으로 흡수
- 운영 부담 0
- Vendor 리스크는 인지. 회사 1000+ 학원 도달 시 Keycloak 이행 path 보유

### Q3. "처음부터 identity-api service 만들지 않으면 나중에 비용 크지 않나?"

**A.** **인터페이스를 패키지로 만든다는 것 자체가 service 추출 준비**다. `packages/identity` 의 함수 시그니처는 service 추출 시 HTTP endpoint 시그니처가 된다. 추가 비용은:
- API contract 정의 (≈ 0.5 일)
- HTTP wrapping (≈ 1 일)
- 배포 + 모니터링 (≈ 2 일)

= 약 3~4 일. 반대로 처음부터 service 만들면:
- HTTP latency, 디버깅 부담, 별도 배포 = 매주 비용
- 첫 product (academy-api) 1개에 비해 ROI 음수

→ 패키지 단계의 비용 회피 > service 추출 비용.

### Q4. "Membership tuple 만으로 권한 다 표현 가능한가? trust_relationship 같은 거 어떻게?"

**A.** Tuple = **포괄적 관계 (멤버십)**. 그 안의 세밀한 비즈니스 룰은 Layer 3 에 둔다.

```
membership: (user=42, academy=1, role=TEACHER)
   ↓
Layer 3 (academy-api):
  trust_relationship 조회 → "이 teacher 는 view_all_lectures grant 보유" → CASL ability 확장
```

membership 은 "어디 소속이냐", trust_relationship 은 "그 소속에서 추가로 무엇을 허락받았냐". 두 layer 가 다른 추상.

### Q5. "agent 도 사람과 같은 테이블에 두면 user_pk 충돌 / 사용량 통계 왜곡 안 나나?"

**A.** `user_type` 필드로 구분. 통계·UI 에서 `WHERE user_type='human'` 필터링.

```sql
-- 사람만 카운트
SELECT COUNT(*) FROM identity_user WHERE user_type='human';

-- agent 로그 추적
SELECT * FROM permission_audit_log
JOIN identity_user ON identity_user.user_pk = permission_audit_log.actor_user_pk
WHERE identity_user.user_type='agent';
```

장점은 인증·권한·revoke 흐름 통합. 단점은 통계 시 type 필터 1줄 추가.

### Q6. "기존 user 테이블 (회원 1개 + 상점 1개 가정) 마이그레이션은?"

**A.** 본 monorepo 의 academy-mvp 는 **신규 서비스** 라 기존 user 테이블이 없다. 즉 즉시 새 모델로 출발 가능. 

기존 회사 서비스의 마이그레이션은 별도 작업:
- 단계 1: 기존 서비스의 user → identity_user 로 sync (read-only mirror, 양방향 dual-write)
- 단계 2: 기존 서비스가 packages/identity 채택 → membership 으로 권한 이전
- 단계 3: 기존 user 테이블 deprecate

→ 본 정책은 신규 표준. 기존 서비스 이행은 점진적.

### Q7. "한 사용자가 여러 학원의 강사인 경우 어떻게?"

**A.** Membership tuple 의 핵심 가치. row 2개:

```
(user_pk=42, product='academy', workspace_pk=1, role='TEACHER')
(user_pk=42, product='academy', workspace_pk=2, role='TEACHER')
```

CASL ability 빌드 시 두 membership 모두 반영. 학원 컨텍스트 스위처는 UI 에서 `workspace_pk` 선택으로 처리.

### Q8. "OpenFGA 같은 정책 엔진 안 쓰는 이유?"

**A.** §5.4 비교. 요약:
- OpenFGA 는 표현력 강력하지만 학원 100개 규모에서 overkill
- 별도 service 운영 부담
- 학습 곡선 (auth model DSL)
- CASL + tuple table 로 95% 의 케이스 커버 가능
- 학원 1000+ 도달 시 OpenFGA 도입 재검토 (정책 엔진은 동일 tuple 모델 위에서 운영 가능)

### Q9. "Firebase 가 한국 OAuth (Kakao/Naver) 미지원인데 어떻게?"

**A.** Custom Token 방식:
1. 클라이언트가 Kakao/Naver SDK 로 OAuth 토큰 받음
2. 백엔드가 그 토큰 검증
3. 검증 통과 시 `auth.createCustomToken(uid, claims)` 로 Firebase Custom Token 발급
4. 클라이언트가 `signInWithCustomToken()` 으로 Firebase 세션 진입
5. 이후 일반 Firebase 사용자처럼 동작

→ MVP 는 email/password, v0.5+ Kakao/Naver 추가.

### Q10. "신규 서비스 onboarding 시 정확히 무엇을 하나?"

**A.** 5 단계:

1. `package.json` 에 `@company/identity` 추가
2. NestJS 에 `FirebaseAuthGuard` 등록 (제공 함수 호출)
3. product-specific role 정의 (예: academy 의 `DIRECTOR`/`TEACHER`)
4. CASL ability builder 구현 (per-product Layer 3)
5. `membership` 테이블에 새 product 값 등록 (옵션: ENUM 확장)

신규 서비스 인증 작업 시간: **반나절 ~ 1 일**.

### Q11. "성능 측정은? Layer 추가하면 느려지나?"

**A.** 측정 기준 (목표):
- Read 요청 latency overhead: < 1ms (JWT verify + claims parse)
- Sensitive write overhead: < 10ms (DB membership fetch)
- Token 발급: Firebase Admin SDK 기준 < 50ms

→ 사용자 체감 영향 없음. 측정은 v0.1 MVP 첫 주에 baseline 확보.

### Q12. "Membership tuple 이 1000만 row 넘으면? 성능 / 인덱스?"

**A.** 인덱스 설계:
- `idx_membership_user (user_pk, status)`: 사용자별 권한 조회 (가장 hot path)
- `idx_membership_workspace (product, workspace_pk, status)`: 워크스페이스 멤버 조회

학원 1000개 × 평균 5명 = 5000 row. 100만 사용자 × 평균 2 membership = 200만 row. 인덱스 정상 작동.

10억 row 시점 (회사 거대화) 도달 시:
- 파티셔닝 (`product` 별 또는 `workspace_pk` hash)
- read replica
- 또는 OpenFGA 같은 전용 정책 엔진으로 이행

→ Phase 별 진화 계획.

### Q13. "권한 변경이 즉시 반영 안 되면 사용자 혼란 안 나나?"

**A.** UI 측면 대응:
- 권한 변경 후 affected 사용자에게 FCM push: "권한이 갱신되었습니다. 화면을 새로고침하세요."
- 클라이언트가 자동 `getIdToken(true)` 호출 → 즉시 새 claims
- 사용자가 못 받았어도 sensitive 액션은 DB 재검증으로 차단됨

→ Bad UX 는 발생 가능하나 보안 violation 은 차단됨.

### Q14. "회사 다른 서비스가 본 monorepo 안 쓰면 어떻게?"

**A.** 본 정책의 핵심은 **identity 표준의 채택**. monorepo 형태는 부수적.

- 현재 monorepo 안 쓰는 서비스 = `packages/identity` 를 npm publish 형태로 배포 → 외부 서비스가 import
- 또는 차후 `apps/identity-api` 추출 후 HTTP API 제공 → 어떤 언어든 사용 가능
- 핵심은 `identity_user` + `membership` 테이블의 DB 가 공유되는 것 (또는 dual-write)

→ monorepo vs polyrepo 는 정책 채택의 형식. 본질은 identity SSOT 표준화.

### Q15. "PIPA / GDPR / 개인정보 처리 방침은?"

**A.** Layer 1 (Firebase) 는 사용자 데이터 US 저장. 학원 가입 시 "국외 이전 동의" 약관 명시.

Layer 2 (identity_user) 는 본 monorepo 의 MySQL (AWS Seoul) 에 저장. 한국 내 저장.

학원 폐업 / 사용자 탈퇴 시:
- `identity_user.status='deleted'` soft delete (30일)
- 30일 후 hard delete (`identity_user` row + `membership` rows)
- 감사 로그 (`permission_audit_log`) 는 의무 보관 기간 (3년 또는 회사 정책) 유지 후 삭제

상세는 [`risks.md`](risks.md) §1.2, §2.5 참조.

---

## 7. Migration 경로 (Phased)

### 7.1 단계

```
[v0.1 MVP — 2026 Q2]
   packages/identity (TS package)
   academy-api 가 같은 MySQL 사용 (identity_user / membership 테이블)
   
       ↓ (2번째 user-facing product 출범 신호)

[v1.0 ~ v2.0 — 2026 Q3+]
   identity 전용 DB schema 분리 (같은 MySQL 의 별도 schema)
   여러 product 가 packages/identity import
   각 product 는 자체 product-specific 테이블 (academy 의 trust_relationship 등) 보유

       ↓ (3rd+ product, polyglot 필요)

[v2.0+ ~ v3.0 — 2027+]
   apps/identity-api 추출
   gRPC / HTTP API 제공
   언어 independent
   API Gateway (Kong/Envoy) 가 front 에서 1차 token verify

       ↓ (회사 1000+ 학원 / 멀티 product 안정)

[v3.0+ — Long-term]
   OpenFGA 도입 검토 (membership tuple 위에 정책 엔진)
   ZTA (Zero Trust Architecture) 통합
```

### 7.2 회사 다른 서비스 이행 가이드

기존 서비스가 본 표준으로 옮길 때:

| 단계 | 작업 | 영향 |
|---|---|---|
| 1 | 기존 user 를 identity_user 에 mirror (read-only dual-write) | 0 (병행 운영) |
| 2 | Firebase Auth 도입 + 기존 인증과 병행 | 사용자 별 마이그레이션 |
| 3 | 기존 user 테이블의 권한 컬럼을 membership 으로 이전 | 기존 권한 로직 코드 변경 |
| 4 | `packages/identity` 채택 + 자체 인증 코드 제거 | 코드 lean |
| 5 | 기존 user 테이블 deprecate | 정리 |

각 단계 1~3개월 예상. 회사 전체 표준화 ~ 1년.

---

## 8. 본 정책의 효과 (KPI 측정)

### 8.1 단기 (v0.1 종료)

- academy-mvp 가 identity_user / membership / Firebase Auth / CASL 풀 스택으로 동작
- 학원장 → 강사 grant 부여 워크플로 검증
- agent 1종 (예: scheduling agent) 도 동일 모델로 인증

### 8.2 중기 (v1.0 ~ v2.0)

- 회사 다른 서비스 2개 이상이 `packages/identity` 채택
- 한 사용자 = 회사 N 서비스 동일 계정
- 통합 audit (한 사용자가 회사 자산 전체에서 뭘 했는지 조회 가능)

### 8.3 장기 (v3.0+)

- 신규 서비스 onboarding 시간: 매번 0 → **0.5 일**
- 회사 전체 권한 변경: 다중 코드 베이스 작업 → **DB UPDATE 1 회**
- 보안 사고 시 한 사용자 회사 전체에서 즉시 차단: 불가 → **`identity_user.status='suspended'` 1 회**

---

## 9. 참고 / 외부 패턴

본 정책의 설계는 다음 업계 패턴에서 차용:

- [**Google Zanzibar**](https://research.google/pubs/pub48190/) — 관계 기반 권한 시스템 (tuple model)
- [**OpenFGA**](https://openfga.dev/) — Zanzibar 오픈소스 구현, Auth0 후원
- [**Google Cloud IAM**](https://cloud.google.com/iam/docs/overview) — resource hierarchy + role bindings
- [**AWS IAM**](https://docs.aws.amazon.com/IAM/latest/UserGuide/intro-structure.html) — ARN-based identity model
- [**CASL**](https://casl.js.org/) — TypeScript native authorization library
- [**Firebase Authentication**](https://firebase.google.com/docs/auth) — managed identity provider
- [**Cerbos**](https://cerbos.dev/) — policy engine (참고용, MVP 미도입)

---

## 10. 변경 이력

| 일자 | 변경 | 비고 |
|---|---|---|
| 2026-05-23 | v3 in-place 갱신: identity-policy.md 신규 + data-model.md 의 `user` → `identity_user + membership` 분리 + auth-and-policy.md 의 Layer 3 정합 | 본 정책 채택 시점 |

---

## 11. 요약 (TL;DR)

> **회사 전체 사용자·권한·agent 를 통합하는 3-Layer identity 표준. 본 monorepo + academy-mvp 가 reference 구현.**

- Layer 1 = Firebase Auth (인증)
- Layer 2 = `packages/identity` (membership tuple, MVP), 차후 service 분리
- Layer 3 = product 별 CASL ability + biz-specific 룰 (trust_relationship 등)
- 인간·agent·service account 모두 같은 모델
- 권한 전달은 hybrid (Custom Claims + DB 재검증)
- 신규 서비스 onboarding 0.5 일 표준화
