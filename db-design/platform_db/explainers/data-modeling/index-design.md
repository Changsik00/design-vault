---
difficulty: 중
tags:
  - platform-db
  - explainer
  - p1
  - db-ops
  - mysql
  - index
  - performance
aliases:
  - 인덱스 설계
  - 복합 인덱스
  - 커버링 인덱스
  - 풀스캔
---

# 인덱스 설계 원리 설명

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[schema-reference]] §D.12, §D.18, §E.2 · [[architecture]] §2 불변식 #9

인덱스는 DB 성능의 핵심입니다. 잘 만든 인덱스 하나가 100ms짜리 쿼리를 1ms로 만들고, 없거나 잘못 만든 인덱스 하나가 서버 전체를 느리게 만듭니다. `platform_db`에는 목적별로 설계된 인덱스가 여러 개 있는데, 각각 왜 그렇게 생겼는지를 이해하면 새 기능을 만들 때도 인덱스를 올바르게 설계할 수 있습니다.

---

## Q1. 인덱스가 뭔가요? 없으면 어떤 일이 생기나요?

인덱스는 **책 뒤의 색인(index)** 과 똑같습니다. 책 500페이지 중에서 "MySQL"이라는 단어가 어디 있는지 찾으려면 두 가지 방법이 있습니다.

- 방법 A: 1페이지부터 500페이지를 전부 읽는다 (풀스캔)
- 방법 B: 책 뒤의 색인에서 "MySQL → 23, 47, 189 페이지"를 찾아 그 페이지만 읽는다 (인덱스 스캔)

테이블에 1,000만 건의 데이터가 있고 인덱스가 없다면, `WHERE user_pk = 12345` 한 줄로도 1,000만 건을 전부 읽어야 합니다.

```sql
-- 인덱스 없는 경우: 100만 건 테이블에서 1건 찾기
SELECT * FROM membership WHERE user_pk = 12345;
-- → 100만 건 전부 스캔. 수백 ms ~ 수 초

-- 인덱스 있는 경우 (idx_membership_user)
SELECT * FROM membership WHERE user_pk = 12345;
-- → 인덱스 트리를 타고 몇 건만 읽음. 수 ms
```

MySQL InnoDB는 인덱스를 **B-Tree(균형 이진 트리)** 구조로 저장합니다. 데이터를 정렬된 트리 형태로 관리하기 때문에 원하는 값을 찾을 때 트리를 타고 내려가기만 하면 됩니다. 1,000만 건이어도 탐색 횟수는 약 24번(log₂(10,000,000) ≈ 24)에 불과합니다.

> 💡 **한 줄 요약**: 인덱스는 책 색인처럼 특정 컬럼 기준으로 데이터 위치를 미리 정리해 놓은 구조로, 없으면 매번 전체 데이터를 뒤져야 합니다.

---

## Q2. "풀스캔(full scan)"이 왜 문제인가요? 그냥 느린 거 아닌가요?

"그냥 조금 느린 것"이 아닙니다. 세 가지 이유로 심각한 문제입니다.

**첫째, 데이터가 늘수록 선형으로 느려집니다.** 지금 1만 건일 때 10ms라면, 100만 건이 되면 1,000ms(1초)가 됩니다. 인덱스가 있으면 100만 건이 돼도 거의 차이가 없습니다.

**둘째, 풀스캔은 다른 쿼리도 느리게 만듭니다.** DB는 디스크 I/O와 CPU를 공유합니다. 한 쿼리가 풀스캔으로 디스크를 독점하면 다른 쿼리들이 줄을 서야 합니다.

**셋째, 핫패스(hot path)에서는 치명적입니다.** `[[explainers/auth/gate-b-billing-grace|Gate B]]`(서비스 이용 가능 여부 확인)는 **모든 API 요청마다 실행**됩니다. 사용자가 버튼 하나 누를 때마다 쿼리가 실행되는데, 여기서 풀스캔이 일어나면:

```
사용자 100명이 동시에 요청
→ 100개의 풀스캔이 동시 실행
→ DB CPU 100% 도달
→ 전체 API 응답 불가
→ 서버 다운
```

실제로 작은 스타트업에서 인덱스 빠진 쿼리 하나 때문에 서비스 전체가 멈춘 사례가 흔합니다.

```
풀스캔 발견 방법:
  EXPLAIN SELECT * FROM org_entitlement WHERE org_pk=1 AND service='ACADEMY';
  → type: ALL     (풀스캔, 위험)
  → type: ref     (인덱스 사용, 안전)
  → type: range   (인덱스 범위 스캔, 안전)
```

> 💡 **한 줄 요약**: 풀스캔은 데이터 증가에 비례해 느려지고, 동시 요청이 많으면 서버 전체를 멈출 수 있습니다.

---

## Q3. 복합 인덱스(여러 컬럼 인덱스)가 뭔가요? 컬럼 순서가 중요한가요?

복합 인덱스는 **두 개 이상의 컬럼을 묶어 하나의 인덱스로 만드는 것**입니다. 인덱스를 책 색인에 비유하면, 단일 인덱스는 "저자 이름 색인"이고, 복합 인덱스는 "저자 이름 + 출판연도 색인"입니다.

