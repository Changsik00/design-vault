---
type: core
aliases:
  - E2E 여정
  - e2e-journeys
  - 블랙박스 e2e
  - black-box e2e
tags:
  - platform-db
  - core
  - bdd
  - e2e
  - black-box
---

# platform_db — E2E 사용자 여정 (black-box)

> 상위: [[requirements]] §6 · 화이트박스: [[bdd-scenarios]] · 설계: [[architecture]] · [[schema-reference]]
>
> **이 문서**: 사용자/머신 입장의 **black-box e2e 여정**. API 경계에서만 관찰하고, platform 행위(3-gate·billing projection·perm_version·consent 게이트·BOLA·audit)를 *결과로* 검증한다.

## 왜 별도인가 — 테스트 피라미드

| | [[bdd-scenarios]] (Part A/B) | **이 문서 (Part C)** |
|---|---|---|
| 성격 | 화이트박스 통합 | **블랙박스 e2e** |
| 경계 | DB row·컬럼·트랜잭션 내부 | **서비스 REST API(`/v1/...`)** |
| 단언 | `payment_ledger INSERT type='CHARGE'` 등 내부 상태 | **HTTP status·`body.code`·가시성(보임/안 보임)** |
| 스텝 | 대개 단일 CRUD | **멀티스텝 사용자 여정(여러 액터·호출)** |
| 하베스 | vitest + 레포지토리/서비스 계층 | **서비스 앱 기동 + supertest/Playwright, PG는 sandbox/mock webhook** |

> **규율**: 이 문서의 어떤 시나리오도 DB row/컬럼/테이블을 단언하지 않는다. 상태 변화는 *후속 API 호출(GET)* 로만 검증한다. 내부 정합은 [[bdd-scenarios]]가 담당한다 — 여기서는 **사용자가 관찰할 수 있는 것**만 본다.
>
> **경계 주석**: platform_db 자체에는 HTTP API가 없다. e2e는 서비스 앱(academy-api 등)을 통해 실행되며, `/v1/...` 엔드포인트와 `body.code`는 **행위 계약**(illustrative) — 실제 라우트/에러 카탈로그가 고정되면 그에 맞춘다. ⚠️ 표기는 현재 미구현(구현 후 활성화).

## 고정 페르소나·어휘

| 페르소나 | 역할 |
|---|---|
| 김지영 | 한울학원(A학원) OWNER / ACADEMY DIRECTOR |
| 박교사 (`park@x.com`) | TEACHER |
| 이수민 (`soo@x.com`) | 학생 (academy·market 양 서비스 멤버) |
| 최정훈 | 멀티 워크스페이스 — A학원 TEACHER + B학원 DIRECTOR (단일 firebase_uid) |
| B학원 | 타 테넌트 |
| 운영자 (operator) | 테넌트 role이 **아닌** 별도 신원 평면(MFA) · 승인자(approver) |
| B2B 머신 | `api_key` 신원(SERVICE) |

공통 헤더: `Authorization: Bearer <token>` · `x-org-pk: <org>` · 응답 `X-Perm-Version`.

---

## Part C — Black-box E2E 사용자 여정

### C-01: 신규 학원 가입부터 유료 강의 게시까지 — 결제 게이트가 막고 연다

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
Scenario: 가입 → 강사 초대 → 결제 게이트 차단 → 결제 → 게시 → 타 테넌트 격리
  # 1. 학원장 가입 (신원 + 테넌트 부트스트랩)
  When 김지영이 POST /v1/signup { email, displayName:"김지영", orgName:"한울학원" } 를 호출한다
  Then 201 Created — 응답에 org publicId(ULID)와 accessToken
  And GET /v1/me 는 200, { role:"OWNER", services:[] }

  # 2. 강사 초대 → 수락 (멀티 액터)
  When 김지영이 POST /v1/invites { email:"park@x.com", service:"ACADEMY", role:"TEACHER" } 를 호출한다
  Then 201 Created — 응답에 inviteToken
  When 박교사가 그 토큰으로 가입 후 POST /v1/invites/{token}/accept 를 호출한다
  Then 200 OK
  And 박교사의 GET /v1/me 는 ACADEMY 서비스에 { role:"TEACHER" }

  # 3. 결제 전: 게시 시도가 결제 게이트(Gate B)에 막힌다
  When 박교사가 POST /v1/lectures { title:"중1 수학", publish:true } 를 호출한다
  Then 402 Payment Required — body.code="ENTITLEMENT_REQUIRED" ("구독이 필요합니다")
  And GET /v1/lectures 는 200이되 빈 목록 (게시 안 됨 — 관찰 가능한 부재)

  # 4. 학원장이 구독 결제 (billing → entitlement projection, 내부는 안 봄)
  When 김지영이 POST /v1/billing/subscribe { plan:"ACADEMY_PRO" } 후 PG 결제를 완료한다
  And PG가 결제 성공 webhook을 전송한다 (재전송해도 결과 동일 — 멱등)
  Then GET /v1/billing/status 는 200, { status:"ACTIVE" }

  # 5. 결제 후: 같은 게시가 이제 성공한다 (perm_version 전파 후 권한이 열림)
  When 박교사의 클라이언트가 X-Perm-Version 불일치를 감지해 토큰을 갱신한다
  And 박교사가 동일한 POST /v1/lectures { title:"중1 수학", publish:true } 를 재호출한다
  Then 201 Created — 응답에 lecture publicId
  And GET /v1/lectures 는 방금 게시한 강의 1건을 포함한다
  And 학생 계정의 GET /v1/lectures 에서도 그 강의가 보인다 (게시 가시성)

  # 6. 테넌트 경계: 타 학원은 존재조차 모른다 (BOLA — 403 아님 404)
  When B학원 강사가 GET /v1/lectures/{publicId} 로 직접 접근한다
  Then 404 Not Found (존재 여부 비공개)
```

> **검증 관점**: 가입·초대·게이트·결제·권한열림·격리를 *단일 가치 흐름*으로 묶고, 단언은 전부 HTTP status·`body.code`·가시성뿐. 결제 전 402 → 결제 후 201로 같은 호출이 전환되는 것이 핵심 관찰.
> **통합 화이트박스 / 커버**: P1-01 · P1-02 · P4-01 · P4-03 · P8-01 · P7-01 / USR-7 · RBAC-1 · BILL-3 · ABAC-4 · RBAC-5 · TEN-2 · SEC-1

### C-02: 강사 이전 — consent 게이트 + 데이터 경계가 실제로 작동한다

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
Scenario: 박교사가 A학원에서 B학원으로 이전 — 동의 게이트가 막고 열며, 과거 데이터는 A에 갇힌다
  # 1. 사전 상태: 박교사는 A학원(한울학원) TEACHER, 강의 1건 보유
  Given 박교사(park@x.com)는 A학원에서 POST /v1/lectures { title:"중1 영어", publish:true } 로 강의를 게시해 둔 상태다
  And GET /v1/lectures (헤더 x-org-pk: A학원-pk, Authorization: Bearer park-token) 는 200, 목록에 "중1 영어"가 보인다

  # 2. 동의 없이 이전 시도 → consent 게이트가 막는다
  When 김지영(A학원 OWNER)이 POST /v1/members/{박교사-id}/transfer { toOrg:"B학원-pk" } (x-org-pk: A학원-pk) 를 호출한다
  Then 403 Forbidden — body.code="CONSENT_REQUIRED"
  And GET /v1/me (Bearer park-token) 는 200이되 services 에 B학원 멤버십이 없다 (이전 미발생, 관찰 가능한 부재)

  # 3. 박교사 본인이 동의 → 게이트가 열린다
  When 박교사가 POST /v1/consents { types:["platform.content_ownership","platform.data_transfer"] } (Bearer park-token) 를 호출한다
  Then 201 Created — 응답에 두 동의의 GRANTED 상태

  # 4. B학원이 박교사를 초대하고 OWNER가 이전을 처리한다
  When B학원 OWNER가 POST /v1/invites { email:"park@x.com", role:"TEACHER" } (x-org-pk: B학원-pk) 를 호출한다
  Then 201 Created — 응답에 invite token
  When 박교사가 POST /v1/invites/{token}/accept (Bearer park-token) 를 호출한다
  Then 200 OK
  When 김지영이 POST /v1/members/{박교사-id}/transfer { toOrg:"B학원-pk" } (x-org-pk: A학원-pk) 를 다시 호출한다
  Then 200 OK — body.code="TRANSFERRED"

  # 5. 데이터 경계 관찰 — 본인 identity는 유지, org 데이터는 A에 남는다
  When 박교사가 POST /v1/lectures { title:"중1 영어(B)", publish:true } (x-org-pk: B학원-pk, Bearer park-token) 를 호출한다
  Then 201 Created — B학원 컨텍스트에서 강의 생성 가능 (identity는 이전됨)
  And GET /v1/me (Bearer park-token) 는 200, displayName:"박교사" 와 동일 publicId 유지 (본인 프로필·identity 불변)

  # 6. A학원 시절의 강의·학생·결제는 A에 남아 박교사에게 안 보인다
  When 박교사가 GET /v1/lectures (x-org-pk: A학원-pk, Bearer park-token) 를 호출한다
  Then 403 Forbidden — body.code="NOT_A_MEMBER" (A학원 멤버십이 SUSPENDED — 더 이상 A 컨텍스트 접근 불가)
  And GET /v1/lectures (x-org-pk: B학원-pk, Bearer park-token) 는 200이되 목록에 "중1 영어(B)"만 있고 A학원 시절 "중1 영어"는 없다 (org_pk 격리)
```

> **검증 관점**: 이전이 (a) 동의 없으면 403, (b) 동의·초대·처리 후 200, (c) 사후 가시성으로 "본인 identity·프로필은 따라오고 org 소유 데이터(강의·학생·결제)는 안 따라온다"를 *후속 GET·HTTP status·body.code·목록 가시성만으로* 검증한다.
> **통합 화이트박스 / 커버**: P9-01 · P9-02 · P9-03 / USR-12 · CON-8 · CON-9 · REBAC-1 · ABAC-2 · TEN-2

### C-03: 회원 탈퇴 → 즉시 전서비스 차단 → 동일 이메일 재가입

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
Scenario: 이수민이 탈퇴하면 모든 서비스가 즉시 닫히고, 같은 이메일로 새 계정을 다시 만들 수 있다
  # 1. 사전 상태: 이수민은 academy·market 양쪽 멤버, 토큰 유효
  Given 이수민(soo@x.com)은 GET /v1/me (Bearer soo-token) 호출 시 200, services 에 academy 와 market 이 모두 보인다
  And GET /v1/lectures (x-org-pk: 한울학원-pk, Bearer soo-token) 는 200을 반환한다

  # 2. 탈퇴 — 최종 확인 동의 포함
  When 이수민이 POST /v1/account/withdraw { consent:"platform.withdrawal" } (Bearer soo-token) 를 호출한다
  Then 200 OK — body.code="WITHDRAWN"

  # 3. 즉시 효과: 같은 토큰으로 어떤 서비스 API도 막힌다 (academy·market 동일)
  When 이수민이 GET /v1/me (Bearer soo-token) 를 호출한다
  Then 401 Unauthorized — body.code="ACCOUNT_DELETED"
  When 이수민이 GET /v1/lectures (x-org-pk: 한울학원-pk, Bearer soo-token) 를 호출한다
  Then 403 Forbidden — body.code="ACCOUNT_DELETED" (academy 차단)
  When 이수민이 GET /v1/lectures (x-org-pk: B학원-pk, Bearer soo-token) 를 호출한다
  Then 403 Forbidden — body.code="ACCOUNT_DELETED" (market·타 서비스도 동일하게 차단 — 전 서비스 즉시 무효)

  # 4. 토큰 갱신·재로그인 시도해도 복구 불가
  When 이수민이 POST /v1/auth/refresh (Bearer soo-token) 로 세션 갱신을 시도한다
  Then 401 Unauthorized — body.code="ACCOUNT_DELETED" (DELETED 는 가역 아님 — SUSPENDED 와 달리 복구 경로 없음)

  # 5. 동일 이메일로 신규 가입 — 별개의 새 계정
  When soo@x.com 으로 POST /v1/signup { email:"soo@x.com", displayName:"이수민", orgName:"새출발학원" } 를 호출한다
  Then 201 Created — 응답에 신규 org publicId(ULID)와 새 accessToken (이전과 다른 새 publicId)
  And GET /v1/me (Bearer new-soo-token) 는 200, { role:"OWNER", services:[] } — 과거 academy·market 멤버십이 전혀 보이지 않는다 (이전 활동과 연결 0)
