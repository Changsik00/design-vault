---
tags:
  - platform-db
  - explainer
  - p2
  - db-ops
  - mysql
  - partitioning
  - performance
  - audit
aliases:
  - DB 파티셔닝
  - RANGE 파티션
  - audit_log 파티션
  - p_future
---

# DB 파티셔닝 설명 (audit_log 파티션 예시)

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[schema-reference]] §D.8, [[architecture]] §12.11

[[audit-hash-chain|audit_log]]는 서비스가 살아있는 한 매일 쌓이는 테이블입니다. 1년만 운영해도 수천만 건이 넘을 수 있어요. 이 문서는 "왜 파티셔닝을 썼는지, 어떻게 동작하는지"를 처음 접하는 분도 이해할 수 있도록 설명합니다.

---

## Q1. DB 파티셔닝이 뭔가요? 테이블을 나눈다는 게 어떤 의미인가요?

쉽게 말하면 **하나의 큰 테이블을 내부적으로 여러 조각(파티션)으로 쪼개 저장**하는 방식입니다.

파일 시스템에 비유하면 이렇습니다. 회사 문서를 하나의 거대한 폴더에 전부 몰아넣으면 찾기 힘들잖아요. 그래서 "2026년 1월 문서", "2026년 2월 문서"처럼 월별로 폴더를 나눠 관리하는 거죠. DB 파티셔닝도 같은 개념입니다.

```
audit_log 테이블 (개발자 눈에는 하나의 테이블)
  ├── 파티션 p202601  → 2026년 1월 데이터 저장
  ├── 파티션 p202602  → 2026년 2월 데이터 저장
  ├── 파티션 p202603  → 2026년 3월 데이터 저장
  └── 파티션 p_future → 위 어디에도 안 들어가는 데이터
```

**중요한 점**: 쿼리를 쓸 때는 여전히 `SELECT * FROM audit_log WHERE ...`처럼 하나의 테이블로 다룹니다. 파티션이 몇 개든 상관없어요. 물리적 저장만 나뉘고, SQL 인터페이스는 동일합니다.

> 💡 **한 줄 요약**: 파티셔닝은 "겉으로는 한 테이블, 속으로는 월별 서랍장"처럼 동작하는 물리적 분할 전략입니다.

---

## Q2. audit_log에만 파티션을 쓰는 이유가 뭔가요? 다른 테이블은 왜 안 하나요?

파티셔닝은 **데이터가 무한히 증가하고, 오래된 데이터는 삭제(또는 아카이빙)해야 하는 테이블**에 효과적입니다.

`audit_log`가 딱 그 케이스입니다. 증가 속도를 수치로 보면 감이 옵니다.

```
하루 API 호출 10만 건
→ 1달: 300만 row
→ 1년: 3,600만 row
→ 3년: 1억 row
```

1억 row 테이블이 되면, 인덱스가 있어도 "이번 달 학원 A의 감사 기록"을 찾기 위해 거대한 단일 저장 구조를 훑어야 합니다.

- 서비스 운영 중 모든 행위(로그인, 권한 체크, 결제 등)가 INSERT됩니다.
- ISMS-P 기준상 5년 이상 보존이 권장됩니다.
- 반면 5년 지난 로그는 더 이상 필요 없어 아카이빙하거나 삭제할 수 있습니다.

다른 테이블(`organization`, `membership`, `payment_ledger` 등)은 왜 파티셔닝을 안 할까요?

| 테이블 | 이유 |
|---|---|
| `organization` | 행 수가 테넌트 수만큼만 증가. 수백만 행 가능성 낮음 |
| `membership` | 행 수가 사용자×조직 수. 급증하지 않음 |
| `payment_ledger` | 증가하긴 하나, 날짜 범위로 아카이빙할 명확한 정책이 없음 |

파티셔닝은 만능이 아닙니다. 설정과 유지보수 비용이 있어서, **데이터가 폭발적으로 쌓이고 주기적 파기 정책이 있는 테이블**에만 적용하는 게 현명합니다.