컬럼 순서가 매우 중요합니다. 복합 인덱스는 **왼쪽에서 오른쪽으로만 동작**합니다.

```sql
-- 인덱스: (A, B, C) 순서로 생성됐다고 가정

-- 사용 가능한 쿼리 패턴
WHERE A = 1                    -- A만 사용 → 인덱스 활용 가능
WHERE A = 1 AND B = 2          -- A, B 사용 → 인덱스 활용 가능
WHERE A = 1 AND B = 2 AND C=3  -- A, B, C 사용 → 인덱스 풀 활용

-- 사용 불가능한 쿼리 패턴
WHERE B = 2                    -- A 없이 B만 → 인덱스 활용 불가
WHERE B = 2 AND C = 3          -- A 없이 B, C → 인덱스 활용 불가
WHERE C = 3                    -- A, B 없이 C만 → 인덱스 활용 불가
```

인덱스를 전화번호부라고 생각하면 이해하기 쉽습니다. 전화번호부는 "성 → 이름" 순으로 정렬됩니다. "이름이 '민준'인 사람"을 찾으려면 전화번호부 전체를 봐야 하지만, "성이 '김'인 사람"은 'ㄱ' 섹션만 보면 됩니다. 인덱스 컬럼 순서도 같은 원리입니다.

> 💡 **한 줄 요약**: 복합 인덱스는 왼쪽 컬럼부터 순서대로만 사용할 수 있으므로, 자주 조회되는 컬럼을 왼쪽에 배치해야 합니다.

---

## Q4. Gate B 인덱스(`idx_org_service_status`)는 왜 이렇게 설계됐나요?

Gate B는 "이 org가 지금 이 서비스를 이용할 수 있나?"를 확인하는 체크포인트입니다. **모든 API 요청마다** 실행됩니다. 이 쿼리가 1ms가 아니라 100ms가 걸린다면 전체 서비스가 느려집니다.

```sql
-- Gate B 핵심 쿼리 (매 API 요청마다 실행)
SELECT status, valid_until, feature_limits
FROM org_entitlement
WHERE org_pk = ?
  AND service = ?
  AND status IN ('ACTIVE', 'GRACE');
```

이를 위한 인덱스:

```sql
INDEX idx_org_service_status (org_pk, service, status, valid_until)
```

컬럼 순서를 하나씩 따져봅니다:

```
1) org_pk: "어느 학원의" → 가장 먼저 범위를 확 좁힘
            (전체 수천 개 org 중 1개 org만 남음)

2) service: "어떤 서비스" → 그 학원의 여러 서비스 중 하나로 좁힘
            (ACADEMY, MARKET, AGENT... 중 하나)

3) status: "어떤 상태" → 살아있는 구독만 필터
           (ACTIVE, GRACE만 남김)

4) valid_until: 인덱스에 포함 → 테이블 본체를 안 읽어도 됨
               (인덱스 안에서 만료 여부까지 확인)
```

`valid_until`을 인덱스에 포함한 것이 중요한 포인트입니다. 이를 **커버링 인덱스(covering index)** 라고 합니다. 인덱스 자체에 필요한 모든 값이 있어서 테이블 본체(실제 데이터 파일)를 추가로 읽지 않아도 됩니다.

```
커버링 인덱스 없는 경우:
  인덱스 → "3번 행에 있음" → 테이블 접근 → 데이터 읽기
  (2번 I/O)

커버링 인덱스 있는 경우:
  인덱스 안에 valid_until 포함 → 인덱스만 읽으면 끝
  (1번 I/O)
```

> 💡 **한 줄 요약**: Gate B 인덱스는 모든 API 요청의 핫패스를 위해 설계됐으며, `valid_until`까지 포함해 테이블 본체 접근 없이 인덱스만으로 응답합니다.

---

## Q5. 만료 배치 인덱스(`idx_entitlement_expiry`)는 뭘 위한 건가요?

매일 자정, 배치 작업이 실행됩니다. 만료 기한이 지난 구독을 `EXPIRED` 상태로 바꾸는 작업입니다.

```sql
-- 매일 실행되는 배치 쿼리
UPDATE org_entitlement
SET status = 'EXPIRED'
WHERE valid_until < NOW()
  AND status = 'ACTIVE';
```

이 쿼리는 Gate B 쿼리와 다른 패턴입니다. Gate B는 "특정 org_pk"([[multitenancy-rls|멀티테넌시]] 격리 선두 컬럼)로 좁히지만, 배치는 **날짜 기준으로 전체 테이블**을 봅니다. 그래서 인덱스도 다르게 설계됩니다.

```sql
INDEX idx_entitlement_expiry (valid_until, status)
-- 배치 쿼리: WHERE valid_until < NOW() AND status='ACTIVE'
```

이 인덱스가 없으면 배치가 실행될 때 수백만 건의 `org_entitlement` 전체를 스캔해야 합니다. 배치 실행 시간이 길어지면:

1. 배치 실행 중에도 Gate B 쿼리가 계속 실행됨
2. 배치가 테이블 락(lock)을 잡으면 Gate B가 대기해야 함
3. 서비스 전체가 느려짐

인덱스가 있으면 `valid_until < NOW()`에 해당하는 적은 수의 행만 빠르게 찾아 처리합니다.

```
만료 배치 인덱스 효과:

인덱스 없음:
  1,000,000건 전부 스캔 → valid_until 확인 → 1,000건 업데이트
  (예상 시간: 수십 초)

인덱스 있음:
  valid_until < NOW() 범위를 인덱스에서 바로 찾음 → 1,000건만 접근
  (예상 시간: 수십 ms)
```

> 💡 **한 줄 요약**: 배치는 날짜 기준 전체 스캔 패턴이라 `valid_until`이 선두 컬럼인 전용 인덱스가 필요합니다.

---

## Q6. 인덱스를 많이 만들면 좋은 거 아닌가요?

아닙니다. 인덱스는 **공짜가 아닙니다.** 모든 인덱스는 **INSERT/UPDATE/DELETE가 일어날 때마다 함께 갱신**됩니다.

```
인덱스 없는 테이블에 INSERT:
  데이터 파일에 1건 쓰기 → 끝

인덱스가 5개인 테이블에 INSERT:
  데이터 파일에 1건 쓰기
  + 인덱스 1 갱신
  + 인덱스 2 갱신
  + 인덱스 3 갱신
  + 인덱스 4 갱신
  + 인덱스 5 갱신
  → 쓰기 비용 6배
```

`payment_ledger`나 `pg_webhook_event`처럼 결제 이벤트가 빠르게 쌓이는 테이블에 인덱스를 남발하면 INSERT 성능이 급격히 나빠집니다.

또 인덱스는 **디스크 공간**도 차지합니다. 컬럼 수가 많은 인덱스는 테이블 크기만큼 공간을 더 사용할 수 있습니다.

인덱스 설계 원칙을 정리하면:

```
인덱스를 만들어야 할 때:
  ✅ 자주 실행되는 쿼리의 WHERE 절 컬럼
  ✅ JOIN에 사용되는 FK 컬럼
  ✅ ORDER BY에 사용되는 컬럼
  ✅ 핫패스(매 요청마다 실행)의 필터 컬럼

인덱스를 만들지 말아야 할 때:
  ❌ 카디널리티(고유 값 수)가 낮은 컬럼 (예: is_deleted BOOLEAN)
  ❌ 거의 조회하지 않는 컬럼
  ❌ 쓰기 빈도가 매우 높은 테이블의 불필요한 컬럼
  ❌ 이미 다른 인덱스의 선두 컬럼에 포함된 것과 중복되는 단일 인덱스
```

`platform_db`의 인덱스들은 각각 명확한 목적이 있습니다:

| 인덱스 | 목적 | 쿼리 패턴 |
|---|---|---|
| `idx_org_service_status` | Gate B 핫패스 | `WHERE org_pk=? AND service=? AND status IN(...)` |
| `idx_entitlement_expiry` | 만료 배치 | `WHERE valid_until < NOW() AND status='ACTIVE'` |
| `idx_pg_webhook_status` | 재처리 워커 | `WHERE status='FAILED' ORDER BY created_at` |
| `idx_audit_break_glass` | 보안 감사 | `WHERE break_glass=TRUE ORDER BY created_at` |

> 💡 **한 줄 요약**: 인덱스는 읽기를 빠르게 하지만 쓰기를 느리게 하고 공간을 차지합니다. 필요한 곳에만, 목적에 맞게 만드는 것이 핵심입니다.

---

## 마치며

인덱스 설계는 **"어떤 쿼리가 얼마나 자주 실행되나"를 먼저 파악하는 것**에서 시작합니다.

- Gate B처럼 매 요청마다 실행되는 쿼리 → 커버링 인덱스까지 고려한 정밀 설계
- 배치처럼 날짜 범위로 스캔하는 쿼리 → 날짜 컬럼이 선두인 전용 인덱스
- 재처리 워커처럼 상태 + 시간 기준 → 상태 컬럼이 선두인 복합 인덱스

새 기능을 만들 때 쿼리를 작성했다면 반드시 `EXPLAIN`으로 인덱스가 제대로 타는지 확인하는 습관을 들이세요. `type: ALL`이 보이면 인덱스 추가를 검토해야 합니다.

---

## 연결된 개념

- [[explainers/auth/gate-b-billing-grace|Gate B 유예 기간 설계]] — idx_org_service_status에 valid_until이 포함된 설계 결정
- [[multitenancy-rls|Pool 모델 + RLS]] — org_pk가 모든 복합 인덱스 선두인 이유
- [[partitioning|DB 파티셔닝]] — 파티션과 인덱스를 함께 설계하는 방법
- [[enum-vs-varchar-check|ENUM vs VARCHAR+CHECK]] — CHECK constraint가 인덱스 설계에 미치는 영향
> 소스 문서
- [[schema-reference]] — D.12 Gate B 핫패스 인덱스, D.17 org_subscription 인덱스, D.18 pg_webhook 인덱스