```

> **검증 관점**: 탈퇴 후 (a) 기존 토큰의 전 서비스 호출이 즉시 401/403, (b) 갱신·재로그인 불가, (c) 동일 이메일 재가입이 201이며 새 계정의 me 에 과거 멤버십 없음을 검증. 30일 익명화 배치는 내부라 단언하지 않고 "재가입 가능"이라는 관찰 결과로만 확인.
> **통합 화이트박스 / 커버**: P9-04 · P9-05 / USR-5 · AUTHN-6 · USR-6 · CON-10 · CON-12

### C-04: 위임 가치흐름 — 부여 → 행사 → 회수 → 즉시 차단

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 김지영(OWNER)이 박교사에게 멤버 관리 권한을 위임 → 박교사가 행사 → 김지영이 회수 →
#         박교사가 stale 토큰으로 같은 작업 재시도하면 sensitive write는 DB 재검증으로 즉시 차단.
Scenario: 위임 capability의 부여·행사·회수와 stale 토큰 즉시 차단
  Given 김지영은 한울학원 OWNER/DIRECTOR로 로그인했다 (Bearer 김지영토큰, x-org-pk: 한울)
  And 박교사(park@x.com)는 한울학원 TEACHER로 로그인했다 (Bearer 박교사토큰, x-org-pk: 한울)
  And 박교사는 멤버 역할 변경 권한이 없다

  # 1. 위임 부여 — 김지영이 박교사에게 ACADEMY.MANAGE_MEMBERS 위임
  When 김지영이 POST /v1/delegations { grantee:"park@x.com", capability:"ACADEMY.MANAGE_MEMBERS" } 를 호출한다
  Then 201 Created — 응답에 { id:<delegationId>, capability:"ACADEMY.MANAGE_MEMBERS", status:"ACTIVE" }

  # 2. 위임 행사 — 부여 직후 stale 토큰은 막히고, 갱신 후 성공한다
  When 박교사가 (위임 반영 전 토큰으로) POST /v1/members/{교사B}/role { role:"TEACHER" } 를 호출한다
  Then 403 Forbidden — body.code="PERM_VERSION_STALE" (재발급 유도)
  When 박교사가 X-Perm-Version 불일치를 감지해 토큰을 갱신한 뒤 같은 POST /v1/members/{교사B}/role 를 호출한다
  Then 200 OK — 위임 행사 성공 (멤버 관리 작업 허용)
  And GET /v1/members/{교사B} 는 200, { role:"TEACHER" } (작업 결과가 후속 조회로 관찰됨)

  # 3. 위임 회수 — 김지영이 같은 위임을 DELETE
  When 김지영이 DELETE /v1/delegations/{delegationId} 를 호출한다
  Then 204 No Content
  And GET /v1/delegations 는 200, 목록에 {delegationId}가 더 이상 ACTIVE로 보이지 않는다

  # 4. stale 토큰으로 같은 민감 작업 재시도 — perm_version 불일치, DB 재검증으로 즉시 403
  When 박교사가 *회수 사실이 아직 반영 안 된 stale 토큰*으로 다시 POST /v1/members/{교사B}/role { role:"TEACHER" } 를 호출한다
  Then 403 Forbidden — body.code="DELEGATION_REVOKED"
  And 응답 헤더 X-Perm-Version 이 클라이언트 토큰의 값보다 높다 (불일치 → 재요청 신호)
  And 동일 호출이 단계 2의 200 에서 단계 4의 403 으로 전환됨이 관찰된다 (sensitive write는 stale 토큰이라도 막힘)
```

> **검증 관점**: 같은 `POST /v1/members/{id}/role` 호출이 위임 부여 후 **200**, 회수 후 stale 토큰으로도 **403**으로 전환되는 것 — 민감 쓰기가 토큰 캐시가 아닌 **DB 재검증(perm_version 불일치)**으로 즉시 차단됨 — 을 status·`body.code`·`X-Perm-Version` 헤더로만 단언.
> **통합 화이트박스 / 커버**: P3-01 · P3-02 · P8-01 / REBAC-1 · REBAC-3 · RBAC-5 · NFR-1

### C-05: 멀티 워크스페이스 — 단일 신원, org별 권한·가시성 격리

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 최정훈은 단일 firebase_uid로 A학원 TEACHER + B학원 DIRECTOR다.
#         같은 토큰이라도 x-org-pk 헤더만 바꾸면 권한·가시성이 org별로 격리됨을 관찰한다.
Scenario: 단일 신원이 org 헤더에 따라 다른 권한·가시성을 갖는다
  Given 최정훈은 단일 firebase_uid로 로그인했다 (Bearer 최정훈토큰)
  And 최정훈은 A학원에서 TEACHER, B학원(타 테넌트)에서 DIRECTOR 다

  # 1. 단일 신원 — GET /v1/me 가 두 org 멤버십을 모두 보여준다
  When 최정훈이 GET /v1/me 를 호출한다
  Then 200 OK — memberships 에 { org:"A학원", role:"TEACHER" } 와 { org:"B학원", role:"DIRECTOR" } 가 모두 보인다
  And 두 멤버십은 동일한 단일 사용자(같은 토큰)에 귀속된다

  # 2. A 컨텍스트 — TEACHER라 멤버 역할 변경 거부
  When 최정훈이 같은 토큰에 x-org-pk: A학원 을 실어 POST /v1/members/{교사C}/role { role:"TEACHER" } 를 호출한다
  Then 403 Forbidden — body.code="FORBIDDEN" (A학원에서는 TEACHER, 멤버 관리 권한 없음)
  And GET /v1/members/{교사C} (x-org-pk: A학원) 는 변경되지 않은 역할을 그대로 보여준다

  # 3. B 컨텍스트 — 같은 토큰·같은 작업이지만 DIRECTOR라 허용
  When 최정훈이 같은 토큰에 x-org-pk: B학원 을 실어 POST /v1/members/{교사D}/role { role:"TEACHER" } 를 호출한다
  Then 200 OK (B학원에서는 DIRECTOR, 멤버 관리 권한 보유)
  And GET /v1/members/{교사D} (x-org-pk: B학원) 는 200, { role:"TEACHER" } (작업 결과가 후속 조회로 관찰됨)

  # 4. BOLA — A 컨텍스트에서 B학원 리소스 ID 직접 접근 → 존재 비공개
  Given B학원에 lecture {id: B강의999} 가 존재한다
  When 최정훈이 x-org-pk: A학원 컨텍스트에서 GET /v1/lectures/B강의999 를 호출한다
  Then 404 Not Found — body.code="NOT_FOUND" (타 테넌트 리소스는 존재 자체를 노출하지 않음)
  And 같은 토큰으로 x-org-pk: B학원 을 실어 GET /v1/lectures/B강의999 를 호출하면 200 OK 로 보인다
  And 즉, 동일 토큰·동일 리소스 ID라도 org 헤더에 따라 보임(200)/안 보임(404) 이 갈린다
```

> **검증 관점**: 같은 firebase_uid·같은 토큰으로 `x-org-pk`만 바꿔도 (1) me 는 두 멤버십을 노출, (2) 같은 작업이 A에서 403·B에서 200, (3) 타 org 리소스 ID 접근은 404(존재 비공개)로 갈림. 권한·가시성 격리를 서비스 API status·`body.code`·가시성으로만 단언.
> **통합 화이트박스 / 커버**: P1-02 · P7-01 / USR-7 · TEN-2 · SEC-1 · RBAC-1

### C-06: ⚠️ break-glass 운영자 개입 — 장애 복구 + 전건 감사 + 만료 자동회수

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# ⚠️ 미구현 — operator-plane·support action·break-glass 구현 후 활성화.
Scenario: ⚠️ entitlement projection 장애를 break-glass 승인 경로로 복구하고 만료 후 권한이 자동 회수된다
  # 0. 전제 — 한울학원 결제는 성공했으나 entitlement projection 장애로 권한이 미반영 상태다.
  Given 한울학원의 결제는 성공했으나 entitlement projection 장애로 ACADEMY 권한이 반영되지 않았다
  And 박교사(TEACHER)는 강의 게시 권한을 가진 정상 멤버다
  And 운영자(operator)는 테넌트 role 이 아닌 별도 신원 평면(MFA)으로 로그인했다

  # 1. 장애 증상 — 박교사가 게시하면 Gate B 에서 막힌다.
  When 박교사가 x-org-pk 한울학원으로 POST /v1/lectures 를 호출한다
  Then 402 Payment Required — body.code="ENTITLEMENT_REQUIRED"

  # 2. 운영자는 일반 권한으로 테넌트 데이터를 못 고친다 — operator 는 tenant role 이 아니다.
  When 운영자가 break-glass 없이 POST /v1/support/entitlements 로 entitlement 강제부여를 직접 시도한다
  Then 403 Forbidden — body.code="BREAK_GLASS_REQUIRED"

  # 3. 운영자가 사유·승인자를 명시해 break-glass 승인을 요청한다.
  When 운영자가 POST /v1/support/break-glass { reason:"결제 정상인데 entitlement 미반영", approver:"cto@...", targetOrg:"한울학원" } 를 호출한다
  Then 202 Accepted — body.status="PENDING_APPROVAL", body.id 가 발급된다

  # 4. 승인자가 승인하면 만료시각이 포함된 임시 권한이 부여된다.
  When 승인자가 POST /v1/support/break-glass/{id}/approve 를 호출한다
  Then 200 OK — body.status="ACTIVE", body.expiresAt 가 명시된다

  # 5. 운영자가 임시 권한으로 entitlement 강제부여(support action)를 수행한다.
  When 운영자가 break-glass 세션으로 POST /v1/support/entitlements { org:"한울학원", service:"ACADEMY" } 를 호출한다
  Then 200 OK — body.code="ENTITLEMENT_GRANTED"

  # 6. 모든 운영자 행위가 감사된다 — who/when/why + break_glass 표식 (관찰: 후속 audit API).
  When 운영자가 GET /v1/support/audit?break_glass=true 를 호출한다
  Then 200 OK — body[0] 에 actorType="OPERATOR", reason="결제 정상인데 entitlement 미반영", approver="cto@...", breakGlass=true, occurredAt 가 보인다

  # 7. 복구 확인 — 박교사 게시 재시도가 성공한다.
  When 박교사가 다시 x-org-pk 한울학원으로 POST /v1/lectures 를 호출한다
  Then 201 Created

  # 8. break-glass 만료 후 — 같은 support action 을 재시도하면 권한이 자동 회수돼 막힌다.
  Given break-glass 세션의 expiresAt 이 경과했다
  When 운영자가 같은 break-glass 세션으로 POST /v1/support/entitlements 를 재시도한다
  Then 403 Forbidden — body.code="BREAK_GLASS_EXPIRED"
```

> **검증 관점**: 운영자는 tenant role 이 아니므로 break-glass 없이는 403, 승인된 임시 권한으로만 support action 이 200 이 되고 모든 행위가 `GET /v1/support/audit` 에서 OPERATOR·reason·approver·breakGlass 표식으로 관찰된다. 복구는 박교사 게시 201, 자동 회수는 만료 후 403 으로만 검증.
> **통합 화이트박스 / 커버**: P8-02 · 17-04 · 17-06 / OPER-1 · OPER-2 · SUPP-1 · OPS-2 · AUD-2  ·  ⚠️ 구현 후 활성화