> 💡 **한 줄 요약**: 파티셔닝은 "무한 증가 + 주기적 파기" 두 조건이 맞을 때만 쓰는 도구로, audit_log가 그 전형적인 케이스입니다.

---

## Q3. RANGE 파티셔닝이 뭔가요? 어떻게 동작하나요?

MySQL에는 여러 파티셔닝 방식이 있습니다. 우리가 쓰는 건 **RANGE COLUMNS** 방식입니다.

`RANGE`는 "특정 컬럼 값의 범위(range)에 따라 데이터를 어느 파티션에 넣을지 결정"하는 방식입니다.

```sql
PARTITION BY RANGE COLUMNS(created_at) (
  PARTITION p202601 VALUES LESS THAN ('2026-02-01 00:00:00'),
  -- created_at < 2026-02-01 이면 이 파티션으로
  PARTITION p202602 VALUES LESS THAN ('2026-03-01 00:00:00'),
  -- created_at < 2026-03-01 이면 이 파티션으로 (단, 이미 p202601 통과 이후)
  PARTITION p202603 VALUES LESS THAN ('2026-04-01 00:00:00'),
  -- ...
  PARTITION p_future VALUES LESS THAN (MAXVALUE)
  -- 나머지 전부
)
```

데이터가 INSERT될 때 MySQL은 `created_at` 값을 보고 자동으로 올바른 파티션에 저장합니다. 개발자가 신경 쓸 게 없어요.

조회할 때도 마찬가지입니다. `WHERE created_at BETWEEN '2026-02-01' AND '2026-02-28'`처럼 날짜 조건이 있으면, MySQL이 `p202602` 파티션만 스캔합니다. 나머지 파티션은 건드리지 않아요. 이걸 **파티션 프루닝(partition pruning)** 이라 합니다.

```
쿼리: SELECT * FROM audit_log WHERE created_at BETWEEN '2026-02-01' AND '2026-02-28'

MySQL 내부 동작:
  p202601 → 스킵 (날짜 범위 밖)
  p202602 → 스캔 ← 여기만!
  p202603 → 스킵 (날짜 범위 밖)
  p_future → 스킵 (날짜 범위 밖)
```

> 💡 **한 줄 요약**: RANGE 파티셔닝은 날짜 범위로 데이터를 자동 분류하고, 조회 시 관련 파티션만 읽는 방식입니다.

---

## Q4. audit_log의 created_at이 TIMESTAMP가 아닌 DATETIME인 이유가 뭔가요?

DDL을 보면 `created_at DATETIME NOT NULL DEFAULT (NOW())` 으로 선언되어 있습니다. 다른 테이블들은 대부분 `TIMESTAMP`를 쓰는데 왜 여기서만 `DATETIME`을 쓸까요?

**MySQL 8.0의 파티셔닝 버그 때문입니다.**

MySQL 8.0에서 `PARTITION BY RANGE COLUMNS`에 `TIMESTAMP` 타입 컬럼을 쓰면 제대로 동작하지 않는 버그가 있습니다. 그래서 파티셔닝 키로 쓸 컬럼을 `DATETIME`으로 선언하는 우회책을 적용했습니다.

```sql
-- 이렇게 하고 싶었지만 MySQL 8.0에서 버그 발생:
created_at TIMESTAMP NOT NULL DEFAULT NOW()
-- PARTITION BY RANGE COLUMNS(created_at) → 오류 또는 오작동

-- 실제 사용하는 우회책:
created_at DATETIME NOT NULL DEFAULT (NOW())
-- PARTITION BY RANGE COLUMNS(created_at) → 정상 동작
```

`TIMESTAMP`와 `DATETIME`의 주요 차이는 다음과 같습니다.

| | TIMESTAMP | DATETIME |
|---|---|---|
| 저장 방식 | UTC로 변환 후 저장, 조회 시 시스템 타임존으로 변환 | 입력값 그대로 저장 |
| 범위 | 1970~2038년 | 1000~9999년 |
| 파티셔닝 호환 | MySQL 8.0 버그 있음 | 정상 동작 |

