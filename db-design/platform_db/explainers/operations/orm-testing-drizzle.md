---
difficulty: 중
tags: [platform-db, explainer, operations, testing, orm, drizzle]
aliases: [ORM 테스트, Drizzle 테스트, orm-testing-drizzle]
---

# ORM(Drizzle) 테스트 전략 — "타입은 통과하는데 진짜 DB에선 터지는" 갭을 메우는 법

> **대상**: ORM은 써봤지만 ORM 코드를 *어떻게 테스트하는지*는 막막한 개발자 (공부용)
> **연관 문서**: [[testing-strategy|테스트 전략]] · [[architecture|architecture.md §1 토폴로지]] · [[schema-reference|schema-reference.md §A.2 접근 계층]] · [[bola-object-authz|BOLA 객체 인가]] · [[multitenancy-rls|멀티테넌시 격리]]

이 프로젝트는 **PostgreSQL 16 + Drizzle ORM** 스택입니다. 그리고 DB 접근은 오직 `@db-platform` 패키지 함수로만 합니다 — Drizzle 스키마를 직접 참조하는 것은 금지입니다([[schema-reference|§A.2]]). ORM은 SQL을 손으로 안 써도 되게 해주는 편한 도구지만, 바로 그 "편함" 때문에 **테스트에서 잘못된 안도감**을 주기 쉽습니다. 타입 체크는 초록불인데 운영에서 터지는 일이 ORM 코드에서 자주 납니다. 이 문서는 Drizzle 코드를 어디까지 mock하고 어디부터 진짜 DB로 검증해야 하는지, 그리고 그걸 실제로 어떻게 짜는지 공부하는 노트입니다.

핵심 비유 하나를 먼저 깔고 갑니다. **ORM = 통역사**입니다. 내가 한국어(타입 안전한 메서드 체인)로 말하면 통역사가 현지어(SQL)로 옮겨줍니다. 편하죠. 그런데 *통역된 말이 현지에서 진짜 통하는지*는 현지에 가서 대화해 봐야 압니다. 타입 체크는 "내 한국어 문법이 맞나" 검사일 뿐이고, real DB 테스트가 "현지에서 실제로 대화"하는 것입니다.

---

## Q1. ORM이 테스트에 주는 것과, 숨은 함정은 뭔가요?

Drizzle 같은 ORM(정확히는 type-safe **query builder**)이 주는 가장 큰 선물은 **타입 안전 쿼리**입니다. 컬럼명을 오타 내면 컴파일이 안 되고, 존재하지 않는 컬럼으로 `WHERE`를 걸면 IDE가 즉시 빨간 줄을 긋습니다.

```ts
// 컬럼 오타 → 컴파일 단계에서 잡힘 (런타임 SQL 에러 전에)
db.select().from(orgEntitlement).where(eq(orgEntitlement.servce, "ACADEMY"));
//                                                          ^^^^^ TS2551: 'service' 아닌가요?
```

여기까지는 좋습니다. 함정은 **그다음**입니다.

ORM이 검증해 주는 건 "내가 쓴 메서드 체인이 타입에 맞는가"뿐입니다. 정작 **그 체인이 *생성한 SQL*이 진짜 DB에서 의도대로 도는가**는 ORM이 보장하지 않습니다. 그리고 SQL이 의도대로 도는지는 **진짜 DB로만** 검증됩니다.

```text
타입 체크가 통과해도 DB에서 터지는 것들:
  · 복합 인덱스를 안 타서 풀스캔 (타입은 만족, 성능은 폭망)
  · UNIQUE / CHECK / FK 제약 위반 (타입엔 제약 정보가 없음)
  · 트랜잭션 격리·잠금 동작 (ORM은 SQL을 만들 뿐, 격리수준은 DB가 결정)
  · NULL/기본값/타임존이 DB와 스키마 정의가 어긋남
```

