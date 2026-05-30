---
difficulty: 고급
tags:
  - platform-db
  - explainer
  - p1
  - perm-version
  - security
  - authz
aliases:
  - perm_version 전파
  - 권한 변경 즉시 반영
  - perm-version-propagation
  - 캐시 무효화
---

# 권한 변경 즉시 반영 설명 — perm_version 전파

> **대상**: JWT·캐시·인가를 다루는 개발자 (공부용)
> **연관 문서**: [[architecture|architecture.md §1.2]] · [[schema-reference|schema-reference.md §E.3·E.5]]

권한을 다루다 보면 반드시 부딪히는 골치 아픈 문제가 있습니다. **"권한을 방금 회수했는데, 그 사람 토큰은 아직 1시간 살아 있다."** 강사를 해고했는데, 그 강사의 JWT에는 여전히 "나는 강사다"라고 적혀 있습니다. 토큰이 만료될 때까지 최대 1시간, 회수가 반영되지 않는 위험한 창(window)이 생깁니다. 이 문서는 우리 `platform_db`가 `perm_version`이라는 단순한 정수 카운터 하나로 이 "1시간의 틈"을 어떻게 메우는지 설명합니다.

---

## Q1. stale이 뭔가요? 토큰이 왜 낡아버리나요?

**stale**은 "낡은·뒤처진" 상태입니다. 캐시된 정보가 원본보다 오래돼서, 더 이상 진실이 아닌 상태를 말합니다.

JWT(JSON Web Token)는 **claims**(주장)를 담은 토큰입니다. "이 사람의 firebase_uid는 X다", "org Y의 TEACHER다" 같은 정보가 토큰 안에 박혀 발급됩니다. 문제는 이 claims가 **발급 시점의 스냅샷**이라는 점입니다. Firebase는 보통 **TTL(Time To Live) 1시간**짜리 토큰을 줍니다.

비유하자면 JWT는 **1시간짜리 출입증**입니다.

```
09:00  강사 박씨, 출입증 발급받음 → 출입증에 "TEACHER" 도장
09:30  학원장이 박씨를 해고 (DB에서 권한 회수)
09:31  박씨, 여전히 09:00 출입증 들고 있음 → 도장은 아직 "TEACHER"
       └─ 이 출입증은 stale (낡음). 회수가 반영 안 됨.
10:00  출입증 만료 → 그제서야 새 출입증 못 받음 (해고 반영)

       09:30 ~ 10:00 = stale window (최대 ~1h) ← 위험 구간
```

해고는 **즉시**여야 하는데, 출입증(JWT)은 **1시간** 유효합니다. 이 시간차가 stale window입니다. routine read를 매번 DB로 검증하면 이 문제가 없지만, 그러면 모든 요청이 DB를 때려 성능이 무너집니다(다음 Q에서 설명). 그래서 "토큰은 빠르게, 무효화는 즉시"라는 두 마리 토끼를 잡는 장치가 필요합니다 — 그게 `perm_version`입니다.

> 💡 **한 줄 요약**: JWT는 1시간짜리 출입증이라, 발급 후 권한이 바뀌어도 토큰 안의 claims는 최대 1시간 동안 낡은(stale) 상태로 남습니다. 이 "1시간의 틈"이 perm_version이 풀어야 할 문제입니다.

---

## Q2. 그냥 권한 판단(can())을 캐싱 안 하면 안 되나요? 왜 굳이?

이건 정말 좋은 질문이고, 답이 우리 설계의 핵심입니다. "매 요청마다 DB로 권한을 확인하면 stale이 없잖아?" 맞습니다. 하지만 그러면 **성능이 무너집니다.**

```
강의 목록 1번 보기 → Gate A(membership) DB 조회
                   → Gate B(entitlement) DB 조회
                   → Gate C(role·delegation) DB 조회
= 화면 한 번에 권한 DB 쿼리 3~4번. 트래픽 1만 req/s면 권한 쿼리만 3~4만 req/s
```

그래서 우리는 **NFR-1**이라는 원칙을 둡니다.

> **NFR-1**: routine read(일상 조회) 권한 평가는 **DB hit 0**(JWT claims로 판단). **sensitive write(민감 쓰기)만 DB 재검증.**

