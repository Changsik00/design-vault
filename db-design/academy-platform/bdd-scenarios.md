# BDD Scenarios — 사용자 행동 시나리오

> 작성일: 2026-05-23
> **본 문서는 v0.1 MVP 의 사용자 검증 기준이다.** 코드 시작 전 사용자 합의 → 각 spec 의 acceptance criteria 로 사용 → 통합 테스트로 자동 실행.
> Gherkin 형식 (English keywords, Korean content). 도구 자동화 (Cucumber.js + Playwright) 는 v0.5+ 검토.

---

## 0. 사용 방법

### 0.1 본 문서의 역할

1. **코드 작성 전**: 학원장 / 강사 페르소나가 읽고 "내가 기대하는 흐름이 맞다" 합의
2. **spec 작성 시**: 각 spec 의 acceptance criteria 로 시나리오 인용
3. **통합 테스트 작성 시**: 시나리오를 vitest + supertest 기반으로 자동 실행 (도구는 자유, 시나리오는 일치)
4. **PR review 시**: PR 본문에서 본 문서의 해당 시나리오 ID 참조 → 누락 검증

### 0.2 Gherkin 키워드 (English)

- `Feature:` — 기능 단위 그룹
- `Background:` — 모든 시나리오 공통 가정
- `Scenario:` — 개별 시나리오
- `Given` — 사전 조건 (상태)
- `When` — 행동 (event)
- `Then` — 결과 (검증)
- `And`, `But` — 직전 키워드와 같은 절 (가독성 향상)

### 0.3 페르소나 (이름 매핑)

본 문서에 일관되게 사용되는 인물:

| 이름 | 역할 | 학원 | firebase_uid |
|---|---|---|---|
| **김지영** | 학원장 (DIRECTOR) | A학원 (한울학원) | `fb_uid_director_a01` |
| **박민준** | 강사 (TEACHER) | A학원 | `fb_uid_teacher_a01` |
| **이서영** | 강사 (TEACHER) | A학원 | `fb_uid_teacher_a02` |
| **최정훈** | 강사 (TEACHER) | A학원 + B학원 (멀티 워크스페이스) | `fb_uid_teacher_a03` |
| **정민호** | 학원장 (DIRECTOR) | B학원 (별의학원) | `fb_uid_director_b01` |
| **김민서** | 학생 (v1.0+) | A학원 학생 (중3) | `fb_uid_student_a01` |
| **박엄마** | 학부모 (별도 회원 X, MVP) | 김민서 학부모 | (전화번호 010-XXXX) |
| **agent-scheduler-01** | scheduling agent | A학원 | `fb_uid_agent_a01` (type='SERVICE') |

### 0.4 데이터 가정 (시나리오 공통)

- A학원 (organization.org_pk=1, type='ACADEMY', name='한울학원') 의 YouTube 채널 = `UC_a01_test`
- B학원 (organization.org_pk=2, type='ACADEMY', name='별의학원')
- 표준 강의 길이 = 50분 (mp3, ~25MB)
- 표준 청크 분할 = 3개 (각 ~15분)
- 학원 일 영상 cap = 6편 (`org_entitlement.feature_limits.daily_uploads` 기본값)
- 검수 정책 = `review_required` (`academy_config.youtube_auto_mode` 기본 설정)

### 0.5 시나리오 ID 체계

- `F##-##`: Feature ## 의 시나리오 ##
- 예: `F11-02` = Feature 11 (강의 오디오 업로드) 의 2번째 시나리오

---

# Part A. Foundation & Identity (spec-01-01 영역)

## Feature 1: 학원 신규 가입 + 학원장 Onboarding

> 신규 학원 등록 시 identity_user + user_profile + organization + academy_config + org_entitlement + membership 이 단일 트랜잭션으로 생성된다.

### F1-01: 신규 학원장이 가입한다 (Happy)

```gherkin
Scenario: 신규 학원장이 처음 가입한다
  Given identity_user 테이블에 "kim.jiyoung@example.com" 이메일로 등록된 사용자가 없고
  And organization 테이블에 "한울학원" 이라는 학원이 없다
  
  When 김지영이 학원 가입 페이지에서 다음 정보를 입력한다:
    | 이메일       | kim.jiyoung@example.com  |
    | 비밀번호     | <강도 8자+ 영문/숫자/특수> |
    | 학원 이름    | 한울학원                  |
    | 학원장 이름  | 김지영                    |
  And "가입" 버튼을 누른다
  
  Then 시스템은 Firebase Auth 에 사용자 계정을 생성한다
  And firebase_uid 가 발급된다
  And 단일 트랜잭션 안에서 다음 5개 row 가 동시에 INSERT 된다:
    | 테이블            | 주요 값 |
    | identity_user     | firebase_uid, email='kim.jiyoung@example.com', type='HUMAN', status='ACTIVE' |
    | user_profile      | display_name='김지영', locale='ko', timezone='Asia/Seoul' |
    | organization      | type='ACADEMY', name='한울학원', status='ACTIVE' |
    | academy_config    | youtube_auto_mode='review_required', default_visibility='unlisted' |
    | org_entitlement   | service='ACADEMY', status='ACTIVE', source='FREE', feature_limits={"daily_uploads":6} |
  And membership 테이블에 (user_pk=김지영의user_pk, org_pk=한울학원의org_pk, role='DIRECTOR', status='ACTIVE') row 가 INSERT 된다
  And 김지영은 학원장 대시보드로 자동 리다이렉트된다
  And 김지영의 ID token custom claims 에 (org_pk=한울학원의org_pk, role='DIRECTOR') 가 포함된다
```

### F1-02: 가입 시 약관 동의 — PIPA 국외 이전 포함

```gherkin
Scenario: 가입 시 약관 동의 필수
  Given 김지영이 학원 가입 페이지에 있다
  And 다음 약관 동의 항목이 표시된다:
    | 항목                                                | 필수 |
    | 이용 약관                                           | Y    |
    | 개인정보 처리 방침                                  | Y    |
    | Firebase Auth 국외 이전 동의 (이메일·비밀번호 US 저장) | Y    |
    | 마케팅 정보 수신                                    | N    |
  
  When 김지영이 필수 항목 중 하나라도 체크 없이 "가입" 버튼을 누른다
  Then 시스템은 가입을 진행하지 않는다
  And "필수 약관에 동의해 주세요" 메시지를 표시한다
  
  When 김지영이 모든 필수 항목에 체크 후 "가입" 버튼을 누른다
  Then 가입이 진행된다
  And 동의 시각이 audit log 에 기록된다
```

### F1-03: 이미 가입된 이메일로 재가입 시도

```gherkin
Scenario: 중복 이메일 차단
  Given identity_user 에 email="kim.jiyoung@example.com" 사용자가 이미 존재한다
  
  When 김지영이 같은 이메일로 가입을 시도한다
  Then 시스템은 가입을 거부한다
  And "이미 등록된 이메일입니다. 로그인을 시도하거나 다른 이메일을 사용하세요" 메시지를 표시한다
  And Firebase Auth 에 새 계정이 생성되지 않는다
  And identity_user 가 추가 INSERT 되지 않는다
```

---

## Feature 2: YouTube 채널 OAuth 연결

> 학원장이 학원의 YouTube 채널을 시스템에 연결한다 (최초 1회).

### F2-01: 학원장이 YouTube 채널을 연결한다 (Happy)

```gherkin
Background:
  Given 김지영이 학원장으로 로그인되어 있고
  And 한울학원의 youtube_channel 테이블에 row 가 없다
  And 김지영이 "한울학원" YouTube 채널의 소유자다

Scenario: YouTube 채널 OAuth 연결
  When 김지영이 학원장 대시보드 → "설정 → 외부 연동" → "YouTube 채널 연결" 을 누른다
  And Google OAuth 동의 화면에서 다음 scope 에 동의한다:
    | youtube.upload     |
    | youtube.readonly   |
    | youtube            |
  And "한울학원" 채널을 선택한다
  
  Then 시스템은 Google 로부터 refresh_token 을 발급받는다
  And refresh_token 은 AWS KMS envelope encryption 으로 암호화된다
  And youtube_channel 테이블에 다음 row 가 INSERT 된다:
    | org_pk             | 1 (한울학원)            |
    | youtube_channel_id     | UC_a01_test             |
    | channel_name           | 한울학원                |
    | oauth_refresh_token_kms| <암호화된 토큰>         |
    | auto_upload_mode       | review_required (기본)  |
    | default_visibility     | unlisted (기본)         |
  And 김지영의 화면에 "YouTube 채널 연결 완료" 메시지가 표시된다
```

### F2-02: 학원장이 자동 업로드 정책을 설정한다

```gherkin
Scenario: 검수 정책 변경
  Given youtube_channel 의 auto_upload_mode='review_required' 이다
  
  When 김지영이 "자동 업로드 정책" 을 "per_teacher" 로 변경한다
  And "신뢰 강사: 박민준" 을 자동 publish 대상으로 추가한다
  
  Then youtube_channel.auto_upload_mode 가 'per_teacher' 로 UPDATE 된다
  And 박민준의 trust_relationship 에 scope='auto_publish_own' 이 INSERT 된다
  And audit_log 에 grant 행위가 기록된다
```

### F2-03: refresh_token 만료 / OAuth 재인증

```gherkin
Scenario: refresh_token 만료 시 재인증
  Given youtube_channel.oauth_refresh_token_kms 가 만료되었다 (Google 측)
  
  When 시스템이 YouTube 업로드를 시도한다
  Then YouTube API 가 401 Unauthorized 를 반환한다
  And 시스템은 youtube_channel.last_error='OAUTH_EXPIRED' 를 기록한다
  And 학원장 김지영에게 알림톡 + 이메일이 발송된다: "YouTube 채널 재연결이 필요합니다"
  And 대시보드에 "YouTube 채널 재연결" 배너가 표시된다
  
  When 김지영이 재연결 OAuth 흐름을 완료한다
  Then 새 refresh_token 이 발급되어 KMS 로 재암호화·저장된다
  And 보류된 YouTube 업로드 job 이 재시도된다
```

---

## Feature 3: 강사 초대 + 첫 로그인

> 학원장이 강사를 이메일로 초대 → `membership_invite` 생성 + 이메일 발송 → 강사 링크 클릭 후 가입 완료 시에만 identity_user + membership INSERT.

### F3-01: 학원장이 강사를 초대한다 (Happy)

```gherkin
Background:
  Given 김지영이 학원장으로 로그인되어 있고
  And 강사 박민준 (park.minjun@example.com) 은 아직 시스템에 가입되지 않았다

Scenario: 강사 초대 메일 발송
  When 김지영이 "강사 관리 → 강사 초대" 화면에서 다음 정보를 입력한다:
    | 이메일       | park.minjun@example.com |
    | 강사 이름    | 박민준                  |
  And "초대 보내기" 버튼을 누른다
  
  Then membership_invite 테이블에 다음 row 가 INSERT 된다:
    | org_pk     | 한울학원의 org_pk       |
    | email      | park.minjun@example.com |
    | role       | TEACHER                 |
    | status     | PENDING                 |
    | expires_at | 지금으로부터 24시간 후  |
  And AWS SES 를 통해 박민준에게 초대 이메일이 발송된다
  And 이메일에는 membership_invite.token 이 포함된 가입 링크가 들어있다
  But identity_user 에는 아직 박민준 row 가 없다 (초대 단계에서 계정 선생성 금지)
  But membership 에도 아직 박민준 row 가 없다
  And audit_log 에 김지영의 invite 행위가 기록된다
```

### F3-02: 강사가 초대 링크를 통해 가입한다

```gherkin
Background:
  Given 김지영이 박민준을 초대했고
  And membership_invite 에 status='PENDING' 인 박민준 초대 row 가 존재한다
  And 만료 시각은 아직 지나지 않았다

Scenario: 강사 첫 로그인 (초대 수락)
  When 박민준이 초대 메일의 가입 링크를 클릭한다
  And 비밀번호 설정 페이지에서 강도 조건을 만족하는 비밀번호를 입력한다
  And "가입 완료" 를 누른다
  
  Then Firebase Auth 에 박민준 사용자가 생성되고 firebase_uid 가 발급된다
  And 단일 트랜잭션 안에서 다음이 처리된다:
    | 처리 | 내용 |
    | identity_user INSERT | type='HUMAN', status='ACTIVE', firebase_uid=발급된uid |
    | user_profile INSERT  | display_name='박민준', locale='ko' |
    | membership INSERT    | org_pk=1, role='TEACHER', status='ACTIVE' |
    | membership_invite UPDATE | status='ACCEPTED' |
  And 박민준은 강사 대시보드로 자동 로그인된다
  And 박민준의 ID token custom claims 에 (org_pk=1, role='TEACHER') 가 포함된다
```

### F3-03: 만료된 초대 토큰

```gherkin
Scenario: 24시간 경과 후 초대 링크 접속
  Given 박민준의 초대 메일 발송 후 25시간 경과
  And membership_invite.expires_at 이 이미 지났다
  
  When 박민준이 초대 링크를 클릭한다
  Then "초대가 만료되었습니다. 학원장에게 재초대를 요청하세요" 메시지가 표시된다
  And membership_invite.status 가 'EXPIRED' 로 UPDATE 된다
  And identity_user 에 박민준 row 가 생성되지 않는다
  And membership 에도 박민준 row 가 생성되지 않는다
```

### F3-04: 학원장이 강사를 제거한다

```gherkin
Scenario: 강사 membership 회수
  Given 박민준이 A학원의 TEACHER 로 등록되어 있다
  And membership 에 (user_pk=박민준, org_pk=1, role='TEACHER', status='ACTIVE') 가 존재한다
  
  When 김지영이 "강사 관리 → 박민준 → 제거" 버튼을 누른다
  And 확인 다이얼로그에서 "예" 를 선택한다
  
  Then membership 의 박민준 row 가 status='SUSPENDED' 로 UPDATE 된다
  And audit_log 에 김지영의 revoke 행위가 기록된다
  And 박민준의 ID token 이 다음 API 호출에서 perm_version 불일치로 강제 refresh 된다
  And 박민준이 다음 API 호출 시 academy 데이터 접근이 차단된다 (Gate A 실패)
  And identity_user 의 박민준 row 는 유지된다 (다른 학원의 멤버일 수 있으므로)
```

---

## Feature 4: 멀티 워크스페이스 (한 사용자 = N 학원)

> 한 강사가 여러 학원의 TEACHER 멤버가 될 수 있다.

### F4-01: 강사가 두 학원의 멤버가 된다

```gherkin
Background:
  Given 최정훈 (choi.junghoon@example.com) 이 A학원의 강사로 가입되어 있다
  And membership 에 (user_pk=최정훈, org_pk=1, role='TEACHER', status='ACTIVE') 가 존재한다

Scenario: B학원도 최정훈을 초대
  When B학원 학원장 정민호가 같은 이메일 (choi.junghoon@example.com) 로 최정훈을 초대한다
  
  Then identity_user 에 새 row 가 생성되지 않는다 (기존 user_pk 재사용)
  And membership 에 (user_pk=최정훈, org_pk=2, role='TEACHER', status='ACTIVE') 가 추가 INSERT 된다
  And 최정훈에게 "B학원에서 강사 권한이 부여되었습니다" 알림이 발송된다
  And 최정훈은 이메일 인증 없이 즉시 B학원 데이터 접근 가능 (이미 Firebase 계정 보유)
```

### F4-02: 학원 컨텍스트 스위처

```gherkin
Background:
  Given 최정훈이 A학원과 B학원의 강사다
  And 최정훈이 로그인되어 있다

Scenario: 학원 간 컨텍스트 전환
  When 최정훈이 강사 대시보드에서 학원 스위처 (드롭다운) 를 클릭한다
  Then 두 학원이 표시된다: ["A학원 (한울학원)", "B학원 (별의학원)"]
  
  When 최정훈이 "B학원" 을 선택한다
  Then 모든 후속 API 호출에 X-Org-Pk: 2 헤더가 추가된다
  And 대시보드는 B학원의 데이터만 표시한다 (강의·자료·평가)
  And A학원의 데이터는 화면에 나타나지 않는다
```

### F4-03: 잘못된 학원 컨텍스트 시도 (보안)

```gherkin
Scenario: 멤버가 아닌 학원 접근 시도
  Given 박민준이 A학원만의 TEACHER 다 (B학원 멤버 X)
  
  When 박민준이 임의로 X-Org-Pk: 2 헤더를 추가해 B학원의 강의 목록을 요청한다
  Then AcademyScopeInterceptor 가 박민준의 membership 을 확인한다
  And B학원 (org_pk=2) 에 대한 membership 이 없음을 발견한다 (Gate A 실패)
  And HTTP 403 Forbidden 응답을 반환한다 ("No access to this organization")
  And B학원 데이터는 일체 응답에 포함되지 않는다
  And 보안 위반 시도가 audit_log 에 기록된다 (action='access_denied')
```

### F4-04: 한 학원에서 회수된 권한이 다른 학원에 영향 X

```gherkin
Background:
  Given 최정훈이 A학원·B학원의 강사다
  And A학원에서 trust grant "auto_publish_own" 을 받았다

Scenario: A학원에서만 권한 회수
  When A학원 학원장 김지영이 최정훈의 auto_publish_own grant 를 회수한다
  
  Then trust_relationship 의 (org_pk=1, teacher_pk=최정훈, scope='auto_publish_own') 가 deleted_at 으로 soft delete 된다
  And B학원의 최정훈 권한은 변경되지 않는다
  And 최정훈이 A학원의 영상을 publish 시도 → DB 재검증에서 차단 (Sensitive write)
  And 최정훈이 B학원의 영상은 정상 publish 가능
```

---

## Feature 5: Trust Grant 부여/회수

> DIRECTOR 가 TEACHER 에게 academy-specific 추가 권한 (trust_relationship) 을 부여/회수한다.

### F5-01: 학원장이 강사에게 자동 publish 권한을 부여한다

```gherkin
Background:
  Given 김지영이 A학원 DIRECTOR 로 로그인되어 있다
  And 박민준은 A학원 TEACHER (기본 권한만 보유)

Scenario: auto_publish_own grant 부여
  When 김지영이 "강사 관리 → 박민준 → 권한 관리" 화면에서
  And "auto_publish_own" 체크박스를 활성화한다
  And "저장" 을 누른다
  
  Then trust_relationship 테이블에 다음 row 가 INSERT 된다:
    | org_pk      | 1                  |
    | teacher_user_pk | 박민준의 user_pk   |
    | granted_by_pk   | 김지영의 user_pk   |
    | scope           | auto_publish_own   |
    | effective_from  | <current_timestamp>|
    | effective_to    | NULL (무기한)      |
  And audit_log 에 (actor=김지영, target=박민준, action='grant', scope='auto_publish_own') 이 INSERT 된다
  And 박민준의 Firebase custom claims 가 다음 token refresh 시 갱신된다
  And 박민준에게 알림이 발송된다: "자동 publish 권한이 부여되었습니다"
```

### F5-02: 강사가 grant 를 받은 후 자동 publish 시도

