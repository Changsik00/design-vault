---
difficulty: 초
tags:
  - platform-db
  - explainer
  - p1
  - billing
  - entitlement
  - feature
aliases:
  - feature_limits
  - 기능 한도
  - 불변식 10
  - feature_limits SSOT
---

# feature_limits 3중 정의 우선순위 설명

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[architecture]] §2.1 불변식 #10, [[schema-reference]] §D.10, §D.12, §D.15

코드를 처음 보면 `feature_limits`가 여러 테이블에 나뉘어 있어서 "이걸 어디서 읽어야 하지?"라는 의문이 생깁니다. 결론부터 말하면: **[[gate-b-entitlement|org_entitlement]]`.feature_limits`만 읽으면 됩니다.** 나머지는 이 값을 만들 때 참고용입니다.

---

## Q1. feature_limits가 뭔가요? 왜 JSON으로 저장하나요?

`feature_limits`는 어떤 org가 어떤 기능을 얼마나 사용할 수 있는지 정의하는 한도값입니다.

```json
{
  "daily_uploads": 10,
  "members": 50,
  "storage_gb": 100,
  "ai_queries_per_day": 500
}
```

예를 들어 "이 학원은 하루에 영상을 10개까지 업로드할 수 있다"는 것을 이 JSON이 담습니다.

**왜 JSON인가요?**

서비스마다, 요금제마다 한도 항목 자체가 다릅니다:

```
ACADEMY 서비스 한도 항목:
  daily_uploads, members, storage_gb, ai_queries_per_day

AGENT 서비스 한도 항목:
  agent_sessions_per_day, tokens_per_month, concurrent_agents

MARKET 서비스 한도 항목:
  active_listings, monthly_orders, api_calls_per_hour
```

이걸 컬럼으로 만들면:

```sql
-- 이렇게 하면 안 됩니다
CREATE TABLE org_entitlement (
  daily_uploads           INT,
  members                 INT,
  storage_gb              INT,
  ai_queries_per_day      INT,
  agent_sessions_per_day  INT, -- AGENT 전용
  tokens_per_month        INT, -- AGENT 전용
  active_listings         INT, -- MARKET 전용
  monthly_orders          INT, -- MARKET 전용
  -- 서비스 추가될 때마다 컬럼 추가 필요...
);
```

서비스가 5개, 요금제가 10개라면 수십 개의 컬럼이 생기고 대부분은 NULL입니다. 새 서비스를 추가할 때마다 테이블 스키마를 바꿔야 합니다.

JSON은 이 문제를 해결합니다:
- 서비스별로 다른 항목을 유연하게 담을 수 있음
- 새 서비스 추가 시 스키마 변경 없음
- NULL 컬럼 폭증 없음

**주의**: `feature_limits`는 `JSONB` 컬럼으로 저장합니다. JSONB는 GIN 인덱스로 `@>` 같은 포함 검색을 걸 수 있지만, 런타임 한도 판단은 불변식 #10에 따라 **org별로 entitlement 한 row를 통째로 읽어** 그 안의 JSON을 꺼내는 방식이라(검색·집계가 아니라 단건 조회), 한도 값을 조건으로 검색하거나 집계하는 용도로는 쓰지 않습니다. 즉 GIN을 깔 수는 있어도 이 경로에선 굳이 필요하지 않습니다.

> 💡 **한 줄 요약**: feature_limits를 JSONB로 저장하는 이유는 서비스마다 한도 항목 자체가 달라서, 컬럼으로 만들면 서비스 추가 때마다 스키마를 바꿔야 하는 문제를 피하기 위해서입니다.

---

## Q2. feature_limits가 세 군데(product_feature, plan_definition, org_entitlement)에 있는 이유가 뭔가요?

각 테이블의 역할이 다르기 때문입니다. 마치 "원가 → 정가 → 실제 계산서" 같은 계층 구조입니다:

```
product_feature.limit_value        ← 상품 카탈로그의 기본값 (정적)
         ↓ "BASIC 플랜의 기본 제공 스펙은..."
plan_definition.default_limits     ← 플랜 템플릿의 기본값
         ↓ "이 org가 이 플랜을 구독했을 때 초기값으로 복사"
org_entitlement.feature_limits     ← 이 org의 실제 런타임 한도 (SSOT)
```

각각을 좀 더 설명하면:

**`product_feature.limit_value`** — 상품 카탈로그의 스펙