### C-07: ⚠️ api_key B2B 통합 — 발급 → 3-gate 통과 → IP 차단 → revoke 즉시 401

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# ⚠️ 미구현 — api_key 전체(§J 설계레벨) 구현 후 활성화.
Scenario: ⚠️ api_key 를 발급해 사람과 동일 3-gate 로 통합하고 IP·구독·revoke·rotation 을 경계에서 통제한다
  # 0. 전제 — 김지영(OWNER)이 한울학원 구독 ACTIVE 상태로 로그인했다.
  Given 김지영(OWNER)이 로그인했고 한울학원 ACADEMY 구독이 ACTIVE 다

  # 1. 김지영이 api_key 를 발급한다 — secret 은 1회만 노출, 이후엔 prefix 만 조회된다.
  When 김지영이 POST /v1/api-keys { scopes:["lecture:read","lecture:write"], allowedIpCidr:"203.0.113.0/24" } 를 호출한다
  Then 201 Created — body.secret="ak_live_..." (평문 1회 노출), body.id, body.keyPrefix="ak_live_"
  When 김지영이 GET /v1/api-keys/{id} 를 호출한다
  Then 200 OK — body.keyPrefix="ak_live_" 만 보이고 body 에 secret 평문은 없다

  # 2. 머신 클라이언트가 api_key 로 호출하면 사람과 동일 3-gate 를 통과한다(SERVICE 신원).
  When 머신이 Authorization: Bearer ak_live_... 와 x-org-pk 한울학원으로 GET /v1/lectures 를 호출한다
  Then 200 OK — Gate A/B/C 동일 통과, 호출 신원은 SERVICE 다

  # 3. 허용 IP(allowedIpCidr) 밖에서 호출하면 환경속성에서 차단된다.
  When 머신이 203.0.113.0/24 밖의 IP 에서 같은 api_key 로 GET /v1/lectures 를 호출한다
  Then 403 Forbidden — body.code="IP_NOT_ALLOWED"

  # 4. 학원 구독이 만료되면 같은 api_key 도 Gate B 동일 적용으로 막힌다.
  Given 한울학원 ACADEMY 구독이 EXPIRED 로 전이됐다
  When 머신이 허용 IP 에서 같은 api_key 로 POST /v1/lectures 를 호출한다
  Then 402 Payment Required — body.code="ENTITLEMENT_REQUIRED"

  # 5. revoke 후 같은 키로 호출하면 즉시 차단된다.
  Given 한울학원 구독이 다시 ACTIVE 다
  When 김지영이 DELETE /v1/api-keys/{id} 로 revoke 한다
  Then 204 No Content
  When 머신이 revoke 된 api_key 로 GET /v1/lectures 를 호출한다
  Then 401 Unauthorized — body.code="API_KEY_REVOKED"

  # 6. rotation — 새 키 발급 후 구 키도 만료 전까지 동시 동작한다(overlap).
  Given 김지영이 POST /v1/api-keys 로 새 키(키B)를 발급했고, 기존 키A 는 아직 만료 전이다
  When 머신이 키A 로 GET /v1/lectures 를 호출한다
  Then 200 OK — 키A 는 만료 전까지 유효하다
  When 머신이 키B 로 GET /v1/lectures 를 호출한다
  Then 200 OK — 키A·키B 가 overlap 구간에서 동시 동작한다
```

> **검증 관점**: secret 평문 노출은 발급 응답 1회뿐(후속 GET 은 prefix만). 사람·머신 동등성은 동일 3-gate 통과(200)로, 환경속성·구독·revoke 통제는 각각 403(IP)·402(Gate B)·401(revoke) 의 HTTP 경계 결과로만 검증. rotation overlap 은 키A·키B 동시 200 으로 관찰.
> **통합 화이트박스 / 커버**: 16-01~08 / AUTHN-5 · RBAC-4 · SEC-5 · SEC-7 · ABAC-6  ·  ⚠️ 구현 후 활성화

---

## Part D — 예외·불만·생명주기 대처

> 행복 경로(Part C)만으로는 실제 운영의 대부분을 못 본다. 멤버 강퇴·초대 만료·결제 실패·환불·분쟁·역할 거부 등 **예외·불만·생명주기**를 같은 black-box 규율(API 경계·DB 단언 0)로 검증한다. 서브에이전트 3개 병렬 작성(메인 조립) + 설계 대조.

### Domain D1 — 멤버십·초대 생명주기

#### D1-01: 멤버 강제 퇴출 — org 컨텍스트 즉시 차단, identity·타 org는 보존

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 김지영(OWNER)이 박교사를 한울학원에서 제거 → 해당 org 컨텍스트만 즉시 닫히고,
#         본인 identity·프로필·타 org 멤버십은 유지되며, 박교사가 만든 강의는 org에 잔존한다.
Scenario: OWNER가 멤버를 퇴출하면 그 org 컨텍스트만 즉시 403, 본인 신원·타 org·org 데이터는 분리 보존된다
  Given 박교사(park@x.com)는 한울학원 TEACHER로 로그인했다 (Bearer 박교사토큰, x-org-pk: 한울)
  And 박교사는 GET /v1/lectures (x-org-pk: 한울) 가 200이고, POST /v1/lectures { title:"중1 수학", publish:true } 로 강의 1건을 게시해 둔 상태다
  And 박교사는 B학원에서도 TEACHER이며 GET /v1/me 에 한울·B학원 두 멤버십이 모두 보인다

  # 1. OWNER가 박교사를 한울학원에서 제거한다 (멤버십 비활성 — hard delete 아님)
  When 김지영이 DELETE /v1/members/{박교사-id} (x-org-pk: 한울) 를 호출한다
  Then 200 OK — body.code="MEMBER_REMOVED"

  # 2. 즉시 효과: 박교사의 한울 컨텍스트 API 가 차단된다 (민감 쓰기는 @VerifyOnDb 로 stale 토큰이라도 막힘)
  When 박교사가 stale 토큰으로 POST /v1/lectures { title:"중2 수학", publish:true } (x-org-pk: 한울) 를 호출한다
  Then 403 Forbidden — body.code="NOT_A_MEMBER"
  And 박교사의 GET /v1/lectures (x-org-pk: 한울) 도 403 — body.code="NOT_A_MEMBER" (더 이상 한울 멤버 아님)

  # 3. identity·프로필은 불변 — 본인 신원은 살아 있다
  When 박교사가 GET /v1/me (Bearer 박교사토큰) 를 호출한다
  Then 200 OK — displayName 과 publicId 가 동일하게 유지된다 (identity·프로필 보존)
  And memberships 에 한울학원이 더 이상 보이지 않는다 (관찰 가능한 부재)

  # 4. 타 org 멤버십은 영향 없음 — 격리된 컨텍스트
  When 박교사가 GET /v1/lectures (x-org-pk: B학원) 를 호출한다
  Then 200 OK — B학원 컨텍스트는 정상 동작 (퇴출은 해당 org 에만 국한)

  # 5. org 데이터는 org 에 잔존 — 박교사가 만든 강의는 사라지지 않는다
  When 김지영이 GET /v1/lectures (x-org-pk: 한울) 를 호출한다
  Then 200 OK — 박교사가 게시했던 "중1 수학" 이 목록에 그대로 보인다 (org 소유 데이터는 org 에 남음)

  # 6. 재초대로 복귀 가능 — 가역적 관계
  When 김지영이 POST /v1/invites { email:"park@x.com", service:"ACADEMY", role:"TEACHER" } (x-org-pk: 한울) 를 호출한다
  And 박교사가 그 토큰으로 POST /v1/invites/{token}/accept (Bearer 박교사토큰) 를 호출한다
  Then 200 OK — 박교사의 GET /v1/lectures (x-org-pk: 한울) 가 다시 200 (복귀)
```

> **검증 관점**: 퇴출이 (a) 해당 org 컨텍스트만 즉시 403/`NOT_A_MEMBER`, (b) 본인 me·publicId·타 org 200 유지, (c) 게시했던 강의는 OWNER 조회로 잔존 확인, (d) 재초대 수락 후 다시 200 — 멤버십이 *hard delete 가 아닌 가역적 비활성*임을 후속 GET·status·가시성으로만 검증.
> **설계 노트**: 멤버십은 hard delete 금지 — SUSPENDED/제거는 행 보존, org 데이터 소유권은 user 가 아닌 org 에 귀속.
> **통합 화이트박스 / 커버**: P2-01 · P2-02 / RBAC-5 · USR-6 · TEN-2

#### D1-02: 초대 무응답 → 24h 자동 만료 — 만료 후 수락 거부

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 정선생에게 보낸 초대가 24h 무응답으로 자동 만료된다. 자동 만료는 내부 배치이나,
#         관찰은 (a) 만료 후 accept 거부 (b) 목록 GET 의 status="EXPIRED" 로만 한다.
Scenario: 초대가 24h 무응답으로 만료되면 그 토큰의 수락은 거부되고 목록에 EXPIRED 로 보인다
  Given 김지영은 한울학원 OWNER 로 로그인했다 (Bearer 김지영토큰, x-org-pk: 한울)

  # 1. OWNER 가 정선생에게 초대를 발송한다 — 응답에 만료시각(+24h)이 명시된다
  When 김지영이 POST /v1/invites { email:"jung@x.com", service:"ACADEMY", role:"TEACHER" } 를 호출한다
  Then 201 Created — body.inviteToken 과 body.expiresAt (발송 시점 +24시간) 이 명시된다
  And GET /v1/invites (x-org-pk: 한울) 는 200, 해당 초대가 status="PENDING" 으로 보인다

  # 2. 24h 경과 — 정선생이 뒤늦게 수락을 시도하면 거부된다 (자동 만료는 내부 배치, 관찰은 거부로)
  Given 발송 후 24시간이 경과했다
  When 정선생이 그 토큰으로 POST /v1/invites/{token}/accept (Bearer 정선생토큰) 를 호출한다
  Then 410 Gone — body.code="INVITE_EXPIRED"
  And 정선생의 GET /v1/me 에 한울학원 멤버십이 생기지 않는다 (수락 미발생, 관찰 가능한 부재)

  # 3. 목록 관찰 — 만료된 초대는 EXPIRED 상태로 보인다
  When 김지영이 GET /v1/invites (x-org-pk: 한울) 를 호출한다
  Then 200 OK — 해당 초대가 status="EXPIRED" 로 보인다 (PENDING 아님)
```

> **검증 관점**: 24h 만료를 (a) 만료 후 accept 가 410/`INVITE_EXPIRED`, (b) 정선생 me 에 멤버십 부재, (c) 목록 GET status="EXPIRED" 로만 검증. 자동 만료 배치는 내부라 단언하지 않고 *관찰 가능한 수락 거부·목록 상태*로만 확인.
> **통합 화이트박스 / 커버**: P1-03 / RBAC-1 · AUTHN-6

#### D1-03: 초대 수동 취소(REVOKED) — 발송 후 OWNER 가 회수

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: OWNER 가 발송한 초대를 만료 전에 수동 취소(DELETE)하면, 그 토큰의 수락이 즉시 막힌다.
Scenario: OWNER 가 초대를 취소하면 발급된 토큰으로는 더 이상 수락할 수 없다
  Given 김지영은 한울학원 OWNER 로 로그인했다 (Bearer 김지영토큰, x-org-pk: 한울)
  And 김지영이 POST /v1/invites { email:"jung@x.com", service:"ACADEMY", role:"TEACHER" } 로 정선생을 초대했고, 응답의 inviteToken 과 invite id 를 보유한다
  And GET /v1/invites 에 해당 초대가 status="PENDING" 으로 보인다

  # 1. OWNER 가 만료 전에 초대를 수동 취소한다
  When 김지영이 DELETE /v1/invites/{id} (x-org-pk: 한울) 를 호출한다
  Then 204 No Content
  And GET /v1/invites (x-org-pk: 한울) 는 200, 해당 초대가 status="REVOKED" 로 보인다

  # 2. 취소된 토큰으로 수락 시도 → 거부된다
  When 정선생이 그 토큰으로 POST /v1/invites/{token}/accept (Bearer 정선생토큰) 를 호출한다
  Then 403 Forbidden — body.code="INVITE_REVOKED"
  And 정선생의 GET /v1/me 에 한울학원 멤버십이 생기지 않는다 (수락 미발생, 관찰 가능한 부재)
```

