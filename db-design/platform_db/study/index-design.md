# 복합 인덱스 설계 — 순서가 왜 중요한가

## 배경

인덱스(Index)는 DB가 row를 빠르게 찾기 위해 미리 만들어두는 정렬된 사본이다.

```
인덱스 없음: "status = 'ACTIVE'인 row 찾아라"
  → 테이블 전체를 처음부터 끝까지 읽음 (Full Table Scan)
  → 1,000,000 row = 1,000,000번 확인

인덱스 있음: INDEX idx_status (status)
  → B-tree에서 'ACTIVE' 위치로 바로 이동
  → 수 ms
```

그런데 인덱스 하나에 컬럼을 여러 개 포함하는 **복합 인덱스**는 순서가 중요하다.

---

## 복합 인덱스 — 왼쪽 우선 규칙 (Leftmost Prefix Rule)

```sql
INDEX idx_org_service_status (org_pk, service, status)
```

이 인덱스는 세 컬럼의 조합을 정렬해서 저장한다:

```
(org_pk=1, service='ACADEMY', status='ACTIVE')   ← 첫 번째
(org_pk=1, service='ACADEMY', status='EXPIRED')
(org_pk=1, service='MARKET',  status='ACTIVE')
(org_pk=2, service='ACADEMY', status='ACTIVE')
...
```

**인덱스를 사용할 수 있는 쿼리 패턴**:

```sql
-- ✅ 사용 가능: org_pk만 (leftmost prefix)
WHERE org_pk = 1

-- ✅ 사용 가능: org_pk + service
WHERE org_pk = 1 AND service = 'ACADEMY'

-- ✅ 사용 가능: 세 개 모두
WHERE org_pk = 1 AND service = 'ACADEMY' AND status = 'ACTIVE'

-- ❌ 사용 불가: org_pk 없이 service만 (leftmost 없음)
WHERE service = 'ACADEMY'

-- ❌ 사용 불가: org_pk 없이 status만
WHERE status = 'ACTIVE'
```

---

## 왜 org_pk를 첫 번째로?

```sql
-- Gate B 핵심 쿼리
SELECT * FROM org_entitlement
WHERE org_pk = ?        -- 이게 먼저
  AND service = ?       -- 이게 두 번째
  AND status IN ('ACTIVE', 'GRACE');
```

인덱스 설계 원칙: **카디널리티(Cardinality)가 높은 컬럼을 앞에**.

```
org_pk 기준 필터: 전체 row의 1/1000 이하 (학원이 1000개면)
service 기준 필터: 전체 row의 1/5 (서비스 5개)
status 기준 필터: 전체 row의 1/4 (상태 4가지)
```

`org_pk`가 먼저 걸리면 대부분의 row가 걸러지고, 나머지 필터는 소수 row에만 적용된다.

---

## 만료 배치 쿼리용 인덱스

```sql
-- 매일 실행: 만료된 entitlement 상태 변경
SELECT pk FROM org_entitlement
WHERE valid_until < NOW()   -- 인덱스 첫 번째 컬럼
  AND status = 'ACTIVE';    -- 인덱스 두 번째 컬럼

-- 이를 위한 인덱스
INDEX idx_entitlement_expiry (valid_until, status)
```

`status` 하나로 필터하면 전체 row의 1/4. `valid_until < NOW()`로 필터하면 훨씬 적다.
→ `valid_until`을 앞에 두면 만료 기간 내 row만 빠르게 조회.

---

## 파티션 + 인덱스

```sql
-- audit_log: 월별 파티션 + 인덱스
INDEX idx_audit_org_created (org_pk, created_at)

PARTITION BY RANGE COLUMNS(created_at) (
  PARTITION p202605 VALUES LESS THAN ('2026-06-01'),
  ...
)
```

```sql
-- 쿼리
WHERE org_pk = 1 AND created_at BETWEEN '2026-05-01' AND '2026-05-31'
```

MySQL이 하는 일:
1. **파티션 프루닝**: created_at 범위로 p202605 파티션만 선택
2. **인덱스 사용**: 그 파티션 안에서 org_pk로 빠르게 탐색

두 최적화가 함께 작동 → 전체 테이블의 극히 일부만 읽음.

---

## 트레이드오프

| 항목 | 인덱스 없음 | 단일 인덱스 | 복합 인덱스 (우리) |
|---|---|---|---|
| 읽기 속도 | 느림 (Full Scan) | 부분 개선 | 빠름 (선택적) |
| 쓰기 속도 | 빠름 | 약간 느림 | 더 느림 (인덱스 갱신) |
| 저장 공간 | 적음 | 중간 | 더 많음 |
| 유지 관리 | 없음 | 낮음 | 설계 주의 필요 |

인덱스는 읽기를 빠르게 하는 대신 쓰기를 느리게 하고 공간을 더 쓴다.
**자주 조회되는 패턴에만** 인덱스를 추가한다.

---

## 관련 파일

| 파일 | 내용 |
|---|---|
| `core/schema-reference.md §D.12` | org_entitlement 인덱스 설계 |
| `core/schema-reference.md §D.8` | audit_log 파티션 + 인덱스 |
| `study/partition-range.md` | RANGE 파티셔닝 상세 |
