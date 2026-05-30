---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p0
  - observability
  - reliability
  - slo
  - operations
aliases:
  - 관측성
  - observability
  - SLO
  - SLI
  - 신뢰성 계약
---

# 관측성과 신뢰성 계약 (SLO·SLI·모니터링)

> **대상**: 운영/SRE 경험이 적은 개발자
> **연관 문서**: [[operability|operability.md O5·O6]] · [[requirements|requirements.md AVAIL-1·AUD-3]] · [[audit-two-lane|audit-two-lane.md]]

"관측성", "SLO", "P99", "error budget" — 인가 시스템을 만들었으니 "잘 돌아가는지"를 알아야 하는데, 막상 이 단어들이 뜨면 막막합니다. 이 문서는 `platform_db`가 *무엇을 보고, 무엇을 약속하고, 어떻게 사고를 빨리 알아채는지*를 설명합니다. 핵심 질문 하나로 요약됩니다: **"장애가 났는지 어떻게 아는가, 그리고 무엇이 죽어도 무엇은 버틴다고 약속할 수 있는가."**

---

## Q1. 관측성(observability)이 뭔가요? 모니터링이랑 다른 건가요?

**관측성**은 시스템 *안에서* 무슨 일이 일어나는지를 *밖에서* 알아낼 수 있는 능력입니다. 시스템을 뜯어보지 않고도, 시스템이 밖으로 내보내는 신호(signal)만으로 내부 상태를 추론할 수 있으면 "관측 가능하다"고 합니다.

비유하자면 자동차 계기판입니다. 엔진을 열어보지 않아도 속도계·연료계·경고등만 보고 "지금 시속 100, 연료 절반, 엔진 정상"을 압니다. 관측성은 시스템에 계기판을 달아주는 일입니다.

관측성은 보통 **3축(three pillars)**으로 이야기합니다:

```
metrics (지표)  │ 숫자 시계열. "초당 DENY 몇 건", "P99 지연 180ms"
                │ → 계기판의 속도계·연료계. 집계·알림에 강함.
logs (로그)     │ 개별 사건의 기록. "userPk=42가 PUBLISH_VIDEO 거부됨, 사유 entitlement EXPIRED"
                │ → 블랙박스. 사후에 "정확히 무슨 일이 있었나"를 본다.
traces (추적)   │ 한 요청이 거쳐간 경로. "요청 X가 GateA→GateB→GateC를 0.18초에 통과"
                │ → 정비 이력 추적. 어느 구간이 느렸나/어디서 막혔나.
```

**모니터링과의 차이**는 미묘하지만 중요합니다. 모니터링은 "*미리 정해둔* 질문에 답하는 것"(예: "CPU가 80% 넘었나?")이고, 관측성은 "*미리 예상 못 한* 질문에도 답할 수 있는 능력"입니다. 모니터링은 관측성 위에서 돌아가는 한 가지 활용이라고 보면 됩니다.

> 💡 **한 줄 요약**: 관측성은 시스템에 계기판(metrics)·블랙박스(logs)·이력추적(traces)을 달아서, 안을 열어보지 않고도 "장애가 났는지" 알 수 있게 하는 능력입니다.

---

## Q2. SLI, SLO, SLA가 자꾸 헷갈려요. 정확히 뭐가 다른가요?

세 단어는 비슷하게 생겼지만 **측정값 → 목표 → 계약** 순으로 단계가 다릅니다. 하나씩 보면 명확합니다.

```
SLI (Service Level Indicator)  = 측정값. "실제로 지금 어떤가"
   예: 인가 응답 P99 지연 = 180ms (지금 측정된 값)

SLO (Service Level Objective)  = 목표. "우리가 지키려는 선"
   예: 인가 응답 P99 < 200ms 를 30일 중 99.9% 시간 동안 유지

SLA (Service Level Agreement)  = 계약. "못 지키면 어떻게 보상한다"
   예: 월 99.9% 미달 시 요금 10% 크레딧 환급 (법적 약속)
```

비유로 정리하면:

