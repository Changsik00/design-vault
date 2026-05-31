---
difficulty: 중
tags:
  - platform-db
  - explainer
  - p1
  - payment
  - billing
  - idempotency
aliases:
  - 멱등성
  - idempotency_key
  - 중복 결제 방지
---

# 멱등성 키 (payment_ledger) 설명

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[architecture]] §1.3, [[schema-reference]] §D.17~D.18, §F.4

결제는 실수가 허용되지 않는 영역입니다. "같은 결제를 두 번 처리했다"는 건 사용자에게 이중 청구가 된다는 뜻이고, 이건 단순 버그가 아니라 법적 분쟁까지 이어질 수 있습니다. `idempotency_key`는 이 문제를 DB 레벨에서 원천 차단하는 장치입니다.

---

## Q1. 멱등성(idempotency)이 뭔가요? 어려운 단어인데 쉽게 설명해주세요.

멱등성은 수학에서 온 개념인데, 실생활로 말하면 이렇습니다:

> **"같은 작업을 여러 번 해도 결과가 한 번 한 것과 똑같다."**

예시:

```
멱등적인 것:
  - 엘리베이터 버튼: 이미 눌린 버튼을 또 눌러도 한 번만 오름
  - 파일 삭제: 이미 없는 파일을 삭제해도 오류 없이 "없음" 상태 유지
  - HTTP GET: 몇 번 조회해도 데이터 안 바뀜

멱등적이지 않은 것:
  - 돈 송금: 같은 요청 두 번 보내면 두 번 빠져나감 ← 문제!
  - 댓글 작성: 두 번 누르면 댓글 두 개 생김
```

결제는 본질적으로 멱등적이지 않습니다. 코드가 같은 결제를 두 번 실행하면 두 번 청구됩니다.

그런데 네트워크 환경에서 "두 번 실행"이 의도하지 않게 일어날 수 있습니다:

```
타임아웃 시나리오:
  1. 앱 서버가 PG에 결제 요청 전송
  2. PG가 결제 처리 중 (3초 소요)
  3. 네트워크 지연으로 앱 서버에 응답 안 옴
  4. 앱 서버: "타임아웃! 실패한 줄 알고 재시도!"
  5. 이미 결제된 상태에서 두 번째 결제 요청 전송
  → 이중 결제 발생
```

멱등성 키는 이 재시도 요청을 "아, 이건 이미 처리한 거야"라고 식별해서 중복 처리를 막습니다.

> 💡 **한 줄 요약**: 멱등성은 "같은 요청을 여러 번 해도 한 번과 같은 결과"를 보장하는 성질입니다. 결제에서 이게 없으면 네트워크 재시도로 이중 청구가 생깁니다.

---

## Q2. 결제 요청을 두 번 보내면 어떻게 되나요? 왜 중복 결제가 생길 수 있나요?

개발 환경에서는 보통 이런 일이 없어서 실감이 안 나지만, 운영 환경에서는 다양한 이유로 같은 결제가 두 번 시도될 수 있습니다.

**시나리오 1: 네트워크 타임아웃 + 재시도 로직**

```
[사용자 클릭] → [앱 서버] → [PG API 호출]
                              ↑
                         응답 3초 초과 (PG는 성공했지만 응답이 늦음)
                              ↓
                    [앱 서버: 타임아웃 → 재시도]
                              ↓
                    [PG API 두 번째 호출] → 이중 결제!
```

**시나리오 2: 사용자가 결제 버튼을 빠르게 두 번 클릭**

```
[사용자 클릭] [사용자 또 클릭]
     ↓               ↓
[요청 1]         [요청 2]  ← 거의 동시에 서버 도달
     ↓               ↓
[결제 처리]      [결제 처리]  ← 둘 다 실행되면?
```

**시나리오 3: 서버 재시작 + retry queue**

