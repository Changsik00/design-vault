---
tags:
  - platform-db
  - explainer
  - p1
  - schema
  - mysql
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

하지만 ENUM의 진짜 문제는 **나중에 값을 추가할 때** 터집니다.

우리 서비스는 지금 `ACADEMY`, `MARKET`, `AGENT`, `YOUTUBE`, `STORE`를 지원하는데, 내일 `FITNESS` 서비스가 추가된다고 해봅시다. ENUM이라면 이렇게 해야 합니다:

```sql
-- ENUM 방식: 컬럼 자체를 수정해야 함
ALTER TABLE product
  MODIFY COLUMN service ENUM('ACADEMY','MARKET','AGENT','YOUTUBE','STORE','FITNESS');
```

이 한 줄이 **수백만 행 테이블에서 서비스를 수십 분 동안 멈출 수 있습니다.**

반면 `VARCHAR+CHECK` 방식이라면:

```sql
-- VARCHAR+CHECK 방식: 제약 조건만 교체하면 됨
ALTER TABLE product
  DROP CONSTRAINT chk_product_service,
  ADD CONSTRAINT chk_product_service
    CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE','FITNESS'));
```

이 방식은 InnoDB에서 **테이블 락 없이** 온라인으로 처리됩니다. 즉, 서비스 중단 없이 새 서비스를 추가할 수 있습니다.

> 💡 **한 줄 요약**: ENUM은 "작성"은 편하지만 "변경"이 위험하고, VARCHAR+CHECK는 "변경"이 안전합니다. 서비스는 계속 늘어나니 VARCHAR+CHECK가 답입니다.

---

## Q2. "[[online-ddl-migration|테이블 락]](lock)"이 뭔가요? ENUM 변경이 왜 서비스를 멈추게 하나요?

테이블 락은 "이 테이블 지금 공사 중이니 아무도 읽거나 쓰지 말것"을 의미합니다.

상상해보세요. 공사 중인 도로를 통제하는 것처럼, DB가 테이블 전체를 잠근 채로 작업하는 겁니다. 그 동안 API 요청은 전부 대기 상태가 됩니다.

MySQL에서 `ENUM` 타입 컬럼을 변경하면 내부적으로 이런 일이 생깁니다:

```
1. 테이블 락 획득 (이 순간부터 읽기/쓰기 모두 대기)
2. 새로운 컬럼 정의로 테이블 구조 변경
3. 기존 행을 새 구조로 전부 복사 (1000만 행이면 1000만 번 복사)
4. 락 해제
```

예시로 `product` 테이블에 행이 1000만 개라면:

```
ENUM 변경 소요 시간 예시:
  - 100만 행  → 약 5~10분 락
  - 1000만 행 → 약 수십 분~1시간 락
  - 그 동안 해당 테이블을 읽는 API는 전부 timeout 또는 대기
```

> ⚠️ **MySQL 8.0 참고**: MySQL 8.0은 일부 ENUM 변경을 instant/inplace DDL로 빠르게 처리하지만, **값 추가**는 여전히 테이블 rebuild가 필요한 경우가 많습니다. "ENUM은 항상 빠르다"가 아니라 "값 추가 시 rebuild(락) 위험이 남는다"로 이해하는 게 안전합니다.

실제 SaaS 서비스라면 이게 바로 **서비스 장애**입니다.

반면 CHECK constraint는 테이블 행을 복사하지 않고 메타데이터만 변경하므로 거의 즉시 처리됩니다.

```
CHECK constraint 변경:
  - 행 복사 없음
  - 메타데이터(스키마 정의)만 변경
  - 소요 시간: 대부분 1초 미만
```

> 💡 **한 줄 요약**: 테이블 락은 "전체 테이블 공사 중" 표시판입니다. ENUM 변경은 이 공사를 강제로 시작시키고, 행이 많을수록 공사 시간이 길어져 서비스가 멈춥니다.

---

## Q3. CHECK constraint가 뭔가요? ENUM이랑 결과는 같은데 왜 더 나은가요?

CHECK constraint는 "이 컬럼에 들어올 수 있는 값의 조건"을 정의하는 규칙입니다.

```sql
-- 현재 product 테이블의 실제 정의
CREATE TABLE product (
  pk        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  service   VARCHAR(50) NOT NULL,
  -- ...

  -- CHECK constraint: 이 값들만 허용
  CONSTRAINT chk_product_service
    CHECK (service IN ('ACADEMY','MARKET','AGENT','YOUTUBE','STORE'))
);
```

