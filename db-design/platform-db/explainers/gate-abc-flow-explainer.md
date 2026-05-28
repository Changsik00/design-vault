---
tags:
  - platform-db
  - explainer
  - p0
  - gate
  - auth
aliases:
  - Gate A/B/C
  - 3-gate 인가
  - Gate 흐름
---

# Gate A/B/C 전체 흐름 설명

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[architecture|architecture.md]] §5 권한모델 · [[schema-reference|schema-reference.md]] §E 3-gate 인가 모델

HTTP 요청 하나가 실제로 허용되거나 거부되기까지, `platform_db`는 딱 3개의 체크포인트를 통과시킨다. 이 3개를 Gate A, Gate B, Gate C라고 부른다. 이 문서는 그 3개가 무엇인지, 왜 3개인지, 어떤 순서로 동작하는지를 설명한다.

---

## Q1. Gate A/B/C가 뭔가요? 그냥 미들웨어 아닌가요?

맞습니다. 코드 관점에서는 미들웨어(NestJS의 Guard/Interceptor)지만, 왜 3개로 나뉘는지가 중요합니다.

각 Gate는 서로 다른 질문에 답합니다.

| Gate | 질문 | 한 마디로 |
|------|------|----------|
| **Gate A** | "이 사람이 이 조직의 구성원인가?" | 소속 확인 |
| **Gate B** | "이 조직이 지금 서비스를 쓸 수 있는 상태인가?" | 결제/이용권 확인 |
| **Gate C** | "이 구성원이 이 리소스에 이 행동을 할 수 있나?" | 세부 권한 확인 |

마치 어느 건물에 들어가는 것과 비슷합니다. "이 건물 거주자인가(A)" → "임대료가 밀리지 않았나(B)" → "이 층, 이 방에 접근 권한이 있나(C)". 3개가 모두 YES여야 통과입니다.

왜 굳이 3개로 쪼개냐고요? 각 Gate의 **실패 이유**와 **HTTP 에러 코드**가 다르고, 각각이 다른 테이블을 조회하기 때문입니다. 하나의 거대한 권한 체크 함수로 합치면 "왜 거부됐는지"를 디버깅하기가 매우 어려워집니다.

> 💡 **한 줄 요약**: Gate A/B/C는 "소속 → 이용권 → 세부 정책" 순서로 점점 세밀해지는 3단계 체크포인트입니다.

---

## Q2. HTTP 요청 하나가 들어오면 순서가 어떻게 되나요?

```
클라이언트 요청
  │
  │  Authorization: Bearer <Firebase JWT>
  │  X-Org-Pk: 42
  │
  ▼
┌─────────────────────────────────────────────────────┐
│  FirebaseAuthGuard                                  │
│  · Firebase로 JWT 검증                               │
│  · firebase_uid 추출                                 │
│  · identity_user 조회 → user.pk 획득                 │
│  실패 시: 401 Unauthorized                           │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│  AcademyScopeInterceptor                            │
│  · X-Org-Pk 헤더 → org_pk 바인딩                     │
│  · [Gate A] getActiveMembership(user.pk, org.pk)    │
│  · membership.status === 'ACTIVE' 확인              │
│  실패 시: 403 Forbidden                              │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│  AcademyPolicyGuard                                 │
│  · [Gate B] checkGateB(org.pk, service)             │
│  · entitlement.status IN ('ACTIVE','GRACE') 확인    │
│  · valid_until > NOW() 확인                         │
│  실패 시: 402 Payment Required                       │
│                                                     │
│  · [Gate C] buildAbility(membership, grants, ent.)  │
│  · CASL ability 객체 구성                            │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│  @CheckAbility 데코레이터                            │
│  · ability.can(action, resource) 평가               │
│  실패 시: 403 Forbidden                              │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼ (민감한 쓰기 요청만)
┌─────────────────────────────────────────────────────┐
│  @VerifyOnDb 데코레이터                              │
│  · JWT가 stale할 수 있으므로 DB에서 최신 상태 재확인    │
│  · perm_version 불일치 시 강제 refresh               │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
              비즈니스 로직 실행
                         │
                         ▼
              audit_log INSERT (결과 기록)
```

