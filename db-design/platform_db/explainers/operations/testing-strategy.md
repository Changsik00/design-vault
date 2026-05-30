---
difficulty: 고급
tags:
  - platform-db
  - explainer
  - operations
  - testing
  - ci
  - production
aliases:
  - 테스트 전략
  - testing strategy
  - 실제 DB 테스트
  - production 테스트
  - Testcontainers
---

# 테스트 전략 설명 — 실제 DB에서, 그리고 운영 DB에서 안전하게 테스트하기

> **대상**: "테스트는 짜야 하는데 *어디서 어떻게* 돌리지?"가 막막한 개발자 (공부용 · 사수 모드)
> **연관 문서**: [[bdd-scenarios]] · [[e2e-journeys]] · [[orm-testing-drizzle]] · [[operability|operability.md]]

각 설계 문서의 **테스트 방법** 절은 "*무엇을* 단언하나"를 알려줍니다. 이 문서는 그 한 단계 위 — "*어디서* 그 테스트를 돌리나, 그리고 **운영 중인 DB**는 어떻게 건드리나"를 다룹니다. platform_db는 MySQL 특화(파티션·GRANT·CHECK·FK·트랜잭션)라 *진짜 DB*에서만 검증되는 게 많고, 동시에 **모든 서비스가 의존하는 공유 코어**라 운영 DB를 잘못 건드리면 전 서비스가 흔들립니다. 그 둘을 어떻게 다루는지가 핵심입니다.

---

## Q1. 왜 "진짜 DB"로 테스트해야 하나요? mock으로 빠르게 하면 안 되나요?

비즈니스 로직(순수 함수)은 mock으로 빠르게 테스트하는 게 맞습니다. 하지만 **platform_db의 핵심 보장 대부분은 *DB 엔진*이 강제**하므로, mock DB로는 그게 진짜 작동하는지 알 수 없습니다.

mock으로는 못 잡는 것들:

```
❌ mock DB가 통과시키는 진짜 버그들
- org_pk 빠진 쿼리 → BOLA (mock은 WHERE 절을 평가 안 함)
- CHECK 제약 위반 (service='academy' 소문자) → mock은 ENUM/CHECK를 모름
- append-only GRANT 위반 (UPDATE payment_ledger) → mock엔 권한 개념 없음
- 트랜잭션 롤백 (중간 실패 시 전부 취소) → mock은 진짜 트랜잭션이 아님
- FK·UNIQUE·파티션 프루닝 → 전부 엔진 기능
```

비유하면, mock으로 인가·격리를 테스트하는 건 **운전 시뮬레이터로 실제 브레이크를 점검하는 것**과 같습니다. 핸들 조작(로직)은 시뮬레이터로 연습되지만, 브레이크가 *실제로 차를 멈추는지*(DB가 실제로 막는지)는 진짜 차로만 확인됩니다.

그래서 우리 테스트는 두 층으로 나뉩니다:

```
순수 로직 (CASL 규칙 조합, 금액 계산 …)  → mock/단위 (빠름, 수천 개)
DB가 강제하는 것 (격리·제약·트랜잭션·권한) → 진짜 MySQL (Testcontainers, 수백 개)
사용자 여정                                → e2e ([[e2e-journeys]])
```

> 💡 **한 줄 요약**: org_pk 격리·CHECK·GRANT·트랜잭션 같은 핵심 보장은 *DB 엔진*이 강제하므로, mock이 아니라 진짜 MySQL로만 검증됩니다. 순수 로직만 mock으로 빠르게 합니다.

---

## Q2. 테스트용 "진짜 DB"는 어떻게 띄우나요? (Testcontainers)

매번 테스트마다 공용 DB를 쓰면 서로 데이터를 오염시킵니다. 해법은 **테스트가 시작할 때 일회용(ephemeral) MySQL을 띄우고 끝나면 버리는 것** — **Testcontainers**입니다.

