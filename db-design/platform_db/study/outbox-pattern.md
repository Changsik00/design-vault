# Outbox 패턴 — 이벤트 유실 없이 async 처리

## 배경

결제가 완료되면 여러 일이 동시에 일어나야 한다:
- 영수증 이메일 발송
- 분석 시스템 기록
- 알림 메시지 발송

"결제 처리 → 이메일 발송"을 순서대로 하면 간단해 보인다. 그런데 실제로는:

---

## 문제 — 두 단계 사이 실패

```
[방법 1: 트랜잭션 후 직접 발송]

BEGIN;
  UPDATE org_entitlement SET status='ACTIVE';
COMMIT;  ← 성공

sendEmail(user, "결제 완료");  ← 여기서 실패?
```

COMMIT 성공 후 이메일 서버 오류 → 이메일 유실. 재시도 로직을 추가해도 서버가 여기서 재시작하면 재시도 자체가 사라진다.

```
[방법 2: 이메일 먼저]

sendEmail(user, "결제 완료");  ← 성공

BEGIN;
  UPDATE org_entitlement SET status='ACTIVE';
COMMIT;  ← 여기서 실패?
```

이메일은 발송됐는데 DB는 업데이트 안 됨 → 사용자는 메일 받았는데 서비스 접근 불가.

**두 작업을 완전히 원자적으로 묶을 방법이 필요하다.**

---

## Outbox 패턴

이벤트를 **같은 트랜잭션 안에 DB에 기록**하고, 별도 워커가 나중에 처리한다.

```
[우리 방법: Outbox]

BEGIN;
  UPDATE org_entitlement SET status='ACTIVE';
  INSERT INTO outbox_event (event_type, payload_json, status)
  VALUES ('subscription.activated', '{"orgPk":1, ...}', 'PENDING');
  -- 이메일 발송 안 함. outbox에 기록만.
COMMIT;  ← 두 작업이 동시에 성공하거나 동시에 실패

-- 별도 워커 프로세스:
SELECT * FROM outbox_event WHERE status = 'PENDING' FOR UPDATE SKIP LOCKED;
-- 이메일 발송 성공 →
UPDATE outbox_event SET status = 'SENT', sent_at = NOW() WHERE pk = ?;
-- 이메일 발송 실패 →
UPDATE outbox_event SET status = 'FAILED' WHERE pk = ?;  -- 다음 워커 재시도
```

---

## 핵심 구조

```sql
CREATE TABLE outbox_event (
  pk             BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  aggregate_type VARCHAR(50) NOT NULL,    -- 어떤 도메인 이벤트인가 (subscription, user...)
  aggregate_pk   BIGINT UNSIGNED NOT NULL, -- 해당 도메인 row의 pk
  event_type     VARCHAR(80) NOT NULL,    -- 이벤트 종류 (subscription.activated)
  payload_json   JSON NOT NULL,           -- 이벤트 내용
  status         ENUM('PENDING', 'SENT', 'FAILED') NOT NULL DEFAULT 'PENDING',
  created_at     TIMESTAMP NOT NULL DEFAULT NOW(),
  sent_at        TIMESTAMP,
  INDEX idx_outbox_status_created (status, created_at)
);
```

---

## 워커 동작

```typescript
// 워커: PENDING 이벤트를 하나씩 처리
async function processOutbox() {
  // FOR UPDATE SKIP LOCKED: 다른 워커가 이미 처리 중인 row는 건너뜀
  const events = await db.execute(sql`
    SELECT * FROM outbox_event
    WHERE status = 'PENDING'
    ORDER BY created_at
    LIMIT 10
    FOR UPDATE SKIP LOCKED
  `);

  for (const event of events) {
    try {
      await deliver(event);  // 이메일 발송, 슬랙 알림 등
      await markSent(event.pk);
    } catch (e) {
      await markFailed(event.pk);  // 다음 실행에 재시도
    }
  }
}
```

---

## at-least-once delivery

워커가 `deliver()` 후 `markSent()` 전에 죽으면?
→ 다음 워커 실행에 같은 이벤트를 다시 처리 → 이메일 두 번 발송 가능

이것이 **at-least-once** 의미다. 최소 한 번은 보장하지만 두 번 갈 수 있다.

대응: 수신자가 중복을 무시하도록 이메일 template에 멱등성 처리. 또는 idempotency_key로 재발송 방지.

---

## 왜 Kafka를 안 쓰나

| 항목 | Kafka | Outbox 패턴 (우리) |
|---|---|---|
| 메시지 유실 방지 | 브로커가 보장 | DB 트랜잭션이 보장 |
| 인프라 복잡도 | 높음 (브로커, ZooKeeper) | 낮음 (DB만) |
| 처리량 | 매우 높음 | 중간 |
| 현 규모 적합성 | 과도 | 적합 |

Kafka는 처리량이 필요할 때 또는 ISMS-P 감사 요건(T4) 충족 시 도입 예정.
outbox_event → Debezium CDC → Kafka로 전환 시 소비 앱 코드는 변경 없음.

---

## 관련 파일

| 파일 | 내용 |
|---|---|
| `core/schema-reference.md §D.19` | outbox_event 테이블 DDL |
| `core/schema-reference.md §F.1` | 결제 단일 트랜잭션 (outbox INSERT 포함) |
| `decisions/payment-atomicity.md` | Kafka를 쓰지 않은 결정 |
