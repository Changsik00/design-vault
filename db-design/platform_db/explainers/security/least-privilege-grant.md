---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p0
  - security
  - database
  - least-privilege
  - append-only
  - compliance
aliases:
  - 최소권한
  - least privilege
  - GRANT 강제
  - append-only GRANT
  - WORM 계정
---

# DB 계정 최소권한 & append-only를 GRANT로 강제 설명 — "UPDATE 금지"를 코드 규약이 아니라 DB 권한으로

> **대상**: DB 지식이 많지 않은 개발자 (공부용)
> **연관 문서**: [[architecture|architecture.md §2.2 · §4]] · [[schema-reference|schema-reference.md §M · §N.1]]

`platform_db`에는 절대 고치면 안 되는 기록이 있습니다. 결제 원장(`payment_ledger`), 감사 로그(`audit_log`), 법적 동의 기록(`user_consent_event`). 그런데 "고치지 마세요"라고 주석에 적어두는 것과, **DB가 권한 자체를 안 줘서 고칠 수 없게 만드는 것**은 전혀 다른 차원의 안전장치입니다. 이 문서는 우리가 왜 후자를 택했고, "최소권한"과 "append-only GRANT 강제"가 정확히 무엇을 막는지 설명합니다.

---

## Q1. "최소권한 원칙(least privilege)"이 뭔가요?

**최소권한 원칙(principle of least privilege)**은 *"각 주체(계정·사람·프로세스)에게 자기 일을 하는 데 꼭 필요한 권한만 주고, 그 이상은 주지 않는다"*는 보안 원칙입니다.

비유를 들면 이렇습니다. 호텔에서 청소 직원에게는 "객실 마스터키"만 주고, 금고 키나 회계실 키는 주지 않습니다. 청소에 금고 키가 필요 없으니까요. 만약 청소 직원 카드가 분실·복제되더라도, 그 카드로는 금고를 못 엽니다. **권한을 안 줬으니 피해 범위가 객실로 한정**됩니다.

DB도 똑같습니다. 흔한 실수는 모든 앱이 `root`나 전권 계정 하나로 DB에 붙는 것입니다.

```
❌ 안티패턴: 앱 전체가 root 계정 하나 공유
   academy-api ─┐
   agent-api  ─┼─→ root (모든 DB · 모든 권한 · DROP까지)
   billing    ─┘
   → 한 곳이 SQL injection 뚫리면 전 DB가 노출·삭제 가능
```

```
✅ 최소권한: 서비스마다 자기 DB 전용 계정 + 필요한 권한만
   academy-api ─→ academy_rw  (academy_db만, SELECT/INSERT/UPDATE)
   billing     ─→ platform_rw (platform_db만, DELETE 없음)
   리포팅 잡    ─→ platform_ro (platform_db, SELECT만)
```

핵심은 "혹시 한 계정이 탈취되어도, 그 계정이 할 수 있는 일이 작으면 피해도 작다"는 것입니다. 보안에서는 이걸 **폭발 반경(blast radius) 축소**라고 부릅니다.

> 💡 **한 줄 요약**: 최소권한은 "각 계정에 꼭 필요한 권한만 주기"입니다. 탈취되어도 피해가 그 계정의 권한 범위로 한정됩니다.

---

## Q2. 우리 설계에서 계정은 어떻게 나뉘어 있나요? (§M)

`schema-reference.md §M`에 계정 목록이 정의돼 있습니다. 규칙은 단순합니다: **각 서비스는 자기 DB에만 접근하는 전용 계정을 쓰고, cross-DB(여러 DB를 넘나드는) 계정은 만들지 않는다.**

| 계정 | 접근 범위 | 권한 |
|---|---|---|
| `platform_rw` | `platform_db` (append-only 4종 제외) | SELECT, INSERT, UPDATE — **DELETE 없음** |
| `platform_ro` | `platform_db` | SELECT only (리포팅·분석) |
| `academy_rw` | `academy_db` | SELECT, INSERT, UPDATE |
| `academy_ro` | `academy_db` | SELECT only |
| `audit_append` | `platform_db.audit_log` | **INSERT only** |
| `ledger_append` | `platform_db.payment_ledger`·`billing_event` | **INSERT only** |
| `consent_append` | `platform_db.user_consent_event` | **INSERT only** |
| `migrator` | 전체 | DDL 전용 (migration 실행 시에만, **상시 접속 금지**) |