```typescript
// 테스트 시작 시 일회용 MySQL 8 컨테이너 기동
import { MySqlContainer } from "@testcontainers/mysql";
import { migrate } from "drizzle-orm/mysql2/migrator";

let container, db;
beforeAll(async () => {
  container = await new MySqlContainer("mysql:8.0").start();   // 깨끗한 MySQL 1개
  db = drizzle(await mysql.createConnection(container.getConnectionUri()));
  await migrate(db, { migrationsFolder: "./drizzle" });         // 실제 마이그레이션 적용
});
afterAll(async () => { await container.stop(); });               // 끝나면 폐기
```

핵심: 스키마를 손으로 만들지 말고 **실제 마이그레이션(drizzle-kit)으로 적용**합니다. 그래야 "테스트 DB = 운영 DB와 같은 스키마"가 보장됩니다(스키마 drift 방지 — [[orm-testing-drizzle]]).

```
로컬 SQLite로 대체하면 안 되나요? → ❌
  MySQL 특화(파티션·CHECK·VARBINARY·ENUM·GRANT)를 SQLite는 다르게/안 지원.
  "테스트는 운영과 같은 엔진(MySQL 8)으로." 안 그러면 테스트 거짓 통과.
```

> 💡 **한 줄 요약**: Testcontainers로 테스트마다 일회용 MySQL 8을 띄우고, 스키마는 실제 마이그레이션으로 적용합니다. SQLite 대체는 MySQL 특화 기능을 못 살려 거짓 통과를 부릅니다.

---

## Q3. 테스트끼리 데이터가 섞이지 않게 하려면요? (격리 전략)

테스트 100개가 같은 DB를 쓰면 한 테스트가 만든 행이 다음 테스트를 깨뜨립니다. 격리 방법 세 가지, 빠른 순:

**① per-test 트랜잭션 롤백 (가장 빠름, 기본)**

각 테스트를 트랜잭션으로 감싸고 끝에 **commit 대신 rollback**합니다. 디스크에 아무것도 안 남습니다.

```typescript
beforeEach(async () => { tx = await db.transaction(); });   // 시작
afterEach(async () => { await tx.rollback(); });            // 무조건 롤백 → 흔적 0
// 테스트는 tx로 쿼리 → 끝나면 전부 사라짐
```

**② truncate-and-seed (트랜잭션으로 못 감쌀 때)**

append-only **GRANT 테스트**나 **DDL(파티션 DROP) 테스트**처럼 트랜잭션 밖에서 검증해야 하는 경우, 테스트 후 테이블을 비우고 시드를 다시 깝니다.

```typescript
afterEach(async () => { await truncateAll(); await seedBaseline(); });
```

**③ DB-per-worker (병렬 테스트)**

vitest를 병렬로 돌리면 워커마다 별도 DB(또는 스키마)를 줘서 서로 안 섞이게 합니다.

```
워커1 → platform_db_test_1
워커2 → platform_db_test_2   ← org_pk 격리와 같은 원리를 테스트 인프라에도 적용
```

우리 설계 특성상 주의: **append-only(GRANT)·파티션·`@VerifyOnDb`** 테스트는 ①(트랜잭션 롤백)로 안 되는 경우가 있어 ②를 씁니다. 권한 거부는 트랜잭션 안에서도 일어나니 ①로 가능하지만, 실제 커넥션 계정을 바꿔야 하면 별도 커넥션이 필요합니다([[least-privilege-grant]] 테스트).

> 💡 **한 줄 요약**: 기본은 per-test 트랜잭션 롤백(흔적 0). GRANT·DDL처럼 트랜잭션으로 못 감싸는 건 truncate-and-seed, 병렬은 DB-per-worker로 격리합니다.

---

## Q4. (핵심) "실제 운영 중인 DB"에서는 어떻게 테스트하나요?

가장 궁금한 부분이자 가장 위험한 부분입니다. 먼저 **철칙**:

> 🚫 **운영(production) DB에 *쓰기 테스트*를 직접 하지 않는다.** 테스트가 실데이터를 변조·삭제하면 복구 불가 사고다.

그럼 "운영에서 테스트한다"는 건 뭘 뜻할까요? 안전한 방법들이 있습니다.

**① staging = 운영 복제본(마스킹)** — 가장 흔함

운영 데이터를 복제하되 **PII를 마스킹**([[secret-encryption]]·[[pipa-consent]])해서 staging에 올립니다. "운영과 같은 데이터 모양·규모"에서 테스트하되 실고객 정보는 없습니다.

