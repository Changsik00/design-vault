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
> 목적: platform_db 도메인 행위 시나리오 — **화이트박스 통합 테스트**(DB row·상태 단언 포함). 블랙박스 e2e 사용자 여정은 [[e2e-journeys]].  
> **구성**: Part A(Domain 1~10, 핵심 흐름) · Part B(Domain 11~19, 요구·설계 전수) · 끝에 요구사항 추적 매트릭스  
> 상위: [[requirements]] · [[schema-reference]] · 블랙박스 e2e: [[e2e-journeys]]
>
> **현행 스키마 반영**: `membership.platform_role` + `service_membership.role_code` 2-layer 구조 적용  
> **현행 스키마**: `organization.type`에서 `ACADEMY` 제거 → `COMPANY/TEAM/PERSONAL`  
> **현행 스키마**: `delegation_grant.capability` 네임스페이스 `ACADEMY.*` 적용  
> **엔진 전제**: **PostgreSQL 우선**. 타입은 `BIGINT GENERATED ALWAYS AS IDENTITY`·`TIMESTAMPTZ`·`JSONB`·`BYTEA`·`VARCHAR+CHECK`, 격리는 MVCC 트랜잭션, 에러는 SQLSTATE(중복 23505·CHECK 23514·FK 23503·권한 42501)로 표기한다. MySQL 등가물이 유용한 곳에만 `🐬 **MySQL이라면**` 콜아웃으로 병기한다. (주의: `pg_provider`·`pg_webhook_event`·"PG webhook"의 PG는 결제대행사(payment gateway)이며 DB Postgres와 무관하다.)  
> Gherkin 형식 (English keywords, Korean content). 통합 테스트: vitest + supertest

>
> 📄 **설계 시나리오 — 구현 코드는 없습니다.** ⚠️ 표시 시나리오는 *기능 코드가 생기면* 테스트로 활성화하며, 설계 자체는 본 문서·[[schema-reference]] 기준으로 완결입니다. 실행 전략(실제 DB·ORM)은 [[testing-strategy]]·[[orm-testing-drizzle]].

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

  Then CHECK constraint 위반(SQLSTATE 23514)으로 INSERT 실패
  And delegation_grant row가 생성되지 않는다

Scenario: 구 네임스페이스(prefix 없음) 거부
  When delegation_grant INSERT: capability='PUBLISH_VIDEO'

  Then CHECK constraint 위반(SQLSTATE 23514)으로 INSERT 실패 (ACADEMY. prefix 필수)
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

  Then expires_at < now() 인 grant는 결과에서 제외된다
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

  Then pg_webhook_event INSERT가 UNIQUE 제약 위반(SQLSTATE 23505)으로 실패 (또는 ON CONFLICT DO NOTHING으로 SKIPPED 처리)
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

  Then DB 계정에 audit_log UPDATE 미부여로 "permission denied"(SQLSTATE 42501) 반환 (GRANT 강제, 앱 레이어 우회 불가)
  And row가 변경되지 않는다
```

### P5-03: 월 파티셔닝 — 조회 성능

```gherkin
Scenario: 특정 월 감사 로그 조회
  Given audit_log에 2026-05 파티션에 1만 건 데이터 존재

  When SELECT * FROM audit_log WHERE org_pk=1 AND created_at BETWEEN '2026-05-01' AND '2026-05-31'

  Then 선언적 파티션 가지치기(partition pruning)로 audit_log_2026_05 파티션만 스캔 (EXPLAIN에 다른 월 파티션 미등장)
  And 전체 테이블 스캔 없음
```

> 🐬 **MySQL이라면**: `EXPLAIN ... PARTITIONS`의 `partitions=p202605` 컬럼으로 가지치기를 확인한다. Postgres는 `EXPLAIN` 계획 트리에서 제외된 파티션이 아예 등장하지 않는 방식으로 prune된다.

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

  Then 모든 행위가 audit_log에 (actor_type='OPERATOR', action='break_glass_access',
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

  Then identity_user.status='DELETED', deleted_at=now()
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

  Then UNIQUE 제약 uq_identity_user_email이 email=NULL이라 충돌 없음 (Postgres: NULL은 서로 distinct로 취급)
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
  And identity_user.email_verified=true, email_verified_at=now() UPDATE
  And 이후 이메일 인증 필요 기능 접근 허용
```

### P10-03: SMS OTP 전화번호 인증

```gherkin
Scenario: 전화번호 OTP 인증 성공
  Given identity_user.phone_e164='+821012345678', phone_verified=false
  And OTP SMS가 발송됐다 (TTL 5분)

  When 사용자가 올바른 OTP를 입력한다

  Then identity_user.phone_verified=true, phone_verified_at=now() UPDATE
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

---

# Part B — 확장 커버리지 (요구사항·설계 전수 검증)

> 2026-05-30 추가. 서브에이전트 8 pillar 병렬 작성 + schema-reference DDL 대조 검증(71건 전부 통과).  
> Part A(Domain 1~10)의 happy-path를 넘어 **요구 ID·핵심 불변식·운영 안전장치까지 행위 검증**으로 채운다.  
> 각 시나리오 끝에 커버 요구 ID 태그. 미구현 요구는 ⚠️(구현 후 통합 테스트 전환).  
> 전수 매핑은 문서 끝 **요구사항 추적 매트릭스** 절.

---

## Domain 11: Identity 경계 — 소셜·토큰·MFA·세션

### 11-01: 소셜(Kakao) custom token 최초 로그인 → 단일 firebase_uid 매핑

```gherkin
Scenario: Kakao 로그인으로 발급된 Firebase custom token 최초 진입
  Given identity_user에 firebase_uid='fb_kakao_77'인 row가 없다
  And 클라이언트가 Kakao OIDC로 받은 사용자 정보로 백엔드에서 Firebase custom token을 발급받았다
  And 그 토큰의 firebase_uid='fb_kakao_77', email='hong@kakao.com', email_verified=true

  When 가입/로그인 단일 트랜잭션이 실행된다

  Then identity_user에 (firebase_uid='fb_kakao_77', email='hong@kakao.com', email_verified=TRUE, type='HUMAN', status='ACTIVE') INSERT
  And user_profile에 (user_pk=<신규 pk>, display_name='홍길동') INSERT
  And identity_user.public_id는 신규 ULID(CHAR(26))로 채워진다
  And uq_identity_user_firebase_uid 제약으로 firebase_uid는 전 서비스 유일하다
```

**커버**: USR-8 · AUTHN-1

### 11-02: 동일인이 다른 소셜 provider(Naver)로 재로그인 시 동일 firebase_uid 재사용

```gherkin
Scenario: 이미 Kakao로 가입한 사용자가 Naver로 로그인해도 동일 사용자로 인식
  Given identity_user (pk=50, firebase_uid='fb_kakao_77', status='ACTIVE')가 존재한다
  And 사용자가 Naver로 로그인했으나 Firebase가 동일 계정을 동일 firebase_uid='fb_kakao_77'로 federate했다

  When 로그인 API가 firebase_uid='fb_kakao_77'로 사용자를 조회한다

  Then 기존 identity_user.pk=50이 그대로 반환된다
  And identity_user에 새 row가 INSERT되지 않는다 (firebase_uid 조회 키 재사용)
  And user_profile, membership, service_membership 모두 user_pk=50 기준으로 그대로 유지된다
```

**커버**: USR-8 · USR-1

### 11-03: 위조·만료 토큰 FirebaseJwtGuard 401 빠른 차단 (DB 미접근)

```gherkin
Scenario Outline: 서명 위조 또는 만료된 JWT는 Guard에서 401
  Given 요청 헤더에 <상태>인 Firebase JWT가 실려 있다

  When FirebaseJwtGuard가 verifyIdToken()을 수행한다

  Then 서명/만료 검증이 실패한다
  And 401 Unauthorized가 반환된다
  And identity_user·membership 등 어떤 DB 조회도 수행되지 않는다 (fail-fast)
  And 후속 GateB/GateC 가드는 실행되지 않는다

  Examples:
    | 상태                       |
    | 서명이 위조된(변조) 토큰   |
    | exp가 과거인 만료 토큰     |
    | aud/iss가 프로젝트 불일치  |
```

**커버**: AUTHN-3 · AUTHN-1

### 11-04: ⚠️ JWT 필수 claim 누락 fail-closed + custom claim stale 시 DB fallback

```gherkin
Scenario: 서명은 유효하나 필수 claim 누락 → fail-closed 401
  Given JWT 서명·exp는 유효하다
  And custom claim 'allMemberships'와 표준 claim 'sub' 중 하나가 누락됐다

  When FirebaseJwtGuard가 claim 무결성을 검사한다

  Then deny-by-default로 401 Unauthorized 반환 (claim 없음=권한 없음으로 간주, 통과 아님)
  And audit_log에 (actor_type='SYSTEM', action='jwt_claim_missing', result='DENY') INSERT

Scenario: allMemberships claim이 비어 sensitive write에서 DB fallback
  Given JWT는 유효하나 allMemberships claim이 빈 배열이다(갱신 전 stale)
  And 사용자는 실제로는 service_membership (user_pk=50, org_pk=1, service='ACADEMY', role_code='TEACHER', status='ACTIVE')을 보유한다

  When @VerifyOnDb가 걸린 sensitive write API를 호출한다

  Then service_membership을 DB에서 직접 재조회해 role_code='TEACHER'를 확인한다
  And 권한이 충족되면 처리하되 perm_version 불일치 시 클라이언트에 forceRefresh 유도
  And DB 재조회 중 오류가 나면 503을 반환한다 (허용 아님 — fail-closed)
```

**커버**: AUTHN-8 · AUTHN-1  ·  ⚠️ 구현 후 활성화

### 11-05: ⚠️ MFA 미충족 OWNER의 민감 운영 차단 (platform_role='OWNER')

```gherkin
Scenario: MFA 미완료 OWNER의 멤버 정지 시도 차단
  Given membership (user_pk=10, org_pk=1, platform_role='OWNER', status='ACTIVE')
  And 해당 사용자의 Firebase JWT에 MFA(second factor) 통과 claim이 없다

  When OWNER가 다른 멤버를 정지시키는 민감 운영 API를 호출한다

  Then 403 Forbidden ("관리자 작업에는 2단계 인증이 필요합니다")
  And membership.status 변경 없음
  And audit_log에 (actor_type='HUMAN', actor_pk=10, action='mfa_required_for_owner_action', result='DENY') INSERT

Scenario: MFA 충족 OWNER는 민감 운영 허용
  Given membership (user_pk=10, org_pk=1, platform_role='OWNER', status='ACTIVE')
  And JWT에 MFA 통과 claim이 존재한다

  When 동일 민감 운영 API를 호출한다

  Then 정상 처리되고 audit_log에 (actor_type='HUMAN', actor_pk=10, action='revoke_membership', result='ALLOW') INSERT
```

**커버**: AUTHN-4  ·  ⚠️ 구현 후 활성화

### 11-06: 계정 정지(SUSPENDED) 시 전 서비스 즉시 차단 — sensitive write는 DB 재검증으로 즉각

```gherkin
Scenario: 운영상 정지된 사용자의 sensitive write 즉시 차단
  Given identity_user (pk=50, status='ACTIVE')가 academy·market 양 서비스 멤버다
  And 운영자가 방금 identity_user.status='SUSPENDED'로 UPDATE했다
  And 사용자의 JWT custom claims는 아직 stale(ACTIVE)이다

  When 사용자가 @VerifyOnDb가 걸린 sensitive write API를 호출한다

  Then DB에서 identity_user.status='SUSPENDED'를 재확인한다
  And 403 Forbidden 반환 (서비스 무관 — academy·market 모두 동일하게 차단)
  And audit_log에 (actor_type='HUMAN', actor_pk=50, action='blocked_suspended_user', result='DENY') INSERT

Scenario: 정지 해제 후 다시 ACTIVE면 정상 접근 복원
  Given identity_user (pk=50)이 status='SUSPENDED'에서 status='ACTIVE'로 복원됐다

  When 사용자가 토큰 갱신 후 API를 호출한다

  Then DB status='ACTIVE' 확인되어 정상 처리된다 (SUSPENDED는 가역 — DELETED와 달리 익명화 없음)
```

**커버**: AUTHN-6 · USR-6

### 11-07: 다중 디바이스 동시 세션 허용 + 이상 로그인 감사 기록

```gherkin
Scenario: 동일 사용자의 다중 디바이스 동시 접근 허용
  Given identity_user (pk=50, firebase_uid='fb_kakao_77', status='ACTIVE')
  And 동일 firebase_uid로 모바일·데스크톱 두 디바이스가 각각 유효한 JWT를 보유한다

  When 두 디바이스가 동시에 API를 호출한다

  Then 두 요청 모두 FirebaseJwtGuard를 통과한다 (세션 단일화 제약 없음 — Firebase 위임)
  And platform_db에는 별도 device/session row가 생성되지 않는다

Scenario: 이상 로그인(평소와 다른 국가 IP) 감사 기록
  Given identity_user (pk=50)의 최근 로그인 ip는 국내 대역이었다
  And Firebase 이상 로그인 신호가 해외 IP 로그인을 백엔드로 전달했다

  When 로그인 후처리 훅이 실행된다

  Then audit_log에 (actor_type='HUMAN', actor_pk=50, action='anomalous_login', result='ALLOW', ip=<해외 IP raw>) INSERT
  And ip는 INET(PG 네이티브 IP 타입)로 저장된다 (PIPA 감사 요건)

  Note: 차단/추가인증 정책은 Firebase·Gateway 레이어. platform_db는 감사 기록만 소유.
```

**커버**: AUTHN-7

### 11-08: ⚠️ email ACTIVE-unique — 활성 사용자 중복 가입 차단, 탈퇴 후 재가입 허용

```gherkin
Scenario: 같은 이메일의 ACTIVE 사용자가 이미 있으면 신규 가입 거부
  Given identity_user (pk=50, email='hong@kakao.com', status='ACTIVE')가 존재한다

  When 동일 email='hong@kakao.com', 다른 firebase_uid로 신규 가입을 시도한다

  Then ACTIVE-unique 가드(앱 트랜잭션)가 충돌을 감지한다
  And 409 Conflict ("이미 가입된 이메일입니다")
  And 신규 identity_user row INSERT 없음

Scenario: 정지(SUSPENDED) 상태 사용자가 점유한 이메일도 재가입 차단
  Given identity_user (pk=50, email='hong@kakao.com', status='SUSPENDED')

  When 동일 email로 신규 가입을 시도한다

  Then 409 Conflict (SUSPENDED는 여전히 email 점유 — 가역 상태이므로 unique 대상)
  And 신규 row INSERT 없음

  Note: status='DELETED' → 30일 hard anonymize(email=NULL) 이후 재가입은 P9-05에서 허용. 본 시나리오는 anonymize 이전 ACTIVE/SUSPENDED 점유 케이스다.
```

**커버**: USR-5 · USR-6  ·  ⚠️ 구현 후 활성화

### 11-09: display_name SSOT는 user_profile — 서비스 DB는 표시이름을 자체 저장하지 않음

```gherkin
Scenario: 사용자가 표시이름을 변경하면 전 서비스가 단일 출처를 읽는다
  Given user_profile (user_pk=50, display_name='홍길동')
  And academy_db.teacher_profile은 display_name 컬럼을 갖지 않고 user_pk만 참조한다

  When 사용자가 PATCH /me/profile { display_name: '홍길순' }를 호출한다

  Then user_profile.display_name='홍길순' UPDATE (단일 SSOT)
  And academy·market 어디서 조회해도 user_profile.display_name='홍길순'을 그대로 노출한다
  And 서비스 DB에 display_name 사본이 갱신/존재하지 않는다

Scenario: 신규 사용자는 user_profile.display_name이 NOT NULL이라 누락 불가
  Given 가입 트랜잭션이 user_profile 없이 identity_user만 INSERT하려 한다

  When 트랜잭션이 커밋된다

  Then user_profile.display_name NOT NULL 제약으로 실패하거나 가입 트랜잭션이 user_profile을 함께 INSERT해야 한다 (표시이름 없는 사용자 불가)
