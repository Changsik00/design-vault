---
difficulty: 중
tags:
  - platform-db
  - explainer
  - p1
  - billing
  - webhook
  - payment
  - idempotency
aliases:
  - 웹훅 처리
  - PG webhook
  - pg_webhook_event
  - Toss webhook
  - Stripe webhook
---

# PG 웹훅 수신·처리 흐름 설명

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[schema-reference]] §D.18, §F.4 · [[architecture]] §1.3, §4

결제 PG(Payment Gateway)가 보내는 웹훅은 "돈이 들어왔어요/나갔어요"를 우리 서버에 알려주는 신호입니다. 이 신호를 잘못 처리하면 "결제는 됐는데 서비스 이용이 안 된다"는 최악의 UX가 발생합니다. `pg_webhook_event` 테이블은 그 신호를 안전하게 받고 처리하는 안전망입니다.

---

## Q1. 웹훅(webhook)이 뭔가요? 결제 PG가 왜 우리한테 HTTP 요청을 보내나요?

결제는 "즉시 완료"가 아닌 경우가 많습니다. 카드사 승인, 은행 이체 확인 같은 과정이 비동기로 진행됩니다. 그래서 PG는 결과가 확정되면 **우리 서버 URL로 HTTP POST 요청을 보내** 알려줍니다. 이것이 웹훅입니다.

```
사용자 ──결제 버튼 클릭──▶ 우리 서버 ──결제 요청──▶ Toss/Stripe
                                                         │
                            (비동기, 수 초~수 분 후)     │
                                                         ▼
우리 서버(webhook endpoint) ◀──HTTP POST "결제 완료"── Toss/Stripe
```

쉽게 말하면, 우리가 Toss에 "결제 됐으면 여기로 연락 줘"라고 콜백 URL을 등록해 두는 것입니다. 마치 택배기사에게 "배달 완료하면 문자 보내 주세요"라고 부탁하는 것과 같습니다.

웹훅 이벤트 종류는 대략 이렇습니다:
- `payment.completed` — 결제 완료
- `payment.failed` — 결제 실패
- `subscription.renewed` — [[subscription-lifecycle|구독]] 자동 갱신
- `refund.completed` — 환불 완료

> 💡 **한 줄 요약**: PG는 결제 결과를 우리 서버에 HTTP POST로 능동적으로 알려주는데, 그게 웹훅입니다.

---

## Q2. Toss/Stripe에서 웹훅이 오면 DB에 어떤 순서로 반영되나요?

웹훅을 받자마자 모든 걸 한 번에 처리하면 실패했을 때 복구가 어렵습니다. 그래서 **수신 → 기록 → 처리** 3단계로 분리합니다.

```
1단계: 수신 (빠르게 200 OK 반환)
  Toss/Stripe ──HTTP POST──▶ /webhook/pg
                              │
                              ▼
              pg_webhook_event에 INSERT (status='RECEIVED')
                              │
                              ▼
              HTTP 200 OK 즉시 반환 ──▶ Toss/Stripe
              (PG는 200 받으면 "전달 완료"로 간주)

2단계: 서명 검증
  signature_ok 확인
  ├── FALSE → status='SKIPPED' (의심스러운 요청, 처리 안 함)
  └── TRUE  → 3단계로

3단계: 비즈니스 처리 (단일 트랜잭션)
  BEGIN;
    INSERT payment_ledger (결제 원장 기록)
    UPDATE org_subscription (구독 상태 갱신)
    UPSERT org_entitlement  (서비스 이용 권한 활성화)
    UPDATE organization SET perm_version = perm_version + 1
    INSERT outbox_event     (알림·영수증 발송 요청)
  COMMIT;
  UPDATE pg_webhook_event SET status='PROCESSED', processed_at=NOW()
```

1단계에서 200 OK를 빨리 반환하는 이유가 있습니다. PG는 정해진 시간(보통 5~10초) 안에 응답이 없으면 "실패"로 보고 재전송합니다. 처리가 오래 걸리더라도 "잘 받았어"를 먼저 알리는 것입니다.

> 💡 **한 줄 요약**: 수신은 즉시 기록 후 200 OK, 실제 처리는 그 뒤에 단일 트랜잭션으로 합니다. 처리 완료 후 [[outbox-pattern|Outbox 패턴]]을 통해 알림·영수증 발송이 비동기로 이루어집니다.

---

## Q3. 같은 웹훅이 두 번 올 수 있나요? (PG는 왜 재전송하나요?)

네, **반드시 온다고 가정해야 합니다.** PG는 200 OK를 받지 못하면 재전송합니다. 네트워크 일시 장애, 우리 서버 순간 다운, 처리 시간 초과 — 어떤 이유든 재전송이 발생할 수 있습니다.

```
상황: 우리 서버가 처리 중에 잠깐 다운됨

Toss ──웹훅 1차 전송──▶ 우리 서버 (처리 중 다운, 응답 없음)
Toss ──웹훅 2차 전송──▶ 우리 서버 (복구됨)
Toss ──웹훅 3차 전송──▶ 우리 서버 (이미 처리됨)
```