핵심은 **요청을 두 등급으로 나누는 것**입니다.

```
routine read  (강의 목록·프로필 보기 등 — 위험 낮음)
   → JWT claims만 보고 판단. DB 안 침. 빠름. 단, ~1h stale 가능 (감수)

sensitive write (publish·delete·결제·역할변경 — 위험 높음)
   → @VerifyOnDb로 DB 재검증. 느리지만 stale 토큰도 즉시 차단
```

위험이 낮은 동작은 stale을 *감수*하고 빠르게(JWT), 위험이 높은 동작만 *비용을 치르고* 정확하게(DB) 처리합니다. 그래서 **최종 `can()` 결과는 캐싱하지 않습니다**(architecture §1.2). 캐싱하는 건 입력 블록(membership/grants/entitlement)뿐이고, TTL은 60초입니다.

```typescript
// architecture §1.2 — 입력만 캐싱, 최종 can()은 캐싱 금지
// 캐싱 O: membership/grants/entitlement 블록 (TTL 60s)
// 캐싱 X: ability.can(action, resource) 의 결과
//   → 같은 입력이라도 매번 평가. 권한 변경이 입력 갱신으로 즉시 반영되게.
```

만약 `can()` 결과 자체를 캐싱하면, "박씨는 회원관리 가능 = true"가 캐시에 박혀서 회수 후에도 통과합니다. 입력만 캐싱하면, 입력이 바뀌는 순간 다음 평가가 자동으로 새 결과를 냅니다.

> 💡 **한 줄 요약**: 매번 DB로 권한을 보면 성능이 무너집니다(요청당 3~4 쿼리). 그래서 routine read는 JWT로 빠르게(DB hit 0, NFR-1), sensitive write만 DB 재검증합니다. 최종 can() 결과는 절대 캐싱하지 않습니다.

---

## Q3. perm_version은 어떻게 동작하나요? bump하면 무슨 일이 일어나나요?

`perm_version`은 **권한 변경 카운터**입니다. `organization.perm_version`과 `identity_user.perm_version` 두 컬럼에 `BIGINT UNSIGNED NOT NULL DEFAULT 1`로 박혀 있습니다.

```sql
-- schema-reference.md §D.1, §D.3
organization.perm_version  BIGINT UNSIGNED NOT NULL DEFAULT 1
identity_user.perm_version BIGINT UNSIGNED NOT NULL DEFAULT 1
```

원리는 단순합니다. **권한이 바뀔 때마다 이 숫자를 +1(bump)하고, 클라이언트가 자기가 아는 숫자를 매 요청에 들고 오게 해서, 서버 숫자와 다르면 "너 토큰 낡았어"라고 알려줍니다.**

비유하자면 **문서의 버전 번호**입니다. 내 손엔 v5 사본이 있는데, 서버 원본이 v6이면 "당신 사본은 구버전입니다"라고 즉시 알 수 있죠.

전체 흐름:

```
1. 학원장이 박씨 역할 변경 → bumpPermVersion
   UPDATE organization SET perm_version = perm_version + 1 WHERE pk = ?
   (5 → 6 으로 증가)

2. 박씨가 다음 API 호출 (토큰엔 아직 perm_version: 5)
   요청과 함께 클라이언트가 아는 버전을 보냄

3. 서버 응답에 현재 버전을 실어 보냄
   X-Perm-Version: 6   ← 서버는 6인데

4. 클라이언트가 불일치 감지 (내 토큰=5, 서버=6)
   → "내 토큰 낡았다" → forceRefresh(true)로 토큰 강제 갱신
   → 새 토큰엔 최신 claims (perm_version: 6, 변경된 역할)
```

코드로 보면:

```typescript
// schema-reference.md §E.3 — 역할 변경 시 perm_version bump
async function patchMemberRole(orgPk, memberPk, role) {
  await db.transaction(async (tx) => {
    await tx.update(serviceMembership).set({ roleCode: role })./* ... */;
    await bumpPermVersion(tx, orgPk);   // UPDATE organization SET perm_version = perm_version + 1
  });
  // 다음 토큰 갱신(~1h) 시 custom claims 반영 / 즉각 적용은 클라가 forceRefresh(true)
}
```