그래서 **mock DB**(DB를 가짜 객체로 바꿔치기)는 ORM 테스트에서 가장 위험한 선택입니다. mock은 "내가 이 메서드를 부르면 이 값을 돌려줘"라고 *내가 직접 정한 답*을 돌려줄 뿐, **SQL을 실행하지 않습니다**. 즉 SQL 문법 버그, 제약 위반, 트랜잭션 동작은 *원리적으로* 못 잡습니다. mock으로 통과한 ORM 테스트는 "통역사가 옮긴 말을 안 듣고, 내가 미리 적어둔 모범답안만 읽은" 셈입니다.

```ts
// ❌ ORM을 mock하면 — SQL은 실행조차 안 됨. "내가 정한 답"만 돌아옴
const fakeDb = { query: { lecture: { findFirst: vi.fn().mockResolvedValue({ pk: 1n }) } } };
// → org_pk 조건이 빠졌든, FK가 깨졌든, 인덱스를 안 타든 이 테스트는 늘 초록불

// ✅ 진짜 DB에 같은 쿼리를 던져 — 제약·SQL·격리가 실제로 검증됨
const found = await lectureRepo.findById(orgPk, lectureId); // 실제 Postgres에 SELECT 발사
```

원칙으로 압축하면: **"ORM을 믿되 SQL은 검증하라."** 타입을 믿고 코드를 짜되, 그 코드가 만든 SQL은 진짜 DB로 한 번 더 확인해야 합니다.

> 💡 **한 줄 요약**: ORM은 타입 안전을 주지만 *생성된 SQL*의 정확성은 진짜 DB로만 검증되며, mock DB는 SQL·제약·트랜잭션 버그를 원리적으로 못 잡습니다 — "ORM을 믿되 SQL은 검증하라."

---

## Q2. Drizzle 스키마, 마이그레이션, 테스트 DB는 어떻게 동기화하나요?

Drizzle에는 두 종류의 "진실"이 있습니다.

1. **스키마 정의**(TypeScript): `schema.ts`에 적힌 테이블·컬럼·인덱스 선언. 타입의 원천.
2. **실제 테이블**(PostgreSQL): `CREATE TABLE`로 진짜 만들어진 물리 테이블. 데이터의 원천.

이 둘이 어긋난 상태가 **migration drift(마이그레이션 표류)**입니다. 예를 들어 `schema.ts`에는 `status` 컬럼을 추가했는데 테스트 DB에는 아직 그 컬럼이 없는 상태죠. drift가 위험한 이유는 **테스트를 *거짓 통과*시키기 때문**입니다.

```text
drift 시나리오:
  schema.ts:  lecture 테이블에 (org_pk, status) 복합 UNIQUE 추가함
  테스트 DB:  아직 옛날 스키마 (UNIQUE 없음)

  → "중복 INSERT가 막히는가" 테스트가 통과해버림 (DB에 제약이 없는데도!)
  → 운영 DB엔 제약이 있으니, 운영에서만 다르게 동작 = 최악의 거짓 통과
```

해결은 단순합니다. **테스트 DB에 운영과 *똑같은 마이그레이션*을 적용**하면 됩니다. `drizzle-kit`이 그 도구입니다.

- `drizzle-kit generate` — 스키마 정의 변경분으로 **SQL 마이그레이션 파일**을 만듭니다(버전 관리 대상).
- `drizzle-kit migrate` — 그 마이그레이션 파일들을 **순서대로** DB에 적용합니다. **테스트 DB에도 이걸 써야 합니다.**
- `drizzle-kit push` — 스키마 정의를 DB에 직접 밀어넣습니다(마이그레이션 파일 없이). 빠른 로컬 프로토타이핑엔 편하지만, **운영 경로와 달라서** 테스트에선 권장하지 않습니다 — 운영이 `migrate`로 가는데 테스트가 `push`로 가면, 그 자체가 또 다른 drift입니다.

