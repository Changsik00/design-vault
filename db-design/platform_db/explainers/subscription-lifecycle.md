---
tags:
  - platform-db
  - explainer
  - p1
  - billing
  - subscription
  - state-machine
aliases:
  - 구독 상태 머신
  - TRIALING
  - ACTIVE
  - CANCELED
  - EXPIRED
  - org_subscription
---

# 구독 상태 머신 설명 (TRIALING → ACTIVE → CANCELED → EXPIRED)

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[architecture]] §6, [[schema-reference]] §D.12~D.13, §F.2~F.3, [[explainers/gate-b-billing-grace|Gate B 유예 기간 설계]]

SaaS 구독에는 단순히 "결제됨/안됨" 이상의 상태가 있습니다. 무료 체험, 결제 실패, 취소 후 만료까지 — 각 상태가 사용자 경험과 직결됩니다. `platform_db`는 이 흐름을 두 개의 테이블(`org_subscription`, [[gate-b-entitlement|org_entitlement]])로 나눠서 관리합니다.

---

## Q1. org_subscription.status 값들(TRIALING/ACTIVE/PAST_DUE/CANCELED/EXPIRED)이 각각 어떤 상태인가요?

`org_subscription.status`는 **"결제 관계의 상태"**를 나타냅니다. 쉽게 말하면 PG(결제사)와의 계약 상태입니다.

```
TRIALING  — 무료 체험 중. 아직 카드를 긁지 않음.
ACTIVE    — 정상 결제 완료. 구독이 활성 상태.
PAST_DUE  — 결제 시도했는데 실패함. 아직 완전히 끊지는 않고 유예 기간 부여.
CANCELED  — 사용자가 직접 취소 요청. 그러나 기간 종료 전까지는 쓸 수 있음.
EXPIRED   — 완전히 끝남. 더 이상 서비스 접근 불가.
```

각 상태를 실생활로 비유하면:

| 상태 | 비유 | 서비스 접근 |
|---|---|---|
| TRIALING | 헬스장 1개월 무료 체험권 | 가능 |
| ACTIVE | 헬스장 월회원 정상 납부 중 | 가능 |
| PAST_DUE | 이번 달 회비 미납, 아직 잠금은 안 됨 | 가능 (유예 기간) |
| CANCELED | 해지 신청했지만 이번 달 말까지는 이용 가능 | 기간 내 가능 |
| EXPIRED | 회원권 완전 만료 | 불가 |

중요한 점은 `org_subscription.status`가 PAST_DUE라고 해서 **즉시 서비스가 차단되지는 않는다**는 것입니다. 실제 접근 제어는 `org_entitlement.status`가 담당합니다 (Q5 참고).

> 💡 **한 줄 요약**: `org_subscription.status`는 PG와의 결제 계약 상태, `PAST_DUE`는 "결제 실패했지만 아직 완전히 끊지 않은" 중간 단계입니다.

---

## Q2. TRIALING에서 ACTIVE로 넘어가는 과정이 어떻게 되나요?

무료 체험이 끝나면 등록된 카드로 자동 결제가 시도됩니다. 이 과정은 PG(결제사 — Toss, Stripe 등)와 `platform_db` 사이의 협력으로 일어납니다:

```
1. 체험 만료 시점 도달 (trial_ends_at < NOW())

2. PG가 등록된 카드로 자동 결제 시도

3-A. 결제 성공 → PG가 webhook 전송
   └── pg_webhook_event에 event_id로 기록 (중복 방지)
   └── 단일 트랜잭션:
       ├── payment_ledger에 CHARGE 기록
       ├── org_subscription.status → 'ACTIVE'
       ├── org_entitlement.status → 'ACTIVE', valid_until 갱신
       ├── organization.perm_version 증가
       └── outbox_event에 'subscription.activated' 삽입
   └── 사용자 입장: 아무 변화 없이 그냥 계속 씀

3-B. 결제 실패 → PG가 실패 webhook 전송
   └── org_subscription.status → 'PAST_DUE'
   └── org_entitlement.status는 아직 유지 (grace 기간 부여)
   └── 사용자 입장: "결제 실패, X일 내 갱신 필요" 배너 표시
```

코드에서는 이렇게 단일 트랜잭션으로 처리됩니다. 결제 원장에는 [[idempotency-key|멱등성]] 키(`idempotency_key`)를 함께 기록하고, [[feature-limits|feature_limits]]도 함께 갱신합니다:

```sql
BEGIN;
  -- 결제 원장 기록
  INSERT INTO payment_ledger (org_pk, type, amount_minor, idempotency_key, status)
  VALUES (?, 'CHARGE', ?, ?, 'SUCCEEDED');

  -- 구독 상태 → ACTIVE
  UPDATE org_subscription
  SET status = 'ACTIVE',
      current_period_start = ?,
      current_period_end = ?
  WHERE pk = ?;

  -- 권한 활성화 (upsert)
  INSERT INTO org_entitlement (org_pk, service, status, valid_until, feature_limits)
  VALUES (?, 'ACADEMY', 'ACTIVE', ?, ?)
  ON DUPLICATE KEY UPDATE
    status = 'ACTIVE',
    valid_until = VALUES(valid_until),
    feature_limits = VALUES(feature_limits);

  -- 클라이언트 캐시 무효화 신호
  UPDATE organization SET perm_version = perm_version + 1 WHERE pk = ?;

  -- 이메일/알림 등 비동기 처리용 이벤트
  INSERT INTO outbox_event (event_type, payload_json)
  VALUES ('subscription.activated', ?);
COMMIT;
```

이 모든 게 **하나의 트랜잭션**이라서 "결제는 됐는데 서비스 접근이 안 된다"는 상황이 구조적으로 발생할 수 없습니다. 이메일·알림 등 비동기 fan-out은 [[outbox-pattern|Outbox 패턴]](`outbox_event` 테이블)으로 처리합니다.

> 💡 **한 줄 요약**: TRIALING → ACTIVE는 [[webhook-processing|PG 웹훅]]이 트리거가 되고, 결제 기록 + 구독 상태 + 권한 활성화를 단일 트랜잭션으로 처리합니다.

---

## Q3. 결제 실패하면 바로 EXPIRED가 되나요? PAST_DUE와 EXPIRED 차이가 뭔가요?

결제 실패 시 **즉시 EXPIRED가 되지 않습니다.** SaaS의 표준 패턴은 이렇습니다:

```
결제 실패 발생
    ↓
PAST_DUE (일단 유예)
    ├── PG가 자동 재시도 (보통 3~5회, 며칠 간격)
    │     ├── 재시도 성공 → ACTIVE 복구
    │     └── 모두 실패 → [[explainers/gate-b-billing-grace|유예 기간]] 시작
    ↓
유예 기간 (grace period) 부여
    │   - org_subscription.grace_until 시각까지
    │   - org_entitlement.status → 'GRACE' (서비스 접근 가능)
    │   - 사용자에게 "결제 수단 업데이트" 배너 표시
    ↓
유예 기간 만료
    ↓
org_entitlement.status → 'EXPIRED' (서비스 차단)
org_subscription.status → 'EXPIRED'
```

**왜 즉시 차단하지 않냐고요?**

신용카드 한도 초과나 일시적 오류로 결제가 실패하는 경우가 의외로 많습니다. 결제 실패 즉시 서비스를 막으면 정상 고객을 잃게 됩니다. 유예 기간을 주면 "카드 업데이트하면 계속 쓸 수 있어"라는 메시지를 전달할 수 있습니다.

두 상태의 차이를 명확히 정리하면:

| 구분 | PAST_DUE | EXPIRED |
|---|---|---|
| 위치 | org_subscription.status | 둘 다 |
| 의미 | 결제 실패, 유예 중 | 완전히 끝남 |
| 서비스 접근 | GRACE 기간 내 가능 | 불가 |
| 복구 가능? | 카드 갱신 후 재결제로 복구 | 재구독 필요 |

> 💡 **한 줄 요약**: 결제 실패 → PAST_DUE(유예 시작) → 유예 만료 → EXPIRED(차단) 순서입니다. 즉시 차단은 UX 관점에서 최악이라 SaaS에서는 유예 기간이 표준입니다.

---

## Q4. 사용자가 "취소"를 눌렀을 때 CANCELED가 되면 바로 서비스가 막히나요?

아닙니다. CANCELED가 되더라도 **기간이 끝날 때까지는 서비스를 계속 이용할 수 있습니다.**

이건 Stripe, Netflix 등 모든 SaaS의 표준 동작입니다. 이미 결제한 기간의 서비스를 제공하는 것이 당연하기 때문입니다.