```typescript
// 클라이언트 측 — X-Perm-Version 불일치 감지 → 강제 갱신
const res = await api.get("/v1/lectures");
const serverPv = Number(res.headers["x-perm-version"]);
if (serverPv > myTokenPv) {
  await firebaseUser.getIdToken(/* forceRefresh */ true);  // 토큰 즉시 재발급
  // 새 토큰에 최신 claims·perm_version 반영
}
```

`forceRefresh(true)`는 "캐시된 토큰 말고, 서버에서 새로 받아와"라는 명령입니다. 평소엔 Firebase가 캐시된 토큰을 주지만, 이 플래그가 있으면 강제로 새 토큰을 발급받아 최신 권한이 즉시 반영됩니다.

> 💡 **한 줄 요약**: 권한이 바뀌면 perm_version을 +1 하고(bump), 응답 `X-Perm-Version` 헤더로 서버 버전을 알립니다. 클라이언트는 자기 토큰 버전과 다르면 forceRefresh(true)로 토큰을 강제 갱신해 최신 권한을 받습니다.

---

## Q4. 그래도 forceRefresh 전까지는 stale이잖아요. 민감 작업은 어떻게 막나요?

날카로운 지적입니다. perm_version 불일치를 클라이언트가 감지해서 forceRefresh하기까지는 *클라이언트의 협조*가 필요합니다. 악의적 클라이언트라면 "일부러 갱신 안 하고" stale 토큰으로 민감 작업을 시도할 수 있습니다.

그래서 **민감 쓰기에는 클라이언트를 믿지 않는 두 번째 방어선**이 있습니다 — **@VerifyOnDb (§E.5)**입니다.

```
출입증 비유로:
- routine read = 평범한 출입 → 출입증(JWT)만 보고 통과 (빠름)
- sensitive write = 금고실 입장 → 입구 경비가 본부에 무전으로 실시간 대조
                                  "이 사람 아직 권한 있나?" → DB 재조회
```

민감 구역(publish·delete·결제·역할변경)은 출입증이 멀쩡해 보여도, **입구에서 DB와 실시간 대조**합니다. stale 토큰엔 "TEACHER"라 적혀 있어도, DB를 보면 이미 회수됐으면 즉시 막힙니다.

```typescript
// schema-reference.md §E.5 — Sensitive Write는 DB 재검증
@VerifyOnDb()   // 민감 쓰기 직전 DB로 최신 권한 재확인
async function exerciseDelegation(userPk, orgPk, capability) {
  // JWT claims는 stale일 수 있으니, DB로 진짜 상태를 본다
  const grant = await getActiveDelegation(userPk, orgPk, capability);
  if (!grant || grant.status !== "ACTIVE") {
    throw new ForbiddenException("DELEGATION_REVOKED");  // stale 토큰이어도 즉시 403
  }
  // perm_version 불일치도 함께 감지 → 응답에 X-Perm-Version 실어 재발급 유도
}
```

이게 바로 [[gate-abc-flow|Gate A의 ~1h stale window]]를 메우는 핵심입니다. Gate A의 실시간 멤버십 재검증(`GateAGuard`)은 현재 Icebox(미구현)이지만, **@VerifyOnDb가 민감 쓰기 경로에서 그 틈을 메웁니다.** 일상 읽기는 stale을 감수하되(위험 낮음), 민감 쓰기는 stale을 허용하지 않습니다(위험 높음).

정리하면 우리는 **두 겹**으로 막습니다.

```
1겹 (자가치유): perm_version 불일치 → X-Perm-Version 헤더 → 클라가 forceRefresh
                → 협조적 클라이언트의 토큰을 즉시 최신화
2겹 (강제차단): @VerifyOnDb → 민감 쓰기는 DB 재검증 → stale 토큰도 즉시 막힘
                → 비협조적 클라이언트도 우회 불가
```

> 💡 **한 줄 요약**: forceRefresh는 협조적 클라이언트용 자가치유이고, @VerifyOnDb는 비협조적 클라이언트도 막는 강제 방어선입니다. 민감 쓰기는 DB로 재검증하므로, stale 토큰을 들고 와도 즉시 403으로 차단됩니다.

---

## Q5. user_pv랑 org_pv는 왜 따로 있나요? 하나면 안 되나요?