**핵심 원칙**: 앞 단계가 실패하면 뒤 단계는 실행되지 않습니다. Gate A에서 막히면 Gate B, C는 아예 돌지 않아요. 이게 중요한 이유는 쿼리 비용 때문입니다. 소속도 없는 사람이 들어왔을 때 결제 테이블까지 조회할 필요가 없으니까요.

> 💡 **한 줄 요약**: 요청은 인증(Firebase) → Gate A(소속) → Gate B(이용권) → Gate C(정책) → 비즈니스 로직 순으로 순차 통과합니다.

---

## Q3. Gate A가 체크하는 게 뭔가요? membership이 뭐죠?

Gate A는 딱 하나를 확인합니다. "`이 사람(user_pk)`이 `이 조직(org_pk)`의 `ACTIVE` 멤버인가?"

```typescript
// Gate A 핵심 로직
const membership = await getActiveMembership(userPk, orgPk);
if (!membership || membership.status !== 'ACTIVE') {
  throw new ForbiddenException(); // 403
}
```

`membership` 테이블은 "누가 어느 조직에 소속되어 있는가"를 저장하는 테이블입니다.

```sql
-- 이런 데이터가 들어있어요
-- user 5번이 학원 org 42번에 TEACHER로 소속됨
SELECT * FROM membership WHERE user_pk = 5 AND org_pk = 42;
-- 결과: { user_pk: 5, org_pk: 42, role: 'TEACHER', status: 'ACTIVE' }
```

**왜 `status` 확인이 필요한가요?** membership이 있다고 해서 무조건 ACTIVE가 아닐 수 있습니다. `SUSPENDED` 상태일 수 있어요. 예를 들어 문제가 생긴 강사 계정을 관리자가 일시 정지시켰을 때, DB에서 row를 삭제하는 게 아니라 `status = 'SUSPENDED'`로만 바꿉니다. 이렇게 해야 "누가 언제 정지됐는지" 이력이 남으니까요.

**`org_pk`는 어디서 오나요?** (→ [[multitenancy-rls-explainer|멀티테넌시]] 행 격리의 전제) 클라이언트가 `X-Org-Pk` 헤더로 보내는 값을 그대로 신뢰하는 게 아닙니다. 헤더의 값은 단서로만 쓰고, 실제로 DB에서 해당 조직이 존재하는지, 해당 사용자가 그 조직 멤버인지를 직접 조회합니다. 클라이언트가 헤더 값을 위조해도 본인이 소속되지 않은 조직의 데이터에는 접근할 수 없는 이유가 이것입니다.

> 💡 **한 줄 요약**: Gate A는 `membership` 테이블에서 `(user_pk, org_pk)` 쌍이 ACTIVE 상태인지만 확인합니다.

---

## Q4. [[gate-b-entitlement-explainer|Gate B]]와 Gate A가 다른 점이 뭔가요? 둘 다 "권한 확인"이잖아요?

표면적으로는 둘 다 "막는 것"이지만 막는 이유가 완전히 다릅니다.

| | Gate A | Gate B |
|---|---|---|
| **확인 대상** | "사람"이 조직 구성원인가 | "조직"이 서비스를 구독 중인가 |
| **조회 테이블** | `membership` | `org_entitlement` |
| **실패 이유** | 소속 없음 / 정지됨 | 구독 만료 / 결제 실패 |
| **HTTP 코드** | 403 Forbidden | 402 Payment Required |
| **누가 해결** | 관리자가 멤버 추가 | 요금제 결제/갱신 |