```sql
-- "BASIC 요금제 상품은 일 업로드 5개, 멤버 20명까지" 정의
INSERT INTO product_feature (product_pk, feature_key, limit_value)
VALUES
  (1, 'daily_uploads', 5),
  (1, 'members', 20);
```

이건 쇼핑몰 상품 설명에 적힌 "스펙"입니다. 실제 org별 한도가 아닙니다.

**`plan_definition.default_limits`** — 플랜 템플릿

```sql
-- "BASIC 플랜 기본 한도"
INSERT INTO plan_definition (plan_code, default_limits)
VALUES ('BASIC', '{"daily_uploads": 5, "members": 20, "storage_gb": 50}');
```

새 org가 BASIC 플랜을 구독하면 이 값을 복사해서 `org_entitlement.feature_limits`의 초기값으로 씁니다. 틀(template)입니다.

**`org_entitlement.feature_limits`** — 이 org의 실제 한도

```sql
-- org_pk=42가 실제로 사용하는 한도
SELECT feature_limits FROM org_entitlement
WHERE org_pk = 42 AND service = 'ACADEMY';
-- 결과: {"daily_uploads": 10, "members": 30, "storage_gb": 50}
```

영업팀이 "이 학원은 VIP라서 업로드 2배로 늘려줘"라고 요청하면 `plan_definition`을 바꾸지 않고 이 org의 `feature_limits`만 수정합니다. 다른 org에 영향 없이 개별 조정이 가능합니다.

> 💡 **한 줄 요약**: `product_feature`는 상품 스펙, `plan_definition`은 플랜 템플릿, `org_entitlement`는 이 org의 실제 적용값입니다. 세 개가 있는 이유는 "상품 정의 → 플랜 기본값 → org별 개별 조정"이라는 계층이 필요해서입니다.

---

## Q3. 런타임에 "이 org의 하루 업로드 한도가 몇 개야?"를 알고 싶을 때 어느 테이블을 봐야 하나요?

**무조건 `org_entitlement.feature_limits`입니다.**

이건 선택이 아닌 규칙입니다. architecture.md 불변식 #10에 명시되어 있습니다:

> `org_entitlement.feature_limits`가 런타임 한도의 최종 권위(SSOT). `product_feature`·`plan_definition.default_limits`는 entitlement 생성 시 초기값 복사용. 런타임 한도 판단 시 이 두 테이블 직접 조회 금지.

코드 예시:

```typescript
// 올바른 방법: org_entitlement에서 읽기
const entitlement = await getEntitlementByService(orgPk, 'ACADEMY');
const limits = entitlement.featureLimits as Record<string, number>;
const dailyUploadLimit = limits['daily_uploads'] ?? Infinity;

if (todayUploadCount >= dailyUploadLimit) {
  throw new UploadLimitExceededException();
}
```

```typescript
// 잘못된 방법: product_feature에서 읽기 (절대 하지 말 것)
const feature = await db.query.productFeature.findFirst({
  where: and(
    eq(productFeature.productPk, productPk),
    eq(productFeature.featureKey, 'daily_uploads')
  )
});
const limit = feature?.limitValue; // ← 이 값은 org별 조정이 반영 안 됨
```

`@platform-db` 패키지에 이미 헬퍼 함수가 있습니다:

```typescript
// billing/index.ts
const limit = await getFeatureLimit(orgPk, 'ACADEMY', 'daily_uploads');
// 내부적으로 org_entitlement.feature_limits를 읽음
// null이면 무제한 처리
```

**왜 `product_feature`를 직접 읽으면 안 되냐고요?**

`org_entitlement.feature_limits`는 처음엔 `plan_definition.default_limits`를 복사해서 만들어집니다. 하지만 이후에 org별로 개별 조정될 수 있습니다.

```
시나리오:
  1. org_42가 BASIC 플랜 구독 → feature_limits = {"daily_uploads": 5}
  2. 영업팀이 VIP 고객이라 daily_uploads를 20으로 올려줌
     → org_entitlement.feature_limits = {"daily_uploads": 20}  ← 변경됨
     → product_feature.limit_value = 5  ← 원래 값 그대로
  3. product_feature를 읽으면 5 (틀린 값)
     org_entitlement를 읽으면 20 (맞는 값)
```

> 💡 **한 줄 요약**: 런타임 한도는 항상 `org_entitlement.feature_limits`에서 읽어야 합니다. `product_feature`나 `plan_definition`은 org별 개별 조정이 반영되지 않아 틀린 값을 줄 수 있습니다.

---

## Q4. 요금제가 변경되면 기존 org의 feature_limits는 어떻게 되나요?

