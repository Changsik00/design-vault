# ENUM vs VARCHAR+CHECK — 타입 선택이 DDL 비용을 어떻게 바꾸나

## 배경

컬럼 값이 고정된 목록일 때 어떤 타입을 쓸 것인가.

```sql
-- 선택 A: ENUM
service ENUM('ACADEMY', 'MARKET', 'AGENT') NOT NULL

-- 선택 B: VARCHAR + CHECK constraint
service VARCHAR(50) NOT NULL,
CONSTRAINT chk_service CHECK (service IN ('ACADEMY', 'MARKET', 'AGENT'))
```

두 방식 모두 허용되지 않는 값을 막는다. 그런데 차이가 있다 — **새 값을 추가할 때**.

---

## ENUM의 문제 — ALTER TABLE 비용

새 서비스 `STORE`를 추가해야 한다:

```sql
-- ENUM에 값 추가
ALTER TABLE org_entitlement
  MODIFY COLUMN service ENUM('ACADEMY', 'MARKET', 'AGENT', 'STORE') NOT NULL;
```

**MySQL에서 이 ALTER가 하는 일**:
1. 테이블 전체 row를 새 스키마로 재작성 (table rebuild)
2. 재작성 동안 exclusive lock → 다른 쿼리 전부 대기
3. 테이블이 클수록 소요 시간 비례 증가

```
org_entitlement row 100만 건
→ ALTER 실행 시간: 수 분
→ 그 동안 Gate B 체크 전부 대기
→ 서비스 중단과 다름없음
```

MySQL 8.0은 일부 ENUM 변경에 instant/inplace DDL을 지원하지만, 값 **추가**는 여전히 테이블 rebuild가 필요한 경우가 있음.

---

## VARCHAR + CHECK constraint — 온라인 DDL

```sql
-- CHECK constraint만 변경 (Online DDL, 테이블 lock 없음)
ALTER TABLE org_entitlement
  DROP CONSTRAINT chk_entitlement_service,
  ADD CONSTRAINT chk_entitlement_service
    CHECK (service IN ('ACADEMY', 'MARKET', 'AGENT', 'STORE'));
```

InnoDB의 CHECK constraint 변경은 메타데이터 수정만 → **테이블 lock 없음** → 운영 중 즉시 적용.

---

## 우리 적용 사례

```sql
-- org_entitlement.service: 서비스 추가 가능성이 높아 VARCHAR+CHECK 사용
CREATE TABLE org_entitlement (
  ...
  service VARCHAR(50) NOT NULL,
  CONSTRAINT chk_entitlement_service CHECK (service IN (
    'ACADEMY', 'MARKET', 'AGENT', 'YOUTUBE', 'STORE'
  )),
  ...
);

-- product.service도 동일
CREATE TABLE product (
  ...
  service VARCHAR(50) NOT NULL,
  CONSTRAINT chk_product_service CHECK (service IN (
    'ACADEMY', 'MARKET', 'AGENT', 'YOUTUBE', 'STORE'
  )),
  ...
);
```

새 서비스 FITNESS 추가 시:
```sql
-- 두 테이블 CHECK만 수정 → 무중단
ALTER TABLE org_entitlement
  DROP CONSTRAINT chk_entitlement_service,
  ADD CONSTRAINT chk_entitlement_service
    CHECK (service IN ('ACADEMY', 'MARKET', 'AGENT', 'YOUTUBE', 'STORE', 'FITNESS'));
```

---

## ENUM을 써도 괜찮은 경우

```sql
-- 절대 바뀌지 않는 값 목록
type ENUM('HUMAN', 'SERVICE', 'SYSTEM') NOT NULL  -- identity_user.type
-- 3가지 이상 추가되지 않을 것이 확실

action ENUM('GRANTED', 'REVOKED') NOT NULL  -- user_consent_event.action
-- 동의/철회 두 가지 외에 다른 action이 생길 일 없음
```

기준: **미래에 값이 추가될 가능성이 있으면 VARCHAR+CHECK, 없으면 ENUM**.

---

## 트레이드오프

| 항목 | ENUM | VARCHAR+CHECK (우리) |
|---|---|---|
| 저장 효율 | 1~2 bytes (정수 내부 저장) | 실제 문자열 저장 |
| 값 추가 DDL | Table rebuild (lock) | Metadata only (무중단) |
| 가독성 | DB 툴에서 바로 보임 | 동일 |
| 코드 자동완성 | ORM 타입 지원 | ORM 타입 지원 (동일) |

---

## 관련 파일

| 파일 | 내용 |
|---|---|
| `core/schema-reference.md §K` | 서비스 확장 방법 (DDL 예시) |
| `core/architecture.md §3.1` | 불변식 #5: service는 VARCHAR+CHECK |
