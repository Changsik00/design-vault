---
difficulty: 고
tags:
  - platform-db
  - explainer
  - p2
  - db-ops
  - postgres
  - ddl
  - migration
  - drizzle
aliases:
  - 온라인 DDL
  - 마이그레이션
  - 테이블 락
  - ALTER TABLE
  - Drizzle 마이그레이션
---

# 온라인 DDL과 대형 테이블 마이그레이션 위험 설명

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[architecture]] §3.1 D6, §4, [[schema-reference]] §K

"컬럼 하나 추가했을 뿐인데 서비스가 멈췄다" — 실제 운영에서 자주 발생하는 사고입니다. 이 문서는 DDL 마이그레이션이 왜 위험하고, 어떻게 안전하게 할 수 있는지를 설명합니다.

---

## Q1. DDL이 뭔가요? (ALTER TABLE, CREATE INDEX 같은 거)

SQL 문법은 크게 두 종류로 나뉩니다.

| 구분 | 예시 | 역할 |
|---|---|---|
| **DML** (Data Manipulation Language) | `SELECT`, `INSERT`, `UPDATE`, `DELETE` | 데이터를 읽고 쓰는 것 |
| **DDL** (Data Definition Language) | `ALTER TABLE`, `CREATE INDEX`, `DROP COLUMN` | 테이블 구조를 바꾸는 것 |

평소에 코드에서 쓰는 건 대부분 DML입니다. DDL은 테이블 자체의 설계를 변경할 때 씁니다.

```sql
-- DML: 데이터 조작 (매일 수백만 번)
INSERT INTO audit_log (action, actor_pk, ...) VALUES (...);
SELECT * FROM membership WHERE user_pk = ?;

-- DDL: 구조 변경 (마이그레이션 때만)
ALTER TABLE membership ADD COLUMN display_order INT DEFAULT 0;
CREATE INDEX idx_membership_display ON membership(display_order);
DROP COLUMN old_unused_column FROM user_profile;
```

Drizzle ORM을 쓸 때 `pnpm drizzle-kit generate`로 생성되는 `.sql` 마이그레이션 파일이 바로 DDL 파일입니다.

> 💡 **한 줄 요약**: DDL은 테이블의 "설계도"를 바꾸는 SQL로, Drizzle migration 파일에 담기는 `ALTER TABLE` 같은 명령들이 이에 해당합니다.

---

## Q2. 운영 중인 테이블에 컬럼을 추가하면 왜 서비스가 멈출 수 있나요?

대부분의 DDL은 내부적으로 이런 과정을 거칩니다.

```
1. 테이블 전체에 배타적 잠금(EXCLUSIVE LOCK) 획득
2. 내부적으로 새 구조의 임시 테이블 생성
3. 기존 테이블 전체를 임시 테이블로 복사 (행 수가 많을수록 오래 걸림!)
4. 임시 테이블을 원본 테이블과 교체
5. 잠금 해제
```

3번 단계에서 수천만 건을 복사하면 **수십 분**이 걸릴 수 있습니다. 이 동안 테이블에 잠금이 걸려 있으니, 다른 `INSERT`/`UPDATE` 요청들이 전부 대기 상태로 쌓입니다.

```
운영 중 ALTER TABLE 실행 시 타임라인 (전체 rewrite를 유발하는 DDL의 경우):

t=0s   ALTER TABLE payment_ledger ALTER COLUMN amount TYPE NUMERIC(...) 실행
t=0s   테이블 배타적 잠금(ACCESS EXCLUSIVE) 획득 → 이후 모든 INSERT/UPDATE/SELECT 대기 큐에 쌓임
t=0s   ~ t=1800s  수백만 행 복사 중 (30분!)
  ↕  이 동안 결제 API 요청 전부 타임아웃 → 503 응답
t=1800s  복사 완료, 잠금 해제
t=1800s  대기 중이던 요청들 순차 처리 (이미 클라이언트는 포기한 상태)
```

행 수가 100만 건이 넘는 테이블에서는 이 위험이 현실입니다. `payment_ledger`나 `audit_log` 같은 테이블은 바로 이 케이스입니다.

> 💡 **한 줄 요약**: DDL은 테이블을 통째로 복사하는 과정이 필요할 수 있고, 그 수십 분 동안 테이블이 잠겨 서비스 요청 전체가 멈춥니다.

---

## Q3. "테이블 락(table lock)"이 구체적으로 어떤 상황을 말하는 건가요?