```
prod 스냅샷 → 마스킹(email→test_*@x, phone→NULL, 이름 익명화) → staging
  → 진짜 규모·분포로 인덱스·파티션·성능 테스트 가능, 유출 위험 0
```

**② read-replica로 읽기 전용 검증**

운영 read-replica에 붙어 **읽기만** 하는 검증(데이터 정합성, 격리 쿼리 점검)은 안전합니다. 쓰기는 절대 안 함.

```sql
-- read-replica에서: "운영에 org_pk 없는 행이 있나?"(불변식 #3 위반 탐지) 같은 점검
SELECT COUNT(*) FROM org_entitlement WHERE org_pk IS NULL;  -- 0이어야 함
```

**③ synthetic 테넌트 (전용 org_pk)** — 운영에서 가장 실전적

운영 DB에 **합성 테스트 org**(예: `org_pk = 9999999`, slug `__smoke__`)를 만들어, *그 org 안에서만* 스모크 테스트를 돌립니다. org_pk 격리([[multitenancy-rls]])가 *테스트 격리에도* 그대로 쓰입니다 — 합성 org의 데이터는 실테넌트와 완전히 분리됩니다.

```
운영 배포 후 스모크: 합성 org(9999999)로 가입→구독→게시 e2e 1회전
  → 실제 운영 경로를 검증하되 실고객 org는 안 건드림
  → 끝나면 합성 org 데이터 cleanup (또는 영구 합성 org 재사용)
```

**④ shadow / dark launch** — 새 경로를 실트래픽으로, 결과는 안 씀

새 쿼리/로직을 실제 요청에 *병렬로* 태워 결과를 기존과 비교(diff)만 하고 응답엔 안 씁니다. "운영 트래픽으로 검증하되 영향 0".

**⑤ canary** — 소수 트래픽에만 새 버전, 지표 보고 확대/롤백

**⑥ 마이그레이션 리허설** — 운영 클론에서 먼저

스키마 변경은 **운영 복제본에서 먼저 리허설**해 잠금·시간을 측정합니다([[online-ddl-migration]]). 대형 테이블 `ALTER`가 운영을 멈추지 않는지 prod-clone에서 확인 후 적용.

정리하면 "운영에서 테스트"는 *운영 DB를 직접 쓰는 게 아니라*, **복제본·replica·합성 테넌트·shadow**로 "운영의 현실"을 안전하게 재현하는 것입니다.

> 💡 **한 줄 요약**: 운영 DB엔 절대 쓰기-테스트 금지. 대신 마스킹한 staging 복제본, read-replica(읽기 검증), **합성 테넌트 org**(org_pk 격리를 테스트 격리로 재사용), shadow/canary, 마이그레이션 리허설로 "운영의 현실"을 안전하게 검증합니다.

---

## Q5. 무엇을 어느 레이어에서 테스트하나요? (테스트 피라미드 매핑)

같은 보장을 여러 곳에서 중복 테스트하면 느리고, 빈 곳이 생기면 샙니다. platform_db 보장별 권장 레이어:

| 검증 대상 | 레이어 | 어디서 | 근거 문서 |
|---|---|---|---|
| 순수 권한 규칙(역할→action) | 단위(mock) | 메모리 | [[casl-ability]] |
| org_pk 격리·BOLA | 통합(real DB) | Testcontainers | [[bola-object-authz]]·[[multitenancy-rls]] |
| CHECK·FK·UNIQUE | 통합(real DB) | Testcontainers | [[enum-vs-varchar-check]]·[[fk-strategy]] |
| append-only GRANT | 통합(real DB, 계정별 커넥션) | Testcontainers | [[least-privilege-grant]] |
| 결제↔권한 트랜잭션 원자성 | 통합(real DB) | Testcontainers | [[consistency-model]] |
| 사용자 여정(가치흐름·예외) | e2e(블랙박스) | 서비스 기동 + 합성 org | [[e2e-journeys]] |
| 거부 매트릭스(403/404/402) | 통합/e2e | Testcontainers/e2e | [[bdd-scenarios]] D3 |
| 운영 배포 정상성 | 스모크 | 운영 + 합성 org(Q4③) | — |

