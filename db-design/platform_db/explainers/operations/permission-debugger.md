---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p0
  - operability
  - authz
  - debugging
  - supportability
aliases:
  - Permission Debugger
  - 권한 디버거
  - 인가 trace
  - DBG-1
---

# Permission Debugger 설명 — "왜 막혔나"를 1초에 답하는 계약 (DBG-1)

> **대상**: 인가·운영을 공부하는 개발자 (공부용 · 사수 모드)
> **연관 문서**: [[operability|operability.md O2]] · [[gate-abc-flow|Gate A/B/C 흐름]] · [[requirements|requirements.md DBG-1]]

"왜 403이 떴어요?"는 SaaS 운영에서 가장 자주, 가장 답하기 어려운 질문입니다. 사용자는 막혔는데 이유를 모르고, CS는 개발자에게 묻고, 개발자는 `membership`·`org_entitlement`·`service_membership`·`delegation_grant` **네 테이블을 직접 뒤지며** 원인을 추적합니다. 이게 매번 반복되면 운영이 마비됩니다. **Permission Debugger**는 "이 거부가 *어느 게이트에서, 왜* 발생했는지"를 **1초에 trace로 반환**하는 계약입니다. 이 문서는 그게 왜 UI가 아니라 *계약*이고, 왜 platform_db의 책임인지 설명합니다.

> ⚠️ **구현 상태**: DBG-1은 **미설계(🔴)**입니다([[requirements]] §4). 다만 "ROI 최상"으로 꼽힌 항목 — 적은 구현으로 운영 비용을 크게 줄입니다. 이 문서는 *무엇을 만들어야 하는지(계약)*를 정의합니다.

---

## Q1. Permission Debugger가 뭔가요? 그냥 로그 보면 되는 거 아닌가요?

**Permission Debugger**는 "특정 요청이 왜 허용/거부됐는지"를 **3-gate 단계별 결과 + 사유**로 설명해 주는 도구(계약)입니다. 거부된 요청 하나를 주면 이렇게 답합니다:

```
입력: user=박교사, org=한울학원, action=lecture:publish
출력 (1초 이내):
  Gate A (소속)   = PASS   (membership ACTIVE, role=TEACHER)
  Gate B (결제)   = FAIL   (org_entitlement.status=EXPIRED, valid_until=2026-05-01)  ← 여기서 막힘
  Gate C (정책)   = SKIP   (B에서 이미 거부되어 평가 안 함)
  결론: 402 Payment Required — "구독이 만료되어 게시할 수 없음"
```

"로그 보면 되지 않나?"가 안 되는 이유: 로그는 *결과*("403 떴다")만 남기지 *왜*를 단계별로 안 남깁니다. 원인을 알려면 개발자가 그 시점의 `membership`·`org_entitlement`·`service_membership`·`delegation_grant`를 *직접 조회*해 머릿속에서 3-gate를 재현해야 합니다. 이게 [[gate-abc-flow|3-gate]]를 사람이 손으로 디버깅하는 것 — 느리고, 실수하고, 매번 반복됩니다.

비유하자면 **공항 보안 검색에서 막혔을 때**입니다. "그냥 못 갑니다"가 아니라 "여권은 OK(Gate A), 비자가 만료됨(Gate B에서 막힘), 그래서 탑승 불가"라고 *어느 관문에서 왜* 막혔는지 알려주면, 승객은 비자만 갱신하면 됩니다. Permission Debugger가 그 "어느 관문에서 왜"를 자동으로 답합니다.

> 💡 **한 줄 요약**: Permission Debugger는 거부된 요청에 "Gate A/B/C 중 어디서, 왜 막혔는지"를 단계별 trace로 답하는 도구입니다. 로그는 결과만 남기지만 이건 *원인*을 단계별로 설명합니다.

---

## Q2. 왜 "UI"가 아니라 "계약(contract)"이라고 부르나요?

여기가 핵심입니다. requirements DBG-1은 이를 **"UI가 아니라 *계약*으로서 platform_db 책임"**으로 못 박았습니다.

