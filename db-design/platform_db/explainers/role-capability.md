---
tags:
  - platform-db
  - explainer
  - p0
  - auth
  - role
  - capability
  - rbac
aliases:
  - role 2단 분리
  - capability
  - RBAC
  - ABAC
  - ReBAC
---

# role 2단 분리와 capability 설명

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[architecture|architecture.md]] §3.1 불변식, §4 D1-D3, §5.3 RBAC/ABAC/ReBAC · [[schema-reference|schema-reference.md]] §D.4 membership, §D.6 delegation_grant

`platform_db`의 role 체계는 **0008 마이그레이션으로 2단 분리가 구현**됐습니다(`membership.platform_role` + `service_membership.role_code`). 이 문서는 *왜* academy 종속 단일 ENUM에서 2단 구조로 바꿨는지, platform_role과 service_role이 무엇인지, capability가 무엇인지를 설명합니다.

---

## Q1. 원래 membership.role에 OWNER, DIRECTOR, TEACHER, MEMBER, STUDENT, PARENT가 있었는데, 이게 왜 문제였나요?

0008 이전 `membership` 테이블의 `role` 컬럼은 이렇게 생겼었습니다.

```sql
CREATE TABLE membership (
  user_pk  BIGINT UNSIGNED NOT NULL,
  org_pk   BIGINT UNSIGNED NOT NULL,
  role     ENUM('OWNER','DIRECTOR','TEACHER','MEMBER','STUDENT','PARENT') NOT NULL,
  status   ENUM('ACTIVE','SUSPENDED') NOT NULL DEFAULT 'ACTIVE',
  PRIMARY KEY (user_pk, org_pk)
);
```

TEACHER, STUDENT, PARENT는 학원(academy) 서비스 전용 개념입니다. 이 테이블은 `platform_db`, 즉 academy/agent/market/store/fitness가 **모두 공유하는 공통 DB**에 있습니다. (org_pk 기반 [[multitenancy-rls|멀티테넌시]] 행 격리가 전제입니다)

**문제 1: 다른 서비스에서 이 role이 무의미하다.**

마켓 서비스에서 "이 사람은 TEACHER인가?"를 체크하는 코드가 있다면, 그게 맞는 체크일까요? 마켓에는 판매자(SELLER), 구매자(BUYER) 개념이 있지 TEACHER가 있으면 이상합니다.

**문제 2: 새 서비스 추가 시 ENUM이 폭증한다.**

피트니스 서비스를 추가하면 TRAINER, TRAINEE가 필요하고, 마켓을 추가하면 SELLER, BUYER가 필요합니다. 이걸 전부 하나의 ENUM에 넣으면:

```sql
-- 이 방향으로 가면 망합니다
role ENUM(
  'OWNER','DIRECTOR','TEACHER','MEMBER','STUDENT','PARENT', -- academy
  'SELLER','BUYER',                                          -- market
  'TRAINER','TRAINEE',                                       -- fitness
  'AGENT_OWNER','AGENT_USER',                                -- agent
  ...
)
```

게다가 ENUM을 변경하려면 `ALTER TABLE MODIFY COLUMN`을 실행해야 하는데, 수백만 건이 쌓인 대형 테이블에서는 이게 **테이블 전체 잠금**을 유발합니다.

**문제 3: "platform-level 권한"과 "서비스 도메인 역할"이 섞여 있다.**

OWNER는 조직의 최고 관리자라는 platform-level 개념입니다. TEACHER는 academy 서비스 안의 도메인 역할입니다. 이 둘이 같은 ENUM 안에 섞여 있으면, "이 사람이 조직의 OWNER인가?"와 "이 사람이 TEACHER인가?"가 같은 컬럼으로 판단되어 개념이 혼재됩니다.

> 💡 **한 줄 요약**: 현재 role ENUM은 platform-level 권한(OWNER)과 academy-전용 도메인 역할(TEACHER/STUDENT)이 뒤섞여 있어, 멀티서비스로 확장할 수록 폭증·충돌합니다.

---

## Q2. platform_role과 service_role로 나누면 어떻게 달라지나요?

**0008로 구현된 구조** — role을 두 계층으로 분리했습니다.

