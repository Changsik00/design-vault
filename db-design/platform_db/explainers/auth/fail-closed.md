---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p0
  - fail-closed
  - security
  - authz
aliases:
  - fail-closed
  - 페일클로즈드
  - deny-by-default
  - 거부 우선
---

# fail-closed / deny-by-default 설명 — 의심스러우면 막는다

> **대상**: 보안·인가 로직을 처음 다루는 개발자 (공부용)
> **연관 문서**: [[architecture|architecture.md §2.2]] · [[schema-reference|schema-reference.md §E.6]]

권한 코드를 짜다 보면 누구나 한 번쯤 고민하게 됩니다. "토큰 파싱이 실패했는데, 일단 통과시키고 나중에 막을까? 아니면 지금 바로 막을까?", "DB가 잠깐 응답을 안 하는데, 사용자 불편하니 그냥 보여줄까?" 이 질문에 우리 `platform_db`는 단 하나의 답을 갖고 있습니다. **의심스러우면 막는다.** 이것이 fail-closed이고, deny-by-default입니다. 이 문서는 왜 이 원칙이 보안의 기본값이어야 하는지, 우리 설계에서 정확히 어디에 박혀 있는지 설명합니다.

---

## Q1. fail-closed가 뭔가요? fail-open과 뭐가 다른가요?

**fail-closed**는 "장애·오류·불확실 상황에서 시스템을 *닫힌(차단)* 상태로 떨어뜨린다"는 설계 원칙입니다. 반대말이 **fail-open** — "장애 시 *열린(허용)* 상태로 떨어진다"입니다.

비유가 가장 빠릅니다. 전자식 출입문을 생각해 보세요.

```
정전이 났을 때...

fail-open 문 : 잠금이 풀려서 누구나 들어옴 (화재 대피용 비상문 — 사람 안전 우선)
fail-closed 문: 잠긴 채로 멈춤      (금고실·서버실 문 — 자산 보호 우선)
```

화재 대피문은 fail-open이 맞습니다. 사람이 갇히면 안 되니까요. 하지만 **금고실 문**은 정전이 났다고 열리면 안 됩니다. 권한·인가는 금고실 쪽입니다. "시스템이 헷갈리는 순간"에 데이터를 열어주면, 그 순간이 바로 유출 사고입니다.

코드로 보면 차이가 분명합니다.

```typescript
// ❌ fail-open — 오류가 나면 통과시킴 (위험)
function canAccess(req): boolean {
  try {
    const claims = parseJwt(req.token);
    return checkPermission(claims);
  } catch (e) {
    return true;   // ← 파싱 실패 시 "일단 허용"... 유출의 문
  }
}

// ✅ fail-closed — 오류가 나면 막음 (우리 원칙)
function canAccess(req): boolean {
  try {
    const claims = parseJwt(req.token);
    return checkPermission(claims);
  } catch (e) {
    return false;  // ← 의심스러우면 거부
  }
}
```

`return true`에서 `return false`로 바뀐 한 줄. 이 한 줄이 보안 시스템의 성격을 통째로 바꿉니다.

> 💡 **한 줄 요약**: fail-closed는 오류·장애·불확실 상황에서 "막는 쪽"으로 떨어지는 설계이고, fail-open은 "열어주는 쪽"으로 떨어지는 설계입니다. 인가는 항상 fail-closed여야 합니다.

---

## Q2. deny-by-default는요? allow-list랑 deny-list는 또 뭔가요?

**deny-by-default**는 "기본값은 거부, 명시적으로 허용된 것만 통과"라는 정책입니다. fail-closed가 *장애 시 동작*에 관한 것이라면, deny-by-default는 *평상시 판단 기준*에 관한 것입니다. 둘은 형제 개념입니다.

여기서 **allow-list(허용 목록)** vs **deny-list(차단 목록)** 구분이 등장합니다.

```
deny-list 방식 : "이것들은 막아라. 나머지는 다 통과" → 빠뜨린 게 곧 구멍
allow-list 방식: "이것들만 통과. 나머지는 다 막아라" → 빠뜨려도 안전(막힘)
```