여기서 짚어야 할 설계 결정 세 가지가 있습니다.

**(1) cross-DB 계정 없음.** 예를 들어 academy-api는 `academy_rw`(쓰기)와 `platform_ro`(플랫폼 데이터 읽기)만 씁니다. academy-api 계정으로는 `market_db`에 손도 못 댑니다. 서비스 경계가 곧 권한 경계입니다.

**(2) `migrator`는 DDL 전용 + 상시 접속 금지.** 테이블 구조를 바꾸는(`ALTER`, `CREATE`, `DROP`) 강력한 권한은 `migrator` 한 계정에만 있고, 이 계정은 **마이그레이션을 실행하는 그 순간에만 CI/CD 파이프라인 안에서** 접속합니다. 앱 런타임에는 절대 이 계정으로 붙지 않습니다. 일상 트래픽이 DDL 권한을 들고 다니지 않게 하는 겁니다. `DROP`·`TRUNCATE`도 `migrator` + 파이프라인 안에서만 가능합니다.

**(3) DELETE 권한은 어떤 계정에도 없습니다.** `platform_rw`조차 DELETE를 못 합니다. `platform_db`는 데이터를 물리적으로 지우지 않고 **논리 삭제(soft-delete)**만 하기 때문입니다(`status='DELETED'` 또는 `deleted_at` 세팅 → 자세히는 [[delete-patterns]]). "DELETE를 쓰지 말자"는 규약을 넘어서, **DELETE 권한 자체를 안 줘서** 실수로도 DELETE 쿼리가 통하지 않게 했습니다.

추가로 계정 비밀번호는 AWS Secrets Manager가 관리하고 90일마다 자동 로테이션됩니다.

> 💡 **한 줄 요약**: 서비스마다 자기 DB 전용 계정, cross-DB 계정 없음, `migrator`만 DDL(상시 접속 금지), DELETE는 전 계정에서 0 — 논리삭제만 합니다.

---

## Q3. "append-only를 GRANT로 강제한다"는 게 정확히 무슨 뜻인가요?

이게 이 문서의 핵심이자, 최근에 확정된 중요한 설계 결정입니다.

먼저 **append-only(추가 전용)**가 뭔지부터. append-only 테이블은 *"행을 새로 추가(INSERT)하는 것만 허용하고, 기존 행의 수정(UPDATE)이나 삭제(DELETE)는 허용하지 않는"* 테이블입니다. 같은 개념을 스토리지 업계에서는 **WORM(Write Once, Read Many)** — "한 번 쓰면 읽기만, 못 고침" — 이라고 부릅니다.

`platform_db`에서 append-only로 묶인 테이블은 4종입니다:

| 테이블 | 무엇 | 왜 못 고치게 해야 하나 |
|---|---|---|
| `audit_log` | 누가·언제·무엇을 했는지 감사 기록 | 감사 기록이 사후 수정되면 감사의 의미가 없음 (ISMS-P/SOC2) |
| `payment_ledger` | 결제·환불·차지백 금융 원장 | 금융 원장 위변조 = 회계 부정 |
| `billing_event` | 구독 상태 변화 로그 | 구독 분쟁 시 증거 |
| `user_consent_event` | PIPA 법적 동의/철회 이력 | 전자서명법상 서면 서명과 동일 효력 — 위변조 시 법적 문제 |

문제는, **"이 테이블은 INSERT만"이라고 주석에 적어두는 것만으로는 아무것도 막지 못한다**는 점입니다. 앱 코드 어딘가에 버그가 있어서 `UPDATE payment_ledger SET amount_minor = 0 WHERE ...`가 실행되면, 주석은 그걸 멈추지 못합니다. 주석은 사람에게 하는 부탁이지 컴퓨터에 대한 강제가 아니니까요.

그래서 우리는 **권한 구조 자체로** 막습니다. 두 가지를 동시에 합니다:

**(1) 이 4종에는 전용 INSERT-only 계정으로만 write.** `audit_log`는 `audit_append` 계정으로만, `payment_ledger`·`billing_event`는 `ledger_append`로만, `user_consent_event`는 `consent_append`로만 INSERT합니다. 이 계정들은 **INSERT 권한 외에는 아무것도 없습니다.** UPDATE도 DELETE도 GRANT되지 않았습니다.

**(2) `platform_rw`에서 이 4종에 대한 UPDATE를 테이블 단위로 미부여.** `platform_rw`는 `platform_db`의 다른 테이블에는 UPDATE를 할 수 있지만, **append-only 4종 테이블에 대해서만은 UPDATE GRANT를 받지 못합니다.** MySQL의 GRANT는 테이블 단위로 줄 수 있어서, "DB 전체에 UPDATE를 주되 이 4개 테이블만 빼고"가 가능합니다.

```sql
-- platform_rw 에게 DB의 (append-only 4종을 제외한) 테이블에만 UPDATE 부여
-- → audit_log·payment_ledger·billing_event·user_consent_event 에는 UPDATE를 주지 않음

-- INSERT 전용 계정 예시
GRANT INSERT ON platform_db.payment_ledger TO 'ledger_append'@'%';
GRANT INSERT ON platform_db.billing_event  TO 'ledger_append'@'%';
-- ledger_append 에게 부여된 권한은 INSERT 뿐. UPDATE·DELETE 없음.

-- 결과: 앱이 버그로 이런 쿼리를 던져도...
UPDATE payment_ledger SET amount_minor = 0 WHERE pk = 42;
-- → ERROR 1142 (42000): UPDATE command denied to user 'ledger_append'@'...'
--   권한이 없어서 거부됨. 데이터는 안전.
```

즉 **"UPDATE 금지"가 코드 규약이 아니라 DB 권한이 됩니다.** 앱이 아무리 버그가 있어도, 누가 악의로 UPDATE를 시도해도, **권한이 없으니 DB가 거부**합니다.

`architecture.md §4`는 이 결정을 이렇게 못 박았습니다:

> **append-only 강제**: `audit_log`·`user_consent_event`·`payment_ledger`·`billing_event`는 INSERT만 — 전용 INSERT-only 계정(`audit_append`·`consent_append`·`ledger_append`)으로만 write하고 `platform_rw`는 해당 4종에 UPDATE GRANT 미보유. **주석이 아니라 GRANT가 강제.**

그리고 `schema-reference.md §N.1`(DB가 구조적으로 강제하는 것)은 이 4종 append-only를 ✅ 보장 항목으로 올려두었습니다.

> 💡 **한 줄 요약**: append-only 4종은 INSERT-only 전용 계정으로만 쓰고 `platform_rw`는 이 4종에 UPDATE 권한이 아예 없습니다. "수정 금지"가 주석이 아니라 DB 권한으로 강제됩니다.

---

## Q4. 비유로 다시 — 왜 "권한을 안 주는 것"이 "고치지 말라고 하는 것"보다 강한가요?

은행 출납창구를 떠올려 보세요.

```
출납창구 직원의 권한:
  ✅ 입금 기록 추가 (새 거래를 장부에 적기)        ← INSERT
  ❌ 과거 거래 기록 수정 (이미 적힌 금액 바꾸기)    ← UPDATE (권한 없음)
  ❌ 과거 거래 기록 삭제                          ← DELETE (권한 없음)
```

출납 직원은 새 거래를 장부에 *추가*할 수만 있습니다. 어제 적힌 입금액을 *고치는* 권한은 애초에 없습니다. 그래서 직원이 실수를 하든 마음을 나쁘게 먹든, **과거 기록은 건드릴 수 없습니다.** 권한 자체가 없으니까요.

이게 `ledger_append` 계정과 정확히 같습니다. INSERT만 줬으니, 결제 원장에 새 거래(환불도 새 행으로 추가)를 *적을* 수만 있고, 과거 결제 행을 *고칠* 수는 없습니다.

여기서 "고치지 말라고 하는 것"과의 차이가 분명해집니다:

| 방식 | 막는 힘 | 실수가 통과하나? |
|---|---|---|
| ❌ 주석/코드 규약 ("UPDATE 쓰지 마세요") | 사람의 선의에 의존 | 통과함 (버그 한 줄이면 끝) |
| ❌ 앱 레이어 검증 ("UPDATE 쿼리 막는 코드") | 그 코드가 정확히 동작할 때만 | 우회 경로 있으면 통과 |
| ✅ GRANT 미부여 (DB가 권한 거부) | DB 엔진이 무조건 강제 | **통과 못 함** (권한 거부) |

DB가 강제하는 것의 강점은 *"앱이 무엇을 하든 상관없다"*는 점입니다. 앱에 버그가 있어도, 새 개발자가 규약을 몰라도, 심지어 SQL injection으로 임의 쿼리가 주입돼도 — `ledger_append` 계정으로는 UPDATE가 안 됩니다.

> 💡 **한 줄 요약**: "고치지 말라"는 부탁은 사람이 어기면 끝이지만, "고칠 권한이 없다"는 DB 엔진이 무조건 막습니다. 출납 직원이 과거 기록을 못 고치는 것과 같습니다.

---

## Q5. 그냥 트리거(BEFORE UPDATE)로 막으면 안 되나요? 왜 GRANT인가요?

좋은 질문입니다. MySQL에는 트리거라는 기능이 있어서, "이 테이블에 UPDATE가 들어오면 에러를 던져라"를 선언할 수도 있습니다.

```sql
-- 이렇게 막을 수도 있어 보입니다...
CREATE TRIGGER block_ledger_update BEFORE UPDATE ON payment_ledger
FOR EACH ROW SIGNAL SQLSTATE '45000'
  SET MESSAGE_TEXT = 'payment_ledger is append-only';
```

`platform_db`는 이 방식을 **의도적으로 거부**했습니다. `architecture.md §2.4 비목표`에 명시돼 있습니다:

> | audit `BEFORE UPDATE` 트리거 | 해시 체이닝→WORM — **트리거는 root가 DROP 가능** |

이유는 `schema-reference.md §N.3`과 `§H.2`에도 반복됩니다. **트리거는 root(또는 DDL 권한자)가 `DROP TRIGGER` 한 줄로 없앨 수 있습니다.** 위변조를 막으려는 장치가, 위변조를 하려는 사람이 가장 먼저 지워버릴 수 있는 장치라면 안전망이 아닙니다.

GRANT 방식은 이 약점이 덜합니다. 권한 부여 자체를 바꾸려면 GRANT/REVOKE를 할 수 있는 관리 권한이 필요하고, 우리는 그런 DDL/권한 작업을 `migrator` + CI/CD 파이프라인 안으로만 묶었습니다(§M). 게다가 **회귀를 GRANT 점검 CI로 상시 감시**합니다(다음 단락). 즉 트리거 대비:

```
트리거:  DROP TRIGGER 한 줄로 무력화 → 채택 안 함
GRANT:   권한 변경은 migrator/파이프라인 경유 + GRANT 점검 CI가 회귀 탐지 → 채택
```

물론 GRANT도 절대적이지 않습니다. root는 GRANT도 바꿀 수 있습니다. 그래서 *진짜 root 위협*까지 막으려면 DB 외부의 **WORM 스토리지**(S3 Object Lock)가 추가로 필요하고, 이건 P2(T4 트리거) 과제입니다([[audit-hash-chain]]). 정리하면 방어는 계층적입니다:

```
Level 1: GRANT 미부여  → 앱 버그·일반 계정 탈취·injection을 막음 (현재 ✅)
Level 2: 해시 체이닝    → root의 사후 변조를 탐지 (P1, [[audit-hash-chain]])
Level 3: 외부 WORM     → root의 물리적 변조까지 차단 (P2/T4)
```

이 문서가 다루는 GRANT는 Level 1 — 일상에서 가장 많이 발생하는 위협(앱 버그, 일반 계정 탈취, injection)을 막는 1차 방어선입니다.

