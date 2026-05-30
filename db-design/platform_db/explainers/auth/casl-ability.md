---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p1
  - security
  - authz
  - casl
  - rbac
  - abac
  - rebac
aliases:
  - CASL ability
  - ability 빌드
  - Gate C 정책
  - can 평가
---

# CASL ability 설명 — RBAC·ABAC·ReBAC을 하나의 can()으로 합치기 (Gate C 내부)

> **대상**: 인가(authorization) 로직을 처음 다루는 개발자 (공부용 · 사수 모드)
> **연관 문서**: [[schema-reference|schema-reference.md §E.2 Gate C]] · [[architecture|architecture.md §1.2]] · [[role-as-code]]

3-gate의 마지막 관문 **Gate C(정책)**는 "이 사람이 *이 행동*을 *이 리소스*에 해도 되나?"를 판단합니다. 그런데 그 판단은 한 종류가 아닙니다 — 역할(RBAC)도 보고, 위임(ReBAC)도 보고, 소유권(ABAC)도 봅니다. 이 세 가지를 *매번 if문으로* 흩어 쓰면 코드가 누더기가 됩니다. **CASL**은 이 셋을 하나의 `ability` 객체로 합쳐, `ability.can(action, resource)` 한 줄로 묻게 해주는 라이브러리입니다. 이 문서는 그 ability가 무엇으로 조립되고 왜 매 요청 새로 만드는지 설명합니다.

---

## Q1. CASL이 뭔가요? ability.can()은 무슨 뜻인가요?

**CASL**은 자바스크립트 권한(authorization) 라이브러리입니다. 핵심은 **ability** — "이 주체가 무엇을 할 수 있는지"를 담은 규칙 묶음 객체입니다. 그걸 이렇게 묻습니다:

```typescript
ability.can('publish', lecture)   // 이 강의를 게시할 수 있나? → true / false
ability.can('read', student)      // 이 학생 정보를 볼 수 있나?
```

`can(action, resource)` — **동사(action)**와 **목적어(resource)**를 주면 boolean을 돌려줍니다. 권한 판단이 흩어진 if문이 아니라 *한 곳에 모인 규칙*에 질문하는 형태가 됩니다.

비유하자면 **출입 권한이 적힌 사원증 한 장**입니다. 문마다 "이 사람 들어와도 돼?"를 각 경비원이 따로 판단하는 게 아니라, 사원증(ability)에 적힌 규칙을 리더기(`can`)에 대보는 것이죠. 규칙은 사원증을 *발급할 때* 한 번 모아 적습니다(= ability 빌드).

> 💡 **한 줄 요약**: CASL의 `ability`는 "이 사람이 뭘 할 수 있는지" 규칙을 모은 객체이고, `ability.can(action, resource)`로 동사·목적어를 물어 허용 여부를 받습니다.

---

## Q2. ability는 무엇으로 조립되나요? (RBAC + ReBAC + ABAC)

이게 핵심입니다. Gate C의 ability는 **세 가지 입력을 합쳐** 만들어집니다(§E.2):

```typescript
// schema-reference §E.2 — Gate C: 세 축을 하나의 ability로 빌드
const ability = buildAbility(
  serviceRole,       // RBAC : service_membership.role_code → ROLE_PERMISSION (코드 상수)
  delegationGrants,  // ReBAC: delegation_grant.capability (위임받은 권한)
  entitlement,       // ABAC : 소유권·feature_limits·구독 상태 등 속성
);
if (!ability.can(action, resource)) throw new ForbiddenException();
```

세 축을 한 줄로 비교하면:

| 축 | 질문 | 우리 출처 |
|---|---|---|
| **RBAC** (역할 기반) | "네 *역할*이 이걸 할 수 있나?" | `service_membership.role_code` → `ROLE_PERMISSION` 코드 상수 |
| **ReBAC** (관계 기반) | "누가 너에게 이걸 *위임*했나?" | `delegation_grant.capability` |
| **ABAC** (속성 기반) | "이 *리소스/상황 속성*이 맞나?" | 소유권(`owner_pk==principal`)·`feature_limits`·구독 상태 |

세 축은 **OR로 더해집니다** — 역할로 되든, 위임으로 되든, 소유자라서 되든 하나라도 허용되면 `can`이 true입니다. 단, ABAC의 일부(테넌트 경계·소유권)는 *제약*으로 작동해 false를 강제하기도 합니다(예: 내 강의가 아니면 수정 불가).