**UI**는 화면입니다 — 관리자 콘솔의 "권한 조회" 페이지 같은 것. 화면은 별도 ops 제품이 만들 수 있습니다([[operability]] 범위 규율: 콘솔 UI는 enable만).

**계약(contract)**은 *"이런 입력을 주면 이런 trace를 반드시 돌려준다"*는 **API/함수의 약속**입니다. 화면이 있든 없든, platform_db 패키지가 다음을 보장해야 합니다:

```typescript
// 계약: 이 함수는 어떤 거부든 Gate별 결과+사유를 1초 안에 돌려준다
async function explainDecision(input: {
  userPk: number; orgPk: number; action: string; resourcePk?: number;
}): Promise<PermissionTrace>;

interface PermissionTrace {
  gateA: { result: 'PASS' | 'FAIL' | 'SKIP'; reason: string };  // 소속
  gateB: { result: 'PASS' | 'FAIL' | 'SKIP'; reason: string };  // 결제
  gateC: { result: 'PASS' | 'FAIL' | 'SKIP'; reason: string };  // 정책
  conclusion: { allowed: boolean; httpStatus: number; message: string };
  evaluatedAt: string;
}
```

왜 계약이 중요할까요? **"개발자가 4개 테이블을 직접 뒤지는 것은 설계 실패"**(DBG-1)이기 때문입니다. 인가 로직(3-gate)은 platform_db가 소유합니다. 그렇다면 *"그 인가가 왜 그렇게 판단했는지 설명하는 것"*도 platform_db의 책임이어야 합니다. 이걸 각 서비스나 CS 도구가 테이블을 직접 조회해 재현하면:

```
❌ 계약이 없으면:
   - academy-api가 직접 4테이블 조회해 3-gate 재현 (로직 중복)
   - agent-api도 똑같이 또 재현 (또 중복)
   - CS 도구도 또... → 3-gate 로직이 곳곳에 복제됨 → 불일치·버그
```

```
✅ 계약이 있으면:
   - platform_db.explainDecision() 하나가 권위
   - UI든 CLI든 CS 도구든 이 계약만 호출 → 항상 같은 답
   - 3-gate 로직은 한 곳(platform)에만 존재
```

즉 UI는 선택이지만 *계약은 필수*입니다. 인가를 소유한 자가 인가 설명도 소유한다 — 이게 "UI가 아니라 계약"의 뜻입니다.

> 💡 **한 줄 요약**: UI(화면)는 ops 제품이 만들 수 있지만, "거부 사유를 Gate별 trace로 반환한다"는 *계약(API)*은 인가를 소유한 platform_db의 책임입니다. 4테이블 직접 조회는 3-gate 로직 중복을 부르는 설계 실패입니다.

---

## Q3. trace는 정확히 무엇을 담아야 하나요? 왜 PASS/FAIL 말고 사유까지요?

trace의 가치는 **사유(reason)**에 있습니다. "Gate B FAIL"만으로는 부족합니다 — *왜* FAIL인지가 행동을 결정합니다.

```
Gate B = FAIL 만 알면:  "결제 문제구나... 근데 뭐가?" → 또 조사
Gate B = FAIL (status=EXPIRED, valid_until=2026-05-01) 까지 알면:
        → "구독 갱신하세요" 즉시 안내 가능
```

각 게이트가 담아야 할 사유:

```
Gate A (소속, membership):
  PASS → "membership ACTIVE, platform_role=MEMBER"
  FAIL → "membership SUSPENDED" 또는 "NOT_A_MEMBER (org에 소속 없음)"

Gate B (결제, org_entitlement):
  PASS → "entitlement ACTIVE (valid_until=2026-12-31)"
  FAIL → "status=EXPIRED" 또는 "status=GRACE 만료" 또는 "entitlement 없음"

Gate C (정책, CASL ability):
  PASS → "role=DIRECTOR가 action 허용" 또는 "delegation으로 허용"
  FAIL → "role=STUDENT는 lecture:publish 권한 없음 (RBAC)" 또는 "위임 만료"
  SKIP → "선행 게이트(A/B)에서 거부되어 미평가"
```