```

**커버**: USR-4 · USR-1

---

## Domain 12: 인가 심화 — 계층·소유권·한도·visibility

### 12-01: org 계층 생성 — HQ_BRANCH / HOLDING 관계 등록 및 조회

```gherkin
Scenario: 본사-지점 및 지주-자회사 org 계층 등록
  Given organization에 pk=1(본사), pk=2(지점A), pk=3(지점B), pk=10(지주사), pk=11(자회사)가 ACTIVE 상태로 존재한다

  When 다음 org_relation row들이 INSERT된다:
    | parent_org_pk | child_org_pk | relation_type |
    | 1             | 2            | HQ_BRANCH     |
    | 1             | 3            | HQ_BRANCH     |
    | 10            | 11           | HOLDING       |

  Then 3건 모두 INSERT 성공
  And getChildOrgs(parentOrgPk=1)는 child_org_pk=2, child_org_pk=3을 반환한다 (idx_org_relation_child 활용)
  And relation_type은 VARCHAR+CHECK(relation_type IN ('HQ_BRANCH','HOLDING'))로 두 값만 허용된다
```

> org_relation은 HQ_BRANCH/HOLDING 2단만. 깊은 다단계 그래프는 범위 밖(requirements §7.2 ORGGRAPH-1).

**커버**: REBAC-4

### 12-02: org 계층 중복 등록 거부 — UNIQUE(parent_org_pk, child_org_pk)

```gherkin
Scenario: 동일 부모-자식 쌍 중복 INSERT 차단
  Given org_relation에 (parent_org_pk=1, child_org_pk=2, relation_type='HQ_BRANCH')가 이미 존재한다

  When org_relation에 (parent_org_pk=1, child_org_pk=2, relation_type='HOLDING') INSERT를 시도한다

  Then uq_org_relation_parent_child UNIQUE 제약 위반으로 INSERT 실패
  And 기존 (parent_org_pk=1, child_org_pk=2, relation_type='HQ_BRANCH') row는 변경되지 않는다
```

**커버**: REBAC-4

### 12-03: org 자기참조 계층 거부 — chk_no_self_ref CHECK

```gherkin
Scenario: parent_org_pk == child_org_pk 자기참조 INSERT 차단
  Given organization.pk=1(본사)이 존재한다

  When org_relation에 (parent_org_pk=1, child_org_pk=1, relation_type='HQ_BRANCH') INSERT를 시도한다

  Then chk_no_self_ref CHECK 제약 위반으로 INSERT 실패 (parent_org_pk != child_org_pk)
  And org_relation에 row가 생성되지 않는다
```

**커버**: REBAC-6

### 12-04: 계층은 권한 근거 아님 — 부모 org 멤버가 자식 org 리소스 자동 접근 불가

```gherkin
Scenario: 본사 OWNER가 계층만으로 지점 org 리소스 접근 시도
  Given org_relation에 (parent_org_pk=1, child_org_pk=2, relation_type='HQ_BRANCH')가 존재한다
  And identity_user.pk=10(본사 대표)이 org_pk=1에서 membership.platform_role='OWNER', status='ACTIVE'다
  And user_pk=10은 org_pk=2(지점A)에 membership row가 없다

  When user_pk=10이 org_pk=2 컨텍스트로 API를 호출한다

  Then checkGateA(userPk=10, orgPk=2)는 false를 반환한다 (org_pk=2에 membership 없음 → 403)
  And org_relation의 부모-자식 관계는 Gate A/B/C 평가에 전혀 입력되지 않는다
  And 지점A 접근이 필요하면 user_pk=10에 (org_pk=2) membership을 명시적으로 INSERT해야 한다
```

> 불변식: org_relation은 데이터 모델일 뿐 인가 입력이 아니다. Gate A/B/C 어디에서도 org_relation을 읽지 않는다. 명시적 membership/service_membership만 권한을 만든다.

**커버**: REBAC-5 · REBAC-4

### 12-05: ⚠️ delegation_grant(platform) ↔ 서비스 trust_relationship(도메인) 경계

```gherkin
Scenario: platform capability 위임은 delegation_grant, 서비스 도메인 신뢰는 서비스 DB
  Given DIRECTOR(grantor_pk=10)가 강사(grantee_pk=20)에게 platform 차원 위임을 부여한다

  When delegation_grant에 (grantor_pk=10, grantee_pk=20, org_pk=1, capability='ACADEMY.MANAGE_MEMBERS', status='ACTIVE') INSERT된다

  Then platform_db의 delegation_grant에만 기록된다 (capability 네임스페이스 <service>.<action>)
  And academy 서비스 고유의 강사↔학생 trust 관계는 academy_db의 도메인 테이블이 소유한다 (platform_db에 INSERT되지 않음)
  And platform_db → 서비스DB 방향 참조는 금지이므로 delegation_grant는 서비스 trust 행을 FK로 참조하지 않는다 (cross-schema FK 전면 금지 — ARCH-5)

  Note: Gate C(CASL)는 delegation_grant.capability(platform)와 서비스측 trust를 각자 평가 — 권한 경계가 DB 소유권으로 분리됨
```

> REBAC-8: 규칙은 OK, 구현 진행 중. platform_db는 platform 차원의 capability 위임(delegation_grant)만 소유하고, 서비스 고유 trust 관계(예: academy_db.trust_relationship)는 서비스 DB가 소유한다. cross-schema FK 금지 원칙(ARCH-5/A.1)과 정합.

**커버**: REBAC-8 · REBAC-1  ·  ⚠️ 구현 후 활성화

### 12-06: 소유권 기반 통제 — owner_pk == principal (ABAC) + org 경계 동시 강제

```gherkin
Scenario: 본인 소유 리소스는 수정 허용, 타인 소유는 거부
  Given identity_user.pk=20(강사 최정훈)이 org_pk=1에서 service_membership.role_code='TEACHER', status='ACTIVE'다
  And lecture.pk=100이 (org_pk=1, teacher_pk=20)으로 소유된다
  And lecture.pk=200이 (org_pk=1, teacher_pk=21)으로 소유된다 (다른 강사)

  When user_pk=20이 lecture.pk=100 수정을 요청한다

  Then Gate C에서 ability.can('update', lecture) 평가 시 lecture.teacher_pk(20) == principal(20) → 허용

  When user_pk=20이 lecture.pk=200 수정을 요청한다

  Then lecture.teacher_pk(21) != principal(20) → 403 Forbidden (소유권 불일치)
  And 두 요청 모두 org_pk=1 경계는 충족(ABAC-2)하나 소유권 속성에서 갈린다
  And audit_log에 (actor_type='HUMAN', actor_pk=20, action='update_lecture', result='DENY') INSERT