비유하자면 클럽 입구의 두 가지 정책입니다. deny-list는 "블랙리스트에 없으면 입장 OK" — 새로운 위험인물이 명단에 없으면 그냥 들어옵니다. allow-list는 "게스트 명단에 있어야만 입장" — 누가 와도 명단에 없으면 못 들어옵니다. 새로운 공격이 등장해도 allow-list는 자동으로 막습니다. "허용한다고 적어둔 적이 없으니까."

우리 Gate C(정책 게이트)가 정확히 allow-list입니다.

```typescript
// ✅ deny-by-default + allow-list — 명시적 allow만 통과
const ability = buildAbility(serviceRole, delegationGrants, entitlement);
if (!ability.can(action, resource)) {
  throw new ForbiddenException();   // 허용 규칙에 안 걸리면 → 무조건 거부
}
// 여기 도달 = ability에 "할 수 있다"가 명시적으로 있었던 경우뿐
```

`ROLE_PERMISSION[service][role]`에 적힌 동사만 `can`이 됩니다. 새 액션을 추가하고 권한 매핑에 적는 걸 깜빡하면? 그 액션은 **막힙니다.** 버그가 "유출" 쪽이 아니라 "차단" 쪽으로 떨어지는 것 — 이게 deny-by-default의 핵심 안전 속성입니다.

> 💡 **한 줄 요약**: deny-by-default는 "명시적 allow만 통과, 나머지는 전부 거부"입니다. allow-list(허용 목록)를 쓰면 빠뜨린 항목이 자동으로 막혀서, 실수가 유출이 아니라 차단으로 떨어집니다.

---

## Q3. 우리 platform_db는 어디서 fail-closed를 강제하나요?

`schema-reference.md §E.6`에 세 줄로 압축돼 있습니다.

```typescript
// schema-reference.md §E.6 — fail-closed 원칙
// JWT claims 파싱 실패 또는 perm_version 불일치 → 무조건 401
// DB 재검증 중 오류 → 503 (허용 아님)
// 모든 권한 판단: deny-by-default, 명시적 allow만 통과
```

각 줄을 풀어 보겠습니다. 우리 인가는 [[gate-abc-flow|3-gate]] 파이프라인을 거치는데, **각 단계마다 "막는 쪽"으로 떨어지는 분기**가 들어 있습니다.

```
HTTP 요청
  ↓
FirebaseJwtGuard      → JWT 서명 위조·만료      → 401 (TOKEN_INVALID)
  ↓                   → 필수 claim 누락         → 401 (CLAIMS_MALFORMED)
  ↓                   → perm_version 불일치     → 401 (재발급 유도)
GateBGuard            → x-org-pk 없음(다중 멤버십) → 400 (ORG_HEADER_REQUIRED)
  ↓                   → 위조·미소속 org 헤더     → 403 (NOT_A_MEMBER)
AcademyPolicyGuard    → Gate C: ability.can 실패 → 403 (deny-by-default)
  ↓
@VerifyOnDb           → DB 재검증 중 오류        → 503 (UPSTREAM_UNAVAILABLE, 허용 아님)
  ↓
비즈니스 로직
```

핵심은 마지막 `@VerifyOnDb`의 **503**입니다. 민감 쓰기(publish·delete·결제)를 처리하기 전에 DB로 권한을 재검증하는데([[perm-version-propagation|§E.5]]), 이때 DB가 일시적으로 응답하지 않으면 어떻게 할까요?

```typescript
// ✅ fail-closed — 재검증 실패 시 503 (절대 통과 아님)
async function verifyOnDb(userPk, orgPk, action) {
  let fresh;
  try {
    fresh = await getActiveMembership(userPk, orgPk);  // DB 재조회
  } catch (e) {
    // DB가 응답 안 함 → "권한 있는지 확인 불가" → 확인 안 되면 막는다
    throw new ServiceUnavailableException("UPSTREAM_UNAVAILABLE");  // 503
  }
  if (!fresh || fresh.status !== "ACTIVE") throw new ForbiddenException();
  // ...
}
```

"확인할 수 없다"는 "허용한다"가 아닙니다. **확인 안 되면 막는다** — 이것이 503의 의미입니다. 200(허용)으로 새지 않습니다.

