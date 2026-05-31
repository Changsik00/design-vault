---
type: decision
status: 채택
aliases:
  - 결제 단일 트랜잭션
  - payment atomicity
  - Kafka 거부
tags:
  - platform-db
  - decision
  - billing
  - transaction
  - consistency
---

# 결제↔권한 단일 트랜잭션

## 배경

결제가 성공했는데 권한이 풀리지 않는 상황:

```
[사용자 화면]
  "결제가 완료됐습니다" → 새로고침 → "구독이 필요합니다"

[실제 일어난 일]
  PG 결제 성공 → webhook 발송 → 서버 처리 중 crash
                                  ↑
                         권한 갱신이 여기서 실패
```

SaaS에서 "결제됐는데 서비스 못 씀"은 가장 심각한 UX 실패다. 이 문제를 어떻게 구조적으로 없앨 것인가.

---

## 일반 권고 — Kafka + eventual consistency

```
PG webhook → Kafka topic → 결제 워커 → org_subscription 갱신
                        → 권한 워커 → org_entitlement 갱신
                        → 알림 워커 → 이메일 발송
```

**권고 근거**:
- 서비스 독립: 워커 하나 죽어도 다른 워커 영향 없음
- 재시도 내장: Kafka consumer group이 실패 메시지를 자동 재처리
- 확장성: 워커를 수평 증설 가능

**문제점**:
- 결제 워커와 권한 워커는 독립 실행 → 결제는 됐는데 권한 워커가 늦으면 "결제됐는데 권한 미반영" 창이 생김
- Kafka 브로커 장애 = 전체 흐름 장애
- at-least-once delivery → 멱등성을 모든 워커에 따로 구현해야 함
- 현 규모(서비스 1~2개)에서 Kafka 운영 오버헤드가 너무 큼

---

## 우리 결정 — 단일 Postgres 트랜잭션

결제와 권한 갱신을 **하나의 트랜잭션**으로 묶는다.

```sql
BEGIN;
  -- 1. 결제 원장 기록 (append-only, 멱등키로 중복 방지)
  INSERT INTO payment_ledger
    (org_pk, type, amount_minor, idempotency_key, status)
  VALUES (?, 'CHARGE', ?, ?, 'SUCCEEDED');

  -- 2. 구독 상태 갱신
  UPDATE org_subscription
  SET status='ACTIVE', current_period_end=?
  WHERE pk=?;

  -- 3. 권한 활성화 (upsert)
  INSERT INTO org_entitlement (org_pk, service, status, feature_limits, valid_until)
  VALUES (?, ?, 'ACTIVE', ?, ?)
  ON CONFLICT (org_pk, service) DO UPDATE SET
    status='ACTIVE', feature_limits=EXCLUDED.feature_limits, valid_until=EXCLUDED.valid_until;

  -- 4. 권한 변경 전파 (클라이언트 캐시 무효화)
  UPDATE organization SET perm_version = perm_version + 1 WHERE pk=?;

  -- 5. async 부수효과만 outbox에 (이메일·알림·분석은 나중에)
  INSERT INTO outbox_event (event_type, aggregate_pk, payload_json)
  VALUES ('subscription.activated', ?, ?);
COMMIT;
-- 여기서 commit이 성공하면 결제+권한이 동시에 반영됨이 보장됨
```

> 🐬 **MySQL이라면**: 단일 InnoDB 트랜잭션으로 동일하게 구현된다. UPSERT는 `ON CONFLICT … DO UPDATE` 대신 `ON DUPLICATE KEY UPDATE c=VALUES(c)` 구문을 쓴다.

**채택 근거**:
1. **"결제됐는데 권한 미반영" 창이 0**: BEGIN~COMMIT 사이에 서버가 죽으면 전체 rollback → 일관된 상태 유지
2. **같은 Postgres 인스턴스**: 분산 트랜잭션(2PC), Kafka 없이 단순 트랜잭션 가능
3. **멱등성 구현 위치 단순화**: `idempotency_key UNIQUE` + `UNIQUE(pg_provider, event_id)` 두 곳만
4. **async 부수효과는 outbox**: 이메일·알림·검색 인덱싱만 eventual consistency — 권한은 동기

---

## 트레이드오프

| 항목 | Kafka + eventual | 단일 트랜잭션 (우리) |
|---|---|---|
| 결제↔권한 일관성 | 보장 안 됨 (워커 지연) | **항상 보장** |
| 인프라 복잡도 | 높음 (Kafka 브로커) | 낮음 (Postgres만) |
| 워커 독립 확장 | 가능 | 불가 (platform_db 중심) |
| 현 규모 적합성 | 과도 | 적합 |
| MQ 장애 전파 | 있음 | **없음** |

---

## 향후 조건

아래 조건이 충족되면 Kafka/Debezium CDC 도입 재검토:

1. ISMS-P / SOC2 감사 요건 충족 필요 (T4 트리거)
2. platform_db 단일 지점 부하가 감당 불가한 수준
3. 결제·권한 처리를 담당하는 전담 팀 분리

도입 시: outbox_event → Debezium CDC → Kafka로 전환. 인터페이스 동일 유지.

---

## 관련 문서

| 파일 | 내용 |
|---|---|
| [[schema-reference]] §F.1 | 결제 단일 트랜잭션 SQL 전문 |
| [[architecture]] §1.3 | 결제 단일 트랜잭션 설계 원칙 |
| [[auth-projection]] | 이 트랜잭션에서 org_entitlement가 투영되는 이유 |
| [[outbox-pattern]] | outbox_event가 async 부수효과를 처리하는 방법 |
| [[idempotency-key]] | idempotency_key로 중복 결제를 막는 방법 |