PostgreSQL을 쓰면 이런 우회책이 필요 없습니다. 하지만 현재 스택이 MySQL이라 `DATETIME`으로 처리하고 있습니다.

> 💡 **한 줄 요약**: MySQL 8.0의 파티셔닝 + TIMESTAMP 버그를 피하기 위해 `DATETIME`을 사용하는 어쩔 수 없는 우회책입니다.

---

## Q5. 파티션이 자동으로 추가되지 않으면 어떤 일이 생기나요? (p_future 문제)

현재 DDL을 보면 파티션이 2026년 6월까지만 정의되어 있습니다.

```sql
PARTITION p202606 VALUES LESS THAN ('2026-07-01 00:00:00'),
PARTITION p_future VALUES LESS THAN (MAXVALUE)  -- ← 2026년 7월 이후 데이터가 모두 여기로!
```

`p_future`는 "아직 파티션이 없는 미래 데이터를 임시로 받아두는 파티션"입니다. 문제는 이 파티션에 데이터가 계속 쌓이면 어떻게 되냐는 거죠.

**p_future가 가득 차면 생기는 문제들:**

```
p202601: 1월 데이터 (500만 행) → 인덱스 작고 빠름
p202602: 2월 데이터 (500만 행) → 인덱스 작고 빠름
...
p_future: 7월~무한대 데이터 (5000만 행!) → 거대한 파티션 → 느림
```

1. 파티션 프루닝 효과가 사라집니다. 7월 데이터를 조회해도 `p_future` 전체를 스캔합니다.
2. 개별 파티션 DROP으로 오래된 데이터를 빠르게 삭제할 수 없습니다.
3. 파티션 하나가 지나치게 커져 성능이 월별 파티션과 비슷해집니다.

그래서 **운영 정기 작업(분기 1회)** 으로 새 파티션을 미리 [[online-ddl-migration|온라인 DDL]]로 추가해야 합니다.

```sql
-- architecture.md §12.11 정기 운영 작업:
-- "audit_log 파티션 추가: 분기 1회"
ALTER TABLE audit_log
  REORGANIZE PARTITION p_future INTO (
    PARTITION p202607 VALUES LESS THAN ('2026-08-01 00:00:00'),
    PARTITION p202608 VALUES LESS THAN ('2026-09-01 00:00:00'),
    PARTITION p202609 VALUES LESS THAN ('2026-10-01 00:00:00'),
    PARTITION p_future VALUES LESS THAN (MAXVALUE)  -- p_future는 항상 마지막에 유지
  );
```

이 작업을 깜빡하면 `p_future`가 무한정 커져 파티셔닝의 이점이 사라집니다. 달력 알림을 꼭 걸어두세요.

> 💡 **한 줄 요약**: `p_future`는 임시 안전망이지만, 분기마다 새 파티션을 추가하지 않으면 이 안전망 하나에 모든 데이터가 쌓여 파티셔닝 효과가 없어집니다.

---

## Q6. 파티션을 쓰면 쿼리가 어떻게 빨라지나요?

두 가지 방식으로 성능이 좋아집니다.

### 1) 파티션 프루닝 — 읽는 데이터를 줄임

```sql
-- 파티션 없는 경우:
-- audit_log 전체 (예: 6000만 행) 스캔 후 필터
SELECT * FROM audit_log
WHERE created_at >= '2026-02-01' AND created_at < '2026-03-01'
AND actor_pk = 42;

-- 파티션 있는 경우:
-- MySQL이 자동으로 p202602만 스캔 (예: 500만 행)
-- 읽는 데이터가 1/12로 줄어듦!
```

EXPLAIN 결과로 파티션 프루닝이 실제로 일어났는지 확인할 수 있습니다.

```sql
EXPLAIN SELECT * FROM audit_log
WHERE created_at >= '2026-02-01' AND created_at < '2026-03-01';
-- partitions: p202602 → 프루닝 성공 확인
```

### 2) 파티션 단위 DROP — 아카이빙이 즉시 완료