> 💡 **한 줄 요약**: 우리는 §E.6에서 파싱 실패·perm_version 불일치 → 401, DB 재검증 오류 → 503, Gate C 미허용 → 403으로, 3-gate 전 구간을 "막는 쪽"으로 떨어뜨립니다. 어디서도 200으로 새지 않습니다.

---

## Q4. 왜 fail-open이 아니라 fail-closed인가요? 사용자 불편하잖아요.

맞습니다. 솔직하게 인정하고 시작하겠습니다. **fail-closed는 사용자를 불편하게 합니다.** DB가 잠깐 흔들리면 멀쩡한 사용자도 503을 받습니다. fail-open이었다면 그 사용자는 아무 일 없이 작업을 마쳤을 겁니다.

그래서 이건 **가용성(availability) vs 보안(security) 트레이드오프**입니다.

```
선택지 A — fail-open (가용성 우선)
  DB 흔들림 → 일단 허용 → 사용자 행복
  단, 그 순간 권한 없는 사람도 통과 → 데이터 유출 가능

선택지 B — fail-closed (보안 우선)  ← 우리 선택
  DB 흔들림 → 503으로 차단 → 사용자 불편 (잠시 후 재시도하면 됨)
  유출은 0
```

왜 B를 고르나요? **불편과 유출의 비대칭** 때문입니다.

- 503으로 막힌 사용자: 몇 초 뒤 다시 누르면 됩니다. 손실은 "잠깐의 짜증"입니다. **복구 가능**합니다.
- 잘못 허용된 한 번의 접근: 다른 학원의 학생 명단, 결제 정보, 강의가 노출됩니다. 한 번 유출된 데이터는 **되돌릴 수 없습니다.** PIPA 신고 의무, 신뢰 붕괴, 법적 책임이 따라옵니다.

복구 가능한 불편과 복구 불가능한 유출을 저울에 올리면 답은 정해져 있습니다. **잠깐 막는 게, 잘못 여는 것보다 항상 낫습니다.** 멀티테넌트 SaaS에서는 더욱 그렇습니다 — fail-open의 유출은 "내 데이터"가 아니라 "남의 학원 데이터"이기 때문입니다([[multitenancy-rls|테넌트 격리]]).

물론 503을 *덜 나게* 만드는 노력은 별개로 합니다. 그게 **graceful degradation**입니다 (Q5에서 설명). fail-closed는 "막는다"는 결정이고, graceful degradation은 "막더라도 곱게 막는다"는 기술입니다. 둘은 모순되지 않습니다.

> 💡 **한 줄 요약**: 503으로 막힌 불편은 재시도로 복구되지만, 잘못 허용된 유출은 되돌릴 수 없습니다. 복구 가능한 불편 < 복구 불가능한 유출 — 그래서 fail-closed입니다.

---

## Q5. 그럼 사용자는 매번 막히기만 하나요? graceful degradation은요?

아닙니다. fail-closed가 "전부 또는 전무(all-or-nothing)"를 뜻하지는 않습니다. 우리 설계는 **막는 강도를 위험 등급에 따라 분리**해서, 안전한 동작은 최대한 살립니다. 이것이 **graceful degradation(우아한 성능 저하)** — "일부가 죽어도 전체가 죽지 않게 단계적으로 떨어뜨리는" 기술입니다.

우리 설계의 두 가지 장치를 보세요.

**① routine read는 DB를 안 친다 (NFR-1)**

일상적인 조회(강의 목록 보기 등)는 JWT claims만으로 권한을 판단합니다. DB를 조회하지 않으므로, **DB가 흔들려도 일상 읽기는 살아 있습니다.**

```typescript
// routine read — JWT claims만으로 판단, DB hit 0 (NFR-1)
//   → billing DB·재검증 경로가 죽어도 영향 없음
// sensitive write만 @VerifyOnDb로 DB 재검증
//   → 여기서만 DB 장애가 503으로 이어짐
```

위험이 낮은 동작(read)은 장애의 영향을 안 받고, 위험이 높은 동작(write·결제)만 fail-closed의 엄격한 차단을 받습니다. 이게 graceful degradation입니다.

