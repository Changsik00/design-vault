# Pool 모델 멀티테넌시와 RLS 개념 설명

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [`architecture.md §8`](../architecture.md) · [`schema-reference.md §G`](../schema-reference.md)

"멀티테넌트", "RLS", "Pool 모델" — `platform_db` 코드를 처음 보면 이 단어들이 낯설게 느껴집니다. 이 문서는 우리 시스템이 왜 이 구조를 택했고, 어떻게 테넌트 간 데이터를 격리하는지 설명합니다.

---

## Q1. 멀티테넌트(multi-tenant)가 뭔가요? 왜 org마다 DB를 따로 안 쓰나요?

**멀티테넌트**는 하나의 시스템에 여러 고객(테넌트)이 공존하는 구조입니다. 여기서 테넌트는 `organization` — 즉, 각 학원이나 회사입니다.

비유를 들면 이렇습니다. 아파트 한 동에 여러 세대가 살지만, 세대마다 건물을 따로 짓지 않습니다. 각 호수가 독립된 공간을 갖되 건물 인프라(엘리베이터, 배관 등)는 공유합니다. 멀티테넌트 DB도 같습니다.

**왜 org마다 DB를 따로 안 쓰나요?**

"DB-per-tenant" 방식의 문제점:

```
학원 1개당 DB 1개라면...

학원 100개 → DB 100개 → 커넥션 풀 100개 × 최소 5 = 500개 커넥션
학원 1000개 → DB 1000개 → 커넥션 풀 1000개 × 5 = 5000개 커넥션
                                                      └─ MySQL 서버 폭발
```

- 운영 비용: DB 인스턴스 100개를 각각 백업·모니터링·패치해야 합니다
- 스키마 마이그레이션: 컬럼 하나 추가하면 DB 100개 전부 DDL 실행
- 신규 학원 가입: DB 생성 + 마이그레이션 실행 → 가입 속도 느려짐

그래서 `platform_db`는 **Pool 모델**을 씁니다. 하나의 DB에 모든 학원 데이터가 공존하되, 모든 테이블에 `org_pk` 컬럼을 두어 행(row) 단위로 격리합니다:

```sql
-- 학원 A의 데이터
SELECT * FROM org_entitlement WHERE org_pk = 1 AND service = 'ACADEMY';
--                                  ^^^^^^^^^^^^ 항상 이 조건 필수

-- 학원 B의 데이터
SELECT * FROM org_entitlement WHERE org_pk = 2 AND service = 'ACADEMY';
--                                  ^^^^^^^^^^^^ 다른 학원, 다른 데이터
```

불변식 #3이 이것을 강제합니다:

```
불변식 #3: 모든 도메인 테이블은 org_pk NOT NULL. 예외 없음 — 테넌트 경계는 불변식.
```

> 💡 **한 줄 요약**: DB 하나에 모든 학원 데이터를 담되, 모든 쿼리에 `org_pk` 조건을 붙여서 다른 학원 데이터가 섞이지 않게 합니다.

---

## Q2. RLS(Row-Level Security)가 뭔가요? 들어봤는데 정확히 모르겠어요.

RLS는 "어떤 유저가 어떤 행을 읽을 수 있는지"를 **DB 자체가 정책으로 강제하는** 기능입니다.

비유하자면, 도서관에서 책에다 "이 책은 성인 회원만 볼 수 있음"이라는 라벨을 붙여두면, 사서(앱 코드)가 실수로 책을 꺼내주려 해도 도서관 시스템(DB)이 알아서 막아주는 것과 같습니다. 앱 코드가 "WHERE 조건을 빠뜨려도" DB 레벨에서 이미 필터링이 걸립니다.

PostgreSQL에서 RLS가 어떻게 동작하는지 보면 이해가 쉽습니다:

