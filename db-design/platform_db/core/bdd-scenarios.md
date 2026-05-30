---
type: core
aliases:
  - BDD 시나리오
  - bdd-scenarios
  - 통합 테스트 시나리오
tags:
  - platform-db
  - core
  - bdd
  - test-scenarios
---

# platform_db — BDD 시나리오

> 분리: [[requirements]] §B/C에서 2026-05-30 독립 파일로 분리  
> 목적: platform_db 도메인 행위 시나리오 — e2e / 통합 테스트 기반  
> 상위: [[requirements]] · [[schema-reference]]
>
> **현행 스키마 반영**: `membership.platform_role` + `service_membership.role_code` 2-layer 구조 적용  
> **현행 스키마**: `organization.type`에서 `ACADEMY` 제거 → `COMPANY/TEAM/PERSONAL`  
> **현행 스키마**: `delegation_grant.capability` 네임스페이스 `ACADEMY.*` 적용  
> Gherkin 형식 (English keywords, Korean content). 통합 테스트: vitest + supertest

**상태 범례**: ⚠️ 구현 후 활성화

---

## Domain 1: Identity — 조직 & 사용자

### P1-01: 신규 학원 onboarding (단일 트랜잭션)

```gherkin
Scenario: 신규 학원장 가입 시 platform_db 원자적 생성
  Given identity_user에 firebase_uid='fb_new_001'인 row가 없다
  And organization에 slug='hanwool'인 row가 없다

  When 가입 API가 단일 트랜잭션으로 실행된다

  Then 다음 row들이 동일 트랜잭션 내 INSERT된다:
    | 테이블           | 주요 값                                                              |
    | identity_user    | firebase_uid='fb_new_001', type='HUMAN', status='ACTIVE'            |
    | user_profile     | display_name='김지영', locale='ko'                                  |
    | organization     | slug='hanwool', type='COMPANY', status='ACTIVE'                     |
    | membership       | platform_role='OWNER', status='ACTIVE'                              |
    | service_membership | service='ACADEMY', role_code='DIRECTOR', status='ACTIVE'          |
    | org_entitlement  | service='ACADEMY', status='ACTIVE', source='FREE'                   |
  And organization.perm_version=1
  And identity_user.perm_version=1
```

> `organization.type='COMPANY'` — 현행 ACADEMY 타입 제거. 서비스 종류는 `org_entitlement.service='ACADEMY'`로 결정.

### P1-02: 멀티 워크스페이스 — 1 user, N org

```gherkin
Scenario: 기존 사용자가 두 번째 학원에 초대 수락
  Given identity_user.pk=10 (최정훈)이 org_pk=1의 TEACHER다
    (membership.platform_role='MEMBER', service_membership.role_code='TEACHER', service='ACADEMY')
  And membership_invite에 token='abc123', org_pk=2, role_code='TEACHER', status='PENDING'이 존재한다
  And expires_at이 아직 유효하다

  When acceptMembershipInvite(token='abc123', userPk=10)가 호출된다

  Then membership에 (user_pk=10, org_pk=2, platform_role='MEMBER', status='ACTIVE') INSERT
  And service_membership에 (user_pk=10, org_pk=2, service='ACADEMY', role_code='TEACHER') INSERT
  And membership_invite.status='ACCEPTED' UPDATE
  And identity_user에 새 row가 생성되지 않는다 (기존 user_pk 재사용)
```

### P1-03: 만료된 초대 토큰 차단

```gherkin
Scenario: expires_at 지난 토큰으로 수락 시도
  Given membership_invite.expires_at이 1시간 전에 지났다

  When acceptMembershipInvite(token)가 호출된다

  Then ForbiddenException 발생
  And membership_invite.status가 'EXPIRED'로 UPDATE된다
  And membership에 새 row가 INSERT되지 않는다
  And service_membership에 새 row가 INSERT되지 않는다
```

---

## Domain 2: Membership & 권한 동기화