```
1. 서버가 결제 요청을 처리 중
2. 서버가 갑자기 재시작 (배포, 크래시)
3. 큐에 남은 작업을 재시작 후 재처리
4. 이미 처리됐는지 모르고 다시 실행
```

이런 상황들은 버그가 아니라 **분산 시스템의 정상적인 동작 패턴**입니다. "절대 재시도 안 함"은 현실적으로 불가능합니다. 재시도는 안정성을 위해 필요하지만, 결제에서 재시도가 이중 청구로 이어지면 안 됩니다.

> 💡 **한 줄 요약**: 이중 결제는 악의적인 공격이 아니라 네트워크 타임아웃, 재시도 로직, 서버 재시작 같은 정상적인 분산 시스템 동작에서 자연스럽게 발생합니다.

---

## Q3. idempotency_key가 어떻게 중복 결제를 막나요? 실제로 어떻게 동작하나요?

`payment_ledger` 테이블에는 `idempotency_key` 컬럼이 있고, 이 컬럼에 UNIQUE 제약이 걸려 있습니다:

```sql
CREATE TABLE payment_ledger (
  pk              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  org_pk          BIGINT NOT NULL,
  idempotency_key VARCHAR(255) NOT NULL,    -- 중복 방지 키
  type            VARCHAR(16) NOT NULL CHECK (type IN ('CHARGE','REFUND','CHARGEBACK','CREDIT')),
  amount_minor    BIGINT NOT NULL,          -- 환불은 음수 가능 → CHECK 없음
  status          VARCHAR(16) NOT NULL CHECK (status IN ('PENDING','SUCCEEDED','FAILED')),
  -- ...

  CONSTRAINT uq_idempotency_key UNIQUE (idempotency_key)  -- ← 이게 핵심
);
```

동작 원리:

```
첫 번째 결제 요청:
  idempotency_key = "pay_org42_sub7_20260528_abc123"
  → INSERT INTO payment_ledger ... (성공)
  → 결제 처리 진행

두 번째 동일 요청 (재시도):
  idempotency_key = "pay_org42_sub7_20260528_abc123"  ← 똑같은 키
  → INSERT INTO payment_ledger ... 
  → ERROR: duplicate key value violates unique constraint "uq_idempotency_key" (SQLSTATE 23505)
  → 앱이 에러를 잡아서 → "이미 처리됨"으로 인식 → 기존 결제 결과 반환
```

코드 레벨에서 어떻게 처리하는지:

```typescript
async function chargeSubscription(orgPk: number, amount: number, key: string) {
  try {
    await db.insert(paymentLedger).values({
      orgPk,
      idempotencyKey: key,
      type: 'CHARGE',
      amountMinor: amount,
      status: 'PENDING',
    });
  } catch (error) {
    // UNIQUE 제약 위반 = 이미 처리된 요청
    if (isDuplicateKeyError(error)) {
      // 기존 처리 결과를 찾아서 반환
      const existing = await db.query.paymentLedger.findFirst({
        where: eq(paymentLedger.idempotencyKey, key)
      });
      return existing; // 새로 처리하지 않고 기존 결과 반환
    }
    throw error;
  }
}
```

DB의 UNIQUE 제약이 "검사 후 INSERT" 패턴의 race condition을 막아줍니다:

```
Race condition 없는 이유:
  일반적인 방법:
    1. SELECT 해서 없으면 INSERT (틈이 있음)
    2. 동시 요청이 둘 다 SELECT → 둘 다 "없음" → 둘 다 INSERT → 중복!

  UNIQUE 제약 방법:
    → DB가 INSERT 시점에 원자적으로 보장
    → 동시 요청이 와도 하나만 성공, 나머지는 에러
    → 틈이 없음
```

> 💡 **한 줄 요약**: UNIQUE 제약이 걸린 `idempotency_key`에 같은 키로 INSERT를 시도하면 DB가 에러를 냅니다. 앱은 이 에러를 "이미 처리됨"으로 해석해서 중복 결제를 막습니다.