```sql
-- PostgreSQL에서는 DB 자체가 정책 강제
CREATE POLICY tenant_isolation_policy ON org_entitlement
  USING (org_pk = current_setting('app.org_pk')::bigint);
-- 이 정책이 있으면, 아무리 앱이 실수해도 다른 org 데이터가 나오지 않음

-- 관리자 오버라이드 정책
CREATE POLICY admin_override ON org_entitlement
  USING (current_setting('app.role') = 'SUPER_ADMIN');
-- SUPER_ADMIN은 모든 org 접근 가능
```

PostgreSQL의 RLS는 강력한 안전망입니다. 개발자가 쿼리에서 `WHERE org_pk = ?`를 빠뜨려도, DB가 현재 세션의 `app.org_pk` 설정값으로 자동 필터를 추가합니다. "사서가 책을 주려 해도 잠금 장치가 막는" 구조입니다.

> 💡 **한 줄 요약**: RLS는 앱 코드 실수와 상관없이 DB 자체가 "당신은 이 행만 볼 수 있어요"를 강제하는 DB 레벨 보안 기능입니다.

---

## Q3. PostgreSQL은 RLS가 있는데 우리가 쓰는 MySQL은 없다고요? 그러면 어떻게 막나요?

맞습니다. MySQL 8은 RLS를 지원하지 않습니다. 이것은 이 설계의 가장 큰 구조적 제약 중 하나로, `architecture.md`에도 솔직하게 명시되어 있습니다:

> **D10**: 멀티테넌시: RLS 없음 → CI 린트 보강. MySQL은 RLS 없음 (정직한 자인).

그래서 현재는 **앱 레이어에서 강제**합니다. 세 가지 방어선을 씁니다.

**방어선 1: Gate 함수에 orgPk 파라미터 필수화**

```typescript
// 앱에서 강제 — 모든 Gate 함수에 orgPk 필수 파라미터
const entitlement = await getEntitlementByService(orgPk, service);
//                                                 ↑↑↑↑↑↑ 없으면 함수 자체가 동작 안 함

// 함수 시그니처 자체가 orgPk를 강제
async function getEntitlementByService(
  orgPk: number,   // ← 이게 없으면 TypeScript 컴파일 오류
  service: EntitlementService
): Promise<OrgEntitlement | null> {
  return db.query.orgEntitlement.findFirst({
    where: and(
      eq(orgEntitlement.orgPk, orgPk),  // ← 항상 포함
      eq(orgEntitlement.service, service)
    )
  });
}
```

**방어선 2: 스키마 레벨 NOT NULL 강제**

```sql
-- 모든 도메인 테이블의 org_pk는 NOT NULL
CREATE TABLE org_entitlement (
  org_pk BIGINT UNSIGNED NOT NULL,  -- ← NULL이면 INSERT 자체 실패
  ...
);
```

**방어선 3: CI 린트 (P1 — 현재 미구현, 곧 추가 예정)**

```typescript
// 금지 패턴 (P1 CI 린트로 차단 예정)
// 이런 쿼리가 코드에 있으면 빌드 실패
SELECT * FROM org_entitlement WHERE service = 'ACADEMY'; -- org_pk 누락!
```

세 방어선의 효과:

```
방어선         │ 막는 것
───────────────┼──────────────────────────────────
NOT NULL       │ 삽입 시 org_pk 누락 → DB 오류
Gate 함수 시그니처 │ 함수 호출 시 orgPk 빠뜨리면 → 타입 오류
CI 린트 (P1)  │ 리뷰 전에 빌드 단계에서 → 차단
```

PostgreSQL RLS와 비교하면 분명 약합니다. 하지만 MySQL이 회사 표준이라 어쩔 수 없는 선택이고, 이 세 방어선으로 최대한 보완합니다.

> 💡 **한 줄 요약**: MySQL에는 RLS가 없어서, Gate 함수 파라미터 강제 + NOT NULL 스키마 + CI 린트 세 겹으로 대신 막습니다.

---

## Q4. SUPER_ADMIN은 여러 org 데이터를 봐야 하는데, 격리를 어떻게 하나요?

`platform_db`는 SUPER_ADMIN role 자체를 거부합니다(ADR-042). 대신 **아키텍처 분리** 방식을 씁니다.