비유를 들면, 헬스장 회원권으로 생각해보세요. Gate A는 "이 사람이 우리 헬스장 회원 명부에 있는가"이고, Gate B는 "이 헬스장의 이번 달 시설 이용 계약이 유효한가"입니다. 사람이 문제냐 vs. 조직이 문제냐가 다르죠.

이렇게 분리하는 이유는 **오류 메시지와 해결책이 다르기 때문입니다**.

```
Gate A 실패 → "접근 권한이 없습니다" (관리자에게 문의)
Gate B 실패 → "구독이 만료되었습니다. 결제를 갱신해주세요" (결제 페이지로 이동)
```

하나의 "권한 거부"로 뭉뚱그리면, 프론트엔드가 올바른 안내를 사용자에게 줄 수가 없습니다.

> 💡 **한 줄 요약**: Gate A는 "사람의 소속", Gate B는 "조직의 이용권"을 확인하며, 실패 HTTP 코드도 403 vs 402로 다릅니다.

---

## Q5. [[role-capability-explainer|Gate C]]에서 [[role-capability-explainer|RBAC/ABAC/ReBAC]]를 다 쓴다는데 각각 뭔가요?

Gate C는 가장 세밀한 단계입니다. 세 가지 권한 판단 방식을 조합해서 씁니다.

### RBAC (Role-Based Access Control) — 역할 기반

"TEACHER 역할이면 영상을 업로드할 수 있다"처럼, **역할(role)에 따라 할 수 있는 행동이 정해지는 방식**입니다.

```typescript
// 코드 상수로 관리 (DB에 저장하지 않음)
const ROLE_PERMISSION = {
  ACADEMY: {
    TEACHER:  ['upload:video', 'view:lecture', 'manage:schedule'],
    STUDENT:  ['view:lecture'],
    DIRECTOR: ['upload:video', 'approve:video', 'manage:members', ...],
  }
};

// 사용 시
const canUpload = ROLE_PERMISSION['ACADEMY'][membership.role].includes('upload:video');
```

### ABAC (Attribute-Based Access Control) — 속성 기반

"이 강의를 올린 강사 본인만 수정할 수 있다"처럼, **리소스의 속성(owner, 테넌트 등)을 비교해서 판단하는 방식**입니다.

```typescript
// 리소스 소유권 확인 (ABAC)
const lecture = await getLecture(lectureId, orgPk);
const isOwner = lecture.teacherPk === userPk; // 내가 올린 강의인가?

// 조직 경계 확인 (ABAC)
const isSameTenant = lecture.orgPk === orgPk; // 같은 조직 리소스인가?
```

ABAC에는 환경(Environment) 속성도 포함됩니다. 예를 들어 API 키에 `allowed_ip_cidr` 필드가 있어서, 허용된 IP 대역에서만 요청을 받는 것도 ABAC입니다.

### ReBAC (Relationship-Based Access Control) — 관계 기반

"김 강사가 박 선생님에게 '영상 승인' 권한을 위임했다"처럼, **두 사람 사이의 관계(delegation)로 권한을 부여하는 방식**입니다.

```sql
-- delegation_grant 테이블: "grantor가 grantee에게 capability를 위임"
SELECT * FROM delegation_grant
WHERE grantee_pk = 99        -- 박 선생님
  AND org_pk = 42
  AND capability = 'APPROVE_VIDEO'
  AND status = 'ACTIVE'
  AND (expires_at IS NULL OR expires_at > NOW());
```

### 세 가지를 합쳐서 판단

```typescript
// Gate C: 세 가지를 조합
const ability = buildAbility(membership, delegationGrants, entitlement);

// RBAC: 역할로 허용된 행동인가?
// ReBAC: 위임받은 capability가 있는가?
// ABAC:  리소스 소유자인가? 같은 테넌트인가?
if (!ability.can('approve', lecture)) {
  throw new ForbiddenException(); // 403
}
```