> **검증 관점**: 수동 취소가 (a) DELETE 204, (b) 목록 status="REVOKED", (c) 취소된 토큰 accept 가 403/`INVITE_REVOKED` 로 전환됨을 검증. 만료(EXPIRED, D1-02)와 달리 *능동적 회수*라는 점이 `body.code` 로 구분된다.
> **통합 화이트박스 / 커버**: P1-03 / RBAC-1 · RBAC-5

#### D1-04: 초대 재발송 — 구 토큰 무효화, 새 토큰만 유효

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 만료/취소된 초대 후 같은 email 로 재발송하면 새 토큰이 발급되고 구 토큰은 무효가 된다.
Scenario: 같은 email 로 초대를 재발송하면 새 토큰만 수락 가능하고 구 토큰은 막힌다
  Given 김지영은 한울학원 OWNER 로 로그인했다 (Bearer 김지영토큰, x-org-pk: 한울)
  And 정선생(jung@x.com)에게 보낸 이전 초대가 만료/취소되어 GET /v1/invites 에 status="EXPIRED"(또는 "REVOKED") 로 보인다
  And 김지영은 그 이전 초대의 구 토큰(old-token)을 보유한다

  # 1. 같은 email 로 재발송 → 새 토큰이 발급된다
  When 김지영이 POST /v1/invites { email:"jung@x.com", service:"ACADEMY", role:"TEACHER" } 를 다시 호출한다
  Then 201 Created — body.inviteToken(new-token)이 old-token 과 다르고, body.expiresAt 가 +24h 로 갱신된다
  And GET /v1/invites 에 해당 email 의 활성 초대가 status="PENDING" 1건으로 보인다 (PENDING 이 중복 누적되지 않음)

  # 2. 구 토큰은 무효 — 새 발송이 이전 토큰을 대체한다
  When 정선생이 old-token 으로 POST /v1/invites/{old-token}/accept (Bearer 정선생토큰) 를 호출한다
  Then 403 Forbidden 또는 410 Gone — body.code IN ("INVITE_REVOKED","INVITE_EXPIRED") (구 토큰은 더 이상 유효하지 않음)

  # 3. 새 토큰으로 수락 → 성공한다
  When 정선생이 new-token 으로 POST /v1/invites/{new-token}/accept (Bearer 정선생토큰) 를 호출한다
  Then 200 OK
  And 정선생의 GET /v1/me 는 ACADEMY 서비스에 { role:"TEACHER" } 가 보인다 (멤버십 생성됨)
```

> **검증 관점**: 재발송이 (a) old-token 과 다른 new-token·갱신된 expiresAt, (b) 목록에 활성 PENDING 1건만(중복 누적 없음), (c) 구 토큰 accept 거부·새 토큰 accept 200 으로 전환됨을 검증.
> **통합 화이트박스 / 커버**: P1-03 / RBAC-1 · USR-7

#### D1-05: 중복 초대 / 이미 멤버 — 멤버십 중복 생성 방지

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 이미 TEACHER 인 박교사에게 다시 초대를 보내면 멤버십이 중복 생성되지 않는다.
Scenario: 이미 멤버인 대상에게 재초대하면 충돌로 거부되고 멤버십은 중복 생성되지 않는다
  Given 박교사(park@x.com)는 한울학원의 ACADEMY TEACHER 다 (GET /v1/me 에 { service:"ACADEMY", role:"TEACHER" })
  And 김지영은 한울학원 OWNER 로 로그인했다 (Bearer 김지영토큰, x-org-pk: 한울)

  # 1. 이미 멤버인 박교사에게 동일 service 로 다시 초대 → 충돌
  When 김지영이 POST /v1/invites { email:"park@x.com", service:"ACADEMY", role:"TEACHER" } 를 호출한다
  Then 409 Conflict — body.code="ALREADY_MEMBER"

  # 2. 멤버십은 그대로 — 중복 생성 안 됨
  When 박교사가 GET /v1/me (Bearer 박교사토큰) 를 호출한다
  Then 200 OK — ACADEMY 멤버십이 { role:"TEACHER" } 1건으로 유지된다 (중복·역할 변동 없음)

  # 3. 역할 변경이 목적이라면 별도 경로 — 재초대로는 권한이 바뀌지 않는다
  When 김지영이 박교사의 역할을 바꾸려면 POST /v1/members/{박교사-id}/role { role:"DIRECTOR" } 를 사용한다
  Then 200 OK — 멤버 역할 변경은 초대가 아닌 role 변경 API 가 담당한다 (관찰: GET /v1/members/{박교사-id} 가 role:"DIRECTOR")
```

> **검증 관점**: 중복 초대가 (a) 409/`ALREADY_MEMBER`, (b) me 의 멤버십이 1건으로 불변임을 검증.
> **설계 노트**: 구현이 멱등이라면 재초대를 201 + 기존 멤버 role 갱신으로 처리할 수도 있으나, 어느 쪽이든 *멤버십 중복 행이 생기지 않음* 이 관찰 불변식 — service_membership PK `(user_pk, org_pk, service)` 가 중복을 구조적으로 차단.
> **통합 화이트박스 / 커버**: P1-02 · P2-03 / RBAC-1 · USR-7

#### D1-06: 마지막 OWNER lockout 방지 — 다른 OWNER 지정 전엔 강등·탈퇴 거부

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 유일한 OWNER 가 본인을 강등/탈퇴하려 하면 org 좀비화를 막기 위해 거부되고,
#         다른 OWNER 를 지정한 뒤에야 허용된다.
Scenario: 유일 OWNER 의 자기 강등·탈퇴는 거부되고, 새 OWNER 지정 후에는 허용된다
  Given 김지영은 한울학원의 유일한 OWNER 로 로그인했다 (Bearer 김지영토큰, x-org-pk: 한울)
  And 박교사(park@x.com)는 한울학원 MEMBER/TEACHER 다

  # 1. 유일 OWNER 가 본인을 강등 시도 → lockout 방지로 거부
  When 김지영이 POST /v1/members/{김지영-id}/role { role:"MEMBER" } (x-org-pk: 한울) 를 호출한다
  Then 400 Bad Request — body.code="LAST_OWNER" ("마지막 OWNER 는 강등할 수 없습니다")
  And GET /v1/members/{김지영-id} (x-org-pk: 한울) 는 여전히 platformRole:"OWNER" (변경 없음)

  # 2. 유일 OWNER 가 본인 탈퇴(멤버십 이탈) 시도 → 동일하게 거부
  When 김지영이 DELETE /v1/members/{김지영-id} (x-org-pk: 한울) 를 호출한다
  Then 400 Bad Request — body.code="LAST_OWNER"
  And GET /v1/members (x-org-pk: 한울) 에 OWNER 가 여전히 김지영 1건으로 보인다 (최소 1 OWNER 보존)

  # 3. 다른 멤버를 OWNER 로 지정한 뒤에는 강등이 허용된다
  When 김지영이 POST /v1/members/{박교사-id}/role { role:"OWNER" } 를 호출한다
  Then 200 OK — GET /v1/members 에 OWNER 가 김지영·박교사 2건으로 보인다
  When 김지영이 다시 POST /v1/members/{김지영-id}/role { role:"MEMBER" } 를 호출한다
  Then 200 OK — 이제 OWNER 가 박교사 1명 남으므로 강등 허용
  And GET /v1/members 에 OWNER 가 박교사 1건으로 보인다 (강등 후에도 최소 1 OWNER 유지)
```

> **검증 관점**: 마지막 OWNER 보호를 (a) 자기 강등·탈퇴가 동일하게 400/`LAST_OWNER`, (b) 거부 후 OWNER 가 그대로 1건, (c) 새 OWNER 지정 후 같은 강등이 200 으로 전환됨을 검증. §N.3 의 `FOR UPDATE` 가드는 내부라 단언하지 않고 *관찰 가능한 거부·OWNER ≥1 유지*로만 확인.
> **통합 화이트박스 / 커버**: 17-10 · P2-04 / OWN-1 · RBAC-6 · RBAC-5

#### D1-07: 마지막 OWNER 동시 강등(동시성) — 레이스에서 최소 1 OWNER 보존

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: OWNER 2명이 동시에 서로를 강등 시도하는 레이스에서, 하나만 성공하고
#         최소 1명의 OWNER 가 항상 보존됨을 사후 GET 으로만 관찰한다.
Scenario: 두 OWNER 가 동시에 서로를 강등하면 하나는 200·하나는 409 로 갈리고 OWNER 는 ≥1 유지된다
  Given 한울학원에 OWNER 가 김지영·박교사 2명이다 (GET /v1/members 에 OWNER 2건)
  And 김지영(Bearer 김지영토큰)과 박교사(Bearer 박교사토큰)가 각자 관리 권한으로 로그인했다

  # 1. 두 관리자가 *동시에* 서로를 강등 시도한다 (레이스)
  When 김지영이 POST /v1/members/{박교사-id}/role { role:"MEMBER" } 를 호출하고
  And 동시에 박교사가 POST /v1/members/{김지영-id}/role { role:"MEMBER" } 를 호출한다

  # 2. 정확히 하나만 성공한다 — 직렬화로 동시 강등이 직렬화된다
  Then 한 요청은 200 OK 이고, 다른 한 요청은 409 Conflict — body.code="LAST_OWNER"
  And 두 요청이 동시에 200 이 되지는 않는다 (둘 다 성공 → OWNER 0 인 상황은 발생하지 않음)

  # 3. 사후 관찰 — OWNER 는 최소 1명 보존된다 (정확히 어느 쪽이 남는지는 레이스 결과)
  When 사후에 GET /v1/members (x-org-pk: 한울) 를 호출한다
  Then 200 OK — platformRole="OWNER" 인 멤버가 정확히 1명 보인다 (OWNER ≥ 1 보존, org 좀비화 방지)
```

> **검증 관점**: 동시 강등 레이스에서 (a) 한쪽 200·다른 쪽 409/`LAST_OWNER`, (b) 둘 다 200 은 발생하지 않음, (c) 사후 GET 의 OWNER count 가 정확히 ≥1 임을 검증. `FOR UPDATE` 직렬화(§N.3 1차 가드)는 내부라 단언하지 않고 *결과로서의 최소 1 OWNER 보존*만 관찰.
> **통합 화이트박스 / 커버**: 17-10 (동시 강등 레이스) / OWN-1 · RBAC-6

#### D1-08: SUSPENDED 멤버 재활성화 — 정지는 가역(DELETED 와 대비)

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 정지(SUSPENDED)된 멤버를 OWNER 가 복원하면 정상 접근이 돌아온다 —
#         탈퇴(DELETED, C-03)가 비가역인 것과 대비되는 *가역적* 상태 전이다.
Scenario: 정지된 멤버를 OWNER 가 재활성화하면 다시 정상 접근이 가능해진다
  Given 박교사(park@x.com)는 한울학원 TEACHER 였으나 멤버십이 SUSPENDED 상태다
  And 박교사가 GET /v1/lectures (x-org-pk: 한울, Bearer 박교사토큰) 를 호출하면 403 — body.code="NOT_A_MEMBER" (정지 중 차단)
  And 김지영은 한울학원 OWNER 로 로그인했다 (Bearer 김지영토큰, x-org-pk: 한울)

  # 1. OWNER 가 정지된 멤버를 복원한다
  When 김지영이 POST /v1/members/{박교사-id}/reactivate (x-org-pk: 한울) 를 호출한다
  Then 200 OK — body.code="MEMBER_REACTIVATED"
  And GET /v1/members/{박교사-id} (x-org-pk: 한울) 는 200, status="ACTIVE" (정지 해제)

  # 2. 복원 후: 박교사의 정상 접근이 돌아온다 (perm_version 전파 후)
  When 박교사가 X-Perm-Version 불일치를 감지해 토큰을 갱신한 뒤 GET /v1/lectures (x-org-pk: 한울) 를 호출한다
  Then 200 OK — 한울 컨텍스트 접근이 다시 열린다 (가역적 복원)

  # 3. 대비 관찰 — SUSPENDED 는 복원되지만 DELETED(탈퇴)는 복원 경로가 없다
  Then SUSPENDED 멤버십은 reactivate 로 200 이 되는 반면, 탈퇴(account DELETED, C-03)는 어떤 호출로도 복구되지 않는다 (가역 vs 비가역 대비)