API 서버 입장에서 보면 이렇게 됩니다.

```typescript
// 결제 처리 코드 (academy-api)
async function processPayment(orgPk: number, amount: number) {
  await db.insert(paymentLedger).values({
    orgPk,
    type: 'CHARGE',
    amountMinor: amount,
    // ...
  });
  // ↑ 이 INSERT가 DB 커넥션을 잡고 응답을 기다림
  // ALTER TABLE이 잠금 중이라면, 이 줄에서 무한 대기...
}
```

DB 커넥션 풀(예: 최대 10개)이 전부 대기 상태가 되면, 11번째 요청부터는 커넥션 자체를 얻지 못해 즉시 500 에러가 납니다.

```
커넥션 풀 상태 (ALTER TABLE 실행 중):

[conn-1]  INSERT payment_ledger → 대기 중 (3초 경과)
[conn-2]  INSERT payment_ledger → 대기 중 (2초 경과)
...
[conn-10] SELECT org_entitlement → 대기 중 (1초 경과)
[conn-11] 새 요청 → "커넥션 풀 고갈" → 즉시 503 반환
```

락이 오래 유지되면 메모리 부족으로 DB 서버 자체가 불안정해질 수 있습니다.

> 💡 **한 줄 요약**: 테이블 락은 "DB가 마치 화장실 문을 잠그고 혼자 장시간 사용 중인 상태"로, 다른 모든 요청이 문 앞에서 줄 서며 타임아웃 나는 상황입니다.

---

## Q4. PostgreSQL에서 어떤 DDL이 안전(비차단)하고 어떤 게 위험한가요?

PostgreSQL의 DDL은 작업마다 잡는 **락 레벨**과 **테이블 rewrite 여부**가 다릅니다. 핵심은 "전체 행을 다시 쓰는가(rewrite)"와 "쓰는 동안 다른 트랜잭션을 막는가"입니다.

특히 인덱스 생성은 `CONCURRENTLY` 옵션으로 **읽기·쓰기를 막지 않고** 만들 수 있습니다 — PostgreSQL의 큰 강점입니다.

```sql
-- 비차단 인덱스 생성 (트랜잭션 블록 밖에서 단독 실행해야 함)
CREATE INDEX CONCURRENTLY idx_membership_display ON membership(display_order);
-- 빌드 동안에도 membership에 대한 INSERT/UPDATE/SELECT가 계속 가능
```

아래 표를 참고하세요.

| DDL 작업 | 비차단인가? | 비고 |
|---|---|---|
| CHECK 제약 추가 (`NOT VALID` 후 `VALIDATE`) | ✅ 거의 즉시, 검증 락 분리 가능 | platform_db에서 자주 씀 |
| 컬럼 기본값 변경 (`SET DEFAULT`) | ✅ 카탈로그만 변경 |  |
| 인덱스 추가 (`CREATE INDEX CONCURRENTLY`) | ✅ 비차단 (DML 허용) | PostgreSQL 강점 |
| 인덱스 추가 (`CREATE INDEX`, CONCURRENTLY 없이) | ❌ 빌드 동안 쓰기 차단 | 작은 테이블만 |
| 컬럼 추가 (`ADD COLUMN`, NULL 또는 비휘발성 default) | ✅ 즉시 (메타데이터만, rewrite 없음) | PG 11+ 강점 |
| `ENUM` 값 추가 (`ALTER TYPE ADD VALUE`) | ✅ 비차단 | 단 트랜잭션 블록 내 실행 불가 |
| 컬럼 타입 변경 (`ALTER COLUMN ... TYPE`) | ❌ 대개 전체 rewrite + ACCESS EXCLUSIVE | 대형 테이블 위험 |
| `ENUM` 타입 자체 교체(값 제거/재정렬) | ❌ 타입 교체 → 컬럼 rewrite | [[enum-vs-varchar-check\|D6]]에서 ENUM 기피 이유 |

비휘발성 default를 가진 `ADD COLUMN`이 즉시 끝나는 건, PostgreSQL 11부터 기존 행을 실제로 채우지 않고 "기본값 메타데이터"만 저장하기 때문입니다. 반대로 `ALTER COLUMN ... TYPE`은 거의 항상 전체 행을 다시 써야 해서, 대형 테이블에서는 [[enum-vs-varchar-check|VARCHAR+CHECK]]처럼 rewrite를 피하는 설계가 중요합니다.