```
SLI = 속도계가 지금 가리키는 숫자 (시속 95)
SLO = "시속 100을 유지하겠다"는 스스로의 목표
SLA = "100 못 지키면 환불"이라는 손님과의 계약서
```

핵심 구분점:

| 구분 | 무엇 | 누가 보나 | 위반하면 |
|---|---|---|---|
| SLI | 객관적 측정값(숫자) | 엔지니어 | (위반 개념 없음 — 그냥 현재값) |
| SLO | 내부 목표 | 엔지니어·운영 | error budget 소진 → 배포 동결 등 *내부* 조치 |
| SLA | 외부 계약 | 영업·법무·고객 | *금전/법적* 보상 |

대부분의 팀은 SLA(외부 계약)는 보수적으로 잡고, 내부적으로는 더 빡빡한 SLO를 둡니다. 예를 들어 고객에겐 SLA 99.9%를 약속하되 내부 SLO는 99.95%로 잡아서, SLO를 깨도 SLA는 아직 안 깨진 "여유 구간"을 둡니다.

> 💡 **한 줄 요약**: SLI는 *측정값*(P99 180ms), SLO는 *목표*(P99<200ms를 99.9%), SLA는 *위반 시 보상하는 계약*입니다. 단계가 측정→목표→계약 순으로 올라갑니다.

---

## Q3. error budget가 뭔가요? "99.9%"라는 숫자는 어떻게 쓰는 건가요?

**error budget(오류 예산)**는 SLO를 뒤집어 본 개념입니다. SLO가 "99.9%는 성공해야 한다"면, error budget는 "그럼 0.1%까지는 실패해도 된다"는 *허용된 실패량*입니다.

```
SLO 99.9%  →  error budget = 100% - 99.9% = 0.1%

30일 = 43,200분 기준:
  허용 다운타임 = 43,200분 × 0.1% = 43.2분 / 월
                                    └─ 이게 한 달치 "예산"
```

이게 왜 유용하냐면, "완벽하게 무장애"라는 비현실적 목표 대신 **"예산 안에서는 마음껏 위험을 감수해도 된다"**는 운영 규칙을 줍니다.

```
✅ error budget가 남음 (이번 달 다운타임 5분 / 예산 43분)
   → 새 기능 배포해도 됨, 리스크 감수 가능

❌ error budget 소진 (이번 달 다운타임 50분 / 예산 43분 초과)
   → 신규 배포 동결, 안정화에 집중 (정책에 따라)
```

비유하자면 용돈입니다. 한 달 용돈(예산)이 남으면 군것질(위험한 배포)을 해도 되지만, 다 쓰면 다음 달까지 아껴야 합니다. error budget는 "안정성"과 "개발 속도"라는 충돌하는 두 가치를 *숫자로 중재*하는 장치입니다.

`platform_db` 맥락에서 가장 중요한 SLI 후보는 **인가 응답 지연**과 **entitlement 조회 가용성**입니다([[operability|operability O6]]). 인가가 느려지거나 실패하면 *전 서비스*가 영향을 받기 때문입니다(platform_db는 SPOF다 — [[operability|O5]]).

> 💡 **한 줄 요약**: error budget는 SLO를 뒤집은 "허용된 실패량"(99.9% → 월 43분)이고, 남으면 배포해도 되고 소진하면 안정화에 집중하라는 운영 규칙을 줍니다.

---

## Q4. P99 지연이 뭔가요? 왜 평균이 아니라 P99를 보나요?

**P99(99th percentile, 99분위)**는 "전체 요청을 느린 순서로 줄 세웠을 때, 빠른 쪽부터 99%가 들어오는 지점의 값"입니다. 즉 "100번 중 99번은 이 시간 안에 끝난다, 가장 느린 1번만 이보다 느리다"는 뜻입니다.

```
요청 1000건의 응답시간을 정렬:
  P50 (중앙값) = 50ms   ← 절반은 이보다 빠름
  P95          = 120ms  ← 95%는 이보다 빠름
  P99          = 180ms  ← 99%는 이보다 빠름 (느린 1%만 더 걸림)
  P100 (max)   = 4000ms ← 최악의 1건
```