```gherkin
Background:
  Given 박민준이 auto_publish_own grant 를 받았다
  And 한울학원 youtube_channel.auto_upload_mode='per_teacher' 이다
  And 박민준의 영상 #501 이 HyperFrames render 완료 상태다 (검수 대기 직전)

Scenario: 검수 우회 자동 publish
  When 시스템이 영상 #501 의 publish 흐름을 진행한다
  Then 박민준의 ability 를 평가한다 (sensitive write → DB 재검증)
  And 박민준이 'auto_publish_own' grant 보유 + lecture.teacher_pk=박민준 확인
  And 영상은 검수 단계를 우회한다 (status='pending_review' 단계 skip)
  And 즉시 youtube_video.status='uploading' 으로 진행한다
  And 학원장 김지영에게 알림은 발송되지 않는다 (검수 큐 X)
  And 강사 박민준에게만 "영상 자동 publish 완료" 알림이 발송된다
```

### F5-03: 학원장이 grant 를 회수한다

```gherkin
Scenario: auto_publish_own 회수
  Given 박민준이 auto_publish_own grant 를 보유 중이다
  
  When 김지영이 박민준의 grant 를 회수한다 (체크박스 비활성)
  
  Then trust_relationship 의 해당 row 가 deleted_at=<current> 로 soft delete 된다
  And audit_log 에 (actor=김지영, target=박민준, action='revoke', scope='auto_publish_own') 이 INSERT 된다
  And 박민준의 Firebase custom claims 가 다음 refresh 시 갱신된다
```

### F5-04: Grant 회수 직후 sensitive write 차단 (DB 재검증)

```gherkin
Background:
  Given 박민준이 auto_publish_own grant 를 보유했었다
  And 김지영이 방금 grant 를 회수했다 (1분 전)
  And 박민준의 ID token 은 아직 refresh 되지 않아 stale claims 보유

Scenario: Stale claims 로 publish 시도
  When 박민준이 영상 #502 의 publish API 를 호출한다 (token = stale claims 포함)
  And FirebaseAuthGuard 가 JWT claims 의 ability 빠른 체크 → 통과 (stale)
  
  Then VerifyOnDb 데코레이터가 sensitive write 임을 감지한다
  And DB 에서 최신 membership + trust_relationship 을 재 fetch 한다
  And auto_publish_own grant 가 deleted_at 으로 soft delete 됐음을 확인한다
  And 새 ability 빌드 시 publish 권한이 없음을 발견한다
  And HTTP 403 Forbidden 응답: "Permission revoked"
  And 영상 #502 의 publish 는 진행되지 않는다
  And 박민준에게 알림이 발송된다: "권한이 갱신되었습니다. 다시 로그인하세요" (FCM push 또는 알림톡)
```

### F5-05: 학원장이 view_all_lectures grant 부여 후 강사가 타강사 강의 조회

```gherkin
Background:
  Given 박민준이 view_all_lectures grant 를 부여받았다

Scenario: 강사가 동료 강사 강의 조회
  When 박민준이 "강의 목록" 페이지에 접속한다
  And 필터 "강사: 이서영" 선택
  
  Then 박민준의 ability 는 view 권한을 통과시킨다 (grant 기반)
  And 이서영의 강의 목록이 정상 표시된다
  And 박민준은 이서영의 강의 영상을 시청 가능 (publish 된 것에 한함)
  But 박민준은 이서영의 강의를 delete 할 수 없다 (delete_all_lectures grant 별도)
```

---

## Feature 6: Agent (Service Account) 인증

> 자동화 봇 (예: scheduling agent) 도 사람과 동일한 identity 모델로 인증된다.

### F6-01: 학원장이 agent 를 등록한다

```gherkin
Background:
  Given 김지영이 A학원 DIRECTOR 로 로그인되어 있다
  And "Scheduling Agent" 가 백엔드에 등록된 agent 종류다

Scenario: Agent 등록
  When 김지영이 "외부 연동 → Agent 관리 → Scheduling Agent 활성화" 를 누른다
  
  Then 시스템은 agent 용 Firebase 계정을 생성한다 (custom token 발급용)
  And 단일 트랜잭션 안에서 다음이 처리된다:
    | 테이블         | 주요 값 |
    | identity_user  | firebase_uid=<agent uid>, email='agent-scheduler-01@...', type='SERVICE', status='ACTIVE' |
    | user_profile   | display_name='Scheduling Agent' |
    | membership     | org_pk=1, role='AGENT_SCHEDULER', status='ACTIVE' |
  And agent 의 service account 키 가 KMS 에 보관된다 (academy 별 KEK)
  And audit_log 에 (actor=김지영, target=agent, action='grant') 이 기록된다
```

### F6-02: Agent 가 인증하여 강의 조회 (사람과 동일 path)

```gherkin
Background:
  Given Scheduling Agent 가 A학원에 등록되어 있다
  And agent 의 service account 키로 Firebase Custom Token 이 발급되었다

Scenario: Agent 의 API 호출 흐름
  When agent process 가 Firebase Custom Token 으로 signInWithCustomToken 호출
  Then Firebase 가 ID token (with custom claims = AGENT_SCHEDULER membership) 발급
  
  When agent 가 GET /lectures?status=published 호출 (Authorization: Bearer <token>)
  Then FirebaseAuthGuard 가 token verify (사람 사용자와 동일 path)
  And AcademyPolicyGuard 가 agent 의 AGENT_SCHEDULER role 에 대한 ability 빌드
  And ability.can('view', 'Lecture') = true 확인
  And 강의 목록이 정상 응답된다
  And rag_query_log 또는 audit log 에 actor_user_pk=agent 가 기록된다 (사람과 분리 가능)
```

### F6-03: Agent 권한 회수

```gherkin
Scenario: Agent 비활성화
  When 김지영이 "Scheduling Agent → 비활성화" 를 누른다
  Then membership 의 agent row 가 status='SUSPENDED' 로 UPDATE 된다
  And identity_user.status='SUSPENDED' 가 가능 (옵션)
  And agent 의 다음 API 호출 시 ForbiddenException 발생
  And 보류된 agent 작업이 중단된다
```

### F6-04: Agent 와 사람 통계 분리

```gherkin
Scenario: 학원 사용 통계에서 agent 행위 분리
  Given 한 달간 다음 통계가 누적되어 있다:
    | identity_user.type | API 호출 수 |
    | HUMAN              | 2,400       |
    | SERVICE            | 14,000      |
  
  When 김지영이 "학원 사용 통계" 페이지를 조회한다
  Then "사람 사용자 활동" 섹션에 2,400 호출이 표시된다
  And "Agent 활동" 섹션에 14,000 호출이 별도 표시된다
  And 둘은 합산되지 않는다 (UI 분리)
```

---

## Feature 7: 권한 거부 / 입구컷

> 권한 없는 사용자의 접근을 가능한 한 빠른 지점에서 차단한다.

### F7-01: 인증되지 않은 요청

```gherkin
Scenario: JWT 없는 요청
  When 클라이언트가 GET /lectures 를 Authorization 헤더 없이 호출한다
  Then FirebaseAuthGuard 가 401 Unauthorized 응답
  And 응답 body 는 application code 에 도달하지 않는다
  And 응답 시간은 < 50ms (Guard 단계에서 차단)
```

### F7-02: 만료된 JWT

```gherkin
Scenario: 만료된 token 사용
  Given 박민준의 ID token 이 발급 후 70분 경과 (TTL 1시간 초과)
  
  When 박민준의 클라이언트가 만료된 token 으로 API 호출
  Then Firebase Admin SDK verifyIdToken 이 throws (TOKEN_EXPIRED)
  And FirebaseAuthGuard 가 401 응답 + "TOKEN_EXPIRED" 코드
  And 클라이언트는 자동으로 getIdToken(true) 호출 → refresh 시도
  And 새 token 으로 재시도 시 정상 응답
```

### F7-03: 위조된 JWT (signature 불일치)

```gherkin
Scenario: 잘못된 서명의 token
  When 누군가가 위조된 JWT (잘못된 signature) 로 API 호출
  Then Firebase Admin SDK 가 signature 검증 실패
  And 401 응답 + 보안 위반 로그 기록
  And 동일 IP 에서 N회 반복 시 rate limit 적용 (옵션, v0.5+)
```

### F7-04: 권한은 없지만 인증된 사용자

```gherkin
Scenario: TEACHER 가 admin 전용 endpoint 호출
  Given 박민준이 TEACHER 로 인증되어 있다
  
  When 박민준이 POST /admin/youtube-channels (YouTube 채널 관리, DIRECTOR only) 호출
  Then AcademyPolicyGuard 가 ability.can('manage', 'YoutubeChannel') = false 확인
  And HTTP 403 Forbidden 응답
  And 비즈니스 로직은 실행되지 않는다
  And youtube_channel 테이블은 변경되지 않는다
```

### F7-05: 다른 학원의 리소스 ID 추측 접근

```gherkin
Background:
  Given 박민준이 A학원 (org_pk=1) 의 TEACHER 다
  And B학원에 강의 ID 999 (lecture_pk=999) 가 존재한다

Scenario: 다른 학원 리소스 ID 추측 시도
  When 박민준이 GET /lectures/999 호출
  Then AcademyScopeInterceptor 가 lecture.org_pk=2 인 것을 발견
  And 박민준의 org_pk=1 의 membership 으로 차단
  And HTTP 404 Not Found (lecture 존재 비공개) 응답
  And 보안 이벤트 로그: "박민준이 학원 1 의 권한으로 학원 2 의 lecture 999 접근 시도"
```

---

# Part B. RAG Infrastructure (spec-01-02 영역)

## Feature 8: 강사 사전 자료 업로드 (RAG seed)

> 강사가 본인 강의안·교과서 발췌 등을 사전 업로드하여 RAG 컨텍스트 강화.

### F8-01: 강사가 강의안 PDF 를 업로드한다 (Happy)

```gherkin
Background:
  Given 박민준이 강사로 로그인되어 있다

Scenario: PDF 강의안 업로드
  When 박민준이 "자료 관리 → 자료 업로드" 화면에서 다음을 입력한다:
    | 파일       | math_ch3_quadratic.pdf (5MB) |
    | 과목       | 수학                          |
    | 학년       | 중3                           |
    | 단원       | 이차함수와 그래프              |
    | 자료 형식  | 강의안 PDF                    |
  And "업로드" 버튼을 누른다
  
  Then S3 에 파일이 업로드된다 (prefix=s3://academy-mvp/academy_1/material/...)
  And lecture 테이블에 다음 row 가 INSERT 된다:
    | type           | material                          |
    | org_pk     | 1                                 |
    | teacher_pk     | 박민준의 user_pk                  |
    | title          | <자동 추출: math_ch3_quadratic>   |
    | subject        | 수학                              |
    | grade_level    | 중3                               |
    | source_material_s3 | <S3 url>                       |
    | status         | queued                            |
  And BullMQ 에 'index-material' job 이 큐 적재된다
  And 박민준에게 "자료 인덱싱 중" 메시지가 표시된다
```

### F8-02: PDF 텍스트 추출 + 청킹

```gherkin
Background:
  Given F8-01 의 자료가 lecture.status='queued' 상태다

Scenario: PDF → text → chunks
  When BullMQ 워커가 'index-material' job 을 실행한다
  And pdf-text-extractor (content-api 의 패턴 차용) 가 PDF 에서 텍스트 추출
  
  Then 추출된 텍스트가 토픽 단위로 청킹된다 (Claude 사용)
  And lecture_chunk 테이블에 N row (예: 5개) INSERT 된다
  And 각 chunk 의 (start_sec, end_sec, srt_content) 가 채워진다 (PDF의 경우 sec=NULL, srt_content=텍스트)
  And lecture.status='ready' 로 UPDATE 된다
```

### F8-03: 자료 인덱싱 (Qdrant + Neo4j)

```gherkin
Background:
  Given F8-02 의 자료 청크 5개가 ready 상태다

Scenario: RAG 인덱싱
  When BullMQ 'index-rag' job 이 각 chunk 에 대해 실행된다
  And Upstage Solar Embedding API 가 호출되어 1024-dim vector 생성
  And Qdrant 에 각 chunk 의 point 가 upsert 된다 (payload: org_id=1, teacher_id=박민준, lecture_id=<id>, chunk_type='material', subject='수학', grade_level='중3', ...)
  And lecture_chunk.qdrant_point_ids 에 point ID 배열이 UPDATE 된다
  And lecture_chunk.embedding_model='upstage-solar-1024' 가 기록된다
  And lecture_chunk.rag_indexed_at=<current> 가 기록된다
  
  When 'extract-neo4j-concepts' job 이 실행된다
  And Claude Haiku 가 자료 텍스트에서 개념·관계 JSON 추출
  And Cypher MERGE 로 Neo4j 에 (Subject, Chapter, Concept, Definition, Example) 노드 + 관계 12종이 upsert 된다
  
  Then 모든 노드/엣지에 org_id=1 속성이 있다
  And 박민준은 "자료 인덱싱 완료" 알림을 받는다
```

### F8-04: 잘못된 자료 거부 (저작권 / 부적절)

```gherkin
Scenario: 타인 강의 영상 업로드 시도
  When 박민준이 "타학원-인강.mp4" 를 자료로 업로드 시도
  Then 시스템은 파일 형식이 video/mp4 인 것을 감지 (PDF/image/text 만 허용)
  And HTTP 400 Bad Request: "지원되지 않는 자료 형식입니다. PDF, image, text 만 가능합니다."
  And 자료는 업로드되지 않는다
```

### F8-05: 자료 메타 가이드 표시

```gherkin
Scenario: 효과적인 자료 가이드
  When 박민준이 자료 업로드 화면을 연다
  Then 다음 가이드 텍스트가 표시된다:
    """
    📚 효과적인 자료
      ✓ 강의안 PDF (개념 + 예제가 정돈)
      ✓ 교과서 발췌 (정의·공식 카드)
      ✓ 예제 풀이 노트 (단계별 풀이)
      ✓ 자주 묻는 질문 모음
    
    ❌ 효과 적은 자료
      ✗ 긴 텍스트 위주 (요약·도식 없음)
      ✗ 타인 강의 영상 (저작권 충돌)
      ✗ 학생 개인정보 포함 자료 (이름·전화 등)
    """
```

---

## Feature 9: 자료/강의 인덱싱 lifecycle

> 인덱싱 실패 복구 + 재인덱싱 + 삭제 처리.

### F9-01: 인덱싱 실패 → 재시도

```gherkin
Scenario: Upstage API 일시 장애
  Given lecture_chunk #1001 이 인덱싱 대기 중이다
  
  When 'index-rag' job 이 실행되어 Upstage API 호출
  And Upstage API 가 503 Service Unavailable 반환
  
  Then job 은 BullMQ 의 지수 백오프 재시도 큐로 이동 (낮은 비용, 무한 재시도)
  And lecture_chunk.rag_indexed_at 은 여전히 NULL
  
  When 1분 후 Upstage API 가 정상 응답
  Then 재시도 job 이 성공
  And lecture_chunk.qdrant_point_ids + rag_indexed_at 이 UPDATE 된다
```

### F9-02: 강의 삭제 시 Qdrant + Neo4j 정리

```gherkin
Background:
  Given lecture_pk=42 가 publish 된 상태고, lecture_chunk 3개 + Qdrant points 3개 + Neo4j Lecture 노드가 있다

Scenario: 강의 영구 삭제
  When 김지영이 lecture #42 의 "삭제" 버튼을 누른다 (DIRECTOR 권한)
  And 확인 다이얼로그에서 "예" 선택
  
  Then lecture.deleted_at=<current> 로 soft delete
  And BullMQ 에 'delete-rag-traces' job 적재
  
  When job 이 실행된다
  And Qdrant 에서 chunk.qdrant_point_ids 의 모든 point 가 hard delete 된다
  And Neo4j 에서 Lecture 노드가 detach + delete 된다 (Concept 노드는 유지 — 다른 lecture 참조 가능)
  And YouTube Data API 의 videos.delete 가 호출된다
  And youtube_video.status='deleted' UPDATE
  
  Then 학생/학부모가 더 이상 영상에 접근할 수 없다
  And RAG 검색에서도 해당 chunk 가 나오지 않는다
```

### F9-03: 첫 강의 — RAG empty 시점 graceful fallback

```gherkin
Background:
  Given A학원에 lecture_chunk row 가 0개 (신규 학원)
  And Qdrant 의 org_id=1 의 point 가 0개

Scenario: 첫 강의 처리 시 RAG retrieval 빈 결과
  When 박민준이 첫 강의를 업로드한다
  And 'generate-html' job 이 실행되어 RAG 검색을 시도
  And Qdrant search 가 0 results 반환
  
  Then Claude 프롬프트는 retrieved_materials / retrieved_past_lectures 섹션이 빈 상태로 호출된다
  And Claude 는 transcript + Neo4j (역시 빈) 만으로 HTML 생성
  And HTML 품질은 후속 강의 대비 낮을 수 있으나, 정상 생성된다
  And 본 강의가 인덱싱되어 두 번째 강의부터는 RAG 컨텍스트가 풍부해진다
```

---

## Feature 10: RAG 검색 (분석 단계)

> 강의 HTML 생성 시 RAG retrieval 동작.

### F10-01: HTML 생성 시 RAG 컨텍스트 주입 (Happy)

```gherkin
Background:
  Given A학원에 lecture_chunk row 가 50개 (누적)
  And 박민준이 과거에 "이차함수" 관련 자료 + 강의 6편을 인덱싱했다
  And 박민준의 신규 강의 transcript 청킹 결과: 청크 3개 ([0:00~15:00 "이차함수 정의", 15:00~32:00 "정점 변환", 32:00~50:00 "응용 풀이"])

Scenario: 청크 2 의 HTML 생성
  When 'generate-html' job 이 청크 2 ("정점 변환") 에 대해 실행된다
  And Claude 가 청크 SRT 요약문 생성 (예: "이차함수 정점 변환 5문제")
  And Qdrant search (filter: org_id=1 AND teacher_id=박민준, top_k=8) 호출
  And 결과: material 2개 ("강의안 정점 변환 페이지") + 과거 lecture 3개 ("같은 강사 4월 강의의 정점 부분")
  And Neo4j 조회: 핵심 개념=[이차함수, 정점], 선수=[이차식], 관련=[판별식]
  
  Then 위 정보가 schema 형식 프롬프트로 Claude 에 주입된다
  And Claude 가 HyperFrames HTML 을 생성한다
  And HTML 에는 GSAP timeline, class="clip" 요소, audio src 등 컨벤션 준수
  And video_asset (type='hyperframes_html') 이 S3 에 저장된다
```

---

# Part C. Pipeline (spec-01-03 영역)

## Feature 11: 강사 강의 오디오 업로드 + Validation

> 강사가 수업 오디오를 업로드. 검증 후 파이프라인 시작.

### F11-01: 정상 업로드 (Happy)