```

> **검증 관점**: 재활성화가 (a) 200/`MEMBER_REACTIVATED`, (b) 멤버 status 가 ACTIVE 로 관찰, (c) 정지 중 403 → 복원 후 200 으로 같은 호출이 전환됨을 검증. SUSPENDED 의 *가역성* 을 DELETED(C-03 의 비가역 탈퇴)와 대비.
> **설계 노트**: 멤버십은 hard delete 금지 — 제거/정지는 모두 행 보존이며 ACTIVE↔SUSPENDED 는 가역, identity DELETED 만 비가역.
> **통합 화이트박스 / 커버**: P2-01 · P2-02 / RBAC-5 · USR-6 · AUTHN-6

### Domain D2 — 결제·구독 예외·금전 불만

#### D2-01: 결제 실패 → 상태 전이 (ACTIVE → PAST_DUE → GRACE → EXPIRED)

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 한울학원 ACADEMY 구독이 ACTIVE인데 갱신 결제가 실패한다.
#         PAST_DUE → GRACE(유예) 동안에는 박교사 게시가 여전히 200,
#         GRACE가 만료되면 EXPIRED로 전이돼 같은 게시가 402로 막힌다.
Scenario: 갱신 결제 실패가 GRACE 동안은 서비스를 열어두고 만료 후 닫는다
  Given 김지영(OWNER)이 로그인했고 한울학원 ACADEMY 구독이 ACTIVE 다
  And 박교사(park@x.com)는 한울학원 TEACHER 로 게시 권한을 가진 정상 멤버다
  And GET /v1/billing/status 는 200, { status:"ACTIVE" } (배너 없음)

  # 1. 갱신 시점에 카드 결제가 실패한다 — PG mock이 결제 실패 webhook을 전송한다.
  When PG가 갱신 결제 실패(card_declined) webhook을 전송한다 (재전송해도 결과 동일 — 멱등)
  Then GET /v1/billing/status 는 200, { status:"PAST_DUE" } — body.banner 에 결제 실패 안내가 실린다

  # 2. PAST_DUE → GRACE 유예 — 배너는 떠도 게시는 여전히 열려 있다.
  When 시스템이 PAST_DUE 를 GRACE 로 전이시킨다 (유예 기간 시작)
  Then GET /v1/billing/status 는 200, { status:"GRACE" }, body.banner="결제 실패 — N일 내 갱신 필요"
  When 박교사가 POST /v1/lectures { title:"중1 수학", publish:true } 를 호출한다
  Then 201 Created — GRACE 동안 게시 가능 (Gate B 통과)
  And GET /v1/lectures 는 200, 방금 게시한 강의를 포함한다

  # 3. GRACE 만료 → EXPIRED — 같은 게시 호출이 이제 402로 막힌다.
  When GRACE 유예가 만료돼 구독이 EXPIRED 로 전이된다
  Then GET /v1/billing/status 는 200, { status:"EXPIRED" }, body.banner="구독 만료"
  When 박교사가 동일한 POST /v1/lectures { title:"중1 영어", publish:true } 를 호출한다
  Then 402 Payment Required — body.code="ENTITLEMENT_REQUIRED"
  And GET /v1/lectures 는 200이되 EXPIRED 이후 게시한 강의는 목록에 없다 (관찰 가능한 부재)
```

> **검증 관점**: 같은 게시 호출이 ACTIVE/GRACE 에서 201, EXPIRED 에서 402 로 전환되는 것 — 그리고 각 단계가 GET /v1/billing/status 의 status·banner 로만 관찰됨 — 을 단언. PAST_DUE/GRACE 구분과 "GRACE는 접근 유지"가 핵심.
> **통합 화이트박스 / 커버**: P4-03 · P4-04 · 17-08 / BILL-6 · AVAIL-1

#### D2-02: 카드 거절 후 재결제 복구 (GRACE → ACTIVE)

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
Scenario: GRACE 중 카드 교체·재결제가 성공하면 구독이 ACTIVE로 복귀하고 게시가 다시 열린다
  Given 한울학원 ACADEMY 구독이 GRACE 다 (D2-01 의 유예 상태)
  And GET /v1/billing/status 는 200, { status:"GRACE" }, body.banner="결제 실패 — N일 내 갱신 필요"
  And 박교사는 GRACE 동안 게시가 아직 201 로 가능하다

  # 1. 김지영이 카드를 교체하고 재결제를 진행한다.
  When 김지영이 POST /v1/billing/payment-method { card:"new-card-token" } 를 호출한다
  Then 200 OK — 새 결제수단이 등록된다
  When 김지영이 POST /v1/billing/retry 로 재결제를 요청하고 PG mock 이 결제 성공 webhook 을 전송한다 (재전송해도 멱등)
  Then GET /v1/billing/status 는 200, { status:"ACTIVE" } — body.banner 가 사라진다 (배너 없음)

  # 2. 복구 확인 — 박교사 게시가 GRACE 배너 없이 정상 201.
  When 박교사가 X-Perm-Version 불일치를 감지해 토큰을 갱신한 뒤 POST /v1/lectures { title:"중2 수학", publish:true } 를 호출한다
  Then 201 Created
  And GET /v1/lectures 는 200, 방금 게시한 강의를 포함한다 (정상 운영 복귀)
```

> **검증 관점**: GRACE → 재결제 → ACTIVE 복귀가 status 변화와 배너 소멸로 관찰되고, 같은 게시가 끊김 없이 201 을 유지함을 status·배너로만 단언. (D2-01 의 EXPIRED 이전 회복 경로.)
> **통합 화이트박스 / 커버**: P4-01 · P4-04 · 13-05 / BILL-6 · BILL-4

#### D2-03: ⚠️ webhook 유실 → 사용자 불만 → reconciliation 복구

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# ⚠️ 미구현 — reconciliation(대사) 잡·운영자 복구 경로 구현 후 활성화.
Scenario: ⚠️ 결제는 됐는데 webhook 유실로 권한이 안 열려 사용자가 불만을 제기하고, 대사로 복구된다
  Given 김지영(OWNER)이 POST /v1/billing/subscribe { plan:"ACADEMY_PRO" } 후 PG 결제를 정상 완료했다
  And 박교사는 한울학원 TEACHER 로 게시 권한을 가진 정상 멤버다

  # 1. webhook 유실 — 결제는 성공했으나 성공 webhook 이 전달되지 않아 권한이 미반영이다.
  When PG 가 결제 성공 webhook 을 전송하지 못한다 (네트워크 유실 — 시뮬레이션)
  Then GET /v1/billing/status 는 200이되 { status } 가 아직 ACTIVE 가 아니다 (반영 지연)

  # 2. 사용자 불만 — 박교사가 게시하면 결제했는데도 막힌다.
  When 박교사가 POST /v1/lectures { title:"중1 과학", publish:true } 를 호출한다
  Then 402 Payment Required — body.code="ENTITLEMENT_REQUIRED" ("결제했는데 왜 막히나" — 사용자 불만 지점)
  And 김지영의 GET /v1/billing/receipts 는 200, 결제 영수증 1건이 보인다 (결제 자체는 성공 — 불일치 증거)

  # 3. 대사(reconciliation) 또는 운영자 개입으로 결제↔권한 불일치를 복구한다.
  When ⚠️ reconciliation 잡이 결제 성공과 미반영 권한의 불일치를 감지해 권한을 복구한다
    (또는 운영자가 ⚠️ support action 으로 entitlement 를 강제 부여한다 — D2-07 참조)
  Then GET /v1/billing/status 는 200, { status:"ACTIVE" }

  # 4. 복구 확인 — 박교사 게시 재시도가 성공한다.
  When 박교사가 토큰 갱신 후 동일한 POST /v1/lectures { title:"중1 과학", publish:true } 를 재호출한다
  Then 201 Created
  And GET /v1/lectures 는 200, 방금 게시한 강의를 포함한다
```

> **검증 관점**: "결제 영수증은 있는데(=GET /v1/billing/receipts 1건) 게시는 402"라는 사용자 관찰 가능한 불일치가 핵심. 복구는 status ACTIVE 복귀와 게시 201 로만 검증하고, 대사/운영자라는 *내부 메커니즘*은 단언하지 않는다.
> **통합 화이트박스 / 커버**: 13-07 · 17-06 · 17-08 / BILL-2 · SUPP-1 · AVAIL-1  ·  ⚠️ reconciliation 미구현, 구현 후 활성화

#### D2-04: 중복 결제 방지 (이중과금 불만)

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
Scenario: 동일 결제를 네트워크 재시도로 두 번 제출해도 멱등하게 1건만 청구된다
  Given 김지영(OWNER)이 한울학원 ACADEMY 구독 결제를 진행 중이다
  And GET /v1/billing/receipts 는 200, 빈 목록이다 (아직 청구 없음)

  # 1. 같은 결제를 동일 idempotency-key 로 두 번 제출한다 (클라이언트 네트워크 재시도).
  When 김지영이 POST /v1/billing/charge (헤더 Idempotency-Key: pay-001) 를 호출한다
  Then 201 Created — body 에 결제 결과(receiptId) 가 실린다
  When 김지영이 동일 POST /v1/billing/charge (헤더 Idempotency-Key: pay-001) 를 재제출한다 (재시도)
  Then 200 OK — 첫 호출과 동일한 receiptId 를 반환한다 (새 청구 아님 — 멱등 재생)

  # 2. 결제 성공 webhook 도 중복 수신될 수 있으나 멱등 처리된다.
  When PG 가 같은 결제의 성공 webhook 을 두 번 전송한다 (PG 재전송)
  Then 두 번째 webhook 도 200 으로 수용되되 추가 청구를 만들지 않는다

  # 3. 사용자 관점 — 이중 청구가 일어나지 않았다.
  When 김지영이 GET /v1/billing/receipts 를 호출한다
  Then 200 OK — 영수증이 정확히 1건이다 (이중과금 없음 — 관찰 가능한 단일 청구)
  And GET /v1/billing/status 는 200, { status:"ACTIVE" }
```

> **검증 관점**: 동일 Idempotency-Key 재제출이 새 청구가 아니라 *같은 결과 재생*(2번째 200·동일 receiptId)이고, 최종 GET /v1/billing/receipts 가 1건임으로 이중과금 부재를 관찰. webhook 중복도 추가 청구를 만들지 않음.
> **통합 화이트박스 / 커버**: P4-02 · 13-05 / BILL-9 · BILL-4

#### D2-05: 환불 요청 → 접근 회수 (REFUND → CANCELED)

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
Scenario: 김지영이 환불을 요청하면 환불 처리 후 구독이 CANCELED되고 정책에 따라 접근이 회수된다
  Given 한울학원 ACADEMY 구독이 ACTIVE 다
  And 박교사는 게시 권한을 가진 정상 멤버이며 POST /v1/lectures 가 201 로 가능하다
  And GET /v1/billing/receipts 는 200, CHARGE 영수증 1건이 보인다

  # 1. 김지영이 환불을 요청하고 환불이 처리된다.
  When 김지영이 POST /v1/billing/refund { reason:"오결제" } 를 호출한다
  Then 202 Accepted — body.status="REFUND_PENDING"
  When PG 가 환불 완료 webhook 을 전송한다 (멱등)
  Then GET /v1/billing/receipts 는 200, CHARGE 1건 + REFUND 1건 (영수증은 append — CHARGE 가 사라지지 않음)
  And GET /v1/billing/status 는 200, { status:"CANCELED" }

  # 2. 접근 회수 — 정책에 따라 [즉시 차단]을 가정한다(아래 노트의 설계 결정).
  When 박교사가 POST /v1/lectures { title:"중1 사회", publish:true } 를 호출한다
  Then 402 Payment Required — body.code="ENTITLEMENT_REQUIRED" (환불 = 즉시 접근 회수)
  And GET /v1/lectures 는 200이되 환불 이후 게시한 강의는 없다 (관찰 가능한 부재)

  # 3. 기간말 유지 정책을 택했다면(대안) — current_period_end 까지는 201, 이후 402 (D2-09 형태).
```

