---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p1
  - event
  - async
  - pattern
  - reliability
aliases:
  - Outbox 패턴
  - transactional outbox
  - outbox_event
  - dual write problem
---

# Outbox 패턴 설명

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[architecture]] §2.1 불변식 #7, §1.3, [[schema-reference]] §D.19, §F.1

결제 성공 후 이메일을 보내고, 알림을 전송하고, 검색 인덱스를 갱신해야 합니다. 이런 "부수효과"를 어떻게 안정적으로 처리할까요? Outbox 패턴은 이 문제에 대한 검증된 해법입니다.

---

## Q1. Outbox 패턴이 뭔가요? 왜 필요한가요?

Outbox 패턴은 DB 트랜잭션과 외부 시스템 호출을 **원자적으로 연결**하는 패턴입니다.

"원자적"이 뭔지 이해하려면 먼저 문제를 봐야 합니다.

결제 성공 후 해야 할 일들:

```
1. payment_ledger에 CHARGE 기록 (DB)
2. org_subscription.status → 'ACTIVE' (DB)
3. org_entitlement 활성화 (DB)
4. 이메일 발송 → SendGrid API 호출 (외부)
5. 푸시 알림 → FCM 호출 (외부)
6. 분석 이벤트 기록 → Analytics API 호출 (외부)
```

1~3번은 하나의 DB 트랜잭션으로 묶을 수 있습니다. 하지만 4~6번은 외부 API 호출이라서 트랜잭션 안에 넣을 수 없습니다.

**문제**: 1~3번 성공 후 4번 호출 사이에 서버가 크래시하면?

```
BEGIN;
  INSERT payment_ledger ...   ← 성공
  UPDATE org_subscription ... ← 성공
  UPDATE org_entitlement ...  ← 성공
COMMIT;                        ← DB는 완료됨

이메일 발송 → ...서버 크래시!  ← 이메일 안 감
```

서버가 재시작되면 이 상황을 어떻게 복구해야 할지 알 수 없습니다. 이메일을 보내야 하는지, 보내지 말아야 하는지 알 방법이 없습니다.

**Outbox 패턴의 해법**:

DB 트랜잭션 안에 "이 이벤트를 나중에 발송해야 해"라는 기록을 함께 남깁니다:

```sql
BEGIN;
  INSERT payment_ledger ...   ← 결제 기록
  UPDATE org_subscription ... ← 구독 상태
  UPDATE org_entitlement ...  ← 권한 활성화
  INSERT outbox_event (event_type='subscription.activated', ...) ← "이메일 보내야 함" 기록
COMMIT;  ← 이게 성공하면 이메일도 반드시 보내야 함을 DB가 기억
```

별도 워커(poller)가 `outbox_event` 테이블을 주기적으로 읽어서 실제 이메일/알림을 보냅니다.

> 💡 **한 줄 요약**: Outbox 패턴은 "DB 변경"과 "외부 이벤트 발송"을 같은 트랜잭션에 넣어, 서버가 크래시해도 이벤트를 반드시 처리하도록 보장하는 패턴입니다.

---

## Q2. 결제가 성공하면 "이메일 발송" 같은 부수효과를 왜 직접 안 보내나요?

직관적으로는 이렇게 하고 싶을 겁니다:

```typescript
// 이렇게 하면 안 되는 이유
async function handlePaymentSuccess(orgPk: number) {
  await db.transaction(async (tx) => {
    await tx.insert(paymentLedger).values(...);
    await tx.update(orgSubscription)...;
    await tx.update(orgEntitlement)...;
  });

  // 트랜잭션 밖에서 이메일 발송
  await sendEmail(user.email, 'payment-success');  // ← 여기서 문제 발생 가능
  await sendPushNotification(user.fcmToken, ...);
  await updateAnalytics(orgPk, 'payment');
}
```

**문제 1: 부분 실패**

```
트랜잭션 성공 → 이메일 발송 성공 → 푸시 알림 실패!
→ 이메일은 갔는데 푸시는 안 감
→ 재시도하면? 이메일이 두 번 감 (멱등성 문제)
→ 재시도 안 하면? 푸시 영원히 안 감
```

**문제 2: 외부 API 지연이 응답 시간에 영향**