[[subscription-lifecycle|구독]] 요금제(plan)가 변경되어도 **기존 org의 `feature_limits`는 자동으로 바뀌지 않습니다.** 의도적인 설계입니다.

```
시나리오 A: plan_definition의 기본값을 변경
  plan_definition['BASIC'].default_limits = {"daily_uploads": 8}  (5 → 8로 올림)

  → 기존에 BASIC 요금제를 쓰고 있는 org_42의 feature_limits: 여전히 {"daily_uploads": 5}
  → 변경된 플랜으로 새로 구독하는 org들만 {"daily_uploads": 8} 적용
```

이렇게 동작하는 이유:

1. **격리**: 기존 고객의 서비스 조건이 갑자기 바뀌는 것은 계약 위반일 수 있습니다.
2. **유연성**: 일부 org에게만 새 한도를 적용하고 싶을 때 개별 제어 가능합니다.
3. **안전성**: `plan_definition` 실수로 변경해도 기존 고객에게 영향 없습니다.

만약 기존 org에도 새 기본값을 적용하고 싶다면 **명시적으로 마이그레이션**해야 합니다:

```sql
-- 특정 플랜을 쓰는 모든 org의 feature_limits 업데이트
-- (주의: 이건 의도적 일괄 변경, 신중하게 실행)
UPDATE org_entitlement
SET feature_limits = feature_limits || '{"daily_uploads": 8}'::jsonb
WHERE plan_code = 'BASIC'
  AND service = 'ACADEMY';
-- 중첩 키만 바꾸려면 jsonb_set(feature_limits, '{daily_uploads}', '8'::jsonb)

-- 변경 후 perm_version 갱신 (bulk라서 신중하게 처리)
```

**업그레이드/다운그레이드 시:**

org가 BASIC → PRO로 플랜을 업그레이드하면 결제 성공 webhook 처리 트랜잭션에서 `feature_limits`도 함께 갱신됩니다:

```sql
-- 결제 단일 트랜잭션 내에서
INSERT INTO org_entitlement (org_pk, product_code, service, status, feature_limits, plan_code, valid_until)
VALUES (?, ?, 'ACADEMY', 'ACTIVE', ?, 'PRO', ?)
ON CONFLICT (org_pk, service) DO UPDATE SET
  status = 'ACTIVE',
  feature_limits = EXCLUDED.feature_limits,  -- PRO 플랜 한도로 교체
  plan_code = EXCLUDED.plan_code,
  valid_until = EXCLUDED.valid_until;
```

> 💡 **한 줄 요약**: `plan_definition`을 바꿔도 기존 org의 한도는 자동으로 안 바뀝니다. 기존 고객에게 새 한도를 적용하려면 명시적인 마이그레이션 스크립트가 필요합니다.

---

## Q5. feature_limits를 JSON이 아닌 컬럼으로 만들면 안 되나요?

"JSON보다 컬럼이 더 명확하지 않나?"라는 생각은 자연스럽습니다. 컬럼으로 만들었을 때 어떻게 되는지 살펴보겠습니다.

**컬럼 방식 시도:**

```sql
CREATE TABLE org_entitlement (
  -- ...기존 컬럼들...

  -- feature limits 컬럼 방식
  limit_daily_uploads       INT,
  limit_members             INT,
  limit_storage_gb          INT,
  limit_ai_queries_per_day  INT,

  -- 아직 없지만 나중에 추가될 수 있는 것들
  limit_agent_sessions      INT,   -- AGENT 서비스 전용
  limit_tokens_per_month    INT,   -- AGENT 서비스 전용
  limit_active_listings     INT,   -- MARKET 서비스 전용
  limit_monthly_orders      INT    -- MARKET 서비스 전용
);
```

문제가 바로 보입니다:

1. **서비스별 컬럼 혼재**: ACADEMY org는 `limit_agent_sessions`가 항상 NULL, AGENT org는 `limit_daily_uploads`가 항상 NULL. 테이블이 희소(sparse)해집니다.

2. **새 서비스 = 스키마 변경**: FITNESS 서비스를 추가하면 `limit_workout_plans`, `limit_video_calls` 등의 컬럼을 추가해야 합니다. 대형 테이블에 컬럼 추가 = 잠재적 락.

3. **유연한 한도 추가 불가**: 갑자기 "동시 접속자 수" 한도를 추가해야 한다면? 컬럼을 또 추가해야 합니다.

4. **서비스별 org 분리 후 마이그레이션 어려움**: 향후 FITNESS가 별도 DB로 분리되면 컬럼을 이동해야 합니다.