> 💡 **한 줄 요약**: Gate C는 RBAC(역할), ABAC(소유권/환경), ReBAC(위임 관계) 세 가지를 조합해 "이 특정 리소스에 이 행동을 할 수 있는지"를 판단합니다.

---

## Q6. Gate 3개를 통과해야 API 응답이 오면 너무 느리지 않나요?

좋은 질문입니다. 느릴 수도 있습니다. 그래서 캐싱 전략이 중요합니다.

**캐싱 방식**: Gate 판단의 "입력 재료"만 캐싱합니다.

```
캐싱 대상 (TTL 60초):
  - membership (Gate A 입력)
  - org_entitlement (Gate B 입력)
  - delegation_grants (Gate C 입력)

캐싱 금지:
  - ability.can() 최종 결과값 자체
```

왜 최종 결과를 캐싱하지 않을까요? ABAC 조건 때문입니다. "이 강의의 소유자인가?"는 강의마다 다르기 때문에, "A가 강의 수정 가능"이라는 결과를 캐싱해버리면 리소스가 바뀌어도 이전 결과가 남아있을 수 있습니다.

**perm_version으로 캐시 무효화**: 권한이 변경되면 (예: 멤버 정지, 구독 만료) `perm_version`이 증가합니다. 클라이언트는 `X-Perm-Version` 헤더로 자신이 가진 버전을 보내고, 서버의 버전과 다르면 즉시 캐시를 버리고 새로 조회합니다.

```
사용자의 X-Perm-Version: 5
DB의 org.perm_version:    6  ← 변경됨!
→ 캐시 flush + 재조회
```

**실제 성능**: 세 Gate가 각각 별도 DB 쿼리를 날리는 게 아닙니다. `getPermissionContext(userPk, orgPk)` 한 번의 호출로 Gate A/B/C에 필요한 데이터를 함께 가져오도록 설계되어 있습니다. 각 Gate의 [[index-design-explainer|인덱스]] 설계와 캐싱이 적용되면, 대부분의 요청에서는 실제 DB 쿼리가 발생하지 않습니다.

```
첫 번째 요청: DB 조회 (3개 테이블) → Redis 캐시 저장
두 번째~N번째 요청: Redis 캐시 읽기 (DB 부하 없음)
60초 후 또는 perm_version 불일치: 캐시 갱신
```

> 💡 **한 줄 요약**: 최종 권한 판단 결과가 아닌 "입력 재료"만 TTL 60초로 캐싱하고, perm_version으로 즉시 무효화합니다.

---

## 마치며

Gate A/B/C는 각각 `membership`, `org_entitlement`, `delegation_grant` + 코드 상수 `ROLE_PERMISSION`을 보고 판단합니다. 실제 코드에서 새 API 엔드포인트를 만들 때, "이 엔드포인트는 어느 Gate가 필요한가?"를 먼저 생각하는 습관을 들이면 권한 설계 실수를 크게 줄일 수 있습니다. 모든 엔드포인트는 Gate A와 B를 무조건 통과해야 하고, Gate C는 리소스 종류에 따라 구체적인 ability 체크를 추가합니다.

---

## 연결된 개념

- [[gate-b-entitlement-explainer|Gate B & 엔타이틀먼트]] — Gate B가 무엇을 체크하는지 상세히
- [[role-capability-explainer|role 2단 분리 + capability]] — Gate C의 RBAC/ReBAC/ABAC 구현
- [[multitenancy-rls-explainer|Pool 모델 + RLS]] — org_pk 행 격리가 Gate A의 전제 조건
- [[pk-ulid-strategy-explainer|BIGINT pk + ULID public_id]] — firebase_uid로 user_pk 찾는 흐름
- [[index-design-explainer|인덱스 설계]] — Gate B 핫패스 복합 인덱스 설계
> 소스 문서
- [[architecture]] — §5 권한모델 3-gate 전체 구조
- [[schema-reference]] — E.1-E.4 3-gate 인가 모델 DDL과 구현