---

## Q4. idempotency_key를 어떻게 만들어야 하나요? 아무 랜덤값이면 되나요?

완전 랜덤값(UUID, `crypto.randomUUID()`)을 쓰면 **재시도 방지가 안 됩니다**.

```
잘못된 방법: 매번 새로운 랜덤값
  첫 번째 요청: key = "a1b2c3d4-..." → 성공
  재시도 요청: key = "e5f6g7h8-..." ← 다른 랜덤값!
  → UNIQUE 충돌 안 남 → 두 번째 결제도 처리됨 → 이중 결제!
```

핵심은 **같은 의도의 결제 요청은 같은 키를 써야 한다**는 것입니다. 그러면서 **다른 결제는 다른 키**여야 합니다.

좋은 idempotency_key 구성 방법:

```typescript
// 방법 1: 구독 주기 기반 (월 구독 결제)
function buildSubscriptionChargeKey(orgPk: number, subPk: number, periodStart: Date): string {
  const period = periodStart.toISOString().slice(0, 7); // "2026-05"
  return `charge_${orgPk}_sub${subPk}_${period}`;
  // 예: "charge_42_sub7_2026-05"
  // → 5월 결제는 항상 이 키 → 재시도해도 중복 없음
  // → 6월에는 "charge_42_sub7_2026-06" → 다른 키 → 정상 새 결제
}

// 방법 2: 사용자 세션 + 상품 기반 (1회성 결제)
function buildOneTimePaymentKey(
  orgPk: number,
  productCode: string,
  sessionId: string // 구매 세션 ID (페이지 로드 시 생성)
): string {
  return `onetime_${orgPk}_${productCode}_${sessionId}`;
}

// 방법 3: 외부 시스템 ID 활용 (PG에서 제공하는 경우)
function buildFromPgKey(pgProvider: string, pgPaymentId: string): string {
  return `${pgProvider}_${pgPaymentId}`;
  // Toss/Stripe가 이미 고유 ID를 제공하면 그걸 그대로 씀
}
```

키 설계 원칙:

```
포함해야 할 것:
  ✅ 어떤 org의 결제인지 (org_pk)
  ✅ 어떤 구독/상품인지 (sub_pk 또는 product_code)
  ✅ 어떤 기간/시점인지 (청구 월, 세션 ID 등)

피해야 할 것:
  ❌ 순수 랜덤값만 (재시도 시 다른 키 생성됨)
  ❌ 타임스탬프만 (밀리초 차이로 다른 키 생성 가능)
  ❌ 너무 짧은 키 (충돌 가능성)
```

Stripe나 Toss는 클라이언트에서 키를 생성해서 헤더로 전달하는 방식을 씁니다:

```typescript
// Stripe 방식 참고
const idempotencyKey = `sub_${subscriptionId}_${billingCycleStart}`;
const charge = await stripe.paymentIntents.create({
  amount: 29000,
  currency: 'krw',
}, {
  idempotencyKey,  // Stripe API 레벨에서도 멱등성 보장
});
```

> 💡 **한 줄 요약**: idempotency_key는 랜덤값이 아니라, "같은 청구 의도면 항상 같은 키, 다른 청구면 다른 키"가 되도록 org_pk + [[subscription-lifecycle|구독]] 정보 + 기간을 조합해서 만들어야 합니다.

---

## Q5. pg_webhook_event에도 UNIQUE(pg_provider, event_id)가 있는데, 이것도 같은 멱등성 개념인가요?

맞습니다. 같은 개념이고 같은 문제를 다른 방향에서 막습니다.

`idempotency_key`는 **우리 쪽에서 결제사에 요청할 때** 중복을 막는 것이고,  
`UNIQUE(pg_provider, event_id)`는 **결제사가 우리에게 [[webhook-processing|웹훅]]을 보낼 때** 중복을 막는 것입니다.

결제사도 webhook을 여러 번 보낼 수 있습니다:

```
왜 webhook이 중복으로 올까?:
  - 우리 서버가 일시적으로 다운 → 결제사가 재시도 (3회, 1시간 간격 등)
  - 우리 서버가 처리 중 200 OK를 안 보냄 → 결제사는 "못 받은 줄 알고" 재전송
  - 네트워크 분리 (split-brain): 실제로는 받았지만 결제사 측에선 불확실
```

이를 처리하는 구조:

```sql
CREATE TABLE pg_webhook_event (
  pg_provider  VARCHAR(16) NOT NULL CHECK (pg_provider IN ('TOSS','STRIPE','PAYPAL')),
  event_id     VARCHAR(255) NOT NULL,          -- PG가 준 고유 이벤트 ID
  status       VARCHAR(16) NOT NULL CHECK (status IN ('RECEIVED','PROCESSED','SKIPPED','FAILED')),
  -- ...

  CONSTRAINT uq_provider_event UNIQUE (pg_provider, event_id)  -- 중복 방지
);
```

처리 흐름:

```
1. Toss에서 webhook 수신 (event_id = "evt_toss_abc123")

2. pg_webhook_event에 INSERT 시도:
   INSERT INTO pg_webhook_event (pg_provider, event_id, ...)
   VALUES ('TOSS', 'evt_toss_abc123', ...)

3-A. 처음 수신:
   → INSERT 성공 → 구독 상태 처리 진행
   → status → 'PROCESSED'
   → 결제 단일 트랜잭션 실행

3-B. 동일 event_id 재수신 (중복):
   → UNIQUE 충돌 에러
   → 앱: "이미 처리한 이벤트"로 인식
   → 200 OK 반환 (PG에게 "잘 받았어"라고 해야 재전송 중단)
   → 아무 처리 없음
```

두 멱등성 보호 장치 비교:

| 구분 | idempotency_key (payment_ledger) | UNIQUE(pg_provider, event_id) |
|---|---|---|
| 방향 | 우리 → PG (아웃바운드) | PG → 우리 (인바운드) |
| 막는 것 | 우리가 PG에 같은 결제를 두 번 요청 | PG가 우리에게 같은 이벤트를 두 번 전달 |
| 키 생성자 | 우리 (앱 서버) | PG (Toss/Stripe) |
| 에러 시 응답 | 기존 결제 결과 반환 | 200 OK 반환 (PG 재전송 방지) |

두 장치가 함께 있어서 "결제됐는데 권한 두 번 활성화" 같은 상황이 막힙니다.

> 💡 **한 줄 요약**: `idempotency_key`는 우리가 PG에 요청할 때의 멱등성, `UNIQUE(pg_provider, event_id)`는 PG가 우리에게 webhook을 보낼 때의 멱등성입니다. 결제는 양방향 모두 중복 방지가 필요합니다.

---

## 테스트 방법

> 🧪 *실제 DB·ORM·운영에서 돌리는 법*: [[testing-strategy]] · [[orm-testing-drizzle]]

멱등성의 핵심 보장은 "같은 `idempotency_key`로 동시에 두 번 INSERT해도 하나만 성공하고, 두 번째는 UNIQUE 위반(SQLSTATE 23505)으로 잡혀 중복 결제가 안 된다"입니다. PostgreSQL 16 + Testcontainers(`PostgreSqlContainer`) + Drizzle(node-postgres) + vitest로 실제 UNIQUE 제약을 띄워 동시성까지 검증합니다.

검증된 테스트 패턴 3가지:
- **`recordRefund` 동일 키 재시도**: DB UNIQUE 위반(23505)으로 거부되거나, app-level 사전조회로 `DUPLICATE`(409)를 반환합니다(이중 보호 — DB 제약 + 앱 사전조회 둘 중 하나가 항상 막음).
- **동시 중복 INSERT**: `Promise.allSettled`로 동시에 던져도 정확히 1건만 성공.
- **`pg_webhook_event` 재수신**: 같은 `(pg_provider, event_id)`가 다시 와도 두 번째는 skip(처리 로직 재실행 없음).