> 🐬 **MySQL이라면**: 락 회피를 `ALGORITHM=INPLACE, LOCK=NONE` 힌트로 명시합니다(`LOCK=NONE`은 "잠금 없이 못 하면 오류로 알려달라"는 안전망). MySQL 8.0의 인스턴트 `ADD COLUMN`이 PG의 즉시 추가에 대응하지만, `MODIFY COLUMN`이나 ENUM 값 추가는 전체 테이블 재빌드 + 락을 유발할 수 있어 PG보다 위험합니다.

> 💡 **한 줄 요약**: PostgreSQL은 `CREATE INDEX CONCURRENTLY`, 비휘발성 default `ADD COLUMN`, `ALTER TYPE ADD VALUE`를 비차단으로 처리하지만, `ALTER COLUMN ... TYPE`(rewrite)은 ACCESS EXCLUSIVE 락이 걸려 대형 테이블에서 위험합니다.

---

## Q5. Drizzle 마이그레이션을 실행할 때 조심해야 할 점이 뭔가요?

Drizzle이 편리한 이유는 스키마 파일을 변경하면 자동으로 migration SQL을 생성해줘서입니다. 하지만 그 SQL이 대형 테이블에 안전한지는 **개발자가 직접 확인해야 합니다.**

```bash
# 스키마 변경 후 마이그레이션 파일 생성
pnpm drizzle-kit generate

# 생성된 파일 확인! → 반드시 SQL을 직접 읽어봐야 함
cat drizzle/migrations/0007_add_column.sql
```

생성된 SQL 예시:

```sql
-- Drizzle이 생성한 migration (위험한 케이스)
ALTER TABLE "payment_ledger"
  ALTER COLUMN "amount" TYPE NUMERIC(18,0);  -- ← 타입 변경! 전체 rewrite + ACCESS EXCLUSIVE 락 위험
```

**마이그레이션 실행 전 체크리스트:**

```
1. 변경 대상 테이블의 현재 행 수 확인
   SELECT COUNT(*) FROM payment_ledger;
   → 100만 건 이상이면 반드시 아래 단계 진행

2. 이 DDL이 테이블 rewrite를 유발하는지 / 어떤 락을 잡는지 확인
   - ALTER COLUMN ... TYPE 은 대개 전체 rewrite (위험)
   - 인덱스 추가라면 CREATE INDEX CONCURRENTLY 로 비차단 전환 가능
   - lock_timeout / statement_timeout 을 짧게 걸어 폭주 방지

3. staging 환경에서 실행 시간 측정
   프로덕션 행 수와 비슷한 데이터로 테스트

4. rewrite가 불가피하면 pg_repack 또는 단계적 마이그레이션(Q6) 검토

5. 배포 시간은 트래픽 최저점(새벽)에 맞춤
```

> 🐬 **MySQL이라면**: 비차단 가능 여부를 `ALTER TABLE ... ALGORITHM=INPLACE, LOCK=NONE`으로 시도해 보고(불가능하면 오류), 30초 이상 걸릴 위험이 있으면 `pt-online-schema-change`(Percona)로 원본을 유지한 채 그림자 테이블에 복사하는 방식을 씁니다.

> 💡 **한 줄 요약**: Drizzle이 migration SQL을 자동 생성해줘도, 실행 전에 "이 SQL이 대형 테이블에서 잠금을 거는지" 직접 확인하는 습관이 필수입니다.

---

## Q6. ENUM → VARCHAR+CHECK 마이그레이션을 pg_provider 컬럼에 해야 하는데, 어떻게 안전하게 하나요?

현재 `payment_ledger.pg_provider`는 네이티브 ENUM 타입(`pg_provider_enum`)입니다. 새 PG(결제 대행사)를 *추가*만 한다면 `ALTER TYPE ADD VALUE`로 비차단 처리되지만, 값을 *빼거나 재정렬*해야 하면 타입 교체 → `ALTER COLUMN ... TYPE` → **전체 행 rewrite**가 일어나 대형 테이블에서 위험합니다.

그래서 D6 원칙에 따라 `VARCHAR(50) + CHECK constraint`로 마이그레이션할 예정입니다.

**왜 VARCHAR+CHECK가 안전한가요?**