**왜 평균이 아니라 P99인가?** 평균은 소수의 극단값을 가려버리기 때문입니다.

```
응답시간: [10ms × 99건, 5000ms × 1건]

평균 = (10×99 + 5000) / 100 = 59.9ms   ← "빠르네!" (착각)
P99  = 10ms, P100 = 5000ms             ← "1%는 5초나 걸림!" (진실)
```

평균만 보면 59ms로 멀쩡해 보이지만, 실제로는 100명 중 1명이 5초를 기다립니다. 트래픽이 많으면 이 "1명"이 매분 수백 명입니다. 그래서 사용자 체감 성능은 평균이 아니라 **꼬리(tail) 지연**, 즉 P95/P99/P99.9로 봅니다.

`platform_db`의 신뢰성 계약상 핵심 SLI는 **permission 평가 P99 · entitlement 조회 P99 · membership 조회 P99**입니다([[operability|O6]] 레이턴시 축). 이 조회들은 모든 게이트([[gate-abc-flow]])가 거치는 경로라, 여기 P99가 무너지면 전 서비스가 느려집니다.

> 💡 **한 줄 요약**: P99는 "100건 중 99건이 들어오는 응답시간"으로, 평균이 숨기는 *느린 꼬리*를 잡아내는 지표입니다. 사용자 체감 성능은 P99로 봅니다.

---

## Q5. 신뢰성 계약(O5, AVAIL-1)이 뭔가요? billing이 죽어도 권한이 유지된다는 게 무슨 뜻인가요?

**신뢰성 계약(reliability contract)**은 "무엇이 죽으면 무엇이 어떻게 영향받고, 무엇은 그래도 버티는가"를 *미리 명문화한 약속*입니다. 장애는 "정상의 반대"가 아니라 "정상 운영의 일부"라는 전제에서 출발합니다 — 의존성은 *언젠가 반드시* 죽기 때문입니다.

`platform_db`의 가장 중요한 신뢰성 계약이 **AVAIL-1**입니다([[requirements|requirements AVAIL-1]], [[operability|O5]]):

> **AVAIL-1**: billing/PG 장애 시 기존 entitlement read는 **영향 없이 최소 N시간 유지**된다.

무슨 뜻이냐면 — 결제 시스템(PG)이나 billing 파이프라인이 통째로 죽어도, *이미 권한을 가진 사용자는 계속 서비스를 쓸 수 있어야* 한다는 약속입니다. "결제 서버 다운 = 전 고객 서비스 정지"가 되면 안 됩니다.

**이게 왜 구조적으로 보장되나요?** 핵심은 [[auth-projection]]입니다. entitlement(런타임 권한)는 billing 원장과 **분리된 사영(projection)**으로 저장됩니다:

```
billing/PG (결제 발생) ──webhook──> [outbox] ──> org_entitlement (런타임 권위)
   │                                                    │
   │ 죽어도                                              │ Gate B는 여기만 읽음
   ▼                                                    ▼
 새 결제·갱신 멈춤                              기존 권한 read는 멀쩡 ✅
```

```typescript
// ✅ Gate B는 org_entitlement만 읽는다 (불변식 #4 — ledger 직접 조회 금지)
const ent = await getEntitlementByService(orgPk, service);
if (ent.status === "ACTIVE" || ent.status === "GRACE") return PASS;
// → 이 조회는 billing/PG와 무관. PG가 죽어도 이 read는 영향 없음.

// ❌ 만약 Gate B가 매번 PG에 "이 결제 유효해?"를 물어봤다면
// → PG 죽는 순간 전 고객 인가 실패 (이렇게 설계하지 않았다)
```

추가로 `validUntil` 복합 체크가 2차 방어입니다 — webhook이 지연/유실돼도 `valid_until` 기간 동안은 권한이 유지됩니다([[operability|O5]] 표). 즉 AVAIL-1은 *나중에 추가할 기능*이 아니라 **이미 설계에 내장된 격리**이고, O5는 그걸 "N시간 유지"라는 약속으로 명문화할 뿐입니다.

