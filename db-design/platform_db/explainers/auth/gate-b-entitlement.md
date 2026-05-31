---
difficulty: 초
tags:
  - platform-db
  - explainer
  - p0
  - gate
  - billing
  - entitlement
aliases:
  - Gate B
  - 엔타이틀먼트
  - org_entitlement
---

# Gate B와 엔타이틀먼트 개념 설명

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[architecture|architecture.md]] §2.1 불변식 4/9/10 · [[schema-reference|schema-reference.md]] §D.12 org_entitlement · §E.2 Gate B · §F Billing 흐름

`platform_db`에서 결제와 서비스 접근 권한은 의도적으로 다른 테이블에 분리되어 있습니다. 이 문서는 [[gate-abc-flow|Gate A/B/C]] 인가 흐름 중 그 중심에 있는 `org_entitlement` 테이블과 Gate B가 어떻게 동작하는지를 설명합니다.

---

## Q1. 엔타이틀먼트(entitlement)가 뭔가요? 처음 듣는 단어예요.

`entitlement`를 한국어로 직역하면 "자격" 또는 "권한 부여"입니다. SaaS에서는 보통 **"이 고객이 이 서비스의 어느 기능을 어느 수준까지 쓸 수 있는가"의 현재 상태**를 의미합니다.

예를 들어 학원 SaaS에서:

```
org_pk = 42 (행복학원)의 entitlement:
  - ACADEMY 서비스: ACTIVE (구독 중)
  - feature_limits: { "daily_uploads": 10, "members": 100, "storage_gb": 50 }
  - valid_until: 2026-06-30
```

이 데이터만 보면 "행복학원은 2026년 6월 30일까지 ACADEMY 서비스를 쓸 수 있고, 하루 영상 10개, 최대 100명까지 가능하다"를 알 수 있습니다. 결제가 어떤 방식으로 이루어졌는지(신용카드인지 계좌이체인지, 어느 PG를 썼는지, 인보이스 번호가 뭔지)는 `org_entitlement` 테이블에는 없습니다. **"현재 쓸 수 있냐"만 담겨 있는 것**입니다.

마치 공연 입장권 스탬프처럼 생각하면 됩니다. 티켓을 어디서 샀는지, 얼마를 냈는지는 스탬프에 없지만 "입장 가능" 여부는 스탬프 하나로 판단합니다.

> 💡 **한 줄 요약**: entitlement는 결제 이력이 아니라 "지금 이 조직이 어느 서비스를 어느 수준으로 쓸 수 있는가"의 현재 상태입니다.

---

## Q2. 결제 정보가 있는데 왜 `org_entitlement` 테이블을 따로 만들었나요?

결제 테이블(`org_subscription`, `payment_ledger`)을 두고도 엔타이틀먼트 테이블을 따로 만든 이유는 **두 테이블이 다른 질문에 답하기 때문**입니다. ([[subscription-lifecycle|구독]] 상태 머신과의 관계는 별도 문서 참고)

```
org_subscription, payment_ledger → "결제가 어떻게 됐나?"
org_entitlement                  → "지금 서비스를 쓸 수 있나?"
```

이 설계를 "Billing Truth → Projection → Authorization"이라고 부릅니다.

```
PG Webhook (Toss/Stripe 이벤트)
    ↓
org_subscription  ← 결제 정보의 원본 (Billing Canonical Truth)
    ↓ [단일 트랜잭션으로 투영]
org_entitlement   ← 접근 권한의 단일 진실 (Authorization Projection)
    ↓
Gate B            ← "쓸 수 있나?"만 판단. subscription 직접 조회 금지
```

**왜 이 구조가 필요한가?** 결제 테이블에는 매우 복잡한 정보가 들어있습니다.

```sql
-- org_subscription의 실제 컬럼들
-- provider, external_sub_id, trial_ends_at,
-- current_period_start, current_period_end, grace_until,
-- cancelled_at, retry_count... 등등
```

Gate B는 이 모든 정보가 필요하지 않습니다. "이 조직이 지금 서비스를 쓸 수 있나?"라는 질문에는 `status`, `valid_until` 두 개면 충분합니다. 권한 체크마다 결제 테이블의 복잡한 로직을 다시 계산하면:

