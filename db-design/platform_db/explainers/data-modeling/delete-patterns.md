---
difficulty: 초
tags:
  - platform-db
  - explainer
  - p1
  - schema
  - soft-delete
  - append-only
  - audit
  - design-decision
aliases:
  - 삭제 패턴
  - soft-delete
  - deleted_at
  - append-only
  - 삭제 3종 패턴
---

# 삭제·이력 보존 패턴 설명 (status / deleted_at / append-only)

> **대상**: DB 지식이 많지 않은 개발자  
> **연관 문서**: [[schema-reference]] §D.1·§D.4·§I, [[architecture]] §2.1, [[pipa-consent]], [[audit-hash-chain]]

`platform_db`에는 `DELETE FROM ...`로 row를 진짜 지우는 코드가 거의 없습니다. 대신 세 가지 "삭제처럼 보이는" 패턴을 씁니다. 왜 그런지, 언제 무엇을 쓰는지 설명합니다.

---

## Q1. 그냥 `DELETE`로 지우면 안 되나요?

지우면 **기록이 사라집니다.** 대부분의 비즈니스 시스템에서 이게 문제입니다.

```sql
DELETE FROM membership WHERE user_pk = 10 AND org_pk = 1;
-- row가 사라진다
-- 이 강사가 언제 이 학원 소속이었는지 알 수 없다
-- audit_log엔 "actor_pk = 10의 행위"가 남아있는데 user_pk = 10 join 불가
```

감사 추적, 분쟁 대응, 복원 요구가 들어오는 순간 hard DELETE는 답이 없습니다. 그래서 "삭제"를 세 가지 보존형 패턴으로 나눠 처리합니다.

> 💡 **한 줄 요약**: hard DELETE는 이력을 파괴합니다. 감사·복원·법적 증거가 필요한 시스템에선 "지우는 대신 표시하거나 쌓는" 방식을 씁니다.

---

## Q2. 삭제에 어떤 방식들이 있나요?

세 가지입니다.

**패턴 1 — `status` ENUM 변경**: 비즈니스 상태가 바뀌는 것. 복원 가능한 상태 전환.

```sql
-- "삭제" = status 변경
UPDATE membership SET status = 'SUSPENDED' WHERE user_pk = 10 AND org_pk = 1;
```

우리 사용처: `membership.status`(ACTIVE↔SUSPENDED), `organization.status`(ACTIVE/SUSPENDED/CLOSED), `org_entitlement.status`(ACTIVE/GRACE/SUSPENDED/EXPIRED).

**패턴 2 — `deleted_at` TIMESTAMPTZ**: "삭제됐지만 복원·참조할 수 있어야 하는 것". 삭제 *시점*이 중요.

```sql
UPDATE identity_user SET deleted_at = now() WHERE pk = 10;
SELECT * FROM identity_user WHERE deleted_at IS NULL;  -- 살아있는 것만
```

우리 사용처: `identity_user.deleted_at`(탈퇴 후 30일 뒤 hard anonymize), `organization.deleted_at`(학원 폐쇄).

**패턴 3 — append-only (새 row INSERT)**: 이력 전체가 법적 증거이거나 감사 대상인 것. "어떻게 바뀌었나"가 중요. → 자세한 건 아래 Q3.

세 패턴 비교:

| 항목 | status | deleted_at | append-only |
|---|---|---|---|
| row 수 | 변하지 않음 | 변하지 않음 | 계속 증가 |
| 이력 보존 | 현재 상태만 | 삭제 시점만 | 전체 이력 |
| 복원 | 쉬움 (status 변경) | 쉬움 (NULL로) | 새 GRANTED row |
| 쿼리 복잡도 | 낮음 | 중간 (`IS NULL`) | 중간 (`ORDER BY ... LIMIT 1`) |
| 법적 증거력 | 낮음 | 중간 | **높음** |
| 스토리지 | 적음 | 적음 | 많음 |

> 💡 **한 줄 요약**: status는 "상태가 돌아올 수 있는 것", deleted_at은 "숨기되 복원/시점이 중요한 것", append-only는 "이력 전체가 증거인 것"에 씁니다.