```ts
// 테스트 셋업: 운영과 "같은 마이그레이션"을 테스트 DB에 적용
import { migrate } from "drizzle-orm/node-postgres/migrator";

await migrate(db, { migrationsFolder: "./drizzle" });
// ↑ 운영에서 돌리는 그 폴더 그대로. 테스트 DB = 운영 스키마의 복제본이 됨
```

여기서 [[online-ddl-migration|온라인 DDL 마이그레이션]]과 이어집니다. 운영에서는 마이그레이션을 무중단으로 적용해야 하므로 "어떤 DDL이 락을 거는가"가 중요한데, **테스트 DB에 같은 마이그레이션을 적용**해 두면 그 마이그레이션이 새 스키마에서 실제로 잘 도는지(컬럼 타입, 기본값, 제약)까지 검증됩니다. drift 감지를 CI에 넣으면(아래 테스트 방법 ③) "스키마 고치고 마이그레이션 안 만든" 실수를 빌드 단계에서 잡습니다.

> 💡 **한 줄 요약**: 스키마 정의와 실제 테이블이 어긋나면(drift) 테스트가 거짓 통과하므로, `drizzle-kit migrate`로 **운영과 똑같은 마이그레이션**을 테스트 DB에 적용해 둘을 일치시킵니다.

---

## Q3. mock으로 할 것과 진짜 DB로 할 것을 어떻게 나누나요?

모든 걸 real DB로 하면 느리고, 모든 걸 mock하면 거짓 통과합니다. **무엇을 검증하느냐**로 가릅니다.

| 검증 대상 | 방법 | 이유 |
|---|---|---|
| 순수 비즈니스 로직 (계산·분기·정책 판단) | **mock** (repo를 가짜로) | DB가 필요 없음. 빠르고 결정적 |
| 쿼리 정확성 / 제약(UNIQUE·FK·CHECK) | **real DB** (Testcontainers) | SQL과 제약은 DB로만 검증 가능 |
| org_pk 테넌트 격리([[bola-object-authz]]·[[multitenancy-rls]]) | **real DB** | 실제로 타 org row가 안 나오는지 확인 |
| 트랜잭션 원자성·롤백 | **real DB** | 트랜잭션 동작은 DB가 결정 |
| 외부 의존(PG 결제·이메일·webhook) | **sandbox / mock** | 외부 시스템은 통제 불가, 비결정적 |

결정 트리로 정리하면:

```text
이 테스트가 무엇을 증명하려 하는가?
│
├─ "주어진 입력에 로직이 올바른 값을 내는가" (DB 무관)
│     → mock (repo를 mock). 수십 ms. 수백 개 돌려도 빠름
│
├─ "SQL/제약/격리/트랜잭션이 실제로 동작하는가"
│     → real DB (Testcontainers Postgres). [[testing-strategy]] 정책 따름
│
└─ "외부(PG·이메일)와의 연동이 맞는가"
      → 외부는 sandbox 또는 계약(contract) mock. 외부 자체를 띄우지 않음
```

여기서 **Testcontainers**는 테스트 시작 시 도커로 진짜 PostgreSQL 16 컨테이너를 띄우고 끝나면 버리는 도구입니다. "진짜 Postgres인데 일회용"이라, real DB의 신뢰성과 테스트의 격리성을 동시에 얻습니다. 프로젝트의 테스트 정책([[testing-strategy]])은 "쿼리·제약·격리는 real DB로"를 기본선으로 잡습니다.

주의할 함정: **`@db-platform` 패키지 함수를 테스트할 때는 패키지를 mock하지 마세요.** 그 패키지 함수의 *존재 이유*가 "org_pk를 강제하고 올바른 SQL을 만드는 것"이므로, 그걸 mock하면 검증 대상 자체가 사라집니다. 반대로 그 패키지를 *소비하는 서비스 로직*을 단위 테스트할 때는 패키지 함수를 mock해도 됩니다(Q4).