원칙: **"이 보장을 *DB가* 강제하나?" → 예면 real DB, 아니면 mock.** 그리고 *경계 거동*(403/404/402 전환)은 e2e로 한 번 더.

> 💡 **한 줄 요약**: 순수 로직은 단위(mock), DB가 강제하는 것(격리·제약·GRANT·트랜잭션)은 통합(real MySQL), 사용자 여정은 e2e. 보장의 *강제 주체*가 레이어를 정합니다.

---

## Q6. 운영에서 테스트할 때, 그 테스트가 운영 지표·데이터를 오염시키지 않게 하려면요?

합성 테넌트(Q4③)나 스모크가 운영을 더럽히면 안 됩니다. 안전장치:

```
□ 합성 데이터 태깅      합성 org_pk·이메일에 식별 마커(__smoke__, is_synthetic)
                       → 분석·과금·audit 집계에서 제외 가능
□ 관측 분리            테스트 트래픽을 SLI/SLO 집계에서 제외([[observability-slo]])
                       → 스모크 실패가 운영 error budget을 까먹지 않게
□ cleanup 보장         스모크 후 합성 org 데이터 정리(또는 영구 합성 org 고정 재사용)
□ kill switch          합성 테스트 경로를 feature flag로 즉시 끌 수 있게
□ 권한 최소화          스모크 계정도 합성 org에만 멤버십 → 실 org 접근 구조적 불가
```

핵심은 **격리가 곧 안전**이라는 점입니다. 운영에서 테스트가 안전한 이유는 우리 설계가 이미 org_pk로 테넌트를 격리하기 때문입니다 — 합성 org는 그 격리 위에 올라탄 또 하나의 "테넌트"일 뿐, 실고객과 데이터·지표가 섞이지 않습니다.

> 💡 **한 줄 요약**: 합성 데이터에 마커를 달아 분석·과금·SLO 집계에서 빼고, cleanup·kill switch·최소권한으로 운영 오염을 막습니다. org_pk 격리가 그대로 테스트 안전장치가 됩니다.

---

## 용어 정리

| 용어 | 한 줄 정의 |
|---|---|
| **Testcontainers** | 테스트 시작 시 일회용 DB(컨테이너)를 띄우고 끝나면 폐기하는 도구 |
| **fixture / seed** | 테스트 전 깔아두는 기준 데이터 |
| **teardown** | 테스트 후 정리(롤백·truncate·컨테이너 폐기) |
| **트랜잭션 롤백 격리** | 테스트를 트랜잭션으로 감싸고 commit 대신 rollback → 흔적 0 |
| **DB-per-worker** | 병렬 테스트 워커마다 별도 DB로 격리 |
| **staging** | 운영과 같은 구성의 사전 환경(보통 마스킹한 운영 복제) |
| **read-replica** | 운영의 읽기 전용 복제본. 읽기 검증에 안전 |
| **synthetic 테넌트** | 운영 DB 안의 합성 테스트 org(전용 org_pk, 실데이터와 격리) |
| **data masking** | PII를 가짜값으로 치환해 운영 데이터를 안전하게 재사용 |
| **shadow / dark launch** | 실트래픽을 새 경로에 병렬로 태우되 결과는 안 쓰고 비교만 |
| **canary** | 소수 트래픽에만 새 버전 배포 후 지표로 확대/롤백 |
| **smoke test** | 배포 직후 핵심 경로 1회전 점검(운영 + 합성 org) |
| **마이그레이션 리허설** | 운영 클론에서 스키마 변경을 먼저 돌려 잠금·시간 측정 |

---

## 테스트 방법 (메타 — CI 파이프라인 구성)

이 문서 자체의 "테스트 방법"은 *파이프라인을 어떻게 짜는가*입니다.

```yaml
# CI 단계 (개념)
1) 단위(mock)        : vitest, DB 없음 — 빠른 피드백 (수초)
2) 통합(real MySQL)  : Testcontainers MySQL 8 기동 → drizzle-kit migrate
                       → org_pk 격리·CHECK·GRANT·트랜잭션·BOLA 테스트
3) e2e(블랙박스)     : 서비스 기동 + 합성 org → 가치흐름·거부 매트릭스
4) (배포 후) 스모크  : 운영 + 합성 org로 핵심 경로 1회전, 실패 시 자동 롤백
```