**② Gate B의 GRACE 상태**

결제 실패가 곧 즉시 차단은 아닙니다. `org_entitlement.status`에 `GRACE`(유예)가 있어서, 결제가 잠깐 실패해도 유예 기간 동안 서비스가 유지됩니다([[explainers/auth/gate-b-billing-grace|gate-b-billing-grace]]). "결제 실패 → 즉시 차단"이라는 거친 fail-closed 대신, "유예 배너 후 차단"이라는 단계적 저하를 둡니다.

```
ACTIVE  → 정상
GRACE   → "결제 실패 — N일 내 갱신 필요" 배너, 서비스는 유지  ← graceful
EXPIRED → 차단 (402)
```

정리하면, fail-closed는 **위험한 경로**에 적용하고, graceful degradation으로 **안전한 경로는 살려서** 사용자 불편을 최소화합니다.

> 💡 **한 줄 요약**: fail-closed는 "전부 막기"가 아닙니다. routine read는 DB 없이 처리하고(NFR-1), Gate B는 GRACE로 유예하면서, 위험한 동작에만 엄격한 차단을 겁니다. 이게 graceful degradation입니다.

---

## Q6. 실수로 fail-open 코드를 짜면 어떻게 되나요? 막을 방법이 있나요?

가장 위험한 fail-open은 "악의"가 아니라 "선의의 편의"에서 나옵니다. "사용자가 자꾸 막혀서 항의하니까, 일단 통과시키자." 이런 코드가 슬그머니 들어옵니다.

```typescript
// ❌ 가장 흔한 fail-open 사고 — "에러나면 일단 보여주자"
async function getLectures(userPk, orgPk) {
  let entitlement;
  try {
    entitlement = await getEntitlementByService(orgPk, "ACADEMY");
  } catch (e) {
    return db.lectures.findMany({ where: { orgPk } });  // ← 재앙: 검증 우회
  }
  if (!gateB(entitlement)) throw new PaymentRequiredException();
  return db.lectures.findMany({ where: { orgPk } });
}
```

이 코드는 "Gate B 확인이 실패하면 결제 안 한 학원도 강의를 본다"는 fail-open입니다. 어떻게 막나요?

**방어 1: catch에서 통과시키지 않는다 (코드 규약)**

catch 블록은 *예외를 다시 던지거나 거부 상태를 반환*해야 합니다. catch에서 정상 데이터를 `return`하는 패턴은 코드 리뷰에서 반드시 잡아야 할 안티패턴입니다.

**방어 2: 명시적 allow만 통과하는 구조 (deny-by-default)**

`ability.can()`이 통과 조건이고, 그 외 모든 경로(예외 포함)는 자연히 거부로 떨어지도록 짭니다. "허용을 적극적으로 증명해야만 통과"하게 만들면, 빠뜨림은 차단이 됩니다.

**방어 3: D3-08 같은 회귀 테스트 (자동화)**

"비정상 입력 N가지가 전부 200이 아님"을 단언하는 테스트를 CI에 박아두면, 누군가 fail-open을 넣는 순간 빌드가 깨집니다 (아래 [테스트 방법] 참고).

**자가진단 체크리스트:**

```
① 모든 catch 블록이 거부(throw/return false)로 끝나나?     → 예
② can()이 false면 무조건 막히나? (allow-list)              → 예
③ DB 재검증 오류가 200으로 새지 않나? (503으로 막히나)      → 예
④ "비정상 입력은 200이 아니다" 테스트가 CI에 있나?          → 예
```

> 💡 **한 줄 요약**: fail-open은 보통 "편의를 위한 catch 통과"에서 생깁니다. catch는 거부로 끝내고, allow-list로 짜고, "비정상 입력은 200이 아니다"를 회귀 테스트로 고정하면 막힙니다.

---

## 용어 정리

