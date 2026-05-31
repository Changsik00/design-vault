---
difficulty: 중
tags:
  - platform-db
  - explainer
  - p1
  - gate
  - billing
  - grace
  - design-decision
aliases:
  - Gate B 유예
  - validUntil
  - GRACE 기간
  - 배치 실패 안전망
---

# Gate B 유예 기간 설계 결정 설명 (status + validUntil 복합 체크)

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[decisions/gate-b-billing-grace|gate-b-billing-grace.md]] · [[schema-reference|schema-reference.md §E.2]] · [[architecture|architecture.md §2.1 불변식 #9]]

[[gate-b-entitlement|Gate B]]는 "이 학원이 지금 서비스를 이용할 수 있나요?"를 확인하는 관문입니다. 그런데 코드를 보면 `status` 하나만 확인하는 게 아니라 `validUntil`까지 같이 체크합니다. 왜 두 개를 다 확인할까요? 이 문서가 그 이유를 설명합니다.

---

## Q1. Gate B에서 `status`와 `validUntil`을 왜 둘 다 확인하나요? status 하나만 보면 안 되나요?

결론부터: **배치(결제 갱신 작업)가 실패했을 때를 대비한 2차 방어선**이기 때문입니다.

현재 구현을 보면:

```typescript
// 현재 구현 — status + validUntil 복합 체크
function isEntitlementValid(e: OrgEntitlement | null): boolean {
  if (e === null) return false;
  const statusOk = e.status === "ACTIVE" || e.status === "GRACE";
  const notExpired = e.validUntil === null || e.validUntil > new Date();
  return statusOk && notExpired;  // 둘 다 true여야 통과
}
```

`status`만 체크하는 방식도 있었습니다:

```typescript
// status만 체크하는 방식 (채택 안 함)
function isEntitlementValid(e: OrgEntitlement | null): boolean {
  if (e === null) return false;
  return e.status === "ACTIVE" || e.status === "GRACE";
}
```

두 방식의 차이가 드러나는 시나리오가 있습니다. 배치가 실패해서 `status`는 여전히 `ACTIVE`인데 `validUntil`은 이미 과거 날짜라면:

```
status = "ACTIVE"       ← 배치가 갱신 못 한 상태
validUntil = 2026-04-30 ← 이미 지난 날짜 (오늘이 5월 28일이면)

status-only:   ACTIVE니까 통과 → 영구 무료 접근 가능 ❌
복합 체크:     validUntil이 과거 → 차단                ✅
```

| 상태 | status-only | 복합 체크 |
|---|---|---|
| status=ACTIVE, validUntil=과거 | 통과 (영구 무료 위험) | 차단 |
| status=ACTIVE, validUntil=미래 | 통과 | 통과 |
| status=GRACE, validUntil=과거 | 통과 | 차단 |
| status=EXPIRED, validUntil=미래 | 차단 | 차단 |

`validUntil`은 배치가 `status`를 제때 갱신하지 못했을 때도 최소한 만료일 이후 접근을 막아주는 안전망입니다.

불변식 #9에 이것이 명시되어 있습니다:

```
불변식 #9: Gate B는 status + valid_until 복합 체크.
           status만 보면 배치 실패 시 영구 무료 위험.
```

> 💡 **한 줄 요약**: 배치가 실패해서 `status`가 `ACTIVE`에 묶여 있어도, `validUntil`이 만료되면 접근을 차단합니다.

---

## Q2. "배치(batch)"가 뭔가요? 결제랑 어떤 관계인가요?

**배치**는 정해진 시간에 주기적으로 실행하는 자동화 작업입니다. 여기서는 결제 갱신과 관련된 두 가지 역할을 합니다.

**1. [[subscription-lifecycle|구독 상태]] 만료 처리 배치**

매일 자정 근처에 실행되어, 기간이 지난 구독을 `EXPIRED`로 바꿉니다:

```sql
-- 매일 1회 실행 (architecture.md §4 정기 운영)
UPDATE org_entitlement
SET status = 'EXPIRED'
WHERE valid_until < NOW()
  AND status = 'ACTIVE';
```

**2. 구독 자동 갱신 배치 (결제 재시도)**

월/연 구독의 경우, 구독 기간이 끝나기 전에 PG(결제 게이트웨이)에 자동 청구를 요청합니다. 청구가 성공하면 `valid_until`을 다음 기간으로 연장하고, 실패하면 GRACE 처리합니다.

```
결제 성공 흐름:
PG 청구 → 성공 → webhook 수신 → 단일 트랜잭션 처리
                                   ├── payment_ledger INSERT
                                   ├── org_subscription UPDATE
                                   ├── org_entitlement UPSERT (status=ACTIVE, valid_until 연장)
                                   └── perm_version 갱신

결제 실패 흐름:
PG 청구 → 실패 → 재시도 (3회) → 전부 실패 → status=GRACE 전환
```

배치와 Gate B의 관계:

```
배치가 정상이라면:
  valid_until 만료 전 → status=ACTIVE, valid_until 갱신
  → Gate B: ACTIVE + 미래 날짜 → 통과

배치가 실패했다면:
  valid_until은 과거 → status는 여전히 ACTIVE
  → Gate B: ACTIVE + 과거 날짜 → 차단 (validUntil 안전망 동작)
```

> 💡 **한 줄 요약**: 배치는 구독 상태를 주기적으로 동기화하는 자동 작업입니다. Gate B는 이 배치가 실패했을 경우에도 잘못된 접근을 막아야 합니다.

---

## Q3. 배치가 실패하면 어떤 일이 생기나요?

배치 실패 시나리오를 단계별로 따라가 보겠습니다.

```
타임라인 시나리오:
────────────────────────────────────────────────────────
5월 1일  학원이 월 구독 결제 (valid_until = 5월 31일)
5월 31일 자동 갱신 시도 → PG 서버 장애로 배치 실패!
         org_entitlement.status = ACTIVE (그대로)
         org_entitlement.valid_until = 2026-05-31 23:59:59 (그대로)
6월 1일  valid_until 과거가 됨
         배치 실패로 status는 아직 ACTIVE

6월 1일 오전 9시, 학원 원장이 로그인 시도
         Gate B 체크: status=ACTIVE ✅ AND validUntil < NOW() ❌
         → 접근 차단 (복합 체크 안전망 작동)
         → 학원 원장에게 "구독 갱신 필요" 안내
────────────────────────────────────────────────────────
```

**status-only 체크였다면:**

```
6월 1일 오전 9시, 학원 원장이 로그인 시도
         Gate B 체크: status=ACTIVE ✅
         → 통과! (영구 무료 접근 가능)
         → 6월 한 달 내내 결제 없이 이용
         → 월 매출 손실 발생
```

이것이 불변식 #9가 "status만 보면 배치 실패 시 영구 무료 위험"이라고 명시한 이유입니다.

배치 실패는 현실에서 충분히 발생할 수 있습니다:
- 서버 재시작 타이밍과 겹침
- PG API 일시 장애
- DB 연결 타임아웃
- 코드 배포 중 배치 스킵

이런 상황에서 `validUntil`이 "최소한 이 날짜 이후엔 쓸 수 없어"를 보장해주는 최후 방어선입니다.

> 💡 **한 줄 요약**: 배치가 실패하면 status는 ACTIVE에 묶이지만, validUntil이 과거 날짜가 되면서 Gate B가 접근을 차단합니다. 영구 무료 접근을 막습니다.

---

## Q4. `validUntil = null`이면 어떻게 처리되나요?

`null`은 "무기한(indefinite)"을 의미합니다. 이 경우 `notExpired = true`로 처리해서 차단하지 않습니다.

```typescript
const notExpired = e.validUntil === null || e.validUntil > new Date();
//                              ↑
//                         null이면 무기한 → 통과
```

`validUntil = null`이 쓰이는 경우:

| 경우 | 설명 |
|---|---|
| 수동 부여 영구 구독 | 관리자가 파일럿 학원에 무기한 무료 이용권 부여 |
| PROMO 계정 | 특정 프로모션 기간 중 무제한 접근 |
| FREE 플랜 | 유효기간이 없는 무료 티어 |

```sql
-- 수동 부여 영구 구독 예시
INSERT INTO org_entitlement (org_pk, product_code, service, status, source, valid_until)
VALUES (42, 'ACADEMY_PRO', 'ACADEMY', 'ACTIVE', 'MANUAL', NULL);
--                                                          ↑↑↑↑
--                                                    null = 무기한
```

이때 Gate B 동작:

```typescript
const e = { status: "ACTIVE", validUntil: null };

const statusOk = e.status === "ACTIVE";        // true
const notExpired = e.validUntil === null;       // null이므로 true
return statusOk && notExpired;                  // true → 통과
```

주의할 점: `source='MANUAL'`이나 `source='PROMO'`인 entitlement에서 `validUntil=null`이 아닌 경우는 일반 로직과 동일하게 처리됩니다. `null`은 특별한 경우를 위한 명시적 "무기한" 표시입니다.

> 💡 **한 줄 요약**: `validUntil = null`은 "만료 없음"을 의미하며, Gate B는 이 경우 시간 체크를 건너뛰고 status만 봅니다.

---

## Q5. GRACE 상태가 뭔가요? 결제 실패하면 바로 서비스를 끊어야 하는 거 아닌가요?

바로 끊으면 안 됩니다. 그 이유를 생각해보면: 결제 실패에는 여러 원인이 있습니다.

```
결제 실패 원인:
  - 카드 한도 초과 (다음 날 결제 시도하면 성공할 수도)
  - 카드 분실 후 재발급 중 (1~3 영업일 소요)
  - 은행 서버 일시 장애 (몇 시간 후 자동 재시도)
  - 단순 실수 (카드번호 만료일 갱신 깜빡)
```

이런 경우에 서비스를 즉시 끊으면:
- 오랫동안 충성 고객이었던 학원이 하루아침에 서비스 차단
- 학원 수업 중 갑자기 자료 접근 불가 → 최악의 사용자 경험
- 재결제 후에도 "다시 차단될 수 있다"는 불신 → 이탈

그래서 **유예 기간(Grace Period)**을 줍니다:

```
상태 머신:

결제 성공 ──▶ ACTIVE (정상 서비스)
               │
               │ 결제 실패
               ▼
            PAST_DUE (재시도 중)
               │
               │ 재시도 모두 실패
               ▼
             GRACE (유예 기간, grace_until까지)
               │
               │ 유예 기간에 결제 성공   │ 유예 기간 만료
               ▼                        ▼
            ACTIVE                   EXPIRED (서비스 차단)
```

GRACE 기간 동안 Gate B는 여전히 통과시킵니다:

```typescript
const statusOk = e.status === "ACTIVE" || e.status === "GRACE";
//                                         ↑↑↑↑↑
//                              GRACE도 서비스 이용 가능
```

단, GRACE 상태에서는 사용자에게 "결제 실패 배너"를 보여줍니다:

```
org_entitlement.status 별 사용자 경험 (schema-reference.md §F.3):

ACTIVE  → 정상 (배너 없음)
GRACE   → 통과 + "결제 실패 — X일 내 갱신 필요" 배너 표시
SUSPENDED → 차단 (402)
EXPIRED   → 차단 (402)
```

실제 SaaS 서비스(Stripe, AWS 등)도 동일한 방식을 씁니다. 보통 7~14일의 유예 기간을 주고, 그 안에 결제가 성공하면 정상 전환합니다.

> 💡 **한 줄 요약**: 결제 실패는 다양한 이유가 있고, 즉시 차단은 최악의 UX입니다. GRACE 기간을 주어 재결제 기회를 제공하되, 배너로 안내합니다.

---

## Q6. 나중에 status만 체크하는 방식으로 단순화할 수 있다고 하는데, 언제 가능한가요?

`gate-b-billing-grace.md`에는 단순화 조건이 명시되어 있습니다. 아래 세 가지가 모두 충족되면 재검토할 수 있습니다:

```
단순화 조건 (모두 충족 시):

1. 배치(청구 주기 갱신 잡)가 SLA 내 실패 시 알람 + 자동 재시도 보장
   → "배치 실패를 모를 수 없다"는 감지 시스템 완비

2. status와 valid_until의 불일치를 탐지하는 정합성 모니터링 구축
   → 예: "status=ACTIVE인데 valid_until이 3일 이상 과거인 row를 알람"

3. 배치 실패가 3개월간 0건이었음을 운영에서 확인
   → 실증된 신뢰성
```

왜 이 조건들인지 생각해보면:

```
현재 상황:                    단순화 가능한 상황:

배치 실패 가능성 있음          배치가 SLA 내 항상 재시도됨
실패 감지 시스템 없음          불일치 즉시 알람 옴
validUntil = 안전망 필요       validUntil = 배치가 항상 동기화
                               → status만으로 신뢰 가능
```

단순화가 가능해지면 이렇게 바뀝니다:

```typescript
// 미래 단순화 버전 (조건 충족 후)
function isEntitlementValid(e: OrgEntitlement | null): boolean {
  if (e === null) return false;
  return e.status === "ACTIVE" || e.status === "GRACE";
  // validUntil 체크 제거 — 배치가 항상 status를 정확히 반영하므로
}
```

현재로서는 그 조건이 아직 갖춰지지 않았습니다. 배치 실패 감지 알람도, 불일치 모니터링도 아직 P1 수준으로 구현 중입니다. 그래서 지금은 복합 체크를 유지합니다.

**언제 단순화하면 안 되나요?**

- 배치 SLA 알람이 없는 상태에서 단순화 → 배치 실패 며칠 지나도 모름 → 영구 무료
- 불일치 모니터링 없이 단순화 → 정합성 깨져도 발견 못함
- 운영 실적 없이 단순화 → 근거 없는 신뢰

> 💡 **한 줄 요약**: 배치 알람 + 불일치 모니터링 + 3개월 무장애 실적 이 세 가지가 갖춰지면 status-only로 단순화를 검토할 수 있습니다.

---

## 마치며

Gate B의 `status + validUntil` 복합 체크는 "왜 이렇게 복잡하게 짰지?"가 아니라 "배치가 실패해도 영구 무료가 되면 안 된다"는 비즈니스 요구사항에서 나온 결정입니다.

세 개념의 연결 관계를 정리하면:

```
결제 성공
    │ 단일 트랜잭션
    ▼
org_entitlement 갱신 (status=ACTIVE, validUntil=다음 기간)
    │
    │ 매일 배치
    ▼
만료 전 자동 재청구 → 성공 시 validUntil 연장
                  → 실패 시 GRACE 전환
    │
    │ 매 요청마다
    ▼
Gate B 체크: status ∈ {ACTIVE, GRACE} AND (validUntil IS NULL OR validUntil > NOW())
    │
    ├─ 통과 → 서비스 이용
    └─ 차단 → 402 결제 필요
```

코드에서 `isEntitlementValid()`를 수정하거나 Gate B 로직을 변경하려 할 때, 반드시 다음 표의 4가지 케이스를 모두 테스트하세요:

```
| status=ACTIVE, validUntil=과거  | → 차단이어야 함 (배치 실패 안전망)
| status=ACTIVE, validUntil=미래  | → 통과이어야 함
| status=GRACE,  validUntil=과거  | → 차단이어야 함
| status=EXPIRED, validUntil=미래 | → 차단이어야 함
```

`packages/db-platform/src/gates.test.ts`에 이 케이스들이 단위 테스트 18건으로 존재합니다. 새로운 변경사항도 이 테스트를 통과해야 합니다.

---

## 연결된 개념

- [[gate-b-entitlement|Gate B & 엔타이틀먼트]] — org_entitlement 테이블과 status 값의 의미
- [[subscription-lifecycle|구독 상태 머신]] — PAST_DUE에서 GRACE로 전이되는 흐름
- [[index-design|인덱스 설계]] — idx_org_service_status에 valid_until이 포함된 이유
> 소스 문서
- [[decisions/gate-b-billing-grace]] — 원본 설계 결정 문서 (이 explainer의 원천)
- [[architecture]] — §2.1 불변식 #9 (Gate B 복합 체크)
- [[schema-reference]] — D.12 org_entitlement DDL, E.2 Gate B 구현 코드