> 💡 **한 줄 요약**: 트리거는 root가 `DROP TRIGGER`로 지울 수 있어 채택 안 했습니다. GRANT는 권한 변경을 파이프라인으로 묶고 CI로 회귀를 감시할 수 있어 더 견고합니다. root 위협은 위층(해시 체인·WORM)이 담당합니다.

---

## Q6. "GRANT 점검 CI"는 뭘 검사하나요? 회귀는 어떻게 막나요?

GRANT로 막아도 한 가지 위험이 남습니다: **나중에 누군가 실수로 권한을 다시 줘버리는 것.** 예컨대 신규 append-only 테이블을 추가하면서 깜빡하고 `platform_rw`에 UPDATE를 GRANT하거나, 권한 정리 스크립트가 잘못 돌아 UPDATE가 재유입될 수 있습니다. `schema-reference.md §N.2`는 이걸 **"append-only GRANT 회귀"** 위험으로 명시합니다.

이걸 막는 게 **GRANT 점검 CI(least-privilege regression test)**입니다. 핵심 아이디어: *DB에 실제 부여된 권한을 조회해서, append-only 4종에 위험한 권한이 없는지 CI가 매번 단언한다.*

권한 조회는 MySQL의 표준 메타데이터 뷰인 **`information_schema.table_privileges`**(또는 `schema_privileges`)로 합니다. 이 뷰에는 "어느 계정(GRANTEE)이 어느 테이블에 어떤 권한(PRIVILEGE_TYPE)을 가졌는지"가 행으로 들어 있습니다.

```sql
-- platform_rw 가 append-only 4종에 UPDATE를 가졌는지 조회
-- 결과가 한 행이라도 나오면 회귀 = CI 실패여야 함
SELECT grantee, table_schema, table_name, privilege_type
FROM information_schema.table_privileges
WHERE grantee LIKE "'platform_rw'@%"
  AND privilege_type = 'UPDATE'
  AND table_name IN ('audit_log','payment_ledger','billing_event','user_consent_event');
-- 기대 결과: 0 행 (UPDATE 권한이 없어야 정상)
```

이 검사가 막는 위협의 본질은 **권한의 분리(separation of duties)**입니다. "기록을 추가하는 권한(append 계정)"과 "기록을 수정하는 권한"이 같은 계정에 모이면 안 된다는 원칙이고, CI가 그 분리가 무너지지 않았는지 매 배포마다 확인합니다.

> 💡 **한 줄 요약**: GRANT 점검 CI는 `information_schema.table_privileges`로 실제 권한을 조회해, append-only 4종에 UPDATE 권한이나 DELETE 권한이 새어들어왔는지를 매번 단언합니다. 회귀를 자동으로 잡습니다.

---

## 용어 정리

| 용어 | 한 줄 정의 |
|---|---|
| 최소권한(least privilege) | 각 계정에 자기 일에 꼭 필요한 권한만 주는 보안 원칙. 탈취 시 피해 범위를 줄인다. |
| GRANT / REVOKE | DB 권한을 *부여*(GRANT) / *회수*(REVOKE)하는 SQL 명령. |
| append-only | 행 추가(INSERT)만 허용하고 수정(UPDATE)·삭제(DELETE)는 막는 테이블 정책. |
| WORM (Write Once, Read Many) | "한 번 쓰면 못 고침" — append-only의 스토리지 용어. |
| INSERT-only 계정 | INSERT 권한만 가진 전용 계정. 우리 설계의 `audit_append`·`ledger_append`·`consent_append`. |
| 테이블 단위 GRANT | "DB 전체에 UPDATE를 주되 이 테이블만 제외" 식으로 테이블별로 권한을 다르게 주는 것. |
| `information_schema.table_privileges` | 어떤 계정이 어떤 테이블에 어떤 권한을 가졌는지 담은 MySQL 표준 메타데이터 뷰. |
| least-privilege regression test | 권한이 의도치 않게 늘어났는지(회귀) CI에서 자동 검사하는 테스트. |
| separation of duties(직무 분리) | "기록 추가"와 "기록 수정" 같은 상충 권한을 한 계정에 몰지 않는 원칙. |
| blast radius(폭발 반경) | 한 계정이 탈취됐을 때 피해가 미치는 범위. 최소권한이 이걸 줄인다. |
| DDL | `CREATE`/`ALTER`/`DROP` 등 스키마 구조를 바꾸는 명령. 우리 설계에선 `migrator`만 보유. |
| 논리 삭제(soft-delete) | 행을 물리적으로 지우지 않고 `status`/`deleted_at`으로 "지워진 상태"만 표시([[delete-patterns]]). |