일반 비즈니스 로직과 운영 집계를 **코드 위치로 분리**합니다:

```
일반 모듈 (academy-api, agent-api...)
  └── 항상 org_pk 필수. 다른 org 데이터 접근 불가

internal/ 모듈 또는 *-admin 서비스 (별도 인증, 내부망만 접근)
  └── raw SQL로 cross-tenant 조회 가능
  └── 외부 라우팅에 노출 금지
  └── 감사 로그 별도 기록
```

예를 들어 "전체 학원 가입 현황을 집계하는 관리자 기능"은 이렇게 분리됩니다:

```typescript
// ❌ 일반 코드에서 org_pk 없이 조회 — 금지
const allOrgs = await db.query.organization.findMany();  // 전 테넌트 노출

// ✅ internal/ 모듈에서만 가능 (별도 인증 필요)
// apps/admin-api/src/internal/org-stats.service.ts
const stats = await db.execute(sql`
  SELECT COUNT(*) as total_orgs, org.type
  FROM organization org
  GROUP BY org.type
`);  // raw SQL로 의도를 코드 위치로 드러냄
```

**왜 SUPER_ADMIN role을 거부하나요?**

Admin role이 있으면:
1. 탈취 시 전 테넌트 데이터 노출 (단일 장애점)
2. 비즈니스 로직 코드에 보안 우회 코드가 섞임
3. "이 개발자는 Admin이니까 org_pk 없이 조회해도 되겠지" 관례가 생김

코드 위치로 분리하면 cross-tenant 조회가 어디에 있는지 grep 한 줄로 찾을 수 있고, 접근 권한도 별도로 제어할 수 있습니다.

> 💡 **한 줄 요약**: SUPER_ADMIN role은 없습니다. cross-tenant 조회는 `internal/` 모듈이나 별도 admin 서비스에서만 가능하고, 코드 위치 자체가 접근 통제선입니다.

---

## Q5. 실수로 쿼리에서 `org_pk` 조건을 빠뜨리면 어떻게 되나요? 막을 방법이 있나요?

`org_pk` 조건을 빠뜨리면 **다른 학원 데이터가 그대로 나옵니다**. MySQL에는 RLS가 없어서 DB가 자동으로 막아주지 않습니다.

어떻게 이 실수를 막나요?

**현재 구현된 방어:**

```typescript
// @aiagent/db-platform 패키지의 모든 게이트 함수는 orgPk를 필수 파라미터로 받음
// → TypeScript 컴파일러가 빠뜨리면 오류

export async function getActiveMembership(
  userPk: number,
  orgPk: number  // ← 빠뜨리면 컴파일 오류
): Promise<Membership | null>

export async function checkGateB(
  orgPk: number,  // ← 빠뜨리면 컴파일 오류
  service: EntitlementService = "ACADEMY"
): Promise<void>
```

**자가진단 DoD (Definition of Done):**

```
① 개발자가 org_pk 빠뜨리면 다른 테넌트 데이터 유출되나?
   → 차단 (NOT NULL + gate 함수 필수 파라미터)

② 클라이언트가 tenant_id를 위조하면 뚫리나?
   → 차단 (JWT 검증 후 server-side에서 orgPk 조회)
```

**P1 CI 린트 (곧 추가 예정):**

```typescript
// 이 패턴이 코드에 있으면 빌드 실패
db.select().from(orgEntitlement)  // org_pk 조건 없음 → 차단
  .where(eq(orgEntitlement.service, 'ACADEMY'))

// 올바른 패턴
db.select().from(orgEntitlement)
  .where(
    and(
      eq(orgEntitlement.orgPk, orgPk),  // ← 반드시 포함
      eq(orgEntitlement.service, 'ACADEMY')
    )
  )
```

**클라이언트 위조 방어:**

클라이언트가 요청에 `orgPk: 999`를 임의로 넣어도, 서버는 그 값을 그대로 쓰지 않습니다:

```typescript
// ❌ 위험한 패턴 — 클라이언트 값 직접 사용
const orgPk = req.body.orgPk;  // 위조 가능

// ✅ 안전한 패턴 — JWT에서 추출한 firebase_uid로 DB 조회
const user = await getUserByFirebaseUid(jwt.uid);
// X-Org-Pk 헤더로 org를 받더라도, membership 확인으로 접근 권한 검증
const membership = await getActiveMembership(user.pk, orgPk);
// membership이 없으면 403 — 다른 org에 접근 불가
```

> 💡 **한 줄 요약**: 현재는 TypeScript 타입 강제 + NOT NULL 스키마로 막고, P1에서 CI 린트까지 추가하면 세 겹 방어가 완성됩니다.

---

## Q6. Pool 모델 말고 다른 방법(DB-per-tenant)은 왜 안 썼나요? 나중에 전환할 수 있나요?

**다른 방법들과 비교:**

| 방식 | 설명 | 장점 | 단점 |
|---|---|---|---|
| Pool 모델 (현재) | 공유 DB + org_pk 행 격리 | 단순, 저비용, 쉬운 마이그레이션 | RLS 없으면 앱 책임 |
| Schema-per-tenant | org마다 별도 스키마 | 격리 명확 | MySQL에서 db=schema라 커넥션 풀 폭발 |
| DB-per-tenant | org마다 별도 DB | 완전 격리 | 운영비↑, 마이그레이션 지옥, 과도한 현재 규모 |

MySQL에서 "schema-per-tenant"는 실질적으로 "DB-per-tenant"와 같습니다. 테넌트 100개면 DB 100개, 커넥션 풀 100개가 되어 메모리와 운영 비용이 선형으로 올라갑니다.

**나중에 전환할 수 있나요?**

네, 가능합니다. 그리고 전환 조건(트리거)도 미리 정해두었습니다:

```
분리 트리거 — 하나라도 충족되면 분리 착수:

T1: 단일 org가 고속 테이블 행수의 20% 이상
    → 해당 테넌트만 전용 DB로 분리

T2: 조회 P99 > 500ms
    → 해당 테이블 별도 인스턴스로 분리

T3: usage_log 월 500만 건 또는 50GB+
    → OLAP(ClickHouse/BigQuery)으로 이관

T4: ISMS-P/GDPR 계약 체결
    → 물리 격리 DB + 리전 분리
```

`org_pk` 단위로 격리되어 있기 때문에, 분리도 `org_pk` 단위로 가능합니다:

```
분리 단계:
1. Read replica 추가
2. 고속 테이블 분리 (chat_db, analytics_db)
3. 특정 org → 전용 DB 이사 (무중단, org_pk 기준)
```

지금의 Pool 모델은 "현재 규모에서 합리적인 선택"이고, 트리거에 도달하면 분리할 수 있도록 처음부터 `org_pk`를 모든 테이블에 박아둔 것입니다. 미래를 위한 복선이라고 생각하면 됩니다.

> 💡 **한 줄 요약**: 지금은 비용·복잡도 대비 Pool 모델이 맞고, T1~T4 트리거에 도달하면 `org_pk` 단위로 무중단 분리할 수 있도록 설계되어 있습니다.

---

## 마치며

멀티테넌시와 RLS는 처음엔 추상적으로 느껴지지만, 결국 하나의 질문으로 귀결됩니다: **"내 쿼리가 다른 학원 데이터를 절대 반환하지 않는가?"**

이 질문에 "예"라고 답하려면:

1. 모든 쿼리에 `WHERE org_pk = ?` 조건이 있어야 합니다
2. Gate 함수를 통해서만 데이터에 접근해야 합니다 (`getEntitlementByService`, `getActiveMembership` 등)
3. cross-tenant 집계가 필요하면 `internal/` 모듈에서만 해야 합니다

새 기능을 만들 때 DB 쿼리를 작성한다면 "이 쿼리에 org_pk 조건이 있나?"를 반드시 확인하세요. CI 린트가 아직 완성되지 않았기 때문에, 지금은 코드 리뷰에서 서로 체크해주는 것이 중요합니다.