> **검증 관점**: 환불이 영수증에 REFUND 로 *추가*되어 CHARGE 가 보존되고(append-only 가시), 구독이 CANCELED 로 가며 게시가 402 로 회수됨을 status·영수증 목록·`body.code` 로만 단언.
> **설계 결정(노트)**: 환불 시 **즉시 차단** vs **기간말 유지** 중 택1이 필요 — 본 여정은 즉시 차단을 단언하되, 기간말 유지 정책이면 D2-09 형태(current_period_end 까지 201 → 이후 402)로 분기. 제품 정책 확정 전까지 열린 결정.
> **통합 화이트박스 / 커버**: P4-05 · 13-04 / BILL-5 · BILL-6

#### D2-06: chargeback 분쟁 (지불거절)

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
Scenario: PG가 chargeback(지불거절)을 통지하면 구독이 정지되고 분쟁 해소 전까지 접근이 즉시 차단된다
  Given 한울학원 ACADEMY 구독이 ACTIVE 다
  And 박교사는 게시 권한을 가진 정상 멤버이며 POST /v1/lectures 가 201 로 가능하다

  # 1. PG 가 chargeback 통지 webhook 을 전송한다 (카드사 지불거절).
  When PG 가 chargeback 통지 webhook 을 전송한다 (재전송해도 멱등)
  Then GET /v1/billing/status 는 200, { status:"SUSPENDED" }, body.banner 에 분쟁 안내가 실린다
  And GET /v1/billing/receipts 는 200, CHARGE 1건 + CHARGEBACK 1건 (원장 append — 보정 항목 추가)

  # 2. 접근 즉시 차단 — chargeback 은 환불과 달리 즉시·강제 회수.
  When 박교사가 POST /v1/lectures { title:"중2 영어", publish:true } 를 호출한다
  Then 402 Payment Required — body.code="PAYMENT_DISPUTED" (분쟁 중 접근 차단)

  # 3. 분쟁 해소 전까지 유지 — 재시도해도 동일하게 막힌다.
  When 박교사가 잠시 후 동일 POST /v1/lectures 를 재호출한다
  Then 402 Payment Required — body.code="PAYMENT_DISPUTED" (분쟁 미해소 동안 차단 유지)
  And GET /v1/billing/status 는 여전히 { status:"SUSPENDED" } (분쟁 해소 전까지 변동 없음)
```

> **검증 관점**: chargeback 이 GRACE 유예 없이 즉시 SUSPENDED·402/`PAYMENT_DISPUTED` 로 차단되고(환불 D2-05 와 구분되는 강제성), 분쟁 해소 전까지 재시도해도 동일하게 막힘을 status·`body.code` 로만 단언. 영수증에 CHARGEBACK 보정 항목이 append 됨을 가시성으로 확인.
> **통합 화이트박스 / 커버**: P4-03 · P4-05 / BILL-5 · BILL-6

#### D2-07: ⚠️ 오과금 불만 → 운영자 정정 (support action)

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# ⚠️ 미구현 — operator-plane · support action 구현 후 활성화. 운영자는 테넌트 role이 아닌 별도 신원 평면(MFA).
Scenario: ⚠️ 잘못 청구된 org를 운영자가 support action으로 정정·환불하고 전건이 감사된다
  Given 한울학원이 실수로 잘못 청구돼 GET /v1/billing/receipts 에 오과금 CHARGE 1건이 보인다
  And 김지영(OWNER)이 오과금 불만을 접수했다
  And 운영자(operator, role='FINANCE')는 테넌트 role 이 아닌 별도 신원 평면(MFA)으로 로그인했다

  # 1. 운영자는 사유(why) 없이는 정정을 못 한다 — 컴플라이언스 가드.
  When 운영자가 사유 없이 POST /v1/support/billing/correct { org:"한울학원" } 를 호출한다
  Then 400 Bad Request — body.code="SUPPORT_REASON_REQUIRED" ("정정 사유(why)가 필수입니다")

  # 2. 운영자가 사유를 명시해 오과금을 정정(환불 + entitlement 보정)한다 — support action.
  When 운영자가 POST /v1/support/billing/correct { org:"한울학원", action:"REFUND", reason:"플랜 오적용 과금" } 를 호출한다
  Then 200 OK — body.code="BILLING_CORRECTED"
  And 김지영의 GET /v1/billing/receipts 는 200, 오과금 CHARGE + 정정 REFUND 가 함께 보인다 (원장 보정, CHARGE 보존)
  And GET /v1/billing/status 는 정정 후 올바른 { status } 로 관찰된다

  # 3. 모든 운영자 행위가 who/when/why 와 함께 전건 감사된다 (관찰: 후속 support audit API).
  When 운영자가 GET /v1/support/audit?action=support_action_billing_correct 를 호출한다
  Then 200 OK — body[0] 에 actorType="OPERATOR", reason="플랜 오적용 과금", who·occurredAt·targetOrg="한울학원" 이 보인다

  # 4. 일반 테넌트 토큰으로는 같은 정정이 불가능하다 — 운영자 평면 전용.
  When 김지영이 테넌트 토큰으로 POST /v1/support/billing/correct 를 호출한다
  Then 403 Forbidden — body.code="OPERATOR_PLANE_REQUIRED" (cross-tenant 정정은 운영자 평면 전용)
```

> **검증 관점**: 오과금 정정이 운영자 평면(MFA)에서만·사유(why) 필수로만 수행되고(사유 누락 400, 테넌트 토큰 403), 정정 결과가 영수증의 REFUND append 와 GET /v1/support/audit 의 who/when/why·OPERATOR 표식으로 관찰됨을 단언.
> **통합 화이트박스 / 커버**: 17-04 · 17-05 · 17-06 · 14-04 / SUPP-1 · OPER-1 · OPER-2 · AUD-2  ·  ⚠️ operator-plane 미구현, 구현 후 활성화

#### D2-08: 다운그레이드(예약) / 업그레이드(즉시) 한도 변화

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
Scenario: PRO→BASIC 다운그레이드는 기간말 예약(즉시 축소 아님), BASIC→PRO 업그레이드는 즉시 상향
  Given 한울학원 ACADEMY 구독이 ACTIVE, 현재 플랜 PRO 다
  And GET /v1/billing/status 는 200, { status:"ACTIVE", plan:"PRO", limits:{ dailyUploads:30 } }
  And current_period_end 는 미래(예: 2026-06-30)다

  # 1. 다운그레이드 PRO→BASIC — 기간말 예약, 현재 한도는 그대로 유지된다.
  When 김지영이 POST /v1/billing/change-plan { plan:"BASIC" } 를 호출한다
  Then 200 OK — body.code="DOWNGRADE_SCHEDULED", body.effectiveAt=current_period_end
  And GET /v1/billing/status 는 200이되 { plan:"PRO", limits:{ dailyUploads:30 } } 그대로다 (즉시 축소 아님)
  And body 에 scheduledPlan:"BASIC" 이 예약으로 보인다 (관찰 가능한 예약 상태)

  # 2. 기간말 도래 — 갱신 시점에 BASIC 한도로 하향된다.
  When current_period_end 가 지나 기간 갱신이 처리된다
  Then GET /v1/billing/status 는 200, { plan:"BASIC", limits:{ dailyUploads:6 } } (기간말에 비로소 하향)

  # 3. 업그레이드 BASIC→PRO — 기간말을 기다리지 않고 즉시 한도가 상향된다.
  When 김지영이 POST /v1/billing/change-plan { plan:"PRO" } 후 차액 결제를 완료한다
  Then 200 OK — body.code="UPGRADE_APPLIED" (즉시 적용)
  And GET /v1/billing/status 는 200, { plan:"PRO", limits:{ dailyUploads:30 } } (즉시 상향 — 기간말 대기 없음)
```

> **검증 관점**: 다운그레이드는 즉시 status/limits 가 변하지 않고 scheduledPlan 으로만 예약 노출(기간말에 비로소 하향), 업그레이드는 같은 change-plan 이 즉시 limits 상향 — 두 비대칭을 GET /v1/billing/status 의 plan·limits·effectiveAt 으로만 관찰.
> **통합 화이트박스 / 커버**: 13-03 · 13-04 / BILL-7

#### D2-09: 구독 취소 후 기간말 만료 (즉시 차단 아님)

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
Scenario: 취소는 즉시 차단이 아니라 current_period_end까지 접근을 유지하고 기간말에 EXPIRED된다
  Given 한울학원 ACADEMY 구독이 ACTIVE 다
  And 박교사는 게시 권한을 가진 정상 멤버이며 POST /v1/lectures 가 201 로 가능하다
  And current_period_end 는 미래(예: 2026-06-30)다

  # 1. 김지영이 구독을 취소한다 — 상태는 CANCELED지만 기간말까지 접근은 유지된다.
  When 김지영이 POST /v1/billing/cancel 을 호출한다
  Then 200 OK — body.code="CANCEL_SCHEDULED", body.accessUntil=current_period_end
  And GET /v1/billing/status 는 200, { status:"CANCELED", accessUntil:"2026-06-30" } (취소됐으나 접근 유지)

  # 2. 기간말 전 — 박교사 게시는 여전히 201 (사용자 기대치: 낸 만큼은 쓴다).
  When 박교사가 POST /v1/lectures { title:"중3 수학", publish:true } 를 호출한다
  Then 201 Created — current_period_end 전까지 접근 유지 (즉시 차단 아님)
  And GET /v1/lectures 는 200, 방금 게시한 강의를 포함한다

  # 3. 기간말 도래 → EXPIRED — 같은 게시가 이제 402로 막힌다.
  When current_period_end 가 지나 만료 배치가 구독을 EXPIRED 로 전이시킨다
  Then GET /v1/billing/status 는 200, { status:"EXPIRED" }
  When 박교사가 동일한 POST /v1/lectures { title:"중3 영어", publish:true } 를 호출한다
  Then 402 Payment Required — body.code="ENTITLEMENT_REQUIRED" (기간말 이후 회수)
```

> **검증 관점**: 취소 직후에도 게시가 201 로 유지되고(accessUntil 노출), current_period_end 경과 후에야 같은 호출이 402 로 전환됨 — "취소=즉시 차단 아님"이라는 사용자 기대치를 status·accessUntil·`body.code` 로만 단언. (chargeback D2-06 의 즉시 차단과 대비.)
> **통합 화이트박스 / 커버**: 13-04 · 17-08 / BILL-6 · AVAIL-1

### Domain D3 — 역할 거부 매트릭스·데이터 권리

#### D3-01: 역할별 권한초과 거부 매트릭스

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 호출자 역할 × 시도 작업 → 기대 응답. 권한 밖 작업은 deny-by-default(403/402)로 막히고,
#         거부 후에도 가시성(목록·조회)으로 "변경 미발생"이 관찰된다.
Scenario Outline: <역할>가 <작업>을 시도하면 권한 모델대로 거부된다
  Given <역할> 계정이 한울학원 컨텍스트로 로그인했다 (Bearer <토큰>, x-org-pk: 한울학원-pk)
  And 한울학원 ACADEMY 구독은 ACTIVE 다 (Gate B 통과 — 거부는 순수 권한 사유)

  # 1. 권한 밖 작업 호출
  When <역할>가 <메서드> <엔드포인트> 를 호출한다

  # 2. 기대 응답 — 명시적 allow가 없으므로 거부
  Then <기대상태> — body.code="<코드>"

  # 3. 거부 후 부재 관찰 — 작업이 실제로 일어나지 않았다
  And <검증GET> 는 변경이 반영되지 않은 상태를 그대로 보여준다 (관찰 가능한 부재)

  Examples:
    | 역할 | 토큰 | 메서드 | 엔드포인트 | 기대상태 | 코드 | 검증GET |
    | STUDENT(이수민) | soo-token | POST | /v1/lectures { publish:true } | 403 Forbidden | FORBIDDEN | GET /v1/lectures 에 새 강의 없음 |
    | STUDENT(이수민) | soo-token | POST | /v1/members/{교사B}/role | 403 Forbidden | FORBIDDEN | GET /v1/members/{교사B} 역할 불변 |
    | TEACHER(박교사) | park-token | POST | /v1/members/{교사B}/role { role:"DIRECTOR" } | 403 Forbidden | FORBIDDEN | GET /v1/members/{교사B} 역할 불변 |
    | TEACHER(박교사) | park-token | POST | /v1/delegations { grantee:"soo@x.com" } | 403 Forbidden | FORBIDDEN | GET /v1/delegations 목록 불변 |
    | MEMBER(일반멤버) | mbr-token | POST | /v1/billing/subscribe { plan:"ACADEMY_PRO" } | 403 Forbidden | FORBIDDEN | GET /v1/billing/status 불변 |
    | MEMBER(일반멤버) | mbr-token | DELETE | /v1/api-keys/{id} | 403 Forbidden | FORBIDDEN | GET /v1/api-keys/{id} 200 (여전히 유효) |
    | SERVICE(머신) | ak_live_ | POST | /v1/consents { types:["platform.marketing_email"] } | 403 Forbidden | SERVICE_CANNOT_CONSENT | GET /v1/me 동의상태 불변 (사람 전용 행위) |
    | DIRECTOR(김지영) | 김지영-token | POST | /v1/support/entitlements | 403 Forbidden | OPERATOR_PLANE_REQUIRED | GET /v1/billing/status 불변 (운영자 평면 전용) |