```

> 소유권 속성은 리소스를 보유한 서비스 DB(예: academy_db.lecture.teacher_pk)에 있고, principal(user_pk)는 platform 인증에서 옴. Gate C(CASL)가 둘을 결합 평가.

**커버**: ABAC-1 · ABAC-2

### 12-07: feature_limits 한도 평가 — entitlement는 한도 정의, 카운터는 서비스측

```gherkin
Scenario: daily_uploads 한도 도달 시 거부, 미만이면 허용
  Given org_entitlement (org_pk=1, service='ACADEMY', status='ACTIVE', feature_limits={"daily_uploads":6})가 존재한다
  And academy_db의 서비스측 카운터가 오늘 org_pk=1의 업로드를 5건으로 집계한다

  When 6번째 업로드를 시도한다

  Then getFeatureLimit(orgPk=1, service='ACADEMY', 'daily_uploads')는 6을 반환한다 (feature_limits가 권위)
  And 서비스 카운터 5 < 6 → 허용

  When 7번째 업로드를 시도한다 (카운터 6)

  Then 6 >= 6 → 한도 초과로 서비스 레이어가 거부
  And 런타임 한도 판단 시 product_feature·plan_definition.default_limits는 조회하지 않는다 (불변식 #10)

  Note: limit_value/feature_limits 값이 NULL이면 무제한으로 평가(product_feature.limit_value NULL 규약)
```

> ABAC-3: org_entitlement.feature_limits(JSONB)가 한도 SSOT(불변식 #10, D.12 설계포인트). product_feature/plan_definition.default_limits는 초기값 복사용일 뿐 런타임 조회 금지. 실시간 사용량 카운터·차단은 서비스 DB.

**커버**: ABAC-3

### 12-08: 리소스 visibility 속성 필터 (ABAC) — PRIVATE은 소유자만

```gherkin
Scenario: visibility=PRIVATE 리소스는 소유자만, PUBLISHED는 org 멤버 전체
  # 주의: visibility는 academy_db(서비스 DB)의 도메인 속성이다. platform_db DDL(L.1 lecture 템플릿)에는 status만 있고 visibility는 비범위 — A.1 cross-DB 규칙상 서비스 DB가 소유. 아래 'PUBLISHED'는 visibility 값이며 lecture.status('DRAFT'/'PUBLISHED'/'ARCHIVED')와는 별개 속성이다.
  Given org_pk=1에 lecture.pk=300(visibility='PRIVATE', teacher_pk=20), lecture.pk=301(visibility='PUBLISHED', teacher_pk=20)가 있다 (visibility는 academy_db 도메인 컬럼)
  And identity_user.pk=21(같은 org의 다른 강사)이 org_pk=1에서 service_membership.role_code='TEACHER', status='ACTIVE'다

  When user_pk=21이 org_pk=1 컨텍스트에서 강의 목록을 조회한다

  Then Gate C(CASL) visibility 필터에 의해 lecture.pk=301(visibility=PUBLISHED)은 결과에 포함된다
  And lecture.pk=300(visibility=PRIVATE, teacher_pk=20)은 user_pk=21 결과에서 제외된다 (소유자 아님)
  And org_pk 경계(ABAC-2)는 두 강의 모두 충족하나 visibility 속성에서 추가 필터링된다
```

> ABAC-5: visibility 속성은 리소스(서비스 DB)에 있고 Gate C(CASL)가 read 평가에 결합. org 경계(ABAC-2) 통과 후에도 visibility로 추가 필터. [검증주의] platform_db 문서의 academy_db.lecture 템플릿(L.1)에는 visibility 컬럼이 명시돼 있지 않다(status VARCHAR+CHECK('DRAFT','PUBLISHED','ARCHIVED')만 존재). visibility는 academy_db 실제 스키마의 도메인 속성으로 전제하며, platform_db DDL 정합 대상 아님(서비스 DB 소유 — A.1 cross-DB 규칙).

**커버**: ABAC-5

### 12-09: can() 최종 결정 캐싱 금지 — 입력 블록만 TTL 60s

```gherkin
Scenario: 위임 회수 직후 can() 재평가가 즉시 거부로 바뀐다
  Given user_pk=20에 delegation_grant(capability='ACADEMY.PUBLISH_VIDEO', status='ACTIVE')가 있고 입력 블록이 Redis에 TTL 60s로 캐시됐다
  And 직전 can('ACADEMY.PUBLISH_VIDEO', resource) 평가는 허용이었다

  When 같은 요청이 1초 뒤 다시 들어온다

  Then can() 최종 결정 결과는 Redis에서 읽지 않고 매 요청 재평가된다 (결정 캐싱 금지)
  And 입력 블록(delegation_grant 등)은 캐시 TTL 60s 내에서 재사용될 수 있다

  When delegation_grant.status='REVOKED' UPDATE 후 sensitive write(@VerifyOnDb)가 호출된다

  Then @VerifyOnDb가 DB를 재조회해 status='REVOKED'를 확인하고 403을 반환한다 (캐시 TTL과 무관하게 즉시 차단)
  And Redis가 다운돼도 can()은 DB 직접 조회로 평가된다 (캐시는 권위 아님, fail-open 금지)
```

> ABAC-7: 최종 can() 판정은 절대 캐싱하지 않는다(권한 변경 즉시 반영). 캐싱 대상은 입력 블록(membership/entitlement/role 등)만 TTL 60s. Redis는 가속용이지 권위 아님(N.2 entitlement 캐시 미스 — fail-open 아님, DB 직격).

**커버**: ABAC-7

### 12-10: role→action 매핑은 코드 상수 ROLE_PERMISSION (DB 저장 금지)

```gherkin
Scenario: role_code는 DB에, 권한 매핑은 코드 상수에서만 해석
  Given service_membership (user_pk=30, org_pk=1, service='ACADEMY', role_code='TEACHER', status='ACTIVE')가 존재한다
  And platform_db 어느 테이블에도 'TEACHER가 무슨 action을 갖는지'를 저장하는 행은 없다

  When Gate C가 user_pk=30의 권한을 빌드한다

  Then service_membership.role_code='TEACHER'를 읽어 ROLE_PERMISSION['ACADEMY']['TEACHER'] (코드 상수)로 action 집합을 해석한다
  And SUSPENDED service_membership은 도메인 역할 미부여 (status 필터)

  When 신규 서비스 역할 'MARKET'/'SELLER'의 권한을 추가한다

  Then service_membership에 (service='MARKET', role_code='SELLER') 행 INSERT만으로 멤버십이 생기고 (chk_svc_mbr_service에 MARKET 포함 — 무마이그레이션, EXT-4)
  And SELLER→action 매핑은 ROLE_PERMISSION['MARKET'] 코드 상수 배포로 추가된다 (platform DDL 변경 0 — RBAC-3/EXT-3)

  Note: delegation_grant.chk_capability CHECK는 코드 ROLE_PERMISSION과 중복된 DB측 안전망일 뿐 권한 출처는 코드 상수(EXT-2 비대칭). 단 MARKET 위임 capability를 쓰려면 chk_capability는 §K대로 별도 ALTER 필요(service_membership과 비대칭)
```

> RBAC-3/EXT-3: role→action은 ROLE_PERMISSION[service][roleCode] 코드 상수가 권위. DB에는 role_code(VARCHAR)만 저장하고 그 role이 무슨 action을 가지는지는 저장하지 않는다. 새 서비스 역할 권한 추가 = 코드 배포(DB DDL 0).

**커버**: RBAC-3 · EXT-3

---

## Domain 13: Billing 심화 — N:M 번들·업다운그레이드·projection

### 13-01: N:M 번들 구독 — org_subscription 헤더 1건 + subscription_item 여러 SKU

```gherkin
Scenario: 한 구독에 ACADEMY PRO SKU + 추가 좌석 SKU를 묶어 번들 구독 생성
  Given organization.pk=1 (한울학원)이 존재한다
  And identity_user.pk=10 (김지영, payer)이 org_pk=1의 OWNER다
  And product_sku.pk=100 (code='ACADEMY_PRO_MONTHLY', billing_cycle='MONTHLY', price_krw=99000, status='ACTIVE')가 존재한다
  And product_sku.pk=101 (code='ACADEMY_SEAT_ADDON_MONTHLY', billing_cycle='MONTHLY', price_krw=10000, status='ACTIVE')가 존재한다

  When 번들 구독 생성이 단일 트랜잭션으로 실행된다

  Then org_subscription에 INSERT: (pk=500, org_pk=1, payer_user_pk=10, status='ACTIVE', pg_provider='TOSS', current_period_start=now(), current_period_end=+30일)
  And org_subscription 행에는 sku_pk 컬럼이 존재하지 않는다 (불변식 #11 — 진실 원천은 subscription_item)
  And subscription_item에 INSERT: (subscription_pk=500, sku_pk=100, quantity=1, status='ACTIVE')
  And subscription_item에 INSERT: (subscription_pk=500, sku_pk=101, quantity=5, status='ACTIVE')
  And subscription_item의 UNIQUE KEY uq_sub_sku (subscription_pk, sku_pk)에 의해 동일 (500,100) 중복 INSERT는 거부된다
```

> §D.13 org_subscription에 sku_pk 없음(불변식 #11) 확인. payer_user_pk·status·pg_provider·current_period_start/end 모두 실재 컬럼. §D.14 subscription_item UNIQUE uq_sub_sku(subscription_pk, sku_pk) 정확. product_sku.status VARCHAR+CHECK('ACTIVE','RETIRED')에 'ACTIVE' 유효.

**커버**: BILL-1 · BILL-3

### 13-02: subscription_item 단독 조회 금지 — 부모 org_subscription JOIN으로만 org 격리

```gherkin
Scenario: 타 org가 subscription_item을 직접 조회 시도
  Given org_subscription.pk=500이 org_pk=1 소속이다
  And subscription_item (subscription_pk=500, sku_pk=100)이 존재한다
  And 호출자 컨텍스트는 org_pk=2다

  When subscription_item을 org_subscription JOIN으로 조회한다
    (WHERE subscription_item.subscription_pk = org_subscription.pk AND org_subscription.org_pk = 2)

  Then 결과 row가 없다 (org_pk=1 구독은 org_pk=2 질의에 포함되지 않음)
  And subscription_item을 org_subscription JOIN 없이 단독 SELECT하는 패턴은 금지된다 (CI 린트 P1 대상)
  And 404 NotFoundException 반환 (존재 여부 비공개)
```

> §D.14 subscription_item에 자체 org_pk 없음(불변식 #3 예외) 확인. FK fk_sub_item_sub(subscription_pk→org_subscription.pk) 실재. 단독 조회 금지·CI 린트 P1(§G.1)·404 비공개(§H.1 BOLA) 모두 schema 정합.

**커버**: BILL-1

### 13-03: ⚠️ 업그레이드 — 즉시 반영 (entitlement.feature_limits 상향 + perm_version bump)

```gherkin
Scenario: BASIC → PRO 플랜 업그레이드 즉시 적용
  Given org_entitlement (org_pk=1, product_code='ACADEMY_BASIC', service='ACADEMY', status='ACTIVE', source='SUBSCRIPTION', plan_code='BASIC', feature_limits={"daily_uploads":6})가 존재한다
  And plan_definition (plan_code='PRO', default_limits={"daily_uploads":30,"members":200})가 존재한다
  And organization.perm_version=10이다

  When 업그레이드 결제가 성공하여 즉시 적용 트랜잭션이 실행된다

  Then 단일 트랜잭션 내에서:
    | 처리 | 내용 |
    | payment_ledger INSERT | type='CHARGE', amount_minor=<차액 proration>, idempotency_key='up_001', status='SUCCEEDED' |
    | subscription_item UPDATE/INSERT | 신규 PRO sku_pk로 교체 (기존 BASIC subscription_item 행 status='CANCELED') |
    | org_entitlement UPDATE | plan_code='PRO', feature_limits={"daily_uploads":30,"members":200} (즉시 상향) |
    | organization UPDATE | perm_version=11 |
    | billing_event INSERT | event_type='PLAN_CHANGE', plan_code='PRO' |
  And 변경은 current_period_end를 기다리지 않고 즉시 유효하다
```

> payment_ledger.type='CHARGE'/status='SUCCEEDED'(§D.17), subscription_item.status='CANCELED'(§D.14 CHECK 어휘에 존재), org_entitlement.plan_code/feature_limits(§D.12), billing_event.event_type='PLAN_CHANGE'/plan_code(§D.16) 모두 실재. plan_definition.default_limits→entitlement.feature_limits 복사·perm_version bump는 §F.1 단일 트랜잭션 패턴과 일치.

**커버**: BILL-7  ·  ⚠️ 구현 후 활성화

### 13-04: ⚠️ 다운그레이드 — 기간말 예약 (즉시 차감 없음, 현 entitlement 유지)

```gherkin
Scenario: PRO → BASIC 다운그레이드 요청은 즉시 차감하지 않고 기간말로 예약
  Given org_entitlement (org_pk=1, plan_code='PRO', feature_limits={"daily_uploads":30}, status='ACTIVE', current_period_end='2026-06-30')가 존재한다
  And organization.perm_version=11이다

  When 사용자가 BASIC으로 다운그레이드를 요청한다

  Then payment_ledger에 즉시 CHARGE/REFUND가 기록되지 않는다 (이미 결제한 기간 보존)
  And org_entitlement.feature_limits는 현재 {"daily_uploads":30} 그대로 유지된다 (즉시 하향 없음)
  And org_entitlement.plan_code는 'PRO' 그대로다 (current_period_end 전까지)
  And billing_event에 INSERT: event_type='PLAN_CHANGE', plan_code='BASIC' (예약 기록)
  And outbox_event에 INSERT: event_type='subscription.downgrade_scheduled', payload_json={effective_at:'2026-06-30', target_plan:'BASIC'}
  And organization.perm_version은 변경되지 않는다 (현재 권한 동일)

  When 2026-06-30 기간 갱신 배치가 실행된다

  Then org_entitlement.plan_code='BASIC', feature_limits={"daily_uploads":6}으로 하향된다
  And organization.perm_version=12로 bump된다
```

> billing_event.event_type='PLAN_CHANGE'/plan_code='BASIC'(§D.16), outbox_event.event_type(VARCHAR80)/payload_json(§D.19), org_entitlement.current_period_end/plan_code/feature_limits(§D.12) 모두 실재. 예약을 전용 테이블 없이 billing_event+outbox로 표현하는 설계가 schema와 정합(전용 scheduled 테이블 미존재 확인).

**커버**: BILL-7  ·  ⚠️ 구현 후 활성화

### 13-05: ⚠️ webhook replay tolerance — 타임스탬프 ±5분 초과 시 SKIPPED

```gherkin
Scenario: webhook 타임스탬프가 서버 시각 대비 ±5분을 벗어나면 처리 차단
  Given TOSS webhook (event_id='evt_replay_001')이 수신됐다
  And HMAC 서명은 일치한다 (signature_ok=TRUE)
  And payload_json.timestamp가 서버 시각보다 7분 과거다 (tolerance ±5분 초과)

  When webhook handler가 INSERT 전 replay tolerance를 검증한다

  Then pg_webhook_event INSERT: (pg_provider='TOSS', event_id='evt_replay_001', signature_ok=TRUE, status='SKIPPED')
  And payment_ledger에 INSERT가 발생하지 않는다 (재생공격 방어)
  And org_entitlement 변경 없음
  And 보안 audit_log에 INSERT: (actor_type='SYSTEM', action='webhook_replay_window_exceeded', result='DENY')

Scenario: ±5분 이내 webhook은 정상 처리
  Given TOSS webhook (event_id='evt_fresh_002', signature_ok=TRUE)의 payload_json.timestamp가 서버 시각 대비 2분 차이다

  When handler가 replay tolerance를 검증한다

  Then tolerance 통과 → 결제 처리 트랜잭션이 진행된다
  And pg_webhook_event.status가 최종 'PROCESSED'로 기록된다
```

> pg_webhook_event.signature_ok·status VARCHAR+CHECK('RECEIVED','PROCESSED','SKIPPED','FAILED')·payload_json(JSONB)·pg_provider VARCHAR+CHECK('TOSS','STRIPE','PAYPAL')(§D.18) 모두 실재. audit_log.actor_type='SYSTEM'/action(VARCHAR100)/result='DENY'(§D.8) 유효. replay tolerance는 §F.4 외 핸들러 코드 규약으로 schema와 모순 없음.

**커버**: BILL-9 · BILL-4  ·  ⚠️ 구현 후 활성화

### 13-06: source=FREE/MANUAL/PROMO — 결제 없이 access 부여 (payment_ledger 무관)

```gherkin
Scenario: MANUAL source로 운영자가 결제 없이 entitlement 부여
  Given organization.pk=1이 존재하나 org_subscription도 payment_ledger row도 없다

  When 운영자가 entitlement를 수동 부여한다 (Support Action)

  Then org_entitlement에 INSERT: (org_pk=1, product_code='ACADEMY_PRO', service='ACADEMY', status='ACTIVE', source='MANUAL', feature_limits={"daily_uploads":30}, valid_until='2026-12-31')
  And payment_ledger에는 대응 row가 없다 (결제 없이 access)
  And checkGateB(orgPk=1, service='ACADEMY')가 true를 반환한다 (status='ACTIVE')

Scenario: PROMO source — 프로모션 무료 체험 access
  Given org_entitlement (org_pk=2, service='ACADEMY', status='ACTIVE', source='PROMO', valid_until=+14일)가 부여됐다

  When checkGateB(orgPk=2, service='ACADEMY')가 호출된다

  Then true 반환 (source 무관, status='ACTIVE' AND valid_until > now())
  And valid_until 경과 후 배치가 status='EXPIRED'로 전환하면 Gate B는 402를 반환한다
```

> org_entitlement.source VARCHAR+CHECK('SUBSCRIPTION','PROMO','MANUAL','FREE')(§D.12) 정확. Gate B 판정 status IN('ACTIVE','GRACE') AND (valid_until IS NULL OR valid_until>now())(§E.2·불변식 #9)와 일치. source 무관 entitlement 권위(불변식 #4)·EXPIRED 전환 시 402(§F.3) 정합.

**커버**: BILL-10

### 13-07: canXXX는 org_entitlement만 읽는다 — payment_ledger 직접 조회 금지 (불변식 #4)

```gherkin
Scenario: payment_ledger는 SUCCEEDED지만 entitlement가 EXPIRED면 Gate B는 차단
  Given payment_ledger (org_pk=1, type='CHARGE', status='SUCCEEDED', idempotency_key='inv_late_001')가 존재한다
  And org_entitlement (org_pk=1, service='ACADEMY', status='EXPIRED', valid_until='2026-04-30')다
    (결제는 성공했으나 outbox 유실로 entitlement 미갱신 상태)

  When checkGateB(orgPk=1, service='ACADEMY')가 호출된다

  Then Gate B는 org_entitlement만 읽는다 (payment_ledger를 조회하지 않음 — 불변식 #4)
  And status='EXPIRED'이므로 false 반환 → 402 Payment Required
  And 복구는 Webhook Replay 또는 Support Action으로 entitlement를 갱신해야 한다 (Gate B가 ledger를 직접 보는 우회 금지)

Scenario: entitlement 우선 — payment_ledger 없이 source='FREE'면 통과
  Given org_entitlement (org_pk=3, service='ACADEMY', status='ACTIVE', source='FREE')다
  And payment_ledger에 org_pk=3 row가 전혀 없다

  When checkGateB(orgPk=3, service='ACADEMY')가 호출된다

  Then true 반환 (entitlement가 권위 — ledger 부재가 access를 막지 않는다)
```

> payment_ledger.type='CHARGE'/status='SUCCEEDED'/idempotency_key(§D.17), org_entitlement.status='EXPIRED'/valid_until/source(§D.12) 모두 실재. 불변식 #4(Gate B는 ledger 미조회)·§F.3(EXPIRED→402) 정확. 복구 경로(Webhook Replay/Support Action)는 §N.2 갭 보강과 정합.

**커버**: BILL-2

### 13-08: plan_definition.default_limits → org_entitlement.feature_limits 최초 복사 (불변식 #10)

```gherkin
Scenario: 구독 활성화 시 plan_definition.default_limits를 entitlement.feature_limits로 복사
  Given plan_definition (plan_code='PRO', default_limits={"daily_uploads":30,"members":200}, is_active=TRUE)가 존재한다
  And product_sku.pk=100 (plan_code='PRO')로 결제가 성공했다
  And org_entitlement (org_pk=1, product_code='ACADEMY_PRO')가 아직 없다

  When 결제 → 권한 활성화 트랜잭션이 실행된다 (entitlement 최초 생성)

  Then org_entitlement INSERT: (org_pk=1, service='ACADEMY', status='ACTIVE', source='SUBSCRIPTION', plan_code='PRO', feature_limits={"daily_uploads":30,"members":200})
  And feature_limits 값은 plan_definition.default_limits에서 복사된 것이다 (최초 1회)

Scenario: 런타임 한도 판단은 org_entitlement.feature_limits만 읽는다 (plan_definition 직접 조회 금지)
  Given org_entitlement (org_pk=1, plan_code='PRO', feature_limits={"daily_uploads":50})다
    (운영자가 entitlement.feature_limits만 50으로 상향, plan_definition.default_limits는 여전히 30)

  When 업로드 한도 평가(ABAC-3)가 수행된다

  Then 한도는 org_entitlement.feature_limits의 50으로 판단된다 (entitlement가 SSOT)
  And plan_definition.default_limits(30)는 런타임에 조회되지 않는다 (불변식 #10 — 복사 후 권위는 entitlement)
```

> plan_definition.default_limits JSONB NOT NULL/plan_code/is_active(§D.15), org_entitlement.feature_limits/plan_code/source='SUBSCRIPTION'(§D.12), product_sku.plan_code(§D.11) 모두 실재. 불변식 #10(최초 복사 후 entitlement가 SSOT, plan_definition 런타임 직접 조회 금지)과 정확히 일치.

**커버**: BILL-1 · BILL-3

---

## Domain 14: 감사 2-lane — 보안 이벤트 vs 텔레메트리·append-only GRANT

### 14-01: ⚠️ 2-lane 분기 — 일상 read ALLOW는 텔레메트리(샘플링), audit_log INSERT 아님

```gherkin
Scenario: 일상 read ALLOW는 audit_log가 아니라 텔레메트리 lane으로 라우팅
  Given user_pk=10(HUMAN)이 org_pk=1의 ACADEMY 강의 목록을 조회한다 (read, 비민감)
  And 3-gate(A/B/C) 모두 통과한다

  When 감사 라우터가 이벤트를 분류한다

  Then 이 이벤트는 보안유의 분류(DENY·ERROR·민감 ALLOW·운영자)에 해당하지 않는다
  And audit_log에 row가 INSERT되지 않는다
  And 텔레메트리 lane(OLAP, 샘플링·30~90일 TTL)에만 기록된다
  And 컴플라이언스 audit volume이 read 트래픽으로 오염되지 않는다

  Note: 텔레메트리 lane은 미구현(OLAP TTL, requirements AUD-2 🟡). audit_log는 INSERT-only(§M audit_append 계정).
```

> 텔레메트리 lane은 미구현(OLAP TTL, requirements AUD-2 🟡). audit_log INSERT-only는 §M으로 GRANT 강제(audit_append).

**커버**: AUD-2  ·  ⚠️ 구현 후 활성화

### 14-02: 2-lane 분기 — 민감 write ALLOW는 audit_log로 기록

```gherkin
Scenario: 민감 write ALLOW는 보안유의로 분류되어 audit_log INSERT
  Given user_pk=10(HUMAN)이 org_pk=1에서 강의 publish(민감 write)를 수행한다
  And 3-gate 모두 통과한다

  When 감사 라우터가 이벤트를 분류한다

  Then 민감 write이므로 보안유의 lane으로 분류된다
  And audit_log에 INSERT된다:
    actor_type='HUMAN', actor_pk=10, org_pk=1, action='publish_video', result='ALLOW'
  And 동일 이벤트는 텔레메트리 lane으로 중복 라우팅되지 않는다 (보안유의는 audit가 권위)
```

> 민감 ALLOW는 보안유의 lane → audit_log INSERT. read ALLOW와의 분류 근거 대비 명시.

**커버**: AUD-2

### 14-03: 2-lane 분기 — ERROR(시스템 오류)는 result='ERROR'로 audit_log 기록

```gherkin
Scenario: 권한 판정 중 DB 재검증 오류는 ERROR로 audit_log에 기록
  Given user_pk=10(HUMAN)이 org_pk=1에서 sensitive write(@VerifyOnDb)를 호출한다
  And DB 재검증 중 오류가 발생한다 (fail-closed → 503, §E.6)

  When 감사 라우터가 이벤트를 분류한다

  Then 요청은 허용되지 않는다 (503, deny-by-default)
  And audit_log에 INSERT된다:
    actor_type='HUMAN', actor_pk=10, org_pk=1, action='publish_video', result='ERROR'
  And ERROR는 텔레메트리가 아니라 보안유의 lane(audit_log)으로 기록된다 (관측성 O6 DENY/ERROR 급증 감시 근거)
```

> 기존 P5-01이 다루지 않은 result CHECK 어휘 세 번째 값 ERROR. 관측성 O6(DENY/ERROR 급증 감시) 근거.

**커버**: AUD-2 · SEC-3

### 14-04: ⚠️ 2-lane 분기 — 운영자(OPERATOR) 행위는 read라도 100% audit_log

```gherkin
Scenario: 운영자 read는 샘플링 대상이 아니라 전건 audit_log
  Given operator_account(actor_type='OPERATOR')가 Support Action으로 org_pk=1의 entitlement를 조회(read)한다

  When 감사 라우터가 이벤트를 분류한다

  Then read ALLOW임에도 텔레메트리 lane(샘플링)으로 강등되지 않는다
  And audit_log에 INSERT된다:
    actor_type='OPERATOR', org_pk=1, action='support_view_entitlement', result='ALLOW',
    meta_json={who:..., when:..., why:'CS 문의 대응'}
  And 운영자 행위는 컴플라이언스 audit 100%로 보존된다

  Note: operator plane·Support Action 미구현(OPER-1/SUPP-1 🔴). actor_type='OPERATOR' CHECK 어휘 값은 §D.8 DDL에 존재.
```

> operator plane·Support Action 미구현(OPER-1/SUPP-1 🔴). actor_type='OPERATOR' CHECK 어휘 값은 §D.8 DDL에 실재.

**커버**: AUD-2 · OPER-1 · SUPP-1  ·  ⚠️ 구현 후 활성화

### 14-05: ⚠️ 분산추적 — trace_id + audit_event_id 상관(meta_json 경유, 컬럼 미구현)

```gherkin
Scenario: 거부 이벤트를 분산추적 trace와 상관
  Given user_pk=10의 요청이 W3C traceparent로 trace_id='4bf92f3577b34da6a3ce929d0e0e4736'를 가진다
  And Gate B에서 entitlement EXPIRED로 차단된다 (402)

  When audit_log에 거부 이벤트가 기록된다

  Then audit_log에 INSERT된다:
    result='DENY', action='publish_video', actor_pk=10, org_pk=1,
    meta_json={trace_id:'4bf92f3577b34da6a3ce929d0e0e4736', audit_event_id:'<생성된 ULID>', gate:'B', reason:'entitlement EXPIRED'}
  And trace_id로 분산추적 시스템의 스팬과 audit row가 1:1로 상관된다
  And audit_event_id로 동일 요청의 여러 audit row(Gate별)가 묶인다

  Note: trace_id·audit_event_id 전용 컬럼은 §D.8에 미구현(AUD-3 🟡). 현행은 meta_json(JSONB) 경유.
```

> trace_id·audit_event_id 전용 컬럼은 §D.8 DDL에 없음(AUD-3 🟡 P1). meta_json(JSONB) 경유가 유일 경로 — 환각 아닌 의도적 우회.

**커버**: AUD-3  ·  ⚠️ 구현 후 활성화

### 14-06: 사람·머신 통계 분리 — actor_type 필터(HUMAN vs API_KEY)

```gherkin
Scenario: 사람 활동과 머신 활동을 actor_type으로 분리 집계
  Given audit_log 2026-05 파티션(p202605)에 HUMAN 행위 8000건, API_KEY(type='SERVICE' 사용자) 행위 2000건이 혼재한다

  When 사람 활동 통계 쿼리를 실행한다:
    SELECT COUNT(*) FROM audit_log WHERE org_pk=1 AND actor_type='HUMAN' AND created_at >= '2026-05-01'

  Then HUMAN 8000건만 집계되고 API_KEY/SYSTEM/OPERATOR는 제외된다

  When 머신 활동 통계 쿼리를 실행한다 (actor_type='API_KEY')

  Then API_KEY 2000건만 집계된다
  And identity_user.type='SERVICE' 계정의 행위는 audit_log.actor_type='API_KEY'로 매핑되어 머신 통계에 잡힌다
  And SYSTEM(배치·org 무관)은 양쪽 통계에서 분리된다
```

> audit_log.actor_type='API_KEY'가 SERVICE 사용자 행위에 매핑됨(§D.8 어휘 매핑 주석). 통계는 actor_type 필터로 사람/머신 분리.

**커버**: AUD-4

### 14-07: append-only GRANT 강제 — platform_rw의 payment_ledger UPDATE 권한거부

```gherkin
Scenario: platform_rw 계정이 payment_ledger row를 UPDATE 시도
  Given payment_ledger에 (pk=500, org_pk=1, type='CHARGE', amount_minor=990000, status='SUCCEEDED') row가 존재한다
  And 애플리케이션은 platform_rw 계정으로 연결돼 있다

  When platform_rw가 UPDATE payment_ledger SET status='FAILED' WHERE pk=500 을 실행한다

  Then Postgres가 권한 거부 오류 "permission denied for table payment_ledger"(SQLSTATE 42501)를 반환한다 (platform_rw에 payment_ledger UPDATE 미부여, §M)
  And row가 변경되지 않는다
  And 정정이 필요하면 ledger_append(INSERT only)로 REFUND/CHARGEBACK 신규 row를 추가해야 한다 (append-only)

  Note: GRANT가 위변조를 구조적으로 차단(§N.1 ✅). 회귀는 GRANT 점검 CI(§N.2).
```

> §M: platform_rw payment_ledger UPDATE 테이블단위 미부여. write는 ledger_append(INSERT only). GRANT가 위변조 차단(§N.1 ✅).

**커버**: BILL-5 · SEC-3

### 14-08: append-only GRANT 강제 — platform_rw의 billing_event UPDATE 권한거부

```gherkin
Scenario: platform_rw 계정이 billing_event를 사후 수정 시도
  Given billing_event에 (pk=700, org_pk=1, event_type='INVOICE_FAILED', plan_code='PRO_MONTHLY') row가 존재한다
  And 애플리케이션은 platform_rw 계정으로 연결돼 있다

  When platform_rw가 UPDATE billing_event SET event_type='INVOICE_PAID' WHERE pk=700 을 실행한다

  Then Postgres가 권한 거부 오류 "permission denied for table billing_event"(SQLSTATE 42501)를 반환한다 (platform_rw에 billing_event UPDATE 미부여, §M)
  And row가 변경되지 않는다
  And billing_event write는 ledger_append(INSERT only) 계정으로만 가능하다
  And lifecycle 정정은 신규 event_type row INSERT로만 표현된다 (append-only)
```

> §M ledger_append 계정이 payment_ledger·billing_event 양쪽 INSERT-only. billing_event는 FK 없음(의도적, §D.16)·구독 lifecycle 로그.

**커버**: SEC-3

### 14-09: ⚠️ append-only GRANT 강제 — platform_rw의 user_consent_event UPDATE/DELETE 권한거부

```gherkin
Scenario: platform_rw 계정이 consent row를 UPDATE/DELETE 시도
  Given user_consent_event에 (user_pk=10, consent_type='platform.marketing_email', action='GRANTED', version='2026-05-01') row가 존재한다
  And 애플리케이션은 platform_rw 계정으로 연결돼 있다

  When platform_rw가 UPDATE user_consent_event SET action='REVOKED' WHERE user_pk=10 을 실행한다

  Then Postgres가 권한 거부 오류 "permission denied for table user_consent_event"(SQLSTATE 42501)를 반환한다 (platform_rw에 user_consent_event UPDATE 미부여, §M)
  And DELETE도 어떤 계정에서도 거부된다 (전 계정 DELETE 0, §M)
  And 철회는 consent_append로 (consent_type='platform.marketing_email', action='REVOKED') 신규 row를 INSERT해 표현한다
  And 최신 created_at의 action이 현재 동의 상태(PIPA §37 철회권 증거)다

  Note: user_consent_event는 설계 확정·미구현(CON-1 ⚠️). consent_append 계정 INSERT-only.
```

> user_consent_event는 설계 확정·미구현(CON-1 ⚠️, §I.2 DDL 실재). consent_append INSERT-only, 전 계정 DELETE 0(§M).

**커버**: CON-1 · SEC-3  ·  ⚠️ 구현 후 활성화

### 14-10: ⚠️ 월 파티션 자동추가 배치 — 다음 달 PARTITION OF 선제 생성

```gherkin
Scenario: 월초 배치가 audit_log에 다음 달 파티션을 추가
  Given audit_log가 created_at 기준 선언적 RANGE 파티션(PARTITION BY RANGE)이고
  And 마지막 명시 파티션이 audit_log_2026_06 (FROM '2026-06-01' TO '2026-07-01')이다
  And 현재 2026-06-01 월초 배치가 트리거된다

  When 파티션 자동추가 배치가 실행된다:
    CREATE TABLE audit_log_2026_07 PARTITION OF audit_log
      FOR VALUES FROM ('2026-07-01 00:00:00+09') TO ('2026-08-01 00:00:00+09');

  Then audit_log_2026_07 파티션이 선제적으로 신설된다
  And 기존 audit_log_2026_01~_06 데이터는 이동·손실 없이 유지된다 (선언적 파티션은 자식 테이블 ATTACH라 기존 데이터 재배치 없음)
  And 2026-07 데이터가 들어올 때 대응 파티션이 없어 INSERT가 실패하는 사태가 방지된다

  Note: 파티션 자동추가 미구현(§D.8 ⚠️, §N.2). 다음 달이 시작되기 전 선제 실행해야 INSERT 누락이 없다.
```

> 🐬 **MySQL이라면**: `p_future(VALUES LESS THAN MAXVALUE)` 캐치올 파티션을 두고 `ALTER TABLE ... REORGANIZE PARTITION p_future INTO (...)`로 분할한다. Postgres 선언적 파티셔닝은 캐치올 없이 `CREATE TABLE ... PARTITION OF`로 다음 달 파티션을 미리 붙인다(미생성 구간 INSERT는 23514/무파티션 오류로 실패하므로 선제 생성이 관건).

> §D.8: 선언적 RANGE 파티션, 월별 자동생성 배치 미구현(미생성 시 다음 달 INSERT 실패 위험). operability O3·§N.2: 월초 `CREATE TABLE ... PARTITION OF` 배치.

**커버**: AUD-1 · RETN-1  ·  ⚠️ 구현 후 활성화

### 14-11: ⚠️ 5년 보존 후 파티션 DROP — 파기 트리거

```gherkin
Scenario: 보존기간 경과한 월 파티션을 WORM EXPORT 후 DROP하여 파기
  Given audit_log에 보존기간(5년, ISMS-P)을 넘긴 audit_log_2026_01 파티션이 존재한다
  And 해당 파티션 데이터는 더 이상 컴플라이언스 보존 의무가 없다
  And audit_log_2026_01 파티션 데이터가 외부 WORM 스토리지로 EXPORT 완료된 상태다 (§H.2)

  When 보존·파기 배치가 migrator 계정으로 실행된다:
    ALTER TABLE audit_log DETACH PARTITION audit_log_2026_01;
    DROP TABLE audit_log_2026_01;

  Then audit_log_2026_01 파티션과 그 안의 audit row 전체가 platform_db에서 물리적으로 파기된다
  And DETACH 후 DROP TABLE은 row 단위 DELETE가 아니므로 append-only/DELETE 0 원칙(§M)과 충돌하지 않는다 (DDL 권한, migrator 계정 전용)
  And 외부 WORM EXPORT 본은 별도 보존·파기 정책을 따른다 (§H.2)
  And 다른 월 파티션은 영향받지 않는다

  Note: retention 배치 미구현(operability O3 🔴). §H.2 'EXPORT 후 DROP' 원칙 준수. 파티션 DETACH/DROP은 migrator 계정(DDL)·CI/CD 내에서만.
```

> 🐬 **MySQL이라면**: `ALTER TABLE audit_log DROP PARTITION p202601` 한 문장으로 즉시 파기한다. Postgres 선언적 파티션은 `DETACH PARTITION`으로 부모에서 떼어낸 뒤 `DROP TABLE`로 자식 테이블을 제거한다(논블로킹 운영 시 `DETACH ... CONCURRENTLY` 사용 가능).

> retention 배치 미구현(operability O3 🔴). §H.2: audit_log는 외부 WORM EXPORT 선행 후에만 파티션 파기. 파티션 DETACH + DROP TABLE은 migrator 계정(DDL)·CI/CD 내에서만(§M).

**커버**: AUD-1 · RETN-1  ·  ⚠️ 구현 후 활성화

---

## Domain 15: 동의 법적 요건 — PIPA·전자서명법

### 15-01: ⚠️ P6L-01: 14세 미만 가입 시 법정대리인 동의 필수 (PIPA §22③)

```gherkin
Scenario: 만 14세 미만 사용자가 법정대리인 동의 없이 가입 시도
  Given 가입 폼에 생년월일이 입력되어 만 13세로 판별된다
  And platform.under14_guardian 동의 항목이 필수로 표시된다
  And 사용자가 법정대리인 동의 항목을 체크하지 않았다

  When 가입 API가 호출된다

  Then 400 Bad Request 반환 ("만 14세 미만은 법정대리인 동의가 필요합니다")
  And identity_user row가 INSERT되지 않는다
  And user_consent_event에 아무 row도 INSERT되지 않는다

Scenario: 만 14세 미만 사용자가 법정대리인 동의 완료 후 가입
  Given 가입 폼에 생년월일이 입력되어 만 13세로 판별된다
  And 법정대리인 동의(보호자 성명·관계·연락처)가 입력 완료됐다

  When 모든 필수 약관 + 법정대리인 동의 후 가입 API가 호출된다 (단일 트랜잭션)

  Then identity_user에 (type='HUMAN', status='ACTIVE') INSERT
  And user_consent_event에 다음 row가 INSERT된다 (보호자 값은 예시 가상값):
    consent_type='platform.under14_guardian',
    action='GRANTED',
    version='2026-05-01',
    meta_json={
      "guardian_name": "김부모",
      "guardian_relation": "PARENT",
      "guardian_contact": "+821099998888",
      "child_birth_year": 2013
    }
  And platform.terms_of_service·platform.privacy_policy·platform.third_party_firebase GRANTED row도 함께 INSERT
  And 모든 row에 ip(INET), user_agent 기록
  And row는 이후 UPDATE·DELETE 불가 (append-only, consent_append 계정)
```

> consent_type='platform.under14_guardian'는 §I.2 네임스페이스 표의 '법적 필수' 항목. 대리인 식별정보는 meta_json에 담되 PII이므로 app-level 암호화 대상(SEC-8 guardian 분류).

**커버**: CON-3  ·  ⚠️ 구현 후 활성화

### 15-02: ⚠️ P6L-02: 제3자 제공 4요건 meta_json — Toss 첫 결제 시 (PIPA §17)

```gherkin
Scenario: 첫 Toss 결제 전 제3자 제공 동의 기록
  Given org_pk=1의 결제자(payer_user_pk=10)가 Toss로 첫 구독 결제를 시도한다
  And user_consent_event에 (user_pk=10, consent_type='pg.toss_third_party') row가 없다
  And 제3자 제공 동의 항목이 결제 직전 필수로 표시된다

  When 결제자가 제3자 제공에 동의하고 결제를 진행한다

  Then user_consent_event에 INSERT:
    user_pk=10,
    consent_type='pg.toss_third_party',
    action='GRANTED',
    version='2026-05-01',
    meta_json={
      "recipient": "토스페이먼츠(주)",
      "purpose": "결제 처리 및 결제내역 관리",
      "items": ["name", "email", "payment_method"],
      "retention": "전자상거래법에 따른 5년"
    }
  And 결제 트랜잭션(payment_ledger INSERT)은 동의 row INSERT 커밋 확인 이후에 실행된다 (consent_append·ledger_append 별도 계정 — 순서는 app-level 보장)

Scenario: 제3자 제공 동의 거부 시 결제 차단
  Given 결제자가 pg.toss_third_party 동의 항목을 거부했다

  When 결제 API가 호출된다

  Then 400 Bad Request ("결제를 위한 제3자 정보제공 동의가 필요합니다")
  And payment_ledger에 row INSERT 없음
  And user_consent_event에 pg.toss_third_party row INSERT 없음
```

> 기존 P6-04는 Firebase Auth 국외이전(가입 시)을 다룸. 본 시나리오는 결제 PG(pg.toss_third_party)의 제3자 제공 — 시점·recipient가 다른 별개 케이스. §I.2 'pg.toss_third_party: 첫 Toss 결제 시 결제 시 필수'.

**커버**: CON-4  ·  ⚠️ 구현 후 활성화

### 15-03: ⚠️ P6L-03: 제3자 제공 meta_json 4요건 누락 시 JSON Schema 검증 거부

```gherkin
Scenario: 제3자 제공 동의 meta_json에 retention 누락 시 INSERT 차단
  Given consent_type='pg.stripe_third_party' 동의를 기록하려 한다
  And meta_json={"recipient":"Stripe Inc.","purpose":"해외 결제 처리","items":["email","card_last4"]} (retention 누락)

  When user_consent_event INSERT 전 JSON Schema 검증이 실행된다 (Postgres JSONB도 구조 강제 없음 — app-level 검증이 권위)

  Then 제3자 제공 스키마(required: recipient, purpose, items, retention) 검증 실패
  And 422 Unprocessable Entity ("제3자 제공 동의는 보유기간(retention)이 필수입니다")
  And user_consent_event에 row INSERT 없음

Scenario: items가 빈 배열이면 검증 실패
  Given consent_type='platform.third_party_firebase'
  And meta_json.items=[] (제공 항목 비어 있음)

  When JSON Schema 검증이 실행된다

  Then minItems=1 위반으로 검증 실패
  And user_consent_event에 row INSERT 없음

Scenario: platform.terms_of_service는 4요건 스키마 미적용
  Given consent_type='platform.terms_of_service', meta_json=NULL

  When JSON Schema 검증이 실행된다

  Then 제3자 제공 4요건 스키마가 적용되지 않는다 (third_party/pg.* consent_type만 대상)
  And meta_json=NULL이어도 INSERT 성공 (meta_json 컬럼 nullable)
```

> CON-4의 4요건(recipient/purpose/items/retention)을 CON-13의 JSON Schema 검증으로 강제. third_party/pg.* consent_type에 대해서만 4요건 필수 스키마를 적용. Postgres JSONB 컬럼은 구조 강제가 없으므로 app-level 검증이 권위.

**커버**: CON-4 · CON-13  ·  ⚠️ 구현 후 활성화

### 15-04: ⚠️ P6L-04: meta_json RFC 8785 JCS canonical 직렬화 후 저장 (row_hash 안정성)

```gherkin
Scenario: 키 순서가 뒤섞인 meta_json이 JCS canonical로 정규화되어 저장된다
  Given 입력 meta_json={"retention":"30일","recipient":"Google Firebase","items":["email","password_hash"],"purpose":"인증"} (키 비정렬)
  And consent_type='platform.third_party_firebase'

  When user_consent_event INSERT 전 RFC 8785 JCS canonicalization이 적용된다 (앱 레이어 — DB 측 JCS 정규화 내장 없음)

  Then 저장된 meta_json은 키 사전순(items, purpose, recipient, retention) + 불필요 공백 제거된 canonical form이다
  And 같은 내용을 키 순서만 바꿔 다시 canonicalize하면 byte-identical 문자열이 나온다

Scenario: canonical form 기반 row_hash 재계산이 결정론적
  Given 해시 컬럼(prev_hash·row_hash CHAR(64))이 활성화된 이후 위 canonical meta_json을 포함한 row가 저장됐고 row_hash가 기록됐다

  When 해시 검증 배치가 동일 row를 읽어 row_hash를 재계산한다

  Then JCS canonical 직렬화 결과가 저장 시점과 동일하다
  And 재계산 row_hash == 저장된 row_hash (위변조 없음으로 판정)
```

> CON-13 — meta_json을 RFC 8785 JCS(키 사전순 정렬, 공백 제거)로 canonicalize 후 저장해야 row_hash 재계산이 결정론적. 키 순서가 다른 동일 내용은 동일 canonical form·동일 row_hash를 가져야 한다.

**커버**: CON-13  ·  ⚠️ 구현 후 활성화

### 15-05: ⚠️ P6L-05: 약관 버전 변경 시 재동의 인터셉터

```gherkin
Scenario: 약관 버전이 올라간 뒤 구버전 동의 사용자가 API 호출 시 재동의 요구
  Given user_pk=10의 platform.terms_of_service 최신 GRANTED row의 version='2026-05-01'이다 (ORDER BY created_at DESC LIMIT 1 기준)
  And 현행 활성 약관 버전이 '2026-07-01'로 변경됐다

  When user_pk=10이 인증이 필요한 API를 호출한다

  Then 재동의 인터셉터가 최신 동의 version='2026-05-01' < 활성 version='2026-07-01'을 ISO 날짜 문자열 사전순 비교로 감지한다 (version VARCHAR(20))
  And 409 Conflict ("개정된 약관에 재동의가 필요합니다") 반환
  And 응답에 재동의 대상 consent_type 목록이 포함된다
  And 기존 GRANTED row는 변경되지 않는다 (append-only)

Scenario: 재동의 완료 후 신규 version GRANTED row 추가
  Given user_pk=10이 재동의 화면에서 개정 약관에 동의한다

  When 재동의 API가 호출된다

  Then user_consent_event에 신규 row INSERT:
    (user_pk=10, consent_type='platform.terms_of_service', action='GRANTED', version='2026-07-01')
  And 인터셉터가 최신 version='2026-07-01' == 활성 version으로 판정하여 이후 호출을 통과시킨다

Scenario: 동일 버전 재호출은 재동의 요구하지 않음
  Given user_pk=10의 최신 GRANTED version='2026-07-01'이고 활성 version도 '2026-07-01'이다

  When user_pk=10이 API를 호출한다

  Then 재동의 인터셉터가 통과시킨다 (409 미발생)
```

> CON-7 — 현재 동의 버전과 활성 약관 버전을 version 컬럼으로 비교. 최신 GRANTED row의 version이 현행 버전보다 낮으면 재동의 게이트. user_consent_event.version VARCHAR(20)('2026-05-01' 형식) 기준.

**커버**: CON-7  ·  ⚠️ 구현 후 활성화

### 15-06: ⚠️ P6L-06: 해시 체인 baseline — 활성화 첫 row가 genesis (PIPA 위변조 감지)

```gherkin
Scenario: 해시 컬럼 활성화 이전 NULL row는 검증 대상에서 제외
  Given user_consent_event에 prev_hash=NULL, row_hash=NULL인 과거 row들이 존재한다
  And 해시 사슬 검증 배치가 실행된다

  When 배치가 NULL 구간 row들을 평가한다

  Then computeSHA256(row) != NULL 비교로 인한 거짓 위변조 알림이 발생하지 않는다
  And NULL row들은 검증 대상에서 스킵된다

Scenario: 활성화 첫 row가 genesis로 기록된다
  Given 해시 활성화 시점 이후 첫 user_consent_event row가 INSERT된다

  When 해시 컬럼이 계산되어 기록된다

  Then 해당 row의 prev_hash = 고정 genesis seed (직전 row의 row_hash가 아님)
  And 해당 row의 row_hash = computeSHA256(row 전체 canonical 직렬화)
  And 이후 row들은 prev_hash = 직전 row의 row_hash로 연결된다

Scenario: 중간 row가 위변조되면 사슬 검증이 불일치 탐지
  Given genesis 이후 연속된 row 3건의 해시 사슬이 정상 연결돼 있다
  And 누군가 2번째 row의 action을 GRANTED→REVOKED로 직접 조작했다 (consent_append GRANT 우회 가정)

  When 월 1회 해시 사슬 검증 배치가 재계산한다

  Then 2번째 row의 재계산 row_hash != 저장된 row_hash → 위변조 감지
  And 3번째 row의 prev_hash != 2번째 재계산 row_hash → 사슬 단절 감지
  And 검증 실패가 보안 audit_log에 (actor_type='SYSTEM', org_pk=NULL, action='consent_hash_chain_breach', result='ERROR') 기록된다 (SYSTEM 이벤트 org_pk nullable 예외, §D.8)
```

> §I.2 baseline 규칙 — 활성화 이전 NULL 구간은 검증 제외, 활성화 첫 row를 genesis(prev_hash=고정 seed)로 삼아 NULL row 거짓 위변조 알림을 방지. prev_hash/row_hash CHAR(64) SHA-256.

**커버**: CON-13 · SEC-3  ·  ⚠️ 구현 후 활성화

### 15-07: ⚠️ P6L-07: user_pk 스코프 — consent는 org_pk 없음 (불변식 #3 예외)

```gherkin
Scenario: 멀티 워크스페이스 사용자의 법정대리인 동의는 user 단위 1건으로 전 org에 적용
  Given user_pk=10이 org_pk=1과 org_pk=2 두 워크스페이스의 멤버다
  And user_consent_event에 (user_pk=10, consent_type='platform.under14_guardian', action='GRANTED') 1건이 존재한다

  When org_pk=2 컨텍스트에서 user_pk=10의 법정대리인 동의 상태를 조회한다

  Then 조회 쿼리는 WHERE user_pk=10 AND consent_type='platform.under14_guardian' ORDER BY created_at DESC LIMIT 1 이다 (org_pk 필터 없음 — §I.2 현재 상태 쿼리 패턴)
  And action='GRANTED'가 반환된다 (org 무관 — user 스코프)
  And user_consent_event에는 org_pk 컬럼이 존재하지 않는다 (불변식 #3 예외, §G.1)
```

> §G.1 — user_consent_event는 user-scoped(user_pk)이며 org_pk 컬럼이 없다(불변식 #3의 명시적 예외). 동의는 사람에 귀속되지 테넌트에 귀속되지 않으므로, 멀티 워크스페이스라도 법정대리인·제3자 동의는 user_pk 1건으로 전 org에 적용된다.

**커버**: CON-3 · CON-4 · CON-13  ·  ⚠️ 구현 후 활성화

---

## Domain 16: 머신·B2B 신원 — api_key

### 16-01: ⚠️ P11-01: api_key 인증 — key_prefix 조회 + secret_hash(bcrypt) 검증

```gherkin
Scenario: 유효한 api_key 제시 시 SERVICE 신원으로 인증
  Given api_key (pk=500, org_pk=1, user_pk=900, key_prefix='ak_live_', secret_hash=bcrypt('sefull'), revoked_at=NULL, expires_at=2027-01-01)이 존재한다
  And identity_user.pk=900 (type='SERVICE', status='ACTIVE')
  And membership (user_pk=900, org_pk=1, platform_role='SERVICE', status='ACTIVE')

  When 요청이 Authorization: 'ak_live_<secret평문>'을 제시한다

  Then key_prefix='ak_live_'로 api_key 행을 조회한다
  And bcrypt.compare(<secret평문>, secret_hash)=true로 인증 성공한다
  And 요청 principal은 user_pk=900 (type='SERVICE')으로 확정된다
  And api_key.last_used_at=now() UPDATE

Scenario: secret_hash 불일치 시 인증 거부
  Given 동일 key_prefix='ak_live_' 행이 존재한다

  When 잘못된 secret 평문을 제시한다

  Then bcrypt.compare(...)=false로 401 Unauthorized 반환
  And principal이 확정되지 않아 Gate A/B/C에 진입하지 않는다 (deny-by-default, fail-closed §E.6)
  And audit_log에 INSERT: actor_type='API_KEY', api_key_pk=500, action='api_key_auth', result='DENY'
```

> schema 정합 통과. 참조 컬럼(key_prefix, secret_hash, revoked_at, expires_at, last_used_at)·audit_log(actor_type='API_KEY', api_key_pk, action, result='DENY')·identity_user.type='SERVICE'/status='ACTIVE'·membership.platform_role='SERVICE' 모두 §J·D.1·D.4·D.8 DDL에 실재. fail-closed(§E.6)·deny-by-default와 모순 없음. bcrypt('sefull')은 예시 placeholder로 무해. 수정 없음.

**커버**: AUTHN-5 · RBAC-4 · SEC-4  ·  ⚠️ 구현 후 활성화

### 16-02: ⚠️ P11-02: api_key도 사람과 동일 3-gate(Gate A/B/C) 통과

```gherkin
Scenario: 인증된 api_key의 호출이 Gate A/B/C를 동일하게 통과
  Given api_key (pk=500, org_pk=1, user_pk=900) 인증이 성공했다
  And membership (user_pk=900, org_pk=1, platform_role='SERVICE', status='ACTIVE')  -- Gate A
  And org_entitlement (org_pk=1, service='ACADEMY', status='ACTIVE')             -- Gate B
  And api_key.scopes=["lecture:read"]                                            -- Gate C

  When api_key가 lecture 조회 API(org_pk=1, service='ACADEMY')를 호출한다

  Then checkGateA(userPk=900, orgPk=1)=true (membership ACTIVE)
  And checkGateB(orgPk=1, service='ACADEMY')=true (entitlement ACTIVE)
  And Gate C에서 요청 action 'lecture:read'가 scopes에 포함되어 통과
  And audit_log에 INSERT: actor_type='API_KEY', api_key_pk=500, org_pk=1, action='lecture_read', result='ALLOW'

Scenario: scopes 밖 action은 Gate C에서 거부 (deny-by-default)
  Given api_key.scopes=["lecture:read"]만 가진다

  When 동일 api_key가 'membership:write' 권한이 필요한 API를 호출한다

  Then Gate C에서 scopes에 'membership:write'가 없어 403 Forbidden 반환
  And audit_log에 INSERT: actor_type='API_KEY', api_key_pk=500, action='membership_write', result='DENY'
```

> schema 정합 통과. §J '3-gate 동일 적용' 설계포인트와 일치. Gate A=membership.status(D.4), Gate B=org_entitlement(D.12, ACTIVE/GRACE+valid_until §E.2), Gate C=scopes(JSONB, §J). audit_log actor_type='API_KEY'·result CHECK 어휘('ALLOW'|'DENY') 실재. 403 Forbidden(Gate C deny)도 E.6 deny-by-default와 일치. 수정 없음.

**커버**: AUTHN-5 · RBAC-4  ·  ⚠️ 구현 후 활성화

### 16-03: ⚠️ P11-03: api_key의 entitlement 미충족 → Gate B 차단 (사람과 동일)

```gherkin
Scenario: org의 entitlement가 EXPIRED면 api_key도 402로 차단
  Given api_key (pk=500, org_pk=1, user_pk=900) 인증이 성공했다
  And membership (user_pk=900, org_pk=1, platform_role='SERVICE', status='ACTIVE') (Gate A 통과)
  And org_entitlement (org_pk=1, service='ACADEMY', status='EXPIRED')

  When api_key가 service='ACADEMY' API를 호출한다

  Then checkGateB(orgPk=1, service='ACADEMY')=false (status NOT IN ('ACTIVE','GRACE'))
  And 402 Payment Required 반환 (머신이라고 결제 게이트가 면제되지 않는다)
  And audit_log에 INSERT: actor_type='API_KEY', api_key_pk=500, action='lecture_read', result='DENY'
```

> schema 정합 통과. org_entitlement.status='EXPIRED'는 D.12 VARCHAR+CHECK('ACTIVE','GRACE','SUSPENDED','EXPIRED')에 실재. Gate B 판정식 status NOT IN ('ACTIVE','GRACE')→402는 §E.2·F.3 표(EXPIRED→차단 402)와 정확히 일치. audit_log 필드 실재. 수정 없음.

**커버**: AUTHN-5 · RBAC-4  ·  ⚠️ 구현 후 활성화

### 16-04: ⚠️ P11-04: allowed_ip_cidr 불일치 차단 (NIST 환경속성 — Confused Deputy 방어)

```gherkin
Scenario: 허용 CIDR 밖 IP에서 온 요청 차단
  Given api_key (pk=500, org_pk=1, allowed_ip_cidr='203.0.113.0/24')이 존재한다
  And 요청의 source IP가 198.51.100.10 (CIDR 밖)이다

  When 이 api_key로 인증된 요청이 들어온다

  Then secret_hash 검증은 성공했더라도 IP가 203.0.113.0/24에 속하지 않아 403 Forbidden 반환
  And 비즈니스 로직이 실행되지 않는다
  And audit_log에 INSERT: actor_type='API_KEY', api_key_pk=500, action='api_key_ip_denied', result='DENY', ip=<198.51.100.10 BYTEA>

Scenario: 허용 CIDR 안 IP는 통과
  Given 동일 api_key (allowed_ip_cidr='203.0.113.0/24')
  And 요청 source IP가 203.0.113.42 (CIDR 안)이다

  When 이 api_key로 인증된 요청이 들어온다

  Then IP 환경속성 검사를 통과하고 Gate A/B/C 평가로 진행한다

Scenario: allowed_ip_cidr=NULL이면 IP 제한 없음
  Given api_key (pk=501, allowed_ip_cidr=NULL)이 존재한다

  When 임의 IP에서 인증된 요청이 들어온다

  Then IP 환경속성 검사를 건너뛴다 (제한 미설정)
  And Gate A/B/C 평가로 진행한다
```

> schema 정합 통과. allowed_ip_cidr VARCHAR(50)(§J)·audit_log.ip INET(D.8) 실재. §J 설계포인트 'Confused Deputy 방어(NIST SP 800-162 환경속성)'와 일치. NULL이면 제한 미설정으로 스킵하는 동작도 nullable 컬럼 의미와 정합. 인증≠인가 원칙(인증 성공 후에도 IP 환경속성 차단)도 모순 없음. 수정 없음.

**커버**: SEC-5 · SEC-2 · ABAC-6  ·  ⚠️ 구현 후 활성화

### 16-05: ⚠️ P11-05: allowed_services 범위 밖 서비스 호출 차단

```gherkin
Scenario: allowed_services에 없는 서비스 API 호출 차단
  Given api_key (pk=500, org_pk=1, allowed_services=["ACADEMY"])이 존재한다
  And org_entitlement (org_pk=1, service='MARKET', status='ACTIVE')도 존재한다

  When 이 api_key가 service='MARKET' API를 호출한다

  Then 'MARKET'이 allowed_services에 없어 403 Forbidden 반환 (entitlement가 ACTIVE여도 키 범위 밖)
  And audit_log에 INSERT: actor_type='API_KEY', api_key_pk=500, action='market_call', result='DENY'

Scenario: allowed_services 내 서비스는 통과
  When 동일 api_key가 service='ACADEMY' API를 호출한다

  Then 'ACADEMY'가 allowed_services에 포함되어 Gate A/B/C 평가로 진행한다
```

> schema 정합 통과. allowed_services JSONB(§J)·org_entitlement.service VARCHAR+CHECK(D.12, 'ACADEMY'/'MARKET' 포함) 실재. 'entitlement ACTIVE여도 키 범위 밖이면 거부'는 키 스코프(인가)와 org 권한(Gate B)이 직교한다는 설계와 정합. 수정 없음.

**커버**: SEC-5 · AUTHN-5  ·  ⚠️ 구현 후 활성화

### 16-06: ⚠️ P11-06: revoked / expired api_key 즉시 차단

```gherkin
Scenario: revoke된 api_key 사용 시 즉시 차단
  Given api_key (pk=500, org_pk=1)이 방금 revoked_at=now(), revoked_reason='leaked_in_repo'로 UPDATE됐다

  When 이 api_key로 인증을 시도한다

  Then revoked_at IS NOT NULL이므로 401 Unauthorized 즉시 반환
  And Gate A/B/C에 진입하지 않는다
  And audit_log에 INSERT: actor_type='API_KEY', api_key_pk=500, action='api_key_auth', result='DENY', meta_json={reason:'revoked'}

Scenario: expires_at 지난 api_key 차단
  Given api_key (pk=501, expires_at=2026-05-29 23:59:59, revoked_at=NULL)이고 현재가 2026-05-30이다

  When 이 api_key로 인증을 시도한다

  Then expires_at < now()이므로 401 Unauthorized 반환
  And audit_log에 INSERT: actor_type='API_KEY', api_key_pk=501, action='api_key_auth', result='DENY', meta_json={reason:'expired'}
```

> schema 정합 통과. revoked_at·revoked_reason VARCHAR(255)·expires_at(§J)·audit_log.meta_json JSONB(D.8) 실재. 매 요청 DB 조회로 revoked_at/expires_at 즉시 평가하므로 JWT stale window 없음 — Gate A Icebox(§E.1) 한계가 api_key에는 적용되지 않는다는 설명도 정합. 현재일 2026-05-30 기준 expires_at=2026-05-29는 만료. 수정 없음.

**커버**: AUTHN-5 · SEC-5 · AUTHN-6  ·  ⚠️ 구현 후 활성화

### 16-07: ⚠️ P11-07: rotation overlap — 구·신 키 동시 유효 (무중단 교체)

```gherkin
Scenario: rotation 중 구키와 신키가 모두 인증된다
  Given api_key (pk=500, org_pk=1, user_pk=900, key_prefix='ak_old_', rotated_at=NULL, revoked_at=NULL, expires_at=2026-06-06)  -- 구키
  When 운영자가 rotation을 실행한다
  Then 신규 행 api_key (pk=501, org_pk=1, user_pk=900, key_prefix='ak_new_', revoked_at=NULL) INSERT
  And 구키(pk=500)의 rotated_at=now() UPDATE (교체됨 표시, 아직 revoke 아님)
  And 같은 org_pk=1/user_pk=900에 두 행이 동시 유효하다 (UNIQUE 제약 없음 — 다중 키 허용)

  When 클라이언트가 아직 구키 'ak_old_<secret>'를 제시한다
  Then 구키의 revoked_at=NULL, expires_at>now()이므로 인증 성공한다 (overlap window 내)

  When 동일 클라이언트가 신키 'ak_new_<secret>'를 제시한다
  Then 신키로도 인증 성공한다

Scenario: overlap grace 종료 후 구키 revoke
  Given 구키 api_key (pk=500, rotated_at=2026-05-30)이고 overlap grace 7일이 경과했다
  When 정리 배치가 실행된다
  Then 구키(pk=500) revoked_at=now(), revoked_reason='rotation_grace_expired' UPDATE
  And 이후 구키 'ak_old_<secret>' 인증은 401로 차단된다
  And 신키(pk=501)만 유효하게 남는다
```

> rotated_at·revoked_at·revoked_reason·expires_at(§J) 실재, 다중 키 허용도 schema 정합. 유일 문제는 key_prefix VARCHAR(10) 초과 prefix 값(환각성 제약 위반)이었고 10자 이내로 교체해 verified=true. unique 제약 부재 서술도 DDL과 일치.

**커버**: SEC-7 · AUTHN-5 · SEC-6  ·  ⚠️ 구현 후 활성화

### 16-08: ⚠️ P11-08: api_key 행위는 actor_type='API_KEY'로 audit_log 기록 (사람 활동과 분리)

```gherkin
Scenario: SERVICE 사용자 행위가 actor_type='API_KEY'로 기록된다
  Given api_key (pk=500, org_pk=1, user_pk=900) 인증이 성공했다
  And identity_user.pk=900은 type='SERVICE'다

  When api_key가 lecture 발행 API를 호출하고 성공한다

  Then audit_log에 INSERT:
    | 컬럼          | 값                       |
    | actor_type    | 'API_KEY'                |
    | api_key_pk    | 500                      |
    | actor_pk      | 900 (SERVICE user)       |
    | org_pk        | 1                        |
    | action        | 'publish_video'          |
    | result        | 'ALLOW'                  |
  And actor_type='HUMAN'이 아니다 (머신 행위로 분류)

Scenario: type 필터로 사람·머신 활동 통계 분리 (AUD-4)
  Given audit_log에 actor_type='HUMAN' 1000건, actor_type='API_KEY' 300건이 있다

  When SELECT count(*) FROM audit_log WHERE org_pk=1 AND actor_type='API_KEY'

  Then 머신(api_key) 활동만 집계된다 (사람 HUMAN 활동과 분리)
  And SYSTEM·OPERATOR 이벤트는 포함되지 않는다
```

> schema 정합 통과. audit_log.actor_type VARCHAR+CHECK('HUMAN','API_KEY','SYSTEM','OPERATOR')(D.8) — 'API_KEY','HUMAN' 모두 실재. D.8 주석 "SERVICE 사용자(type='SERVICE')의 행위는 'API_KEY'로 기록(어휘 매핑)"과 정확히 일치. actor_pk·api_key_pk·org_pk·action·result 컬럼 실재. 두번째 시나리오의 actor_type='API_KEY' 필터 집계가 SYSTEM/OPERATOR 제외한다는 서술도 CHECK 어휘 4종 구조와 정합. 수정 없음.

**커버**: AUD-4 · AUD-1 · SEC-7  ·  ⚠️ 구현 후 활성화

---

## Domain 17: 운영 가능성 — O1~O6

### 17-01: ⚠️ P11-01: Permission Debugger — 거부 요청에 Gate A/B/C trace 반환 (entitlement 만료)

```gherkin
Scenario: PUBLISH_VIDEO 거부 원인을 단일 trace로 설명 (Gate B에서 차단)
  Given membership (user_pk=10, org_pk=1, platform_role='MEMBER', status='ACTIVE')
  And service_membership (user_pk=10, org_pk=1, service='ACADEMY', role_code='TEACHER', status='ACTIVE')
  And org_entitlement (org_pk=1, service='ACADEMY', status='EXPIRED', valid_until='2026-04-30')

  When 운영자가 explainPermission(userPk=10, orgPk=1, service='ACADEMY', action='ACADEMY.PUBLISH_VIDEO', resource=null)를 호출한다

  Then 1초 이내에 다음 trace가 반환된다:
    | gate  | pass | reason                                                  |
    | gateA | true | membership ACTIVE                                       |
    | gateB | false| entitlement EXPIRED (valid_until=2026-04-30)            |
    | gateC | null | SKIP (Gate B에서 차단)                                  |
  And gateB.entitlement에 {status:'EXPIRED', validUntil:'2026-04-30'}가 포함된다 (operability.md O2 계약)
  And 운영자는 membership/service_membership/delegation_grant/org_entitlement를 수동 조회하지 않는다
```

**커버**: DBG-1  ·  ⚠️ 구현 후 활성화

### 17-02: ⚠️ P11-02: Permission Debugger — Gate A에서 차단 시 B/C SKIP (정지된 멤버십)

```gherkin
Scenario: 정지된 멤버십 — Gate A FAIL이면 B/C는 평가하지 않는다
  Given membership (user_pk=20, org_pk=1, status='SUSPENDED')
  And org_entitlement (org_pk=1, service='ACADEMY', status='ACTIVE', valid_until 미래)

  When 운영자가 explainPermission(userPk=20, orgPk=1, service='ACADEMY', action='ACADEMY.PUBLISH_VIDEO')를 호출한다

  Then trace가 반환된다:
    | gate  | pass | reason                          |
    | gateA | false| membership SUSPENDED            |
    | gateB | null | SKIP (Gate A에서 차단)          |
    | gateC | null | SKIP (Gate A에서 차단)          |
  And entitlement가 ACTIVE여도 gateB는 평가되지 않는다 (deny-by-default fail-fast, §E.6)
```

**커버**: DBG-1  ·  ⚠️ 구현 후 활성화

### 17-03: ⚠️ P11-03: Permission Debugger — Gate C(ReBAC) 거부 사유 노출 (role/delegation 모두 없음)

```gherkin
Scenario: 소속·결제는 통과하나 역할/위임에 capability가 없어 Gate C 거부
  Given membership (user_pk=30, org_pk=1, status='ACTIVE')
  And service_membership (user_pk=30, org_pk=1, service='ACADEMY', role_code='STUDENT', status='ACTIVE')
  And org_entitlement (org_pk=1, service='ACADEMY', status='ACTIVE', valid_until 미래)
  And delegation_grant에 (grantee_pk=30, org_pk=1, capability='ACADEMY.PUBLISH_VIDEO') 행이 없다

  When 운영자가 explainPermission(userPk=30, orgPk=1, service='ACADEMY', action='ACADEMY.PUBLISH_VIDEO')를 호출한다

  Then trace가 반환된다:
    | gate  | pass | reason                                                       |
    | gateA | true | membership ACTIVE                                            |
    | gateB | true | entitlement ACTIVE                                          |
    | gateC | false| role STUDENT has no ACADEMY.PUBLISH_VIDEO and no delegation |
  And reason에 RBAC(service_membership.role_code)와 ReBAC(delegation_grant) 양쪽 부재가 명시된다
```

**커버**: DBG-1  ·  ⚠️ 구현 후 활성화

### 17-04: ⚠️ P11-04: Operator Plane — 운영자는 테넌트 membership이 아니며 별도 MFA·전건 OPERATOR 감사

```gherkin
Scenario: FINANCE 운영자가 org_pk=1 entitlement를 조회 — 별도 평면 + 전건 감사
  Given 운영자 신원은 operator_account(role='FINANCE')로 별도 신원 평면에 존재한다 (membership/service_membership에 행 없음)
  And 해당 운영자는 별도 평면 인증의 MFA를 통과한 운영자 세션(operator_session)을 보유한다
  And membership에 이 운영자의 (user_pk, org_pk=1) 행이 존재하지 않는다

  When 운영자가 org_pk=1의 org_entitlement를 internal/ 경로로 조회한다

  Then 조회가 허용된다 (운영자 평면 권한 — FINANCE는 billing 조회, 테넌트 membership 불필요)
  And audit_log에 INSERT: (actor_type='OPERATOR', org_pk=1, action='operator_read_entitlement', result='ALLOW')
  And 일반 테넌트 API 토큰으로는 동일 cross-tenant 조회가 불가능하다 (internal/ 코드 경로 전용, cross-tenant-separation)

Scenario: MFA 미통과 운영자 세션 차단
  Given 운영자가 operator_session에서 MFA를 완료하지 않았다

  When 운영자가 org_entitlement 조회를 시도한다

  Then 401 Unauthorized 반환 (운영자 평면 MFA 강제)
  And audit_log에 INSERT: (actor_type='OPERATOR', action='operator_mfa_required', result='DENY')
```

**커버**: OPER-1 · OPER-2  ·  ⚠️ 구현 후 활성화

### 17-05: ⚠️ P11-05: Operator Plane — 최소권한 매트릭스 (AUDITOR는 audit read-only, write 거부)

```gherkin
Scenario: AUDITOR 운영자가 entitlement 강제 부여를 시도 — 권한 매트릭스 거부
  Given 운영자 operator_account(role='AUDITOR')가 MFA 통과 세션을 보유한다
  And AUDITOR 매트릭스는 audit_log read-only만 허용한다 (operator-plane.md 매트릭스, 코드 상수 OPERATOR_PERMISSION)

  When AUDITOR가 org_pk=1에 entitlement 강제 부여(Support Action)를 시도한다

  Then 403 Forbidden 반환 (AUDITOR는 write 권한 없음 — 최소권한)
  And org_entitlement 변경 없음
  And audit_log에 INSERT: (actor_type='OPERATOR', action='operator_support_action_denied', result='DENY')
```

**커버**: OPER-2  ·  ⚠️ 구현 후 활성화

### 17-06: ⚠️ P11-06: Support Action — 운영자 entitlement 강제 부여에 who/when/why 컴플라이언스 감사

```gherkin
Scenario: "결제했는데 권한 안 열림" 복구 — FINANCE 운영자가 entitlement 강제 부여
  Given org_pk=1에 payment_ledger.status='SUCCEEDED'이나 org_entitlement.status='EXPIRED'다 (webhook 유실)
  And 운영자 operator_account(role='FINANCE')가 MFA 세션을 보유한다

  When 운영자가 Support Action으로 org_pk=1에 entitlement를 강제 부여한다 (사유='webhook 유실 복구')

  Then org_entitlement UPSERT(ON CONFLICT (org_pk, product_code) DO UPDATE SET status=EXCLUDED.status, source=EXCLUDED.source, valid_until=EXCLUDED.valid_until): status='ACTIVE', source='MANUAL', valid_until 갱신
  And organization.perm_version이 bump된다 (§F.1 패턴)
  And audit_log에 INSERT: actor_type='OPERATOR', action='support_action_grant_entitlement', result='ALLOW',
      meta_json={who:operator_pk, when:'2026-05-30T...', why:'webhook 유실 복구', target_org_pk:1, source:'MANUAL'}
  And meta_json에 who/when/why 3요소가 모두 기록된다 (컴플라이언스 필수)

Scenario: 사유(why) 누락 시 Support Action 거부
  Given FINANCE 운영자가 사유 없이 Trial 연장을 시도한다

  When Support Action API가 호출된다

  Then 400 Bad Request ("Support Action에는 사유(why)가 필수입니다")
  And org_entitlement 변경 없음
  And org_subscription 변경 없음
```

**커버**: SUPP-1  ·  ⚠️ 구현 후 활성화

### 17-07: ⚠️ P11-07: 데이터 보존·파기 — 테이블별 기간 + 파기 트리거 (audit 5y 파티션 DROP, payment 7y)

```gherkin
Scenario: audit_log 5년 경과 파티션 DROP — 파기 트리거 (EXPORT 후 DROP, §H.2)
  Given audit_log에 audit_log_2021_01(2021-01) 파티션이 존재하고 created_at이 5년 이상 경과했다
  And payment_ledger에는 7년 미만 경과한 결제 원장이 있다

  When 월별 retention 배치가 실행된다

  Then audit_log_2021_01 파티션이 외부 WORM 스토리지로 EXPORT된 후 DETACH PARTITION + DROP TABLE로 제거된다 (행 단위 DELETE 아님, §H.2 WORM-EXPORT 후 DROP 규칙)
  And payment_ledger는 7년 보존 기준 미달이라 파기 대상이 아니다 (append-only, 행 보존)
  And audit_log에 INSERT: (actor_type='SYSTEM', action='retention_drop_partition', result='ALLOW', meta_json={table:'audit_log', partition:'audit_log_2021_01'})

Scenario: pg_webhook_event 90일 경과 sweeper DELETE
  Given pg_webhook_event에 created_at이 91일 전, status='PROCESSED'인 행이 존재한다

  When 일 1회 sweeper 배치가 실행된다

  Then 90일 경과한 pg_webhook_event 행이 DELETE된다 (운영 데이터, 컴플라이언스 보존 대상 아님)
  And payment_ledger·audit_log·user_consent_event는 sweeper DELETE 대상에서 제외된다 (append-only/WORM)
```

**커버**: RETN-1  ·  ⚠️ 구현 후 활성화

### 17-08: ⚠️ P11-08: Entitlement 가용성 — billing/PG 장애 시 기존 entitlement read 영향 없이 유지

```gherkin
Scenario: PG webhook 파이프라인 장애 중에도 기존 ACTIVE entitlement는 정상 통과
  Given org_entitlement (org_pk=1, service='ACADEMY', status='ACTIVE', valid_until='2026-07-01')
  And PG webhook 수신/처리 파이프라인이 장애로 중단됐다 (pg_webhook_event INSERT 실패)
  And billing 도메인 write 경로가 일시 불가능하다

  When checkGateB(orgPk=1, service='ACADEMY')가 호출된다

  Then true 반환 (Gate B 통과)
  And Gate B는 org_entitlement만 읽고 payment_ledger·pg_webhook_event를 직접 조회하지 않는다 (불변식 #4)
  And 기존 entitlement read는 billing/PG 장애와 구조적으로 격리되어 영향이 없다 (auth-projection)

Scenario: 장애로 갱신이 지연되면 valid_until 복합 체크가 2차 방어
  Given org_entitlement (status='ACTIVE', valid_until='2026-05-29') — webhook 지연으로 status 미전환
  And 현재 시각이 2026-05-30이다

  When checkGateB(orgPk=1, service='ACADEMY')가 호출된다

  Then false 반환 (status는 ACTIVE이나 valid_until < now())
  And 402 Payment Required 반환 (배치 실패 시 영구 무료 방지, 불변식 #9)
```

**커버**: AVAIL-1  ·  ⚠️ 구현 후 활성화

### 17-09: ⚠️ P11-09: 사용량 가시성 — usage_snapshot으로 feature_limits 대비 498/500 노출

```gherkin
Scenario: 학생 수 498/500 가시화 — 백오피스가 현재 사용량을 안다
  Given org_entitlement (org_pk=1, service='ACADEMY', feature_limits={"students":500})
  And 서비스(academy)가 주기적으로 집계를 push한다

  When usage_snapshot에 (org_pk=1, service='ACADEMY', metric='students', period='2026-05', used=498, limit=500, source_ts=now()) UPSERT된다

  Then 백오피스가 usage_snapshot 조회로 used=498 / limit=500을 즉시 확인한다
  And limit=500은 org_entitlement.feature_limits.students와 일치한다 (feature_limits가 런타임 권위 SSOT, 불변식 #10 — usage_snapshot.limit은 가시성용 복사값)
  And 실시간 차단 카운터는 서비스측이며 platform usage_snapshot은 가시성·과금 용도다 (O4, ABAC-3)

Scenario: 한도 근접 가시성 — 임계 노출 (알림 임계치는 ops 제품 소관)
  Given usage_snapshot (used=498, limit=500)

  When 백오피스가 한도 대비 사용률을 조회한다

  Then used/limit = 99.6%가 노출된다
  And platform_db는 데이터를 노출할 뿐 알림 발송·임계치 판정은 하지 않는다 (ops 제품 소관, operability.md 범위 규율)
```

**커버**: USAGE-1  ·  ⚠️ 구현 후 활성화

### 17-10: ⚠️ P11-10: 마지막 platform OWNER lockout 방지 — FOR UPDATE 앱 가드 + 2차 모니터

```gherkin
Scenario: org의 마지막 OWNER가 본인을 MEMBER로 강등 시도 — FOR UPDATE 가드 차단
  Given org_pk=1에 membership WHERE platform_role='OWNER' AND status='ACTIVE'가 user_pk=10 1건만 존재한다

  When user_pk=10이 본인 membership.platform_role을 'MEMBER'로 변경 시도한다

  Then 트랜잭션이 SELECT COUNT(*) ... WHERE platform_role='OWNER' AND status='ACTIVE' FOR UPDATE로 OWNER 수를 잠그고 확인한다 (§N.3 1차 가드)
  And 400 Bad Request ("마지막 OWNER는 강등할 수 없습니다")
  And membership 변경 없음
  And audit_log에 INSERT: (actor_type='HUMAN', org_pk=1, action='attempted_owner_lockout', result='DENY')

Scenario: 두 OWNER 동시 강등 레이스 — FOR UPDATE로 직렬화
  Given org_pk=1에 OWNER = user_pk=10 + user_pk=11 2명
  And 두 요청이 서로를 동시에 강등 시도한다

  When 두 트랜잭션이 FOR UPDATE 락을 경합한다

  Then 먼저 락을 얻은 트랜잭션만 강등에 성공한다 (1명 강등 후 OWNER=1명)
  And 두 번째 트랜잭션은 OWNER 수=1을 보고 거부된다 (lockout 방지)
  And 최소 1명의 ACTIVE OWNER가 항상 보장된다

Scenario: 2차 모니터 — OWNER 0건 org 탐지 (앱 가드 우회 대비)
  Given 마이그레이션/직접 SQL로 organization.status='ACTIVE'인 org_pk=2가 ACTIVE OWNER 0건 상태가 됐다

  When 정기 무결성 모니터 배치가 실행된다

  Then OWNER가 없는 org_pk=2가 탐지된다 (CLOSED/SUSPENDED org는 제외, §N.3 거짓 알림 방지)
  And audit_log에 INSERT: (actor_type='SYSTEM', action='owner_lockout_detected', result='ERROR', meta_json={org_pk:2, active_owner_count:0})
```

**커버**: OWN-1 · RBAC-6  ·  ⚠️ 구현 후 활성화

---

## Domain 18: 불변식·확장성·안전장치

### 18-01: INV-01: firebase_uid는 조회 키 — PK·FK·참조키가 아님

```gherkin
Scenario: firebase_uid 변경/소실이 사용자 동일성·참조 무결성을 깨지 않는다
  Given identity_user.pk=10 (firebase_uid='fb_kim_001')이 org_pk=1의 OWNER다
  And membership(user_pk=10, org_pk=1), service_membership(user_pk=10, org_pk=1, service='ACADEMY'),
      delegation_grant(grantor_pk=10), audit_log(actor_pk=10) 행들이 모두 pk=10을 참조한다
  And 이 행들 중 어느 것도 firebase_uid 컬럼을 갖지 않는다

  When 사용자가 Firebase에서 인증 vendor를 교체해 firebase_uid='fb_kim_new_999'로 UPDATE된다
     And 다른 사용자가 탈퇴 후 hard anonymize로 firebase_uid=NULL이 된다

  Then identity_user.pk=10은 불변이고 위 모든 자식 행은 여전히 pk=10으로 유효하게 JOIN된다
  And firebase_uid는 어떤 FK 제약(C.3 관계표)에도 나타나지 않는다 (조회용 인덱스 uq_identity_user_firebase_uid 뿐)
  And firebase_uid=NULL인 행이 여러 건이어도 PK·FK 무결성에 영향이 없다

  Note: firebase_uid는 VARCHAR(128) 인증 연결 키일 뿐 — 모든 내부 JOIN은 BIGINT pk로 수행(B. 식별자 체계 / 불변식 #1).
```

> schema 정합 통과. D.1 firebase_uid VARCHAR(128)·uq_identity_user_firebase_uid 유니크 인덱스·hard anonymize 시 NULL(설계 포인트) 모두 실재. C.3 관계표에 firebase_uid FK 없음 확인. 환각 없음.

**커버**: AUTHN-1 · USR-2 · 불변식 #1

### 18-02: INV-02: service CHECK 위반 — 소문자·오타 service 값 INSERT 거부 (4개 테이블)

```gherkin
Scenario Outline: 정규화 안 된 service 문자열은 fail-closed로 거부된다
  Given org_pk=1, user_pk=10이 유효하게 존재한다

  When <테이블>에 service='<잘못된값>' 행을 INSERT한다

  Then CHECK constraint <제약명> 위반(SQLSTATE 23514)으로 INSERT가 실패한다
  And 해당 행은 생성되지 않는다 (Gate B 핫패스로 잘못된 service가 새지 않음)

  Examples:
    | 테이블             | 제약명                  | 잘못된값   |
    | service_membership | chk_svc_mbr_service     | academy    |
    | service_membership | chk_svc_mbr_service     | ACADMY     |
    | membership_invite  | chk_invite_service      | Market     |
    | product            | chk_product_service     | store      |
    | org_entitlement    | chk_entitlement_service | AGENT_     |

Scenario: 허용 어휘는 통과한다
  When service_membership에 service='ACADEMY', role_code='TEACHER' 행을 INSERT한다
  Then INSERT 성공 (대문자 정규 어휘 'ACADEMY','MARKET','AGENT','YOUTUBE','STORE'만 통과)

  Note: service는 ENUM이 아니라 VARCHAR(50)+CHECK(불변식 #5/D6) — 무마이그레이션 확장을 위한 의도적 설계이나, 현재 어휘는 4개 테이블 CHECK가 fail-closed로 강제한다.
```

> 4개 제약명(chk_svc_mbr_service D.4a · chk_invite_service D.5 · chk_product_service D.9 · chk_entitlement_service D.12) 모두 실재하고 IN 어휘 5종 일치. service_membership.role_code 컬럼 실재. 환각 없음.

**커버**: RBAC-2 · EXT-4 · 불변식 #5

### 18-03: INV-03: feature_limits가 런타임 최종 권위 — product_feature·plan_definition 직접 조회 금지

```gherkin
Scenario: 런타임 한도 판정은 org_entitlement.feature_limits만 읽는다
  Given plan_definition(plan_code='PRO').default_limits = {"daily_uploads": 10}
  And product_feature(feature_key='daily_uploads').limit_value = 10
  And org_pk=1의 org_entitlement(service='ACADEMY', plan_code='PRO').feature_limits = {"daily_uploads": 6}
     (운영자 Support Action으로 이 org만 6으로 하향 조정된 상태)

  When 런타임 한도 평가가 org_pk=1의 daily_uploads 한도를 조회한다

  Then org_entitlement.feature_limits.daily_uploads = 6이 적용된다
  And plan_definition.default_limits(10)·product_feature.limit_value(10)는 조회되지 않는다 (불변식 #10)
  And entitlement 최초 생성(F.1 UPSERT) 시점에만 default_limits가 초기값으로 복사됐을 뿐, 런타임 권위는 feature_limits다

Scenario: entitlement에 feature_limits가 비어도 plan_definition으로 폴백하지 않는다
  Given org_pk=2의 org_entitlement.feature_limits = NULL
  When 런타임이 daily_uploads 한도를 조회한다
  Then plan_definition·product_feature로 우회 조회하지 않고, feature_limits NULL은 '키 미정의'로 평가된다 (직접 조회 금지)

  Note: product_feature·plan_definition.default_limits는 생성 시 초기값 복사 전용 — 런타임 직접 조회 금지(D.12 설계 포인트).
```

> plan_definition.default_limits(D.15)·product_feature.limit_value(D.10)·org_entitlement.feature_limits·plan_code·service(D.12) 컬럼 모두 실재. D.12 설계 포인트 'feature_limits 최종 권위, 두 테이블 런타임 직접 조회 금지(불변식 #10)' 및 F.1 UPSERT 초기값 복사와 정확히 일치. 환각 없음.

**커버**: ABAC-3 · BILL-2 · 불변식 #10

### 18-04: EXT-01: 새 서비스 onboarding — service CHECK 4곳 동반 ALTER, 1곳 누락 시 fail-closed 거부

```gherkin
Scenario: FITNESS 서비스 추가는 platform 스키마 신규 테이블 0 — service CHECK 4곳 1줄 확장만으로 동작
  Given platform_db에 'FITNESS' service 어휘가 아직 없다

  When 마이그레이션이 org_entitlement·product·service_membership·membership_invite 4개 테이블의
       service CHECK를 IN('ACADEMY','MARKET','AGENT','YOUTUBE','STORE','FITNESS')로 ALTER한다
     And 신규 platform 테이블·컬럼은 추가하지 않는다

  Then 변경된 platform DDL은 CHECK 4줄뿐이고 신규 테이블 = 0 (EXT-1)
  And org_entitlement에 service='FITNESS' 행 INSERT가 성공한다
  And service_membership에 service='FITNESS', role_code='COACH' (코드 ROLE_PERMISSION[FITNESS] 권위) 행이
     무마이그레이션으로 INSERT된다

Scenario: 4곳 중 한 곳이라도 CHECK 확장을 빠뜨리면 해당 경로가 fail-closed로 막힌다
  Given org_entitlement·product·membership_invite는 'FITNESS'를 추가했으나
        service_membership.chk_svc_mbr_service는 빠뜨렸다

  When service_membership에 service='FITNESS', role_code='COACH' 행을 INSERT한다

  Then chk_svc_mbr_service 위반으로 INSERT가 실패한다
  And FITNESS 멤버는 도메인 역할을 받지 못해 Gate C가 차단된다

  Note: 동반 대상 4곳(§K 체크리스트). 위임 capability를 쓰면 delegation_grant.chk_capability에
        'FITNESS.<action>' 추가도 동반 — service_membership.role_code(무마이그레이션)와 capability CHECK(ALTER 필요)의 EXT-2 비대칭.
```

> §K(service 확장 방법) 4곳 동반 ALTER 체크리스트(org_entitlement·product·service_membership·membership_invite)와 정확히 일치. role_code 무마이그레이션 vs chk_capability ALTER 필요(D.6 ⚠️)의 EXT-2 비대칭도 §K 주석과 일치. 환각 없음.

**커버**: EXT-1 · EXT-6 · 불변식 #5

### 18-05: SAFE-01: self-delegation 차단 — chk_no_self_delegation (grantor=grantee 거부)

```gherkin
Scenario: 사용자가 자기 자신에게 capability를 위임할 수 없다
  Given identity_user.pk=10 (DIRECTOR)이 org_pk=1에 존재한다

  When delegation_grant에 grantor_pk=10, grantee_pk=10, org_pk=1,
       capability='ACADEMY.MANAGE_MEMBERS', status='ACTIVE' 행을 INSERT한다

  Then CHECK constraint chk_no_self_delegation (grantor_pk <> grantee_pk) 위반(SQLSTATE 23514)으로 INSERT가 실패한다
  And delegation_grant 행이 생성되지 않는다

Scenario: 서로 다른 사용자 간 위임은 허용된다
  Given DIRECTOR(grantor_pk=10), TEACHER(grantee_pk=20)이 org_pk=1에 존재한다
  When delegation_grant에 grantor_pk=10, grantee_pk=20, capability='ACADEMY.MANAGE_MEMBERS' INSERT
  Then INSERT 성공 (grantor≠grantee)

  Note: chk_no_self_delegation은 org_relation의 chk_no_self_ref(REBAC-6)와 대칭인 자기참조 차단(D.6).
        권한 자가확대(self-escalation)를 DB 레벨에서 구조적으로 거부.
```

> D.6 chk_no_self_delegation CHECK (grantor_pk <> grantee_pk)·status VARCHAR+CHECK('ACTIVE','REVOKED')·capability CHECK 목록에 'ACADEMY.MANAGE_MEMBERS' 포함 모두 실재. D.7 org_relation chk_no_self_ref 대칭도 정확. 환각 없음.

**커버**: REBAC-1 · REBAC-6 · 불변식 #5

### 18-06: ⚠️ SAFE-02: outbox/webhook sweeper — 종료 상태 경과 행만 DELETE (outbox SENT+30일 · webhook PROCESSED+90일)

```gherkin
Scenario: outbox_event sweeper는 SENT 후 30일 경과 행만 삭제한다
  Given outbox_event에 (pk=1, status='SENT', sent_at=40일 전), (pk=2, status='SENT', sent_at=5일 전),
        (pk=3, status='PENDING', created_at=40일 전), (pk=4, status='FAILED', created_at=40일 전)이 있다

  When 일 1회 sweeper 배치가 실행된다

  Then pk=1만 DELETE된다 (SENT AND sent_at < now()-interval '30 days')
  And pk=2(미경과)·pk=3(PENDING)·pk=4(FAILED)는 보존된다 (미전송·실패 이벤트 유실 방지)

Scenario: pg_webhook_event sweeper는 PROCESSED 후 90일 경과 행만 삭제한다
  Given pg_webhook_event에 (pk=1, status='PROCESSED', processed_at=100일 전),
        (pk=2, status='PROCESSED', processed_at=10일 전), (pk=3, status='FAILED', created_at=100일 전)이 있다

  When 일 1회 sweeper 배치가 실행된다

  Then pk=1만 DELETE된다 (PROCESSED AND processed_at < now()-interval '90 days')
  And pk=3(FAILED)은 보존된다 — Webhook Replay 복구 경로(O2)·reconciliation 근거로 살려둔다

  Note: 현재 sweeper 미구현(§N.2 갭) — 미구현 시 종료 이벤트 무한 누적. outbox 종료 상태는 DDL상 'SENT'
        (operability O3의 'PROCESSED 후 30일' 표현은 pg_webhook_event 기준 어휘이며 outbox는 SENT 컬럼 sent_at으로 판정).
        append-only 4종(audit_log·payment_ledger·billing_event·user_consent_event)은 sweeper 대상이 아니라 보존 매트릭스(audit 5년 등) 소관.
```

> outbox_event.status VARCHAR+CHECK('PENDING','SENT','FAILED')·sent_at(D.19), pg_webhook_event.status VARCHAR+CHECK('RECEIVED','PROCESSED','SKIPPED','FAILED')·processed_at(D.18) 모두 실재. outbox 종료상태가 SENT(PROCESSED 아님)임을 정확히 구분한 note가 §N.2/O3와 일치. sweeper 미구현(§N.2)·append-only 4종 보존 매트릭스 분리 모두 정합. implemented=false 적절. 환각 없음.

**커버**: ARCH-6 · RETN-1 · BILL-4  ·  ⚠️ 구현 후 활성화

### 18-07: ⚠️ SAFE-03: store_purchase org_pk 격리 — 타 org 구매 행 BOLA 차단

```gherkin
Scenario: store_db.store_purchase는 org_pk로 격리되어 타 테넌트 구매를 노출하지 않는다
  Given org_pk=1 컨텍스트의 buyer가 있다
  And store_purchase에 (pk=500, org_pk=2, product_pk=900, buyer_user_pk=77)이 존재한다 (다른 org 구매)

  When org_pk=1 컨텍스트에서 store_purchase.pk=500을 조회한다

  Then 쿼리에 WHERE org_pk=1 AND pk=500이 강제되어 row가 없다
  And 404 NotFoundException이 반환된다 (존재 여부 비공개, OWASP API1/BOLA)
  And audit_log에 (org_pk=1, result='DENY', action='get_store_purchase') INSERT

Scenario: store_purchase의 org_pk는 NOT NULL이라 격리 키 누락 행이 생성될 수 없다
  When org_pk 없이 store_purchase 행을 INSERT 시도한다
  Then org_pk NOT NULL 위반으로 INSERT 실패 (불변식 #3 — 도메인 테이블 격리 키 필수)

  Note: store_purchase.org_pk는 market_order와 명명 통일된 격리 키(L.4). store_db는 store_rw 전용 계정만
        접근하며 platform_db·academy_db와 peer 접근 금지(A.1 cross-DB 방향 규칙). 본 시나리오는 store_db 미구현이라 implemented=false.
```

> L.4 store_purchase 컬럼 org_pk NOT NULL·product_pk·buyer_user_pk 모두 실재(명칭 정확). audit_log.result VARCHAR+CHECK('ALLOW','DENY','ERROR')·org_pk nullable·action 컬럼 실재. §M store_rw 전용 계정·A.1 peer 금지·불변식 #3 격리 키 모두 정합. 설계 확정(미구현)이므로 implemented=false 적절. 환각 없음.

**커버**: TEN-1 · TEN-2 · 불변식 #3  ·  ⚠️ 구현 후 활성화

### 18-08: SAFE-04: append-only 4종 — GRANT가 위변조를 구조적으로 차단 (주석 아님)

```gherkin
Scenario: platform_rw 계정은 append-only 4종에 UPDATE·DELETE를 할 수 없다
  Given platform_rw 계정으로 audit_log.pk=9999, payment_ledger.pk=8888 행이 존재한다

  When platform_rw로 UPDATE audit_log SET result='ALLOW' WHERE pk=9999를 실행한다
     And platform_rw로 UPDATE payment_ledger SET amount_minor=0 WHERE pk=8888를 실행한다
     And 어떤 계정으로든 DELETE payment_ledger를 실행한다

  Then UPDATE는 테이블 단위 GRANT 미부여로 거부된다 (audit_log·payment_ledger·billing_event·user_consent_event)
  And DELETE는 모든 계정에서 권한 없음으로 거부된다 (전 계정 DELETE 0)
  And 행 내용이 변경되지 않는다 — 위변조 방어가 주석이 아니라 GRANT로 강제됨(§M)

Scenario: append-only 4종 write는 전용 INSERT-only 계정으로만 가능하다
  When audit_append 계정으로 audit_log INSERT, ledger_append로 payment_ledger·billing_event INSERT,
       consent_append로 user_consent_event INSERT
  Then 각 INSERT는 성공한다 (해당 테이블 INSERT-only 권한)

  Note: user_consent_event는 consent 미구현(§I)이라 해당 절은 구현 후 활성화. 회귀는 GRANT 점검 CI(§N.2)로
        신규 append-only 테이블 추가 시 platform_rw UPDATE 재유입을 차단한다.
```

> §M GRANT 매트릭스와 정확히 일치: append-only 4종(audit_log·payment_ledger·billing_event·user_consent_event), 전용 INSERT-only 계정(audit_append=audit_log, ledger_append=payment_ledger+billing_event, consent_append=user_consent_event), platform_rw 테이블단위 UPDATE 미부여, 전 계정 DELETE 0. audit_log.result·payment_ledger.amount_minor 컬럼 실재. user_consent_event 미구현 note(§I)·GRANT 회귀 CI(§N.2) 정합. 환각 없음.

**커버**: SEC-3 · AUD-1 · BILL-5

---

## Domain 19: 갭 보강 — PG 추상화 · 확장성 중립

### 19-01: PG provider 추상화 — MANUAL은 webhook 없이, 나머지는 동일 원장 경로

```gherkin
Scenario: MANUAL 결제는 webhook 없이 직접 entitlement 부여
  Given org_subscription.pg_provider 옵션은 VARCHAR+CHECK(pg_provider IN ('TOSS','STRIPE','PAYPAL','MANUAL'))이다
  And pg_webhook_event.pg_provider는 VARCHAR+CHECK(pg_provider IN ('TOSS','STRIPE','PAYPAL')) — MANUAL 미포함이다

  When 운영자가 pg_provider='MANUAL'로 결제를 기록한다

  Then payment_ledger에 (pg_provider='MANUAL', status='SUCCEEDED') INSERT
  And pg_webhook_event에는 대응 row가 없다 (수동 결제는 webhook 부재 — pg_provider CHECK 어휘에서 의도적 제외)
  And org_entitlement는 동일 트랜잭션으로 활성화된다 (provider 무관 동일 흐름)

Scenario: TOSS·STRIPE 결제는 동일 payment_ledger 경로로 추상화
  Given 동일 org가 한 번은 pg_provider='TOSS', 한 번은 'STRIPE'로 결제한다

  Then 두 결제 모두 payment_ledger에 pg_provider 값만 달리 기록된다
  And idempotency_key UNIQUE로 각각 멱등 보장된다
  And Gate B는 pg_provider를 보지 않는다 (org_entitlement.status만 — PG 추상화)
```

**커버**: BILL-8

### 19-02: ⚠️ 코어 인가의 서비스 어휘 — capability CHECK는 의도적 잔존 (EXT-2)

```gherkin
Scenario: 2번째 서비스 capability는 코어 CHECK에 막힘 — 의도적 트레이드오프
  Given delegation_grant.chk_capability는 현재 'ACADEMY.*' 6종만 허용한다

  When 'MARKET.PUBLISH_ITEM' capability로 delegation_grant INSERT를 시도한다

  Then CHECK constraint 위반(SQLSTATE 23514)으로 거부된다 (코어에 MARKET 어휘 미등록)
  And 해결은 §K ALTER로 'MARKET.*' 추가(온라인 1줄, 테이블 락 없음)
  And role→action 매핑 자체는 ROLE_PERMISSION 코드 상수가 권위라 DB 스키마 무변경 (RBAC-3)

  Note: EXT-2 의도적 트레이드오프 — 코어가 capability 어휘를 아는 건 service-extensibility 결정대로 수용.
        2번째 서비스 churn 시 코드/데이터 기반(Option B/C)으로 전환 트리거.
```

**커버**: EXT-2  ·  ⚠️ 의도적 트레이드오프

### 19-03: ⚠️ Gate B 서비스 중립성 — 명시 service로 비-ACADEMY 동일 동작 (EXT-5)

```gherkin
Scenario: 명시 service로 MARKET entitlement도 동일하게 게이트 통과
  Given checkGateB(orgPk, service)의 기본값은 'ACADEMY'다
  And org_pk=1에 org_entitlement (service='MARKET', status='ACTIVE')가 있다

  When checkGateB(orgPk=1, service='MARKET')를 명시 호출한다

  Then idx_org_service_status (org_pk, service, status, valid_until)로 MARKET entitlement를 조회한다
  And status='ACTIVE'라 통과한다 (ACADEMY 기본값에 편향되지 않음)

Scenario: service 미전달 시 ACADEMY 기본값 — 잔여 편향(중립화 예정)
  Given checkGateB(orgPk=1) — service 인자 미전달

  Then service='ACADEMY'로 평가된다 (불변식 #8 — 신규 서비스는 service 명시 필수)

  Note: EXT-5 잔여 편향. 2번째 서비스 도입 시 기본값 제거(중립화) 트리거 — service-extensibility.
```

**커버**: EXT-5  ·  ⚠️ 2번째 서비스 시 중립화

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
| C-5 youtube_channel의 `org_pk` 컬럼 확인 필요 | 멀티테넌시 불변식 | 서비스 스키마 설계 시 `org_pk NOT NULL` 보강 |

---

## 요구사항 추적 매트릭스

> 요구 ID → 커버 시나리오(Part A `Pn-NN` + Part B `NN-NN`). ⛔/설계·정책 항목은 BDD 불요(사유 표기).

| 요구 ID | 커버 시나리오 |
|---|---|
| USR-1 | P1-01, 11-02, 11-09 |
| USR-2 | 18-01 |
| USR-3 | 설계/정책 — type 모델(P1-01 간접) |
| USR-4 | 11-09 |
| USR-5 | P9-05, 11-08 |
| USR-6 | P9-04, 11-06, 11-08 |
| USR-7 | P1-01, P1-02 |
| USR-8 | 11-01, 11-02 |
| USR-9 | ⛔→🟡 제품결정 |
| USR-10 | P10-01, P10-02 |
| USR-11 | P10-03 |
| USR-12 | P9-01, P9-03 |
| AUTHN-1 | 11-01, 11-03, 11-04, 18-01 |
| AUTHN-2 | 설계/정책 — Firebase정책 |
| AUTHN-3 | 11-03 |
| AUTHN-4 | 11-05 |
| AUTHN-5 | 16-01, 16-02, 16-03, 16-05, 16-06, 16-07 |
| AUTHN-6 | P2-02, P9-04, 11-06, 16-06 |
| AUTHN-7 | 11-07 |
| AUTHN-8 | 11-04 |
| RBAC-1 | P1-01, P1-02, P1-03 |
| RBAC-2 | 18-02 |
| RBAC-3 | 12-10 |
| RBAC-4 | 16-01, 16-02, 16-03 |
| RBAC-5 | P2-01, P2-03, P8-01 |
| RBAC-6 | P2-04, 17-10 |
| RBAC-7 | ⛔ 커스텀롤 트리거 시 |
| RBAC-8 | ❓ 배포SLA 열린결정 |
| ABAC-1 | 12-06 |
| ABAC-2 | 12-06 |
| ABAC-3 | 12-07, 18-03 |
| ABAC-4 | P4-03 |
| ABAC-5 | 12-08 |
| ABAC-6 | 16-04 |
| ABAC-7 | 12-09 |
| REBAC-1 | P3-01, P3-02, P3-03, 12-05, 18-05 |
| REBAC-2 | P3-01 |
| REBAC-3 | P3-02 |
| REBAC-4 | 12-01, 12-02, 12-04 |
| REBAC-5 | 12-04 |
| REBAC-6 | 12-03, 18-05 |
| REBAC-7 | ⛔ Zanzibar 보류 |
| REBAC-8 | 12-05 |
| BILL-1 | 13-01, 13-02, 13-08 |
| BILL-2 | P4-03, 13-07, 18-03 |
| BILL-3 | P4-01, 13-01, 13-08 |
| BILL-4 | P4-01, P4-02, 13-05, 18-06 |
| BILL-5 | P4-05, 14-07, 18-08 |
| BILL-6 | P4-04 |
| BILL-7 | 13-03, 13-04 |
| BILL-8 | 19-01 |
| BILL-9 | P8-03, 13-05 |
| BILL-10 | P1-01, 13-06 |
| TEN-1 | P9-03, 18-07 |
| TEN-2 | P7-01, 18-07 |
| TEN-3 | 설계/정책 — internal/ 분리규칙 |
| TEN-4 | 설계/정책 — CI린트 |
| TEN-5 | P7-02 |
| TEN-6 | P7-03 |
| TEN-7 | 설계/정책 — T1~T4 트리거 |
| TEN-9 | P10-04 |
| AUD-1 | P5-01, P5-02, P5-03, 14-10, 14-11, 16-08, 18-08 |
| AUD-2 | 14-01, 14-02, 14-03, 14-04 |
| AUD-3 | 14-05 |
| AUD-4 | 14-06, 16-08 |
| CON-1 | P6-01, P6-02, 14-09 |
| CON-2 | P6-02 |
| CON-3 | 15-01, 15-07 |
| CON-4 | P6-04, 15-02, 15-03, 15-07 |
| CON-5 | P6-03 |
| CON-6 | P6-03 |
| CON-7 | 15-05 |
| CON-8 | P9-01 |
| CON-9 | P9-01, P9-02 |
| CON-10 | P9-04 |
| CON-12 | P8-04 |
| CON-13 | 15-03, 15-04, 15-06, 15-07 |
| SEC-1 | P7-01 |
| SEC-2 | 16-04 |
| SEC-3 | P5-02, 14-03, 14-07, 14-08, 14-09, 15-06, 18-08 |
| SEC-4 | 16-01 |
| SEC-5 | 16-04, 16-05, 16-06 |
| SEC-6 | 16-07 |
| SEC-7 | 16-07, 16-08 |
| SEC-8 | 설계/정책 — 데이터분류 정책 |
| NFR-1 | P3-02 |
| NFR-2 | 설계/정책 — 무효화 폭(P8-01 간접) |
| NFR-3 | 설계/정책 — 확장 로드맵 |
| NFR-4 | P8-01 |
| NFR-6 | P8-01 |
| ARCH-1 | 설계/정책 — 설계규칙 |
| ARCH-2 | 설계/정책 — 설계규칙(원자성→13-01) |
| ARCH-3 | 설계/정책 — 설계규칙 |
| ARCH-4 | 설계/정책 — 패키지경계 |
| ARCH-5 | 설계/정책 — FK금지규칙 |
| ARCH-6 | 18-06 |
| OPS-1 | 설계/정책 — 모듈경계 원칙 |
| OPS-2 | P8-02 |
| OPS-3 | ⛔ HTTP분리 시 |
| OPS-4 | 설계/정책 — →SEC-6 |
| EXT-1 | 18-04 |
| EXT-2 | 19-02 |
| EXT-3 | 12-10 |
| EXT-4 | 18-02 |
| EXT-5 | 19-03 |
| EXT-6 | 18-04 |
| OPER-1 | 14-04, 17-04 |
| OPER-2 | 17-04, 17-05 |
| SUPP-1 | 14-04, 17-06 |
| DBG-1 | 17-01, 17-02, 17-03 |
| RETN-1 | 14-10, 14-11, 17-07, 18-06 |
| AVAIL-1 | 17-08 |
| OWN-1 | 17-10 |
| USAGE-1 | 17-09 |