1. **속도 문제**: 모든 API 요청마다 결제 테이블 조회 → 느림
2. **결합도 문제**: billing이 터지면 auth도 같이 터짐
3. **캐싱 문제**: 복잡한 조건을 캐싱하기 어려움

`org_entitlement`를 중간에 두면, Gate B는 단일 테이블 단일 인덱스 조회로 끝납니다.

```sql
-- Gate B가 실행하는 쿼리 (단순)
SELECT status, valid_until, grace_until, feature_limits
FROM org_entitlement
WHERE org_pk = ? AND service = ?;
-- 인덱스: idx_org_service_status (org_pk, service, status, valid_until)
```

> 💡 **한 줄 요약**: 결제 복잡도를 권한 체크에서 격리하기 위해 `org_entitlement`를 별도 투영 테이블로 만들었습니다.

---

## Q3. `org_entitlement.status` 값(ACTIVE/GRACE/SUSPENDED/EXPIRED)은 누가 어떻게 바꾸나요?

`status`는 세 가지 경로로 변경됩니다.

### 경로 1: PG Webhook (가장 일반적)

결제가 성공하거나 실패할 때 Toss/Stripe가 webhook을 보내고, 우리 시스템이 이를 처리하면서 단일 트랜잭션으로 `org_entitlement`를 갱신합니다.

```sql
-- 결제 성공 시 (단일 트랜잭션 안에서)
BEGIN;
  INSERT INTO payment_ledger (...); -- 결제 원장 기록
  UPDATE org_subscription SET status='ACTIVE', current_period_end=? WHERE pk=?;
  INSERT INTO org_entitlement (org_pk, service, status, valid_until, ...)
    VALUES (?, 'ACADEMY', 'ACTIVE', '2026-06-30', ...)
    ON CONFLICT (org_pk, service) DO UPDATE SET
      status='ACTIVE', valid_until=EXCLUDED.valid_until;
  UPDATE organization SET perm_version = perm_version + 1 WHERE pk=?;
  INSERT INTO outbox_event (...); -- 알림 등 비동기 처리
COMMIT;
```

"결제됐는데 서비스 접근이 안 된다"는 최악의 SaaS UX가 구조적으로 불가능한 이유가 이것입니다. 결제와 권한 활성화가 **하나의 트랜잭션**이기 때문입니다.

### 경로 2: 배치 잡 (정기 만료 처리)

매일 1회 배치가 돌면서 만료된 entitlement를 자동으로 상태 변환합니다.

```sql
-- 매일 실행하는 만료 처리
UPDATE org_entitlement
  SET status = 'EXPIRED'
WHERE valid_until < now()
  AND status IN ('ACTIVE', 'TRIALING');

-- GRACE 전환: 결제 실패 후 유예기간 처리
UPDATE org_entitlement
  SET status = 'GRACE'
WHERE valid_until < now()
  AND grace_until > now()
  AND status = 'ACTIVE';
```

### 경로 3: 수동/프로모션 (관리자)

관리자가 프로모션이나 무료 체험을 직접 부여할 때:

```sql
INSERT INTO org_entitlement (org_pk, product_code, service, status, source, ...)
VALUES (?, 'ACADEMY_FREE', 'ACADEMY', 'ACTIVE', 'PROMO', ...);
-- source: SUBSCRIPTION / PROMO / MANUAL / FREE
```

`source` 컬럼이 "왜 이 entitlement가 생겼나"를 기록합니다. 나중에 "이 학원은 왜 무료로 쓰고 있지?"를 추적할 때 씁니다.

상태 전환을 정리하면:

```
(신규 구독)  → ACTIVE
ACTIVE  → (결제 실패) → PAST_DUE → (유예기간) → GRACE
GRACE   → (유예만료)  → EXPIRED
ACTIVE  → (취소)      → EXPIRED
ACTIVE  → (관리자 정지) → SUSPENDED
```

> 💡 **한 줄 요약**: status는 webhook 처리(결제 성공/실패), 일일 배치(만료), 관리자 직접 조작 세 경로로 변경됩니다.

---