> 💡 **한 줄 요약**: 신뢰성 계약은 "무엇이 죽어도 무엇은 버틴다"는 명문화된 약속이고, AVAIL-1은 [[auth-projection]] 덕에 *billing/PG가 죽어도 기존 entitlement read는 영향 없다*는 구조적 격리를 약속합니다.

---

## Q6. 무엇을 보면 사고를 빨리 알 수 있나요? (관측 지표, O6)

장애는 "터진 다음"에 알면 늦습니다. **이상 신호를 먼저 보내주는 지표(leading indicator)**를 골라 감시해야 합니다. `platform_db`가 정의하는 핵심 관측 지표([[operability|O6]])는 다음과 같습니다.

**① DENY / ERROR 급증** — 가장 중요한 경고등입니다.

```
DENY 급증  → 공격(권한 스캐닝) 신호이거나, 배포로 인가 로직이 깨진 신호
ERROR 급증 → 코드 버그·의존성 장애 신호

→ "평소 분당 DENY 5건인데 갑자기 500건" = 즉시 알림
```

DENY는 *정상적인 거부*(권한 없는 사용자)일 수도 있지만, **급증**은 거의 항상 무언가 잘못됐다는 뜻입니다.

**② Gate별 통과/거부율** — 어느 게이트에서 막히는지를 봅니다.

```
GateA(membership) 실패율 ↑  → 로그인/멤버십 문제
GateB(entitlement) 실패율 ↑ → billing/결제 만료 문제 (webhook 지연?)
GateC(capability) 실패율 ↑  → 위임/권한 설정 문제
```

**③ break-glass 사용 횟수 — 평상시 0이어야 정상입니다.**

[[break-glass]]는 긴급 운영 접근(비상 권한)입니다. 정상 운영에서는 *절대 쓰일 일이 없어야* 하므로, 카운트가 0에서 벗어나는 순간 = 누군가 비상 권한을 썼다 = 즉시 사람이 확인해야 할 사건입니다.

```
break_glass 사용 count == 0   → ✅ 정상
break_glass 사용 count >  0   → 🚨 알림 (누가/왜/언제 — 전건 컴플라이언스 audit)
```

**④ 마지막 OWNER count = 0 인 org 알림** — org에 OWNER가 한 명도 없으면 그 조직은 *영구히 관리 불능*(lockout)이 됩니다([[requirements|OWN-1]]). 이건 사후 복구가 어려우므로 미리 감시합니다.

```sql
-- 마지막 OWNER가 사라진 org를 탐지 (감시 쿼리 예시)
SELECT org_pk, COUNT(*) AS owner_count
FROM membership
WHERE platform_role = 'OWNER' AND status = 'ACTIVE'
GROUP BY org_pk
HAVING owner_count = 0;   -- 0이면 알림
```

이 지표들의 공통점: **모두 "사고가 커지기 전에" 보내주는 신호**라는 것입니다. 사후 로그 분석이 아니라 사전 경고입니다.

> 💡 **한 줄 요약**: DENY/ERROR 급증(공격·버그), Gate별 거부율(어디서 막히나), break-glass 사용(0이어야 정상), 마지막 OWNER=0 org(lockout 예방) — 이 넷이 platform_db의 핵심 경고등입니다.

---

## Q7. 이 지표들은 audit_log에 쌓이나요? 2-lane이랑 어떻게 연결되나요?

여기서 [[audit-two-lane]] 결정과 만납니다. 권한 이벤트는 **2개 lane**으로 나뉩니다:

```
컴플라이언스 audit_log  │ 보안유의 이벤트만 100% · append-only · WORM · 5년
                        │ (DENY·ERROR·민감 ALLOW·break-glass·운영자 행위)
access 텔레메트리       │ 일상 read ALLOW 대량 · 샘플링 허용 · 단기 TTL(30~90일) · OLAP
                        │ (관측·지표 집계용, 컴플라이언스 증거 아님)
```

**관측은 주로 텔레메트리 lane에서 합니다.** 일상 read ALLOW가 초당 수천 건이라 전부 영구 보존하면 폭발합니다([[audit-two-lane]] 맥락). 그래서 지표 집계·대시보드는 샘플링 가능한 텔레메트리 lane(OLAP, 예: ClickHouse)에서 합니다.

