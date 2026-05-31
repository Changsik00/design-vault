---
difficulty: 초
tags:
  - platform-db
  - explainer
  - p1
  - schema
  - postgres
  - ddl
  - design-decision
aliases:
  - ENUM vs VARCHAR
  - D6 원칙
  - CHECK constraint
---

# ENUM vs VARCHAR+CHECK (D6 원칙) 설명

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[architecture]] §3.1 D6, [[schema-reference]] §D.9, §K

`platform_db`에서 서비스 식별자(`service` 컬럼)는 ENUM 대신 `VARCHAR(50) + CHECK constraint`로 설계되어 있습니다. 처음 보면 "ENUM이 더 간단하지 않나?" 싶겠지만, 여기엔 운영 장애를 막기 위한 명확한 이유가 있습니다.

---

## Q1. ENUM을 쓰면 더 간편한데, 왜 VARCHAR+CHECK 방식을 쓰나요?

ENUM이 편한 건 맞습니다. 값이 목록에 없으면 DB가 자동으로 막아주고, 코드도 간단해 보이죠.

하지만 PostgreSQL의 네이티브 `ENUM`(= `CREATE TYPE ... AS ENUM`)에는 운영상 까다로운 제약이 있습니다.

**① 네이티브 ENUM은 "전역 타입"으로 묶입니다.** ENUM은 컬럼이 아니라 데이터베이스 전역의 타입(`pg_type`)으로 존재하고, 여러 테이블·컬럼이 그 타입을 공유합니다. 값 하나를 다루는 일이 컬럼 로컬이 아니라 타입 전역의 변경이 되어 버립니다.

**② `ALTER TYPE ... ADD VALUE`는 트랜잭션 블록 안에서 실행할 수 없습니다.** PostgreSQL에서 ENUM에 값을 추가하는 `ALTER TYPE ... ADD VALUE`는 (자동 커밋이 아닌) `BEGIN ... COMMIT` 트랜잭션 블록 내부에서 거부됩니다. 마이그레이션 도구가 "여러 DDL을 한 트랜잭션으로 묶어 실패 시 통째 롤백"하는 패턴을 쓰는데, ENUM 값 추가만 그 트랜잭션에서 빠져나와야 합니다. 또 한 번 추가한 값은 간단히 제거할 수도 없습니다.

**③ 값 제거·이름 변경·순서 조정이 번거롭습니다.** 잘못 추가한 값을 되돌리려면 사실상 타입을 새로 만들어 컬럼을 갈아끼우는 큰 작업이 됩니다.

우리 서비스는 지금 `ACADEMY`, `MARKET`, `AGENT`, `YOUTUBE`, `STORE`를 지원하는데, 내일 `FITNESS` 서비스가 추가된다고 해봅시다. `VARCHAR+CHECK` 방식이라면 **그 컬럼의 CHECK 제약 하나만** 교체하면 됩니다.

```sql
-- VARCHAR+CHECK 방식: 그 테이블·그 컬럼의 제약만 교체 (테이블 로컬)
ALTER TABLE product
  DROP CONSTRAINT chk_product_service,
  ADD CONSTRAINT chk_product_service
    CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE','FITNESS'));
```

CHECK 제약 추가는 행을 재작성하지 않고 카탈로그(메타데이터)만 갱신하므로, 일반 트랜잭션 안에서 빠르게 끝납니다(기존 행 검증이 필요하면 `NOT VALID`로 추가 후 `VALIDATE`로 분리할 수도 있습니다). 즉, 서비스 중단 없이 새 서비스를 추가할 수 있고, 변경이 전역 타입이 아니라 그 테이블에만 국한됩니다.

> 🐬 **MySQL이라면**: ENUM 값 추가가 더 위험합니다. `ALTER TABLE ... MODIFY COLUMN service ENUM(...)`이 수백만 행 테이블에서 전체 rebuild + 테이블 락을 유발할 수 있어 수십 분 서비스를 멈출 수 있습니다. (PostgreSQL의 `ALTER TYPE ADD VALUE`는 비차단이지만 위 ①②③ 제약이 따로 있습니다.)

> 💡 **한 줄 요약**: ENUM은 "작성"은 편하지만, PostgreSQL에선 전역 타입 결합·트랜잭션 블록 제약·되돌리기 어려움이 있고, VARCHAR+CHECK는 변경이 테이블 로컬이라 안전합니다. 서비스는 계속 늘어나니 VARCHAR+CHECK가 답입니다.

---

## Q2. ENUM 값을 바꾸는 게 왜 까다로운가요? CHECK는 왜 더 안전한가요?