```gherkin
Background:
  Given 박민준이 강사로 로그인되어 있다
  And A학원 youtube_channel 이 OAuth 연결 완료 상태다
  And org_entitlement.feature_limits.daily_uploads=6 이고 오늘 처리된 영상 = 3편

Scenario: 50분 강의 mp3 업로드
  When 박민준이 "강의 업로드" 화면에서 다음을 입력한다:
    | 파일         | math_ch3_class.mp3 (25MB, 50분) |
    | 학원         | 한울학원 (자동)                  |
    | 반           | 중3 수학반                       |
    | 수업 일시    | 2026-05-21 18:00                |
    | 과목         | 수학                             |
    | 학년         | 중3                              |
  And "업로드" 버튼을 누른다
  
  Then 클라이언트가 academy-api 에 presigned S3 URL 을 요청한다
  And S3 PUT 으로 mp3 가 직접 업로드된다 (academy-api 트래픽 없이)
  And 업로드 완료 후 클라이언트가 POST /lectures 호출 (메타 정보 + S3 url)
  And lecture 테이블에 다음 INSERT:
    | type             | lecture                                 |
    | org_pk       | 1                                       |
    | teacher_pk       | 박민준의 user_pk                        |
    | title            | (자동 또는 빈)                          |
    | subject          | 수학                                    |
    | grade_level      | 중3                                     |
    | class_group      | 중3 수학반                              |
    | recorded_at      | 2026-05-21T18:00:00+09:00              |
    | duration_sec     | 3000                                    |
    | source_audio_s3  | s3://.../math_ch3_class.mp3            |
    | status           | queued                                  |
  And BullMQ 에 'process-lecture' job 이 적재된다
  And 박민준에게 "업로드 완료 — 처리 중" 응답이 1초 이내 반환된다
  And lecture.status 가 polling 또는 WebSocket 으로 강사에게 보여진다
```

### F11-02: 너무 큰 파일 거부

```gherkin
Scenario: 1GB 초과 파일
  When 박민준이 1.2GB mp3 파일을 업로드 시도
  Then 클라이언트가 사전 size 검증 → "파일 크기는 500MB 이하만 허용됩니다" 에러 표시
  And S3 업로드는 시작되지 않는다
  
  When 만약 클라이언트 검증 우회로 S3 업로드 성공해도
  Then academy-api 의 lecture 생성 시 file size 검증
  And lecture 가 status='failed' 로 INSERT + 강사 알림
```

### F11-03: 잘못된 형식 거부

```gherkin
Scenario: 비-오디오 파일
  When 박민준이 "presentation.pptx" 를 강의 오디오로 업로드 시도
  Then 클라이언트가 mime type 검증 → "audio/mpeg, audio/wav, audio/m4a 형식만 허용됩니다"
  And 업로드 차단
```

### F11-04: 일 cap 초과 시 큐 대기

```gherkin
Background:
  Given A학원의 daily_video_limit=6
  And 오늘 0시 이후 처리된 영상 = 6편

Scenario: 7번째 영상 업로드 시도
  When 박민준이 7번째 강의를 업로드한다
  Then 업로드 자체는 성공한다 (S3 저장)
  And lecture 가 INSERT 되지만 status='cap_pending'
  And BullMQ job 은 큐에 적재되지만 처리되지 않는다 (cap 검증 후 hold)
  And 박민준에게 알림: "오늘 학원 영상 한도(6편) 도달. 내일 0시부터 자동 처리됩니다."
  And 학원장 김지영에게도 알림: "오늘 영상 한도 도달"
  
  When 자정 (KST) 이 되어 daily counter 가 reset 된다
  Then status='cap_pending' 인 lecture 가 자동 'queued' 로 전환
  And 파이프라인이 정상 진행된다
```

### F11-05: 60분 초과 강의 경고

```gherkin
Scenario: 70분 강의 업로드
  When 박민준이 4200초 (70분) 길이의 mp3 를 업로드한다
  Then 시스템은 업로드를 수락한다
  But 강사에게 경고 표시: "60분 초과 강의는 처리 시간이 길고 비용이 증가합니다."
  And lecture.status='queued' 로 진행 (거부 X, 단지 경고)
  And HyperFrames 청크 수가 더 많아짐 (5-6개 예상)
```

---

## Feature 12: STT 처리 + 실패 복구

> Google STT V2 batchRecognize 로 transcript 생성. 실패 시 재시도.

### F12-01: STT 정상 처리

```gherkin
Background:
  Given lecture #501 (50분 mp3) 가 status='queued' 다
  And BullMQ 'process-lecture' 가 실행되어 'transcribe' job 으로 진행

Scenario: Google STT V2 호출
  When 'transcribe' job 이 실행된다
  And lecture.status='transcribing' 으로 UPDATE
  And mp3 → flac 변환 (FFmpeg, content-api 패턴)
  And GCS 에 임시 업로드 (chunk 분할 if >20분)
  And Google STT V2 batchRecognize 호출 (multi-URI)
  And ~10분 후 GCS 에서 결과 JSON 다운로드
  
  Then transcript 가 timestamps 포함하여 추출된다
  And video_asset (type='transcript_json') 으로 S3 에 저장
  And BullMQ 에 다음 job 'chunk-by-topic' 이 적재된다
  And SLA: 50분 audio → 10분 이내 transcript (p95)
```

### F12-02: STT 1회 실패 → 자동 재시도

```gherkin
Scenario: 일시적 STT 오류
  Given 'transcribe' job 첫 시도가 Google STT API 의 503 Service Unavailable 로 실패
  
  Then BullMQ 가 지수 백오프 (5초, 25초, 125초) 로 재시도
  
  When 2번째 시도에서 성공
  Then transcript 가 정상 생성된다
  And 강사에게 별도 알림은 발송되지 않는다 (자동 복구)
```

### F12-03: STT 3회 실패 → 강사 알림

```gherkin
Scenario: 영구 STT 실패
  Given 'transcribe' job 이 3회 시도 모두 실패 (예: 잘못된 오디오 파일)
  
  Then lecture.status='failed', failure_reason='STT 처리 실패: 오디오 품질 확인 필요'
  And 강사 박민준에게 알림톡 + 이메일 발송: "강의 #501 STT 처리에 실패했습니다. 오디오 품질을 확인해 주세요."
  And 강사 화면에 "수동 텍스트 입력" 옵션 표시 (V0.5+ 기능, MVP 는 알림만)
  And 학원장 김지영에게도 알림 발송
```

### F12-04: STT 정확도 낮음 (custom vocabulary 미적용)

```gherkin
Scenario: 수학 전문 용어 인식률 저하
  Given 50분 강의에 "판별식", "근의 공식" 등 전문 용어 빈도 높음
  And custom vocabulary 가 등록되지 않음 (MVP 단계)
  
  When STT 가 완료된다
  Then transcript 가 생성되나, 일부 용어가 잘못 인식 가능 ("판별식" → "판별직")
  And 후속 Claude 청킹·HTML 생성 시 부정확한 용어가 슬라이드에 반영 가능
  And v0.5+ 에서 강사 수정 UI 도입 예정 (현 phase 외)
```

---

## Feature 13: Claude 청킹 + SRT 생성

> Transcript 를 토픽 단위로 청킹. 각 청크의 SRT (timestamp 포함) 생성.

### F13-01: 토픽 기반 청킹 (Happy)

```gherkin
Background:
  Given transcript_json 이 video_asset 으로 저장되어 있다
  And lecture.status='transcribing' → 'chunking' 으로 전환됨

Scenario: Claude 가 토픽 경계 식별
  When 'chunk-by-topic' job 이 실행된다
  And Claude Sonnet 4.7 호출 (전체 transcript + RAG 컨텍스트)
  And 프롬프트에 "자연스러운 토픽 경계를 찾아 SRT 형식으로 작성" 명시
  
  Then Claude 응답이 다음 형식 JSON 으로 반환된다:
    """
    {
      "chunks": [
        {
          "seq": 1,
          "title": "이차함수 정의 복습",
          "start_sec": 0,
          "end_sec": 892,
          "srt_content": "1\n00:00:00,000 --> 00:00:08,500\n오늘은 이차함수 정점을 ...\n..."
        },
        { "seq": 2, "title": "정점 변환 공식", "start_sec": 892, "end_sec": 1924, ... },
        { "seq": 3, "title": "응용 5문제 풀이", "start_sec": 1924, "end_sec": 3000, ... }
      ]
    }
    """
  And lecture_chunk 테이블에 3 row 가 INSERT 된다
  And lecture.status='rendering' 으로 UPDATE (다음 단계 진입)
  And BullMQ 에 각 청크별 'generate-tts' + 'generate-html' job 이 병렬 적재된다
```

### F13-02: 청크가 1개로 식별됨 (짧은 강의)

```gherkin
Scenario: 15분 강의 = 1 청크
  Given 15분 강의가 업로드되었다
  
  When Claude 가 청킹한다
  Then 청크가 1개만 식별된다 (자연 토픽 경계 X)
  And lecture_chunk 에 seq=1, start_sec=0, end_sec=900 1 row INSERT
  And 1개 청크로만 HyperFrames 렌더 진행
```

### F13-03: 청킹 실패 → 재시도

```gherkin
Scenario: Claude API 일시 장애
  Given 'chunk-by-topic' job 첫 시도가 Anthropic API 429 (rate limit) 로 실패
  
  Then BullMQ 가 재시도 (최대 2회, Claude 비용 ↑ 고려)
  
  When 2번째 시도 성공
  Then 청킹 결과 정상 적용
```

### F13-04: 청킹 최종 실패

```gherkin
Scenario: 영구 실패 (2회 모두 실패)
  Then lecture.status='failed', failure_reason='청킹 실패: Claude API 응답 불가'
  And 강사 + 학원장 알림 발송
```

---

## Feature 14: Google TTS + HyperFrames HTML 생성 (병렬)

> 청크별로 TTS (MP3) 와 HTML 을 병렬 생성. 양쪽 완료 후 HyperFrames render.

### F14-01: TTS 정상 처리

```gherkin
Background:
  Given lecture_chunk #1001 (seq=2, srt_content 보유) 이 'rendering' 상태다
  And BullMQ 'generate-tts' job 이 청크 1001 에 대해 실행됨

Scenario: 한국어 나레이션 합성
  When Google Cloud TTS Neural2 API 호출 (voice='ko-KR-Neural2-A')
  And SRT 본문에서 텍스트만 추출 → TTS 입력
  
  Then ~30초 후 MP3 파일이 반환된다 (Claude 가 생성한 정리된 스크립트 기반, 약 5-10K char)
  And S3 에 video_asset (type='audio_mp3', chunk_pk=1001) 으로 저장
  And video_asset.status='ready' UPDATE
  And 청크 단위 latency: < 1분 (p95)
```

### F14-02: HyperFrames HTML 생성

```gherkin
Background:
  Given 청크 #1001 의 RAG 컨텍스트가 retrieve 되었다 (Feature 10)

Scenario: Claude 가 HyperFrames HTML 작성
  When 'generate-html' job 이 청크 1001 에 대해 실행된다
  And Claude Sonnet 4.7 호출 with schema 프롬프트 (lecture_context, chunk_transcript, retrieved_materials, retrieved_past_lectures, concept_graph, output_requirements)
  
  Then Claude 가 HyperFrames HTML 을 반환한다
  And HTML 은 다음 컨벤션을 따른다:
    | window.__timelines 등록    | yes |
    | class="clip" 요소 사용     | yes |
    | data-start, data-duration  | yes |
    | <audio src="<S3 url>">     | (F14-01 결과 MP3 URL 참조)   |
    | Math.random() 사용         | no (seeded PRNG)         |
    | GSAP timeline 동기 구축    | yes |
  And HTML 이 S3 에 video_asset (type='hyperframes_html') 으로 저장
  And video_asset.status='ready'
  And 청크 단위 latency: < 2분 (p95)
```

### F14-03: TTS 와 HTML 양쪽 완료 후 다음 단계 진입

```gherkin
Background:
  Given 청크 #1001 의 TTS (audio_mp3) 와 HTML 양쪽 모두 status='ready' 다

Scenario: render-video job 트리거
  When 시스템이 청크 1001 의 모든 의존성을 확인한다
  Then BullMQ 에 'render-video' job (청크 1001) 이 적재된다
  
  But 청크 #1002 의 TTS 가 아직 처리 중이면
  Then 청크 #1002 의 render-video 는 아직 트리거되지 않는다
  And 청크별 독립적 진행 (병렬)
```

### F14-04: TTS 실패 → 재시도

```gherkin
Scenario: Google TTS 일시 장애
  Given 'generate-tts' (청크 1001) 첫 시도 실패
  
  Then BullMQ 재시도 (3회, 지수 백오프)
  
  When 정상 처리
  Then audio_mp3 정상 생성
```

### F14-05: TTS 무료 한도 초과

```gherkin
Background:
  Given Google TTS Neural2 무료 한도 = 1M chars/월
  And A학원의 이번 달 누적 = 1.1M chars (10% 초과)

Scenario: 유료 구간 진입
  When TTS 호출이 발생한다
  Then Google Cloud Billing 에서 자동 과금된다 ($16/1M chars)
  And 시스템 로그: "TTS 무료 한도 초과 — 유료 구간 진입"
  And 비용 cap 검증 (per academy daily limit, settings/limits.ts)
  And TTS 정상 처리 (cap 미초과 시)
```

---

## Feature 15: HyperFrames Render + Webhook

> HyperFrames API 가 HTML+audio → MP4 변환. Webhook 으로 결과 수신.

### F15-01: HyperFrames render 호출

```gherkin
Background:
  Given 청크 1001 의 audio_mp3 + hyperframes_html 양쪽 ready
  And lecture.status='rendering' 중

Scenario: HyperFrames API 호출
  When 'render-video' job 이 청크 1001 에 대해 실행된다
  And HyperFrames API POST /v1/renders 호출:
    """
    {
      "html": "<HTML body with <audio src='https://s3.../chunk-1001.mp3'>",
      "webhook_url": "https://api.academy-mvp.../webhooks/hyperframes",
      "duration_seconds": 1032
    }
    """
  And HyperFrames 응답: { render_id: "rdr_abc123", status: "queued" }
  
  Then video_asset (type='video_mp4', chunk_pk=1001, status='pending', source_engine='hyperframes', metadata={render_id: 'rdr_abc123'}) INSERT
  And BullMQ 'render-video' job 은 'awaiting-webhook' 상태로 대기 (실제 polling 또는 timeout 설정)
```

### F15-02: Webhook 수신 → MP4 다운로드

```gherkin
Scenario: HyperFrames 완료 통보
  Given 청크 1001 의 render_id='rdr_abc123' 가 HyperFrames 큐에서 처리 중
  
  When HyperFrames 가 약 20분 후 다음 webhook POST:
    """
    {
      "render_id": "rdr_abc123",
      "status": "ready",
      "mp4_url": "https://cdn.hyperframes.app/.../rdr_abc123.mp4",
      "duration_seconds": 1032,
      "size_bytes": 124857600
    }
    """
  
  Then 시스템이 webhook 서명을 검증한다 (HMAC SHA256)
  And 시스템이 mp4 를 다운로드해 S3 에 저장
  And video_asset (chunk_pk=1001, type='video_mp4') status='ready' UPDATE
  And BullMQ 에 다음 job 'await-review' (또는 trust grant 시 'upload-youtube' 직행) 적재
```

### F15-03: 모든 청크 render 완료 → 검수 대기

```gherkin
Background:
  Given lecture #501 의 청크 3개 모두 video_mp4 status='ready' 상태

Scenario: 검수 대기 진입
  When 시스템이 모든 청크의 render 완료를 감지한다
  Then lecture.status='pending_review' UPDATE
  And 학원장 김지영에게 알림톡 + 이메일 발송: "검수 대기 영상 1건 (박민준 강사)"
  And 학원장 대시보드의 "검수 큐" 에 영상 #501 이 표시된다
```

### F15-04: HyperFrames render 실패

```gherkin
Scenario: HyperFrames 가 failed webhook 발송
  Given 청크 1001 의 render 가 HyperFrames 내부 오류로 실패
  
  When webhook 으로 { render_id, status: "failed", error: "rendering error" } 수신
  Then video_asset (chunk_pk=1001) status='failed' UPDATE
  And BullMQ 가 'render-video' job 을 1회 재시도 (HyperFrames 비용 ↑ 고려)
  
  When 재시도도 실패
  Then lecture.status='failed', failure_reason='HyperFrames 렌더링 실패 (청크 2)'
  And 강사 + 학원장에게 알림 발송
  And 수동 처리 큐에 등록 (학원장이 사유 검토 후 재시도 또는 포기)
```

### F15-05: HyperFrames timeout (장시간 미응답)

```gherkin
Scenario: render_id 가 1시간 이상 응답 없음
  Given 'render-video' job 의 'awaiting-webhook' 상태가 60분 경과
  
  When timeout 워커가 감지
  Then HyperFrames API 의 GET /v1/renders/<id> 상태 조회 호출 (polling 보완)
  
  When 상태가 여전히 "queued" 또는 "rendering"
  Then 추가 30분 대기 + 재 polling
  
  When 상태가 "failed" 또는 90분 totalize 초과
  Then lecture.status='failed', 학원장 알림: "HyperFrames 렌더링 지연 — 수동 확인 필요"
```

---

## Feature 16: 학원장 검수 + Publish (수동 default)

> 학원장이 영상을 검수해서 publish 결정.

### F16-01: 학원장이 검수 큐 진입

```gherkin
Background:
  Given lecture #501 의 status='pending_review'
  And 김지영이 학원장으로 로그인되어 있다

Scenario: 검수 큐 접근
  When 김지영이 학원장 대시보드 → "검수 대기 영상" 클릭
  Then 검수 대기 영상 목록이 표시된다 (3건 가정)
  And 각 영상 카드에 표시: 강사명, 수업명, 수업 일시, 생성 완료 시각, 청크 수
  
  When 김지영이 영상 #501 (박민준 / 중3수학반) 카드를 클릭한다
  Then 영상 상세 검수 페이지로 진입한다
```

### F16-02: 영상 미리보기 + 메타 편집

```gherkin
Background:
  Given 김지영이 영상 #501 검수 페이지에 있다
  And 청크 3개의 video_mp4 + thumbnail 후보 3종 (S3) 이 ready 상태

Scenario: 검수 화면 표시
  Then 다음 요소가 표시된다:
    | 영상 미리보기 | 청크 3개 시퀀스 또는 첫 청크 video player |
    | 썸네일 후보   | 3개 (라디오 선택)                          |
    | 영상 제목     | 자동 생성된 제목 (편집 가능)                |
    | 영상 설명     | 자동 생성 (편집 가능)                       |
    | 태그          | 자동 추출 (편집 가능)                       |
    | 공개 범위     | 드롭다운 (public/unlisted/private)         |
    | 청크 별도 영상 | 토글 (concat 1편 vs 청크별 N편)            |
```

### F16-03: 학원장이 승인하여 publish

```gherkin
Scenario: 승인 → YouTube 업로드 트리거
  Given 김지영이 검수 페이지에 있다
  
  When 김지영이 다음을 수정 후 "승인" 버튼을 누른다:
    | 썸네일       | #2 선택                                     |
    | 제목         | [중3 수학] 이차함수 정점 — y=ax²+bx+c 변환 5문제 |
    | 설명         | (자동 생성된 내용 그대로)                    |
    | 공개 범위    | unlisted                                    |
    | 청크 별도    | 토글 OFF (FFmpeg concat → 1편)              |
  
  Then lecture.status='uploading' UPDATE
  And BullMQ 'upload-youtube' job 적재
  And video_asset 의 thumbnail (선택된 #2) 가 youtube_video 의 thumbnail_url 로 매핑
  And 김지영의 화면이 "업로드 진행 중" 으로 전환
```

### F16-04: 학원장이 반려