```
TEACHER 역할(RBAC) → '강의 게시' 가능
   +  위임받음(ReBAC) → '회원 관리'도 가능
   +  내가 만든 강의(ABAC 소유권) → 그 강의 '수정' 가능
   ─────────────────────────────────────
   = 이 사람의 ability (세 입력의 합)
```

> 💡 **한 줄 요약**: ability는 RBAC(역할→코드 상수 권한), ReBAC(위임받은 capability), ABAC(소유권·속성)를 합쳐 빌드됩니다. 세 축 중 하나라도 허용하면 통과하되, 테넌트·소유권 제약은 막는 쪽으로 작동합니다.

---

## Q3. RBAC 부분 — 역할→권한 매핑이 왜 코드 상수인가요?

RBAC 입력은 `service_membership.role_code`(예: `'TEACHER'`)에서 출발해, **코드 상수** `ROLE_PERMISSION[service][role]`로 그 역할이 가진 동사 목록을 얻습니다([[role-as-code]]).

```typescript
// 코드 상수 — DB에 저장하지 않음 (RBAC-3)
const ROLE_PERMISSION = {
  ACADEMY: {
    DIRECTOR: ['lecture:publish', 'member:manage', 'billing:view', ...],
    TEACHER:  ['lecture:publish', 'lecture:read'],
    STUDENT:  ['lecture:read'],
  },
} as const;
```

왜 DB가 아니라 코드에 둘까요? 역할→권한 매핑은 *제품 로직*이지 *데이터*가 아니기 때문입니다([[role-capability]]). 코드에 두면 (1) 버전 관리·코드 리뷰·테스트가 되고, (2) 새 서비스 역할 추가가 **DB 마이그레이션 없이 코드 배포**로 됩니다(EXT-3). DB에 권한을 저장하면 런타임에 누가 바꿔치기할 위험과, 환경마다 권한이 달라지는 혼란이 생깁니다.

> 💡 **한 줄 요약**: RBAC 부분은 `role_code`로 코드 상수 `ROLE_PERMISSION`을 조회해 동사 목록을 얻습니다. 권한 매핑은 데이터가 아니라 제품 로직이라 코드에 두고(RBAC-3), 역할 추가는 코드 배포로 합니다.

---

## Q4. ABAC 부분 — 소유권·feature_limits는 ability에서 어떻게 쓰이나요?

ABAC(속성 기반)은 *리소스나 상황의 속성*으로 판단합니다. ability 빌드에서 두 가지로 나타납니다.

**① 소유권(ownership) — `owner_pk == principal` (ABAC-1)**

같은 TEACHER라도 "내가 만든 강의"만 수정할 수 있어야 합니다. CASL은 조건부 규칙으로 이를 표현합니다:

```typescript
// 내 강의만 수정 가능 (소유권 속성)
can('update', 'lecture', { teacherPk: principal.userPk });
// → ability.can('update', someLecture) 는 someLecture.teacherPk === 내 pk 일 때만 true
```

이건 [[bola-object-authz|BOLA 방어]]와 한 묶음입니다 — 역할이 있어도 *그 객체가 내 것/내 테넌트 것*이어야 통과합니다.

**② feature_limits — 한도 평가 (ABAC-3)**

"하루 업로드 6개"처럼 구독에 딸린 한도는 `org_entitlement.feature_limits`에서 옵니다([[feature-limits]]). 이것도 ability의 입력 속성입니다 — 한도를 넘으면 `can`이 막습니다(단, 실시간 카운팅 자체는 서비스측, 평가 기준만 entitlement).

> 💡 **한 줄 요약**: ABAC는 소유권(`owner_pk==principal`으로 "내 것만")과 feature_limits(구독 한도)를 ability의 조건으로 넣습니다. 역할이 있어도 객체가 내 것이 아니거나 한도를 넘으면 막힙니다.

---

## Q5. ReBAC 부분 — 위임은 ability에 어떻게 합쳐지나요?

ReBAC(관계 기반)은 `delegation_grant`에서 옵니다 — "학원장이 강사에게 회원관리를 위임했다"는 *관계*가 권한이 됩니다([[rebac-delegation]]).

```typescript
// 위임받은 capability를 ability에 더함 (ACTIVE·미만료만)
for (const grant of delegationGrants) {           // status='ACTIVE' AND expires_at > now
  can(capabilityToAction(grant.capability));      // 예: 'ACADEMY.MANAGE_MEMBERS' → 'member:manage'
}
```