```

> **검증 관점**: 동일 엔드포인트라도 호출자 역할에 따라 거부됨을 status·`body.code`로만 단언하고, 직후 GET 으로 "변경 미발생"을 가시성으로 확인한다. SERVICE 계정의 사람 전용 행위(동의)·tenant role의 운영자 평면 작업도 같은 거부 매트릭스로 묶인다 — 명시적 allow 없으면 통과 0.
> **통합 화이트박스 / 커버**: 12-xx(인가) · P7-01 · 11-06 / RBAC-1 · RBAC-5 · ABAC-1 · TEN-2 · SEC-1

#### D3-02: BOLA 변형 — 타 org 멤버 ID로 작업

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: A학원 OWNER가 자기 권한(MANAGE_MEMBERS)을 B학원 멤버 ID에 행사하려 한다.
#         타 테넌트 멤버는 존재조차 비공개(404) — 권한이 있어도 org 경계를 넘지 못한다.
Scenario: A학원 OWNER가 B학원 멤버 ID로 역할변경·정지를 시도하면 org 경계가 막는다
  Given 김지영은 한울학원(A학원) OWNER/DIRECTOR로 로그인했다 (Bearer 김지영-token, x-org-pk: A학원-pk)
  And B학원(타 테넌트)에 멤버 {id: B멤버777} 가 존재하고, B학원에서 김지영은 아무 멤버십도 없다

  # 1. A 컨텍스트에서 B학원 멤버 ID로 역할변경 시도 — 존재 비공개
  When 김지영이 x-org-pk: A학원-pk 로 POST /v1/members/B멤버777/role { role:"TEACHER" } 를 호출한다
  Then 404 Not Found — body.code="NOT_FOUND" (타 org 멤버는 A 컨텍스트에 존재하지 않음 — 403조차 정보 누설)

  # 2. A 컨텍스트에서 B학원 멤버 정지 시도 — 동일하게 비공개
  When 김지영이 x-org-pk: A학원-pk 로 POST /v1/members/B멤버777/suspend 를 호출한다
  Then 404 Not Found — body.code="NOT_FOUND"

  # 3. org 헤더를 B로 바꿔 권한 상승 우회 시도 — B학원 멤버십이 없어 거부
  When 김지영이 같은 토큰에 x-org-pk: B학원-pk 를 실어 POST /v1/members/B멤버777/role { role:"TEACHER" } 를 호출한다
  Then 403 Forbidden — body.code="NOT_A_MEMBER" (B학원에 멤버십 없음 — 헤더 위조로 경계 못 넘음)

  # 4. 변이 미발생 관찰 — B 정당 멤버 입장에서 B멤버777 역할은 그대로다
  Given B학원 OWNER 가 로그인했다 (Bearer B오너-token, x-org-pk: B학원-pk)
  When B학원 OWNER가 GET /v1/members/B멤버777 를 호출한다
  Then 200 OK — 역할이 변경되지 않은 원래 값 그대로다 (타 org의 어떤 시도도 반영 0)
```

> **검증 관점**: 권한 보유자(A OWNER)라도 타 org 멤버 ID에는 404(존재 비공개), 헤더 위조로 B를 직접 노려도 멤버십 부재로 403. 변이 미발생은 B 정당 멤버의 GET 으로만 확인 — 호출자 역할이 아니라 org 경계가 변이를 차단함을 status·`body.code`·가시성으로 단언.
> **통합 화이트박스 / 커버**: P7-01 / TEN-2 · SEC-1 · ABAC-2 · RBAC-1

#### D3-03: ⚠️ 데이터 삭제 요청(PIPA 권리) — 본인정보 익명화·동의 이력 법적 보존

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# ⚠️ 미구현 — user_consent_event(§I, CON-1) 구현 후 활성화.
# 줄거리: 이수민이 개인정보 삭제권(PIPA)을 행사한다. 본인 식별정보는 익명화되어 GET 으로 사라지지만,
#         동의 이력(법적 증거)은 5년 보존 의무로 남는다 — 단, 그 보존은 컴플라이언스 내부라 본인에겐 안 보인다.
Scenario: ⚠️ 삭제 요청 후 본인정보는 비워지나 동의 이력의 법적 보존은 본인 조회로 노출되지 않는다
  Given 이수민(soo@x.com)은 로그인 상태이고 GET /v1/me 는 200, { email:"soo@x.com", displayName:"이수민" } 를 반환한다
  And 이수민은 가입 시 platform.terms_of_service·platform.privacy_policy 동의 이력이 있다

  # 1. 개인정보 삭제권 행사 (PIPA — 처리정지·파기 요청)
  When 이수민이 POST /v1/account/data-deletion { consent:"platform.withdrawal" } (Bearer soo-token) 를 호출한다
  Then 202 Accepted — body.code="DELETION_ACCEPTED" (익명화는 30일 배치 — 즉시 효과는 식별정보 처리정지)

  # 2. 본인정보 조회 — 식별정보가 비워진다(익명화)
  When 이수민이 GET /v1/me (Bearer soo-token) 를 호출한다
  Then 401 Unauthorized — body.code="ACCOUNT_DELETED" (세션 무효 — 식별정보 접근 경로 차단)
  When 운영자가 GET /v1/users/{이수민-publicId} (운영자 평면) 를 호출한다
  Then 200 OK — body.email=null, body.displayName="(삭제된 사용자)" (식별 컬럼 익명화 — 본인정보는 비고)

  # 3. 동의 이력 보존은 본인 권리 조회로 노출되지 않는다 (5년 법적 보존 = 컴플라이언스 내부)
  When (재가입한) 이수민이 GET /v1/me/consents 로 본인 동의 이력을 조회한다
  Then 200 OK — 과거 삭제된 계정의 동의 이력은 보이지 않는다 (새 계정 기준 — 보존본은 본인 가시 영역 밖)

  # 4. 동일 이메일 재가입은 가능하다 — 식별정보 익명화로 점유가 풀렸다(관찰 결과)
  When soo@x.com 으로 POST /v1/signup { email:"soo@x.com", displayName:"이수민", orgName:"새출발학원" } 를 호출한다
  Then 201 Created — 새 publicId·새 accessToken (과거 활동과 연결 0)
```

> **검증 관점**: 삭제권 행사가 (a) 본인 식별정보를 익명화(GET 에서 email=null), (b) 그럼에도 동의 이력은 5년 보존 의무로 남되 그 보존은 컴플라이언스 내부라 본인 조회(`/me/consents`)에는 안 보임, (c) 재가입 가능이라는 관찰 결과로만 익명화 완료를 확인. user_consent_event의 보존 row 자체는 black-box에서 단언 금지(append-only 보존은 화이트박스).
> **통합 화이트박스 / 커버**: P6-03 · P9-04 / USR-5 · CON-1 · CON-10 · AUTHN-6 · SEC-3  ·  ⚠️ 구현 후 활성화

#### D3-04: ⚠️ 동의 철회 후 처리 거부 — 마케팅 발송 즉시 차단

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# ⚠️ 미구현 — user_consent_event(§I, CON-1) 구현 후 활성화.
# 줄거리: 이수민이 마케팅 수신 동의를 철회하면(PIPA §37 즉시 효력) 이후 마케팅 발송이 차단된다.
Scenario: ⚠️ 마케팅 동의 철회가 즉시 효력을 가져 이후 발송이 CONSENT_REVOKED로 막힌다
  Given 이수민(soo@x.com)은 가입 시 platform.marketing_email 에 GRANTED 동의했다
  And GET /v1/me/consents 는 200, { "platform.marketing_email":"GRANTED" } 를 보여준다

  # 1. 발송 동의 상태에서는 마케팅 발송이 허용된다 (기준선)
  When 운영자가 POST /v1/marketing/send { to:"soo@x.com", channel:"email" } 를 호출한다
  Then 200 OK — body.code="QUEUED" (동의 상태라 발송 허용)

  # 2. 이수민이 동의를 철회한다 (PIPA §37 — 즉시 처리)
  When 이수민이 POST /v1/consents/revoke { types:["platform.marketing_email"] } (Bearer soo-token) 를 호출한다
  Then 200 OK — body.code="REVOKED"
  And GET /v1/me/consents 는 200, { "platform.marketing_email":"REVOKED" } (철회 즉시 반영)

  # 3. 철회 후 같은 발송 시도 — 즉시 차단 (스테일 캐시 아닌 최신 동의 재검증)
  When 운영자가 동일한 POST /v1/marketing/send { to:"soo@x.com", channel:"email" } 를 재호출한다
  Then 403 Forbidden — body.code="CONSENT_REVOKED" ("수신 동의가 철회된 대상")
  And 동일 발송 호출이 단계 1의 200(QUEUED) 에서 단계 3의 403 으로 전환됨이 관찰된다 (철회 즉시 효력)

  # 4. 재동의하면 다시 발송 가능 — 가역
  When 이수민이 POST /v1/consents { types:["platform.marketing_email"] } (Bearer soo-token) 를 호출한다
  Then 201 Created — body.code="GRANTED"
  When 운영자가 다시 POST /v1/marketing/send { to:"soo@x.com", channel:"email" } 를 호출한다
  Then 200 OK — body.code="QUEUED" (재동의 후 발송 복원)
```

> **검증 관점**: 같은 발송 호출이 동의 GRANTED → 200, 철회 REVOKED → 403/`CONSENT_REVOKED` 로 전환되고 재동의 후 다시 200 으로 복원되는 가역 흐름을 status·`body.code`·`/me/consents` 가시성으로만 단언. PIPA §37 "철회 즉시 처리"가 후속 발송 호출의 403 으로 관찰된다.
> **통합 화이트박스 / 커버**: P6-03 · 15-05 / CON-1 · CON-5 · CON-6 · SEC-3  ·  ⚠️ 구현 후 활성화

#### D3-05: ⚠️ 14세 미만 법정대리인 미동의 차단

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# ⚠️ 미구현 — user_consent_event(§I, CON-3) 구현 후 활성화.
# 줄거리: 14세 미만은 법정대리인 동의(PIPA §22③) 없이는 가입이 차단된다.
Scenario: ⚠️ 14세 미만 가입은 법정대리인 동의가 없으면 차단되고, 동의 첨부 시 통과한다
  # 1. 14세 미만이 대리인 동의 없이 가입 시도 — 차단
  When 만 12세 이용자가 POST /v1/signup { email:"kid@x.com", birthDate:"2014-03-01", consents:["platform.terms_of_service","platform.privacy_policy"] } 를 호출한다
  Then 400 Bad Request — body.code="GUARDIAN_CONSENT_REQUIRED" ("법정대리인 동의가 필요합니다")
  And POST /v1/auth/login { email:"kid@x.com" } 는 401 (계정이 생성되지 않음 — 관찰 가능한 부재)

  # 2. 생년월일이 14세 이상이면 대리인 동의 없이 통과 (경계값)
  When 만 14세 이용자가 POST /v1/signup { email:"teen@x.com", birthDate:"2012-03-01", consents:["platform.terms_of_service","platform.privacy_policy"] } 를 호출한다
  Then 201 Created — 14세 이상은 본인 동의로 충분

  # 3. 14세 미만이라도 법정대리인 동의를 첨부하면 가입이 열린다
  When 만 12세 이용자가 POST /v1/signup { email:"kid@x.com", birthDate:"2014-03-01", consents:["platform.terms_of_service","platform.privacy_policy","platform.under14_guardian"], guardian:{ name:"...", relation:"부" } } 를 호출한다
  Then 201 Created — body 에 신규 accessToken (대리인 동의 충족)
  And GET /v1/me (Bearer kid-token) 는 200 — 정상 계정

  # 4. 동의 누락이 입력 검증 단계에서 막힘 — 부분 생성 없음(fail-closed)
  When 만 10세 이용자가 platform.under14_guardian 없이 다시 가입을 시도한다
  Then 400 Bad Request — body.code="GUARDIAN_CONSENT_REQUIRED" (재시도해도 동일 — 우회 없음)