```gherkin
Scenario: 반려 + 사유 입력
  Given 김지영이 영상 #501 의 청크 2 의 음성 품질이 부족하다고 판단
  
  When 김지영이 "반려" 버튼을 누른다
  And 사유 입력 다이얼로그에 "청크 2 의 음성 일부 끊김 — 재업로드 권장" 입력
  And "반려 확정" 클릭
  
  Then lecture.status='rejected', failure_reason='학원장 반려: 청크 2 의 음성 일부 끊김 — 재업로드 권장'
  And 강사 박민준에게 알림톡: "강의 #501 가 학원장 검수에서 반려되었습니다. 사유: ..."
  And youtube_video 는 INSERT 되지 않는다
  And 강사가 새 강의 업로드 시 #501 의 메타 정보 자동 채움 (재업로드 편의, 옵션)
```

### F16-05: 학원장 검수 SLA 초과

```gherkin
Background:
  Given lecture #501 의 status='pending_review' 가 48시간 이상 경과 (학원장 무응답)

Scenario: 자동 reminder
  When 48시간 timer 도달
  Then 학원장 김지영에게 reminder 알림톡: "검수 대기 영상 N건 (가장 오래된 것 48시간 경과)"
  And 강사에게 알림: "학원장 검수가 지연되고 있습니다 (관리자 문의)"
  
  When 추가 5일 (총 7일) 경과
  Then lecture.status 는 그대로 (자동 publish/취소 X), 단지 알림만 반복 발송
  And 학원장이 수동 처리 필요
```

### F16-06: 학원장이 청크별 별도 영상으로 publish

```gherkin
Scenario: N편 chapter playlist
  Given 김지영이 검수 페이지에서 "청크 별도" 토글을 ON 으로 설정
  
  When 승인을 누른다
  Then 3개 청크가 각각 별도 YouTube 영상으로 업로드된다
  And youtube_video 테이블에 3 row 가 INSERT (각각 chunk_pk 채워짐)
  And YouTube 상에서 playlist 로 자동 묶임 (또는 description 에 chapter 링크)
```

---

## Feature 17: 강사 자동 Publish (Trust Grant)

> auto_publish_own 보유 강사의 영상은 검수 우회.

### F17-01: 신뢰 강사의 자동 publish (Happy)

```gherkin
Background:
  Given 박민준이 auto_publish_own grant 보유 (F5-01 참조)
  And youtube_channel.auto_upload_mode='per_teacher'

Scenario: 검수 단계 skip
  When 박민준의 영상 #502 의 모든 청크 render 완료
  Then 시스템이 박민준의 ability 평가 (sensitive write → DB 재검증)
  And ability.can('publish', 'YoutubeVideo', { teacher: 박민준 }) = true 확인
  And lecture.status='uploading' (pending_review 단계 skip)
  And BullMQ 'upload-youtube' job 즉시 적재
  And 김지영에게 검수 알림 발송 X
  And 박민준에게만 "영상 자동 publish 완료" 알림 발송
```

### F17-02: auto_publish_all (학원 전체 자동)

```gherkin
Background:
  Given youtube_channel.auto_upload_mode='auto' (학원 전체 자동)

Scenario: 모든 강사의 영상 자동 publish
  When 어느 강사가 영상을 publish 단계로 진입시킨다
  Then 검수 단계 skip, 즉시 YouTube 업로드
  And 학원장에게 검수 알림 X (감사 로그만 기록)
```

---

## Feature 18: YouTube 업로드 + Quota

> YouTube Data API v3 resumable upload + thumbnail + metadata.

### F18-01: YouTube 업로드 (Happy)

```gherkin
Background:
  Given lecture #501 의 status='uploading'
  And youtube_channel.oauth_refresh_token_kms 가 유효함

Scenario: Resumable upload
  When 'upload-youtube' job 이 실행된다
  And refresh_token 으로 access_token 획득 (KMS 복호화)
  And YouTube Data API videos.insert 호출:
    """
    snippet: {
      title: "[중3 수학] 이차함수 정점 — y=ax²+bx+c 변환 5문제",
      description: "...",
      tags: ["수학", "이차함수", "중3", "한울학원"],
      categoryId: "27"
    }
    status: {
      privacyStatus: "unlisted",
      selfDeclaredMadeForKids: false,
      license: "youtube"
    }
    """
  And resumable upload URL 획득
  And MP4 (concat 영상 또는 청크별) 청크 PUT 업로드
  And 업로드 완료 후 youtube_video_id 응답 수신
  
  Then thumbnails.set 호출로 #2 썸네일 (S3 URL → YouTube) 업로드
  And youtube_video 테이블 UPDATE:
    | status            | published                                |
    | youtube_video_id  | <YouTube vid>                            |
    | published_at      | <current>                                |
    | thumbnail_url     | https://i.ytimg.com/...                  |
  And lecture.status='published'
  And BullMQ 'notify' + 'index-rag' job 적재
  And SLA: 검수 승인 → publish 완료 < 5분 (p95)
```

### F18-02: YouTube quota 초과

```gherkin
Background:
  Given A학원의 오늘 YouTube quota 사용: 9,600 units (한도 10,000 의 96%)
  And videos.insert 는 1,600 units 소비

Scenario: Quota 부족
  When 'upload-youtube' 가 시작되어 quota 체크
  Then YouTube API 가 quotaExceeded 에러 반환
  And BullMQ job 이 'await-quota-reset' 상태로 대기
  And 자정 (PT, YouTube 기준) 이후 quota reset 까지 hold
  And 학원장에게 알림: "오늘 YouTube quota 도달 — 내일 자동 처리"
  
  When quota reset 후 (PT 자정)
  Then job 이 자동 재개되어 정상 업로드
```

### F18-03: YouTube 업로드 실패 (네트워크)

```gherkin
Scenario: 네트워크 일시 장애
  Given resumable upload 중 네트워크 끊김
  
  When 시스템이 끊김 감지
  Then YouTube resumable upload 의 resume 기능으로 이어서 업로드
  
  When 3회 재시도도 실패
  Then lecture.status='failed', failure_reason='YouTube 업로드 실패'
  And 강사 + 학원장 알림
  And 수동 재시도 옵션 제공
```

### F18-04: YouTube 정책 위반 자동 거부

```gherkin
Scenario: YouTube 가 영상을 정책 위반으로 차단
  Given YouTube 가 업로드된 영상의 AI 콘텐츠 미명시를 감지 (가설 시나리오)
  
  When YouTube API 가 videos.update 시 추가 검토 요구
  Then 영상이 unlisted → 정책 검토 상태로 자동 전환
  And 학원장에게 알림: "YouTube 가 영상 #501 의 추가 검토를 요구합니다"
  And youtube_video.status='under_youtube_review'
  And v0.5+ 에서 AI 콘텐츠 명시 (description 자동 삽입) 보강 예정
```

### F18-05: 학원당 일 YouTube 업로드 cap

```gherkin
Scenario: 학원 일 cap 6편 도달
  Given A학원 오늘 YouTube publish 누적 = 6편
  
  When 7번째 영상이 publish 단계 진입
  Then 시스템 cap 검증 → cap 도달 확인
  And lecture.status='cap_pending' 으로 hold
  And 자정 (KST) 이후 자동 재개
```

---

## Feature 19: 강의 Publish 후 RAG 인덱싱

> publish 완료 직후 chunks 가 Qdrant + Neo4j 에 인덱싱.

### F19-01: 자동 인덱싱 트리거

```gherkin
Background:
  Given lecture #501 의 status='published' 로 전환됨

Scenario: 인덱싱 자동 진행
  When 'upload-youtube' 완료 직후
  Then BullMQ 에 'index-rag' job 이 청크별로 적재 (3개 청크 → 3개 job)
  And 'extract-neo4j-concepts' job 도 청크별 적재
  
  When 'index-rag' job 이 실행된다 (청크 1001 기준)
  And Upstage Solar Embedding 호출 (srt_content → vector)
  And Qdrant 에 point upsert (payload: org_id=1, teacher_id=박민준, lecture_id=501, chunk_id=1001, visibility='unlisted', subject='수학', ...)
  
  Then lecture_chunk.qdrant_point_ids = ["<uuid>"]
  And lecture_chunk.rag_indexed_at = <current>
  And lecture_chunk.embedding_model = 'upstage-solar-1024'
  And SLA: publish → 모든 청크 indexed < 1분/청크 (p95)
```

### F19-02: 후속 강의에서 RAG retrieval 활용

```gherkin
Background:
  Given 박민준의 강의 #501 이 indexed 됨
  And 박민준의 강의 #502 가 청크 생성 단계 진입

Scenario: #502 의 HTML 생성 시 #501 참조
  When 'generate-html' job (강의 #502 청크 1) 이 실행된다
  And Qdrant search 호출 (filter: org_id=1 AND teacher_id=박민준)
  
  Then 결과에 강의 #501 의 chunk 가 포함된다 (관련 토픽 시)
  And Claude 프롬프트에 retrieved_past_lectures 섹션에 #501 의 chunk 정보 주입
  And 강의 #502 의 HTML 이 강의 #501 의 맥락을 활용하여 일관성 ↑
```

### F19-03: 학원장이 RAG 검색 (v0.5 예시)

```gherkin
Scenario: 학원장 자연어 질의
  Given A학원 lecture_chunk 누적 = 100개 (인덱싱 완료)
  
  When 김지영이 학원장 대시보드의 "RAG 검색" 입력란에 "이번 달 학생들이 어려워한 단원?" 입력
  
  Then 시스템이 RAG retrieval (filter: org_id=1, top_k=10) + Neo4j 보강 + Claude 답변 생성
  And 답변: "5월 누적 분석 결과: 이차함수 정점 변환 (5회 검색)... 보강 강의 추천: 김지영 강사 5/21 강의."
  And rag_query_log INSERT (질의 + 응답 + 참조 영상)
  
  Note: 이 시나리오는 v0.5 기능. MVP 는 인덱싱만 검증.
```

---

# Part D. Notifications (spec-01-05 영역)

## Feature 20: 학부모 알림톡 발송

> 영상 publish 후 학부모에게 알림톡 자동 발송.

### F20-01: publish 직후 알림톡 발송 (Happy)

```gherkin
Background:
  Given lecture #501 의 status='published', class_group='중3수학반'
  And student 테이블에 (org_pk=1, class_group='중3수학반') 인 김민서가 등록되어 있음
  And 김민서의 student.parent_phone='010-1234-5678', student.consent_video_appearance=1
  And NHN Cloud 알림톡 비즈채널이 활성 상태

Scenario: 알림톡 자동 발송
  When 'notify' job 이 lecture #501 에 대해 실행된다
  Then 시스템이 다음 쿼리로 알림 대상 조회:
    """
    SELECT s.parent_phone, s.display_name
    FROM student s
    WHERE s.org_pk = :orgPk
      AND s.class_group = :classGroup     -- lecture.class_group 값
      AND s.status = 'ACTIVE'
      AND s.parent_phone IS NOT NULL
    """
  And 김민서의 학부모 박엄마 전화번호 추출: 010-1234-5678
  And NHN Cloud Bizmessage API 호출 (템플릿: "수업영상알림"):
    """
    [한울학원] 김민서 학생 오늘 수업 영상
    📺 중3 수학 — 이차함수 정점
    50분 분량, 강사: 박민준
    
    [영상 보기] (YouTube 링크)
    """
  And 알림톡 응답: success, provider_msg_id
  And notification 테이블 INSERT:
    | channel              | alimtalk                                |
    | template_id          | 수업영상알림                            |
    | parent_phone         | 010-1234-5678 (발송 시점 스냅샷)        |
    | payload              | {student_name, class, video_link}       |
    | provider_msg_id      | <NHN msg id>                            |
    | status               | sent                                    |
    | related_video_pk     | <video_pk>                              |
    | sent_at              | <current>                               |
  And SLA: publish → 알림 발송 < 1분 (p95)
```

### F20-02: 학원 반 N명 학생 → N명 학부모 알림

```gherkin
Scenario: 다수 학부모 발송
  Given lecture #501 의 class_group='중3수학반'
  And student 테이블에 (org_pk=1, class_group='중3수학반', status='ACTIVE') 인 학생 8명 존재
  And 각 학생의 student.parent_phone 이 등록되어 있음 (8건)
  
  When 'notify' job 이 실행된다
  Then NHN Bizmessage 일괄 발송 API 호출 (또는 8회 개별 호출)
  And notification 테이블에 8 row INSERT (parent_phone = 발송 시점 스냅샷)
  And 각 알림톡의 student_name 변수가 해당 학생으로 치환됨 (개인화)
```

### F20-03: 알림톡 템플릿 변수

```gherkin
Scenario: 변수 치환
  Given 알림톡 템플릿:
    """
    [#{academy_name}] #{student_name} 학생 오늘 수업 영상
    📺 #{class_subject} — #{class_topic}
    #{duration}분 분량, 강사: #{teacher_name}
    
    [영상 보기] #{video_link}
    """
  
  When 발송 직전 변수 치환
  Then 다음 값으로 치환:
    | academy_name  | 한울학원                        |
    | student_name  | 김민서                          |
    | class_subject | 중3 수학                        |
    | class_topic   | 이차함수 정점                   |
    | duration      | 50                              |
    | teacher_name  | 박민준                          |
    | video_link    | https://youtu.be/<video_id>     |
```

---

## Feature 21: 알림 Fallback Chain

> 알림톡 실패 시 SMS → 이메일 순으로 fallback.

### F21-01: 알림톡 실패 → SMS 폴백

```gherkin
Scenario: 학부모가 카카오톡 미사용 (수신 거부)
  Given 박엄마가 카카오톡 비즈채널 차단 상태
  
  When 'notify' job 이 알림톡 발송
  Then NHN 응답: failed (사용자 차단)
  And notification.status='failed', failure_reason='kakao_blocked'
  And BullMQ 'send-sms-fallback' job 적재
  
  When SMS fallback job 실행
  And SOLAPI 호출:
    """
    [한울학원] 김민서 학생 오늘 수업 영상이 업로드되었습니다.
    https://youtu.be/<video_id>
    """
  And SMS 발송 성공
  Then notification (channel='sms') 신규 INSERT, status='sent'
  And related_video_pk 동일
```

### F21-02: SMS 도 실패 → 이메일 폴백

```gherkin
Scenario: SMS 미수신
  Given 박엄마의 전화번호가 잘못 등록되어 있음 (010-XXX-XXXX 형식 오류)
  And 알림톡, SMS 모두 실패
  
  When 'send-email-fallback' job 실행
  And 박엄마의 이메일 (학생 등록 시 입력) 조회 → AWS SES 발송
  
  Then 이메일 발송 성공
  And notification (channel='email') INSERT, status='sent'
```

### F21-03: 모든 채널 실패

```gherkin
Scenario: 학부모 정보가 완전 누락
  Given 학생 김민서의 학부모 정보가 학원에 등록되지 않음
  
  When 'notify' job 실행
  Then 알림 발송 시도 X (수신자 정보 없음)
  And notification 테이블에 INSERT (status='failed', failure_reason='no_contact_info')
  And 강사 박민준에게 알림: "학생 김민서 의 학부모 연락처가 등록되지 않아 알림 발송 불가"
```

---

## Feature 22: 알림 추적 (read_at)

> 학부모가 알림톡의 영상 링크 클릭 시 추적.

### F22-01: 영상 링크 클릭 추적

```gherkin
Background:
  Given 알림톡이 박엄마에게 발송됨 (notification.provider_msg_id 존재)

Scenario: 학부모 클릭
  When 박엄마가 알림톡의 [영상 보기] 버튼을 누른다
  Then YouTube 영상 페이지로 이동한다 (정확히는 추적 URL 경유 → 학원-api 의 /track/click/<notification_pk> → 302 redirect to YouTube)
  
  And 추적 URL 처리 시 notification 테이블 UPDATE: read_at=<current>
  And YouTube 영상 정상 시청 가능
```

### F22-02: 학부모 클릭률 집계

```gherkin
Scenario: 학원장 대시보드의 알림 통계
  When 김지영이 "운영 통계 → 알림 추적" 페이지 접속
  Then 다음 통계가 표시된다:
    | 영상 #501 | 발송 8건 | 읽음 5건 (62.5%) |
    | ...       | ...      | ...               |
  And 학원장은 어떤 학부모가 자녀 영상을 보지 않았는지 (개인정보 X) 비율로 파악 가능
```

---

## Feature 23: 학원장·강사 알림

> 검수 알림, publish 완료 알림, 실패 알림 등 내부 알림.

### F23-01: 학원장 검수 알림

```gherkin
Scenario: 검수 대기 알림
  Given lecture #501 의 status='pending_review'
  
  When 'notify' (audience=director) job 실행
  Then 김지영에게 알림톡:
    """
    [한울학원] 검수 대기 영상 1건
    📺 박민준 강사 / 중3 수학 / 이차함수 정점
    
    [검수하기] (학원장 대시보드 링크)
    """
  And 김지영의 이메일에도 동일 내용 발송
```

### F23-02: 강사 영상 완료 알림

```gherkin
Scenario: 영상 publish 알림 (강사)
  Given lecture #501 이 publish 됨
  
  When 'notify' (audience=teacher) job 실행
  Then 박민준에게 알림톡:
    """
    [한울학원] 영상 publish 완료
    📺 [중3 수학] 이차함수 정점 — y=ax²+bx+c 변환 5문제
    조회수: 0 (방금 publish)
    
    [영상 보기] (YouTube)
    """
```

### F23-03: 실패 알림

```gherkin
Scenario: 파이프라인 실패 알림
  Given lecture #501 의 status='failed' (예: HyperFrames 영구 실패)
  
  When 'notify' (audience=teacher+director) job 실행
  Then 박민준 + 김지영 양쪽에 알림:
    """
    [한울학원] 영상 생성 실패
    📺 박민준 강사 / 중3 수학 / 이차함수 정점
    사유: HyperFrames 렌더링 실패 (청크 2)
    
    [수동 재시도] (학원장 대시보드)
    """
```

---

# Part E. Cross-cutting / Policy

## Feature 24: 비용 Cap

> 학원당 일 영상 한도 + Claude 호출 한도 + HyperFrames 한도 등.

### F24-01: 일 영상 한도

```gherkin
Background:
  Given A학원의 org_entitlement.feature_limits.daily_uploads=6
  And 오늘 publish 누적 = 6편

Scenario: 7번째 영상 대기 (F11-04 참조)
  When 7번째 강의가 publish 단계 진입
  Then canXXX() Gate B: org_entitlement.feature_limits 조회 → daily_uploads 한도 초과 확인
  And 다음 메시지가 학원장 + 강사에게 표시: "오늘 영상 한도 도달. 내일 0시 (KST) 부터 자동 처리"
  And lecture.status='cap_pending'
```

### F24-02: 학원장이 cap 조정

```gherkin
Scenario: cap 상향
  Given A학원이 Pro plan 으로 업그레이드되어 더 많은 영상 처리 가능
  
  When 김지영이 "학원 설정 → 일 영상 한도" 를 6 → 12 로 변경
  Then org_entitlement.feature_limits = {"daily_uploads":12, ...} UPDATE
  And org_entitlement 캐시 무효화 (Redis TTL 만료 또는 perm_version bump)
  And 다음 영상부터 새 한도 적용
```

### F24-03: Claude 비용 cap