PostgreSQL에서 ENUM 값 *추가*(`ALTER TYPE ... ADD VALUE`) 자체는 테이블을 rebuild하지 않고 비차단으로 끝납니다. 즉 "값 추가 = 긴 락"은 PostgreSQL에선 사실이 아닙니다. 문제는 다른 곳에 있습니다.

**① 마이그레이션 트랜잭션과 안 맞습니다.** Drizzle 등 마이그레이션 도구는 보통 여러 DDL을 한 트랜잭션으로 묶어 "전부 성공 아니면 전부 롤백"하려 합니다. 그런데 `ALTER TYPE ... ADD VALUE`는 트랜잭션 블록 안에서 실행할 수 없어서, ENUM 값 추가만 별도 처리해야 하고 부분 실패 시 정리가 번거롭습니다.

```sql
-- 트랜잭션 블록 안에서는 거부됨
BEGIN;
  ALTER TYPE service_enum ADD VALUE 'FITNESS';  -- ❌ 트랜잭션 블록 내 실행 불가
COMMIT;
```

**② 되돌리기·재정렬이 어렵습니다.** 잘못 추가한 값을 빼거나 이름을 바꾸려면 사실상 새 타입을 만들고 컬럼 타입을 통째로 `ALTER COLUMN ... TYPE`으로 바꿔야 하는데, 이때는 **테이블 rewrite**(전체 행 재작성)가 일어나 대형 테이블에서 위험합니다.

```
ENUM 타입 자체를 교체할 때 (값 제거/재정렬 등):
  ALTER TABLE ... ALTER COLUMN ... TYPE new_enum USING ...
  → 전체 행 rewrite + 그동안 ACCESS EXCLUSIVE 락 → 대형 테이블이면 서비스 멈춤
```

**③ 전역 타입이라 변경 영향 범위가 넓습니다.** ENUM은 데이터베이스 전역 타입이라, 그 타입을 공유하는 모든 컬럼이 함께 영향을 받습니다.

반면 CHECK constraint 교체는 그 테이블·그 컬럼에만 국한되고, 행을 재작성하지 않고 카탈로그만 갱신하므로 거의 즉시 처리됩니다.

```
CHECK constraint 변경:
  - 행 rewrite 없음 (NOT VALID 후 VALIDATE로 검증 락도 분리 가능)
  - 카탈로그(스키마 정의)만 변경
  - 영향 범위: 그 테이블 한정
  - 소요 시간: 대부분 1초 미만
```

> 🐬 **MySQL이라면**: 사정이 더 나쁩니다. ENUM 값 추가가 `ALTER TABLE ... MODIFY COLUMN`으로 테이블 전체 rebuild + 락을 유발해, 1000만 행이면 수십 분~1시간 동안 그 테이블을 읽는 API가 전부 timeout/대기에 빠질 수 있습니다 — 즉 값 추가 단계부터 이미 서비스 장애입니다.

> 💡 **한 줄 요약**: PostgreSQL에서 ENUM은 값 추가는 비차단이지만 트랜잭션 블록 제약·되돌리기 시 rewrite·전역 타입 결합이 부담입니다. CHECK는 변경이 테이블 로컬이고 카탈로그만 갱신해 거의 즉시 끝납니다.

---

## Q3. CHECK constraint가 뭔가요? ENUM이랑 결과는 같은데 왜 더 나은가요?

CHECK constraint는 "이 컬럼에 들어올 수 있는 값의 조건"을 정의하는 규칙입니다.

```sql
-- 현재 product 테이블의 실제 정의
CREATE TABLE product (
  pk        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  service   VARCHAR(50) NOT NULL,
  -- ...

  -- CHECK constraint: 이 값들만 허용
  CONSTRAINT chk_product_service
    CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE'))
);
```

사용자 입장에서는 ENUM과 똑같습니다. `service = 'INVALID'`를 넣으려 하면 DB가 에러를 냅니다.

그런데 **구조적으로 완전히 다릅니다**:

| 구분 | ENUM (PostgreSQL 네이티브) | VARCHAR + CHECK |
|---|---|---|
| 정의 범위 | DB 전역 타입(`pg_type`) | 그 테이블·그 컬럼 로컬 |
| 값 추가 | `ALTER TYPE ADD VALUE` (비차단, 단 트랜잭션 블록 내 불가) | 제약 조건만 교체 (카탈로그 변경) |
| 값 제거/재정렬 | 타입 교체 → `ALTER COLUMN TYPE` → 테이블 rewrite | CHECK 목록만 수정 |
| 변경 영향 범위 | 그 타입을 쓰는 모든 컬럼 | 그 테이블 한정 |
| 조건 표현력 | 값 목록만 | 임의 조건식 가능 |