만약 이걸 처리하지 않으면 같은 결제가 세 번 처리되는 참사가 발생합니다. 이를 막는 것이 **[[idempotency-key|멱등성]](Idempotency)** 처리입니다.

`pg_webhook_event` 테이블의 이 [[index-design|인덱스]]가 핵심입니다:

```sql
UNIQUE KEY uq_provider_event (pg_provider, event_id)
-- pg_provider: 'TOSS', 'STRIPE', 'PAYPAL'
-- event_id: PG가 이벤트마다 부여한 고유 ID (예: 'evt_1abc2def3...')
```

같은 `(pg_provider, event_id)` 조합으로 두 번 INSERT하면 DB가 에러를 냅니다. 코드는 이 에러를 잡아 "이미 받은 이벤트"로 처리하고 200 OK를 반환합니다. PG 입장에서는 "잘 받았구나" 하고 재전송을 멈춥니다.

```typescript
try {
  await db.insert(pgWebhookEvent).values({
    pgProvider: 'TOSS',
    eventId: body.eventId,       // Toss가 부여한 이벤트 ID
    signatureOk: false,
    payloadJson: body,
    status: 'RECEIVED',
  });
} catch (e) {
  if (isDuplicateKeyError(e)) {
    // 이미 받은 이벤트 → 200 OK 반환 (PG 재전송 방지)
    return { success: true };
  }
  throw e;
}
```

> 💡 **한 줄 요약**: UNIQUE 제약으로 중복 INSERT를 차단해, 같은 웹훅이 100번 와도 1번만 처리됩니다.

---

## Q4. 웹훅 서명(signature) 검증이 뭔가요? 왜 필요한가요?

웹훅 엔드포인트 URL은 공개되어 있습니다. 누군가 Toss인 척 "100만원 결제 완료"를 위조해서 우리 서버에 보내면 어떻게 될까요? 이를 막는 것이 서명 검증입니다.

원리는 이렇습니다. Toss와 우리는 사전에 **비밀 키(secret key)** 를 공유합니다. Toss는 웹훅을 보낼 때 이 비밀 키로 본문을 암호화한 **서명값(signature)** 을 HTTP 헤더에 함께 보냅니다.

```
Toss가 보내는 요청:
  POST /webhook/pg
  Headers:
    Toss-Signature: hmac-sha256=abc123def456...   ← 서명
  Body:
    { "eventId": "evt_123", "amount": 49000, ... }
```

우리는 받은 본문을 같은 비밀 키로 암호화해서 서명을 직접 계산합니다. 두 서명이 일치하면 진짜 Toss에서 온 것, 다르면 위조된 요청입니다.

```typescript
import * as crypto from 'crypto';

function verifyTossSignature(
  payload: string,
  receivedSig: string,
  secretKey: string,
): boolean {
  const expected = crypto
    .createHmac('sha256', secretKey)
    .update(payload)
    .digest('hex');

  // 타이밍 공격 방어를 위해 timingSafeEqual 사용
  return crypto.timingSafeEqual(
    Buffer.from(`hmac-sha256=${expected}`),
    Buffer.from(receivedSig),
  );
}
```

검증 결과는 `signature_ok BOOLEAN` 컬럼에 저장됩니다. `FALSE`이면 비즈니스 처리 없이 `status='SKIPPED'`로 기록만 남깁니다. 기록을 남기는 이유는 나중에 "이 시간대에 위조 시도가 있었다"를 확인하기 위해서입니다.

> 💡 **한 줄 요약**: 공유 비밀 키로 HMAC 서명을 검증해 위조된 웹훅을 차단하고, 결과는 `signature_ok` 컬럼에 기록합니다.

---

## Q5. 웹훅 처리가 실패하면 어떻게 되나요? 재처리는 어떻게 하나요?

처리 중 에러가 나면 `status='FAILED'`로 저장됩니다. 그냥 두면 사용자가 결제했는데 서비스가 열리지 않는 상황이 됩니다. 이를 방지하는 것이 **재처리 워커(retry worker)** 입니다.

```
정상 흐름:
  RECEIVED → (서명 검증 OK) → PROCESSED

서명 실패:
  RECEIVED → (signature_ok=FALSE) → SKIPPED

처리 중 에러:
  RECEIVED → (처리 중 DB 에러, 타임아웃 등) → FAILED
                                                    │
                                                    ▼
                                          재처리 워커가 주기적으로 재시도
                                          (일정 횟수 실패 시 알람 발송)
```

재처리 워커가 사용하는 인덱스:

```sql
INDEX idx_pg_webhook_status (status, created_at)
-- 워커 쿼리: WHERE status='FAILED' AND created_at < NOW() - INTERVAL 1 HOUR
-- 인덱스가 없으면 전체 테이블을 뒤지는 풀스캔 발생
```