```gherkin
Background:
  Given lecture 1편당 Claude 비용 평균 $0.90 (3 청크)
  And A학원의 월 Claude 한도 = $150 (org_entitlement.feature_limits.claude_monthly_usd)

Scenario: 월 한도 도달
  Given 이번 달 A학원의 Claude 누적 = $148
    """
    -- 집계 쿼리 (usage_log 테이블)
    SELECT SUM(cost_usd) FROM usage_log
    WHERE org_pk=1 AND service='CLAUDE'
      AND created_at >= '2026-05-01' AND created_at < '2026-06-01'
    -- 결과: $148.00
    """
  And 다음 영상이 청크 3개 → 예상 $0.90 추가 → $148.90 (한도 내)
  
  When 영상 처리 진행
  Then 정상 처리됨
  And usage_log 에 서비스='CLAUDE', cost_usd=$0.90 row 3건 INSERT
  
  Given 이번 달 누적 = $149.50 + 다음 영상 $0.90 = $150.40 (한도 초과)
  
  When 다음 영상이 청크 단계 진입
  Then 시스템이 usage_log 집계로 cap 검증 → 한도 초과 예상
  And lecture.status='cap_pending', failure_reason='Claude 월 한도 초과 (이번 달)'
  And 학원장 알림: "Claude 비용 한도 도달. 다음 달까지 대기 또는 plan 조정 검토."
```

### F24-04: HyperFrames render 한도

```gherkin
Scenario: 일 HyperFrames 청크 cap
  Given A학원의 일 HyperFrames render 한도 = 18 청크 (6편 × 평균 3 청크)
  And 오늘 누적 = 18 청크
  
  When 다음 영상의 'render-video' job 시도
  Then cap 검증 → cap 도달
  And lecture.status='cap_pending'
```

---

## Feature 25: 자녀 등장 동의 (PIPA)

> 학생 음성 / 영상 등장 시 본인·법정대리인 동의 필수.

### F25-01: 입학 시 동의

```gherkin
Background:
  Given 학생 김민서가 한울학원에 입학 절차 중

Scenario: 동의 폼 입력
  When 학원 직원 (학원장 또는 강사) 이 학생 등록 화면에서 다음을 입력:
    | 학생 이름        | 김민서                    |
    | 학년             | 중3                       |
    | 학부모 이름      | 박엄마                    |
    | 학부모 전화      | 010-1234-5678             |
    | 학부모 이메일    | mom.park@example.com      |
    | 자녀 등장 동의   | ✓ (체크박스, 학부모 서명) |
    | YouTube 노출 동의| ✓                         |
    | 미성년 (만 14세 미만) | N (중3 = 15세)         |
  
  Then student 테이블 INSERT (v0.1 필수 포함):
    | org_pk                   | 1 (한울학원)              |
    | display_name             | 김민서                    |
    | grade_level              | 중3                       |
    | parent_name              | 박엄마                    |
    | parent_phone             | 010-1234-5678             |
    | parent_email             | mom.park@example.com      |
    | consent_video_appearance | 1                         |
    | status                   | ACTIVE                    |
  And 동의 시점 audit log 저장
```

### F25-02: 14세 미만 학생 — 법정대리인 동의 강제

```gherkin
Scenario: 초등학생 등록
  Given 학생 김초롱이 만 12세 (초6) 이고 학원 등록 중
  
  When 등록 화면에서 학생 정보 입력
  Then "만 14세 미만 — 법정대리인 동의서 PDF 업로드 필수" 안내 표시
  And 법정대리인 (학부모) 서명된 PDF 업로드 입력란 표시
  
  When PDF 업로드 + 모든 항목 동의 후 저장
  Then student INSERT 가능
  And consent_pdf_s3_url 기록
```

### F25-03: 미동의 학생 영상 처리

```gherkin
Background:
  Given 강의 영상에 학생 김민서 (동의) 와 미동의 학생 박예준 양쪽 등장
  And 박예준은 자녀 등장 동의를 거부했다 (consent_video_appearance=false)

Scenario: 미동의 학생 알림
  When lecture 의 publish 직전 단계
  And 시스템이 lecture.class_group → 등록 학생들 중 consent=false 학생 검색
  Then 미동의 학생 박예준 발견
  And 학원장 김지영에게 알림: "강의 #501 의 반에 자녀 등장 미동의 학생 박예준이 포함되어 있습니다. publish 전 확인 필요."
  And lecture.status='pending_review' (수동 검수 필수)
  
  When 학원장이 검수 시점에 "박예준 음성 모자이크/skip 처리" 또는 "publish 보류" 결정
  
  Note: Phase 2 카메라 모드 시 자동 모자이크 파이프라인 도입 예정. MVP 는 알림만.
```

### F25-04: 동의 회수

```gherkin
Background:
  Given 학생 김민서의 학부모 박엄마가 자녀 등장 동의를 회수하고자 한다
  And 김민서가 등장한 영상 5편이 publish 되어 있다

Scenario: 동의 회수 요청
  When 박엄마가 학원에 회수 요청 (오프라인 또는 학부모 PWA v1.0+)
  Then student.consent_video_appearance=false UPDATE
  And 학원장에게 알림: "박엄마 가 자녀 등장 동의를 회수했습니다. 기존 영상 처리 결정 필요."
  
  When 학원장이 "기존 영상 비공개 전환" 선택
  Then 김민서가 등장한 영상 5편 모두 youtube_video.visibility='private' UPDATE
  And YouTube API videos.update 호출로 즉시 private 전환
  And 학부모 박엄마에게 처리 완료 알림
```

---

## Feature 26: 학원 폐업 / 사용자 탈퇴

> 학원 폐업 또는 개별 사용자 탈퇴 시 데이터 처리.

### F26-01: 학원 폐업 — 데이터 export

```gherkin
Background:
  Given A학원이 폐업을 결정
  And 학원장 김지영이 데이터 백업을 원함

Scenario: 학원 데이터 export
  When 김지영이 "학원 설정 → 데이터 export" 클릭
  Then 시스템이 다음 자료를 zip 으로 패키징 (백그라운드):
    | 강의 영상 (S3) 의 다운로드 링크 (60일 유효)         |
    | 강의 PDF 강의노트                                  |
    | 강의 transcript JSON                              |
    | 강사·학생 정보 (CSV)                              |
    | 알림 발송 이력                                     |
  And 패키징 완료 시 학원장 이메일로 다운로드 링크 발송
```

### F26-02: 학원 hard delete (30일 후)

```gherkin
Background:
  Given A학원이 폐업 신청 30일 경과

Scenario: 자동 hard delete
  When 일 1회 배치 job 이 실행된다
  And A학원의 deleted_at = 30일 전 확인
  
  Then 다음 처리:
    | organization row hard delete (+ academy_config CASCADE)              |
    | identity_user 의 A학원 전속 사용자 (다른 org membership X) 의 status='DELETED' |
    | membership 의 A학원 row hard delete                                |
    | lecture / lecture_chunk / video_asset / student hard delete         |
    | S3 의 A학원 prefix 전체 삭제                                       |
    | Qdrant 의 org_id=1 의 모든 point hard delete                       |
    | Neo4j 의 org_id=1 의 모든 노드/엣지 detach + delete               |
    | YouTube 채널의 영상은 학원장 권한으로 별도 처리 (자동 삭제 X)       |
  
  But audit_log 는 보존 (의무 기간 3년)
  And notification 의 학부모 phone/email 도 anonymize (개인정보 삭제)
```

### F26-03: 개별 사용자 탈퇴

```gherkin
Scenario: 강사 본인 탈퇴
  When 박민준이 "계정 설정 → 탈퇴" 클릭 + 사유 입력
  
  Then identity_user.status='DELETED' (soft delete)
  And 박민준의 모든 membership.status='SUSPENDED'
  And 강사가 업로드한 강의는 보존 (학원 자산) — A학원 소유
  And 박민준 의 Firebase 계정은 30일 후 hard delete (안전망)
```

---

## Feature 27: 보안 위반 감지 / 차단

> 비정상 접근 패턴 / 권한 위반 감지.

### F27-01: 동일 IP 에서 다수 실패 시도

```gherkin
Scenario: Brute force 로그인 시도 차단
  Given 동일 IP 가 10분간 50회 로그인 실패
  
  When 51번째 시도
  Then Firebase Auth 가 rate limit (또는 reCAPTCHA challenge)
  And 시스템 로그: "Brute force 의심 — IP <ip>"
```

### F27-02: 권한 escalation 시도

```gherkin
Scenario: TEACHER 가 DIRECTOR 전용 API 반복 호출
  Given 박민준이 TEACHER 권한
  
  When 박민준이 POST /admin/youtube-channels (DIRECTOR only) 를 5분 내 10회 시도
  Then 모두 403 Forbidden 응답
  And 시스템 로그: "권한 위반 의심 — user=박민준, action=manage YoutubeChannel, 시도 10회"
  And 학원장에게 알림: "박민준 강사가 학원장 전용 기능에 반복 접근 시도"
```

### F27-03: 다른 학원 ID 추측 enum 시도

```gherkin
Scenario: ID enumeration 시도
  Given 박민준이 org_pk=1 의 강사
  
  When 박민준이 짧은 시간 내 org_pk=2, 3, 4, 5, ... 의 데이터를 차례로 요청
  Then 각각 403 응답
  And 시스템 로그: "ID enumeration 의심 — 사용자 박민준"
  And 학원장 + 시스템 admin 알림
```

---

## Feature 28: Publish 후 부적절 발견 처리

> publish 된 영상에서 사후 부적절 콘텐츠 발견 시 즉시 비공개 전환.

### F28-01: 학원장의 1-tap 비공개 전환

```gherkin
Background:
  Given lecture #501 이 publish 된 지 1일 경과
  And 학원장 김지영이 영상에서 학생 개인정보 (전화번호) 노출을 확인

Scenario: 즉시 비공개
  When 김지영이 학원장 대시보드 → 영상 #501 → "비공개 전환" 버튼 클릭
  And 사유: "학생 개인정보 노출"
  
  Then YouTube Data API videos.update 호출 (privacyStatus='private')
  And youtube_video.visibility='private' UPDATE
  And youtube_video.status='private_by_director'
  And lecture.failure_reason 보존 (감사 목적)
  And 박민준 (강사) 에게 알림: "영상 #501 이 학원장에 의해 비공개 전환됨"
  And 학부모에게 추가 알림 없음 (이미 발송된 링크는 학부모만 접근, public 검색에는 안 보임)
```

### F28-02: LLM-as-judge 자동 검열 (Publish 직전)

```gherkin
Background:
  Given lecture #501 의 HyperFrames render 완료
  And 학원의 검수 정책 = 'auto' (auto_upload_mode='auto')

Scenario: 자동 검열
  When 'pre-publish-check' job 실행 (publish 직전)
  And Claude Haiku 가 영상 transcript + 슬라이드 텍스트를 검열
  And 검열 항목: 욕설, 혐오, 학생 개인정보 (이름·전화·주소)
  
  When Claude 가 학생 전화번호 노출 감지
  Then lecture.status='pending_review' 강제 (auto 모드여도 검열 fail 시 수동 검수 진입)
  And 학원장에게 알림: "자동 검열에서 학생 개인정보 가능성 감지. 수동 검수 필요."
  And 검열 결과가 검수 화면에 highlight 표시
```

---

# Part F. End-to-End User Journey

## Feature 29: 학원장 Onboarding Journey

> 신규 학원장의 가입 → 첫 영상 publish 까지 단대단.

### F29-01: 0 → 1 학원 onboarding

```gherkin
Scenario: 신규 학원의 첫 영상까지
  
  # Day 1: 가입
  Given 김지영이 학원 가입 페이지 접속
  When 가입 + 학원 정보 입력 + 약관 동의 + 가입 완료 (5분)
  Then identity_user + academy + membership 자동 생성
  And 학원장 대시보드 진입
  
  # Day 1: YouTube 채널 연결
  When 김지영이 YouTube OAuth 흐름 진행 (2분)
  Then youtube_channel 등록 + refresh_token KMS 저장
  
  # Day 1-2: 강사 초대
  When 김지영이 강사 박민준 (이메일) 초대
  And 박민준이 초대 메일 수신 + 첫 로그인 + 비번 설정 (10분)
  Then 박민준의 identity_user 활성 + membership 생성
  
  # Day 2: 강사 자료 사전 업로드 (선택)
  When 박민준이 강의안 PDF 3편 업로드 (15분 + 인덱싱 5분)
  Then 자료 인덱싱 완료 (Qdrant + Neo4j)
  
  # Day 3: 첫 강의
  When 박민준이 첫 강의 (50분 mp3) 업로드
  Then ~90분 후 학원장 검수 큐 적재
  
  # Day 3: 학원장 검수
  When 김지영이 검수 + 승인 (3분)
  Then YouTube 업로드 (~5분) → publish
  
  # Day 3: 학부모 알림
  When 학부모 박엄마에게 알림톡 발송
  And 박엄마가 영상 클릭 → YouTube 시청
  
  Then 학원 onboarding 완료
  And 학원장은 "5분 안에 학원 상태 파악" 의 첫 가치를 체감
  And 학부모는 "앱 없이 자녀 수업 시청" 첫 가치 체감
  And 강사는 "수업 = 영상화 = 포트폴리오" 첫 가치 체감
  
  And SLA: 가입 → 첫 영상 publish 완료 ≤ 3일 (자료 준비 포함)
```

---

## Feature 30: 강사 Daily Flow

> 매일 수업 → 업로드 → publish → 다음 수업의 cycle.

### F30-01: 강사의 하루

```gherkin
Scenario: 박민준의 평일 수업 흐름
  
  # 18:00: 수업
  Given 박민준이 18:00 ~ 18:50 중3수학반 수업 진행
  And 수업 중 본인 voice recorder 로 별도 녹음
  
  # 18:55: 업로드
  When 박민준이 본인 PC 에서 강사 대시보드 접속
  And 강의 업로드 화면에서 mp3 + 메타 입력 + 업로드 (2분)
  
  # 18:55 ~ 20:25: 자동 파이프라인
  Then 시스템이 STT → 청킹 → TTS+HTML → render → 검수 대기 흐름 자동 진행
  And 박민준이 다른 강의 (19:00 ~ 19:50) 진행하는 동안 백엔드 처리
  
  # 20:30: 검수 대기 알림
  When 학원장 김지영에게 검수 알림 도달
  And 김지영이 검수 + 승인 (3분)
  
  # 20:40: publish 완료
  Then 박민준에게 publish 완료 알림 도달
  And 학부모들에게 알림톡 발송
  
  # 다음 날
  And 박민준이 다음 날 본인 강의 통계 (지난 강의 조회수) 확인 가능
  
  And 강사 입력 노동: ~2분/강의 (업로드 메타 입력만)
  And 30분 내 가치 회수 (v3 vision §6 명제 #1, #8 충족)
```

---

## Feature 31: 학부모 Weekly Experience

> 학부모가 자녀 학원 활동을 weekly 단위로 관찰.

### F31-01: 박엄마의 일주일

```gherkin
Scenario: 일주일간 학부모 알림 수신
  
  # 월: 수학 영상
  Given 박민준이 월요일 18:00 수학 강의 publish
  When 박엄마에게 알림톡 발송 (~20:30)
  And 박엄마가 카카오톡에서 알림 확인 + [영상 보기] 클릭
  Then YouTube 모바일 앱에서 영상 시청
  And notification.read_at 기록
  
  # 화: 영어 영상
  Given 다른 강사가 화요일 영어 강의 publish
  When 박엄마에게 영어 영상 알림 발송
  
  # 수~금: 매일 1편씩
  
  # 일요일 저녁
  Then 박엄마가 일주일간 자녀 학원 활동을 시각적으로 파악
  And 학원 신뢰도 ↑ — 영상으로 학습 진행 확인
  And 학원 채널 구독 (옵션) → SEO + 학원 마케팅
  
  # KPI 측정 (학원장 대시보드)
  And 학원장은 알림 클릭률 (62.5%) 확인 가능
  And 영상 attribution (학부모 → 신규 학생 등록 = 0 또는 N) 6개월 누적 추적
```

---

# Part G. Non-Functional Requirements (NFR)

## Feature 32: 파이프라인 SLA

### F32-01: 90분 이내 publish (p95)

```gherkin
Scenario: 50분 강의 SLA
  Given 박민준이 13:00 에 50분 강의를 업로드
  
  When 자동 파이프라인 시작
  Then 다음 시점들이 측정되어야 함:
    | 단계               | 누적 시간 | SLA   |
    | STT 완료           | +10분    | ✓     |
    | Claude 청킹 완료    | +13분    | ✓     |
    | TTS+HTML (병렬)     | +15분    | ✓     |
    | HyperFrames render | +35분    | ✓     |
    | 검수 대기 진입      | +35분    | ✓     |
    | (학원장 검수)      | +N분     | 학원장 의존 |
    | YouTube 업로드      | +5분     | ✓     |
    | 알림 발송           | +1분     | ✓     |
  
  And 전체 (학원장 검수 시간 제외): < 50분 (p95)
  And 전체 (학원장 검수 평균 15분 가정): < 90분 (p95)
  And p99: < 180분
```

---

## Feature 33: 성공률 KPI

### F33-01: 영상 자동 업로드 성공률 ≥ 90%

```gherkin
Scenario: 파일럿 학원 30편 처리 후 측정
  Given 파일럿 학원 2곳에서 30편 강의를 처리
  
  Then 다음 지표:
    | YouTube publish 성공 | ≥ 27편 (90%) |
    | HyperFrames render 성공 | ≥ 28편 (95%) |
    | STT 성공             | ≥ 29편 (97%) |
    | 자동 publish (auto 모드) 성공 | ≥ 27편 |
  
  And 실패 사례는 모두 lecture.failure_reason 기록 + 학원장 검토
```

---

## Feature 34: RAG 인덱싱 신뢰도

### F34-01: 인덱싱 누락 0%

```gherkin
Scenario: publish 된 영상의 100% 인덱싱
  Given lecture 의 status='published' 인 row 가 N개
  
  When 1시간 후 검증 쿼리:
    """
    SELECT COUNT(*) FROM lecture_chunk lc
    JOIN lecture l ON l.lecture_pk = lc.lecture_pk
    WHERE l.status='published'
    AND (lc.rag_indexed_at IS NULL OR lc.qdrant_point_ids IS NULL)
    """
  
  Then 결과 = 0 (인덱싱 누락 없음)
  And 만약 > 0 일 경우 자동 재인덱싱 + 시스템 admin 알림
```

---

## Feature 35: 권한 변경 즉시성

### F35-01: Sensitive write DB 재검증 latency

```gherkin
Scenario: Trust grant 회수 직후 차단 latency
  Given 박민준이 auto_publish_own grant 보유
  And 박민준의 JWT 가 발급된 지 30분 경과 (TTL 1h 내)
  
  When 김지영이 박민준의 grant 회수 (DB UPDATE 즉시)
  And 1초 후 박민준이 영상 publish 시도
  
  Then DB 재검증 latency 측정:
    | DB membership/trust fetch | < 5ms |
    | CASL ability rebuild       | < 1ms |
    | 총 권한 검증 overhead       | < 10ms (p95) |
  
  And 박민준의 publish 시도가 차단됨 (1초 이내 응답)
  And 권한 변경 → 차단까지 latency: < 5초 (수동 회수 → 첫 차단)
```

---

# Part H. Gap Coverage (post-review additions)

> 본 Part 는 1차 작성 후 사용자 review 에서 발견된 갭을 보완한다.
> Feature F36~F47 은 인증 / 권한 / 사용 흐름 / lifecycle 의 edge case 와 현실 시나리오를 추가 커버.

