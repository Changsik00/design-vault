# 삭제 3종 패턴 — status / deleted_at / append-only

## 배경

"이 데이터를 삭제해야 한다"는 요구가 왔을 때 실제 `DELETE FROM table WHERE pk = ?`를 쓰면 어떻게 되는가.

```sql
DELETE FROM membership WHERE user_pk = 10 AND org_pk = 1;
-- row가 사라진다
-- 이 강사가 언제 이 학원 소속이었는지 알 수 없다
-- 감사 로그에 "actor_pk = 10의 행위"가 남아있는데 user_pk = 10 join 불가
```

DB에서 hard DELETE는 **기록이 사라진다**. 대부분의 비즈니스 시스템에서 이것은 문제다.

---

## 삭제 3종 패턴

### 패턴 1 — status ENUM 변경

```sql
-- 테이블에 status 컬럼
CREATE TABLE membership (
  user_pk BIGINT NOT NULL,
  org_pk  BIGINT NOT NULL,
  status  ENUM('ACTIVE', 'SUSPENDED') NOT NULL DEFAULT 'ACTIVE',
  ...
);

-- "삭제" = status 변경
UPDATE membership SET status = 'SUSPENDED' WHERE user_pk = 10 AND org_pk = 1;
```

**언제**: 비즈니스 상태가 바뀌는 것. 복원 가능한 상태 전환.

**우리 사용처**:
- `membership.status` (ACTIVE ↔ SUSPENDED)
- `organization.status` (ACTIVE / SUSPENDED / CLOSED)
- `org_entitlement.status` (ACTIVE / GRACE / SUSPENDED / EXPIRED)

**특징**: row는 살아있음. status 컬럼으로 현재 상태만 읽음.

---

### 패턴 2 — deleted_at TIMESTAMP

```sql
CREATE TABLE identity_user (
  pk         BIGINT NOT NULL,
  email      VARCHAR(255),
  deleted_at TIMESTAMP,  -- NULL이면 살아있음, 값 있으면 삭제됨
  ...
);

-- "삭제" = deleted_at 기록
UPDATE identity_user SET deleted_at = NOW() WHERE pk = 10;

-- 살아있는 사용자만 조회
SELECT * FROM identity_user WHERE deleted_at IS NULL;
```

**언제**: "삭제됐지만 나중에 복원하거나 참조할 수 있어야 하는 것". 삭제 시점이 중요.

**우리 사용처**:
- `identity_user.deleted_at`: 탈퇴 처리. 30일 후 hard anonymize(`email=NULL`)
- `organization.deleted_at`: 학원 폐쇄 기록

**특징**: row는 살아있음. `WHERE deleted_at IS NULL` 조건을 모든 조회에 추가해야 함.

---

### 패턴 3 — append-only (새 row INSERT)

```sql
CREATE TABLE user_consent_event (
  pk           BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_pk      BIGINT NOT NULL,
  consent_type VARCHAR(50) NOT NULL,
  action       ENUM('GRANTED', 'REVOKED') NOT NULL,  -- 철회도 새 row
  created_at   TIMESTAMP NOT NULL DEFAULT NOW()
  -- UPDATE / DELETE 없음. 항상 INSERT만.
);

-- 동의
INSERT INTO user_consent_event VALUES (?, ?, 'platform.marketing_email', 'GRANTED', NOW());

-- 철회 → 기존 row를 바꾸는 게 아니라 새 row 추가
INSERT INTO user_consent_event VALUES (?, ?, 'platform.marketing_email', 'REVOKED', NOW());

-- 현재 상태: 가장 최근 action을 읽음
SELECT action FROM user_consent_event
WHERE user_pk = ? AND consent_type = 'platform.marketing_email'
ORDER BY created_at DESC LIMIT 1;
```

**언제**: 이력 전체가 법적 증거이거나 감사 대상인 것. "어떻게 바뀌었나"가 중요.

**우리 사용처**:
- `user_consent_event`: PIPA 동의 이력 (GRANTED/REVOKED 전체)
- `audit_log`: 모든 권한 결정 (ALLOW/DENY/ERROR)
- `payment_ledger`: 결제 원장 (CHARGE/REFUND는 별도 row)

**특징**: row가 절대 바뀌지 않음. 현재 상태는 최신 row를 읽어서 계산.

---

## 세 가지 비교

| 항목 | status | deleted_at | append-only |
|---|---|---|---|
| row 수 | 변하지 않음 | 변하지 않음 | 계속 증가 |
| 이력 보존 | 현재 상태만 | 삭제 시점만 | 전체 이력 |
| 복원 | 쉬움 (status 변경) | 쉬움 (NULL로) | 새 GRANTED row |
| 쿼리 복잡도 | 낮음 | 중간 (`IS NULL` 필요) | 중간 (`ORDER BY ... LIMIT 1`) |
| 법적 증거력 | 낮음 | 중간 | **높음** |
| 스토리지 | 적음 | 적음 | 많음 |

---

## 선택 기준

```
"이 상태가 다시 돌아올 수 있나?" → status ENUM
"삭제 시점이 중요하고 복원할 수 있어야 하나?" → deleted_at
"이력 전체가 법적으로 필요하나?" → append-only
```

---

## 관련 파일

| 파일 | 내용 |
|---|---|
| `core/schema-reference.md §D.1` | identity_user status + deleted_at |
| `core/schema-reference.md §D.4` | membership status |
| `core/schema-reference.md §I` | user_consent_event append-only 설계 |
| `study/append-only.md` | append-only 패턴 심화 |