> 💡 **한 줄 요약**: 순수 로직은 mock으로 빠르게, 쿼리·제약·org_pk 격리·트랜잭션은 real DB(Testcontainers)로, 외부 시스템은 sandbox/mock으로 — "무엇을 증명하려는가"로 가릅니다.

---

## Q4. repository 패턴은 어떻게 테스트하나요?

**repository pattern(리포지토리 패턴)**은 "DB 접근 코드를 한 계층으로 모아, 서비스 로직이 SQL을 직접 모르게 하는" 구조입니다. 이 프로젝트에선 `@db-platform` 패키지가 그 계층이고, 특히 **base repo가 `org_pk`를 자동 주입**해 BOLA 방어를 프레임워크화합니다([[bola-object-authz]]). 개발자가 매번 손으로 `WHERE org_pk`를 붙이면 언젠가 빠뜨리니, base repo가 대신 강제하는 것이죠.

테스트는 **두 층으로 나뉩니다.**

**층 1 — base repo가 org_pk를 진짜 강제하는가? → real DB로 검증 (가장 중요)**

이건 "통역사가 정말 모든 문장에 테넌트 경계를 붙이는가"를 현지(real DB)에서 확인하는 일입니다. 두 org를 만들고, B 토큰으로 A의 객체를 조회했을 때 *진짜로 안 나오는지* 봅니다.

```ts
// repo.tenant-isolation.spec.ts — Testcontainers Postgres 사용
it("base repo는 타 org row를 절대 반환하지 않는다", async () => {
  const orgA = await seedOrg("한울학원");
  const orgB = await seedOrg("B학원");
  const lectureInA = await lectureRepo.create(orgA.pk, { title: "중1 수학" });

  // B의 org_pk로 A의 lecture를 조회 → base repo가 org_pk 주입 → row 없음
  const found = await lectureRepo.findById(orgB.pk, lectureInA.pk);
  expect(found).toBeNull(); // ★ 핵심 단언: 타 org 객체는 안 보인다

  // 실수로 org_pk를 빼는 API조차 없어야 함 (시그니처가 orgPk 필수)
});
```

**층 2 — 서비스 로직은 repo를 mock해 단위 테스트 (빠름)**

서비스의 관심사는 "정책 분기"지 "SQL"이 아닙니다. 그래서 repo를 mock해 분기만 빠르게 검증합니다.

```ts
// enrollment.service.spec.ts — repo는 mock
it("정원이 꽉 차면 등록을 거부한다", async () => {
  const lectureRepo = { findById: vi.fn().mockResolvedValue({ pk: 1n, capacity: 30, enrolled: 30 }) };
  const svc = new EnrollmentService(lectureRepo as any);
  await expect(svc.enroll(orgPk, lectureId, student)).rejects.toThrow("정원 초과");
  // DB 불필요 — 분기 로직만 검증. 수 ms
});
```

요점: **org_pk 강제 같은 "프레임워크가 책임지는 불변식"은 real DB로, 그 위에 얹힌 "내 정책 로직"은 mock으로.** 두 층을 섞으면(서비스 테스트에서 격리까지 보려 하면) 느리고 초점이 흐려집니다.

> 💡 **한 줄 요약**: base repo의 org_pk 강제는 두 org를 만든 real DB 테스트로 검증하고(타 org → null), 그 위 서비스 정책 로직은 repo를 mock해 빠르게 단위 테스트합니다.

---

## Q5. 트랜잭션은 어떻게 테스트하나요?

트랜잭션은 "여러 SQL을 *전부 성공하거나 전부 실패*하게 묶는" 단위입니다. 결제처럼 원자성이 깨지면 안 되는 흐름의 핵심이죠([[consistency-model|일관성 모델]]). Drizzle은 `db.transaction()`으로 표현합니다.