```
0008 이전 (단일 ENUM, academy 종속):
  membership.role = OWNER | DIRECTOR | TEACHER | MEMBER | STUDENT | PARENT

0008 이후 (2단 분리):
  membership.platform_role            = OWNER | MEMBER | SERVICE     ← 테넌트 소속 권위 (platform-level)
  service_membership(service, role_code) = ('ACADEMY','DIRECTOR')    ← 서비스 도메인 역할
                                           ('ACADEMY','TEACHER')
                                           ('MARKET','SELLER') ...
```

**분리 후 각 계층의 역할**:

```
platform_role (membership 테이블에):
  OWNER   - 조직을 만든 사람. 조직 삭제, 결제 관리, 멤버 초대 최고 권한
  MEMBER  - 일반 구성원. 서비스별 역할로 세부 권한 결정
  SERVICE - API 키 사용자(봇/머신). 사람이 아닌 서비스 계정
  ⚠️ 초기 명세의 ADMIN은 미채택 — owner 아닌 사람은 MEMBER, 관리 권한은 서비스 역할/위임으로 표현

service_membership (platform_db 안, (service, role_code) 컬럼):
  ('ACADEMY','DIRECTOR')  - 학원 원장
  ('ACADEMY','TEACHER')   - 강사
  ('ACADEMY','STUDENT')   - 학생
  ('MARKET','SELLER')     - 마켓 판매자
```

이 구조의 장점은 서비스가 추가돼도 `membership` 테이블을 건드리지 않는다는 것입니다. `service_membership`(platform_db 공통)에 `(service, role_code)` 행만 추가하면 되고, role 어휘는 코드 상수 `ROLE_PERMISSION[service]`가 권위라 DB 스키마 변경이 없습니다.

```sql
-- "행복학원에서 이 사용자의 academy 역할은?"
SELECT role_code FROM service_membership
WHERE user_pk = ? AND org_pk = ? AND service = 'ACADEMY';
-- 결과: 'DIRECTOR'

-- "이 사용자가 조직 OWNER인가?" (platform-level)
SELECT platform_role FROM membership
WHERE user_pk = ? AND org_pk = ?;
-- 결과: 'OWNER'
```

> 💡 **한 줄 요약**: platform_role은 "조직에서의 지위"(OWNER/MEMBER/SERVICE), service_role(role_code)은 "서비스 내 도메인 역할"(DIRECTOR/TEACHER/STUDENT)로 관심사를 분리합니다. 0008로 구현 완료.

---

## Q3. role이 코드에서 어떻게 권한 체크에 쓰이나요? DB에 "TEACHER는 영상을 올릴 수 있다" 같은 규칙이 저장되어 있나요?

DB에 저장되어 있지 **않습니다**. 이것은 의도적인 결정([[architecture|D3 — role→action 매핑은 코드 상수]])입니다.

"TEACHER는 어떤 행동을 할 수 있다"는 규칙은 **코드 상수**로 관리합니다.

```typescript
// @aiagent/db-platform 패키지 어딘가에 이런 상수가 있습니다
const ROLE_PERMISSION = {
  ACADEMY: {
    OWNER: [
      'upload:video', 'approve:video', 'view:all_lectures',
      'manage:schedule', 'manage:members', 'view:billing',
      'delete:lecture', 'export:data',
    ],
    DIRECTOR: [
      'upload:video', 'approve:video', 'view:all_lectures',
      'manage:schedule', 'manage:members', 'view:billing',
    ],
    TEACHER: [
      'upload:video', 'view:own_lectures', 'manage:schedule',
    ],
    STUDENT: [
      'view:enrolled_lectures',
    ],
    PARENT: [
      'view:child_progress',
    ],
  },
  MARKET: {
    SELLER: ['create:listing', 'manage:inventory', 'view:orders'],
    BUYER:  ['view:listing', 'create:order'],
  },
};
```

이 상수를 기반으로 [[gate-abc-flow|Gate C]]에서 CASL ability 객체를 만들어 권한을 판단합니다. `entitlement` 파라미터는 [[gate-b-entitlement|Gate B]]가 통과한 뒤 전달되는 값입니다.