---

## 테스트 방법

이 설계의 핵심은 "권한이 의도대로 *부여되지 않았는지*"를 검증하는 것입니다. 권한은 시간이 지나며 슬그머니 늘어나기 쉬우므로(권한 회귀), 아래 검사를 CI에 넣어 매 배포마다 단언합니다.

### 1. GRANT 점검 CI — `platform_rw`가 append-only 4종에 UPDATE 권한이 없는지

가장 중요한 회귀 테스트입니다. 실제 DB의 권한 메타데이터를 조회합니다.

```sql
-- 검사 A: platform_rw 는 append-only 4종에 UPDATE 권한이 없어야 한다
SELECT grantee, table_name, privilege_type
FROM information_schema.table_privileges
WHERE grantee LIKE "'platform_rw'@%"
  AND privilege_type = 'UPDATE'
  AND table_name IN ('audit_log','payment_ledger','billing_event','user_consent_event');
-- ✅ PASS 조건: 결과가 0행
-- ❌ FAIL: 한 행이라도 나오면 회귀 → 배포 차단
```

```sql
-- 검사 B: append 계정은 INSERT 외의 권한을 갖지 않아야 한다
SELECT grantee, table_name, privilege_type
FROM information_schema.table_privileges
WHERE grantee IN ("'ledger_append'@'%'","'audit_append'@'%'","'consent_append'@'%'")
  AND privilege_type <> 'INSERT';
-- ✅ PASS 조건: 결과가 0행 (INSERT 외 권한이 없어야 함)
```

```sql
-- 검사 C: DELETE 권한은 전 계정에서 0이어야 한다 (논리삭제만, §M)
SELECT grantee, table_schema, table_name
FROM information_schema.table_privileges
WHERE privilege_type = 'DELETE';
-- ✅ PASS 조건: 결과가 0행
-- (스키마 권한 레벨도 함께 보려면 information_schema.schema_privileges 도 동일 검사)
```

CI에서의 의사코드:

```typescript
// least-privilege regression test (CI에서 마이그레이션 직후 실행)
const updateLeaks = await db.execute(sql`
  SELECT table_name FROM information_schema.table_privileges
  WHERE grantee LIKE "'platform_rw'@%"
    AND privilege_type = 'UPDATE'
    AND table_name IN ('audit_log','payment_ledger','billing_event','user_consent_event')
`);
expect(updateLeaks.rows).toHaveLength(0);   // append-only 4종에 UPDATE 누수 0

const deleteAnywhere = await db.execute(sql`
  SELECT grantee FROM information_schema.table_privileges
  WHERE privilege_type = 'DELETE'
`);
expect(deleteAnywhere.rows).toHaveLength(0); // 전 계정 DELETE 0
```

### 2. 실제 거부 검증 — append 계정으로 UPDATE를 시도하면 권한 거부

권한 조회뿐 아니라, *실제로 막히는지*도 확인합니다. `ledger_append` 계정으로 접속해 UPDATE를 던져 권한 오류가 나는지 단언합니다.

```typescript
// ledger_append 커넥션으로 UPDATE 시도 → 권한 거부(ER_TABLEACCESS_DENIED_ERROR)여야 함
await expect(
  ledgerAppendConn.execute(
    sql`UPDATE payment_ledger SET amount_minor = 0 WHERE pk = 1`
  )
).rejects.toThrow(/command denied|ER_TABLEACCESS_DENIED/i);

// INSERT 는 정상 동작해야 함 (append 은 허용)
await expect(
  ledgerAppendConn.execute(
    sql`INSERT INTO payment_ledger (org_pk, type, amount_minor, currency, idempotency_key)
        VALUES (1, 'CHARGE', 10000, 'KRW', 'test-key-1')`
  )
).resolves.toBeDefined();
```