### P2-01: 멤버십 회수 → perm_version bump

```gherkin
Scenario: DIRECTOR(service_membership)가 TEACHER 멤버십을 정지시킨다
  Given membership (user_pk=20, org_pk=1, status='ACTIVE')가 존재한다
  And organization.perm_version=5다

  When membership.status='SUSPENDED' UPDATE + bumpPermVersion(orgPk=1)이 호출된다

  Then membership.status='SUSPENDED'
  And organization.perm_version=6
  And audit_log에 (actor_pk=director_pk, action='revoke_membership', result='ALLOW') INSERT
```

### P2-02: Gate A — 정지된 멤버십 차단

```gherkin
Scenario: SUSPENDED 멤버의 API 호출
  Given membership (user_pk=20, org_pk=1, status='SUSPENDED')

  When checkGateA(userPk=20, orgPk=1)가 호출된다

  Then false 반환 (Gate A 실패)
  And 이 결과로 API 계층에서 403 Forbidden 응답
```

### P2-03: 서비스 역할 변경 → perm_version bump

```gherkin
Scenario: ACADEMY 서비스 역할 TEACHER → DIRECTOR 승격
  Given service_membership (user_pk=30, org_pk=1, service='ACADEMY', role_code='TEACHER')
  And organization.perm_version=7

  When service_membership.role_code='DIRECTOR' UPDATE + bumpPermVersion(orgPk=1) 단일 트랜잭션

  Then service_membership.role_code='DIRECTOR'
  And organization.perm_version=8
  And audit_log에 (actor_pk=director_pk, action='role_change', result='ALLOW') INSERT
```

> `membership.platform_role`은 OWNER/MEMBER 2-tier만 존재. 서비스 역할 승격은 `service_membership.role_code` 변경.

### P2-04: 마지막 DIRECTOR lockout 방지 (RBAC-6)

```gherkin
Scenario: 학원 내 마지막 DIRECTOR 권한 회수 차단
  Given A학원(org_pk=1)에 service_membership WHERE service='ACADEMY' AND role_code='DIRECTOR'가
        user_pk=10 (김지영) 1건만 존재한다

  When 김지영이 본인의 service_membership.role_code를 'TEACHER'로 변경 시도한다

  Then 400 Bad Request ("마지막 DIRECTOR의 권한은 회수할 수 없습니다. 다른 사용자에게 DIRECTOR 권한 부여 후 재시도.")
  And service_membership 변경 없음
  And audit_log에 (action='attempted_lockout', result='DENY') INSERT

Scenario: DIRECTOR가 2명 이상이면 1명 회수 허용
  Given A학원(org_pk=1)에 DIRECTOR = user_pk=10 + user_pk=11 2명

  When user_pk=11이 user_pk=10의 role_code를 'TEACHER'로 변경한다

  Then service_membership(user_pk=10).role_code='TEACHER'
  And org_pk=1의 DIRECTOR는 user_pk=11 1명으로 정상 유지
```

---

## Domain 3: Delegation Grant (ReBAC)

### P3-01: capability 위임 — 네임스페이스 CHECK constraint

```gherkin
Scenario: 허용된 capability로 delegation_grant INSERT
  Given DIRECTOR (grantor_pk=10)가 TEACHER (grantee_pk=20)에게 권한 위임

  When delegation_grant INSERT:
    capability='ACADEMY.PUBLISH_VIDEO', org_pk=1, status='ACTIVE'

  Then INSERT 성공
  And delegation_grant.status='ACTIVE'

Scenario: 허용되지 않은 capability 거부
  When delegation_grant INSERT: capability='ADMIN_ALL'

  Then MySQL CHECK constraint 위반으로 INSERT 실패
  And delegation_grant row가 생성되지 않는다

Scenario: 구 네임스페이스(prefix 없음) 거부
  When delegation_grant INSERT: capability='PUBLISH_VIDEO'

  Then MySQL CHECK constraint 위반으로 INSERT 실패 (ACADEMY. prefix 필수)
```