`perm_version`이 **두 군데**(`identity_user.perm_version`, `organization.perm_version`)에 있는 건 의도된 설계입니다. **NFR-6**(무효화 폭 분리)입니다.

핵심은 **"권한 변경이 누구에게 영향을 주는가"가 다르다**는 점입니다.

```
org_pv (조직 단위) bump → 그 org의 모든 멤버 토큰이 stale 처리됨
   예: 학원이 구독 만료(Gate B 변경), 학원 전체 정책 변경
   → 학원 멤버 전원이 forceRefresh 필요

user_pv (개인 단위) bump → 그 사용자 한 명의 토큰만 stale 처리됨
   예: 박씨 한 명의 역할 변경, 박씨 계정 정지
   → 박씨만 forceRefresh, 나머지 멤버는 영향 없음
```

만약 perm_version이 org 하나뿐이라면, **박씨 한 명의 역할만 바꿔도 학원 멤버 전원의 토큰이 무효화**됩니다. 학원에 1,000명이 있으면 1,000명이 동시에 forceRefresh를 호출 — 이게 **thundering herd(쿵쾅거리는 무리)** 문제입니다. 인증 서버에 갑자기 1,000개의 토큰 재발급 요청이 몰립니다.

비유하자면, 아파트에서 **301호 비밀번호만 바꿨는데 전 세대 비밀번호가 동시에 리셋**되는 격입니다. 301호만 바꾸려면 301호 카운터만 올려야 합니다.

```
무효화 폭(blast radius) 분리:
  개인 변경(역할·정지)  → user_pv만 bump → 본인만 갱신       (좁은 폭)
  조직 변경(구독·정책)  → org_pv만 bump  → org 전체 갱신      (넓은 폭, 필요할 때만)
```

`/me/permissions` 응답에 `{ user_pv, org_pv }` 두 값이 함께 실려서, 클라이언트는 둘 중 하나라도 자기 토큰과 어긋나면 갱신합니다(NFR-6). 변경의 영향 범위에 딱 맞는 무효화 — 이게 두 컬럼을 분리한 이유입니다.

> 💡 **한 줄 요약**: 개인 권한 변경은 user_pv만, 조직 전체 변경은 org_pv만 bump해서, 무효화 폭을 영향 범위에 맞춥니다. 한 명 바꿨는데 전원이 토큰을 재발급하는 thundering herd를 막기 위한 분리입니다(NFR-6).

---

## 용어 정리

| 용어 | 한 줄 정의 |
|---|---|
| stale | 캐시·토큰이 원본보다 낡아서 더 이상 진실이 아닌 상태 |
| claims | JWT 안에 박힌 주장(firebase_uid·org·role 등). 발급 시점의 스냅샷 |
| TTL (Time To Live) | 캐시·토큰의 유효 수명. Firebase JWT는 보통 ~1시간 |
| cache invalidation | 캐시된 값이 더는 유효하지 않다고 알려 다시 받게 하는 것 |
| perm_version | 권한 변경 카운터. bump(+1)하면 기존 토큰이 stale로 판별됨 (user_pv/org_pv 분리) |
| X-Perm-Version | 서버의 현재 perm_version을 실어 보내는 응답 헤더. 불일치가 재발급 신호 |
| forceRefresh | 캐시된 토큰을 버리고 서버에서 새 토큰을 강제 발급받는 호출 |
| @VerifyOnDb | 민감 쓰기 직전 DB로 권한을 재검증하는 장치. stale 토큰도 즉시 차단(§E.5) |
| thundering herd | 한 번에 너무 많은 클라이언트가 동시에 같은 갱신을 몰아쳐 서버가 받는 부하 |

---

## 테스트 방법

검증해야 할 핵심은 두 가지입니다. ① **권한 회수 후 stale 토큰으로 민감 작업을 하면 403으로 막힌다**(@VerifyOnDb), ② **perm_version bump 후 X-Perm-Version이 증가한다**(전파). [[e2e-journeys]] **C-04**(정상 위임 lifecycle)와 **D3-07**(오부여 정정)을 코드로 옮긴 형태입니다.

### 통합 테스트 (vitest + supertest) — API 경계 블랙박스

