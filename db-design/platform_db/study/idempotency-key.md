# 멱등키 — 같은 요청 두 번 와도 안전한 법

## 배경

PG(결제 게이트웨이)는 webhook을 **적어도 한 번(at-least-once)** 전송한다.

```
우리 서버가 webhook 처리 후 응답을 보내기 전에 서버가 재시작되면?
→ PG 입장: "응답 없음 = 실패"
→ PG가 동일 webhook을 재전송
→ 우리 서버: 동일 결제를 두 번 처리 → 이중 청구
```

네트워크 장애, 서버 재시작, PG 재전송 정책… 같은 요청이 두 번 오는 것은 피할 수 없다.

**멱등성(Idempotency)**: 같은 연산을 여러 번 실행해도 결과가 한 번 실행한 것과 동일.

---

## DB UNIQUE constraint로 멱등성 보장

```sql
-- pg_webhook_event: 동일 webhook 중복 처리 방지
CREATE TABLE pg_webhook_event (
  pk          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  pg_provider ENUM('TOSS', 'STRIPE', 'PAYPAL') NOT NULL,
  event_id    VARCHAR(255) NOT NULL,
  ...
  UNIQUE KEY uq_provider_event (pg_provider, event_id)  -- 핵심
);
```

```sql
-- payment_ledger: 동일 결제 중복 기록 방지
CREATE TABLE payment_ledger (
  pk              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  idempotency_key VARCHAR(255) NOT NULL,
  ...
  UNIQUE KEY uq_idempotency_key (idempotency_key)  -- 핵심
);
```

---

## 동작 방식

**첫 번째 webhook 수신**:
```sql
INSERT INTO pg_webhook_event (pg_provider, event_id, ...)
VALUES ('TOSS', 'evt_001', ...);
-- 성공 → 처리 진행
```

**같은 webhook 재수신**:
```sql
INSERT INTO pg_webhook_event (pg_provider, event_id, ...)
VALUES ('TOSS', 'evt_001', ...);
-- ERROR: Duplicate entry 'TOSS-evt_001' for key 'uq_provider_event'
-- → 이미 처리됨 → 200 OK 반환 (PG의 재전송 루프 종료)
```

코드 패턴:
```typescript
try {
  await db.insert(pgWebhookEvent).values({ pgProvider, eventId, ... });
  // INSERT 성공 → 처음 받는 webhook → 처리 진행
  await processWebhook(payload);
} catch (e) {
  if (isDuplicateKeyError(e)) {
    // 이미 처리된 webhook → 아무것도 하지 않고 성공 응답
    return { status: 'already_processed' };
  }
  throw e;
}
```

---

## idempotency_key 생성 전략

```typescript
// PG 인보이스 ID를 idempotency_key로 사용
const idempotencyKey = `toss_${invoice.orderId}`;

// 또는 PG payment ID
const idempotencyKey = `stripe_${paymentIntent.id}`;
```

**좋은 idempotency_key**: PG 측에서 부여한 고유 식별자 (같은 결제 = 항상 같은 key).
**나쁜 idempotency_key**: 서버에서 생성한 UUID (재시도 시 새 UUID → 중복 차단 안 됨).

---

## 두 개의 멱등 레이어

```
[레이어 1] pg_webhook_event UNIQUE(pg_provider, event_id)
  목적: 동일 webhook 이벤트 중복 처리 방지

[레이어 2] payment_ledger UNIQUE(idempotency_key)
  목적: 동일 결제 중복 기록 방지 (webhook과 영수증 검증 경로가 다를 때)
```

둘 다 있는 이유: webhook 이벤트와 결제 원장은 별도 생성 경로가 있을 수 있음. 이중 방어.

---

## 트레이드오프

| 항목 | DB UNIQUE (우리) | Redis SETNX |
|---|---|---|
| 구현 단순성 | 높음 (컬럼 추가만) | 중간 (Redis 연결 관리) |
| 내구성 | DB에 영구 저장 | Redis 재시작 시 소실 |
| 정확성 | 완벽 | TTL 만료 후 재처리 가능 |
| 의존성 | DB만 | DB + Redis |

---

## 관련 파일

| 파일 | 내용 |
|---|---|
| `core/schema-reference.md §D.18` | pg_webhook_event 테이블 DDL |
| `core/schema-reference.md §D.17` | payment_ledger idempotency_key |
| `core/schema-reference.md §F.4` | PG webhook 멱등 처리 흐름 |
| `decisions/payment-atomicity.md` | 결제 단일 트랜잭션 전체 |