### P3-02: 위임 회수 → DB 재검증에서 차단

```gherkin
Scenario: REVOKE 후 stale JWT로 sensitive write 시도
  Given delegation_grant (grantee_pk=20, capability='ACADEMY.PUBLISH_VIDEO', status='ACTIVE')였다
  And 방금 status='REVOKED'로 UPDATE됐다
  And grantee의 JWT claims는 아직 stale(TTL 내)

  When grantee가 sensitive write API(@VerifyOnDb)를 호출한다

  Then DB에서 delegation_grant.status='REVOKED' 재확인
  And 403 Forbidden 반환
  And audit_log에 (result='DENY', action='publish_video') INSERT
```

### P3-03: expires_at 지난 grant 자동 무력화

```gherkin
Scenario: 만료 기간 지난 delegation_grant
  Given delegation_grant.expires_at=어제, status='ACTIVE'

  When getDelegationGrants(granteePk=20, orgPk=1)가 호출된다

  Then expires_at < NOW() 인 grant는 결과에서 제외된다
  And 비즈니스 로직에서 해당 capability가 없는 것으로 평가된다
```

---

## Domain 4: Billing — 결제 & 권한

### P4-01: 구독 결제 → 단일 트랜잭션 완결

```gherkin
Scenario: TOSS 결제 완료 webhook 처리 → org_entitlement 활성화
  Given org_subscription.status='TRIALING', org_entitlement.status='ACTIVE' (FREE)
  And pg_webhook_event에 (pg_provider='TOSS', event_id='evt_001')이 없다

  When 결제 성공 처리 트랜잭션이 실행된다

  Then 단일 트랜잭션 내에서:
    | 처리                  | 내용 |
    | payment_ledger INSERT | type='CHARGE', amount_minor=990000, idempotency_key='inv_001', status='SUCCEEDED' |
    | org_subscription UPDATE | status='ACTIVE', current_period_end=+30일 |
    | org_entitlement UPSERT | status='ACTIVE', source='SUBSCRIPTION', feature_limits={...} |
    | organization UPDATE   | perm_version=perm_version+1 |
    | outbox_event INSERT   | event_type='subscription.activated' |
  And pg_webhook_event INSERT: (pg_provider='TOSS', event_id='evt_001', status='PROCESSED')
```

### P4-02: 중복 webhook 멱등 처리

```gherkin
Scenario: 동일 webhook event_id 재수신
  Given pg_webhook_event에 (pg_provider='TOSS', event_id='evt_001', status='PROCESSED')가 이미 존재

  When 동일 event_id의 webhook이 다시 수신된다

  Then pg_webhook_event INSERT가 UNIQUE constraint 위반으로 실패 (또는 SKIPPED 처리)
  And payment_ledger에 중복 INSERT 없음
  And org_entitlement 중복 변경 없음
```

### P4-03: Gate B — EXPIRED entitlement 차단

```gherkin
Scenario: 만료된 org_entitlement로 서비스 접근 시도
  Given org_entitlement (org_pk=1, service='ACADEMY', status='EXPIRED')

  When checkGateB(orgPk=1, service='ACADEMY')가 호출된다

  Then false 반환 (Gate B 실패)
  And 402 Payment Required 반환
```

### P4-04: GRACE 기간 중 서비스 유지

```gherkin
Scenario: 결제 실패 후 grace 기간 내 접근
  Given org_entitlement (status='GRACE', grace_until=내일)

  When checkGateB(orgPk=1, service='ACADEMY')가 호출된다

  Then true 반환 (Gate B 통과 — GRACE도 허용)
  And 학원장 화면에 "결제 실패 배너" 표시 (앱 레이어 처리)
```

### P4-05: 환불 — append-only 원장