```sql
-- 위험: 타입 자체 교체 (전체 rewrite → ACCESS EXCLUSIVE 락 → 수십 분)
ALTER TABLE payment_ledger
  ALTER COLUMN pg_provider TYPE pg_provider_enum_v2 USING pg_provider::text::pg_provider_enum_v2;

-- 안전: CHECK constraint 변경 (즉시 완료, rewrite 없음)
ALTER TABLE payment_ledger
  DROP CONSTRAINT chk_pg_provider;
ALTER TABLE payment_ledger
  ADD CONSTRAINT chk_pg_provider
    CHECK (pg_provider IN ('TOSS','STRIPE','PAYPAL','MANUAL','KAKAO'));
```

CHECK constraint의 변경은 카탈로그만 업데이트하므로 즉시 완료됩니다(기존 행 재검증이 필요하면 `NOT VALID` 후 `VALIDATE CONSTRAINT`로 락 시간을 분리).

**ENUM → VARCHAR 마이그레이션 절차 (단계적):**

```sql
-- Step 1: VARCHAR 컬럼 추가 (메타데이터만, rewrite·잠금 없음)
ALTER TABLE payment_ledger
  ADD COLUMN pg_provider_new VARCHAR(50);

-- Step 2: 데이터 배치 복사 (대량 UPDATE 대신 배치로 — 긴 트랜잭션 락 방지)
-- 한 번에 1000건씩 업데이트
UPDATE payment_ledger
  SET pg_provider_new = pg_provider::text
  WHERE pk IN (
    SELECT pk FROM payment_ledger
    WHERE pg_provider_new IS NULL
    LIMIT 1000
  );
-- → 이 작업을 반복하는 배치 스크립트로 처리

-- Step 3: 기존 컬럼 사용하는 코드를 새 컬럼으로 전환 후
--         기존 ENUM 컬럼 DROP (메타데이터만)
ALTER TABLE payment_ledger
  DROP COLUMN pg_provider;

-- Step 4: 새 컬럼 이름 변경 + NOT NULL + CHECK 추가
ALTER TABLE payment_ledger
  RENAME COLUMN pg_provider_new TO pg_provider;
ALTER TABLE payment_ledger
  ALTER COLUMN pg_provider SET NOT NULL,
  ADD CONSTRAINT chk_pg_provider
    CHECK (pg_provider IN ('TOSS','STRIPE','PAYPAL','MANUAL'));
```

이 방식은 서비스 중단 없이 진행할 수 있습니다. PostgreSQL에서는 위 DDL들을 하나의 트랜잭션으로 묶을 수도 있지만, Step 2의 대량 UPDATE는 트랜잭션을 작게 끊어 락 보유 시간을 줄이는 게 좋습니다.

> 🐬 **MySQL이라면**: Step 1·3의 컬럼 추가/삭제에 `ALGORITHM=INPLACE, LOCK=NONE`을, Step 4의 NOT NULL+제약에 `MODIFY COLUMN`을 씁니다. ENUM 컬럼은 PG와 달리 *추가*조차 전체 rebuild를 부를 수 있어 이 단계적 절차가 더 절실합니다.

> 💡 **한 줄 요약**: ENUM → VARCHAR+CHECK 마이그레이션은 컬럼을 한 번에 바꾸지 않고, 새 컬럼 추가 → 데이터 배치 복사 → 코드 전환 → 기존 컬럼 제거 순서로 단계적으로 진행해야 안전합니다.

---

## Q7. 마이그레이션을 잘못 실행하면 어떻게 롤백하나요?

좋은 소식: **PostgreSQL의 DDL은 트랜잭셔널**입니다. 대부분의 DDL을 `BEGIN ... COMMIT` 안에 넣을 수 있고, 중간에 `ROLLBACK`하면 스키마 변경이 깔끔하게 되돌려집니다.

```sql
-- PostgreSQL에서는 실제로 롤백됩니다:
BEGIN;
ALTER TABLE membership ADD COLUMN new_col INT;
-- 검증 쿼리 등 확인 후 문제가 있으면:
ROLLBACK;  -- ← new_col이 추가되지 않은 상태로 깔끔히 복귀
```

여러 DDL을 한 트랜잭션으로 묶어 "전부 성공 아니면 전부 무효"로 적용할 수 있어, 마이그레이션 중간 실패로 스키마가 어정쩡하게 남는 일을 막습니다. 단, 예외가 있습니다 — `CREATE INDEX CONCURRENTLY`, `ALTER TYPE ... ADD VALUE`, `VACUUM` 등 일부는 트랜잭션 블록 안에서 실행할 수 없어 단독으로 돌려야 합니다.