```
사용자가 5월 15일에 "취소" 클릭
    ↓
org_subscription.status → 'CANCELED'
org_subscription.cancelled_at → 2026-05-15T...
org_subscription.current_period_end → 2026-05-31T... (이미 정해져 있음)
    ↓
5월 31일까지:
  - org_entitlement.status는 여전히 'ACTIVE'
  - 서비스 정상 이용 가능
  - "6월 1일에 구독이 종료됩니다" 배너 표시
    ↓
6월 1일 (current_period_end 도달):
  - 배치 스크립트 실행
  - org_entitlement.status → 'EXPIRED'
  - 서비스 접근 차단
```

이 동작을 구현하는 배치 쿼리:

```sql
-- 매일 실행되는 배치 (architecture.md §12.11)
UPDATE org_entitlement
SET status = 'EXPIRED'
WHERE valid_until < NOW()
  AND status IN ('ACTIVE', 'GRACE');
```

CANCELED 상태의 구독은 `current_period_end`가 `valid_until`로 반영되어 있어서, 배치가 실행되면 자동으로 EXPIRED로 전환됩니다.

**주의**: 만약 배치가 실패하면 EXPIRED 전환이 안 되고 서비스가 무한정 유지될 위험이 있습니다. 이게 바로 Gate B에서 `status`만 체크하지 않고 `valid_until`도 함께 확인하는 이유입니다 (불변식 #9).

> 💡 **한 줄 요약**: CANCELED는 "이 구독은 갱신 안 한다"는 선언이지, 즉시 차단이 아닙니다. 현재 결제된 기간이 끝날 때 비로소 EXPIRED가 됩니다.

---

## Q5. org_subscription.status와 org_entitlement.status가 따로 있는 이유가 뭔가요?

이게 처음에는 헷갈리는 부분입니다. 왜 같은 것처럼 보이는 상태를 두 테이블에 따로 관리할까요?

두 테이블은 **역할이 완전히 다릅니다**:

```
org_subscription  ← "결제 계약서"
  - PG와의 계약 상태를 추적
  - 인보이스, 재시도 횟수, grace_until 빌링 추적 등 복잡한 billing 정보
  - "이 구독의 PG 측 상태는?"에 답하는 테이블

org_entitlement  ← "서비스 이용권 티켓"
  - "지금 이 org가 서비스를 쓸 수 있나?"에만 답하는 테이블
  - Gate B가 유일하게 읽는 테이블
  - payment_ledger 직접 조회 금지 (불변식 #4)
```

비유하자면:

```
org_subscription = 넷플릭스 계정의 결제 정보 페이지
  (카드 정보, 청구 이력, 다음 결제일, 실패 횟수...)

org_entitlement = 실제로 넷플릭스를 켰을 때 "재생 가능" 여부
  (지금 이 계정으로 재생할 수 있나? 한 줄 답변만 필요)
```

분리한 이유:

**1. 속도**: Gate B는 모든 API 요청마다 호출됩니다. `org_subscription`에는 복잡한 컬럼이 많지만, Gate B는 "ACTIVE인가?" 하나만 알면 됩니다. `org_entitlement`는 이 단순한 쿼리에 최적화되어 있습니다.

**2. 격리**: PG 시스템에 장애가 나도 `org_entitlement`는 정상 동작합니다. 결제사 API가 느려도 서비스 접근 판단은 빠릅니다.

**3. 유연성**: PROMO(무료 쿠폰), MANUAL(어드민 수동 활성화) 같은 결제 없는 entitlement도 표현할 수 있습니다.

```sql
-- 결제 없이 수동으로 30일 무료 이용권 부여 가능
INSERT INTO org_entitlement (org_pk, service, status, source, valid_until)
VALUES (?, 'ACADEMY', 'ACTIVE', 'PROMO', DATE_ADD(NOW(), INTERVAL 30 DAY));
-- org_subscription 없이도 가능
```

> 💡 **한 줄 요약**: `org_subscription`은 PG와의 복잡한 결제 계약 전체, `org_entitlement`는 "지금 쓸 수 있나?"만 담당합니다. 권한 판단은 항상 `org_entitlement`만 봅니다.

---

## Q6. 배치(batch)가 상태를 바꾸지 못했을 때 어떻게 복구하나요?

배치는 매일 실행되지만 실패할 수 있습니다. 서버 재시작, 네트워크 오류, 예외 상황 등으로 EXPIRED 전환이 안 될 수 있습니다.

**증상**: 만료된 구독인데도 `org_entitlement.status = 'ACTIVE'`로 남아있어 서비스를 계속 쓸 수 있음.

**이걸 방어하는 구조가 Gate B의 이중 체크입니다:**

```typescript
// Gate B 실제 구현 (architecture.md §5.2 불변식 #9)
const entitlement = await getEntitlementByService(orgPk, service);
const now = new Date();

const pass =
  (entitlement.status === 'ACTIVE' || entitlement.status === 'GRACE')
  &&
  (entitlement.validUntil === null || entitlement.validUntil > now);
//                                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                         배치가 실패해도 이 조건이 차단함

if (!pass) throw new PaymentRequiredException();
```

`status === 'ACTIVE'`이더라도 `validUntil`이 과거라면 차단됩니다. 배치가 실패해도 서비스가 무한 유지되지 않습니다.

**수동 복구가 필요한 경우:**

```sql
-- 운영 플레이북 (architecture.md §12.7)
-- 1. 상태 확인
SELECT status, valid_until, grace_until
FROM org_entitlement
WHERE org_pk = ? AND service = ?;

-- 2. 정상적이라면 배치가 다음에 실행될 때 자동 처리
-- 3. 즉시 복구가 필요하다면 수동 처리
UPDATE org_entitlement
SET status = 'EXPIRED'
WHERE org_pk = ? AND valid_until < NOW() AND status = 'ACTIVE';

-- 4. perm_version 갱신 (클라이언트 캐시 무효화)
UPDATE organization SET perm_version = perm_version + 1 WHERE pk = ?;
```

배치 실패 모니터링도 중요합니다. `valid_until < NOW() AND status = 'ACTIVE'`인 row가 있다면 배치가 실패한 것입니다.

> 💡 **한 줄 요약**: 배치가 실패해도 Gate B가 `valid_until`을 이중으로 체크해서 차단합니다. status만 보는 코드는 "배치 실패 = 무한 무료"라는 심각한 버그를 유발합니다.

---

## 마치며

구독 상태 머신을 전체적으로 정리하면:

```
신규 가입
    ↓
TRIALING (무료 체험)
    ├── 체험 종료 + 결제 성공 → ACTIVE (정상 구독)
    │       ├── 다음 달 결제 성공 → ACTIVE 유지
    │       ├── 결제 실패 → PAST_DUE → GRACE → (재결제 성공: ACTIVE / 유예만료: EXPIRED)
    │       └── 취소 요청 → CANCELED → (기간 만료) → EXPIRED
    └── 체험 종료 + 결제 실패 → PAST_DUE → ...
```

코드를 짤 때 기억할 핵심 규칙:

1. **서비스 접근 여부**: 항상 `org_entitlement`만 읽어라 (`payment_ledger` 직접 조회 금지)
2. **Gate B 체크**: `status` + `validUntil` 둘 다 확인해라 (`status`만 보면 배치 실패 시 무한 무료)
3. **CANCELED**: 즉시 차단이 아니다, `valid_until`까지는 서비스 이용 가능
4. **상태 변경**: 항상 단일 트랜잭션으로 (`org_subscription` + `org_entitlement` + `perm_version` 동시 갱신)

---

## 연결된 개념

- [[gate-b-entitlement|Gate B & 엔타이틀먼트]] — org_subscription 상태가 org_entitlement에 반영되는 방식
- [[explainers/gate-b-billing-grace|Gate B 유예 기간 설계]] — PAST_DUE 이후 GRACE 처리 설계 결정
- [[feature-limits|feature_limits 우선순위]] — 플랜 변경 시 한도 갱신 시점
- [[idempotency-key|멱등성 키]] — 구독 갱신 결제 처리의 중복 방지
- [[outbox-pattern|Outbox 패턴]] — 구독 활성화 이벤트 비동기 fan-out 처리
- [[webhook-processing|PG 웹훅 처리]] — PG webhook이 구독 상태를 갱신하는 흐름
> 소스 문서
- [[architecture]] — §6 데이터 일관성, §12.11 TRIALING→EXPIRED 자동 전환 배치
- [[schema-reference]] — D.13 org_subscription DDL, F.2 구독 상태 머신