```gherkin
Scenario: 결제 취소 → REFUND 기록
  Given payment_ledger에 (type='CHARGE', amount_minor=990000, status='SUCCEEDED') row 존재

  When 환불 처리가 실행된다

  Then payment_ledger에 신규 row INSERT:
    type='REFUND', amount_minor=-990000, idempotency_key='ref_001', status='SUCCEEDED'
  And 기존 CHARGE row는 수정되지 않는다 (append-only)
```

---

## Domain 5: 감사 로그

### P5-01: 모든 권한 결정 기록

```gherkin
Scenario: ALLOW 결정 감사 기록
  Given Gate A/B/C 모두 통과한 API 호출

  Then audit_log에 INSERT:
    actor_type='HUMAN', actor_pk=user_pk, org_pk=1,
    action='publish_video', result='ALLOW', ip=<클라이언트 IP>

Scenario: DENY 결정 감사 기록
  Given Gate A 실패(membership SUSPENDED)

  Then audit_log에 INSERT:
    result='DENY', action='publish_video', ip=<클라이언트 IP>
  And 403 Forbidden 반환
```

### P5-02: audit_log 불변성 — 수정 불가

```gherkin
Scenario: audit_log row UPDATE 시도
  Given audit_log.pk=9999 row 존재

  When UPDATE audit_log SET result='ALLOW' WHERE pk=9999 실행

  Then 앱 레이어에서 UPDATE 차단 (DB 계정 권한 없음)
  And row가 변경되지 않는다
```

### P5-03: 월 파티셔닝 — 조회 성능

```gherkin
Scenario: 특정 월 감사 로그 조회
  Given audit_log에 2026-05 파티션에 1만 건 데이터 존재

  When SELECT * FROM audit_log WHERE org_pk=1 AND created_at BETWEEN '2026-05-01' AND '2026-05-31'

  Then p202605 파티션만 스캔 (Explain: partitions=p202605)
  And 전체 테이블 스캔 없음
```

---

## Domain 6: Consent — PIPA 동의 (미구현)

> ⚠️ 아래 시나리오는 구현 후 통합 테스트로 전환.

### P6-01: 가입 시 필수 동의 누락 차단

```gherkin
Scenario: PIPA 필수 동의 누락
  Given 김지영이 가입 페이지에서 개인정보 처리방침을 동의하지 않는다

  When 가입 API가 호출된다

  Then 400 Bad Request 반환 ("필수 약관에 동의해 주세요")
  And identity_user row INSERT 없음
  And user_consent_event row INSERT 없음
```

### P6-02: 정상 가입 시 동의 이벤트 기록

```gherkin
Scenario: 모든 필수 약관 동의 후 가입
  When 모든 필수 약관 동의 + 가입 API 호출

  Then user_consent_event에 다음 row들 INSERT:
    | consent_type                    | action  | version    |
    | platform.terms_of_service       | GRANTED | 2026-05-01 |
    | platform.privacy_policy         | GRANTED | 2026-05-01 |
    | platform.third_party_firebase   | GRANTED | 2026-05-01 |
  And 모든 row에 ip, user_agent 기록
  And row는 이후 UPDATE·DELETE 불가 (append-only)
```

### P6-03: 마케팅 동의 철회 — append-only

```gherkin
Scenario: 마케팅 수신 동의 철회
  Given user_consent_event에 (user_pk=10, consent_type='platform.marketing_email', action='GRANTED') 존재

  When 철회 API가 호출된다

  Then user_consent_event에 신규 row INSERT:
    (user_pk=10, consent_type='platform.marketing_email', action='REVOKED')
  And 기존 GRANTED row는 수정되지 않는다 (append-only)
  And 가장 최신 action='REVOKED' → 현재 마케팅 미동의 상태로 판단
```

### P6-04: PIPA §17 제3자 제공 — Firebase Auth 국외이전