## Q4. [[feature-limits|feature_limits]]가 뭔가요? JSONB로 저장한다고요?

`feature_limits`는 이 조직이 사용할 수 있는 기능의 **상한선**을 저장하는 JSONB 컬럼입니다.

```json
{
  "daily_uploads": 10,
  "members": 100,
  "storage_gb": 50,
  "ai_queries_per_month": 1000
}
```

이 값으로 "오늘 업로드를 10개 이상 했으면 더 못한다"는 식의 한도 체크를 합니다.

**왜 JSONB로 저장하나요?** 기능 한도의 종류가 서비스마다, 요금제마다 다르기 때문입니다. 컬럼으로 뽑으면 `daily_uploads INT, members INT, storage_gb INT, ai_queries_per_month INT...` 식으로 끝도 없이 늘어납니다. 새 요금제를 추가할 때마다 DDL을 변경해야 합니다.

**주의: 한도 체크는 항상 `org_entitlement.feature_limits`만 읽어야 합니다.**

```typescript
// 올바른 방법 ✅
const limits = await getFeatureLimit(orgPk, service);
// → org_entitlement.feature_limits를 읽음

// 금지 ❌
const product = await getProduct(productCode);
const limit = product.features.daily_uploads; // product_feature 직접 조회 금지
```

왜 이 규칙이 있을까요? 요금제가 중간에 바뀌거나 관리자가 특정 조직에 한도를 특별히 늘려줬을 때, `product_feature`는 기본값이라 반영이 안 됩니다. `org_entitlement.feature_limits`가 "이 조직의 현재 실제 한도"의 최종 진실이기 때문입니다.

> 💡 **한 줄 요약**: feature_limits는 이 조직에 적용된 기능 한도의 최종 진실이며, JSONB로 저장해 요금제 변경 없이 유연하게 관리합니다.

---

## Q5. `canXXX()` 함수가 `org_subscription`이나 `payment_ledger`를 직접 안 보는 이유가 있나요?

이것은 설계의 불변식으로 명시되어 있는 원칙입니다.

> **불변식 #4**: `canXXX()`는 `org_entitlement`만 읽는다. `payment_ledger` 직접 조회 금지.

이유는 이미 Q2에서 다뤘지만, 코드 관점에서 더 구체적으로 설명하면:

```typescript
// 금지: 결제 테이블을 직접 보는 방식 ❌
async function canAccessService(orgPk: number, service: string) {
  const subscription = await getSubscription(orgPk);
  const lastPayment = await getLastPayment(subscription.pk);
  
  // 이걸 직접 계산하면?
  if (subscription.status === 'PAST_DUE') {
    if (lastPayment.failedAt + GRACE_PERIOD > now) {
      return true; // 유예기간 내
    }
  }
  if (subscription.trialEndsAt > now) return true;
  // ... 수십 가지 조건
}

// 올바른 방식 ✅
async function canAccessService(orgPk: number, service: string) {
  const entitlement = await getEntitlementByService(orgPk, service);
  return (
    (entitlement.status === 'ACTIVE' || entitlement.status === 'GRACE') &&
    (entitlement.validUntil === null || entitlement.validUntil > now)
  );
}
```

첫 번째 방식의 문제:

1. **결합도**: billing 로직이 바뀌면 canXXX도 바꿔야 함
2. **속도**: 쿼리가 2개 이상 필요
3. **캐싱 어려움**: 복잡한 조건은 TTL 캐싱이 불안전
4. **중복**: 모든 서비스마다 이 로직을 따로 구현하면 하나 틀리면 버그

**billing 장애 격리** 효과도 있습니다. 만약 `org_subscription` 테이블이 느려지거나 장애가 생겨도, `org_entitlement`는 이미 계산된 상태를 저장하고 있기 때문에 Gate B는 정상 동작합니다. billing 서비스 장애가 auth 서비스로 전파되지 않습니다.

> 💡 **한 줄 요약**: canXXX가 entitlement만 보는 것은 결합도 제거, 속도, 캐싱, 장애 격리를 동시에 얻기 위한 의도적 설계입니다.

---

## Q6. Gate B가 실패하면 HTTP 응답 코드가 뭔가요? 401이랑 다른가요?