**SKIP이 중요한 정보**라는 점도 짚을 만합니다. 3-gate는 [[fail-closed|deny-by-default]]로 *순차 단락 평가(short-circuit)*합니다 — A가 막히면 B/C는 평가 안 합니다. trace에 "C=SKIP (A에서 거부)"이 있으면 "C도 문제인가?"를 헷갈리지 않습니다. *어디서 처음 막혔는지*가 한눈에 보입니다.

```typescript
// 첫 FAIL에서 단락. 그 이후는 SKIP으로 표시 (왜 평가 안 했는지도 정보)
function explain(input): PermissionTrace {
  const a = checkGateA(input);
  if (!a.pass) return trace({ gateA: a, gateB: SKIP, gateC: SKIP, ... });
  const b = checkGateB(input);
  if (!b.pass) return trace({ gateA: a, gateB: b, gateC: SKIP, ... });
  const c = checkGateC(input);
  return trace({ gateA: a, gateB: b, gateC: c, ... });
}
```

> 💡 **한 줄 요약**: trace는 각 게이트의 PASS/FAIL/SKIP과 *구체적 사유*(어떤 컬럼이 어떤 값이라 막혔는지)를 담아야 하고, SKIP은 "어디서 처음 막혔는지"를 드러내 디버깅을 한눈에 만듭니다.

---

## Q4. "1초 안에" 반환이 왜 요건인가요? 성능이 왜 계약의 일부죠?

DBG-1의 계약에는 **응답 시간(1초)**이 명시돼 있습니다. 이게 단순 희망사항이 아니라 *계약의 일부*인 이유가 있습니다.

Permission Debugger는 **CS·운영자가 사용자와 통화하며 실시간으로** 쓰는 도구입니다. "고객님 왜 막히셨는지 확인해 볼게요" → trace 조회 → "구독이 만료되셨네요". 이 사이가 10초 걸리면 도구로서 쓸모가 없습니다. *대화 속도*로 답해야 합니다.

성능을 위한 설계 함의:

```
✅ explainDecision은 실제 3-gate와 같은 인덱스·경로를 써야 함
   - Gate B: idx_org_service_status (org_pk, service, status, valid_until) 핫패스 그대로
   - 별도의 무거운 분석 쿼리(JOIN 4테이블 풀스캔) ❌
   - "실제 판정과 같은 코드를 설명 모드로 실행"하는 게 이상적
     → 설명이 실제 판정과 100% 일치 보장(재현 오류 없음)
```

여기서 영리한 패턴: **판정 함수와 설명 함수를 같은 코드로** 만드는 것입니다. 판정(`can`)을 돌리되, 각 단계 결과를 *수집*하는 모드를 두면, "설명"이 "실제 판정"과 절대 어긋나지 않습니다. 따로 만들면 "판정은 막혔는데 설명은 통과라고 함" 같은 *재현 불일치*가 생깁니다.

> 💡 **한 줄 요약**: 1초 요건은 CS가 통화 중 실시간으로 쓰기 때문입니다. 그래서 실제 3-gate와 같은 인덱스·코드 경로로 돌려야 하고, 판정과 설명을 같은 코드로 만들면 재현 불일치도 없앱니다.

---

## Q5. 이게 audit_log나 break-glass랑 뭐가 다른가요?

셋 다 "무슨 일이 있었나"와 관련되지만 **목적과 시점**이 다릅니다.

| 도구 | 질문 | 시점 | 대상 |
|---|---|---|---|
| **Permission Debugger** | "*왜* 막혔나?" (원인 진단) | 거부 발생 후 진단 시 | CS·운영자·개발자 |
| **audit_log** | "*무슨* 일이 있었나?" (사실 기록) | 행위 시점에 append | 감사·포렌식 |
| **break-glass** | "긴급히 *직접* 고친다" | 장애 복구 시 | 운영자(승인 필요) |