```typescript
// 통합 테스트 부트스트랩 (Testcontainers + 마이그레이션 + 계정별 커넥션)
beforeAll(async () => {
  container = await new MySqlContainer("mysql:8.0").start();
  await migrate(adminDb, { migrationsFolder: "./drizzle" });   // 스키마 = 운영과 동일
  await applyGrants(adminDb);                                   // §M 계정·GRANT 재현
  platformRw   = connectAs("platform_rw");                      // append-only 거부 테스트용
  ledgerAppend = connectAs("ledger_append");                    // INSERT-only 검증용
});
```

**무엇을 점검하나 (전략 체크리스트)**

- [ ] DB가 강제하는 보장은 **mock이 아니라 real MySQL**로 테스트하는가 (격리·CHECK·GRANT·트랜잭션)
- [ ] 테스트 DB 스키마를 **실제 마이그레이션**으로 적용하는가 (drift 0 — [[orm-testing-drizzle]])
- [ ] 테스트 격리(트랜잭션 롤백 / truncate / DB-per-worker)가 걸려 있는가
- [ ] **운영 DB에 쓰기-테스트가 없는가** — 합성 org·staging·replica만 쓰는가
- [ ] 합성 데이터가 분석·과금·SLO 집계에서 **제외**되는가 ([[observability-slo]])
- [ ] 마이그레이션을 운영 클론에서 **리허설**하는가 ([[online-ddl-migration]])
- [ ] GRANT·append-only는 **계정별 커넥션**으로 실제 거부를 단언하는가 ([[least-privilege-grant]])

---

## 마치며

테스트 전략은 두 질문으로 압축됩니다.

1. **"이 보장을 누가 강제하나?"** — DB 엔진이면 진짜 MySQL(Testcontainers), 순수 로직이면 mock. platform_db는 격리·제약·GRANT·트랜잭션을 *DB가* 강제하므로 real DB 통합 테스트가 1급 시민입니다.
2. **"운영 DB를 직접 쓰나?"** — 절대 아니요. 운영의 현실은 마스킹 staging·read-replica·**합성 테넌트(org_pk 격리 재사용)**·shadow/canary·마이그레이션 리허설로 안전하게 재현합니다.

가장 멋진 점은, 우리가 인가를 위해 만든 **org_pk 격리가 테스트 안전장치로도 그대로 쓰인다**는 것입니다 — 합성 org는 실고객과 격리된 또 하나의 테넌트일 뿐입니다. 새 기능을 만들 때 "이 테스트를 어디서 돌리지?"가 막막하면, 이 문서의 피라미드 매핑(Q5)과 운영 안전 철칙(Q4)을 떠올리세요.

---

## 연결된 개념

- [[orm-testing-drizzle|ORM(Drizzle) 테스트 전략]] — 진짜 DB 위에서 ORM 레이어를 어떻게 테스트하나
- [[bdd-scenarios|BDD 시나리오]] — 통합(화이트박스) 테스트가 단언하는 행위
- [[e2e-journeys|E2E 사용자 여정]] — 블랙박스 e2e(합성 org로 운영 스모크에도 재사용)
- [[multitenancy-rls|Pool 모델 멀티테넌시]] — org_pk 격리 = 테스트 격리(합성 테넌트)의 토대
- [[least-privilege-grant|최소권한 GRANT]] — 계정별 커넥션으로 append-only 거부를 real DB 테스트
- [[consistency-model|일관성 모델]] — 트랜잭션 원자성을 real DB로 검증
- [[online-ddl-migration|온라인 DDL]] — 마이그레이션을 운영 클론에서 리허설
- [[observability-slo|관측성·SLO]] — 테스트 트래픽을 운영 지표에서 분리
> 소스 문서
- [[schema-reference]] — §A.2 접근 계층(Drizzle·패키지), §M 계정·GRANT, §G org_pk 격리, §D.8 파티션
- [[operability]] — 운영 가능성(배포·스모크·관측)
- [[requirements]] — TEN(격리)·SEC(GRANT)·NFR(성능) 테스트 대상