역할(RBAC)엔 없던 권한이 위임(ReBAC)으로 *추가*됩니다. 그래서 같은 TEACHER라도 위임받은 사람은 회원 관리를 할 수 있고, 위임이 회수되면([[perm-version-propagation|@VerifyOnDb]]로 즉시) 다시 막힙니다.

> 💡 **한 줄 요약**: ReBAC는 `delegation_grant`의 ACTIVE·미만료 capability를 ability에 더합니다. 역할에 없던 권한이 위임 관계로 추가되고, 회수되면 즉시 빠집니다.

---

## Q6. ability(can() 결과)를 캐싱하면 안 되나요? 왜 매 요청 새로 만드나요?

**최종 `can()` 결과는 절대 캐싱하지 않습니다(ABAC-7).** ability는 *매 요청* 새로 빌드합니다. 입력 블록(membership·grants·entitlement)은 짧게(60초) 캐싱하되, "할 수 있다=true"라는 *결론*은 캐싱하지 않습니다.

왜? 권한은 *맥락 의존적*이라 캐싱하면 위험합니다.

```
❌ can() 결과 캐싱 시:
   "박씨는 회원관리 가능 = true" 가 캐시에 박힘
   → 위임 회수 후에도 캐시가 true 반환 → 회수가 안 먹힘 (보안 사고)

✅ 매 요청 ability 빌드:
   입력(grants)이 바뀌면 다음 빌드가 자동으로 새 결론을 냄
   → 회수가 즉시 반영
```

ability 빌드는 이미 캐싱된 입력 블록으로 메모리에서 조립하므로 비싸지 않습니다. "결론을 저장"하는 대신 "재료를 빠르게 모아 매번 조립"하는 것 — 이게 [[perm-version-propagation|perm_version]]·[[fail-closed|deny-by-default]]와 맞물려 권한 변경이 즉시 반영되게 합니다.

> 💡 **한 줄 요약**: 최종 can() 결과는 캐싱 금지(ABAC-7). 입력만 짧게 캐싱하고 ability는 매 요청 새로 빌드해, 위임 회수·역할 변경이 다음 요청에 즉시 반영되게 합니다.

---

## 용어 정리

| 용어 | 한 줄 정의 |
|---|---|
| **CASL** | JS 권한 라이브러리. 규칙을 ability로 모아 `can()`으로 질의 |
| **ability** | "이 주체가 뭘 할 수 있는지" 규칙 묶음 객체. 매 요청 빌드 |
| **can(action, resource)** | 동사·목적어를 줘 허용 여부(boolean)를 받는 질의 |
| **RBAC** | Role-Based — 역할로 권한 판단(`role_code`→`ROLE_PERMISSION`) |
| **ABAC** | Attribute-Based — 속성(소유권·한도·상황)으로 판단 |
| **ReBAC** | Relationship-Based — 관계(위임)로 판단(`delegation_grant`) |
| **ownership(소유권)** | `owner_pk==principal` 조건. "내가 만든 것만" |
| **ROLE_PERMISSION** | 역할→동사 매핑 코드 상수(DB 저장 안 함, RBAC-3) |
| **deny-by-default** | 명시적 allow가 없으면 거부 ([[fail-closed]]) |

---

## 테스트 방법
> 🧪 *실제 DB·ORM·운영에서 돌리는 법*: [[testing-strategy]] · [[orm-testing-drizzle]]


ability 테스트의 핵심은 **세 축이 각각 그리고 합쳐서 올바르게 작동하는가**, 그리고 **명시적 allow 없으면 막히는가**입니다. ability는 순수 함수(입력→ability)라 단위 테스트가 쉽고, 경계는 e2e로 봅니다.

### 단위 테스트 (vitest) — ability 빌드 자체