- **audit_log**([[audit-hash-chain]])는 *결과*를 남깁니다("박교사 publish 시도 → DENY"). Permission Debugger는 그 DENY가 *왜* 났는지 *지금 다시 평가해* 설명합니다. audit_log는 과거 기록, Debugger는 현재 상태 진단.
- **break-glass**([[break-glass]])는 *고치는* 도구(운영자가 entitlement 강제부여 등). Debugger는 *진단만* 하는 읽기 전용 도구 — 아무것도 바꾸지 않습니다.

흐름으로 보면 셋이 협업합니다:

```
사용자 "왜 막혀요?"
  → Permission Debugger로 진단 ("Gate B FAIL, 구독 만료")  ← 읽기, 원인 파악
  → (정당하면) break-glass로 임시 복구 또는 정상 결제 안내   ← 쓰기, 승인 필요
  → 모든 조치는 audit_log에 기록                          ← append, 사후 추적
```

> 💡 **한 줄 요약**: Permission Debugger는 "왜 막혔나"를 *지금 다시 평가해 진단*하는 읽기 전용 도구입니다. audit_log(과거 사실 기록)·break-glass(긴급 수정)와 목적·시점이 다르며, 셋이 협업합니다.

---

## 용어 정리

| 용어 | 한 줄 정의 |
|---|---|
| **Permission Debugger** | 거부된 요청에 Gate A/B/C 단계별 결과+사유를 반환하는 진단 도구(계약) |
| **계약(contract)** | "이런 입력 → 이런 출력을 보장"하는 API의 약속. UI와 무관하게 platform_db가 책임 |
| **trace** | 한 요청의 게이트별 평가 경로·결과·사유를 담은 구조체 |
| **3-gate** | Gate A(소속)·B(결제)·C(정책). 순차 단락 평가([[gate-abc-flow]]) |
| **short-circuit(단락 평가)** | 첫 FAIL에서 멈추고 이후는 SKIP. deny-by-default와 한 쌍 |
| **SKIP** | 선행 게이트 거부로 평가하지 않은 상태. "어디서 처음 막혔나"를 드러냄 |
| **supportability(지원 가능성)** | 운영·CS가 문제를 얼마나 빨리 진단·해결할 수 있나 (DBG가 높임) |
| **재현 불일치** | 설명 로직이 실제 판정과 달라 "판정은 막혔는데 설명은 통과"가 나는 버그 |

---

## 테스트 방법
> 🧪 *실제 DB·ORM·운영에서 돌리는 법*: [[testing-strategy]] · [[orm-testing-drizzle]]


Permission Debugger의 핵심 검증은 **(1) 설명이 실제 판정과 일치하는가**(재현 불일치 0), **(2) 어느 게이트에서 막혔는지·SKIP이 정확한가**, **(3) 1초 안에 답하는가**입니다.

```typescript
// permission-debugger.spec.ts  (⚠️ DBG-1 미구현 — 구현 시 이 계약을 만족해야 함)
import { describe, it, expect } from "vitest";
import { explainDecision, checkGates } from "../src/authz";

describe("Permission Debugger 계약 (DBG-1)", () => {
  it("Gate B 만료 거부를 정확히 trace로 설명한다", async () => {
    await seedMembership({ userPk: 9, orgPk: 1, role: "TEACHER", status: "ACTIVE" });
    await seedEntitlement({ orgPk: 1, service: "ACADEMY", status: "EXPIRED" });

    const t = await explainDecision({ userPk: 9, orgPk: 1, action: "lecture:publish" });

    expect(t.gateA.result).toBe("PASS");
    expect(t.gateB.result).toBe("FAIL");
    expect(t.gateB.reason).toMatch(/EXPIRED/);     // 구체적 사유 포함
    expect(t.gateC.result).toBe("SKIP");           // B에서 막혀 C는 미평가
    expect(t.conclusion.httpStatus).toBe(402);
  });

  it("Gate A 거부 시 B·C 모두 SKIP", async () => {
    await seedMembership({ userPk: 9, orgPk: 1, status: "SUSPENDED" });
    const t = await explainDecision({ userPk: 9, orgPk: 1, action: "lecture:publish" });
    expect(t.gateA.result).toBe("FAIL");
    expect(t.gateB.result).toBe("SKIP");
    expect(t.gateC.result).toBe("SKIP");
  });

  // ★ 가장 중요: 설명이 실제 판정과 100% 일치 (재현 불일치 0)
  it("explainDecision의 결론은 실제 checkGates 판정과 항상 일치한다", async () => {
    const cases = await seedManyRandomScenarios(200);
    for (const c of cases) {
      const realAllowed = await checkGates(c).then(() => true).catch(() => false);
      const explained = await explainDecision(c);
      expect(explained.conclusion.allowed).toBe(realAllowed);  // 어긋나면 실패
    }
  });

  it("1초 안에 응답한다 (CS 실시간 사용)", async () => {
    const start = performance.now();
    await explainDecision({ userPk: 9, orgPk: 1, action: "lecture:publish" });
    expect(performance.now() - start).toBeLessThan(1000);
  });
});
```