**단, DENY/ERROR는 양쪽에 들어갑니다.** 보안유의 이벤트이므로:

```
DENY/ERROR 발생 시:
  → audit_log (컴플라이언스): 100% 영구 보존 (증거)
  → 텔레메트리 (관측): 급증 감시·알림 (속도)

일상 read ALLOW:
  → 텔레메트리만 (샘플링) — audit_log에 넣지 않음
```

비유하면, audit_log는 **블랙박스(법적 증거, 영구)**이고 텔레메트리는 **계기판 로그(운영 모니터링, 단기)**입니다. 사고 조사는 블랙박스로, 일상 운전 감시는 계기판으로 합니다. DENY/ERROR 같은 중대 사건만 둘 다에 기록됩니다.

> 💡 **한 줄 요약**: 관측은 주로 샘플링 가능한 텔레메트리 lane에서 하되, DENY/ERROR 같은 보안유의 이벤트는 audit_log(영구 증거)와 텔레메트리(급증 감시) 양쪽에 기록합니다.

---

## Q8. trace_id로 분산 추적을 한다는 게 뭔가요? (AUD-3)

요청 하나가 게이트 여러 개(A→B→C)와 여러 서비스·함수를 거치는데, 로그가 *제각각 흩어져* 있으면 "이 한 요청이 정확히 어디서 느렸나/막혔나"를 재구성하기 어렵습니다. **trace_id**는 한 요청에 붙는 고유 식별자로, 그 요청이 남기는 *모든* 로그·이벤트를 하나로 묶어줍니다([[requirements|AUD-3]]).

```
요청 진입 (trace_id = "abc-123" 발급)
  │
  ├─ GateA membership 조회   [trace_id=abc-123] 12ms
  ├─ GateB entitlement 조회  [trace_id=abc-123] 8ms
  ├─ GateC capability 평가   [trace_id=abc-123] 4ms   ← 여기서 DENY
  └─ audit_log INSERT        [trace_id=abc-123, audit_event_id=...]

→ "abc-123"으로 검색하면 이 요청의 전 여정이 한 줄로 모인다
```

이게 **분산 추적(distributed tracing)**입니다. 비유하면 택배 송장번호입니다. 송장번호 하나로 "집하 → 허브 → 배송"의 전 구간을 추적하듯, trace_id 하나로 요청의 전 경로를 추적합니다. 어느 구간(span)이 느렸는지, 어디서 멈췄는지가 한눈에 보입니다.

`platform_db`에서는 `audit_log`에 `trace_id`와 `audit_event_id`를 두어, 컴플라이언스 기록과 분산 추적을 이어붙입니다([[requirements|AUD-3]], 🟡 P1). "왜 이 사용자가 거부됐나"를 디버깅할 때, trace_id로 그 요청이 거친 게이트와 사유를 한 줄로 따라갈 수 있습니다. 이건 [[permission-debugger]]("이 유저가 왜 PUBLISH_VIDEO가 안 되죠?")와도 직결됩니다.

> 💡 **한 줄 요약**: trace_id는 한 요청에 붙는 송장번호로, 그 요청이 거친 GateA→B→C·서비스·로그를 전부 하나로 묶어 "어디서 느렸나/막혔나"를 추적하게 해줍니다(AUD-3).

---

## 용어 정리