CHECK constraint의 또 다른 장점은 **조건을 자유롭게 표현**할 수 있다는 점입니다:

```sql
-- CHECK는 더 복잡한 조건도 가능
CONSTRAINT chk_amount CHECK (amount_minor > 0)
CONSTRAINT chk_self_ref CHECK (parent_org_pk != child_org_pk)
CONSTRAINT chk_expires CHECK (expires_at > created_at)
```

ENUM은 이런 표현이 불가능합니다.

> 💡 **한 줄 요약**: CHECK constraint는 ENUM과 사용자에게 보이는 결과는 같지만, 전역 타입이 아니라 테이블 로컬 규칙이라 값을 바꿀 때 타입 교체·rewrite 없이 제약 규칙만 갈아끼울 수 있습니다.

---

## Q4. 현재 코드에서 어떤 컬럼이 ENUM이고 어떤 게 VARCHAR+CHECK인가요?

현재 코드베이스 기준으로 정리하면 이렇습니다:

**VARCHAR+CHECK로 구현된 컬럼 (D6 원칙 적용됨):**

```sql
-- product.service — 어떤 서비스 상품인지
service VARCHAR(50) NOT NULL,
CONSTRAINT chk_product_service
  CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE'))

-- org_entitlement.service — 이 org가 어느 서비스 권한을 가졌는지
service VARCHAR(50) NOT NULL,
CONSTRAINT chk_entitlement_service
  CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE'))
```

**여전히 네이티브 ENUM 타입으로 남아있는 컬럼들** (PostgreSQL에서는 `CREATE TYPE ... AS ENUM`으로 정의한 전역 타입을 컬럼이 참조합니다):

```sql
-- 전역 타입 정의 (예시)
CREATE TYPE pg_provider_enum AS ENUM ('TOSS','STRIPE','PAYPAL','MANUAL');
CREATE TYPE billing_event_type_enum AS ENUM
  ('SUBSCRIPTION_START','SUBSCRIPTION_END','PLAN_CHANGE','INVOICE_PAID','INVOICE_FAILED');

-- org_subscription.pg_provider (→ [[architecture]] §3.1 D6 미적용 사례 참조)
pg_provider pg_provider_enum NOT NULL

-- payment_ledger.pg_provider
pg_provider pg_provider_enum NOT NULL

-- billing_event.event_type
event_type billing_event_type_enum NOT NULL

-- pg_webhook_event.pg_provider
pg_provider pg_provider_enum NOT NULL
```

**ENUM이어도 괜찮은 컬럼들 (변경 빈도 낮음):**

```sql
-- 사람/머신 구분: 거의 변경 없음
CREATE TYPE identity_type_enum AS ENUM ('HUMAN','SERVICE','SYSTEM');
type identity_type_enum NOT NULL

-- 결제 유형: 추가될 일이 드묾
CREATE TYPE ledger_type_enum AS ENUM ('CHARGE','REFUND','CHARGEBACK','CREDIT');
type ledger_type_enum NOT NULL

-- 구독 빌링 주기: 추가 가능성 낮음
CREATE TYPE billing_cycle_enum AS ENUM ('MONTHLY','ANNUAL','ONE_TIME','USAGE');
billing_cycle billing_cycle_enum NOT NULL
```

핵심 기준은 **"앞으로 값이 추가될 가능성이 있는가?"**입니다. `service`처럼 새 서비스가 계속 생기는 컬럼은 VARCHAR+CHECK, `CHARGE/REFUND`처럼 거의 안 변하는 컬럼은 ENUM이어도 큰 문제없습니다.

> 💡 **한 줄 요약**: 서비스 식별자처럼 "앞으로 늘어날 목록"은 VARCHAR+CHECK, 결제 타입처럼 "거의 안 변하는 목록"은 ENUM — 이 기준으로 구분합니다.

---

## Q5. pg_provider, billing_event.event_type는 왜 아직 ENUM으로 남아있나요? 언제 바뀌나요?

솔직히 말하면, **실수로 누락된 것**입니다.

`service` 컬럼에 D6 원칙을 적용하면서 `pg_provider`와 `billing_event.event_type`에는 같은 원칙을 적용하지 않았습니다. AI 리뷰(R8)에서 이 불일치가 지적되었습니다:

```
"D6 원칙이 service 컬럼에는 적용됐지만,
 pg_provider(4개 테이블)와 billing_event.event_type에는 미적용.
 특히 pg_provider는 대형 테이블인 payment_ledger에 있어서
 향후 값 제거/재정렬이 필요해 타입 교체로 가면 테이블 rewrite가
 D6가 피하려던 바로 그 위험을 다시 부른다."
```

