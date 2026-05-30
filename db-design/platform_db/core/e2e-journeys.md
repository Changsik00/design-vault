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

## 커버리지 — e2e는 통합 검증

각 여정은 화이트박스 시나리오 여러 개를 *사용자 가치 흐름*으로 묶는다(위 "통합 화이트박스" 참조). 요구 ID 단위 전수 추적은 [[bdd-scenarios]]의 **요구사항 추적 매트릭스**가 권위이고, 이 문서는 그 위에 **관찰 가능한 행위 계약**을 얹는다.

| 여정 | 핵심 platform 행위 | 구현 |
|---|---|---|
| C-01 가입→유료 게시 | 3-gate · billing projection · perm_version · BOLA | ✅ |
| C-02 강사 이전 | consent 게이트 · org_pk 데이터 경계 | 🟡 (consent 미구현) |
| C-03 탈퇴→재가입 | 전 서비스 즉시 차단 · email ACTIVE-unique | ✅ |
| C-04 위임 가치흐름 | 위임 행사·회수 · sensitive write DB 재검증 | ✅ |
| C-05 멀티 워크스페이스 | 단일 신원 · org별 권한·가시성 격리 | ✅ |
| C-06 break-glass | 운영자 평면 · support action · 만료 자동회수 | ⚠️ |
| C-07 api_key B2B | 머신 3-gate 동등 · IP·구독·revoke·rotation | ⚠️ |