```typescript
// idempotency-key.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";
import { PostgreSqlContainer, StartedPostgreSqlContainer } from "@testcontainers/postgresql";
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import { eq } from "drizzle-orm";

let container: StartedPostgreSqlContainer;
let db: ReturnType<typeof drizzle>;

beforeAll(async () => {
  container = await new PostgreSqlContainer("postgres:16").start();
  db = drizzle(new Pool({ connectionString: container.getConnectionUri() }));
  await migrate(db); // payment_ledger (uq_idempotency_key UNIQUE) 생성
}, 60_000);

afterAll(async () => {
  await container.stop();
});

describe("idempotency_key 중복 결제 방지", () => {
  beforeEach(async () => {
    await db.delete(paymentLedger);
  });

  it("같은 키로 두 번 INSERT하면 두 번째는 23505 (unique_violation)", async () => {
    const key = "charge_42_sub7_2026-05";
    await db.insert(paymentLedger).values({
      orgPk: 42, idempotencyKey: key, type: "CHARGE", amountMinor: 29000n, status: "PENDING",
    });

    // 두 번째 동일 키 → DB가 거부
    await expect(
      db.insert(paymentLedger).values({
        orgPk: 42, idempotencyKey: key, type: "CHARGE", amountMinor: 29000n, status: "PENDING",
      }),
    ).rejects.toMatchObject({ code: "23505" }); // unique_violation

    const rows = await db.select().from(paymentLedger).where(eq(paymentLedger.idempotencyKey, key));
    expect(rows).toHaveLength(1); // ✅ 한 건만 존재
  });

  it("동시 중복 INSERT: 둘 다 동시에 실행해도 정확히 하나만 성공", async () => {
    const key = "charge_42_sub7_2026-06";
    const insertOne = () =>
      db.insert(paymentLedger).values({
        orgPk: 42, idempotencyKey: key, type: "CHARGE", amountMinor: 29000n, status: "PENDING",
      });

    const results = await Promise.allSettled([insertOne(), insertOne()]);
    const ok = results.filter((r) => r.status === "fulfilled");
    const failed = results.filter(
      (r) => r.status === "rejected" && (r.reason as { code?: string }).code === "23505",
    );
    expect(ok).toHaveLength(1);     // 하나만 성공
    expect(failed).toHaveLength(1); // 나머지는 23505
  });

  it("recordRefund 동일 키 재시도: DB 23505 또는 app-level DUPLICATE(409) — 이중 보호", async () => {
    const key = "refund_42_pay9_2026-05";

    // recordRefund: app-level 사전조회 → 있으면 409 throw, 없으면 INSERT(REFUND, 음수 금액)
    async function recordRefund(orgPk: number, amountMinor: bigint, k: string) {
      const existing = await db.select().from(paymentLedger)
        .where(eq(paymentLedger.idempotencyKey, k));
      if (existing.length > 0) {
        throw new DuplicateError("DUPLICATE", 409); // 사전조회로 먼저 차단
      }
      // 사전조회를 통과해도 동시성 틈에서 INSERT가 23505로 거부될 수 있음
      await db.insert(paymentLedger).values({
        orgPk, idempotencyKey: k, type: "REFUND", amountMinor, status: "SUCCEEDED",
      });
    }

    await recordRefund(42, -5000n, key); // 첫 환불 성공

    // 재시도 → app-level에서 DUPLICATE(409)
    await expect(recordRefund(42, -5000n, key))
      .rejects.toMatchObject({ message: "DUPLICATE", status: 409 });

    // 사전조회를 우회한 직접 INSERT는 DB UNIQUE(23505)가 막는다 (이중 보호)
    await expect(
      db.insert(paymentLedger).values({
        orgPk: 42, idempotencyKey: key, type: "REFUND", amountMinor: -5000n, status: "SUCCEEDED",
      }),
    ).rejects.toMatchObject({ code: "23505" });

    const rows = await db.select().from(paymentLedger).where(eq(paymentLedger.idempotencyKey, key));
    expect(rows).toHaveLength(1); // ✅ 환불 한 건만
  });

  it("pg_webhook_event 재수신: 같은 (pg_provider, event_id)는 두 번째 skip", async () => {
    const evt = { pgProvider: "TOSS" as const, eventId: "evt_toss_abc123" };

    // 인바운드 멱등성: (pg_provider, event_id) UNIQUE → ON CONFLICT DO NOTHING
    const first = await db.insert(pgWebhookEvent)
      .values({ ...evt, status: "RECEIVED" })
      .onConflictDoNothing({ target: [pgWebhookEvent.pgProvider, pgWebhookEvent.eventId] })
      .returning();
    const second = await db.insert(pgWebhookEvent)
      .values({ ...evt, status: "RECEIVED" })
      .onConflictDoNothing({ target: [pgWebhookEvent.pgProvider, pgWebhookEvent.eventId] })
      .returning();

    expect(first).toHaveLength(1);  // 처음 수신 → 처리 진행
    expect(second).toHaveLength(0); // 재수신 → skip (처리 로직 재실행 없이 200 OK)

    const rows = await db.select().from(pgWebhookEvent);
    expect(rows).toHaveLength(1); // ✅ 이벤트 한 건만
  });
});
```