`pg_provider`가 위험한 이유:

```
현재 payment_ledger.pg_provider 네이티브 ENUM 상황:
  - payment_ledger는 결제마다 행이 추가되는 대형 테이블
  - 1년 운영 시 수백만 행 가능
  - 여기에 'KAKAO_PAY'를 추가만 한다면 ALTER TYPE ADD VALUE로 비차단 처리 가능
  - 그러나 잘못 넣은 값을 빼거나 순서를 바꿔야 하면?
    → 타입 교체 + ALTER COLUMN TYPE → 수백만 행 전체 rewrite → 수십 분 ACCESS EXCLUSIVE 락
  → 이게 바로 D6가 막으려던 상황 (CHECK였다면 목록만 수정하면 끝)
```

마이그레이션 계획:

```sql
-- 후속 예정 마이그레이션
-- 4개 테이블 동시 작업 필요:
-- org_subscription, payment_ledger, billing_event, pg_webhook_event

ALTER TABLE payment_ledger
  ALTER COLUMN pg_provider TYPE VARCHAR(50)
    USING pg_provider::text,                       -- ENUM → 문자열로 캐스팅
  ALTER COLUMN pg_provider SET DEFAULT 'MANUAL';
ALTER TABLE payment_ledger
  ADD CONSTRAINT chk_ledger_provider
    CHECK (pg_provider IN ('TOSS','STRIPE','PAYPAL','MANUAL'));

-- billing_event.event_type도 동일하게 처리
ALTER TABLE billing_event
  ALTER COLUMN event_type TYPE VARCHAR(50)
    USING event_type::text;
ALTER TABLE billing_event
  ADD CONSTRAINT chk_billing_event_type
    CHECK (event_type IN ('SUBSCRIPTION_START','SUBSCRIPTION_END',
                          'PLAN_CHANGE','INVOICE_PAID','INVOICE_FAILED'));
```

> 🐬 **MySQL이라면**: 위 두 단계를 `ALTER TABLE ... MODIFY COLUMN pg_provider VARCHAR(50) ...`로 표현하며, 타입 변경이 대형 테이블에서 전체 rebuild + 락을 유발할 수 있어 [[online-ddl-migration|단계적 마이그레이션]]이 더 절실합니다.

현재 테이블이 아직 대형화되기 전, 서비스 초기인 지금이 마이그레이션 적기입니다. 미룰수록 행이 쌓여서 마이그레이션 비용이 올라갑니다.

> 💡 **한 줄 요약**: `pg_provider` ENUM은 D6 원칙 적용이 누락된 케이스로, 후속에서 VARCHAR+CHECK로 마이그레이션 예정입니다. 지금 수정하지 않으면 나중에 PG 추가할 때 잠금 장애가 납니다.

---

## 마치며

D6 원칙을 한 문장으로 요약하면: **"값 목록이 나중에 늘어날 컬럼은 ENUM 대신 VARCHAR+CHECK를 써라."**

새 서비스를 추가하는 건 비즈니스 성장을 의미합니다. 그 성장을 DB 구조가 발목 잡지 않도록, 처음부터 변경 비용이 낮은 설계를 선택한 것입니다.

신규 서비스 추가 시 체크리스트:

```sql
-- 1. org_entitlement CHECK 업데이트
ALTER TABLE org_entitlement
  DROP CONSTRAINT chk_entitlement_service,
  ADD CONSTRAINT chk_entitlement_service
    CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE','NEW_SERVICE'));

-- 2. product CHECK 업데이트
ALTER TABLE product
  DROP CONSTRAINT chk_product_service,
  ADD CONSTRAINT chk_product_service
    CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE','NEW_SERVICE'));

-- 3. architecture.md §2.1 불변식 #5 목록 업데이트
-- 4. checkGateB() 호출 시 service 파라미터 명시적 전달 확인
```

이 두 줄의 DDL이 운영 무중단으로 실행된다는 것이 D6의 핵심 가치입니다.

---

## 연결된 개념

- [[online-ddl-migration|온라인 DDL & 마이그레이션]] — 테이블 락이 서비스에 미치는 영향 상세
- [[index-design|인덱스 설계]] — CHECK constraint와 인덱스의 조합
> 소스 문서
- [[architecture]] — §3.1 D6 결정 (service VARCHAR+CHECK)과 D6 미적용 사례 (R8 AI 리뷰)
- [[schema-reference]] — D.9 product (service 컬럼 예시), D.16-D.18 (pg_provider ENUM 미적용 사례)
- [[architecture]] — §3.1 D6 미적용 사례 (pg_provider ENUM → VARCHAR+CHECK 계획)