```ts
await db.transaction(async (tx) => {
  await tx.insert(paymentLedger).values({ orgPk, amount, status: "PAID" });
  await tx.update(orgSubscription).set({ status: "ACTIVE" }).where(eq(orgSubscription.orgPk, orgPk));
  // 둘 중 하나라도 throw → tx 전체 rollback (PAID인데 구독 비활성 같은 부분 상태가 안 생김)
});
```

테스트할 것은 **두 가지**입니다.

**① 원자성 — 중간에 실패하면 전부 롤백되는가 (real DB)**

```ts
it("구독 업데이트가 실패하면 결제 ledger도 롤백된다", async () => {
  await expect(
    db.transaction(async (tx) => {
      await tx.insert(paymentLedger).values({ orgPk, amount: 10000, status: "PAID" });
      throw new Error("PG confirm 실패 시뮬레이션"); // 중간 실패
    }),
  ).rejects.toThrow();

  // ★ ledger에 PAID row가 남으면 안 됨 — 전체 rollback 단언
  const rows = await db.select().from(paymentLedger).where(eq(paymentLedger.orgPk, orgPk));
  expect(rows).toHaveLength(0);
});
```

**② 테스트 격리 — 테스트끼리 데이터가 새지 않게 (per-test rollback 헬퍼)**

테스트마다 테이블을 `TRUNCATE`하면 느립니다. 더 빠르고 깔끔한 패턴은 **각 테스트를 트랜잭션으로 감싸고 끝에 무조건 rollback**하는 것입니다([[testing-strategy]] Q3). 테스트가 무슨 짓을 해도 DB는 깨끗한 상태로 돌아옵니다.

```ts
let tx: Tx;
beforeEach(async () => { tx = await beginTestTx(db); });  // BEGIN
afterEach(async () => { await tx.rollback(); });          // 무조건 되돌림 → 다음 테스트 깨끗
```

> ⚠️ 주의: 이 패턴은 **테스트 대상 코드가 자체적으로 `transaction()`을 또 열면**(중첩 트랜잭션) PostgreSQL에서는 진짜 중첩이 아니라 SAVEPOINT로 처리되어 동작이 미묘해집니다(PG도 트랜잭션 중첩은 SAVEPOINT로 구현). 결제처럼 *코드가 트랜잭션을 직접 여는* 경우엔, per-test rollback 대신 **테스트 후 명시적 cleanup**(TRUNCATE/삭제)이 더 안전합니다. "원자성 자체를 보는 테스트"(①)는 절대 바깥 트랜잭션으로 감싸지 마세요 — rollback 동작을 검증하는데 바깥에서 또 감싸면 의미가 사라집니다.

> 💡 **한 줄 요약**: 원자성은 "중간 실패 시 전체 rollback"을 real DB로 단언하고(결제 ledger가 안 남는지), 테스트 격리는 per-test rollback 헬퍼로 — 단 트랜잭션을 직접 여는 코드엔 명시적 cleanup을 씁니다.

---

## Q6. ORM이 표현 못 하는 것들은? 탈출구는 어디에 있나요?

ORM은 만능 통역사가 아닙니다. **통역이 안 되는 말**이 있습니다. 그럴 땐 **raw SQL escape hatch**(Drizzle의 `sql` 템플릿)로 직접 현지어를 말해야 하고, 그 부분은 *반드시* real DB로 테스트해야 합니다.

ORM이 표현하지 못하거나 어색한 것들:

```text
· 파티션 DDL (PARTITION BY ...)           → 마이그레이션 SQL로 직접. ORM 스키마로 안 됨
· GRANT / REVOKE 권한 부여                 → [[least-privilege-grant]] — DCL은 ORM 영역 밖
· cross-tenant raw SQL (운영 집계)         → internal/ 모듈에서만, sql`...`로 의도를 코드 위치로 노출
· 복합 CHECK 제약 (여러 컬럼 조건)         → 마이그레이션 SQL로 정의. ORM이 일부만 표현
· 복잡한 윈도우 함수·CTE                    → sql`...` raw로 작성
```