| 용어 | 뜻 | 한 줄 메모 |
|---|---|---|
| 관측성 (observability) | 외부 신호만으로 내부 상태를 추론하는 능력 | 시스템에 계기판 달기 |
| metrics (지표) | 숫자 시계열 | 초당 DENY 수, P99 지연 — 집계·알림 |
| logs (로그) | 개별 사건의 기록 | 사후에 "무슨 일이었나" |
| traces (추적) | 한 요청이 거친 경로 | 어느 구간이 느렸나 |
| SLI | Service Level **Indicator** — 측정값 | "지금 P99 180ms" |
| SLO | Service Level **Objective** — 내부 목표 | "P99<200ms를 99.9%" |
| SLA | Service Level **Agreement** — 외부 계약 | "위반 시 요금 크레딧" |
| error budget | SLO를 뒤집은 허용 실패량 | 99.9% → 월 43분, 남으면 배포 가능 |
| P99 | 99분위 응답시간 | 100건 중 99건이 들어오는 시간 (꼬리 지연) |
| trace_id | 한 요청에 붙는 고유 식별자 | 택배 송장번호 — 전 경로 추적 |
| 신뢰성 계약 | 장애 시 영향·생존을 명문화한 약속 | AVAIL-1 = billing 죽어도 entitlement read 유지 |
| 알림 임계치 (alert threshold) | 알림을 발화시키는 기준값 | "DENY 분당 100건 초과 시" — *튜닝은 ops 제품 소관* |
| leading indicator | 사고가 커지기 전 미리 보내는 신호 | DENY 급증, break-glass 사용 |

---

## 테스트 방법
> 🧪 *실제 DB·ORM·운영에서 돌리는 법*: [[testing-strategy]] · [[orm-testing-drizzle]]


관측·신뢰성은 "장애를 일부러 일으켜서 신호가 제대로 뜨는가"를 검증합니다. 핵심은 **장애 주입(fault injection)**과 **임계치 모킹**입니다.

**① DENY율 급증 시 알림 발화 (임계치 모킹)**

```typescript
test("DENY가 임계치를 넘으면 알림이 발화된다", async () => {
  const alerter = mockAlerter();
  const monitor = new GateMonitor({ denyThresholdPerMin: 100, alerter });

  // 정상 범위 — 알림 없음
  monitor.recordDeny(50);
  expect(alerter.fired).toBe(false);

  // 임계치 초과 — 알림 발화
  monitor.recordDeny(150);
  expect(alerter.fired).toBe(true);
  expect(alerter.lastAlert.kind).toBe("DENY_SPIKE");
});
```

**② P99 지연 SLI 측정**

```typescript
test("permission 평가 P99를 정확히 계산한다", () => {
  // 99건 빠름 + 1건 느림
  const samples = [...Array(99).fill(10), 5000]; // ms
  const p99 = percentile(samples, 99);

  expect(p99).toBeGreaterThan(10);   // 평균(59.9)에 안 묻힌다
  expect(p99).toBeLessThanOrEqual(5000);
  // SLO 검증: P99 < 200ms 여야 통과
  expect(p99 < 200).toBe(false);     // 이 샘플은 SLO 위반 — 알림 대상
});
```

**③ break-glass 발생 시 모니터 카운트 증가**

```typescript
test("break-glass를 쓰면 모니터 카운트가 0→1로 오르고 알림", async () => {
  const monitor = new BreakGlassMonitor();
  expect(monitor.count).toBe(0);          // 평상시 0이 정상

  await useBreakGlass({ operator: "sre-1", reason: "incident-4821" });

  expect(monitor.count).toBe(1);
  expect(monitor.lastAlert.kind).toBe("BREAK_GLASS_USED"); // 0 이탈 즉시 알림
  // 컴플라이언스 audit_log에도 전건 기록됐는지 확인
});
```

**④ billing 장애 주입 후에도 entitlement read 정상 (AVAIL-1)** — 가장 중요한 신뢰성 계약 테스트입니다.

```typescript
test("PG/billing이 죽어도 기존 entitlement read는 정상 (AVAIL-1)", async () => {
  // 기존 권한이 있는 org
  await seedEntitlement({ orgPk: 1, service: "ACADEMY", status: "ACTIVE" });

  // billing/PG를 통째로 다운시킨다 (장애 주입)
  pgGateway.injectFault("CONNECTION_REFUSED");

  // Gate B는 org_entitlement만 읽으므로 영향 없어야 한다
  const result = await checkGateB(1, "ACADEMY");
  expect(result).toBe("PASS");   // ✅ PG 죽어도 통과

  // 반대 검증: Gate B가 PG를 직접 호출하지 않았음을 보장
  expect(pgGateway.callCount).toBe(0);
});
```

**체크리스트**