```typescript
// Gate C: CASL ability 빌드 (단순화한 예시)
function buildAbility(membership, delegationGrants, entitlement) {
  const { can, build } = new AbilityBuilder(createMongoAbility);

  // RBAC: role에 해당하는 모든 action을 허용
  const permissions = ROLE_PERMISSION[entitlement.service]?.[membership.role] ?? [];
  for (const perm of permissions) {
    const [action, subject] = perm.split(':');
    can(action, subject);
  }

  // ReBAC: delegation_grant로 위임된 capability 추가
  for (const grant of delegationGrants) {
    can(grant.capability, 'all');
  }

  return build();
}
```

**왜 DB에 rule을 저장하지 않나요?** 코드 디버깅이 훨씬 편하기 때문입니다.

DB에 rule이 있으면 "왜 이 사람이 이 버튼을 못 누르나?" 디버깅 시 코드와 DB를 동시에 봐야 합니다. 배포 없이 DB에서 rule을 바꿀 수 있다는 것은 권한이 "언제 어떻게 바뀌었는지" 코드 이력(git log)으로 추적이 안 된다는 뜻이기도 합니다. Stripe, GitHub, Linear 같은 회사들도 RBAC rule을 코드 상수로 관리합니다.

> 💡 **한 줄 요약**: "TEACHER는 무엇을 할 수 있다"는 코드 상수 `ROLE_PERMISSION`에 있고, DB에는 저장하지 않습니다. 이력 추적과 디버깅을 위해서입니다.

---

## Q4. delegation_grant의 capability가 뭔가요? role과 어떻게 다른가요?

**role**은 조직에 소속될 때 자동으로 부여되는 기본 권한 묶음입니다. TEACHER가 되면 자동으로 영상 업로드가 가능해지는 것처럼요.

**capability**는 특정 사람이 특정 다른 사람에게 **1:1로 위임하는 단건 권한**입니다.

```sql
-- delegation_grant: "grantor_pk가 grantee_pk에게 capability를 준다"
CREATE TABLE delegation_grant (
  pk          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  grantor_pk  BIGINT UNSIGNED NOT NULL, -- 권한 주는 사람
  grantee_pk  BIGINT UNSIGNED NOT NULL, -- 권한 받는 사람
  org_pk      BIGINT UNSIGNED NOT NULL,
  capability  VARCHAR(50) NOT NULL,
  status      ENUM('ACTIVE','REVOKED') NOT NULL DEFAULT 'ACTIVE',
  expires_at  TIMESTAMP,
  ...
);
```

실제 사용 예시:

```
시나리오: 김 강사가 박 조교에게 "영상 승인 권한"을 위임

grantor_pk = 5  (김 강사)
grantee_pk = 9  (박 조교)
org_pk     = 42 (행복학원)
capability = 'APPROVE_VIDEO'
expires_at = '2026-07-01'  -- 이번 달까지만
```

박 조교는 role로는 MEMBER이지만, 이 delegation_grant 덕분에 영상 승인 행동(`APPROVE_VIDEO`)을 할 수 있습니다.

**role과 capability의 차이를 표로 정리하면**:

| | role | capability (delegation_grant) |
|---|---|---|
| **부여 방식** | 조직 소속 시 자동 | 사람이 사람에게 1:1 위임 |
| **범위** | 역할 전체 권한 묶음 | 단건 행동 |
| **관리자** | 조직 ADMIN이 멤버 역할 설정 | grantor 본인이 직접 위임 |
| **만료** | 멤버십 유효 기간 | expires_at으로 지정 가능 |
| **취소** | membership.status 변경 | delegation_grant.status = 'REVOKED' |

**한 가지 중요한 원칙**: delegation_grant는 **임퍼소네이션(impersonation)이 아닙니다.** "김 강사인 척 행동한다"가 아니라 "박 조교가 특정 행동만 할 수 있다"는 겁니다. grantee는 자신의 신원으로 행동하고, audit_log에도 grantee의 pk가 actor로 기록됩니다.

> 💡 **한 줄 요약**: capability는 역할 전체가 아닌 특정 행동 하나를 사람에게 직접 위임하는 1:1 관계 기반 권한입니다.

---

## Q5. 현재 capability 6종(PUBLISH_VIDEO, APPROVE_VIDEO 등)이 왜 문제고, 어떻게 바뀔 예정인가요?

현재 `delegation_grant.capability`의 CHECK constraint를 보면:

```sql
CONSTRAINT chk_capability CHECK (capability IN (
  'PUBLISH_VIDEO','APPROVE_VIDEO','VIEW_ALL_LECTURES',
  'MANAGE_SCHEDULE','MANAGE_MEMBERS','VIEW_BILLING'
))
```

6개 모두 academy-only 어휘입니다. 문제는 두 가지입니다.

**문제 1: academy가 아닌 서비스에서 어느 서비스 권한인지 파싱 불가.**

예를 들어 market 서비스가 "상품 리스팅 승인" 위임 기능을 만들고 싶다면 `APPROVE_LISTING` 같은 capability가 필요합니다. 그런데 현재 CHECK constraint에 없으니 쓸 수가 없습니다. 그렇다고 `APPROVE_LISTING`을 추가하면, `APPROVE_VIDEO`와 `APPROVE_LISTING`이 같은 테이블에 섞여 있어서 "이 capability가 어느 서비스 것인지" 알 방법이 없습니다.

**문제 2: CHECK constraint 변경이 잠금을 유발.**

capability 종류를 늘리려면 `ALTER TABLE` DDL을 실행해야 합니다. 이게 대형 테이블에서는 위험합니다.

**목표 구조 (D2)**: `<service>.<action>` 네임스페이스 패턴으로 전환합니다.

```sql
-- 현재 (academy 어휘 고착)
capability VARCHAR(50) CHECK (capability IN ('PUBLISH_VIDEO', 'APPROVE_VIDEO', ...))

-- 목표 (네임스페이스 분리)
capability_code VARCHAR(100)  -- CHECK constraint 없음. 앱 레이어에서 검증
-- 값 예시:
--   'academy.publish_video'
--   'academy.approve_video'
--   'market.approve_listing'
--   'fitness.manage_schedule'
```

코드에서는 각 서비스가 자기 capability 상수를 관리합니다:

```typescript
// academy 서비스 코드
const ACADEMY_CAPABILITIES = {
  PUBLISH_VIDEO:     'academy.publish_video',
  APPROVE_VIDEO:     'academy.approve_video',
  VIEW_ALL_LECTURES: 'academy.view_all_lectures',
  MANAGE_SCHEDULE:   'academy.manage_schedule',
  MANAGE_MEMBERS:    'academy.manage_members',
  VIEW_BILLING:      'academy.view_billing',
} as const;

// market 서비스 코드
const MARKET_CAPABILITIES = {
  APPROVE_LISTING: 'market.approve_listing',
  MANAGE_PRODUCTS: 'market.manage_products',
} as const;
```

이렇게 하면 서비스가 늘어나도 `platform_db` 스키마를 건드리지 않고 새 capability를 추가할 수 있습니다.

> 💡 **한 줄 요약**: 현재 6종 고정 CHECK는 academy-only 어휘라 멀티서비스 확장이 불가능하며, `<service>.<action>` 네임스페이스로 전환하면 스키마 변경 없이 서비스별 capability를 추가할 수 있습니다.

---

## Q6. RBAC / ABAC / ReBAC가 뭔가요? 세 가지 다 쓰는 이유가 있나요?