---

## Feature 36: Auth — 비밀번호 재설정 · 프로필 변경 · 다중 디바이스

### F36-01: 비밀번호 재설정 (Firebase 표준)

```gherkin
Background:
  Given 박민준이 학원 시스템 가입 완료 후 비밀번호를 잊었다

Scenario: 비밀번호 reset 흐름
  When 박민준이 로그인 화면에서 "비밀번호 찾기" 클릭
  And 본인 이메일 (park.minjun@example.com) 입력
  Then Firebase Auth 가 reset 링크가 포함된 이메일 발송
  And 이메일에는 1시간 유효한 비밀번호 재설정 링크 포함
  
  When 박민준이 링크 클릭 + 새 비밀번호 (강도 조건 만족) 입력
  Then Firebase Auth 비밀번호 업데이트
  And 박민준의 기존 ID token 들이 모두 invalidate
  And 모든 device 에서 강제 logout
  And 박민준에게 이메일 알림: "비밀번호가 변경되었습니다. 본인이 아니라면 즉시 학원장 또는 시스템 admin 에게 알리세요."
```

### F36-02: 프로필 변경 (display_name, email)

```gherkin
Background:
  Given 박민준이 로그인 상태

Scenario: display_name 변경
  When 박민준이 "프로필 설정 → 이름 변경" → "박민준A" 로 변경
  Then user_profile.display_name='박민준A' UPDATE (identity_user 는 변경 없음)
  And 강사 목록 UI 등에 즉시 반영 (JOIN user_profile 재조회)
  
Scenario: 이메일 변경
  When 박민준이 "이메일 변경" → 새 이메일 입력
  Then 새 이메일로 Firebase 인증 메일 발송
  And 박민준이 새 이메일의 인증 링크 클릭
  Then identity_user.email='new@example.com' UPDATE
  And Firebase Auth 의 email 도 동기 UPDATE
  And 모든 device 재로그인 강제 (보안)
  But 학원장 김지영에게 알림: "박민준 강사가 이메일을 변경했습니다 (변경 전: old@... → 변경 후: new@...)"
```

### F36-03: 동시 다중 디바이스 로그인

```gherkin
Scenario: 모바일 + 데스크탑 동시 로그인
  Given 김지영이 데스크탑에서 로그인 상태
  
  When 김지영이 모바일 브라우저에서도 동일 계정으로 로그인
  Then 양쪽 device 모두 active session 유지
  And 데스크탑은 강제 logout 되지 않는다
  And 양쪽에서 정상 API 사용 가능
  
Scenario: 이상한 IP 에서 로그인 시도 감지
  Given 김지영이 한국 IP 에서 평소 로그인
  
  When 1시간 이내에 미국 IP 에서 로그인 발생
  Then 시스템이 비정상 로그인 감지 (Firebase Auth 의 user.metadata 분석)
  And 김지영에게 이메일 + 알림톡 발송: "비정상 로그인 감지. 본인이 아니라면 비밀번호 변경 권장"
  And v0.5+ MFA 활성 시 추가 인증 요구
```

---

## Feature 37: 학원장 Lifecycle — Lockout 방지 · 인수인계 · 공동 운영

### F37-01: 마지막 DIRECTOR 본인 회수 차단

```gherkin
Background:
  Given A학원에 DIRECTOR membership 이 김지영 1명만 존재

Scenario: 본인 권한 회수 시도 (Lockout 방지)
  When 김지영이 "강사 관리 → 김지영 (본인) → DIRECTOR 권한 회수" 시도
  Then 시스템이 검증: A학원의 active DIRECTOR membership = 1 (김지영 본인) 확인
  And HTTP 400 Bad Request: "마지막 DIRECTOR 의 권한은 회수할 수 없습니다. 다른 사용자에게 DIRECTOR 권한 부여 후 재시도."
  And membership 변경되지 않는다
  And audit_log 에 시도가 기록된다 (action='attempted_lockout')
```

### F37-02: 학원장 인수인계 (DIRECTOR 권한 이전)

```gherkin
Background:
  Given A학원에 김지영이 단독 DIRECTOR
  And 새 학원장 김후임 (kim.successor@example.com) 이 인수 예정

Scenario: 인수인계 흐름
  # 1단계: 새 학원장에게 DIRECTOR 권한 부여
  When 김지영이 김후임을 강사 (TEACHER) 로 초대 (F3-01 흐름)
  And 김후임이 첫 로그인 완료
  
  When 김지영이 "강사 관리 → 김후임 → role 변경 → DIRECTOR" 클릭
  And 확인 다이얼로그: "학원장 권한을 부여하면 김후임이 본 학원의 모든 admin 권한을 가집니다. 계속?" → "예"
  
  Then membership 의 김후임 row 가 role='DIRECTOR' 로 UPDATE
  And audit_log 에 (actor=김지영, target=김후임, action='grant', role='DIRECTOR') 기록
  And 김후임에게 알림: "DIRECTOR 권한이 부여되었습니다"
  And 김지영의 화면에 "권한 위임 완료. 본인 권한을 회수하려면 별도 진행" 안내
  
  # 2단계: 기존 학원장 본인 회수 (이제 lockout 안 됨)
  When 김지영이 본인 DIRECTOR 권한 회수
  Then F37-01 검증 통과 (김후임이 DIRECTOR 보유)
  And 김지영의 membership 회수
  And A학원의 active DIRECTOR = 김후임 1명
```

### F37-03: 공동 학원장 (N DIRECTOR)

```gherkin
Background:
  Given A학원에 김지영이 DIRECTOR

Scenario: 공동 학원장 추가
  When 김지영이 운영 파트너 박운영 (TEACHER 였음) 을 DIRECTOR 로 승격
  Then A학원의 DIRECTOR membership 2명 (김지영 + 박운영) 가 동시 존재
  And 양쪽 모두 동등 권한 (CASL ability builder 가 멀티 DIRECTOR 처리)
  And 한쪽이 변경한 학원 설정이 다른 쪽에도 즉시 보임
  
Scenario: 한 DIRECTOR 회수 시 lockout 검증
  Given A학원에 DIRECTOR = 김지영 + 박운영 2명
  
  When 박운영이 김지영의 권한 회수 시도
  Then F37-01 검증 통과 (박운영 본인이 DIRECTOR 남음)
  And 김지영의 membership 회수 정상 진행
```

### F37-04: 본인 탈퇴 시도

```gherkin
Background:
  Given 김지영이 A학원의 마지막 DIRECTOR

Scenario: 본인 계정 탈퇴 시도 = 학원 폐업 처리
  When 김지영이 "계정 설정 → 탈퇴" 클릭
  Then 시스템이 검증: A학원의 마지막 DIRECTOR 인지 확인
  And 다음 메시지 표시:
    """
    학원장 1명만 있는 학원에서 본인 탈퇴는 학원 폐업으로 간주됩니다.
    
    옵션:
    1) 학원 폐업 진행 (F26 흐름) — 30일 후 데이터 hard delete
    2) 다른 사용자에게 DIRECTOR 권한 인수인계 후 탈퇴 (F37-02)
    3) 취소
    """
  
  When 김지영이 옵션 1 선택
  Then organization.deleted_at=<current> + 30일 후 hard delete 스케줄
  And 강사·학생·학부모에게 폐업 알림
```

---

## Feature 38: YouTube Channel 관리 Edge Cases

### F38-01: 학원장이 운영 직원에게 채널 관리 위임

```gherkin
Background:
  Given 김지영이 A학원 DIRECTOR
  And 한울학원에 운영 직원 "이수영" (TEACHER role) 등록
  But 김지영의 Google 계정이 YouTube 채널 admin 이 아니다
  And 이수영의 Google 계정이 학원 YouTube 채널 admin

Scenario: manage_youtube_channel grant + 위임 OAuth
  When 김지영이 "강사 관리 → 이수영 → 권한" → 'manage_youtube_channel' 체크 → 저장
  Then trust_relationship INSERT (teacher_pk=이수영, scope='manage_youtube_channel')
  And 이수영에게 알림: "YouTube 채널 관리 권한이 부여되었습니다"
  
  When 이수영이 로그인 → "외부 연동 → YouTube 채널 연결" 클릭
  And 본인 Google 계정으로 OAuth 진행 (Google 측에서 채널 admin 권한 보유 검증)
  Then refresh_token 발급 + KMS 암호화 저장
  And youtube_channel 의 INSERT:
    | org_pk             | 1                                  |
    | connected_by_user_pk   | 이수영의 user_pk                   |
    | oauth_refresh_token_kms| <암호화된 토큰>                    |
  And audit_log: actor=이수영, action='connect_youtube'
```

### F38-02: YouTube 채널 변경 (새 채널로 이전)

```gherkin
Background:
  Given A학원이 youtube_channel "한울학원 (UC_a01)" 에 연결 중
  And 새 채널 "한울학원 공식 (UC_a02_new)" 로 이전 결정

Scenario: 채널 disconnect + reconnect
  When 김지영이 "외부 연동 → YouTube 채널 연결 해제" 클릭
  And 경고 표시: "기존 영상들은 YouTube 에 유지되지만, 새 영상은 업로드 안 됩니다. 채널 변경하려면 새 채널 재연결 필요."
  And "해제 확정" 클릭
  
  Then youtube_channel.disconnected_at=<current> 기록
  And refresh_token 은 KMS 에서 즉시 무효화 (revoke)
  And youtube_video 의 status 변경 없음 (기존 영상 보존)
  
  When 김지영이 새 채널로 OAuth 진행
  Then 새 youtube_channel row 가 INSERT (이전 row 는 disconnected 보존)
  And 향후 영상은 새 채널로 업로드
  And 학원장에게 알림: "YouTube 채널 이전 완료"
```

### F38-03: Google 가 YouTube 채널 정지

```gherkin
Background:
  Given A학원의 YouTube 채널이 정상 운영 중
  
Scenario: 채널 정지 감지
  When 시스템이 일 1회 배치로 YouTube Data API channels.list 호출하여 채널 상태 확인
  And API 응답: { status: "channelSuspended" } 또는 401/403
  
  Then youtube_channel.last_error='CHANNEL_SUSPENDED' 기록
  And 김지영에게 즉시 알림톡 + 이메일: "YouTube 채널이 Google 에 의해 정지되었습니다. Google 측 정책 위반 검토 + 채널 복구 또는 새 채널 연결 필요."
  And 진행 중이던 'upload-youtube' job 들이 모두 'channel_suspended' 상태로 일시 정지
  And 학원장이 새 채널로 재연결 (F38-02) 하거나 Google 측에 이의 신청해야 함
```

### F38-04: 위임받은 직원 퇴사 → OAuth 재연결 필요

```gherkin
Background:
  Given 이수영 이 manage_youtube_channel grant 보유 + OAuth 진행 (refresh_token 이수영 권한 기반)
  And 이수영이 학원에서 퇴사 결정

Scenario: 위임받은 직원 퇴사 처리
  When 김지영이 "강사 관리 → 이수영 → membership revoke" + manage_youtube_channel grant 회수
  Then trust_relationship revoke + membership revoke
  And refresh_token 은 KMS 에서 무효화 (revoke 즉시)
  And youtube_channel.last_error='OWNER_LEFT' 기록
  
  Then 학원장 김지영에게 알림: "YouTube 채널 관리자가 변경되어 재연결이 필요합니다. 본인 또는 다른 위임 직원이 OAuth 진행해야 합니다."
  And 채널 publish 가 일시 정지 (refresh_token 무효 상태)
  And 김지영이 다른 직원에게 grant 부여 → 새 OAuth 진행 → 정상 복구
```

---

## Feature 39: Upload 워크플로 정밀화

### F39-01: 강사가 학원 컨텍스트 잘못 선택 시 visual cue

```gherkin
Background:
  Given 최정훈이 A학원 + B학원 양쪽 TEACHER
  And 최정훈의 현재 학원 컨텍스트 = B학원

Scenario: 업로드 화면에서 학원 명시 + 확인
  When 최정훈이 "강의 업로드" 화면 진입
  Then 화면 상단에 학원 컨텍스트가 visually highlighted 표시:
    """
    [업로드 대상 학원] 별의학원 (B학원)   [학원 변경]
    """
  And 강사가 학원을 변경하려면 명시적으로 "학원 변경" 버튼 클릭 필요
  
  When 최정훈이 업로드 시도
  Then 확인 다이얼로그: "별의학원 (B학원) 의 중3 수학반 영상으로 업로드합니다. 맞나요?"
  And 최정훈이 "예" 선택 시만 업로드 진행
  And 업로드 후 학원 변경 불가 (영구 결정 — 잘못 올린 경우 delete + 재업로드)
```

### F39-02: 업로드 후 메타 수정 (검수 대기 전)

```gherkin
Background:
  Given 박민준이 강의 #501 업로드 + STT 진행 중
  And lecture.status='transcribing'

Scenario: 메타 수정 가능
  When 박민준이 영상 #501 의 "강의 정보 수정" 클릭
  And 다음 필드 수정:
    | 학원 / 반    | (수정 불가 — locked)        |
    | 수업 일시    | 18:00 → 18:15 (수업 시작 지연)|
    | 과목 / 학년  | (수정 가능)                  |
    | 제목         | (수정 가능)                  |
  
  Then lecture 의 해당 컬럼 UPDATE
  And lecture.status 진행에 영향 없음 (계속 transcribing)
  
Scenario: publish 후 메타 수정 시 제한
  Given 영상 #501 이 published 상태 (YouTube 업로드 완료)
  
  When 박민준이 메타 수정 시도
  Then 다음 필드만 수정 가능:
    | 제목 / 설명 / 태그 (YouTube 동기 UPDATE) |
    | 수업 일시 / 학원 / 반 (DB 만, YouTube 미반영) |
  And YouTube 동기 필드 변경 시 YouTube Data API videos.update 호출
```

### F39-03: 강사가 본인 강의 삭제 (publish 전)

```gherkin
Background:
  Given 박민준이 영상 #501 업로드 직후 (lecture.status='queued')

Scenario: 강사가 잘못 업로드한 강의 삭제
  When 박민준이 영상 #501 의 "삭제" 클릭
  And 확인: "이 강의를 삭제합니다. 진행 중인 처리도 중단됩니다." → "예"
  
  Then lecture.deleted_at=<current> + status='deleted_by_teacher'
  And BullMQ 의 모든 진행 중 job (transcribe / chunking / TTS / HTML / render) 가 cancel
  And S3 의 source_audio 가 hard delete (저장소 절약)
  And video_asset 도 cascade delete
  And 박민준에게 확인: "삭제 완료"
  
Scenario: publish 후 강사 본인 삭제는 학원장 권한 필요
  Given 영상 #501 이 published 상태
  
  When 박민준이 삭제 시도
  Then CASL ability.can('delete', 'Lecture') = false
  And 메시지: "publish 된 강의는 학원장 (DIRECTOR) 에게 삭제를 요청하세요."
```

---

## Feature 40: Review 워크플로 정밀화

### F40-01: 학원장 부재 시 검수 위임

```gherkin
Background:
  Given 김지영이 A학원 DIRECTOR
  And 김지영이 1주일 휴가 예정 → 운영 직원 이수영 에게 검수 위임

Scenario: approve_videos grant 부여
  When 김지영이 "강사 관리 → 이수영 → 권한" → 'approve_videos' 체크 + effective_to=1주일 후 설정
  Then trust_relationship INSERT (teacher_pk=이수영, scope='approve_videos', effective_to=<7일 후>)
  
  When 이수영이 로그인 → "검수 큐" 진입
  Then 검수 대기 영상 목록 정상 표시
  And 이수영이 영상 #501 승인 → YouTube 업로드 정상 진행
  And youtube_video.approved_by_user_pk=이수영 의 user_pk 기록
  
Scenario: 위임 기간 만료 자동 회수
  When effective_to 시간 도달 (1주일 후)
  Then trust_relationship.status='expired' 자동 UPDATE (배치)
  And 이수영의 다음 검수 시도 시 권한 없음 응답
  And 이수영에게 알림: "검수 권한이 만료되었습니다"
```

### F40-02: 검수 화면 동시 접근 (낙관적 lock)

```gherkin
Background:
  Given A학원에 DIRECTOR = 김지영 + 박운영 2명 (F37-03 공동 학원장)
  And lecture #501 의 status='pending_review'

Scenario: 동시 검수 시도
  When 김지영이 영상 #501 검수 화면 진입 (T+0)
  And 박운영도 거의 동시에 진입 (T+1초)
  
  Then 양쪽 화면이 모두 진입 가능 (read-only 표시 X, view 는 동시)
  
  When 김지영이 먼저 "승인" 클릭 (T+30초)
  Then lecture.status='uploading' UPDATE, lecture.approved_by_user_pk=김지영
  And 박운영의 화면이 자동 갱신 (WebSocket 또는 polling 5초)
  And 박운영의 화면에 "박운영 강사 — 이미 승인됨 (김지영, 30초 전)" 표시
  And 박운영의 "승인" 버튼 비활성
```

### F40-03: 검수 화면 미저장 작업 복구

```gherkin
Background:
  Given 김지영이 영상 #501 검수 화면에서 제목과 설명을 편집 중
  And 변경 사항이 아직 저장 안 됨

Scenario: 브라우저 새로고침 / 타임아웃 후 복구
  When 김지영의 브라우저가 새로고침되거나 token 만료로 재로그인 필요
  Then 시스템이 미저장 변경 사항을 localStorage 에 자동 백업 (5초 주기)
  
  When 김지영이 재진입
  And 영상 #501 검수 화면에 다시 진입
  Then "복원 가능한 미저장 변경 사항이 있습니다. 복원하시겠습니까?" 알림
  And 김지영이 "복원" 선택 시 편집 중이던 제목/설명이 자동 복원
  And 또는 "버리기" 선택 시 깨끗한 상태로 진입
```

---

## Feature 41: 학생 / 학부모 등록 & 매핑

### F41-01: 학원장 또는 강사가 학생 등록

```gherkin
Background:
  Given 박민준이 A학원 강사 (또는 김지영이 DIRECTOR)
  And A학원에 "중3 수학반" class_group 이 존재

Scenario: 학생 등록 (강사 또는 DIRECTOR)
  When 박민준이 "학생 관리 → 학생 등록" 화면에서 다음 입력:
    | 학생 이름           | 김민서                       |
    | 학년                | 중3 (15세)                   |
    | 반                  | 중3 수학반                   |
    | 학부모 이름         | 박엄마                       |
    | 학부모 전화번호      | 010-1234-5678                |
    | 학부모 이메일       | mom.park@example.com         |
    | 자녀 등장 동의       | ✓ (학부모 서명 영수증)        |
    | 만 14세 미만 동의서  | (해당 없음, 15세)            |
  And "등록" 클릭
  
  Then student 테이블 INSERT (v0.5+ 정식 schema; MVP 는 notification 의 parent_phone/email 만 필수)
  And class_group 과 학생 매핑 INSERT
  And consent_video_appearance=true 기록
  And 동의 시점 audit 저장
  And 학원장 (또는 강사 권한 따라) 시점 기록
  
Scenario: 강사 권한으로 학생 등록 시도
  Given 박민준이 TEACHER 권한 (학생 등록 권한이 academy 정책에 따라 다름)
  
  When 박민준이 학생 등록 시도
  Then 시스템 정책 확인:
    A) DIRECTOR only: 메시지 "학생 등록은 학원장이 진행해야 합니다"
    B) TEACHER 도 가능 (학원장이 grant 부여): 등록 진행
  And MVP 기본: DIRECTOR + 본인 반의 강사 (lecture.class_group 매칭) 만 등록 가능
```