```typescript
import { describe, it, expect, beforeAll } from "vitest";
import request from "supertest";
import { app } from "../src/main";

describe("perm_version 전파 & stale 토큰 차단 (C-04 / D3-07)", () => {
  let ownerToken: string;        // 김지영(OWNER) — 위임 부여/회수 주체
  let teacherStaleToken: string; // 박교사 — 위임 반영 전 stale 토큰
  const ORG = "01H...HANUL";
  const TARGET_MEMBER = "01H...TEACHER_B";

  beforeAll(async () => {
    ownerToken = await issueToken({ orgPk: ORG, role: "DIRECTOR" });
    teacherStaleToken = await issueToken({ orgPk: ORG, role: "TEACHER" }); // pv 낡음
  });

  // C-04 핵심: 부여 후 200, 회수 후 같은 stale 토큰으로 403 (DB 재검증으로 즉시 차단)
  it("위임 부여→행사(200)→회수→stale 토큰 재행사(403)", async () => {
    // 1. 위임 부여
    const grant = await request(app)
      .post("/v1/delegations")
      .set("Authorization", ownerToken).set("x-org-pk", ORG)
      .send({ grantee: "park@x.com", capability: "ACADEMY.MANAGE_MEMBERS" });
    expect(grant.status).toBe(201);
    const delegationId = grant.body.id;

    // 2. 행사 — forceRefresh로 갱신한 토큰이면 성공
    const fresh = await refreshToken(teacherStaleToken);  // 클라가 forceRefresh
    const exercise = await request(app)
      .post(`/v1/members/${TARGET_MEMBER}/role`)
      .set("Authorization", fresh).set("x-org-pk", ORG)
      .send({ role: "TEACHER" });
    expect(exercise.status).toBe(200);   // 위임 행사 성공

    // 3. 회수
    const revoke = await request(app)
      .delete(`/v1/delegations/${delegationId}`)
      .set("Authorization", ownerToken).set("x-org-pk", ORG);
    expect(revoke.status).toBe(204);

    // 4. 회수 미반영 stale 토큰으로 같은 민감 작업 재시도 → 403 (DB 재검증)
    const after = await request(app)
      .post(`/v1/members/${TARGET_MEMBER}/role`)
      .set("Authorization", fresh)   // 회수 사실 모르는 stale 토큰
      .set("x-org-pk", ORG)
      .send({ role: "TEACHER" });
    expect(after.status).toBe(403);
    expect(after.body.code).toBe("DELEGATION_REVOKED");  // stale 토큰도 즉시 막힘

    // 같은 호출이 단계 2의 200 → 단계 4의 403 으로 전환됨이 핵심 단언
  });

  // perm_version bump 후 X-Perm-Version 증가 관찰
  it("역할 변경 bump 후 X-Perm-Version 이 증가한다", async () => {
    const before = await request(app)
      .get("/v1/lectures").set("Authorization", ownerToken).set("x-org-pk", ORG);
    const pvBefore = Number(before.headers["x-perm-version"]);

    // 역할 변경 → perm_version bump 유발
    await request(app)
      .post(`/v1/members/${TARGET_MEMBER}/role`)
      .set("Authorization", ownerToken).set("x-org-pk", ORG)
      .send({ role: "TEACHER" });

    const after = await request(app)
      .get("/v1/lectures").set("Authorization", ownerToken).set("x-org-pk", ORG);
    const pvAfter = Number(after.headers["x-perm-version"]);

    expect(pvAfter).toBeGreaterThan(pvBefore);  // bump 반영 — 클라 forceRefresh 신호
  });

  // stale 토큰이 routine read는 통과하되(NFR-1), X-Perm-Version 불일치를 받는다
  it("stale 토큰의 routine read는 통과하지만 X-Perm-Version 불일치를 신호받는다", async () => {
    const res = await request(app)
      .get("/v1/lectures")
      .set("Authorization", teacherStaleToken).set("x-org-pk", ORG);
    expect(res.status).toBe(200);  // routine read는 stale 감수 (NFR-1)
    expect(Number(res.headers["x-perm-version"]))
      .toBeGreaterThan(tokenPv(teacherStaleToken));  // 갱신 신호는 받음
  });
});
```

### "무엇을 단언하나" 체크리스트