같은 검증을 `audit_append`(→ `audit_log` UPDATE 거부), `consent_append`(→ `user_consent_event` UPDATE/DELETE 거부)에도 합니다.

### 무엇을 단언하나 — 체크리스트

- [ ] `platform_rw` 가 `audit_log`·`payment_ledger`·`billing_event`·`user_consent_event` 에 **UPDATE 권한이 없다** (검사 A = 0행)
- [ ] `audit_append`·`ledger_append`·`consent_append` 는 **INSERT 외 권한이 없다** (검사 B = 0행)
- [ ] **DELETE 권한이 전 계정에서 0** 이다 (검사 C = 0행) — 논리삭제만(§M)
- [ ] append 계정으로 **UPDATE 시도 시 실제로 권한 거부**된다 (거부 검증)
- [ ] append 계정으로 **INSERT 는 정상 동작**한다 (append 기능은 살아 있다)
- [ ] cross-DB 누수 없음: `academy_rw` 가 `platform_db` 에 write 권한이 없다
- [ ] DDL 권한(`CREATE`/`ALTER`/`DROP`)은 `migrator` 외 계정에 없다

이 검사들은 마이그레이션 직후 CI 단계에서 돌리는 게 가장 효과적입니다. 신규 append-only 테이블·계정이 추가될 때 권한이 새어드는 회귀를 그 자리에서 잡기 때문입니다(§N.2).

---

## 마치며

이 문서의 메시지는 하나로 압축됩니다: **"하지 마세요"라고 적는 것과 "할 수 없게 만드는 것"은 다르다.**

`platform_db`의 결제 원장·감사 로그·동의 기록은 위변조되면 회계 부정이나 법적 분쟁으로 이어지는 데이터입니다. 그래서 "수정 금지"를 주석이나 코드 규약에 맡기지 않고, **GRANT를 안 주는 방식으로 DB가 직접 강제**합니다. 앱이 버그가 있어도, injection이 들어와도, 새 개발자가 규약을 몰라도 — INSERT-only 계정과 UPDATE 미부여가 권한 차원에서 막습니다.

새 append-only 테이블을 추가하게 된다면 반드시 기억하세요:

1. 그 테이블 전용 INSERT-only 계정으로만 write하도록 GRANT를 설계할 것
2. `platform_rw`에 그 테이블 UPDATE를 **주지 말 것** (테이블 단위 GRANT)
3. GRANT 점검 CI의 테이블 목록에 새 테이블을 추가해, 회귀가 자동으로 잡히게 할 것

그리고 이 GRANT는 1차 방어선이라는 것도 잊지 마세요. root까지 막으려면 해시 체이닝([[audit-hash-chain]])과 외부 WORM이 위층에서 받쳐줍니다.

---

## 연결된 개념

- [[audit-hash-chain|감사 해시 체이닝 & WORM]] — GRANT(1차) 위에서 root 변조를 탐지·차단하는 2~3차 방어
- [[delete-patterns|삭제 패턴]] — DELETE 권한이 전 계정에서 0인 이유, 논리삭제 3종(status/deleted_at/append-only)
- [[multitenancy-rls|멀티테넌시 & RLS]] — 계정 권한과 함께 데이터 격리를 이루는 또 다른 축(org_pk)
- [[bola-object-authz|BOLA 객체 인가]] — 계정 권한 아래에서 행 단위 접근을 통제하는 앱 레이어 방어
- [[break-glass|Break-glass]] — 평소 차단된 권한을 비상 시에만 감사와 함께 여는 패턴
> 소스 문서
- [[architecture]] — §2.2 보안 규율(감사 불변성: app 계정 UPDATE/DELETE 제거, 트리거 거부), §2.4 비목표(트리거 미채택), §4 append-only 강제(주석 아니라 GRANT)
- [[schema-reference]] — §M DB 계정 최소권한, §N.1 구조적 보장(append-only 4종 ✅), §N.2 GRANT 회귀 갭, §N.3 트리거 미채택 이유, §D.8 audit_log, §D.16 billing_event, §D.17 payment_ledger, §I user_consent_event, §H.2 감사 불변성
- [[requirements]] — append-only·최소권한 관련 요건 추적