```
결제 API 응답 시간 = DB 처리 시간 + 이메일 API 시간 + FCM 시간 + 분석 API 시간
= 100ms + 500ms + 200ms + 300ms = 1100ms

→ 외부 API 하나가 느려지면 결제 응답 전체가 느려짐
```

**문제 3: 외부 API 장애가 결제에 영향**

```
SendGrid 장애 발생 → 이메일 발송 실패 → 예외 던짐
→ 결제는 성공했는데 처리가 안 됨?
→ 이메일 때문에 결제가 실패로 처리되면 UX 최악
```

이 세 가지 문제를 Outbox 패턴이 깔끔하게 해결합니다:

```
결제 처리 시간 = DB 처리 시간만 (이메일/알림은 별도 워커가 비동기 처리)
= 100ms  ← 훨씬 빠름

외부 API 장애 = 워커가 재시도 처리 (결제 응답에 영향 없음)
부분 실패 = outbox_event가 PENDING 상태로 남아있어 재시도 보장
```

> 💡 **한 줄 요약**: 이메일/알림을 직접 보내면 외부 API 장애나 지연이 핵심 비즈니스 로직(결제)을 방해합니다. Outbox는 이것들을 분리해서 결제는 빠르고 안정적으로, 부수효과는 비동기로 처리합니다.

---

## Q3. 이벤트를 직접 Kafka/메시지큐로 보내면 어떤 문제가 생기나요?

"Outbox 말고 그냥 Kafka에 직접 publish하면 되지 않나?"라는 질문이 자연스럽게 나옵니다.

직접 Kafka publish 방식:

```typescript
async function handlePaymentSuccess(orgPk: number) {
  await db.transaction(async (tx) => {
    await tx.insert(paymentLedger).values(...);
    await tx.update(orgSubscription)...;
    await tx.update(orgEntitlement)...;
    // 트랜잭션 안에서 Kafka publish 시도
    await kafka.produce('subscription.activated', payload);  // ← 문제!
  });
}
```

**문제: 트랜잭션 원자성이 깨집니다.**

Kafka(또는 외부 메시지큐)는 DB 트랜잭션의 일부가 아닙니다. 두 개의 독립 시스템입니다:

```
시나리오 A: DB 성공 → Kafka publish 성공 → COMMIT 실패
  결과: Kafka에는 이벤트가 있지만 DB에는 변경 없음
  → 이메일은 갔는데 실제로 결제는 안 됨

시나리오 B: DB 성공 → Kafka publish 실패 → COMMIT 성공
  결과: DB는 변경됐지만 Kafka에 이벤트 없음
  → 결제는 됐는데 이메일이 영원히 안 감

시나리오 C: Kafka publish 후 DB ROLLBACK
  결과: 이벤트는 나갔는데 DB는 롤백됨
  → 결제 안 됐는데 "결제 완료" 이메일 발송
```

이걸 "분산 트랜잭션 문제" 또는 "dual write problem"이라고 합니다. 두 시스템에 동시에 원자적으로 쓰는 것은 매우 어렵습니다.

**Outbox 패턴은 이 문제를 어떻게 우회하나요?**

```
핵심 아이디어:
  "DB와 메시지큐에 동시에 쓰지 말고,
   DB에만 쓰고 (outbox_event),
   그걸 나중에 읽어서 메시지큐에 보내자"
```

```
DB 트랜잭션:
  - payment_ledger INSERT  ← DB
  - org_entitlement UPDATE ← DB
  - outbox_event INSERT    ← DB (같은 트랜잭션!)
COMMIT → 이 모든 게 원자적

별도 워커:
  - outbox_event 읽기 (PENDING)
  - Kafka/FCM/Email API 호출
  - 성공 시 outbox_event.status → 'SENT'
  - 실패 시 재시도 (outbox_event는 DB에 남아있으니까)
```

DB 트랜잭션 내에서는 오직 DB 작업만 합니다. 외부 시스템 호출은 트랜잭션이 완전히 끝난 후에 별도 워커가 담당합니다.

Outbox는 **현재 아키텍처의 의도적 선택**이기도 합니다. architecture.md 불변식 #7에 명시되어 있습니다:

> **strong consistency = 단일 서버 트랜잭션. async = outbox.**

