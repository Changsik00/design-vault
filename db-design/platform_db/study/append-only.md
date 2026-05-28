# append-only 패턴 — 왜 UPDATE를 안 하나

## 배경

"사용자가 마케팅 수신을 거부했다"는 상태 변화를 어떻게 저장할 것인가.

```sql
-- 방법 A: mutable boolean (UPDATE)
CREATE TABLE user_settings (
  user_pk            BIGINT PRIMARY KEY,
  marketing_agreed   BOOLEAN NOT NULL DEFAULT FALSE,
  updated_at         TIMESTAMP
);

UPDATE user_settings SET marketing_agreed = FALSE WHERE user_pk = 10;
```

간단해 보인다. 그런데 PIPA 분쟁이 생겼을 때:

```
"이 사용자가 언제 동의했고, 언제 철회했나요?"
"저 컬럼은 현재 값만 있어서 이력을 알 수 없습니다."
```

현재 `FALSE`라는 사실만 있고, 언제 동의했고 언제 철회했는지 증명할 방법이 없다.

---

## append-only — INSERT만, UPDATE/DELETE 없음

```sql
-- 방법 B: append-only 이벤트
CREATE TABLE user_consent_event (
  pk           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_pk      BIGINT UNSIGNED NOT NULL,
  consent_type VARCHAR(50) NOT NULL,
  action       ENUM('GRANTED', 'REVOKED') NOT NULL,  -- 변경도 새 row
  created_at   TIMESTAMP NOT NULL DEFAULT NOW()
  -- 절대 UPDATE, DELETE 없음
);

-- 최초 동의
INSERT INTO user_consent_event VALUES (?, 10, 'platform.marketing_email', 'GRANTED', '2026-01-15 09:00:00');

-- 한 달 후 철회
INSERT INTO user_consent_event VALUES (?, 10, 'platform.marketing_email', 'REVOKED', '2026-02-20 14:30:00');

-- 두 달 후 재동의
INSERT INTO user_consent_event VALUES (?, 10, 'platform.marketing_email', 'GRANTED', '2026-04-01 11:00:00');
```

이력이 전부 보존된다. "언제 동의했고 언제 철회했나"에 명확히 답할 수 있다.

---

## 현재 상태 조회

```sql
-- 현재 동의 상태: 가장 최근 action을 읽음
SELECT action
FROM user_consent_event
WHERE user_pk = 10
  AND consent_type = 'platform.marketing_email'
ORDER BY created_at DESC
LIMIT 1;

-- 결과:
-- 'GRANTED' → 현재 동의 상태
-- 'REVOKED' → 현재 미동의 상태
-- (없음)   → 아직 동의한 적 없음
```

---

## 우리의 append-only 테이블 세 곳

### 1. user_consent_event — PIPA 법적 증거

```
이유: 동의/철회 반복 이력이 법적 분쟁의 핵심 증거
보존: 5년 (PIPA 기준)
DB 계정 권한: INSERT만. UPDATE/DELETE 거부.
```

### 2. audit_log — 감사 불변성

```
이유: "누가 언제 무엇에 ALLOW/DENY됐나"는 절대 바뀌면 안 됨
     → 보안 감사, ISMS-P 요건
보존: 파티션 단위 아카이빙
DB 계정 권한: INSERT만 (audit_append 계정)
미래: prev_hash/row_hash 해시 체이닝 → 위변조 불가 (P1)
```

### 3. payment_ledger — 금융 원장

```
이유: 회계 원장은 지워지면 안 됨. 환불도 삭제가 아닌 새 row.
     REFUND row: amount_minor = 음수 값으로 기록

CHARGE: (+990,000)  ← 결제
REFUND: (-990,000)  ← 환불 (기존 CHARGE row 수정 안 함)
```

---

## DB 레벨 강제

append-only는 관례만으로는 지키기 어렵다. DB 계정 권한으로 강제:

```sql
-- audit_append 계정에는 INSERT만
GRANT INSERT ON platform_db.audit_log TO 'audit_append'@'%';
-- UPDATE, DELETE 권한 없음 → 앱 코드 실수해도 DB가 차단
```

---

## append-only vs soft-delete 비교

| 항목 | soft-delete (deleted_at) | append-only |
|---|---|---|
| 목적 | row를 숨기되 보존 | 이력 전체를 보존 |
| 현재 상태 | NULL 체크 | 최신 row 읽기 |
| 이력 추적 | 삭제 시점만 | 모든 변화 |
| 법적 증거력 | 낮음 | **높음** |
| 스토리지 | 적음 | 많음 |

---

## 트레이드오프

| 항목 | mutable 상태 | append-only (우리) |
|---|---|---|
| 현재 상태 조회 | 단순 (컬럼 읽기) | 약간 복잡 (ORDER BY LIMIT) |
| 이력 조회 | 불가 | 전체 이력 |
| 법적 증거력 | 없음 | **있음** |
| 스토리지 | 적음 | 지속 증가 |
| 위변조 가능성 | 높음 (UPDATE 가능) | 낮음 (INSERT만) |

---

## 관련 파일

| 파일 | 내용 |
|---|---|
| `core/schema-reference.md §D.8` | audit_log append-only DDL |
| `core/schema-reference.md §I.2` | user_consent_event append-only DDL |
| `core/schema-reference.md §D.17` | payment_ledger append-only |
| `core/schema-reference.md §M` | DB 계정 최소 권한 (INSERT만 부여) |
| `study/soft-delete.md` | 삭제 3종 패턴 전체 비교 |