- [ ] **부여 후 200, 회수 후 같은 stale 토큰으로 403** — 같은 호출이 `200 → 403`으로 전환됨(C-04). 토큰 캐시가 아니라 DB 재검증으로 막힘
- [ ] **회수 차단 사유가 정확** — `body.code === "DELEGATION_REVOKED"` (단순 403이 아니라 *왜* 막혔는지)
- [ ] **bump 후 X-Perm-Version 증가** — `pvAfter > pvBefore`. 전파 신호가 헤더로 나감
- [ ] **routine read는 stale을 감수** — stale 토큰의 일상 읽기는 200(NFR-1), 단 X-Perm-Version 불일치로 갱신 신호는 받음
- [ ] **sensitive write는 stale을 불허** — 민감 쓰기 경로(@VerifyOnDb)는 stale 토큰이어도 403
- [ ] **부분 효과 없음** — 403으로 막힌 역할 변경이 실제로 반영 안 됐는지 후속 `GET /v1/members/{id}`로 확인

> ⚠️ **테스트 함정**: 권한 변경 *직후 새 토큰*으로만 테스트하면 stale 차단을 검증하지 못합니다. 핵심은 "낡은 토큰을 일부러 들고 와서" 민감 작업이 막히는지 보는 것입니다. C-04처럼 *같은 토큰*으로 200 → 403 전환을 관찰해야 @VerifyOnDb가 실제로 동작함을 증명합니다.

---

## 마치며

`perm_version`은 단순한 정수 카운터 하나지만, "토큰은 빠르게, 무효화는 즉시"라는 모순을 우아하게 풉니다. 핵심을 세 줄로 요약하면:

1. **routine read는 JWT로 빠르게**(DB hit 0, NFR-1) — stale ~1h는 감수. 위험이 낮으니까.
2. **변경되면 perm_version bump → X-Perm-Version 헤더 → forceRefresh** — 협조적 클라이언트의 자가치유.
3. **sensitive write는 @VerifyOnDb로 DB 재검증** — stale 토큰도 즉시 차단. 비협조적 클라이언트도 우회 불가.

그리고 무효화 폭은 user_pv/org_pv로 분리해서(NFR-6), 한 명 바꿨다고 전원이 토큰을 재발급하는 thundering herd를 피합니다.

권한 변경 코드를 짤 때 항상 물어보세요: **"이 변경 후 bumpPermVersion을 호출했는가? 이게 user_pv인가 org_pv인가? 이 민감 작업은 @VerifyOnDb로 보호되는가?"** 이 세 질문에 답할 수 있으면, "해고했는데 1시간 동안 안 막히는" 사고는 일어나지 않습니다.

---

## 연결된 개념

- [[gate-abc-flow|Gate A/B/C 흐름]] — @VerifyOnDb가 메우는 Gate A의 ~1h stale window
- [[fail-closed|fail-closed 원칙]] — perm_version 불일치 → 401, 재검증 오류 → 503으로 닫히는 deny-by-default
- [[role-capability|role 2단 분리와 capability]] — 역할 변경이 perm_version bump를 유발하는 지점
- [[gate-b-entitlement|Gate B 이용권]] — 구독 만료 시 org_pv bump로 org 전체를 무효화하는 사례
- [[break-glass|Break-glass]] — 긴급 권한 부여/회수에서도 perm_version 전파가 적용되는 맥락
- [[multitenancy-rls|Pool 모델 멀티테넌시]] — perm_version이 user/org 두 축으로 나뉘는 테넌트 경계
> 소스 문서
- [[schema-reference]] — §E.3 perm_version 동기화(bump·forceRefresh), §E.5 Sensitive Write(@VerifyOnDb), §D.1·D.3 perm_version 컬럼
- [[architecture]] — §1.2 perm_version self-healing(user_pv/org_pv 분리, thundering herd 회피), 최종 can() 캐싱 금지
- [[requirements]] — NFR-1(routine read DB hit 0, sensitive write만 재검증), NFR-6(perm_version 전파·무효화 폭 분리), RBAC-5(역할 변경 즉시 반영)
- [[e2e-journeys]] — C-04 위임 가치흐름(부여→행사→회수→stale 즉시 차단), D3-07 오부여 정정(200→403 전환)