### F41-02: 학부모 정보 업데이트

```gherkin
Background:
  Given 학생 김민서 등록 시 학부모 전화 010-1234-5678 기록

Scenario: 전화번호 변경
  When 학원 직원 (DIRECTOR 또는 위임 받은 TEACHER) 이 "학생 관리 → 김민서 → 학부모 정보 수정"
  And 전화번호 010-9876-5432 로 변경
  Then student.parent_phone UPDATE
  And 향후 알림은 새 번호로 발송
  And 변경 audit log INSERT
  And 학부모 본인 알림: "전화번호가 변경되었습니다. 본인이 아니라면 학원에 문의하세요."
```

### F41-03: 학부모 알림 수신 거부 / 옵트아웃

```gherkin
Background:
  Given 박엄마가 알림톡 수신 거부 결정 (개인 선호)

Scenario: 학부모가 알림 거부 요청
  When 박엄마가 알림톡의 "수신 거부" 링크 클릭
  And 학원 시스템에 도착 (POST /unsubscribe/<token>)
  Then 시스템이 student.notification_opt_out=true UPDATE
  And 향후 모든 알림 발송 시 박엄마 자동 제외
  And 학원장에게 알림: "학부모 박엄마 (자녀 김민서) 가 알림 수신을 거부했습니다."
  And student 의 컨텍스트로 학부모 정보 변경 시 재신청 가능

Scenario: 일부 알림만 거부 (영상 알림 거부, 결제 알림은 받음)
  When 박엄마가 알림 설정 페이지 (학부모 PWA, v1.0+) 에서 영상 알림만 OFF
  Then student 의 notification_preferences JSON 에 { videos: false, payments: true } 저장
  And v0.1 MVP 단계에서는 단일 ON/OFF 만, v1.0+ 세분화
```

---

## Feature 42: Data Lifecycle — Orphan, Archive, RAG Integrity

### F42-01: 강사 제거 후 lecture orphan 처리

```gherkin
Background:
  Given 박민준이 A학원에서 강의 10편 publish 후 퇴사 결정
  And 김지영이 박민준의 membership 회수

Scenario: 박민준의 강의는 학원 자산으로 유지
  When membership.status='SUSPENDED' UPDATE
  Then 다음 처리:
    | lecture.teacher_pk = 박민준 의 user_pk (변경 안 함, historical 기록) |
    | lecture.deleted_at = NULL (보존)                                       |
    | youtube_video 도 보존                                                  |
    | RAG (Qdrant + Neo4j) chunks 도 보존 — 학원 자산                       |
  And 학원 UI 의 강의 목록에서 박민준의 강의는 강사 이름이 "박민준 (이전 강사)" 표시
  And 박민준 본인은 더 이상 본 강의에 접근 불가 (membership 없음)
  
Scenario: 박민준이 본인 강의 export 요청 (퇴사 전)
  When 박민준이 "내 강의 export" 클릭 (탈퇴 또는 퇴사 전)
  Then 시스템이 박민준의 강의 메타 + transcript + PDF 강의노트 zip 패키징
  And S3 의 video_mp4 / audio_mp3 는 학원 자산이므로 download 링크 제공 X (또는 학원장 승인 후 가능)
  And 박민준에게 이메일로 다운로드 링크 (60일 유효)
```

### F42-02: 학원 폐업 시 강사 / 학부모 데이터 처리

```gherkin
Background:
  Given A학원이 폐업 결정, deleted_at 30일 후 hard delete 예정

Scenario: 강사 본인의 강의 export 가능
  Given 박민준이 A학원 폐업 알림 수신
  
  When 박민준이 "내 강의 export" 클릭
  Then F42-01 과 동일 흐름
  And 학원 폐업 30일 전까지 본인 강의 export 가능
  And 30일 후 박민준의 강의도 함께 hard delete (학원 데이터에 포함)

Scenario: 학부모 개인정보 anonymize
  When 학원 hard delete 시점 도달
  Then student 의 학부모 phone/email 이 SHA256 해시로 anonymize
  And student row 자체는 audit 목적으로 보존 (3년 후 hard delete)
  And notification 테이블도 parent_phone/email 도 anonymize
```

### F42-03: RAG 격리 위반 E2E 검증

```gherkin
Scenario: 학원 간 데이터 격리 100% 검증
  Given A학원 (org_pk=1) 의 학원장 김지영이 RAG 검색
  And B학원 (org_pk=2) 의 자료가 100건 인덱싱되어 있음
  
  When 김지영이 "Java 강의" 라는 검색어 (B학원에 Java 자료 다수 보유) 로 RAG 검색
  Then Qdrant filter: org_id=1 강제 적용
  And 결과에 B학원 자료는 단 하나도 포함되지 않음
  And RAG retrieval 단계의 필터 검증을 E2E 테스트에 100% 포함
  And 만약 위반 발견 → 즉시 시스템 admin 알림 + 보안 인시던트 처리
  
Scenario: Neo4j 에서도 동일 격리
  Given Neo4j 에 양쪽 학원 데이터 누적
  
  When 김지영이 학원 Concept 그래프 시각화
  Then Cypher 쿼리에 WHERE c.org_id=1 자동 주입
  And 다른 학원 노드가 결과에 포함되지 않음
```

---

## Feature 43: Plan & Billing (v0.5+ Preview)

### F43-01: Plan 변경 시 cap 즉시 적용

```gherkin
Background:
  Given A학원이 Basic plan, org_entitlement.feature_limits={"daily_uploads":6}
  And 결제 시스템 (v0.5+) 도입 후

Scenario: Basic → Pro 업그레이드
  When 김지영이 "학원 설정 → Plan → Pro 로 업그레이드" 결제 완료
  Then 단일 트랜잭션으로 처리:
    | billing_db.payment_ledger INSERT                                              |
    | billing_db.subscription UPDATE (status='ACTIVE')                             |
    | identity_db.org_entitlement UPDATE (product_code='ACADEMY_PRO', feature_limits={"daily_uploads":12}) |
    | identity_db.organization.perm_version bump                                   |
  And 결제일부터 새 한도 즉시 적용 (org_entitlement 캐시 무효화)
  And cap_pending 상태였던 영상이 새 한도 안에 들어가면 자동 진행
  
Scenario: Pro → Basic 다운그레이드
  When 김지영이 Pro 해지 → Basic 으로 변경
  Then 다음 결제일 (월 단위) 부터 org_entitlement 업데이트 예약
  And 즉시 적용 안 됨 (소비자 보호)
  And feature_limits.daily_uploads=6 으로 회복 시점에 cap_pending 발생 가능
```

### F43-02: 결제 실패 시 자동 처리

```gherkin
Scenario: 카드 만료 / 결제 실패
  Given A학원이 Pro plan, 월 결제일 도달
  And 결제 카드가 만료
  
  When 결제 시스템이 결제 실패
  Then org_entitlement.status='GRACE', billing_db.subscription.status='PAST_DUE' 기록
  And 1차 grace period 7일 (org_entitlement.grace_until = 7일 후, Pro 기능 유지)
  And 학원장에게 알림 (이메일 + 알림톡 + 시스템 배너)
  And 7일 후에도 결제 미해결 시 자동 Basic 으로 downgrade
  And 학원장에게 alert: "Plan 이 Basic 으로 변경되었습니다. 영상 한도 6편/일 적용"
```

### F43-03: 학원 휴면 처리 (v0.5+)

```gherkin
Background:
  Given A학원이 6개월 무활동 (영상 publish 0건)

Scenario: 휴면 안내
  When 일 1회 배치가 무활동 학원 감지
  Then 학원장에게 이메일 + 알림톡 발송: "6개월 무활동 — 데이터 보존됩니다만 1년 무활동 시 hard delete 예정"
  
  When 1년 무활동
  Then 학원장에게 최종 경고 (1개월 전)
  And 1개월 후에도 무활동 시 organization.status='SUSPENDED' UPDATE (dormant 처리)
  And 2년 무활동 시 데이터 anonymize / hard delete
```

---

## Feature 44: 외부 Vendor 장애 / 정책 변경

### F44-01: HyperFrames 단가 인상 알림

```gherkin
Background:
  Given HyperFrames API 가 단가 30% 인상 발표
  And 시스템 admin 이 settings 변경 (HYPERFRAMES_PRICE_PER_RENDER)

Scenario: 학원장 알림
  When settings 변경 후 cap 자동 재계산
  Then academy.estimated_monthly_cost 가 갱신
  And 학원장에게 알림: "HyperFrames 단가 인상으로 영상 1편당 운영비가 X% 증가했습니다. Plan 검토 권장."
  And Plan 별 영상 한도가 동일하게 유지 (단지 운영비 부담만 증가)
  And v0.5+ Plan 가격 인상 협의 시 학원장에게 사전 통보
```

### F44-02: NHN 알림톡 비즈채널 정지

```gherkin
Background:
  Given A학원의 NHN 알림톡 비즈채널이 NHN 측 정책 위반으로 정지

Scenario: 알림 채널 fallback 자동 전환
  When 시스템이 NHN 응답 401/403 감지
  Then 모든 후속 알림이 SMS / 이메일 fallback 으로 전환
  And 학원장에게 alert: "NHN 알림톡 채널이 정지되었습니다. NHN 측 정책 검토 + 재신청 필요. 현재 SMS / 이메일 fallback 진행 중"
  And 학원 운영 페이지에 배너 표시
  
  When NHN 채널 복구
  Then 시스템이 다음 알림부터 알림톡 정상 발송
```

### F44-03: AWS S3 / RDS 장애

```gherkin
Scenario: S3 일시 장애
  Given AWS S3 가 5-20분 일시 장애 (us-east-1 등 region 장애 가정)
  
  When 학원장이 강의 업로드 시도
  Then 시스템이 S3 503 응답 감지
  And 클라이언트에 알림: "일시적 시스템 오류. 잠시 후 다시 시도해 주세요."
  And 시스템 로그: "AWS S3 5xx detected — region <region>"
  And 시스템 admin 자동 알림 (PagerDuty / OpsGenie)
  
  When AWS S3 복구
  Then BullMQ 의 보류된 job 자동 재시도
  And 사용자에게 안내: "시스템 복구. 진행 중이던 업로드를 재개합니다."

Scenario: RDS 장애
  When MySQL primary DB 장애
  Then 시스템이 health check fail 감지 + 자동 fallback (RDS multi-AZ)
  And 30초 ~ 2분 내 자동 복구
  And 사용자에게 임시 응답: "잠시만 기다려 주세요" (HTTP 503)
  And 복구 후 in-flight request 자동 재시도
```

---

## Feature 45: Token Lifecycle Resilience

### F45-01: Token 만료 시 자동 refresh + 작업 보존

```gherkin
Background:
  Given 박민준이 강의 검수 화면에서 작업 중
  And 박민준의 ID token 이 만료 5분 전

Scenario: Token 자동 refresh
  When 박민준이 영상 메타 수정 도중 자동 refresh trigger 시점 도달
  Then 클라이언트가 자동 getIdToken(true) 호출
  And 새 token 으로 진행 중인 작업 끊김 없이 계속
  And 박민준에게 별도 알림 없음 (transparent refresh)
  
Scenario: Refresh 실패 (Firebase Auth 측 장애)
  When getIdToken(true) 가 실패
  Then 클라이언트가 미저장 작업을 localStorage 에 자동 백업
  And 사용자에게 알림: "세션이 만료되었습니다. 재로그인 후 작업이 자동 복원됩니다."
  
  When 박민준이 재로그인
  Then localStorage 의 백업 데이터를 자동 복원
  And 박민준에게 "이전 작업이 복원되었습니다. 저장하시겠습니까?" 확인 모달
```

### F45-02: In-flight Request 자동 재시도

```gherkin
Scenario: 요청 도중 token 만료
  Given 박민준의 token 이 요청 도중 만료
  
  When 백엔드가 401 응답 (TOKEN_EXPIRED)
  Then 클라이언트의 axios interceptor 가 401 감지
  And 자동으로 token refresh
  And 실패한 요청을 새 token 으로 자동 재시도
  And 사용자에게 보이는 영향 없음 (latency 약간 증가)
```

---

## Feature 46: Agent 보안

### F46-01: Agent 가 사람 권한 escalation 시도 차단

```gherkin
Background:
  Given Scheduling Agent (type='SERVICE') 가 정상 등록 + AGENT_SCHEDULER role

Scenario: Agent 가 DIRECTOR role 요청 시도
  When Agent process 가 API 호출 with X-Force-Role: DIRECTOR 또는 임의 조작
  Then 시스템이 user.type='SERVICE' 확인
  And ability builder 가 DIRECTOR role 부여 거부 (agent type 의 role 화이트리스트 검증)
  And HTTP 403 + 시스템 admin 알림: "Agent <id> 권한 escalation 시도 감지"
  And audit_log INSERT (action='escalation_attempt')
```

### F46-02: Agent service account 키 노출 / revoke

```gherkin
Background:
  Given 시스템 admin 이 Agent <id> 의 service account 키가 git commit 등으로 유출됨을 감지

Scenario: 즉시 revoke
  When admin 이 "Agent 관리 → <id> → Revoke" 클릭
  Then identity_user.status='SUSPENDED' UPDATE (agent 도 동일 처리)
  And membership.status='SUSPENDED' UPDATE
  And Firebase Custom Token 발급용 service account 키가 KMS 에서 invalidate
  And Agent 의 다음 인증 시도 시 401 Unauthorized
  And audit_log INSERT (action='emergency_revoke')
  And 보류된 agent 작업 일시 정지
  
  When admin 이 새 service account 키 발급 + Agent 재등록
  Then identity_user.status='ACTIVE' 로 복귀 (또는 신규 SERVICE user 생성)
  And Agent 작업 재개
```

### F46-03: 감사 로그 보존 / Anonymize

```gherkin
Scenario: 감사 로그 3년 보존 정책 (파티셔닝)
  Given audit_log 에 다양한 권한 변경 이력 누적 (월별 RANGE 파티션)
  
  When 월 1회 배치 'audit-log-retention' job 실행
  Then 36개월 초과한 파티션을 ALTER TABLE audit_log DROP PARTITION 으로 일괄 삭제
  And 삭제 전 파티션 내 actor_pk / resource_pk 를 SHA256 해시로 anonymize (PIPA 준수)
  And action, result, created_at 은 보존 (집계 통계 목적)
  And usage_log 도 동일하게 36개월 초과 파티션 DROP (비용 추적 보존 기간)
```

---

## Feature 47: YouTube Channel Sanctions / 사후 처리

### F47-01: YouTube 가 영상 사후 takedown

```gherkin
Background:
  Given A학원의 영상 #501 이 publish 후 1주일 경과
  And Google 의 자동 검열로 정책 위반 감지 (음성 copyright 등)

Scenario: 시스템 감지 + 학원장 알림
  When 시스템이 일 1회 배치로 YouTube videos.list 호출 + 상태 확인
  And 영상 #501 의 상태가 "rejected" 또는 "claimed" 로 변경됨 감지
  
  Then youtube_video.status='taken_down' + 사유 기록 (예: copyright_claim)
  And 학원장에게 즉시 알림톡 + 이메일: "영상 #501 이 Google 정책 위반으로 takedown 되었습니다. 이의 신청 또는 콘텐츠 수정 후 재업로드 검토."
  And 학원 대시보드에 takedown 배너 표시
  And 학부모에게 추가 알림 X (이미 발송된 링크는 cleanly 처리)
```

### F47-02: AI 콘텐츠 명시 자동 보강

```gherkin
Background:
  Given YouTube 2024+ 정책: AI 생성 콘텐츠 명시 의무

Scenario: 자동 명시 + 메타데이터 보강
  When 시스템이 영상 publish 시 description 자동 생성
  Then description 마지막에 다음 자동 삽입:
    """
    * 본 영상은 AI 보조 기술로 생성되었습니다.
    * Original audio: 강사 본인 / Visual generation: AI-assisted
    """
  And YouTube videos.insert 시 selfDeclaredMadeForKids 정확 설정
  And 학원 채널의 "AI 콘텐츠" 메타 태그 자동 추가
```

### F47-03: 학원 채널 정책 위반 사전 검열 (LLM-as-judge 강화)

```gherkin
Background:
  Given 자동 검열 (LLM-as-judge) 이 publish 직전 동작 (F28-02 참조)

Scenario: 정책 검열 항목 확대
  When pre-publish-check job 실행
  And 다음 항목 자동 검사:
    | 부적절 발언 (욕설/혐오)              | LLM-as-judge |
    | 학생 개인정보 (이름·전화·주소)        | regex + NER  |
    | 자녀 등장 미동의 학생 음성/이미지     | F25-03 흐름  |
    | 저작권 위험 (다른 강사 강의 영상)    | metadata 검사 |
    | 광고성 내용 (학원 외부 광고)          | LLM-as-judge |
    | 정치적 / 종교적 콘텐츠                 | LLM-as-judge |
  
  When 검출 항목 발견
  Then lecture.status='pending_review' 강제 (auto 모드여도)
  And 검출 항목이 검수 화면에 highlight + 사유 명시
  And 학원장이 검토 후 수정/취소 결정
```

---

## Feature 48: org_entitlement Gate B — 이용권 상태별 접근 제어

> `canXXX()` Gate B: `org_entitlement.status` 가 ACTIVE/GRACE 이 아닐 때 모든 기능을 차단.
> billing_db 를 직접 조회하지 않고 `org_entitlement` 만 읽는다.

### F48-01: ACTIVE — 정상 접근 (기준선)

```gherkin
Background:
  Given A학원의 org_entitlement: service='ACADEMY', status='ACTIVE', source='FREE'
  And 김지영이 DIRECTOR 로 로그인

Scenario: 정상 강의 업로드
  When 김지영이 강의 목록 조회 + 업로드 시도
  Then Gate A: membership(org_pk=1, status='ACTIVE') → PASS
  And Gate B: org_entitlement(status='ACTIVE') → PASS
  And Gate C: ROLE_PERMISSION[ACADEMY][DIRECTOR] → PASS
  And 정상 응답
```

### F48-02: GRACE — 핵심 기능 유지, 결제 유도 배너

```gherkin
Background:
  Given A학원의 org_entitlement.status='GRACE', grace_until='2026-06-07'
  And 결제 실패로 GRACE 진입 (billing_db.subscription.status='PAST_DUE')

Scenario: GRACE 기간 중 강의 업로드
  When 박민준이 강의 업로드 시도
  Then Gate B: org_entitlement(status='GRACE') → PASS (GRACE 는 허용)
  And 강의 업로드 정상 진행
  And 응답 헤더에 X-Entitlement-Warning: "결제 미완료. 2026-06-07 이후 서비스 일시 중단" 포함
  And 프론트 대시보드 상단에 결제 유도 배너 표시

Scenario: grace_until 경과 후 자동 SUSPENDED
  Given grace_until='2026-06-07' 이고 현재 2026-06-08
  When 일 1회 배치 'entitlement-expire' job 실행
  Then org_entitlement.status='SUSPENDED' UPDATE
  And organization.perm_version bump
  And 학원장 김지영에게 알림: "서비스가 일시 중단되었습니다. 결제 수단 업데이트 후 이용하세요."
```