```ts
// cross-tenant 집계 — 일반 코드 금지, internal/ 모듈에서만 raw SQL로
// apps/admin-api/src/internal/org-stats.service.ts
const stats = await db.execute(sql`
  SELECT org.type, COUNT(*) AS total
  FROM organization org
  GROUP BY org.type
`); // org_pk 없는 의도를 "코드 위치 + raw SQL"로 명시적으로 드러냄
```

그리고 ORM에서 늘 경계해야 할 두 가지:

**N+1 문제.** ORM은 객체를 "자연스럽게" 순회하게 해주는데, 그 자연스러움이 루프 안에서 쿼리 N번을 몰래 날립니다. 부모 1개를 가져오고 자식 N개를 각각 따로 조회하는 패턴이죠. 테스트로 **발생한 쿼리 수를 단언**하면 잡힙니다(아래 테스트 방법 참고).

```ts
// ❌ N+1 — 강의마다 학생 수를 따로 조회 (강의 100개 → 쿼리 101번)
for (const lec of lectures) lec.count = await studentRepo.countByLecture(orgPk, lec.pk);

// ✅ JOIN/집계 한 방 — 쿼리 1번
const counts = await lectureRepo.findWithStudentCounts(orgPk);
```

**타입↔런타임 갭.** 이게 ORM 테스트의 가장 깊은 함정입니다. **타입은 컴파일 시점에만 검사**됩니다. DB의 `CHECK` 제약, `UNIQUE`, `FK`, NOT NULL 위반 같은 것은 **타입에 표현되지 않으므로 컴파일이 통과하고, 런타임에 DB가 거부**합니다. 즉 "TypeScript가 초록불"이라고 안전한 게 아닙니다 — 제약 위반은 *real DB에 실제로 INSERT를 던져봐야* 드러납니다.

```ts
// 타입은 완벽히 통과 (status는 string 타입) — 그런데 DB CHECK는 ('PAID','PENDING','FAILED')만 허용
await db.insert(paymentLedger).values({ orgPk, amount, status: "WHATEVER" });
//                                                        ^^^^^^^^^^ 컴파일 OK, 런타임에 DB가 거부
// → 이 갭은 real DB 테스트로만 잡힘. mock이면 영원히 못 잡음
```

이래서 다시 Q1의 원칙으로 돌아옵니다. **타입(문법 검사)을 통과해도, 진짜 통하는지는 현지(real DB)에서 대화해 봐야** 합니다.

> 💡 **한 줄 요약**: 파티션 DDL·GRANT·cross-tenant·복합 CHECK는 ORM 밖이라 raw SQL escape hatch(internal/)와 마이그레이션 SQL로 처리하고, N+1과 "타입은 통과하지만 DB CHECK는 런타임에 거부하는" 갭은 real DB 테스트로만 잡힙니다.

---

## 용어 정리

| 용어 | 한 줄 정의 |
|---|---|
| **ORM** | Object-Relational Mapping. 객체/메서드로 DB를 다루게 해주는 계층. 통역사 |
| **query builder** | SQL을 문자열이 아니라 타입 안전한 메서드 체인으로 조립하는 도구. Drizzle이 이 방식 |
| **Drizzle** | 이 프로젝트의 ORM/query builder. 타입 안전 + 얇은 추상화(생성 SQL이 예측 가능) |
| **drizzle-kit** | Drizzle의 CLI. `generate`(마이그레이션 생성)·`migrate`(적용)·`push`(직접 반영) |
| **migration drift** | 스키마 정의(코드)와 실제 테이블(DB)이 어긋난 상태. 테스트를 거짓 통과시킴 |
| **repository pattern** | DB 접근을 한 계층으로 모으는 구조. 여기선 `@db-platform` 패키지가 그 계층 |
| **base repo** | 모든 repo의 부모. `org_pk`를 자동 주입해 BOLA 방어를 프레임워크화 |
| **N+1** | 부모 1개 + 자식 N개를 각각 따로 조회해 쿼리가 N+1번 나가는 성능 함정 |
| **raw SQL escape hatch** | ORM으로 표현 못 하는 SQL을 `sql\`...\`` 템플릿으로 직접 쓰는 탈출구 |
| **Testcontainers** | 테스트 때 도커로 진짜 Postgres를 띄우고 끝나면 버리는 도구. real DB + 격리 |
| **타입↔런타임 갭** | 타입은 컴파일에만 검사됨 — DB CHECK/UNIQUE/FK 위반은 런타임에만 드러나는 간극 |
| **per-test rollback** | 각 테스트를 트랜잭션으로 감싸 끝에 rollback해 DB를 깨끗이 유지하는 격리 기법 |