```

> **검증 관점**: 생년월일로 14세 미만이면 `platform.under14_guardian` 동의가 없는 한 400/`GUARDIAN_CONSENT_REQUIRED`, 대리인 동의 첨부 시에만 201. 14세 경계값은 본인 동의로 통과. 차단 시 계정 미생성을 후속 login 401 로 확인(부분 생성 없음 — fail-closed).
> **통합 화이트박스 / 커버**: 15-01 · P6-01 / CON-1 · CON-3 · USR-7  ·  ⚠️ 구현 후 활성화

#### D3-06: ⚠️ 계정 정지 이의제기 → 복원 (가역)

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# ⚠️ 운영자 복원(operator-plane support action)은 미구현 — 구현 후 활성화.
# 줄거리: 정지된 사용자는 모든 API 가 즉시 막히고, 이의제기→운영자 검토→복원 후 정상 복귀(SUSPENDED는 가역).
Scenario: ⚠️ SUSPENDED 사용자가 전 API 차단 → 이의제기 → 운영자 복원 → 정상 복귀
  Given 이수민(soo@x.com)은 academy·market 양 서비스 멤버이고 토큰이 유효하다
  And 운영자가 방금 이수민을 SUSPENDED 로 전이시켰다 (정책 위반 의심)

  # 1. 정지 즉시 효과 — sensitive write 가 stale 토큰으로도 DB 재검증으로 막힌다
  When 이수민이 POST /v1/lectures (x-org-pk: 한울학원-pk, Bearer soo-token) 를 호출한다
  Then 403 Forbidden — body.code="ACCOUNT_SUSPENDED"
  When 이수민이 GET /v1/lectures (x-org-pk: B학원-pk, Bearer soo-token) 를 호출한다
  Then 403 Forbidden — body.code="ACCOUNT_SUSPENDED" (서비스 무관 — academy·market 모두 동일 차단)

  # 2. 이의제기는 허용된 좁은 경로로만 가능 (정지 중에도 열린 채널)
  When 이수민이 POST /v1/account/appeal { reason:"오탐으로 보입니다" } (Bearer soo-token) 를 호출한다
  Then 202 Accepted — body.code="APPEAL_SUBMITTED", body.id 발급

  # 3. 운영자가 검토 후 복원한다 (operator-plane support action — MFA)
  When 운영자가 POST /v1/support/users/{이수민-publicId}/restore { appeal:"{id}" } 를 호출한다
  Then 200 OK — body.code="RESTORED", body.status="ACTIVE"

  # 4. 복원 후 정상 복귀 — 같은 호출이 다시 통과 (SUSPENDED는 가역, DELETED와 달리 익명화 없음)
  When 이수민이 토큰 갱신 후 GET /v1/lectures (x-org-pk: 한울학원-pk, Bearer soo-token) 를 호출한다
  Then 200 OK
  And 동일 호출이 단계 1의 403(ACCOUNT_SUSPENDED) 에서 단계 4의 200 으로 전환됨이 관찰된다 (가역 복원)
```

> **검증 관점**: 정지 사용자의 전 서비스 호출이 403/`ACCOUNT_SUSPENDED`(서비스 무관), 이의제기 채널만 좁게 열림, 운영자 복원 후 같은 호출이 200 으로 전환되는 가역성을 status·`body.code`로만 단언. DELETED(비가역)와 대비되는 SUSPENDED 가역성이 핵심 관찰.
> **통합 화이트박스 / 커버**: 11-06 / AUTHN-6 · USR-6 · OPER-2 · SUPP-1  ·  ⚠️ 운영자 복원 구현 후 활성화

#### D3-07: 잘못 부여된 위임 분쟁 → 회수 후 차단

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 김지영이 실수로 박교사에게 결제 열람(VIEW_BILLING) capability 를 부여했다.
#         분쟁 제기 후 정정(회수)하면, 박교사가 stale 토큰으로 행사해도 sensitive write DB 재검증으로 즉시 403.
#         (C-04는 정상 위임 lifecycle, 본 여정은 '실수 부여→분쟁→정정' 관점)
Scenario: 실수로 부여한 capability 를 분쟁 후 회수하면 행사 시도가 즉시 차단된다
  Given 김지영은 한울학원 OWNER/DIRECTOR로 로그인했다 (Bearer 김지영-token, x-org-pk: 한울)
  And 박교사(park@x.com)는 한울학원 TEACHER다

  # 1. 실수 부여 — 김지영이 박교사에게 ACADEMY.VIEW_BILLING 을 잘못 위임
  When 김지영이 POST /v1/delegations { grantee:"park@x.com", capability:"ACADEMY.VIEW_BILLING" } 를 호출한다
  Then 201 Created — body { id:<delegationId>, capability:"ACADEMY.VIEW_BILLING", status:"ACTIVE" }
  When 박교사가 토큰 갱신 후 GET /v1/billing/status (x-org-pk: 한울) 를 호출한다
  Then 200 OK (잘못 부여된 위임이 일시적으로 행사됨 — 정정 대상)

  # 2. 분쟁 제기 — 박교사가 부여 사실에 이의(또는 김지영이 오부여를 인지)
  When 김지영이 GET /v1/delegations 를 호출한다
  Then 200 OK — 목록에 {delegationId} 가 ACADEMY.VIEW_BILLING / ACTIVE 로 보인다 (오부여 확인)

  # 3. 정정 — 김지영이 잘못된 위임을 회수
  When 김지영이 DELETE /v1/delegations/{delegationId} 를 호출한다
  Then 204 No Content
  And GET /v1/delegations 는 200, {delegationId} 가 더 이상 ACTIVE 로 보이지 않는다

  # 4. 회수 후 stale 토큰으로 재행사 시도 — DB 재검증으로 즉시 403
  When 박교사가 *회수 미반영 stale 토큰*으로 다시 GET /v1/billing/status (x-org-pk: 한울) 를 호출한다
  Then 403 Forbidden — body.code="DELEGATION_REVOKED"
  And 응답 헤더 X-Perm-Version 이 클라이언트 토큰 값보다 높다 (불일치 → 재발급 신호)
  And 동일 호출이 단계 1의 200 에서 단계 4의 403 으로 전환됨이 관찰된다 (오부여 정정이 stale 토큰에도 즉시 효력)
```

> **검증 관점**: '실수 부여→분쟁→정정' 관점에서, 회수된 capability 의 행사가 stale 토큰이라도 200→403/`DELEGATION_REVOKED` 로 전환되고 `X-Perm-Version` 불일치가 재발급을 유도함을 단언. C-04(정상 lifecycle)와 달리 회수 사유가 '오부여 정정'이며 sensitive read/write 가 DB 재검증으로 즉시 닫힘을 강조.
> **통합 화이트박스 / 커버**: P3-02 · P8-01 / REBAC-1 · REBAC-3 · RBAC-5 · NFR-1

#### D3-08: 잘못된 입력·fail-closed — deny-by-default

```gherkin
# Black-box E2E: API 경계에서만 관찰. DB row 단언 없음. 상태는 후속 API로 검증.
# 줄거리: 누락 claim·위조 org 헤더·없는 리소스 등 비정상 입력은 401/400/404 로 닫히고,
#         어떤 경우에도 '허용으로 새지(통과)' 않는다(fail-closed).
Scenario Outline: <비정상 입력>은 deny-by-default 로 닫히고 허용으로 새지 않는다
  Given 한울학원 ACADEMY 구독은 ACTIVE 다 (거부는 입력 정합성 사유)

  # 1. 비정상 입력으로 호출
  When 클라이언트가 <조건>으로 GET /v1/lectures 를 호출한다

  # 2. 기대 거부 응답 — 절대 200(허용)이 아님
  Then <기대상태> — body.code="<코드>"
  And 응답은 어떤 경우에도 200 OK 가 아니다 (deny-by-default — 허용 누수 0)

  Examples:
    | 조건 | 기대상태 | 코드 |
    | Authorization 헤더 없음 | 401 Unauthorized | UNAUTHENTICATED |
    | 만료·서명불일치 Bearer 토큰 | 401 Unauthorized | TOKEN_INVALID |
    | 필수 claim(firebase_uid) 누락 토큰 | 401 Unauthorized | CLAIMS_MALFORMED |
    | x-org-pk 헤더 없음(다중 멤버십) | 400 Bad Request | ORG_HEADER_REQUIRED |
    | 위조·미소속 x-org-pk 헤더 | 403 Forbidden | NOT_A_MEMBER |
    | 형식 위반 x-org-pk(비-ULID) | 400 Bad Request | INVALID_ORG_ID |
    | 존재하지 않는 리소스 ID(GET /v1/lectures/{없는id}) | 404 Not Found | NOT_FOUND |
    | DB 재검증 일시 오류 시 | 503 Service Unavailable | UPSTREAM_UNAVAILABLE |
```

> **검증 관점**: 인증·헤더·리소스·업스트림 오류 전 분기가 각각 401/400/403/404/503 으로 닫히며 단 한 경우도 200(허용)으로 새지 않음을 status·`body.code`로 단언. fail-closed(§E.6) — 파싱 실패·perm_version 불일치는 401, 재검증 오류는 503(허용 아님), 명시적 allow만 통과.
> **통합 화이트박스 / 커버**: P7-01 · 11-06 / SEC-1 · TEN-2 · AUTHN-8 · ABAC-7

---

## 커버리지 — e2e는 통합 검증

Part C는 *가치 흐름*(happy path), Part D는 *예외·불만·생명주기*(unhappy path)를 다룬다. 각 여정은 화이트박스 시나리오 여러 개를 사용자 관점으로 묶는다(위 "통합 화이트박스" 참조). 요구 ID 단위 전수 추적은 [[bdd-scenarios]]의 **요구사항 추적 매트릭스**가 권위이고, 이 문서는 그 위에 **관찰 가능한 행위 계약**을 얹는다.

| 여정 | 핵심 platform 행위 | 구현 |
|---|---|---|
| C-01 가입→유료 게시 | 3-gate · billing projection · perm_version · BOLA | ✅ |
| C-02 강사 이전 | consent 게이트 · org_pk 데이터 경계 | 🟡 (consent 미구현) |
| C-03 탈퇴→재가입 | 전 서비스 즉시 차단 · email ACTIVE-unique | ✅ |
| C-04 위임 가치흐름 | 위임 행사·회수 · sensitive write DB 재검증 | ✅ |
| C-05 멀티 워크스페이스 | 단일 신원 · org별 권한·가시성 격리 | ✅ |
| C-06 break-glass | 운영자 평면 · support action · 만료 자동회수 | ⚠️ |
| C-07 api_key B2B | 머신 3-gate 동등 · IP·구독·revoke·rotation | ⚠️ |
| **D1** 멤버십·초대 생명주기 (8) | 강퇴 · 초대 만료/취소/재발송 · OWNER lockout · 동시성 · 재활성화 | ✅ |
| **D2** 결제·금전 불만 (9) | 결제실패 전이 · 재결제 · 환불 · chargeback · 이중과금 · 다운그레이드 · 오과금정정 | ✅ / ⚠️ |
| **D3** 역할거부·데이터권리 (8) | 역할 거부 매트릭스 · BOLA 변형 · 삭제권 · 동의철회 · 14세 · 정지이의 · fail-closed | ✅ / ⚠️ |