### F48-03: SUSPENDED — 전체 기능 차단

```gherkin
Background:
  Given A학원의 org_entitlement.status='SUSPENDED'

Scenario: SUSPENDED 학원 강의 업로드 시도
  When 박민준이 강의 업로드 API 호출
  Then Gate B: org_entitlement(status='SUSPENDED') → FAIL
  And HTTP 403: {"code": "ENTITLEMENT_SUSPENDED", "message": "서비스가 일시 중단되었습니다."}
  And 강의 업로드 진행 안 됨
  And audit_log INSERT (result='DENY', action='lecture.create')

Scenario: 학원장 결제 완료 → 즉시 복구
  Given A학원이 결제 수단 업데이트 후 결제 성공
  When 결제 시스템이 단일 트랜잭션으로 처리:
    | billing_db.payment_ledger INSERT (type='CHARGE', status='SUCCEEDED') |
    | billing_db.subscription UPDATE (status='ACTIVE')                     |
    | identity_db.org_entitlement UPDATE (status='ACTIVE', grace_until=NULL) |
    | identity_db.organization.perm_version bump                           |
  Then Gate B 즉시 PASS 상태로 복구 (캐시 무효화 포함)
  And 박민준의 다음 API 호출에서 X-Perm-Version 불일치 감지 → snapshot 재요청 → 정상 접근
```

### F48-04: EXPIRED — 구독 만료

```gherkin
Background:
  Given A학원의 org_entitlement.status='EXPIRED', valid_until='2026-05-31'
  And 구독 갱신 없이 만료

Scenario: EXPIRED 후 강의 조회 시도
  When 김지영이 강의 목록 조회
  Then Gate B: org_entitlement(status='EXPIRED') → FAIL
  And HTTP 403: {"code": "ENTITLEMENT_EXPIRED", "message": "이용권이 만료되었습니다. 구독을 갱신하세요."}
  And 기존 강의 데이터는 보존 (삭제 안 됨)
  And 프론트 → 구독 갱신 안내 페이지 리다이렉트
```

### F48-05: FREE source — 무료 학원 정상 접근

```gherkin
Background:
  Given A학원의 org_entitlement.status='ACTIVE', source='FREE'
  And 결제 없이 수동 발급 (파일럿 학원)

Scenario: 무료 학원 강의 업로드
  When 박민준이 강의 업로드 시도
  Then Gate B: org_entitlement(status='ACTIVE', source='FREE') → PASS
  And 정상 접근 (source 값은 Gate 에 영향 없음, 내부 통계용)
```

---

## Feature 49: perm_version 동기화 (프론트 권한 self-healing)

> 권한 변경 발생 시 `perm_version` 을 bump → 프론트가 X-Perm-Version 불일치를 감지 → snapshot 재요청.
> MVP 에서는 403 + version polling 으로 충분. SSE/websocket 는 v1.0+.

### F49-01: 학원장이 강사 권한 변경 → 강사 다음 요청에서 자동 반영

```gherkin
Background:
  Given 박민준이 강사 대시보드 사용 중 (프론트 보유 perm_version=5)
  And 김지영이 박민준에게 'auto_publish_own' grant 부여

Scenario: perm_version bump + 프론트 자동 갱신
  When 김지영이 grant 저장 (trust_relationship INSERT)
  Then identity_db.organization.perm_version = 5 → 6 으로 bump
  And GET /me/permissions 의 X-Perm-Version: 6 반환

  When 박민준이 다음 API 호출 (예: GET /lectures)
  Then 응답 헤더에 X-Perm-Version: 6 포함
  And 박민준 프론트가 보유한 perm_version=5 와 불일치 감지
  And 프론트가 GET /me/permissions 자동 재요청
  And 새 snapshot (perm_version=6, abilities=[...auto_publish_own 포함]) 수신
  And 박민준 UI 에서 자동 publish 버튼 즉시 활성화
  And 사용자에게 보이는 영향 없음 (latency 약간 증가)
```

### F49-02: 403 수신 → 강제 snapshot refresh → 재시도

```gherkin
Background:
  Given 박민준의 프론트 perm_version=7 (stale)
  And DB 실제 perm_version=8 (권한 회수 후 bump)

Scenario: stale 권한으로 403 → 자동 치유
  When 박민준이 영상 publish API 호출
  Then 백엔드가 DB 재검증 → grant 없음 확인 → HTTP 403 반환
  And 프론트의 403 interceptor 가 GET /me/permissions 강제 호출
  And 새 snapshot 수신 (publish 버튼 비활성)
  And 박민준에게 "권한이 변경되었습니다" 토스트 표시
  And 재시도는 자동으로 하지 않음 (권한 없음이 정답이므로)
```

### F49-03: org_entitlement SUSPENDED → 즉시 전원 차단

```gherkin
Background:
  Given A학원 소속 강사 5명이 동시 사용 중
  And 결제 실패로 org_entitlement.status='SUSPENDED' 로 변경

Scenario: SUSPENDED 전환 직후 전원 차단
  When 결제 시스템이 org_entitlement UPDATE + perm_version bump
  Then 강사 5명 각자의 다음 API 호출에서 Gate B 실패 → 403
  And 403 interceptor 가 각자 snapshot 재요청 → SUSPENDED 반영
  And 5명 모두 "서비스가 일시 중단되었습니다" 배너 표시
  And 진행 중이던 파이프라인 job 은 완료 후 더 이상 새 job 미접수
```

### F49-04: perm_version 불일치로 무한루프 방지

```gherkin
Scenario: snapshot 재요청 후에도 perm_version 불일치 지속
  Given 네트워크 오류로 snapshot 재요청이 실패
  
  When 프론트가 snapshot 재요청 → 503 응답
  Then 재시도 최대 3회 (exponential backoff: 1s, 2s, 4s)
  And 3회 실패 시 "네트워크 오류. 새로고침 후 다시 시도하세요" 표시
  And 무한 retry loop 에 빠지지 않음
```

---

## Feature 50: membership_invite 재초대 / 엣지 케이스

> F3-03 의 만료 케이스 이후 — 학원장이 재초대하거나, 이미 가입된 이메일을 초대하는 경우.

### F50-01: 만료된 초대 → 학원장 재초대

```gherkin
Background:
  Given 박민준의 기존 membership_invite.status='EXPIRED' (F3-03 이후 상태)

Scenario: 학원장이 재초대 발송
  When 김지영이 "강사 관리 → 강사 초대" 에서 park.minjun@example.com 다시 입력
  And "초대 보내기" 클릭

  Then 기존 EXPIRED invite row 는 그대로 유지
  And 새 membership_invite row INSERT:
    | email      | park.minjun@example.com |
    | status     | PENDING                 |
    | expires_at | 지금으로부터 24시간 후  |
    | token      | <새 토큰 (기존과 다름)> |
  And 새 초대 이메일 발송
  And audit_log INSERT (action='reinvite')
```

### F50-02: 이미 가입된 이메일 초대 시도

```gherkin
Background:
  Given 박민준이 이미 A학원 TEACHER 로 membership 보유 (status='ACTIVE')

Scenario: 중복 초대 차단
  When 김지영이 park.minjun@example.com 을 다시 초대 시도
  Then 시스템이 membership(org_pk=1, status='ACTIVE') 존재 확인
  And HTTP 409 Conflict: {"code": "ALREADY_MEMBER", "message": "이미 소속된 강사입니다."}
  And membership_invite row 생성 안 됨
  And 이메일 발송 안 됨
```

### F50-03: 다른 학원에 가입된 이메일 초대

```gherkin
Background:
  Given 최정훈이 B학원의 TEACHER (B학원 membership 보유)
  And 최정훈이 A학원에는 아직 소속 없음

Scenario: 타학원 강사를 A학원에 초대
  When 김지영이 choi.junghoon@example.com 초대
  Then 시스템이 A학원의 membership 확인 → 없음 → 초대 진행
  And membership_invite INSERT (org_pk=1, email=최정훈)
  And 초대 이메일 발송

  When 최정훈이 초대 링크 클릭 + 가입 완료
  Then identity_user 새 INSERT 없음 (기존 user_pk 재사용)
  And membership INSERT (org_pk=1, role='TEACHER', status='ACTIVE')
  And membership_invite.status='ACCEPTED'
  And 최정훈은 이제 A학원 + B학원 모두 접근 가능 (F4-01 흐름)
```

### F50-04: REVOKED 강사 재초대

```gherkin
Background:
  Given 박민준이 A학원에서 이전에 제거됨 (membership.status='SUSPENDED')

Scenario: 동일 강사 재영입
  When 김지영이 park.minjun@example.com 다시 초대
  Then 시스템이 SUSPENDED membership 발견
  And 기존 SUSPENDED membership.status='ACTIVE' 로 UPDATE (재활성화)
  And membership_invite 는 불필요 → 생성 안 됨
  And 박민준에게 알림: "A학원에서 강사 권한이 복구되었습니다"
  And audit_log INSERT (action='membership_reactivate')
  And perm_version bump → 박민준 다음 API 호출에서 자동 반영
```

---

## Feature 51: 프랜차이즈 / 다중 학원 운영

> 독립 학원 (org_relation 없음)과 프랜차이즈 체인 (org_relation HQ_BRANCH)이 같은 DB에 공존.
> v0.1: 스키마 생성 + 독립 학원 복수 운영. v1.0+: 그룹 통합 대시보드 + 정책 상속.
> 상세 결정 근거: [`db-design-decisions.md`](db-design-decisions.md) §11.

### F51-01: 학원장이 독립적인 학원 두 개를 운영한다

```gherkin
Background:
  Given 김지영이 한울학원 (org_pk=1, type='ACADEMY') 의 DIRECTOR 다
  And org_relation 에 org_pk=1 관련 row 없음 (독립 학원)

Scenario: 두 번째 학원 개설
  When 김지영이 두 번째 학원 "한울학원 강남" 가입 절차를 진행한다
  Then 단일 트랜잭션으로 신규 학원 생성:
    | organization  | pk=2, type='ACADEMY', name='한울학원 강남', status='ACTIVE' |
    | academy_config| org_pk=2, youtube_auto_mode='review_required'              |
    | org_entitlement| org_pk=2, status='ACTIVE', source='FREE'                  |
    | membership    | user_pk=김지영, org_pk=2, role='DIRECTOR', status='ACTIVE'  |
  And org_relation 에 row 추가 없음 (독립 운영, 위계 없음)

  When 김지영이 대시보드에서 학원 스위처를 연다
  Then ["한울학원 (org_pk=1)", "한울학원 강남 (org_pk=2)"] 두 항목 표시
  And X-Org-Pk 헤더로 컨텍스트 전환하여 각 학원 독립 접근 가능
  And 두 학원 데이터는 서로 격리됨 (공유 없음)
```

### F51-02: 프랜차이즈 그룹을 개설하고 지점을 연결한다

```gherkin
Background:
  Given 김지영이 "한울학원 그룹" 을 개설하려 한다
  And 기존 독립 학원 "한울학원 강남" (org_pk=2) 이 존재한다

Scenario: 그룹 org 생성 + 지점 연결
  When 김지영이 "그룹 관리 → 새 그룹 생성" → "한울학원 그룹" 입력
  Then organization INSERT (pk=10, type='COMPANY', name='한울학원 그룹')
  And membership INSERT (user_pk=김지영, org_pk=10, role='OWNER')

  When 김지영이 "한울학원 강남" 을 그룹에 추가
  Then org_relation INSERT (parent_org_pk=10, child_org_pk=2, relation_type='HQ_BRANCH')
  And audit_log INSERT (action='org_relation.create', actor=김지영)

  When "한울학원 강북" (org_pk=3) 을 추가로 지점 개설
  Then organization INSERT (pk=3, type='ACADEMY', name='한울학원 강북')
  And org_relation INSERT (parent_org_pk=10, child_org_pk=3, relation_type='HQ_BRANCH')
  And membership INSERT (user_pk=김지영, org_pk=3, role='DIRECTOR')
```

### F51-03: 그룹 오너가 각 지점에 접근한다 (명시적 멤버십)

```gherkin
Background:
  Given 한울학원 그룹 (org_pk=10) 아래 강남점 (org_pk=2), 강북점 (org_pk=3)
  And 김지영의 membership:
    | org_pk=10 | role='OWNER'    |
    | org_pk=2  | role='DIRECTOR' |
    | org_pk=3  | role='DIRECTOR' |

Scenario: 강남점 대시보드 접근
  When 김지영이 스위처에서 "한울학원 강남" 선택 (X-Org-Pk: 2)
  Then Gate A: membership(user_pk=김지영, org_pk=2) 존재 → PASS
  And 강남점 데이터 정상 접근

Scenario: 그룹 오너라도 membership 없는 지점은 접근 불가
  Given 김지영이 org_pk=4 (한울학원 분당) 의 membership 이 없다
  And org_relation(parent=10, child=4) 는 존재한다
  
  When 김지영이 임의로 X-Org-Pk: 4 로 API 호출
  Then Gate A: membership(user_pk=김지영, org_pk=4) 없음 → FAIL
  And HTTP 403: "No access to this organization"
  And org_relation 은 UI 표시용이지 권한 부여 근거가 아님을 시스템이 명확히 처리
```

### F51-04: 지점 강사는 해당 지점만 접근 가능

```gherkin
Background:
  Given 박민준이 한울학원 강남 (org_pk=2) 의 TEACHER
  And 한울학원 강북 (org_pk=3) 의 membership 없음

Scenario: 타지점 접근 시도
  When 박민준이 X-Org-Pk: 3 으로 강북점 강의 목록 요청
  Then Gate A 실패 → HTTP 403
  And org_relation 으로 같은 그룹이어도 강사 접근 범위 확장 안 됨
  And 그룹 내 강사 이동은 김지영(그룹 OWNER)이 membership 명시적 추가로만 가능
```

### F51-05: 순환 참조 방지

```gherkin
Scenario: org_relation 순환 시도
  Given org_relation(parent=1, child=2), org_relation(parent=2, child=3) 존재
  
  When 시스템이 org_relation(parent=3, child=1) INSERT 시도
  Then 앱 레벨에서 getAncestors(org_pk=1) 탐지 → 순환 감지
  And INSERT 차단: "순환 참조가 발생합니다. (3 → 1 → 2 → 3)"
  And HTTP 400: {"code": "CIRCULAR_ORG_RELATION"}

Scenario: 자기 참조 시도
  When org_relation(parent=1, child=1) INSERT 시도
  Then DB CHECK constraint (parent_org_pk != child_org_pk) 위반
  And INSERT 차단
```

---

# Appendix

## A. 시나리오 인덱스 (cross-reference)

| Spec | 관련 Features |
|---|---|
| spec-01-01-foundation | F1, F2, F3, F4, F5, F6, F7, **F36, F37, F38, F41, F45, F46, F50, F51** |
| spec-01-02-rag-infra | F8, F9, F10, **F42** |
| spec-01-03-pipeline-core | F11, F12, F13, F14, F15, F18, F19, **F39, F47** |
| spec-01-04-web-frontend | F11, F16, F17, F22, **F39, F40, F45** |
| spec-01-05-notifications | F20, F21, F23, **F41, F44** |
| Cross-cutting | F24, F25, F26, F27, F28, **F42, F43, F44, F46, F48, F49** |
| Journey (전 spec) | F29, F30, F31 |
| NFR (Phase ship 검증) | F32, F33, F34, F35 |

## B. v3 docs cross-reference

각 Feature 는 다음 v3 문서를 근거로 한다:

- F1~F7 → [`identity-policy.md`](identity-policy.md) §2-4, [`auth-and-policy.md`](auth-and-policy.md)
- F8~F10 → [`rag-strategy.md`](rag-strategy.md), [`data-model.md`](data-model.md) §3-4
- F11~F19 → [`pipeline.md`](pipeline.md), [`features.md`](features.md) §1, [`data-model.md`](data-model.md) §1.5-1.10
- F20~F23 → [`features.md`](features.md), [`pipeline.md`](pipeline.md) §5
- F24~F28 → [`risks.md`](risks.md), [`identity-policy.md`](identity-policy.md), PIPA 처리 방침
- F29~F31 → [`vision.md`](vision.md), [`personas.md`](personas.md)
- F32~F35 → [`mvp-scope.md`](mvp-scope.md) §5 졸업 조건
- **F36~F37** → [`identity-policy.md`](identity-policy.md) §3-6, Firebase Auth 표준
- **F38** → [`identity-policy.md`](identity-policy.md), [`risks.md`](risks.md) (YouTube TOS)
- **F39~F40** → [`auth-and-policy.md`](auth-and-policy.md), [`pipeline.md`](pipeline.md)
- **F41** → [`data-model.md`](data-model.md) §1.10 (notification), v0.5+ student/parent schema
- **F42** → [`data-model.md`](data-model.md), [`identity-policy.md`](identity-policy.md) §5.4, [`risks.md`](risks.md)
- **F43** → [`business-model.md`](business-model.md), [`risks.md`](risks.md) — v0.5+ 결제 도입 시
- **F44** → [`risks.md`](risks.md), 외부 vendor 의존성 매트릭스
- **F45** → [`identity-policy.md`](identity-policy.md) §5.5 (권한 전달 hybrid)
- **F46** → [`identity-policy.md`](identity-policy.md) §2.2 (Agent 정책), [`risks.md`](risks.md)
- **F47** → [`features.md`](features.md) §1.7 (검수/거버넌스), [`risks.md`](risks.md)
- **F48** → [`platform-data-design.md`](platform-data-design.md) §6 (3-gate canXXX), [`db-design-decisions.md`](db-design-decisions.md) §5 (org_entitlement)
- **F49** → [`platform-data-design.md`](platform-data-design.md) §9 (perm_version 동기화), [`identity-policy.md`](identity-policy.md)
- **F50** → [`platform-data-design.md`](platform-data-design.md) §2 (membership_invite), [`auth-and-policy.md`](auth-and-policy.md)
- **F51** → [`platform-data-design.md`](platform-data-design.md) §2 (org_relation), [`db-design-decisions.md`](db-design-decisions.md) §11

## C. 사용자 검증 절차

1. **본 문서 자체 review** — 학원장 페르소나가 모든 시나리오 읽고 합의 (2~3시간)
2. **모호 / 누락 지점 표시** — 합의 안 되는 시나리오는 Q&A 로 정리
3. **(선택) 실 학원장 인터뷰** — 1~2명 학원장과 시나리오 review (1주일)
4. **합의 완료 후 spec 작성 시작** — 본 문서 = spec 의 acceptance criteria

## D. 검증 자동화 (v0.5+)

- **vitest + supertest**: spec-별 통합 테스트가 본 문서의 시나리오를 step-by-step 실행
- **Playwright**: E2E (web frontend → backend → DB) 자동 테스트
- **Cucumber.js (v0.5+)**: 본 문서를 `.feature` 파일로 변환 → Cucumber + Playwright step definitions 로 직접 실행

## E. 변경 이력

| 일자 | 변경 | 작성자 |
|---|---|---|
| 2026-05-23 | 초안 — Feature 35개, ~100 시나리오 | (initial) |
| 2026-05-23 | Part H 추가 — Feature 36~47 (Gap Coverage), 약 35 시나리오 추가 | (review-driven) |

본 문서는 v3 의 일부이며, v4 신규 생성 시 본 문서도 v4 로 동기 갱신한다.