Kafka/Debezium 같은 CDC 방식은 나중에 도입 조건이 되면 outbox 위에 얹을 수 있습니다. 지금 단계에서는 Kafka 인프라 없이도 안정적으로 동작합니다.

> 💡 **한 줄 요약**: DB와 Kafka에 동시에 쓰면 둘 중 하나가 실패할 때 데이터 불일치가 생깁니다. Outbox는 DB에만 기록하고 워커가 나중에 처리해서 이 문제를 우회합니다.

---

## Q4. outbox_event 테이블에 INSERT하는 시점은 언제인가요?

**핵심 비즈니스 트랜잭션과 같은 트랜잭션 안에서** INSERT합니다.

```sql
-- 결제 성공 처리 (단일 트랜잭션)
BEGIN;
  -- 1. 결제 원장 기록
  INSERT INTO payment_ledger (org_pk, type, amount_minor, idempotency_key, status)
  VALUES (?, 'CHARGE', ?, ?, 'SUCCEEDED');

  -- 2. 구독 상태 갱신
  UPDATE org_subscription SET status = 'ACTIVE', ... WHERE pk = ?;

  -- 3. 권한 활성화
  INSERT INTO org_entitlement (org_pk, service, status, valid_until, feature_limits)
  VALUES (?, 'ACADEMY', 'ACTIVE', ?, ?)
  ON DUPLICATE KEY UPDATE status = 'ACTIVE', ...;

  -- 4. perm_version 갱신
  UPDATE organization SET perm_version = perm_version + 1 WHERE pk = ?;

  -- 5. outbox 이벤트 기록 ← 여기서 같이!
  INSERT INTO outbox_event (
    aggregate_type, aggregate_pk, event_type, payload_json, status
  )
  VALUES ('subscription', ?, 'subscription.activated', ?, 'PENDING');
COMMIT;
-- COMMIT이 성공하면 결제 기록 + 권한 활성화 + 이벤트 기록이 모두 원자적으로 확정
```

이 방식의 보장:

```
COMMIT 성공 → outbox_event 반드시 존재 → 워커가 반드시 처리
COMMIT 실패 → 전체 롤백 → outbox_event도 없음 → 이메일 발송 없음
```

트랜잭션 밖에서 INSERT하면 어떻게 될까요?

```typescript
// 잘못된 방법
await db.transaction(async (tx) => {
  await tx.insert(paymentLedger)...;
  await tx.update(orgSubscription)...;
  await tx.update(orgEntitlement)...;
}); // ← 트랜잭션 끝

// 트랜잭션 밖에서 outbox INSERT
await db.insert(outboxEvent)...; // ← 여기서 서버 크래시 → 이벤트 없음!
```

트랜잭션 밖에서 INSERT하면 "결제 성공했지만 이메일 미발송"이 됩니다. 그래서 **반드시 같은 트랜잭션 안에서** INSERT해야 합니다.

> 💡 **한 줄 요약**: `outbox_event` INSERT는 반드시 핵심 비즈니스 트랜잭션(결제 처리) 안에서 함께 해야 합니다. 트랜잭션 밖에서 하면 Outbox 패턴의 의미가 없어집니다.

---

## Q5. outbox_event 워커(폴러)가 어떻게 동작하나요? 실패하면 어떻게 되나요?

워커는 주기적으로 PENDING 상태의 이벤트를 읽어서 처리합니다.

**기본 동작:**

```typescript
// outbox 워커 (폴링 방식)
async function processOutboxEvents() {
  // 처리할 이벤트 가져오기 (한 번에 최대 10개)
  const events = await db.query.outboxEvent.findMany({
    where: eq(outboxEvent.status, 'PENDING'),
    orderBy: asc(outboxEvent.createdAt),
    limit: 10,
  });

  for (const event of events) {
    try {
      await processEvent(event);
      // 성공 → SENT로 업데이트
      await db.update(outboxEvent)
        .set({ status: 'SENT', sentAt: new Date() })
        .where(eq(outboxEvent.pk, event.pk));
    } catch (error) {
      // 실패 → FAILED로 업데이트
      await db.update(outboxEvent)
        .set({ status: 'FAILED' })
        .where(eq(outboxEvent.pk, event.pk));
      // 모니터링 알림 발송
      logger.error('outbox event failed', { eventId: event.pk, error });
    }
  }
}

// 30초마다 실행
setInterval(processOutboxEvents, 30_000);
```