| 용어 | 한 줄 정의 |
|---|---|
| fail-closed | 장애·오류·불확실 상황에서 시스템이 "차단(닫힘)" 쪽으로 떨어지는 설계. 인가의 기본값 |
| fail-open | 장애 시 "허용(열림)" 쪽으로 떨어지는 설계. 비상 대피문엔 맞지만 인가엔 위험 |
| deny-by-default | 평상시 판단 기준이 "기본 거부, 명시적 allow만 통과"인 정책 |
| allow-list | "이것들만 통과, 나머지 전부 거부" 목록. 빠뜨린 항목이 자동으로 막혀 안전 |
| deny-list | "이것들만 차단, 나머지 전부 통과" 목록. 빠뜨린 위험이 곧 구멍이 됨 |
| graceful degradation | 일부가 죽어도 전체가 죽지 않게, 안전한 동작은 살리며 단계적으로 떨어뜨리는 기술 |
| @VerifyOnDb | 민감 쓰기 직전 DB로 권한을 재검증하는 장치. 오류 시 503(허용 아님) — fail-closed의 핵심 지점(§E.5) |

---

## 테스트 방법

fail-closed의 본질은 "**어떤 경우에도 200(허용)으로 새지 않는다**"입니다. 따라서 테스트는 *정상 경로*가 아니라 *비정상 입력의 묶음*을 던지고 전부 막히는지 확인합니다. [[e2e-journeys]] **D3-08**의 `Scenario Outline`을 그대로 코드로 옮긴 형태입니다.

### 통합 테스트 (vitest + supertest) — API 경계 블랙박스

```typescript
import { describe, it, expect, beforeAll } from "vitest";
import request from "supertest";
import { app } from "../src/main";

describe("fail-closed / deny-by-default (D3-08)", () => {
  let validToken: string;     // 정상 토큰
  let staleToken: string;     // perm_version 낡은 토큰

  beforeAll(async () => {
    validToken = await issueToken({ orgPk: "01H...HANUL", role: "TEACHER" });
    staleToken = await issueStaleToken({ orgPk: "01H...HANUL" }); // pv 낡음
  });

  // 핵심: 비정상 입력 8종은 전부 닫히고, 단 하나도 200이 아니다
  it.each([
    // [설명,                         헤더 세팅,                              기대상태, 기대코드]
    ["인증 헤더 없음",               {},                                      401, "UNAUTHENTICATED"],
    ["만료·위조 Bearer",             { auth: "Bearer tampered.jwt.value" },   401, "TOKEN_INVALID"],
    ["필수 claim 누락 토큰",         { auth: `Bearer ${tokenMissingUid}` },   401, "CLAIMS_MALFORMED"],
    ["x-org-pk 없음(다중 멤버십)",   { auth: validToken },                    400, "ORG_HEADER_REQUIRED"],
    ["위조·미소속 x-org-pk",         { auth: validToken, org: "01H...OTHER" }, 403, "NOT_A_MEMBER"],
    ["형식 위반 x-org-pk(비-ULID)",  { auth: validToken, org: "not-a-ulid" }, 400, "INVALID_ORG_ID"],
    ["존재하지 않는 리소스 ID",      { auth: validToken, org: "01H...HANUL", path: "/v1/lectures/01H...GHOST" }, 404, "NOT_FOUND"],
  ])("%s → %i (200으로 새지 않음)", async (_desc, setup, status, code) => {
    const req = request(app).get(setup.path ?? "/v1/lectures");
    if (setup.auth) req.set("Authorization", setup.auth);
    if (setup.org) req.set("x-org-pk", setup.org);

    const res = await req;

    expect(res.status).toBe(status);
    expect(res.body.code).toBe(code);
    expect(res.status).not.toBe(200);   // ← fail-closed의 최종 단언
  });

  // perm_version 불일치 → 401 (재발급 유도), 절대 통과 아님
  it("stale perm_version 토큰 → 401, 200으로 새지 않는다", async () => {
    const res = await request(app)
      .get("/v1/lectures")
      .set("Authorization", staleToken)
      .set("x-org-pk", "01H...HANUL");

    expect(res.status).toBe(401);
    expect(res.status).not.toBe(200);
  });

  // DB 재검증 일시 오류 → 503 (허용 아님). 의존성을 강제로 끊어 재현.
  it("민감 쓰기 중 DB 재검증 오류 → 503, 200 아님 (fail-closed)", async () => {
    await db.forceUnavailable();   // 테스트용: 재검증 경로 DB를 일시 차단
    try {
      const res = await request(app)
        .post("/v1/lectures")
        .set("Authorization", validToken)
        .set("x-org-pk", "01H...HANUL")
        .send({ title: "중2 수학", publish: true });

      expect(res.status).toBe(503);
      expect(res.body.code).toBe("UPSTREAM_UNAVAILABLE");
      expect(res.status).not.toBe(200);   // ← 확인 불가 ≠ 허용
    } finally {
      await db.restore();
    }
  });
});
```