사용자 입장에서는 ENUM과 똑같습니다. `service = 'INVALID'`를 넣으려 하면 DB가 에러를 냅니다.

그런데 **구조적으로 완전히 다릅니다**:

| 구분 | ENUM | VARCHAR + CHECK |
|---|---|---|
| 저장 방식 | 내부 숫자 코드로 변환 저장 | 그냥 문자열 그대로 저장 |
| 값 추가 방법 | 컬럼 타입 자체를 변경 (테이블 재구성) | 제약 조건만 교체 (메타데이터 변경) |
| 테이블 락 | 발생 (수십 분 가능) | 발생하지 않음 |
| 인덱스 영향 | 컬럼 변경 → 인덱스 재구성 | 없음 |
| 코드에서 읽을 때 | 문자열로 보임 (MySQL이 변환) | 그냥 문자열 |

CHECK constraint의 또 다른 장점은 **조건을 자유롭게 표현**할 수 있다는 점입니다:

```sql
-- CHECK는 더 복잡한 조건도 가능
CONSTRAINT chk_amount CHECK (amount_minor > 0)
CONSTRAINT chk_self_ref CHECK (parent_org_pk != child_org_pk)
CONSTRAINT chk_expires CHECK (expires_at > created_at)
```

ENUM은 이런 표현이 불가능합니다.

> 💡 **한 줄 요약**: CHECK constraint는 ENUM과 사용자에게 보이는 결과는 같지만, 내부 구현이 달라서 값을 추가할 때 테이블 재구성 없이 제약 규칙만 갈아끼울 수 있습니다.

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

**여전히 ENUM으로 남아있는 컬럼들:**

```sql
-- org_subscription.pg_provider (→ [[architecture]] §3.1 D6 미적용 사례 참조)
pg_provider ENUM('TOSS','STRIPE','PAYPAL','MANUAL') NOT NULL

-- payment_ledger.pg_provider
pg_provider ENUM('TOSS','STRIPE','PAYPAL','MANUAL') NOT NULL

-- billing_event.event_type
event_type ENUM('SUBSCRIPTION_START','SUBSCRIPTION_END','PLAN_CHANGE','INVOICE_PAID','INVOICE_FAILED') NOT NULL

-- pg_webhook_event.pg_provider
pg_provider ENUM('TOSS','STRIPE','PAYPAL') NOT NULL
```

**ENUM이어도 괜찮은 컬럼들 (변경 빈도 낮음):**

```sql
-- 사람/머신 구분: 거의 변경 없음
type ENUM('HUMAN','SERVICE','SYSTEM') NOT NULL

-- 결제 유형: 추가될 일이 드묾
type ENUM('CHARGE','REFUND','CHARGEBACK','CREDIT') NOT NULL

-- 구독 빌링 주기: 추가 가능성 낮음
billing_cycle ENUM('MONTHLY','ANNUAL','ONE_TIME','USAGE') NOT NULL
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
 PG 추가 시 ALTER MODIFY COLUMN 잠금이 D6가 피하려던 바로 그것임."
```

`pg_provider`가 위험한 이유:

```
현재 payment_ledger.pg_provider ENUM 상황:
  - payment_ledger는 결제마다 행이 추가되는 대형 테이블
  - 1년 운영 시 수백만 행 가능
  - 여기에 'KAKAO_PAY'를 추가해야 한다면?
  → ALTER MODIFY COLUMN → 수백만 행 전체 재구성 → 수십 분 락
  → 이게 바로 D6가 막으려던 상황
```

마이그레이션 계획:

```sql
-- 후속 예정 마이그레이션
-- 4개 테이블 동시 작업 필요:
-- org_subscription, payment_ledger, billing_event, pg_webhook_event

ALTER TABLE payment_ledger
  MODIFY COLUMN pg_provider VARCHAR(50) NOT NULL DEFAULT 'MANUAL',
  ADD CONSTRAINT chk_ledger_provider
    CHECK (pg_provider IN ('TOSS','STRIPE','PAYPAL','MANUAL'));

-- billing_event.event_type도 동일하게 처리
ALTER TABLE billing_event
  MODIFY COLUMN event_type VARCHAR(50) NOT NULL,
  ADD CONSTRAINT chk_billing_event_type
    CHECK (event_type IN ('SUBSCRIPTION_START','SUBSCRIPTION_END',
                          'PLAN_CHANGE','INVOICE_PAID','INVOICE_FAILED'));
```

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