**워커가 여러 인스턴스로 실행될 때 (분산 환경):**

같은 이벤트를 두 워커가 동시에 처리하면 이메일이 두 번 발송될 수 있습니다. 이를 막으려면 잠금이 필요합니다:

```typescript
// FOR UPDATE SKIP LOCKED 패턴 (MySQL 지원)
const events = await db.execute(sql`
  SELECT * FROM outbox_event
  WHERE status = 'PENDING'
  ORDER BY created_at ASC
  LIMIT 10
  FOR UPDATE SKIP LOCKED
`);
// → 다른 워커가 이미 처리 중인 row는 건너뜀
// → 같은 이벤트를 두 워커가 동시에 가져가지 않음
```

architecture.md에서 `FOR UPDATE SKIP LOCKED`는 PostgreSQL 네이티브 지원이지만 MySQL 8에서도 동작한다고 명시되어 있습니다.

**실패 시 재시도:**

```
이벤트 처리 실패 (예: SendGrid 일시 장애)
    ↓
status → 'FAILED'
    ↓
별도 재처리 배치 (수동 또는 자동):
  UPDATE outbox_event SET status = 'PENDING' WHERE status = 'FAILED';
    ↓
워커가 다시 집어서 처리
```

현재 인덱스 설계로 FAILED 이벤트를 빠르게 찾을 수 있습니다:

```sql
INDEX idx_outbox_status_created (status, created_at)
-- WHERE status = 'PENDING' ORDER BY created_at → 인덱스 활용
-- WHERE status = 'FAILED' → 재처리 대상 조회
```

**이벤트 타입별 처리 라우팅:**

```typescript
async function processEvent(event: OutboxEvent) {
  switch (event.eventType) {
    case 'subscription.activated':
      // 이메일 발송
      await emailService.sendPaymentSuccessEmail(event.payload);
      // 푸시 알림
      await fcmService.sendSubscriptionActivated(event.payload);
      break;

    case 'subscription.expired':
      await emailService.sendExpirationEmail(event.payload);
      break;

    case 'user.deleted':
      // 각 서비스 DB에 익명화 요청 (fan-out)
      await academyService.anonymizeUser(event.aggregatePk);
      await agentService.anonymizeUser(event.aggregatePk);
      break;

    default:
      logger.warn('unknown outbox event type', { type: event.eventType });
  }
}
```

**at-least-once의 의미**: 워커가 `deliver()`(이메일 발송 등)에 성공한 직후 `markSent()` 전에 죽으면, 다음 워커가 같은 PENDING 이벤트를 다시 집어 **두 번 발송**할 수 있습니다. 즉 "최소 한 번"은 보장하지만 "정확히 한 번"은 아닙니다. 그래서 소비 측이 중복을 흡수하도록 멱등 처리(수신자 측 dedup 또는 [[idempotency-key|`idempotency_key`]])를 함께 둡니다. 여러 워커가 동시에 돌 때는 `SELECT ... FOR UPDATE SKIP LOCKED`로 가져가, 한 row를 두 워커가 동시에 처리하는 일을 막습니다.

> 💡 **한 줄 요약**: 워커는 주기적으로 PENDING 이벤트를 가져와서 처리하고 SENT로 마킹합니다. 실패하면 FAILED로 남고 재처리할 수 있습니다. DB에 기록되어 있으니 어떤 이벤트가 미처리 상태인지 항상 알 수 있습니다. at-least-once 발행과 [[idempotency-key|멱등성]] 소비의 조합으로 중복 처리를 방지합니다.

---

## Q6. outbox_event와 billing_event의 차이가 뭔가요? 둘 다 이벤트 아닌가요?

이름이 비슷해서 헷갈리는 부분입니다. 두 테이블은 완전히 다른 목적입니다:

```
outbox_event    → "앞으로 할 일 목록" (작업 큐)
billing_event   → "이미 일어난 일 기록" (감사 로그)
```

**`billing_event`** — [[subscription-lifecycle|구독]] lifecycle의 감사 로그