---

## 마치며

멱등성은 처음에는 과한 것처럼 보이지만, 운영 환경에서는 필수입니다.

```
결제 시스템의 멱등성 2중 보호:

[앱 서버] ──idempotency_key──▶ [PG 결제 요청]
    ↑                                    |
    └──UNIQUE(provider,event_id)──── [PG Webhook 수신]
```

코드 짤 때 체크리스트:

```typescript
// 결제 처리 전: idempotency_key 생성
const key = buildSubscriptionChargeKey(orgPk, subPk, periodStart);
// ← 재시도해도 같은 key가 나와야 함

// INSERT 후: duplicate key error를 정상 흐름으로 처리
try {
  await insertPaymentLedger({ idempotencyKey: key, ... });
} catch (e) {
  if (isDuplicateKeyError(e)) return getExistingResult(key);
  throw e;
}

// webhook 처리 시: 중복 event_id는 200 OK만 반환
// (절대 처리 로직을 다시 실행하면 안 됨)
```

결제는 "한 번만 정확하게"가 핵심입니다. `idempotency_key`와 webhook UNIQUE 제약이 그 보장을 DB 레벨에서 합니다.

**왜 Redis가 아니라 DB UNIQUE인가** — 멱등 저장소로 Redis `SETNX`도 흔히 쓰지만, 결제처럼 정확성·내구성이 중요한 경로에서는 DB UNIQUE 제약을 택했습니다.

| 항목 | DB UNIQUE (우리) | Redis SETNX |
|---|---|---|
| 구현 단순성 | 높음 (컬럼 추가만) | 중간 (Redis 연결 관리) |
| 내구성 | DB에 영구 저장 | Redis 재시작 시 소실 |
| 정확성 | 완벽 | TTL 만료 후 재처리 가능 |
| 의존성 | DB만 | DB + Redis |

---

## 연결된 개념

- [[subscription-lifecycle|구독 상태 머신]] — 구독 갱신 결제에서 idempotency_key가 쓰이는 시점
- [[webhook-processing|PG 웹훅 처리]] — PG에서 오는 event_id의 인바운드 멱등성
- [[outbox-pattern|Outbox 패턴]] — 결제 성공 후 비동기 처리의 at-least-once 보장
> 소스 문서
- [[schema-reference]] — D.17 payment_ledger (idempotency_key UNIQUE), D.18 pg_webhook_event (event_id UNIQUE)