```
□ DENY/ERROR 급증 시 알림이 임계치에서 정확히 발화하는가
□ P99 SLI가 평균에 묻히지 않고 꼬리 지연을 잡아내는가
□ break-glass 사용 시 카운트 0→증가 + 알림 + audit_log 전건 기록
□ 마지막 OWNER=0 org 탐지 쿼리가 동작하는가 (OWN-1)
□ billing/PG 장애 주입 후 entitlement read가 영향 없는가 (AVAIL-1)
□ Gate B가 PG/ledger를 직접 호출하지 않는가 (불변식 #4 — auth-projection)
□ trace_id가 한 요청의 GateA→B→C 로그를 하나로 묶는가 (AUD-3)
□ 일상 read ALLOW가 audit_log가 아닌 텔레메트리 lane으로 가는가 (2-lane)
```

---

## 마치며

관측성과 신뢰성은 결국 두 개의 질문으로 귀결됩니다:

1. **"장애가 났는지 어떻게 아는가?"** → 관측성(metrics·logs·traces) + 경고등(DENY/ERROR 급증, break-glass, 마지막 OWNER=0)
2. **"무엇이 죽어도 무엇은 버틴다고 약속할 수 있는가?"** → 신뢰성 계약(AVAIL-1: billing 죽어도 entitlement read 유지)

이 둘이 중요한 이유는 `platform_db`가 **SPOF**이기 때문입니다([[operability|O5]]) — 인가가 느려지거나 틀리면 *전 서비스*가 영향받습니다. 그래서 platform은 "무엇을 봐야 하나(O6)"와 "무엇을 약속하나(O5)"를 *데이터·계약으로* 소유합니다.

**범위 규율 — 헷갈리기 쉬운 부분**: platform_db는 관측을 *가능하게(enable)* 하지만, 다음은 별도 **ops 제품**이 소유합니다([[requirements|requirements §4 주석]]):

```
platform_db 책임          │ ops 제품 소유 (platform은 enable만)
──────────────────────────┼──────────────────────────────────
관측 지표 정의 (무엇을 보나) │ 콘솔 UI (대시보드 화면)
신뢰성 계약 (AVAIL-1)      │ 알림 임계치 튜닝 ("분당 몇 건?")
측정 가능하도록 데이터 노출  │ DR 런북 / RTO·RPO
trace_id·audit 스키마      │ Feature Flag rollout
```

즉 "DENY를 측정 가능하게 노출"하는 건 platform 책임이지만, "DENY 분당 100건에서 알림"이라는 *임계치 숫자*와 *알림 화면*은 ops 제품 몫입니다. 새 관측 기능을 만들 때 "이건 platform이 소유하는 지표·계약인가, 아니면 ops가 소비하는 UI·임계치인가?"를 먼저 구분하세요.

---

## 연결된 개념

- [[operability|운영 가능성 O5·O6]] — 이 문서의 모태(신뢰성 계약·관측 지표 정의)
- [[auth-projection|entitlement 사영]] — AVAIL-1이 구조적으로 보장되는 이유 (billing↔entitlement 격리)
- [[audit-two-lane|감사 로그 2-lane]] — 관측(텔레메트리)과 컴플라이언스(audit_log)의 분리, DENY/ERROR는 양쪽
- [[gate-abc-flow|Gate A/B/C 전체 흐름]] — P99로 감시하는 인가 경로, trace_id가 묶는 대상
- [[permission-debugger|Permission Debugger]] — trace_id로 "왜 막혔나"를 진단하는 도구
- [[break-glass|Break-glass 긴급 접근]] — 사용 카운트 0이 정상인 핵심 경고등
- [[fail-closed|Fail-closed 원칙]] — 불확실하면 거부, 그 DENY가 관측 신호가 되는 맥락
> 소스 문서
- [[operability]] — O5 신뢰성 계약(AVAIL-1), O6 관측성(KPI)
- [[requirements]] — AVAIL-1(가용성 계약), AUD-2/3(2-lane·trace_id), §4 운영 가능성 추적표 + 범위 규율 주석
- [[audit-two-lane]] — 컴플라이언스 audit vs access 텔레메트리 분리 결정
