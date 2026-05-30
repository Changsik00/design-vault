---
type: decision
status: 채택
aliases:
  - authorization projection
  - org_entitlement 분리
  - billing projection
tags:
  - platform-db
  - decision
  - billing
  - authorization
  - gate-b
---

# billing truth → authorization projection

## 배경

구독 상태는 복잡하다:

```
org_subscription.status:
  TRIALING → ACTIVE → PAST_DUE → CANCELED → EXPIRED
                          ↓
                       GRACE (결제 실패 유예기간)
```

여기에 더해 PG 재시도, 환불, chargeback, 쿠폰, 수동 부여, 무기한 파일럿 계정까지.

Gate B가 "이 org가 지금 서비스를 쓸 수 있나?"를 판단할 때 이 복잡한 billing을 어떻게 읽어야 할까.

---

## 일반 권고 — billing 직접 조회

```typescript
// 일반적 접근: Gate B에서 org_subscription 직접 읽기
async function checkGateB(orgPk: bigint) {
  const sub = await db.query.orgSubscription.findFirst({
    where: eq(orgSubscription.orgPk, orgPk),
  });
  // PAST_DUE? GRACE? 유예기간 어떻게 계산? PG마다 다른가?
  return sub?.status === 'ACTIVE' || sub?.status === 'TRIALING';
}
```

**권고 근거**:
- 단일 진실 원천. subscription이 최신 상태이므로 직접 읽으면 됨
- 중간 테이블 없이 단순

**문제점**:
- billing semantics가 권한 체크마다 노출됨 (PAST_DUE는 접근 가능? grace_until 계산을 Gate B가 해야 하나?)
- billing 장애 → auth 장애로 직접 전파 (PG API 오류 시 모든 API 403)
- Toss/Stripe/PayPal별 status 해석이 Gate B 코드에 섞임
- 캐싱이 어려움 (복잡한 billing 상태를 언제 무효화할지 판단이 어려움)

---

## 우리 결정 — Authorization Projection 분리

billing의 "읽기 전용 투영"을 별도 테이블로 유지한다.

```
PG Webhook (Stripe/Toss 이벤트)
    ↓
org_subscription          ← Billing Canonical Truth
  (provider, invoice, retry_count, grace_until,
   refund, chargeback, trial, cancel_at_period_end ...)
    ↓ [단일 트랜잭션으로 투영]
org_entitlement           ← Authorization Projection
  (status: ACTIVE | GRACE | SUSPENDED | EXPIRED,
   valid_until, grace_until, feature_limits)
    ↓
Gate B                    ← "접근 가능?" 만 판단
```

```typescript
// Gate B: projection만 읽음. billing 내부 모름.
async function checkGateB(orgPk: bigint, service: EntitlementService) {
  const ent = await getEntitlementByService(orgPk, service);
  if (!ent) return false;
  const statusOk = ent.status === 'ACTIVE' || ent.status === 'GRACE';
  const notExpired = ent.validUntil === null || ent.validUntil > new Date();
  return statusOk && notExpired;
  // payment_ledger, org_subscription은 읽지 않는다 — 불변식 #4
}
```

**채택 근거**:
1. Gate B가 billing 복잡도를 알 필요 없음 — 4가지 상태(ACTIVE/GRACE/SUSPENDED/EXPIRED)만 판단
2. billing 장애 격리 — PG API 오류가 Gate B를 멈추지 않음
3. 캐싱 단순 — entitlement 한 row의 상태 + TTL만 관리
4. `feature_limits`(기능 한도)도 projection에 포함 → 런타임 권위 단일화 (불변식 #10)

---

## 트레이드오프

| 항목 | billing 직접 조회 | projection 분리 (우리) |
|---|---|---|
| 구현 단순성 | 테이블 1개 | 테이블 2개 + 동기화 로직 |
| billing 장애 전파 | Gate B도 멈춤 | 격리됨 |
| 캐싱 난이도 | 어려움 (복잡한 상태) | 쉬움 (단순 4-상태) |
| 상태 지연 | 없음 (항상 최신) | webhook → projection 수 초 |
| Gate B 가독성 | billing 개념 혼재 | auth 개념만 |

**eventual consistency 수용**: webhook → entitlement 투영 사이 수 초 지연 허용. 취소 직후 짧은 창에 권한이 살아있을 수 있음. SaaS 표준 베스트 프랙티스.

---

## 향후 조건

projection 동기화가 신뢰할 수 없다면 (배치 실패가 잦다면):
- `gate-b-billing-grace.md`의 `validUntil` 복합체크가 2차 방어선 역할을 함
- 이미 설계 확정. 배치 실패 시에도 만료일 이후 접근 차단 보장.

---

## 관련 문서

- [[decisions/gate-b-billing-grace|gate-b-billing-grace]] — validUntil 복합체크 세부 결정 (2차 방어선)
- [[payment-atomicity]] — subscription → entitlement 투영이 일어나는 트랜잭션
> 소스 문서
- [[schema-reference]] — §D.12 org_entitlement DDL, §F billing 흐름 전체
- [[architecture]] — §1.3 데이터 일관성(결제·권한 분리 원칙)