> 🐬 **MySQL이라면**: DDL이 **암묵적으로 커밋**되어 트랜잭션으로 되돌릴 수 없습니다. `BEGIN; ALTER TABLE ...; ROLLBACK;`을 해도 변경이 그대로 남습니다. 그래서 아래 "역방향 migration 준비" 전략이 MySQL에서는 사실상 필수입니다.

그래도 PostgreSQL에서도 **이미 커밋된** 마이그레이션을 되돌리거나, 트랜잭션 밖 DDL(위 예외)을 정리하려면 아래 전략이 유용합니다.

**전략 1: 역방향 migration 파일 준비**

```sql
-- 0007_add_column.sql (forward)
ALTER TABLE membership ADD COLUMN display_order INT DEFAULT 0;

-- 0007_add_column_rollback.sql (backward, 미리 준비)
ALTER TABLE membership DROP COLUMN display_order;
```

문제 발생 시 rollback 파일을 실행합니다. 단, `DROP COLUMN`도 대형 테이블에서는 오래 걸릴 수 있습니다.

**전략 2: 컬럼 추가는 하되 코드 전환 전에 확인**

```
배포 순서:
  1. migration 실행 (컬럼만 추가, 아직 코드는 기존 컬럼 사용)
  2. 10~30분 모니터링 → 문제 없으면 계속
  3. 코드 배포 (새 컬럼 사용 시작)
  → 문제 시 Step 3 전에 컬럼 DROP으로 되돌리기 가능
```

**전략 3: pg_repack 활용**

rewrite가 불가피한 작업(예: 테이블 팽창 정리, 일부 타입 변경)에서는 `pg_repack` 확장을 쓰면 원본 테이블을 거의 막지 않고 새 물리 저장으로 재구성할 수 있습니다.

> 🐬 **MySQL이라면**: 같은 역할로 Percona의 `pt-online-schema-change`를 써서 원본 테이블을 유지한 채 그림자 테이블로 복사하며 변경합니다.

**Drizzle migration 실패 시:**

```bash
# Drizzle은 journal 파일로 migration 상태를 추적
# 실패한 migration의 journal 항목을 수동으로 제거하면 재실행 가능
# 단, 실제 DB 상태와 journal이 불일치하지 않도록 주의
```

> 💡 **한 줄 요약**: PostgreSQL DDL은 트랜잭셔널이라 `BEGIN ... ROLLBACK`으로 미커밋 변경을 되돌릴 수 있지만(MySQL은 불가), 이미 커밋됐거나 트랜잭션 밖 DDL을 위해선 역방향 SQL을 미리 준비해두는 게 안전합니다.

---

## 마치며

운영 DB 마이그레이션은 개발 DB에서 잘 되더라도 프로덕션에서 위험할 수 있습니다. 행 수가 10만 건인 개발 DB와 5000만 건인 운영 DB는 같은 SQL에도 실행 시간이 수백 배 차이가 날 수 있거든요.

핵심 원칙 세 가지만 기억하세요.

1. **변경하기 어려운 ENUM 대신 VARCHAR+CHECK**: 네이티브 ENUM은 값 제거·재정렬 시 타입 교체 → 전체 rewrite 위험. `VARCHAR(50) + CHECK`를 씁니다.
2. **staging에서 먼저 시간 측정**: 운영과 비슷한 데이터 양으로 테스트합니다.
3. **배포 전 SQL을 직접 읽을 것**: Drizzle이 생성한 migration이 `ALTER COLUMN ... TYPE`처럼 rewrite를 부르는지 반드시 확인합니다.

관련 아키텍처 결정은 [[architecture]] §3.1 D6 원칙과 [[schema-reference]] §K 서비스 확장 방법을 참고하세요.

---

## 연결된 개념

- [[enum-vs-varchar-check|ENUM vs VARCHAR+CHECK (D6)]] — D6 원칙과 ENUM→VARCHAR 마이그레이션의 연결
- [[index-design|인덱스 설계]] — 인덱스 추가 DDL의 온라인 여부
- [[partitioning|DB 파티셔닝]] — 파티션 추가 운영과 DDL 특성
> 소스 문서
- [[architecture]] — §3.1 D6 결정, D6 미적용 사례 (R8 AI 리뷰)
- [[schema-reference]] — D.13 org_subscription, D.17 payment_ledger (pg_provider 마이그레이션 대상), §K 서비스 확장 DDL