```sql
CREATE TABLE billing_event (
  event_type ENUM('SUBSCRIPTION_START','SUBSCRIPTION_END','PLAN_CHANGE','INVOICE_PAID','INVOICE_FAILED'),
  -- ...
);
```

- 구독 상태가 변경될 때마다 기록하는 **불변 이력**입니다.
- "언제 플랜이 변경됐나?", "몇 번 결제 실패가 있었나?"를 추적합니다.
- 읽기 전용, 삭제/수정 금지 (append-only)
- 처리 상태(PENDING/SENT)가 없습니다. "처리해야 할 것"이 아니라 "일어난 것"의 기록이니까요.

**`outbox_event`** — 비동기 부수효과 실행을 위한 작업 큐

```sql
CREATE TABLE outbox_event (
  event_type VARCHAR(80),  -- 'subscription.activated', 'user.deleted', ...
  status     ENUM('PENDING','SENT','FAILED'),  -- 작업 상태
  sent_at    TIMESTAMP,                         -- 처리 완료 시각
  -- ...
);
```

- "이메일 보내야 함", "알림 보내야 함", "인덱스 갱신해야 함" 같은 **할 일**을 담습니다.
- 워커가 처리하고 나면 SENT로 마킹됩니다.
- 실패하면 재시도할 수 있는 작업 큐 성격입니다.

구체적인 비교:

| 구분 | billing_event | outbox_event |
|---|---|---|
| 목적 | 구독 이력 감사 | 비동기 부수효과 실행 |
| 읽는 주체 | 관리자, 분석, 감사 | 워커 (폴러) |
| 삭제 가능? | 불가 (append-only) | SENT 후 아카이빙 가능 |
| 처리 상태 | 없음 (이미 일어난 일) | PENDING/SENT/FAILED |
| 예시 | "2026-05-28 플랜 변경" | "subscription.activated → 이메일 보내야 함" |

실제로 하나의 결제 성공 이벤트가 두 테이블에 각각 기록됩니다:

```sql
BEGIN;
  -- ... 결제 처리 ...

  -- 감사 로그 (billing_event): "오늘 결제 성공했다"는 사실 기록
  INSERT INTO billing_event (event_type, org_pk, plan_code)
  VALUES ('INVOICE_PAID', ?, ?);

  -- 작업 큐 (outbox_event): "이메일 보내야 함"이라는 할 일 기록
  INSERT INTO outbox_event (event_type, payload_json, status)
  VALUES ('subscription.activated', ?, 'PENDING');
COMMIT;
```

> 💡 **한 줄 요약**: `billing_event`는 "무슨 일이 있었나"를 기록하는 불변 이력이고, `outbox_event`는 "앞으로 비동기로 처리해야 할 일"을 담는 작업 큐입니다. 이름이 비슷하지만 완전히 다른 용도입니다.

---

## 마치며

Outbox 패턴을 한 문장으로 정리하면:

> **"DB 변경과 외부 이벤트를 같은 트랜잭션에 묶어서, 서버가 죽어도 이벤트가 유실되지 않도록 보장한다."**

architecture.md 불변식 #7이 이걸 명시합니다:
> `strong consistency = 단일 서버 트랜잭션. async = outbox.`

코드를 짤 때 체크리스트:

```typescript
// 새로운 비즈니스 이벤트를 추가할 때:

// 1. 트랜잭션 안에 outbox INSERT 포함 (필수)
await db.transaction(async (tx) => {
  await tx.update(someDomainTable)...;
  await tx.insert(outboxEvent).values({   // ← 트랜잭션 안에서
    aggregateType: 'subscription',
    aggregatePk: subPk,
    eventType: 'your.event.type',
    payloadJson: { ... },
    status: 'PENDING',
  });
});

// 2. 워커에 이벤트 타입 처리 로직 추가
case 'your.event.type':
  await externalService.doSomething(event.payload);
  break;

// 3. 직접 외부 API 호출은 하지 않기 (트랜잭션 안에서도, 밖에서도)
// await emailService.send(...)  ← 하지 말 것
```

현재 Kafka 같은 메시지 브로커는 없지만, 나중에 도입하더라도 outbox_event를 CDC로 읽어서 Kafka에 넣는 방식으로 자연스럽게 확장할 수 있습니다. Outbox가 그 확장을 위한 토대입니다.

---