```typescript
import { describe, it, expect } from "vitest";
import { buildAbility } from "../src/authz/ability";

describe("buildAbility — RBAC + ReBAC + ABAC 합성", () => {
  it("RBAC: TEACHER는 강의 게시 가능, 회원 관리는 불가", () => {
    const ability = buildAbility({ service: "ACADEMY", roleCode: "TEACHER" }, [], entitlement);
    expect(ability.can("lecture:publish", lecture)).toBe(true);
    expect(ability.can("member:manage", member)).toBe(false);  // 역할엔 없음
  });

  it("ReBAC: 위임받으면 회원 관리가 추가된다", () => {
    const grants = [{ capability: "ACADEMY.MANAGE_MEMBERS", status: "ACTIVE", expiresAt: future }];
    const ability = buildAbility({ service: "ACADEMY", roleCode: "TEACHER" }, grants, entitlement);
    expect(ability.can("member:manage", member)).toBe(true);   // 위임으로 추가됨
  });

  it("ReBAC: 만료된 위임은 ability에 안 들어간다", () => {
    const grants = [{ capability: "ACADEMY.MANAGE_MEMBERS", status: "ACTIVE", expiresAt: past }];
    const ability = buildAbility({ service: "ACADEMY", roleCode: "TEACHER" }, grants, entitlement);
    expect(ability.can("member:manage", member)).toBe(false);  // 만료 → 제외
  });

  it("ABAC 소유권: 내 강의만 수정 가능", () => {
    const ability = buildAbility({ service: "ACADEMY", roleCode: "TEACHER" }, [], entitlement, { userPk: 10 });
    expect(ability.can("update", { teacherPk: 10 })).toBe(true);   // 내 것
    expect(ability.can("update", { teacherPk: 99 })).toBe(false);  // 남의 것 → 막힘
  });

  it("deny-by-default: 정의 안 한 action은 막힌다", () => {
    const ability = buildAbility({ service: "ACADEMY", roleCode: "STUDENT" }, [], entitlement);
    expect(ability.can("lecture:delete", lecture)).toBe(false);  // 명시 allow 없음 → 거부
  });
});
```

### "무엇을 단언하나" 체크리스트

- [ ] **RBAC**: 역할별 허용 동사가 정확 (TEACHER ✅게시 ❌회원관리)
- [ ] **ReBAC**: 위임 부여 시 추가, 만료·회수 시 제외 (status·expires_at 반영)
- [ ] **ABAC 소유권**: `owner_pk==principal`일 때만 통과 (남의 객체 false) — [[bola-object-authz]]와 함께
- [ ] **deny-by-default**: 정의 안 한 action은 false (allow-list)
- [ ] **can() 캐싱 안 함**: 위임 회수 후 다음 빌드가 false (e2e C-04로 검증)
- [ ] 경계(API)에서는 Gate C 거부가 403으로 나옴 ([[e2e-journeys]] D3-01 역할 거부 매트릭스)

> ⚠️ **테스트 함정**: "할 수 있다"(true)만 테스트하면 절반입니다. ability 테스트의 핵심은 *"하면 안 되는 게 false인가"* — 역할에 없는 동사, 만료된 위임, 남의 객체가 전부 막히는지 봐야 합니다.

---

## 마치며

CASL ability는 흩어진 권한 if문을 **하나의 질의 지점**으로 모읍니다. 그 ability는 세 재료로 조립됩니다:

1. **RBAC** — 역할이 가진 동사 (`ROLE_PERMISSION` 코드 상수)
2. **ReBAC** — 위임받은 capability (`delegation_grant`)
3. **ABAC** — 소유권·한도·상황 속성

그리고 **결론은 캐싱하지 않고 매 요청 재조립**해서, 권한 변경이 즉시 반영되게 합니다. 새 기능에 권한을 더할 때는 "이건 역할인가(RBAC), 위임인가(ReBAC), 소유·속성인가(ABAC)?"를 먼저 구분하고, 그에 맞는 자리에 규칙을 넣으세요. 그리고 항상 *거부 케이스*를 함께 테스트하세요 — deny-by-default는 테스트로만 증명됩니다.

---

## 연결된 개념

- [[gate-abc-flow|Gate A/B/C 흐름]] — ability가 평가되는 Gate C의 위치
- [[role-capability|role 2단 분리와 capability]] — RBAC 입력(role_code)의 출처
- [[role-as-code|역할=코드 상수]] — ROLE_PERMISSION을 DB가 아닌 코드에 두는 결정
- [[rebac-delegation|ReBAC와 위임]] — ReBAC 입력(delegation_grant) 상세
- [[bola-object-authz|BOLA 객체 인가]] — ABAC 소유권과 한 묶음인 객체 수준 방어
- [[feature-limits|feature_limits]] — ABAC 한도 평가의 SSOT
- [[perm-version-propagation|perm_version 전파]] — ability를 매 요청 빌드해 변경을 즉시 반영
- [[fail-closed|fail-closed]] — ability.can false → deny-by-default 403
> 소스 문서
- [[schema-reference]] — §E.2 Gate C(buildAbility: RBAC/ReBAC/ABAC), §D.6 delegation_grant
- [[architecture]] — §1.2 3-gate, 최종 can() 캐싱 금지
- [[requirements]] — RBAC-3(코드 상수)·ABAC-1(소유권)·ABAC-3(한도)·ABAC-7(can 캐싱 금지)·REBAC-1(위임)