---

## Q3. append-only는 왜 UPDATE/DELETE를 아예 안 하나요?

mutable boolean으로 저장하면 **이력이 소실**되기 때문입니다.

```sql
-- 방법 A: mutable boolean (UPDATE)
UPDATE user_settings SET marketing_agreed = FALSE WHERE user_pk = 10;
```

이러면 PIPA 분쟁 시 "이 사용자가 *언제* 동의했고 *언제* 철회했나요?"에 답할 수 없습니다. 현재 `FALSE`라는 사실만 남죠.

append-only는 변경도 **새 row로 쌓습니다.**

```sql
-- 최초 동의
INSERT INTO user_consent_event VALUES (?, 10, 'platform.marketing_email', 'GRANTED', '2026-01-15 09:00:00');
-- 한 달 후 철회
INSERT INTO user_consent_event VALUES (?, 10, 'platform.marketing_email', 'REVOKED', '2026-02-20 14:30:00');
-- 두 달 후 재동의
INSERT INTO user_consent_event VALUES (?, 10, 'platform.marketing_email', 'GRANTED', '2026-04-01 11:00:00');

-- 현재 상태 = 가장 최근 action
SELECT action FROM user_consent_event
WHERE user_pk = 10 AND consent_type = 'platform.marketing_email'
ORDER BY created_at DESC LIMIT 1;
```

우리의 append-only 테이블 세 곳:

- **`user_consent_event`** — PIPA 동의/철회 이력. 5년 보존. 법적 증거. ([[pipa-consent]])
- **`audit_log`** — 모든 권한 결정(ALLOW/DENY/ERROR). 보안 감사·ISMS-P. 미래 해시 체이닝으로 위변조 차단. ([[audit-hash-chain]])
- **`payment_ledger`** — 금융 원장. 환불도 삭제가 아닌 새 row(CHARGE +990,000 / REFUND −990,000).

> 💡 **한 줄 요약**: append-only는 "현재 값"이 아니라 "모든 변화"를 기록합니다. 철회·환불도 기존 row를 고치지 않고 새 row로 남겨, 이력 자체가 증거가 됩니다.

---

## Q4. append-only를 코드 관례가 아니라 어떻게 강제하나요?

DB 계정 권한으로 강제합니다. 관례만으로는 누군가 실수로 UPDATE/DELETE를 칠 수 있으니까요.

```sql
-- audit_append 역할에는 INSERT만 부여
GRANT INSERT ON audit_log TO audit_append;
-- UPDATE, DELETE 권한 없음 → 앱 코드가 실수해도 DB가 차단
```

앱 DB 계정 전반에서 `DELETE` 권한을 제거하는 것도 같은 맥락입니다([[schema-reference]] §M).

> 💡 **한 줄 요약**: append-only는 "DB 계정에 INSERT만 부여"로 강제합니다. 코드 실수가 있어도 DB 레벨에서 막힙니다.

---

## Q5. 결국 뭘 언제 쓰면 되나요?

```
"이 상태가 다시 돌아올 수 있나?"            → status ENUM
"삭제 시점이 중요하고 복원할 수 있어야 하나?" → deleted_at
"이력 전체가 법적으로/감사상 필요하나?"       → append-only
```

> 💡 **한 줄 요약**: 복원 가능한 상태 전환은 status, 숨김+복원은 deleted_at, 증거가 필요한 이력은 append-only.

---

## 연결된 개념

- [[pipa-consent]] — user_consent_event append-only가 PIPA 동의 이력에 쓰이는 이유
- [[audit-hash-chain]] — audit_log append-only + 해시 체이닝 무결성
- [[idempotency-key]] — payment_ledger append-only 원장과 멱등 처리
> 소스 문서
- [[schema-reference]] — §D.1 identity_user(status+deleted_at), §D.4 membership(status), §I user_consent_event(append-only), §M DB 계정 최소 권한
- [[architecture]] — §2.1 불변식(soft-delete 3종), §1.5 정책·동의