---

## 테스트 방법

ORM 테스트의 핵심은 **"타입이 못 보는 것을 real DB가 보게 하기"**입니다. 세 가지를 실제로 짜 봅니다. (실제 DB를 띄우는 방법·운영 안전은 [[testing-strategy]] 참고.)

**① Drizzle repo를 Testcontainers Postgres로 테스트 (vitest)**

```ts
// repo.it.spec.ts
import { PostgreSqlContainer } from "@testcontainers/postgresql";
import { drizzle } from "drizzle-orm/node-postgres";
import { migrate } from "drizzle-orm/node-postgres/migrator";
import { Pool } from "pg";

let container: Awaited<ReturnType<PostgreSqlContainer["start"]>>;
let pool: Pool;
let db: ReturnType<typeof drizzle>;

beforeAll(async () => {
  container = await new PostgreSqlContainer("postgres:16").start(); // 진짜 Postgres 16, 일회용
  pool = new Pool({ connectionString: container.getConnectionUri() });
  db = drizzle(pool);
  await migrate(db, { migrationsFolder: "./drizzle" }); // 운영과 같은 마이그레이션
}, 120_000); // 컨테이너 기동 시간 고려해 타임아웃 넉넉히

afterAll(async () => { await pool.end(); await container.stop(); });

it("org_pk 격리 + UNIQUE 제약이 진짜로 동작한다", async () => {
  const a = await seedOrg("한울학원");
  await lectureRepo.create(a.pk, { code: "MATH-1", title: "중1 수학" });

  // 같은 org + 같은 code → UNIQUE 위반 (타입엔 없는 제약 → 런타임에만 드러남)
  await expect(lectureRepo.create(a.pk, { code: "MATH-1", title: "중복" }))
    .rejects.toThrow(/duplicate key|23505/i);   // PG unique_violation = SQLSTATE 23505
});
```

**② migration drift 감지 (스키마 vs 마이그레이션)**

```bash
# CI에서: 스키마 정의로 새 마이그레이션을 "생성 시도"했을 때 변경분이 나오면 = drift
npx drizzle-kit generate --name=drift-check
git diff --exit-code drizzle/   # 새 파일/변경 있으면 비0 종료 → CI 실패
#   → "스키마는 고쳤는데 마이그레이션을 안 만든" 실수를 빌드에서 차단
```

**③ N+1 감지 — 발생 쿼리 수를 단언**

```ts
it("강의 목록 조회는 쿼리 1번이어야 한다 (N+1 금지)", async () => {
  const spy = spyOnQueries(db);            // 실행된 SQL을 수집하는 래퍼
  await lectureRepo.findWithStudentCounts(orgPk);
  expect(spy.count()).toBe(1);             // 101이면 N+1 회귀 → 즉시 실패
});
```

**④ 무엇을 단언하나 (체크리스트)**

