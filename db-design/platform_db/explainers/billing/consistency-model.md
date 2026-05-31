---
difficulty: 고
tags:
  - platform-db
  - explainer
  - p0
  - consistency
  - transaction
  - acid
  - postgres
aliases:
  - 일관성 모델
  - strong consistency
  - eventual consistency
  - 강한 일관성
  - 결과적 일관성
  - ACID 트랜잭션
---

# 강한 일관성 vs 결과적 일관성 (왜 단일 트랜잭션 + outbox)

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[architecture|architecture.md §1.3 · 불변식 #7]] · [[schema-reference|schema-reference.md §F.1]] · [[payment-atomicity]]

"결제 성공인데 권한이 안 풀린다"는 SaaS에서 가장 치명적인 버그입니다. 이걸 구조적으로 없애려면 어떤 데이터는 **쓰자마자 즉시 일치**해야 하고(강한 일관성), 어떤 데이터는 **조금 늦어도 괜찮습니다**(결과적 일관성). 이 문서는 그 둘을 구분하고, 왜 `platform_db`가 결제↔권한은 **단일 트랜잭션**으로, 알림 같은 부수효과는 **outbox**로 처리하는지 — 그리고 왜 2PC와 Kafka를 일부러 쓰지 않는지 설명합니다.

---

## Q1. 강한 일관성(strong consistency)이 뭔가요? 왜 결제↔권한엔 이게 필수인가요?

**강한 일관성**은 "쓰자마자 읽으면 항상 최신 값이 나온다"는 보장입니다. 영어로는 **read-after-write consistency**라고도 부릅니다. 쓰기가 끝난 순간부터 그 값을 읽는 모든 주체는 예외 없이 새 값을 봅니다.

비유하자면 칠판입니다. 내가 칠판에 "3"을 적으면, 교실 안 누가 봐도 그 즉시 "3"입니다. "방금 적었는데 뒷자리 사람은 아직 2로 본다" 같은 일이 없습니다.

우리 시스템에서 결제와 권한이 정확히 이래야 합니다:

```
결제 성공 → org_entitlement 활성화 → perm_version 증가
   ↓
사용자가 바로 새로고침 → "구독 활성" 상태가 즉시 보여야 함
```

만약 강한 일관성이 깨지면 이런 화면이 나옵니다:

```
[사용자 화면]
  "결제가 완료됐습니다" → 새로고침 → "구독이 필요합니다"
                                       └─ 권한이 아직 반영 안 됨 (불일치 창)
```

결제는 됐는데 서비스를 못 쓰는 상태 — SaaS UX 최악의 실패입니다. 그래서 결제↔권한 경로는 **불일치 창(window)이 0초**여야 합니다. "잠깐 뒤엔 맞겠지"가 허용되지 않습니다.

```typescript
// 우리가 보장하려는 것
await chargeAndActivate(orgPk, paymentInfo);
// ✅ 이 줄이 끝난 직후, 권한 조회는 반드시 ACTIVE
const ent = await getEntitlementByService(orgPk, "ACADEMY");
ent.status === "ACTIVE"; // 항상 true (불일치 창 없음)
```

> 💡 **한 줄 요약**: 강한 일관성은 "쓴 즉시 모두가 최신 값을 본다"는 보장이고, 결제↔권한은 "결제됐는데 권한 미반영"이 단 1초도 있으면 안 되므로 이게 필수입니다.

---

## Q2. 결과적 일관성(eventual consistency)은 뭔가요? 그건 언제 충분한가요?

**결과적 일관성**은 "지금 당장은 불일치할 수 있지만, 시간이 지나면(결과적으로) 일치한다"는 보장입니다. 쓰기 직후 잠깐은 옛 값이 보일 수 있어도, 곧 새 값으로 수렴합니다.

비유하자면 단체 채팅방 알림입니다. 누군가 메시지를 보내면 내 폰엔 즉시 뜨지만, 친구 폰엔 네트워크 사정에 따라 몇 초 늦게 뜰 수 있습니다. 결국 둘 다 같은 메시지를 보게 되지만, 그 "몇 초"는 아무도 문제 삼지 않습니다.

우리 시스템에서 결제의 **부수효과(side effect)**가 정확히 이렇습니다:

```
결제 성공의 부수효과:
  - 결제 완료 이메일 발송      → 3초 늦어도 OK
  - 푸시 알림                  → 5초 늦어도 OK
  - 영수증 생성                → 1분 늦어도 OK
  - 분석/집계 이벤트 기록       → 10분 늦어도 OK
```

이메일이 3초 늦게 가도 사용자는 불편하지 않습니다. 하지만 권한이 3초 늦게 풀리면 "결제했는데 못 쓴다"는 컴플레인이 들어옵니다. **같은 결제 이벤트인데, 데이터마다 요구하는 일관성 수준이 다릅니다.**

```
일관성 레벨 선택 기준:
  "이 데이터가 잠깐 옛 값이면 사용자가 피해를 보나?"

  YES → 강한 일관성 (권한, 잔액, 재고)
  NO  → 결과적 일관성으로 충분 (알림, 영수증, 분석, 검색 인덱스)
```

핵심은, 결과적 일관성은 "더 나쁜 것"이 아니라 **부수효과에 알맞은 도구**라는 점입니다. 모든 걸 강한 일관성으로 묶으면 결제 응답이 외부 API 속도에 발목 잡힙니다(자세한 건 [[outbox-pattern]] Q2).

> 💡 **한 줄 요약**: 결과적 일관성은 "잠깐 늦어도 결국 맞는다"는 보장이고, 알림·영수증·분석처럼 몇 초 늦어도 사용자가 손해 보지 않는 부수효과엔 이걸로 충분합니다.

---

## Q3. ACID 트랜잭션이 강한 일관성을 어떻게 보장하나요? 특히 원자성(atomicity)이 핵심인가요?

네, **ACID 트랜잭션**이 단일 DB가 강한 일관성을 보장하는 무기입니다. ACID는 네 가지 보장의 머리글자입니다:

```
A - Atomicity (원자성)   : 전부 성공 아니면 전부 실패. 중간은 없음
C - Consistency (일관성) : 트랜잭션 전후로 제약(NOT NULL, UNIQUE, FK)이 항상 유효
I - Isolation (고립성)   : 동시에 도는 트랜잭션끼리 서로의 중간 상태를 안 봄
D - Durability (지속성)  : COMMIT되면 서버가 죽어도 데이터 유지 (디스크 기록)
```

이 중 결제↔권한에서 **가장 결정적인 건 원자성(Atomicity)**입니다. 원자성은 "여러 작업을 쪼갤 수 없는 하나의 덩어리로 묶는다"는 뜻입니다.

비유는 **은행 계좌이체**입니다. A 계좌에서 1만 원 출금하고 B 계좌에 1만 원 입금하는 건 한 묶음이어야 합니다:

```
출금 성공 → 입금 직전 크래시
  → ❌ 돈이 허공으로 사라짐 (출금만 됨)

원자성이 있으면:
  출금 + 입금이 한 덩어리 → 둘 다 되거나, 둘 다 안 되거나
  → 입금이 실패하면 출금도 자동 취소(rollback) → 돈이 사라지지 않음
```

우리 결제도 똑같습니다. `payment_ledger` 기록 + `org_entitlement` 활성화 + `perm_version` 증가가 **쪼갤 수 없는 한 덩어리**여야 합니다:

```sql
-- 이체처럼 "전부 또는 전무"로 묶기
BEGIN;
  INSERT INTO payment_ledger (...) VALUES (..., 'SUCCEEDED');  -- 결제 기록
  UPDATE org_subscription SET status='ACTIVE' WHERE pk=?;       -- 구독 활성
  INSERT INTO org_entitlement (...) ON CONFLICT (org_pk, product_code) DO UPDATE SET ...; -- 권한 활성
  UPDATE organization SET perm_version = perm_version + 1 WHERE pk=?; -- 전파
COMMIT;
-- COMMIT 성공 = 네 줄 모두 확정. 중간 크래시 = 네 줄 모두 rollback
```

```typescript
// ✅ 원자성 보장 — 하나라도 실패하면 전부 되돌림
await db.transaction(async (tx) => {
  await tx.insert(paymentLedger).values(...);
  await tx.update(orgSubscription)...;
  await tx.insert(orgEntitlement)...;
  await tx.update(organization).set({ permVersion: sql`perm_version + 1` })...;
}); // 콜백이 throw하면 자동 ROLLBACK → 부분 반영 없음

// ❌ 원자성 없는 코드 — 각각 따로 커밋되면 중간 크래시에 취약
await db.insert(paymentLedger).values(...);  // 여기까지 커밋됨
// ...서버 크래시...
await db.insert(orgEntitlement)...;          // 영영 실행 안 됨 → 결제만 되고 권한 없음
```

원자성 덕분에 "결제는 됐는데 권한은 안 된" 중간 상태가 물리적으로 존재할 수 없습니다. 그래서 강한 일관성이 성립합니다.

> 💡 **한 줄 요약**: ACID의 원자성(Atomicity)은 결제+권한을 "전부 또는 전무"의 한 덩어리로 묶어, 계좌이체처럼 중간 크래시에도 불일치 상태가 생기지 않게 합니다.

---

## Q4. 우리 설계는 정확히 뭘 강하게, 뭘 결과적으로 처리하나요? (단일 트랜잭션 + outbox)

`platform_db`의 핵심 원칙은 architecture.md 불변식 #7에 한 줄로 박혀 있습니다:

> **strong consistency = 단일 서버 트랜잭션. async = outbox.**

즉 데이터를 두 갈래로 나눕니다:

```
결제 이벤트가 들어오면...

[강한 일관성 — 단일 Postgres 트랜잭션] (§F.1)
  ├─ payment_ledger INSERT  (결제 원장, append-only)
  ├─ org_subscription UPDATE (구독 → ACTIVE)
  ├─ org_entitlement UPSERT  (권한 활성화) ← 사용자가 즉시 봐야 함
  └─ perm_version + 1         (클라이언트 캐시 무효화 신호)
       → 전부 COMMIT 또는 전부 ROLLBACK (원자성)

[결과적 일관성 — outbox]  ([[outbox-pattern]])
  └─ outbox_event INSERT (status='PENDING') ← 같은 트랜잭션 안에서!
       → COMMIT 후 별도 워커가 비동기로:
          이메일 발송 · 푸시 알림 · 영수증 · 분석
```

여기서 영리한 점은, **outbox_event INSERT도 같은 트랜잭션 안에 넣는다**는 것입니다. "이메일을 보내야 한다"는 *기록*은 강하게(원자적으로) 남기되, 실제 *발송*은 결과적으로(나중에) 합니다.

```sql
BEGIN;
  -- 강한 일관성 영역: 사용자가 즉시 봐야 하는 것
  INSERT INTO payment_ledger (...) VALUES (..., 'SUCCEEDED');
  UPDATE org_subscription SET status='ACTIVE' WHERE pk=?;
  INSERT INTO org_entitlement (...) ON CONFLICT (org_pk, product_code) DO UPDATE SET status='ACTIVE', ...;
  UPDATE organization SET perm_version = perm_version + 1 WHERE pk=?;

  -- "부수효과를 해야 함"이라는 기록도 강하게 남김 (발송은 나중)
  INSERT INTO outbox_event (event_type, aggregate_pk, payload_json, status)
  VALUES ('subscription.activated', ?, ?, 'PENDING');
COMMIT;
-- COMMIT 성공 = 권한 활성 + "이메일 보내야 함" 기록이 동시에 확정
```

```
이게 주는 두 보장:

① 권한은 강하게: COMMIT 직후 사용자는 ACTIVE를 본다 (불일치 창 0)
② 부수효과는 안 새지만 결과적으로: outbox에 기록됐으니 워커가
   언젠가 반드시 처리한다. 서버가 발송 직전 죽어도 PENDING으로 남아 재시도됨
```

핵심: **권한(동기)과 알림(비동기)을 같은 결제 트랜잭션에서 갈라낸다**는 것. 권한은 칠판처럼 즉시, 알림은 채팅 알림처럼 곧. outbox 워커의 상세 동작(폴링, at-least-once, FOR UPDATE SKIP LOCKED)은 [[outbox-pattern]]에서 다룹니다.

> 💡 **한 줄 요약**: 결제→권한→perm_version은 단일 Postgres 트랜잭션으로 강하게(즉시 일치), 알림·영수증은 같은 트랜잭션에 outbox_event로 기록만 해두고 워커가 비동기로(결과적으로) 처리합니다.

---

## Q5. 왜 2PC(분산 트랜잭션)나 Kafka를 안 쓰나요? 보통 그렇게 하지 않나요?

교과서적 권고는 보통 이렇습니다:

```
PG webhook → Kafka topic → 결제 워커 → org_subscription 갱신
                        → 권한 워커 → org_entitlement 갱신
                        → 알림 워커 → 이메일 발송
```

좋아 보이지만, 이 구조에서는 결제 워커와 권한 워커가 **독립 실행**됩니다. 결제는 처리됐는데 권한 워커가 0.5초 늦으면 그 사이 "결제됐는데 권한 미반영" 창이 생깁니다. Q1에서 절대 안 된다고 한 바로 그 상황입니다.

**그럼 2PC(Two-Phase Commit, 2단계 커밋)로 묶으면?** 2PC는 여러 시스템을 하나의 분산 트랜잭션으로 원자적으로 묶는 프로토콜입니다("준비됐니?" → 전원 OK → "커밋해"). 하지만:

```
2PC의 문제:
  - 느림: 모든 참여자에게 2번 왕복(prepare + commit) → 지연 증가
  - 취약: 코디네이터가 commit 직전 죽으면 참여자들이 락을 쥔 채 무한 대기
          (blocking 문제) → 가용성 저하
  - 운영 부담: 분산 트랜잭션 매니저(XA) 구성·디버깅이 복잡
```

**그런데 — 우리에겐 애초에 묶을 "여러 시스템"이 없습니다.** identity(사용자·조직)도 billing(결제·구독·권한)도 **같은 `platform_db`(단일 Postgres 인스턴스)**에 있습니다([[identity-billing-access]]).

```
일반적 상황 (2PC/Kafka가 필요한 이유)
  결제 DB ─────┐
              ├── 서로 다른 DB라 단일 트랜잭션 불가 → 2PC나 이벤트로 묶어야 함
  권한 DB ─────┘

우리 상황
  ┌─────────────── platform_db (단일 Postgres) ───────────────┐
  │  payment_ledger · org_subscription · org_entitlement     │
  │  → 같은 인스턴스 → 그냥 BEGIN; ... COMMIT; 한 방이면 끝   │
  └──────────────────────────────────────────────────────────┘
```

같은 DB 안의 여러 테이블은 **로컬 트랜잭션** 하나로 원자적으로 묶입니다. 2PC도, Kafka도 필요 없습니다. 굳이 쓰면 **과한 분산(over-distribution)** — 하나면 될 일을 셋으로 쪼개 복잡도만 늘립니다.

| 항목 | Kafka + eventual | 2PC (분산 트랜잭션) | 단일 트랜잭션 (우리) |
|---|---|---|---|
| 결제↔권한 일관성 | 보장 안 됨(워커 지연) | 보장되나 느림 | **항상 보장, 빠름** |
| 인프라 복잡도 | 높음(브로커 운영) | 높음(XA 매니저) | 낮음(Postgres만) |
| 장애 전파 | MQ 장애 = 전체 장애 | 코디네이터 죽으면 blocking | **없음** |
| 현 규모 적합성 | 과도 | 과도 | 적합 |

그리고 이 단순함을 떠받치는 또 하나의 설계가 [[auth-projection]]입니다. billing의 복잡한 로직(플랜·할인·청구주기)을 auth가 직접 다루지 않고, 그 *결과*만 `org_entitlement`로 **투영(projection)**받습니다. 덕분에 권한 판정(Gate B, [[gate-b-entitlement]])은 billing 복잡도와 분리된 채 한 테이블만 읽으면 되고, 결제 트랜잭션도 단순하게 유지됩니다.

> 💡 **한 줄 요약**: identity·billing이 같은 platform_db에 있어 로컬 트랜잭션 하나면 충분합니다. 2PC는 느리고 blocking에 취약하며 Kafka는 복잡도를 키울 뿐 — 같은 DB 안에선 둘 다 과한 분산입니다.

---

## Q6. CAP 정리는 이 결정과 무슨 상관인가요? 우리는 그 트레이드오프를 어떻게 피하나요?

**CAP 정리**는 분산 시스템에 관한 유명한 원칙입니다. 데이터가 여러 노드에 분산돼 있을 때, 다음 셋을 동시에 다 가질 수 없다는 내용입니다:

```
C - Consistency        : 모든 노드가 같은 최신 데이터를 본다
A - Availability        : 요청하면 항상 응답한다
P - Partition tolerance : 노드 간 네트워크가 끊겨도 시스템이 동작한다

분산 환경에선 네트워크 단절(P)이 현실이므로, 단절 순간엔
C와 A 중 하나를 포기해야 한다:
  CP 선택 → 일관성 지키려 일부 요청 거부 (가용성 희생)
  AP 선택 → 응답은 하되 옛 데이터 줄 수 있음 (일관성 희생)
```

여기서 중요한 통찰: **CAP의 트레이드오프는 "데이터가 분산돼 있을 때"만 발생합니다.** 노드가 여럿이라 노드 사이가 끊길 수 있어야 P 문제가 생기고, 그제서야 C냐 A냐를 골라야 합니다.

우리는 결제·권한 데이터가 **단일 DB 인스턴스**에 있습니다. 묶을 노드가 하나뿐이니 그 사이가 끊길 일이 없고, **CAP 트레이드오프 자체를 회피**합니다.

```
분산 시스템 (CAP 트레이드오프 발생)
  노드A ─╳─ 노드B   ← 네트워크 단절 시 C 또는 A 포기 강제

우리 (단일 DB → CAP 회피)
  [platform_db 한 곳]  ← 쪼개진 노드가 없음 → ACID 트랜잭션으로
                          C(강한 일관성)와 A(응답)를 둘 다 누림
```

정리하면 두 세계의 도구가 다릅니다:

```
단일 DB 안  → 무기는 ACID 트랜잭션 (원자성으로 강한 일관성)
분산 시스템 → 무기는 CAP 트레이드오프 관리 + eventual consistency
```

우리는 핵심(결제↔권한)을 단일 DB에 두어 ACID로 강하게 처리하고, **나중에 정말 분산이 필요해지면**(예: usage_log가 폭증해 OLAP로 이관, 또는 ISMS-P로 물리 격리가 필요해질 때) 그 부분만 떼어내 eventual consistency를 받아들입니다. 즉 CAP 트레이드오프를 *지금* 떠안지 않고, 필요한 시점에 필요한 데이터에 한해 선택적으로 받아들이는 전략입니다(분리 트리거 T1~T4는 [[multitenancy-rls]] 참고).

> 💡 **한 줄 요약**: CAP의 C-vs-A 딜레마는 데이터가 분산됐을 때만 생깁니다. 결제·권한을 단일 DB에 두면 묶을 노드가 하나뿐이라 CAP 트레이드오프를 통째로 회피하고, ACID로 강한 일관성을 그냥 누립니다.

---

## 용어 정리

| 용어 | 영어 | 뜻 | 우리 시스템에서 |
|---|---|---|---|
| 강한 일관성 | strong consistency | 쓴 즉시 모두가 최신 값을 봄 (read-after-write) | 결제→권한→perm_version |
| 결과적 일관성 | eventual consistency | 잠깐 불일치해도 결국 수렴 | 알림·영수증·분석 (outbox) |
| read-after-write | — | 쓴 직후 읽으면 반드시 그 값이 나옴 | 결제 후 새로고침 시 ACTIVE 보장 |
| ACID | — | DB 트랜잭션의 4대 보장 | 단일 Postgres 트랜잭션의 근거 |
| 원자성 | Atomicity | 전부 성공 아니면 전부 실패 (중간 없음) | 결제+권한 한 덩어리(§F.1) |
| 일관성(ACID) | Consistency | 제약(NOT NULL/UNIQUE/FK)이 항상 유효 | 스키마 불변식 유지 |
| 고립성 | Isolation | 동시 트랜잭션이 서로 중간 상태를 안 봄 | 동시 결제 처리 안전 |
| 지속성 | Durability | COMMIT 후 크래시에도 데이터 유지 | 결제 기록 디스크 보존 |
| 트랜잭션 | transaction | BEGIN~COMMIT으로 묶인 작업 단위 | `db.transaction(...)` |
| 롤백 | rollback | 트랜잭션 전체를 되돌림 | 중간 실패 시 부분 반영 제거 |
| 2PC | Two-Phase Commit | 여러 시스템을 묶는 분산 트랜잭션 프로토콜 | **안 씀** (단일 DB라 불필요) |
| 분산 트랜잭션 | distributed transaction | 둘 이상의 DB/시스템에 걸친 원자적 처리 | **회피** |
| CAP 정리 | CAP theorem | 분산 시 C·A·P 중 둘만 택함 | 단일 DB라 트레이드오프 회피 |
| outbox | transactional outbox | 부수효과를 DB에 기록 후 워커가 비동기 발송 | `outbox_event` ([[outbox-pattern]]) |
| 투영 | projection | billing 결과를 auth용 권한 테이블로 복사 | `org_entitlement` ([[auth-projection]]) |
| 불일치 창 | inconsistency window | 두 데이터가 어긋나 있는 시간 구간 | 결제↔권한은 0이어야 함 |

---

## 테스트 방법
> 🧪 *실제 DB·ORM·운영에서 돌리는 법*: [[testing-strategy]] · [[orm-testing-drizzle]]


핵심 두 가지를 검증합니다. **(1)** 트랜잭션 중간 실패 시 전부 rollback되어 `payment_ledger`도 `org_entitlement`도 안 생기는가(원자성=강한 일관성). **(2)** 부수효과는 COMMIT 후 별도 워커가 소비하는가(결과적 일관성).

```typescript
// payment-consistency.test.ts
import { describe, it, expect, beforeEach } from "vitest";
import request from "supertest";
import { app } from "../src/app";
import { db } from "../src/db";

describe("결제↔권한 강한 일관성 (단일 트랜잭션)", () => {
  beforeEach(async () => {
    await db.delete(paymentLedger);
    await db.delete(orgEntitlement);
    await db.delete(outboxEvent);
  });

  it("정상: 결제 성공 시 권한이 같은 트랜잭션에서 즉시 활성화된다 (read-after-write)", async () => {
    await request(app)
      .post("/internal/payments/webhook")
      .send({ orgPk: 1, eventId: "evt_001", amount: 10000, status: "PAID" })
      .expect(200);

    // 쓴 즉시 읽어도 ACTIVE — 불일치 창 0
    const ent = await getEntitlementByService(1, "ACADEMY");
    expect(ent?.status).toBe("ACTIVE");
    const ledger = await db.query.paymentLedger.findFirst({
      where: eq(paymentLedger.idempotencyKey, "evt_001"),
    });
    expect(ledger?.status).toBe("SUCCEEDED");
  });

  it("원자성: 권한 UPSERT 단계에서 실패하면 결제 원장까지 전부 rollback된다", async () => {
    // org_entitlement UPDATE에서 강제로 throw하도록 주입
    const spy = vi
      .spyOn(entitlementRepo, "upsert")
      .mockRejectedValueOnce(new Error("DB 실패 주입"));

    await expect(chargeAndActivate(1, { eventId: "evt_002", amount: 10000 }))
      .rejects.toThrow();

    // ✅ 핵심 단언: 트랜잭션 전체가 롤백 → 둘 다 존재하지 않아야 함
    const ledger = await db.query.paymentLedger.findFirst({
      where: eq(paymentLedger.idempotencyKey, "evt_002"),
    });
    const ent = await getEntitlementByService(1, "ACADEMY");
    expect(ledger).toBeUndefined();      // 결제 원장 안 생김
    expect(ent).toBeUndefined();         // 권한도 안 생김
    // → "결제만 되고 권한 미반영" 같은 중간 상태가 물리적으로 없음

    spy.mockRestore();
  });

  it("결과적 일관성: 부수효과는 COMMIT 후 outbox에 PENDING으로 남고, 발송은 워커가 한다", async () => {
    await chargeAndActivate(1, { eventId: "evt_003", amount: 10000 });

    // COMMIT 직후엔 아직 이메일이 안 나갔어야 함 (동기 발송 금지)
    expect(emailService.send).not.toHaveBeenCalled();

    // 대신 outbox에 PENDING 기록만 존재
    const evt = await db.query.outboxEvent.findFirst({
      where: eq(outboxEvent.eventType, "subscription.activated"),
    });
    expect(evt?.status).toBe("PENDING");

    // 별도 워커가 돌면 그제서야 발송되고 SENT로 마킹 (결과적으로 일치)
    await processOutboxEvents();
    expect(emailService.send).toHaveBeenCalledOnce();
    const after = await db.query.outboxEvent.findFirst({
      where: eq(outboxEvent.pk, evt!.pk),
    });
    expect(after?.status).toBe("SENT");
  });
});
```

**체크리스트** — 새 비즈니스 흐름을 만들 때:

```
[ ] 사용자가 즉시 봐야 하는 데이터(권한·잔액 등)는 핵심 트랜잭션 안에서 갱신했는가?
[ ] 외부 API 호출(이메일·알림·분석)을 트랜잭션 안/직후에 직접 호출하지 않았는가?
[ ] 부수효과는 outbox_event INSERT로만 기록했는가? (같은 트랜잭션 안에서)
[ ] 트랜잭션 중간 실패 시 전부 rollback되는지 테스트가 있는가?
[ ] "결제만 되고 권한 미반영" 같은 중간 상태가 단언으로 배제되는가?
[ ] 굳이 2PC/Kafka를 끌어들이고 있지 않은가? (같은 DB면 로컬 트랜잭션이면 충분)
```

---

## 마치며

일관성 모델은 결국 하나의 질문으로 귀결됩니다: **"이 데이터가 잠깐 옛 값이면 사용자가 손해를 보는가?"**

- **예** → 강한 일관성. 핵심 트랜잭션 안에서 동기로 처리합니다(결제·권한·잔액). 무기는 ACID 원자성, 매개는 단일 Postgres 트랜잭션(§F.1).
- **아니오** → 결과적 일관성으로 충분합니다(알림·영수증·분석). 매개는 outbox, 발송은 워커가 비동기로.

`platform_db`가 2PC도 Kafka도 쓰지 않는 이유는 단순합니다. identity와 billing이 같은 DB에 있으니 로컬 트랜잭션 하나면 강한 일관성이 거저 따라오고, CAP 트레이드오프도 회피됩니다. 분산은 "할 수 있어서" 하는 게 아니라 "꼭 필요할 때" 트리거에 맞춰 들이는 것입니다. 그때가 오면 outbox가 이미 깔아둔 토대(`outbox_event → CDC → Kafka`) 위로 자연스럽게 확장하면 됩니다.

새 기능에서 외부 호출을 트랜잭션에 끼워 넣고 싶을 때, 또는 "그냥 Kafka 쓰면 안 되나?"라는 생각이 들 때 — 이 문서를 떠올리세요.

---

## 연결된 개념

- [[outbox-pattern|Outbox 패턴]] — 부수효과(결과적 일관성)를 워커가 비동기 처리하는 방법
- [[auth-projection|권한 투영]] — billing 복잡도를 auth에서 격리해 결제 트랜잭션을 단순하게 유지
- [[gate-b-entitlement|Gate B 권한 판정]] — 강하게 갱신된 org_entitlement를 핫패스에서 읽는 곳
- [[idempotency-key|멱등성 키]] — 같은 결제 webhook이 중복 와도 트랜잭션이 한 번만 반영되게 함
- [[multitenancy-rls|멀티테넌시 & RLS]] — 단일 DB를 유지하는 Pool 모델과 분산 트리거(T1~T4)
- [[webhook-processing|PG 웹훅 처리]] — 이 강한 일관성 트랜잭션을 발동시키는 인바운드 webhook
> 소스 문서
- [[architecture]] — §1.3 데이터 일관성(billing→projection→auth), §2.1 불변식 #7 (strong consistency = 단일 서버 트랜잭션, async = outbox)
- [[schema-reference]] — §F.1 결제→권한 단일 트랜잭션 SQL 전문
- [[payment-atomicity]] — Kafka/2PC 대신 단일 Postgres 트랜잭션을 택한 결정과 트레이드오프
- [[requirements]] — 결제↔권한 원자성 요구사항(ARCH-2)
- [[e2e-journeys]] — 결제 성공 후 권한 즉시 반영 사용자 여정