## Q7. MQ(Kafka/SQS/RabbitMQ)를 쓰면 outbox가 필요 없어지지 않나요?

아니요, **outbox는 MQ가 있어도 사라지지 않습니다.** 오히려 "outbox → MQ → 컨슈머" 구조가 됩니다.

**MQ가 실제로 해결하는 것:**

| 문제 | outbox 단독 | MQ 도입 후 |
|---|---|---|
| 배포 중 webhook 유실 | LB 타이밍에 의존 | MQ가 메시지 보관, 컨슈머만 재배포 |
| 워커 폴링 부하 | DB를 주기적으로 SELECT | MQ push, DB 조회 불필요 |
| `outbox_event` 무한 누적 | sweeper 잡 별도 필요 | MQ TTL로 자동 관리 |
| 워커 스케일 아웃 | 중복 실행 방지 로직 복잡 | 파티션/컨슈머 그룹으로 자연 해결 |

**그런데 MQ를 도입해도 outbox가 여전히 필요한 이유:**

DB 쓰기와 MQ 발행은 서로 다른 시스템이라 원자적으로 처리할 수 없습니다:

```
시나리오: DB commit 성공 → MQ publish 실패
  결과: DB는 변경됐지만 이벤트 소실 → downstream 미반영

시나리오: MQ publish 성공 → DB rollback
  결과: 이벤트는 나갔는데 DB에는 아무것도 없음 → 유령 이벤트
```

그래서 "DB에만 쓰는 outbox"가 MQ의 신뢰할 수 있는 입구 역할을 합니다:

```
현재 구조 (outbox = DB 기반 큐)
  DB 트랜잭션
    ├── 본 작업 (entitlement UPDATE 등)
    └── outbox_event INSERT  ← 큐에 넣는 것과 동일한 효과
  폴링 워커
    └── outbox_event 읽어서 외부 서비스 호출

MQ 도입 후 구조
  DB 트랜잭션
    ├── 본 작업
    └── outbox_event INSERT  ← 여전히 필요 (원자성 보장)
  CDC 워커 (Debezium 등)
    └── outbox_event 변경 감지 → Kafka topic으로 relay
  컨슈머
    └── Kafka에서 구독하여 처리
```

outbox가 없으면 MQ에도 at-least-once 보장이 깨집니다.

**언제 MQ 도입을 고려하나:**

아래 트리거 중 하나가 충족될 때 검토합니다:

- 초당 이벤트 수십 건 이상 → 폴링 워커가 병목
- 컨슈머(이메일/알림/인덱스)가 5개 이상 독립 확장 필요
- 배포 중 이벤트 유실이 실제 장애로 이어진 경험
- 팀이 Kafka 같은 별도 인프라를 운영할 여력이 생긴 시점
- ISMS-P 감사 요건(분리 트리거 T4) 등 컴플라이언스 요구가 생긴 시점 — 전환 시 `outbox_event → Debezium CDC → Kafka` 경로로 소비 앱 코드 변경 없이 얹을 수 있음

현재 규모에서는 outbox 단독으로 충분하고, MQ는 나중에 outbox 위에 CDC로 얹는 방식으로 자연스럽게 확장할 수 있습니다.

> 💡 **한 줄 요약**: MQ는 outbox를 대체하는 게 아니라 outbox를 더 잘 소비하는 수단입니다. DB-MQ 원자성 문제 때문에 outbox는 MQ가 있어도 필요합니다.

---

## 연결된 개념

- [[idempotency-key|멱등성 키]] — at-least-once 발행과 멱등 소비의 조합
- [[webhook-processing|PG 웹훅 처리]] — PG webhook(인바운드) vs outbox(아웃바운드) 역할 구분
- [[subscription-lifecycle|구독 상태 머신]] — subscription.activated 이벤트가 outbox에 INSERT되는 시점
> 소스 문서
- [[architecture]] — §2.1 불변식 #7 (strong consistency = 단일 트랜잭션, async = outbox), §1.3 결제 단일 트랜잭션
- [[schema-reference]] — D.19 outbox_event DDL, F.1 결제-권한 단일 트랜잭션 SQL
- [[payment-atomicity]] — Kafka 대신 단일 트랜잭션 + outbox로 결제 원자성을 보장한 결정