**JSONB의 트레이드오프:**

물론 JSONB에도 단점이 있습니다:

```typescript
// JSONB라 타입 안전성이 약함
const limits = entitlement.featureLimits as Record<string, number>;
//                                          ^^^^ 타입 캐스팅 필요

// 특정 한도로 org 목록을 검색하려면 별도 인덱스가 필요
// (JSONB라 GIN 인덱스로 @> 포함 검색은 가능하지만,
//  런타임 한도는 org별 단건 조회라 이 경로에선 안 씀)
```

하지만 이 시스템에서 feature_limits는 org별로 꺼내는 용도가 주이고, 한도값 기준으로 여러 org를 검색하는 케이스는 드뭅니다. 트레이드오프를 따졌을 때 JSON이 더 나은 선택입니다.

**타입 안전성을 보완하려면:**

```typescript
// 서비스별 feature_limits 타입 정의 (Zod 또는 타입 정의)
interface AcademyFeatureLimits {
  daily_uploads: number | null; // null = 무제한
  members: number | null;
  storage_gb: number | null;
  ai_queries_per_day: number | null;
}

// 읽을 때 검증
const limits = AcademyFeatureLimitsSchema.parse(entitlement.featureLimits);
```

> 💡 **한 줄 요약**: 컬럼 방식은 서비스 추가 때마다 스키마를 바꿔야 하고 NULL 컬럼이 넘칩니다. JSONB는 타입 안전성이 약한 대신 유연하게 확장됩니다. 이 시스템의 "한도 항목이 서비스마다 다르고 늘어난다"는 특성 때문에 JSONB를 선택했습니다.

---

## 테스트 방법

> 🧪 *실제 DB·ORM·운영에서 돌리는 법*: [[testing-strategy]] · [[orm-testing-drizzle]]

핵심 보장은 "런타임 한도는 항상 `org_entitlement.feature_limits`(SSOT)에서만 읽힌다"입니다. PostgreSQL 16 + Testcontainers(`PostgreSqlContainer`) + Drizzle(node-postgres) + vitest로 실제 DB를 띄워 3중 SSOT 우선순위와 JSONB 한도 조회를 검증합니다. 추가로 entitlement UPSERT가 `(org_pk, service)` 충돌 키를 쓰므로 **상품 교체(BASIC→PRO) 시 새 행을 만들지 않고 같은 행을 갱신**하는지도 검증합니다(org는 서비스당 entitlement 1개).