```text
□ org_pk 격리: 타 org row가 real DB에서 절대 안 나온다 (null)
□ 제약: UNIQUE/FK/CHECK 위반이 런타임에 진짜로 거부된다 (타입 통과해도)
□ 트랜잭션: 중간 실패 시 전체 rollback (부분 상태가 안 남는다)
□ drift: 스키마 정의 = 적용된 마이그레이션 (drizzle-kit generate가 빈 diff)
□ N+1: 핵심 목록 쿼리의 발생 쿼리 수가 1(또는 상수)이다
□ raw SQL: escape hatch로 쓴 sql`...`는 real DB로 직접 검증한다
□ mock 경계: @db-platform 패키지 자체는 mock하지 않는다 (검증 대상이므로)
```

> 💡 **테스트 한 줄 요약**: 쿼리·제약·격리·트랜잭션은 Testcontainers Postgres로 real 검증하고, drift는 `drizzle-kit generate`(PG 방언) 빈-diff로, N+1은 쿼리 수 단언으로 잡으세요 — 타입이 못 보는 것을 DB가 보게 하는 게 전부입니다.

---

## 마치며

ORM 테스트는 한 질문으로 압축됩니다: **"내가 쓴 타입 안전한 코드가, 진짜 DB에서도 의도대로 도는가?"**

이 질문에 "예"라고 답하려면:

1. 타입은 **문법 검사**일 뿐 — 제약·SQL·트랜잭션은 **real DB(Testcontainers)**로 확인한다.
2. **mock DB는 SQL·제약·트랜잭션 버그를 못 잡는다** — 순수 로직에만 mock을 쓴다.
3. 테스트 DB는 `drizzle-kit migrate`로 **운영과 같은 마이그레이션**을 적용해 drift를 없앤다.
4. `org_pk` 강제(base repo)는 **두 org를 만든 real DB 테스트**로 검증한다(타 org → null).
5. 트랜잭션은 **중간 실패 시 전체 rollback**을 단언하고, 격리는 per-test rollback으로 한다.
6. ORM 밖(DDL·GRANT·cross-tenant·복합 CHECK)은 **raw SQL escape hatch**로 처리하고 real DB로 검증한다.

ORM은 좋은 통역사지만, 통역된 말이 현지에서 통하는지는 현지에 가봐야 압니다. 새 repo 메서드를 짤 때마다 "이거 진짜 DB로 한 번 돌려봤나?"를 먼저 물어보세요 — 그게 타입↔런타임 갭을 메우는 유일한 방법입니다.

---

## 연결된 개념

- [[testing-strategy|테스트 전략]] — 실제 DB(Testcontainers)·운영 DB에서 *어디서* 돌리나(이 문서는 *ORM 레이어*를 어떻게)
- [[bola-object-authz|BOLA 객체 수준 인가]] — base repo의 org_pk 강제가 BOLA 방어이며, real DB 교차 테넌트 테스트로 검증
- [[multitenancy-rls|Pool 모델 멀티테넌시와 RLS]] — PG RLS로 격리를 DB에서 1차 강제하고 앱·테스트로 보강하는 배경
- [[consistency-model|일관성 모델]] — `db.transaction()` 원자성 검증이 결제 일관성의 토대
- [[online-ddl-migration|온라인 DDL 마이그레이션]] — 테스트 DB에 같은 마이그레이션을 적용해 drift를 없애는 이유
- [[least-privilege-grant|최소 권한 GRANT]] — ORM이 표현 못 하는 DCL(GRANT/REVOKE)의 영역
- [[index-design|인덱스 설계]] — 타입은 통과해도 인덱스를 안 타는 쿼리를 real DB로 잡는 맥락
- [[fk-strategy|FK 전략]] — 타입엔 없는 FK 제약이 런타임에 거부되는 갭의 한 예
> 소스 문서
- [[architecture]] — §1 토폴로지, base repo(org_pk 자동 주입)
- [[schema-reference]] — §A.2 접근 계층(@db-platform·Drizzle, 패키지 함수만 호출), §M 계정, §G org_pk 강제
- [[requirements]] — 테스트·격리 관련 요구사항(테넌트 격리·트랜잭션 원자성)