파티셔닝의 또 다른 이점은 **오래된 데이터 삭제가 빠르다**는 점입니다.

```sql
-- 느린 방식 (수억 건 DELETE → 수시간):
DELETE FROM audit_log WHERE created_at < '2026-01-01 00:00:00';

-- 빠른 방식 (파티션 DROP → 즉시 완료):
ALTER TABLE audit_log DROP PARTITION p202601;
-- 1월 데이터 500만 건이 파일 삭제 수준으로 즉시 제거됨
```

물론 파티션 DROP 전에 해당 데이터를 S3 같은 외부 스토리지로 아카이빙(백업)하는 게 원칙입니다.

```
운영 절차 (오래된 파티션 아카이빙):
  1. p202601 → S3 버킷으로 EXPORT
  2. S3 Object Lock 적용 (T4 트리거 도달 시 WORM 보장)
  3. ALTER TABLE audit_log DROP PARTITION p202601;
```

> 💡 **한 줄 요약**: 파티셔닝은 날짜 조건 쿼리를 "전체 스캔 → 해당 월만 스캔"으로 줄여주고, 오래된 데이터 삭제도 DELETE 수백만 건 대신 파티션 DROP 한 줄로 끝낼 수 있게 해줍니다.

---

## 부록: 복합 PK가 필요한 이유

DDL에서 PK가 `(pk, created_at)` 조합인 걸 보셨을 겁니다. 왜 `pk` 하나로 안 될까요?

```sql
-- 왜 이렇게 안 될까?
PRIMARY KEY (pk)  -- MySQL 파티션 테이블에서 오류!

-- 이렇게 해야 함:
PRIMARY KEY (pk, created_at)
```

MySQL 파티션 테이블의 규칙 때문입니다.

```
MySQL 파티션 테이블 규칙:
  PRIMARY KEY는 반드시 파티션 키(created_at)를 포함해야 한다.

이유:
  MySQL은 특정 row가 어느 파티션에 있는지 파악하기 위해
  파티션 키 값을 PK에서 바로 읽을 수 있어야 합니다.

  pk=12345라는 row를 찾으려면, 그게 몇 월 데이터인지 알아야
  어느 파티션을 뒤져야 하는지 알 수 있거든요.
  pk만으로는 이 정보를 알 수 없습니다.
```

실제 운영에서 `pk`만으로 row를 찾을 일이 있다면 별도 [[index-design|인덱스]]를 추가하면 됩니다.

```sql
-- pk 단독 조회가 필요하다면:
INDEX idx_audit_pk (pk)  -- 필요 시 추가
```

---

## 마치며

파티셔닝은 처음엔 복잡해 보이지만 "월별 서랍장"으로 이해하면 직관적입니다. 핵심은 세 가지입니다.

1. **왜 쓰나**: 무한 증가하는 audit_log를 월별로 쪼개 쿼리 속도를 높이고, 오래된 데이터를 빠르게 아카이빙하기 위해
2. **어떻게 동작하나**: RANGE 파티셔닝으로 `created_at` 날짜 범위에 따라 자동 분류, 조회 시 관련 파티션만 스캔
3. **유지보수 포인트**: 분기마다 새 파티션을 미리 추가해야 `p_future` 비대화를 방지

관련 운영 작업은 [[architecture]] §12.11 정기 운영 섹션을 참고하세요.

---

## 연결된 개념

- [[audit-hash-chain|audit_log 해시 체인]] — 파티션 단위 아카이빙 후 WORM 적용 흐름
- [[index-design|인덱스 설계]] — 파티션 + 인덱스를 함께 설계하는 방법
- [[online-ddl-migration|온라인 DDL & 마이그레이션]] — 파티션 추가가 온라인 DDL인지 확인하는 방법
> 소스 문서
- [[schema-reference]] — D.8 audit_log DDL (파티션 정의, 복합 PK, DATETIME 선택 이유)
- [[review-checklist]] — P2-3 파티션 자동 추가 미구현 이슈