[[role-capability#Q3. role이 코드에서 어떻게 권한 체크에 쓰이나요? DB에 "TEACHER는 영상을 올릴 수 있다" 같은 규칙이 저장되어 있나요?|Q3]]에서 간략히 나왔지만, 여기서 좀 더 구체적인 시나리오로 설명합니다.

### RBAC (Role-Based Access Control)

"어떤 역할인가"에 따라 권한이 결정됩니다.

```typescript
// "TEACHER는 영상을 업로드할 수 있다"
const canUpload = ROLE_PERMISSION['ACADEMY']['TEACHER'].includes('upload:video');
// → true
```

RBAC 하나만 쓰면 충분할 것 같지만, 현실에는 RBAC가 커버하지 못하는 케이스가 있습니다.

### ABAC (Attribute-Based Access Control)

"리소스의 속성이 어떤가"에 따라 권한이 결정됩니다.

```typescript
// "내가 올린 강의는 내가 수정할 수 있다"
// → TEACHER라는 역할만으로는 판단 불가. 리소스의 owner_pk를 봐야 함
const lecture = await getLecture(lectureId, orgPk);
const isOwner = lecture.teacherPk === userPk; // ABAC: 리소스 속성 확인
```

또한 "이 조직의 데이터인가"도 ABAC입니다.

```typescript
// 테넌트 경계 확인 (항상 org_pk와 함께 조회)
const lecture = await db.query.lecture.findFirst({
  where: and(
    eq(lecture.pk, lectureId),
    eq(lecture.orgPk, orgPk)   // ← ABAC: 테넌트 속성 확인
  )
});
```

API 키의 `allowed_ip_cidr`도 ABAC입니다. "어느 IP에서 온 요청인가"라는 환경 속성으로 접근을 제한합니다.

### ReBAC (Relationship-Based Access Control)

"두 주체 사이에 어떤 관계가 있는가"에 따라 권한이 결정됩니다.

```typescript
// "김 강사가 박 조교에게 'APPROVE_VIDEO' 권한을 위임했는가?"
const grants = await getDelegationGrants(grantee_pk=박조교, org_pk=행복학원);
const hasApproveGrant = grants.some(g =>
  g.capability === 'APPROVE_VIDEO' &&
  g.status === 'ACTIVE' &&
  (g.expiresAt === null || g.expiresAt > now)
);
```

### 왜 세 가지를 다 쓰나요?

각 모델이 서로 다른 유형의 권한 질문을 커버하기 때문입니다.

```
RBAC  → "이 역할의 사람은 일반적으로 무엇을 할 수 있나?"
         → TEACHER는 영상 업로드 가능
ABAC  → "이 특정 리소스에 이 특정 사람이 행동할 수 있나?"
         → 이 강의가 내 강의인가? 이 org의 데이터인가?
ReBAC → "누군가가 나에게 이 특정 권한을 위임했는가?"
         → 김 강사가 나에게 영상 승인 권한을 줬는가?
```

세 가지를 AND로 조합합니다. RBAC에서 허용되어도 ABAC에서 막히면 거부입니다 (다른 사람 강의 수정 시도). ReBAC로 위임받아도 ABAC 테넌트 체크는 여전히 통과해야 합니다.

```typescript
// 실제 판단: 세 가지 중 하나라도 허용되면 통과 (OR 아님 - 실은 OR에 가까운 복합 로직)
// 정확히는: RBAC 또는 ReBAC 중 하나로 "행동 허용"이 되어야 하고,
//           ABAC 조건은 항상 만족되어야 함 (테넌트 경계, 리소스 소유권 등)
ability = buildAbility(membership, delegationGrants, entitlement);
const canDo = ability.can('approve', lecture);
// → RBAC: DIRECTOR 이상이면 가능 OR ReBAC: 위임받았으면 가능
// → ABAC: 같은 org_pk의 리소스여야 가능 (AND 조건)
```

> 💡 **한 줄 요약**: RBAC(역할 기반)로 기본 권한을, ABAC(속성 기반)로 리소스 소유권과 테넌트 경계를, ReBAC(관계 기반)로 위임을 처리하는데, 세 가지 모두 다른 유형의 권한 질문에 답하기 때문에 함께 씁니다.

---

## 마치며

현재 코드베이스에는 `membership.role` ENUM이 academy 어휘를 담고 있고 `delegation_grant.capability`도 6종 CHECK로 고정되어 있습니다. 이 상태가 현재 academy MVP에서는 동작하지만, 새 서비스를 붙일 때는 바뀌어야 합니다. 코드에서 role과 capability를 다룰 때 "이게 platform-level인가, service-level인가"를 의식하는 습관을 지금부터 들여두면, phase-17 마이그레이션 시 훨씬 적은 수정으로 전환을 완료할 수 있습니다.

---

## 연결된 개념

- [[gate-abc-flow|Gate A/B/C 전체 흐름]] — Gate C가 전체 3-gate에서 어디에 위치하는지
- [[gate-b-entitlement|Gate B & 엔타이틀먼트]] — Gate B(이용권) vs Gate C(세부 정책) 차이
- [[multitenancy-rls|Pool 모델 + RLS]] — org_pk 격리와 role 체계의 관계
> 소스 문서
- [[architecture]] — §3.1 불변식, §4 D1/D2/D3 결정, §5.3 RBAC/ABAC/ReBAC 정의
- [[schema-reference]] — D.4 membership DDL, D.6 delegation_grant DDL