```gherkin
Scenario: Firebase Auth 국외이전 동의 기록
  Given 가입 시 Firebase Auth 국외이전 동의 항목이 필수로 표시된다

  When 가입이 완료된다

  Then user_consent_event에 INSERT:
    consent_type='platform.third_party_firebase',
    action='GRANTED',
    meta_json={
      "recipient": "Google Firebase",
      "purpose": "이메일/비밀번호 인증",
      "items": ["email", "password_hash"],
      "retention": "서비스 탈퇴 후 30일"
    }
```

---

## Domain 7: 멀티테넌시 & BOLA 방어

### P7-01: BOLA — 타 테넌트 리소스 접근 차단

```gherkin
Scenario: 다른 org의 리소스 ID로 접근 시도
  Given identity_user.pk=10 (김지영)이 org_pk=1 소속이다
  And org_pk=2에 lecture.pk=999 (다른 학원 강의)가 존재한다

  When org_pk=1 컨텍스트에서 lecture.pk=999 조회를 시도한다

  Then 쿼리에서 WHERE org_pk=1 AND pk=999 → row 없음
  And 404 NotFoundException 반환 (존재 여부 비공개)
  And audit_log에 (result='DENY', action='get_lecture') INSERT
```

### P7-02: 멀티테넌시 격리 — Qdrant org_id 필터 강제

```gherkin
Scenario: Qdrant 벡터 검색에서 org 격리
  Given Qdrant collection 'academy_lectures'에 org_id=1 포인트 1000개, org_id=2 포인트 500개가 혼재한다

  When org_pk=1 컨텍스트에서 RAG 검색을 수행한다

  Then 검색 요청에 filter.must=[{key:'org_id', match:{value:1}}] 가 포함된다
  And 반환된 결과에 org_id=2 포인트가 없다
  And org_id=2 포인트는 org_pk=1 검색 결과에 절대 포함되지 않는다
```

### P7-03: Neo4j 멀티홉 경로 전체 org_id 강제

```gherkin
Scenario: 멀티홉 그래프 쿼리에서 경로 전체 org_id 강제
  Given Neo4j에 org_id=1 노드와 org_id=2 노드가 관계로 연결된 케이스가 있다 (비정상적 데이터)

  When org_pk=1 컨텍스트에서 개념 그래프를 조회한다

  Then Cypher 쿼리에 MATCH (c:Concept {orgId: $orgId}) ... MATCH (c2:Concept {orgId: $orgId}) 가 포함된다
  And 양 끝 노드 모두 org_id=1 인 경우만 반환된다
  And org_id=2 노드를 통과하는 경로는 결과에서 제외된다
```

---

## Domain 8: perm_version 전파 & Break-glass

### P8-01: perm_version 불일치 → 클라이언트 캐시 무효화

```gherkin
Scenario: 권한 변경 후 클라이언트 캐시 갱신 강제
  Given 클라이언트가 perm_version=5로 캐시된 권한 스냅샷을 가지고 있다
  And 서버에서 service_membership.role_code가 변경되어 organization.perm_version=6으로 bump됐다

  When 클라이언트가 다음 API를 호출한다

  Then API 응답 헤더에 X-Perm-Version: 6이 포함된다
  And 클라이언트가 캐시의 perm_version=5 와 헤더의 6이 다름을 감지한다
  And 클라이언트가 GET /me/permissions를 재요청한다
  And 새 스냅샷의 perm_version=6으로 캐시가 갱신된다
```

### P8-02: Break-glass 긴급 접근 — 감사 기록 필수

```gherkin
Scenario: 운영자 긴급 접근 (Break-glass)
  Given 운영자(admin)가 platform.break_glass capability를 부여받았다
  And 승인자(approver_pk)가 break_glass 요청을 승인했다

  When 운영자가 긴급 접근으로 platform_db 조회를 수행한다

  Then 모든 행위가 audit_log에 (actor_type='HUMAN', action='break_glass_access',
       meta_json={reason:'서비스장애', approver_pk:...}) INSERT된다
  And audit_log row에 break_glass=true 표시 필수
  And delegation_grant.expires_at 도달 시 자동 회수된다
  And 이후 사후 리뷰(post-review) 대상으로 표기된다
```