Gate B 실패 시 HTTP 코드는 **402 Payment Required**입니다.

```typescript
// Gate B 체크 코드
const now = new Date();
const pass =
  (entitlement.status === 'ACTIVE' || entitlement.status === 'GRACE') &&
  (entitlement.validUntil === null || entitlement.validUntil > now);

if (!pass) throw PaymentRequiredException(); // HTTP 402
```

세 코드의 차이를 정리하면:

| HTTP 코드 | 의미 | 언제 발생 | 해결 방법 |
|----------|------|----------|----------|
| **401 Unauthorized** | 인증 실패 | Firebase JWT가 없거나 유효하지 않음 | 다시 로그인 |
| **403 Forbidden** | 인가 실패 | 소속이 없거나 정지됨 (Gate A/C) | 관리자에게 문의 |
| **402 Payment Required** | 결제 필요 | 구독 만료/정지 (Gate B) | 결제 갱신 |

**402는 현실에서 잘 안 쓰이는 코드 아닌가요?** HTTP 스펙 상 402는 "미래에 사용 예정"으로 오랫동안 방치됐지만, SaaS에서 구독 만료를 표현하는 데 사실상 표준으로 자리잡았습니다. Stripe, GitHub, Slack 등 대형 SaaS들도 402를 이 용도로 씁니다.

**프론트엔드 처리 패턴**:

```typescript
// API 클라이언트
try {
  await api.uploadVideo(data);
} catch (error) {
  if (error.status === 402) {
    // 결제 페이지로 이동 or 업그레이드 모달 표시
    router.push('/billing/upgrade');
  } else if (error.status === 403) {
    // "접근 권한이 없습니다" 메시지
    toast.error('이 기능을 사용할 권한이 없습니다.');
  } else if (error.status === 401) {
    // 로그인 페이지로
    router.push('/login');
  }
}
```

**GRACE 상태에서는 402가 안 나옵니다.** GRACE는 "결제 실패했지만 [[explainers/auth/gate-b-billing-grace|유예 기간]] 중"이므로, 서비스는 정상적으로 이용 가능합니다. 다만 "XX일 안에 결제를 갱신하지 않으면 서비스가 중단됩니다"라는 배너를 보여주는 것이 UX 관행입니다.

```typescript
// GRACE 상태 안내 배너
if (entitlement.status === 'GRACE') {
  const daysLeft = Math.ceil(
    (entitlement.graceUntil.getTime() - now.getTime()) / (1000 * 60 * 60 * 24)
  );
  showBanner(`결제가 실패하였습니다. ${daysLeft}일 내에 갱신해 주세요.`);
}
```

> 💡 **한 줄 요약**: Gate B 실패는 HTTP 402로, 401(인증 실패)·403(권한 없음)과 명확히 구분되어 프론트엔드가 올바른 안내를 줄 수 있습니다.

---

## 마치며

`org_entitlement`는 billing의 복잡함을 한 곳에서 흡수하고 "지금 쓸 수 있나?"라는 단순한 질문에만 답합니다. Gate B 코드를 짤 때는 반드시 `org_entitlement`만 읽어야 한다는 불변식을 기억하세요. `org_subscription`이나 `payment_ledger`를 Gate B 로직에서 직접 참조하는 PR은 코드 리뷰에서 반려됩니다.

---

## 연결된 개념

- [[gate-abc-flow|Gate A/B/C 전체 흐름]] — 전체 3-gate 중 Gate B의 위치
- [[explainers/auth/gate-b-billing-grace|Gate B 유예 기간 설계]] — status + validUntil 복합 체크 결정 배경
- [[feature-limits|feature_limits 우선순위]] — entitlement.feature_limits가 런타임 SSOT인 이유
- [[subscription-lifecycle|구독 상태 머신]] — 구독 상태가 entitlement status에 반영되는 흐름
> 소스 문서
- [[architecture]] — 불변식 #4 (canXXX는 org_entitlement만), #9 (Gate B 복합 체크), #10 (feature_limits SSOT)
- [[schema-reference]] — D.12 org_entitlement DDL, E.2 Gate B 구현 코드