```typescript
// feature-limits.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";
import { PostgreSqlContainer, StartedPostgreSqlContainer } from "@testcontainers/postgresql";
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import { and, eq } from "drizzle-orm";

let container: StartedPostgreSqlContainer;
let db: ReturnType<typeof drizzle>;

beforeAll(async () => {
  container = await new PostgreSqlContainer("postgres:16").start();
  const pool = new Pool({ connectionString: container.getConnectionUri() });
  db = drizzle(pool);
  await migrate(db); // product_feature · plan_definition · org_entitlement 생성
}, 60_000);

afterAll(async () => {
  await container.stop();
});

describe("feature_limits 3중 SSOT 우선순위", () => {
  beforeEach(async () => {
    await db.delete(orgEntitlement);
    await db.delete(planDefinition);
    await db.delete(productFeature);
  });

  it("런타임 한도는 org_entitlement.feature_limits만 읽는다 (불변식 #10)", async () => {
    // product_feature=5, plan_definition=5 인데 org만 20으로 개별 조정
    await db.insert(productFeature).values({ productPk: 1, featureKey: "daily_uploads", limitValue: 5 });
    await db.insert(planDefinition).values({ planCode: "BASIC", defaultLimits: { daily_uploads: 5 } });
    await db.insert(orgEntitlement).values({
      orgPk: 42, productCode: "ACADEMY_BASIC", service: "ACADEMY", status: "ACTIVE",
      planCode: "BASIC", featureLimits: { daily_uploads: 20 }, // VIP 개별 상향
    });

    const limit = await getFeatureLimit(42, "ACADEMY", "daily_uploads");
    expect(limit).toBe(20); // ✅ org_entitlement 값 (5가 아님)
  });

  it("JSONB 한도 조회: org별 단건 row를 읽어 JSON에서 키를 꺼낸다", async () => {
    await db.insert(orgEntitlement).values({
      orgPk: 7, productCode: "ACADEMY_PRO", service: "ACADEMY", status: "ACTIVE",
      featureLimits: { daily_uploads: 10, members: 50, storage_gb: 100 },
    });

    const ent = await db.query.orgEntitlement.findFirst({
      where: and(eq(orgEntitlement.orgPk, 7), eq(orgEntitlement.service, "ACADEMY")),
    });
    const limits = ent!.featureLimits as Record<string, number>;
    expect(limits["members"]).toBe(50);
    expect(limits["ai_queries_per_day"] ?? Infinity).toBe(Infinity); // 미정의 키 = 무제한
  });

  it("plan_definition을 바꿔도 기존 org의 feature_limits는 자동으로 안 바뀐다", async () => {
    await db.insert(orgEntitlement).values({
      orgPk: 99, productCode: "ACADEMY_BASIC", service: "ACADEMY", status: "ACTIVE",
      planCode: "BASIC", featureLimits: { daily_uploads: 5 },
    });
    // 플랜 템플릿만 상향
    await db.insert(planDefinition).values({ planCode: "BASIC", defaultLimits: { daily_uploads: 8 } });

    const limit = await getFeatureLimit(99, "ACADEMY", "daily_uploads");
    expect(limit).toBe(5); // ✅ 명시적 마이그레이션 없이는 그대로
  });

  it("entitlement UPSERT는 (org_pk, service) 충돌로 상품 교체 시 같은 행을 갱신한다 (BASIC→PRO)", async () => {
    // org는 서비스당 entitlement 1개. ON CONFLICT (org_pk, service)로 UPSERT.
    function upsertEntitlement(productCode: string, planCode: string, limits: Record<string, number>) {
      return db.insert(orgEntitlement)
        .values({
          orgPk: 7, productCode, service: "ACADEMY", status: "ACTIVE",
          planCode, featureLimits: limits,
        })
        .onConflictDoUpdate({
          target: [orgEntitlement.orgPk, orgEntitlement.service], // (org_pk, service)
          set: {
            productCode, planCode,
            featureLimits: limits, // PRO 한도로 교체
          },
        });
    }

    await upsertEntitlement("ACADEMY_BASIC", "BASIC", { daily_uploads: 5 });
    await upsertEntitlement("ACADEMY_PRO", "PRO", { daily_uploads: 50 }); // 업그레이드

    const rows = await db.select().from(orgEntitlement)
      .where(and(eq(orgEntitlement.orgPk, 7), eq(orgEntitlement.service, "ACADEMY")));
    expect(rows).toHaveLength(1);                              // ✅ 새 행 안 생기고 1개 유지
    expect(rows[0].productCode).toBe("ACADEMY_PRO");           // product_code 갱신됨
    expect((rows[0].featureLimits as Record<string, number>).daily_uploads).toBe(50); // 한도 갱신됨
  });
});
```

---

## 마치며

feature_limits의 3중 구조를 다시 정리하면:

```
product_feature.limit_value
    = "BASIC 상품의 스펙은 daily_uploads: 5"
    = 카탈로그 정보, 변경 잦지 않음

plan_definition.default_limits
    = "BASIC 플랜을 처음 구독하는 org의 초기 한도"
    = 템플릿, 신규 entitlement 생성 시 복사

org_entitlement.feature_limits  ← 항상 이것만 읽어라
    = "org_42의 현재 실제 한도"
    = 런타임 SSOT, org별 개별 조정 반영됨
```

코드를 짤 때 체크리스트:

```typescript
// 체크: 기능 한도 확인 시 이 함수만 써야 함
const limit = await getFeatureLimit(orgPk, service, 'daily_uploads');

// 위반 패턴: 절대 이렇게 하지 말 것
// await db.query.productFeature.findFirst(...)   ← 금지
// await db.query.planDefinition.findFirst(...)   ← 금지
```

불변식 #10은 단순한 권장사항이 아닙니다. 이걸 지키지 않으면 VIP 고객의 개별 조정이 무시되거나, 플랜 템플릿 변경이 기존 고객에게 의도치 않게 적용되는 버그가 생깁니다.

---

## 연결된 개념

- [[gate-b-entitlement|Gate B & 엔타이틀먼트]] — org_entitlement.feature_limits가 런타임 SSOT인 전체 맥락
- [[subscription-lifecycle|구독 상태 머신]] — 플랜 변경 시 feature_limits 갱신 시점
- [[index-design|인덱스 설계]] — JSON 컬럼 조회 방식
> 소스 문서
- [[architecture]] — §2.1 불변식 #10 (feature_limits SSOT)
- [[schema-reference]] — D.10 product_feature, D.15 plan_definition, D.12 org_entitlement