### P8-03: webhook 서명 검증 — 서명 실패 시 처리 차단

```gherkin
Scenario: PG webhook 서명 검증 실패
  Given TOSS webhook이 수신됐다
  And HMAC-SHA256 서명이 일치하지 않는다

  When webhook handler가 서명을 검증한다

  Then pg_webhook_event INSERT: (signature_ok=false, status='FAILED')
  And 결제 처리 로직이 실행되지 않는다 (payment_ledger INSERT 없음)
  And 보안 audit_log에 (action='webhook_signature_failure', result='DENY') INSERT
```

### P8-04: fan-out anonymize — 탈퇴 시 서비스 DB 정리

```gherkin
Scenario: 회원 탈퇴 → 전 서비스 개인정보 익명화
  Given identity_user.pk=10 (김지영)이 DELETED 상태로 전환됐다
  And 30일이 경과했다

  When 탈퇴 정리 배치가 실행된다

  Then outbox_event에 (event_type='user.deleted', aggregate_pk=10) INSERT
  And outbox 워커가 academy-api에 user.deleted 이벤트 전달
  And academy_db에서 student.display_name, parent_phone 등이 익명화 처리된다
  And platform_db.user_consent_event는 5년 보존 후 파기 대상으로 표기된다
  And identity_user.email, phone_e164가 NULL 처리된다
```

---

## Domain 9: 데이터 이전 — 강사 이전 & 회원탈퇴

> 격리 구조(org_pk NOT NULL) 때문에 이전 가능/불가 경계가 명확하다. admin 페이지에서 처리.

### P9-01: 강사 멤버십 이전 (consent 기반)

```gherkin
Scenario: 강사가 consent 후 admin을 통해 org B로 이전
  Given teacher (identity_user.pk=20)이 org_pk=1(A학원)의 TEACHER 멤버십 보유
  And user_consent_event에 (user_pk=20, consent_type='platform.content_ownership', action='GRANTED') 존재
  And user_consent_event에 (user_pk=20, consent_type='platform.data_transfer', action='GRANTED') 존재
  And org_pk=2(B학원)에서 membership_invite (token='xfer_abc', email=teacher.email) 발송됨

  When admin이 강사 이전 처리를 실행한다

  Then 단일 트랜잭션 내에서:
    | 처리 | 내용 |
    | membership(user_pk=20, org_pk=1) | status='SUSPENDED' |
    | service_membership(user_pk=20, org_pk=1, service='ACADEMY') | status='SUSPENDED' |
    | delegation_grant(org_pk=1, grantee_pk=20) 전체 | status='REVOKED' |
    | membership_invite(token='xfer_abc') | status='ACCEPTED' |
    | membership(user_pk=20, org_pk=2, platform_role='MEMBER') | INSERT |
    | service_membership(user_pk=20, org_pk=2, service='ACADEMY', role_code='TEACHER') | INSERT |
    | outbox_event | event_type='teacher.transferred', payload={from_org:1, to_org:2, user_pk:20} |
  And identity_user, user_profile은 변경 없음 (강사 본인 데이터 — 이전의 대상이 아님)
  And audit_log에 (actor_type='SYSTEM', action='teacher_transfer', org_pk=1, result='ALLOW',
                   meta_json={to_org_pk:2, consent_verified:true}) INSERT
```

### P9-02: consent 없는 이전 시도 차단

```gherkin
Scenario: platform.data_transfer 동의 없이 admin 이전 시도
  Given user_consent_event에 (consent_type='platform.data_transfer') row가 없다

  When admin이 강사 이전 처리를 실행한다

  Then 403 Forbidden ("강사 본인의 데이터 이전 동의가 필요합니다")
  And membership 변경 없음
  And audit_log에 (action='teacher_transfer', result='DENY') INSERT
```

### P9-03: 이전 불가 데이터 경계 확인