**체크리스트:**

```
□ Gate별 result(PASS/FAIL/SKIP)와 구체적 사유(reason)가 채워지는가?
□ 첫 FAIL 이후 게이트가 SKIP으로 표시되는가? (어디서 처음 막혔는지)
□ ★ explainDecision 결론 == 실제 3-gate 판정 (재현 불일치 0)
□ 1초 내 응답 (실제 핫패스 인덱스 사용, 풀스캔 JOIN 금지)
□ 읽기 전용 — 아무 데이터도 변경하지 않음 (audit_log조차 INSERT 안 함, 진단일 뿐)
□ CS가 본 trace와 사용자가 실제 받은 응답(402 등)이 일치하는가?
```

---

## 마치며

Permission Debugger의 본질은 **"인가를 소유한 자가 인가 설명도 소유한다"**입니다. 3-gate를 platform_db가 만들었으니, "그 게이트가 왜 그렇게 판단했는지"도 platform_db가 1급 계약으로 답해야 합니다. 그래야:

1. 4개 테이블을 손으로 뒤지는 반복 노동이 사라지고,
2. 3-gate 로직이 서비스마다 복제되지 않고,
3. CS가 통화 중 실시간으로 "구독 만료네요"를 답할 수 있습니다.

미구현(🔴)이지만 "ROI 최상"으로 꼽힌 이유가 여기 있습니다 — `explainDecision` 함수 하나(판정 코드를 설명 모드로 재사용)가 운영 비용을 크게 줄입니다. 새 게이트나 권한 규칙을 추가할 때는 "이 거부가 Debugger trace에 *이해 가능한 사유*로 나오는가?"를 함께 확인하세요. 막는 것만큼 *왜 막혔는지 설명하는 것*도 인가 시스템의 책임입니다.

---

## 연결된 개념

- [[gate-abc-flow|Gate A/B/C 흐름]] — Debugger가 단계별로 설명하는 바로 그 3-gate
- [[fail-closed|fail-closed]] — 단락 평가(첫 FAIL에서 SKIP)와 deny-by-default
- [[gate-b-entitlement|Gate B 이용권]] — 가장 흔한 거부 지점(구독 만료)의 사유 출처
- [[audit-hash-chain|감사 로그]] — 과거 사실 기록(Debugger의 현재 진단과 대비)
- [[break-glass|Break-glass]] — 진단 후 정당한 긴급 복구(Debugger는 읽기, break-glass는 쓰기)
- [[perm-version-propagation|perm_version]] — stale 토큰 거부도 Debugger가 설명해야 하는 사유
> 소스 문서
- [[operability]] — O2 Supportability·Permission Debugger 계약
- [[requirements]] — DBG-1(거부에 A/B/C 결과+사유 1초 trace, platform_db 책임, ROI 최상)
- [[gate-abc-flow]] · [[schema-reference]] — §E 3-gate 평가 로직