### "무엇을 단언하나" 체크리스트

- [ ] **8종 비정상 입력이 각각 401/400/403/404로 닫힌다** — status와 `body.code`를 함께 단언 (사유까지 정확한가)
- [ ] **단 한 경우도 `status === 200`이 아니다** — `expect(res.status).not.toBe(200)`을 모든 케이스에 건다 (허용 누수 0)
- [ ] **perm_version 불일치 → 401** — stale 토큰이 통과하지 않음
- [ ] **DB 재검증 오류 → 503** — "확인 불가"가 "허용"으로 둔갑하지 않음. 503이지 200이 아님
- [ ] **부분 처리 없음** — 차단된 요청이 DB에 흔적을 남기지 않았는지 후속 GET으로 확인 (예: 계정 미생성을 후속 login 401로 검증)

> ⚠️ **테스트 함정**: 통과(200) 케이스만 테스트하면 fail-open 버그를 절대 못 잡습니다. fail-closed 테스트의 8할은 "막혀야 하는 입력"을 던지는 것입니다. 정상 경로 1개 + 비정상 경로 8개의 비율이 건강한 인가 테스트입니다.

---

## 마치며

fail-closed와 deny-by-default는 결국 하나의 태도입니다: **"확신이 없으면 허용하지 않는다."**

코드를 짤 때마다 스스로 물어보세요.

1. 이 `catch` 블록은 거부로 끝나는가? (정상 데이터를 `return`하고 있지 않은가?)
2. 이 권한 판단은 "명시적 allow만 통과"하는가? (빠뜨리면 막히는가, 새는가?)
3. 의존성(DB·billing)이 죽으면 이 경로는 503으로 막히는가, 200으로 새는가?

세 질문에 전부 "막힌다"라고 답할 수 있으면, 그 코드는 안전합니다. 사용자가 잠깐 불편한 건 재시도로 복구되지만, 한 번 새어 나간 데이터는 돌아오지 않습니다. **잠깐 막는 게 잘못 여는 것보다 낫다** — 이 한 문장이 platform_db 인가의 기본값입니다.

---

## 연결된 개념

- [[gate-abc-flow|Gate A/B/C 흐름]] — 각 게이트가 "막는 쪽"으로 떨어지는 분기들의 전체 그림
- [[perm-version-propagation|perm_version 전파]] — perm_version 불일치 → 401, @VerifyOnDb 503의 메커니즘
- [[bola-object-authz|BOLA 객체 인가]] — 미소속·위조 리소스 접근이 403/404로 닫히는 원리
- [[multitenancy-rls|Pool 모델 멀티테넌시]] — fail-open 시 유출되는 것이 "남의 테넌트 데이터"인 이유
- [[gate-b-entitlement|Gate B 이용권]] — GRACE 유예로 graceful degradation을 구현하는 지점
- [[break-glass|Break-glass]] — fail-closed 원칙에 대한 통제된 예외(긴급 접근)의 처리
> 소스 문서
- [[schema-reference]] — §E.6 fail-closed 원칙(401/503/deny-by-default), §E.5 @VerifyOnDb
- [[architecture]] — §2.2 보안 규율, §1.2 3-gate (A ∧ B ∧ C → ALLOW)
- [[requirements]] — AUTHN-8(claim 누락 fail-closed + DB fallback), ABAC-7(can() 캐싱 금지), SEC-1(BOLA), TEN-2(타 org 404)
- [[e2e-journeys]] — D3-08 잘못된 입력·fail-closed Scenario Outline (8종 비정상 입력 전부 닫힘)