```gherkin
Scenario: 강사 이전 후 org A 데이터는 org A에 남아있음
  Given P9-01 이전이 완료됐다

  Then academy_db 내 (org_pk=1, teacher_pk=20)으로 태그된 기존 강의는 org A에 남아있다
  And 학생 등록 기록 (org_pk=1)은 변경되지 않는다
  And payment_ledger (org_pk=1)은 변경되지 않는다
  And teacher_pk=20은 org_pk=2에서 새 강의를 생성할 수 있다 (identity 이전됐으므로)

  Note: 기존 강의 콘텐츠 이전을 원하면 별도 admin 작업 필요
        (academy_db lecture.org_pk 업데이트 — ToS 콘텐츠 소유권 조항 근거)
```

### P9-04: 회원탈퇴 — platform_db 즉시 처리

```gherkin
Scenario: 회원 탈퇴 요청 → 즉시 처리 + 30일 배치
  Given identity_user.pk=10 (김지영)이 탈퇴 요청
  And user_consent_event에 (consent_type='platform.withdrawal', action='GRANTED') INSERT됨

  When 탈퇴 API가 호출된다 (단일 트랜잭션)

  Then identity_user.status='DELETED', deleted_at=NOW()
  And membership(user_pk=10) 전체 status='SUSPENDED'
  And service_membership(user_pk=10) 전체 status='SUSPENDED'
  And delegation_grant(grantee_pk=10 또는 grantor_pk=10) 전체 status='REVOKED'
  And outbox_event INSERT: event_type='user.deleted', aggregate_pk=10
  And 전 서비스 즉시 차단 (Gate A: status='DELETED' → false)

  When 30일 후 hard anonymize 배치가 실행된다

  Then identity_user.email=NULL, phone_e164=NULL, firebase_uid=NULL UPDATE
  And identity_user.email_verified=FALSE, phone_verified=FALSE
  And user_profile.avatar_url=NULL UPDATE
  And user_consent_event rows는 5년 보존 (user_pk는 익명화 안 함 — PIPA 증거)
  And audit_log에 (action='user_hard_anonymize', actor_type='SYSTEM') INSERT
```

### P9-05: 탈퇴 후 동일 이메일 재가입 허용

```gherkin
Scenario: 탈퇴한 이메일로 재가입
  Given identity_user.pk=10이 탈퇴 후 30일 hard anonymize 완료 (email=NULL)

  When 동일 email='kim@example.com'으로 신규 가입 시도

  Then UNIQUE KEY uq_identity_user_email이 email=NULL이라 충돌 없음
  And 신규 identity_user row INSERT 성공 (새 pk, 새 firebase_uid)
  And 이전 user_pk=10과 연관 없음
```

---

## Domain 10: Email & Phone 인증

### P10-01: 이메일 미인증 → 민감 기능 차단

```gherkin
Scenario: 이메일 미인증 사용자 결제 시도
  Given identity_user.email_verified=false
  And JWT claims에도 email_verified=false

  When 결제 API를 호출한다

  Then 403 Forbidden ("이메일 인증 후 이용 가능합니다")
  And payment_ledger에 row INSERT 없음
  And audit_log에 (result='DENY', action='payment_blocked_unverified_email') INSERT

Scenario: 이메일 미인증 사용자 멤버 초대 시도
  Given identity_user.email_verified=false

  When 멤버 초대 API를 호출한다

  Then 403 Forbidden ("이메일 인증 후 초대 가능합니다")
  And membership_invite에 row INSERT 없음
```

### P10-02: Firebase 이메일 인증 완료 → DB 동기화

```gherkin
Scenario: 사용자가 Firebase 인증 이메일 링크 클릭 후 API 재호출
  Given identity_user.email_verified=false
  And 사용자가 Firebase 인증 이메일 링크를 클릭했다
  And Firebase가 JWT email_verified=true로 갱신했다

  When 사용자가 다음 API를 호출한다 (FirebaseJwtGuard 통과)

  Then FirebaseJwtGuard가 JWT의 email_verified=true를 감지한다
  And identity_user.email_verified=true, email_verified_at=NOW() UPDATE
  And 이후 이메일 인증 필요 기능 접근 허용
```