재처리 시 중요한 점: 비즈니스 처리 단계의 `payment_ledger` INSERT에도 `idempotency_key`가 있어서 재시도를 해도 결제 이중 처리가 되지 않습니다.

```typescript
// 재처리 워커 예시
async function retryFailedWebhooks() {
  const failedEvents = await db
    .select()
    .from(pgWebhookEvent)
    .where(
      and(
        eq(pgWebhookEvent.status, 'FAILED'),
        lt(pgWebhookEvent.createdAt, new Date(Date.now() - 60 * 60 * 1000)),
      ),
    )
    .limit(100);

  for (const event of failedEvents) {
    await processWebhookEvent(event); // 내부적으로 idempotency_key 체크
  }
}
```

> 💡 **한 줄 요약**: 실패한 웹훅은 `status='FAILED'`로 남겨두고, 재처리 워커가 주기적으로 재시도합니다. 재시도도 멱등하게 처리됩니다.

---

## Q6. pg_webhook_event.status 값들(RECEIVED/PROCESSED/SKIPPED/FAILED)은 각각 어떤 상태인가요?

`status` 컬럼은 웹훅의 생애주기를 추적합니다.

```
┌─────────────────────────────────────────────────────────┐
│ status 값      │ 의미                  │ 다음 행동        │
├─────────────────────────────────────────────────────────┤
│ RECEIVED       │ 방금 수신. 처리 전.   │ 서명 검증 → 처리 │
│ PROCESSED      │ 모든 처리 완료.       │ 없음 (종료 상태) │
│ SKIPPED        │ 서명 검증 실패.       │ 없음 (기록만)    │
│ FAILED         │ 처리 중 에러 발생.    │ 재처리 워커 대상 │
└─────────────────────────────────────────────────────────┘
```

실제 상황별로 보면:

**RECEIVED**: 웹훅이 막 도착해서 `pg_webhook_event`에 INSERT됐습니다. 아직 비즈니스 처리 전입니다. 서버가 갑자기 죽으면 이 상태로 남을 수 있습니다. (워커가 1시간 이상 된 RECEIVED도 재처리 대상으로 볼 수 있습니다.)

**PROCESSED**: `payment_ledger`, `org_entitlement`까지 모두 갱신 완료. `processed_at`에 완료 시각이 기록됩니다. 정상 종료 상태입니다.

**SKIPPED**: `signature_ok=FALSE`. 위조된 요청이거나 비밀 키가 맞지 않는 경우입니다. 처리는 건너뛰지만 **기록은 반드시 남깁니다.** 보안 감사 시 "언제 어디서 위조 시도가 있었나"를 파악할 수 있습니다.

**FAILED**: 서명은 통과했지만 비즈니스 처리 중에 에러가 났습니다. DB 연결 실패, 타임아웃, 예상치 못한 예외 등이 원인입니다. 재처리 워커가 주기적으로 이 상태의 이벤트를 집어 처리를 재시도합니다.

```sql
-- 운영 모니터링: status별 현황 한눈에 보기
SELECT
  status,
  COUNT(*) AS cnt,
  MIN(created_at) AS oldest
FROM pg_webhook_event
WHERE created_at > NOW() - INTERVAL 24 HOUR
GROUP BY status;
```

> 💡 **한 줄 요약**: RECEIVED(대기) → PROCESSED(완료) / SKIPPED(서명 실패) / FAILED(처리 실패, 재시도 대상)의 4가지 상태로 웹훅의 전 생애를 추적합니다.

---

## 마치며

웹훅 처리의 핵심은 세 가지입니다.

1. **멱등성**: UNIQUE(pg_provider, event_id)로 같은 웹훅이 100번 와도 1번만 처리
2. **보안**: signature_ok 검증으로 위조 요청 차단
3. **복구 가능성**: status='FAILED'로 실패 이력을 남겨 재처리 워커가 복구

이 구조가 있기 때문에 "결제는 됐는데 권한이 안 열린다"는 최악의 상황을 방지할 수 있습니다. 결제 시스템에서 가장 무서운 버그는 조용히 실패하는 것입니다. `pg_webhook_event` 테이블은 모든 수신 이력을 기록하기 때문에 "분명히 결제했는데"라는 사용자 문의가 와도 정확히 어디서 무엇이 잘못됐는지 추적할 수 있습니다.

---

## 연결된 개념

- [[idempotency-key|멱등성 키]] — event_id UNIQUE가 중복 웹훅을 막는 방식
- [[outbox-pattern|Outbox 패턴]] — 웹훅 처리 완료 후 async fan-out
- [[subscription-lifecycle|구독 상태 머신]] — 웹훅이 org_subscription 상태를 갱신하는 전체 흐름
- [[index-design|인덱스 설계]] — idx_pg_webhook_status 재처리 워커용 인덱스
> 소스 문서
- [[schema-reference]] — D.18 pg_webhook_event DDL, F.4 PG 웹훅 멱등 처리 흐름
