# RANGE 파티셔닝 — audit_log를 왜 월별로 나누나

## 배경

`audit_log`는 모든 API 호출마다 한 row씩 쌓인다.

```
하루 API 호출 10만 건
→ 1달: 300만 row
→ 1년: 3,600만 row
→ 3년: 1억 row
```

1억 row 테이블에서 "이번 달 학원 A의 감사 기록"을 조회하면?

```sql
SELECT * FROM audit_log
WHERE org_pk = 1 AND created_at BETWEEN '2026-05-01' AND '2026-05-31';
```

인덱스가 있어도 1억 row를 가진 하나의 파일을 뒤져야 한다.

---

## 파티셔닝이란

논리적으로는 하나의 테이블이지만, 물리적으로는 여러 조각(파티션)으로 나눠 저장하는 기법.

```
audit_log (논리적으로 하나의 테이블)
├── p202601 (2026년 1월 데이터)  → 파일 A
├── p202602 (2026년 2월 데이터)  → 파일 B
├── p202603 (2026년 3월 데이터)  → 파일 C
├── p202604 (2026년 4월 데이터)  → 파일 D
├── p202605 (2026년 5월 데이터)  → 파일 E
└── p_future (나머지)            → 파일 F
```

---

## RANGE 파티셔닝 DDL

```sql
CREATE TABLE audit_log (
  pk         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_pk     BIGINT UNSIGNED,
  action     VARCHAR(100) NOT NULL,
  created_at DATETIME NOT NULL DEFAULT (NOW()),  -- TIMESTAMP 아님 (아래 주의사항)
  PRIMARY KEY (pk, created_at),                  -- 파티션 키가 PK에 포함 필수 (MySQL 규칙)
  INDEX idx_audit_org_created (org_pk, created_at)

) PARTITION BY RANGE COLUMNS(created_at) (
  PARTITION p202601 VALUES LESS THAN ('2026-02-01 00:00:00'),
  PARTITION p202602 VALUES LESS THAN ('2026-03-01 00:00:00'),
  PARTITION p202603 VALUES LESS THAN ('2026-04-01 00:00:00'),
  PARTITION p202604 VALUES LESS THAN ('2026-05-01 00:00:00'),
  PARTITION p202605 VALUES LESS THAN ('2026-06-01 00:00:00'),
  PARTITION p_future VALUES LESS THAN (MAXVALUE)  -- 새 파티션 추가 전까지 임시 보관
);
```

---

## 파티션 프루닝 (Partition Pruning)

```sql
-- 이 쿼리를 실행하면
SELECT * FROM audit_log
WHERE org_pk = 1
  AND created_at BETWEEN '2026-05-01' AND '2026-05-31';

-- MySQL이 하는 일:
-- 1. created_at 조건을 보고 p202605만 해당
-- 2. 나머지 파티션(p202601~p202604, p_future)은 완전히 건너뜀
-- 3. p202605 내에서만 인덱스 탐색
```

`EXPLAIN PARTITIONS`로 확인:
```
partitions: p202605   ← 이것만 스캔
```

수억 row에서 수백만 row만 읽으므로 I/O가 극적으로 줄어든다.

---

## MySQL의 TIMESTAMP 버그

MySQL 8.0에서 `PARTITION BY RANGE COLUMNS`는 **TIMESTAMP 타입을 지원하지 않는다**.

```sql
-- 이렇게 하면 에러
created_at TIMESTAMP NOT NULL,
PARTITION BY RANGE COLUMNS(created_at)  -- ERROR!
```

그래서 우리는 DATETIME을 쓴다:
```sql
created_at DATETIME NOT NULL DEFAULT (NOW())  -- TIMESTAMP 대신 DATETIME
```

PostgreSQL에서는 이 버그가 없다. TIMESTAMP 그대로 파티셔닝 가능.

---

## 분기별 파티션 추가

`p_future`는 임시 catch-all이다. 매 분기 새 파티션을 미리 추가해야 한다:

```sql
-- 분기마다 실행 (예: 2026년 7월 파티션 추가)
ALTER TABLE audit_log
  REORGANIZE PARTITION p_future INTO (
    PARTITION p202607 VALUES LESS THAN ('2026-08-01 00:00:00'),
    PARTITION p_future VALUES LESS THAN (MAXVALUE)
  );
```

---

## 왜 audit_log만 파티셔닝하나

| 테이블 | 파티셔닝 여부 | 이유 |
|---|---|---|
| audit_log | ✅ 파티셔닝 | 초고속 증가, 기간 조회, 월별 아카이빙 필요 |
| org_entitlement | ❌ | 학원 수만큼만 (수천 row) |
| membership | ❌ | 사용자×학원 (수만 row) |
| payment_ledger | ❌ | 결제는 많아도 일 수십 건 |

파티셔닝은 **운영 오버헤드**가 있다. 필요한 테이블에만 적용.

---

## 관련 파일

| 파일 | 내용 |
|---|---|
| `core/schema-reference.md §D.8` | audit_log DDL 전문 |
| `core/architecture.md §12.11` | 분기별 파티션 추가 운영 절차 |
| `study/index-design.md` | 파티션과 인덱스 조합 |