### P10-03: SMS OTP 전화번호 인증

```gherkin
Scenario: 전화번호 OTP 인증 성공
  Given identity_user.phone_e164='+821012345678', phone_verified=false
  And OTP SMS가 발송됐다 (TTL 5분)

  When 사용자가 올바른 OTP를 입력한다

  Then identity_user.phone_verified=true, phone_verified_at=NOW() UPDATE
  And audit_log에 (action='phone_verified', result='ALLOW') INSERT

Scenario: 전화번호 OTP 만료 후 시도
  Given OTP TTL 5분이 경과했다

  When 사용자가 OTP를 입력한다

  Then 400 Bad Request ("인증번호가 만료됐습니다. 재발송하세요")
  And phone_verified=false 유지
```

### P10-04: OTP 브루트포스 차단 (rate-limit)

```gherkin
Scenario: OTP 5회 연속 실패 → 차단
  Given 동일 phone_e164로 OTP 검증이 5회 연속 실패했다

  When 6번째 OTP 시도

  Then 429 Too Many Requests ("잠시 후 다시 시도하세요")
  And OTP 무효화 (재발송 필요)
  And audit_log에 (action='phone_otp_brute_force', result='DENY') INSERT

  Note: rate-limit은 Gateway 레이어에서 강제 (TEN-9, api_key.rate_limit_tier)
```

---

## Cross-check: academy BDD 수용성

> 상세 시나리오(F1~F65)는 academy 서비스 문서에서 관리(본 볼트 범위 밖).  
> 이 섹션은 platform_db 설계가 academy BDD를 수용하는지 갭만 기록.

| academy BDD | platform_db 설계 수용 여부 | 갭/조치 |
|---|---|---|
| F1-01 신규 학원장 가입 | ✅ | identity_user + org + membership + service_membership + entitlement 단일 트랜잭션 (P1-01) |
| F1-02 약관 동의 | ⚠️ | user_consent_event 미구현. 현재 앱 레이어에서만 처리 |
| F3-01~04 강사 초대/수락/만료/제거 | ✅ | membership_invite + membership + service_membership 설계 완료 (P1-02/03) |
| F4-01~04 멀티 워크스페이스 | ✅ | membership 복합 PK(user_pk, org_pk) 설계 완료 (P1-02) |
| F5-01~05 Trust Grant | ✅ | delegation_grant 구현 완료. capability 네임스페이스 `ACADEMY.*` 적용 (P3-01) |
| F6-01~04 Agent 인증 | 🟡 | identity_user.type='SERVICE' ✅. api_key 테이블은 미구현 |
| F7-01~05 권한 거부 | ✅ | 3-gate(Gate A/B/C) + VerifyOnDb 설계 완료 (P2/P3) |
| F11~F16 파이프라인 | ✅ (academy_db) | platform_db와 직접 무관. entitlement.feature_limits로 cap 확인 |
| F37-01~04 DIRECTOR lockout 방지 | ✅ | service_membership DIRECTOR 보호 (P2-04) |
| F43 구독 만료 | ✅ | org_entitlement.status=EXPIRED 전환 설계 완료 |
| F47~F48 결제 흐름 | ✅ | payment_ledger + pg_webhook + outbox 설계 완료 (P4) |

**잔여 갭**:

| 갭 | 심각도 | 조치 |
|---|---|---|
| C-1 `user_consent_event` 미구현 | PIPA P0 법적 | 후속 구현 |
| C-2 `api_key` 미구현 | B2B 머신 인증 P1 | 후속 구현 |
| C-5 youtube_channel의 `org_pk` 컬럼 확인 필요 | 멀티테넌시 불변식 | spec-16-xx 확인 |
