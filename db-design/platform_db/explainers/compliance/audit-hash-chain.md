---
difficulty: 고급
tags:
  - platform-db
  - explainer
  - p2
  - security
  - audit
  - hash
  - worm
  - compliance
  - integrity
aliases:
  - 해시 체인
  - 감사 무결성
  - prev_hash
  - row_hash
  - WORM
  - S3 Object Lock
---

# audit_log 해시 체인 무결성 설명

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[architecture]] §4 감사 무결성, §4 break-glass, [[schema-reference]] §D.8, §H.2

보안 감사 로그는 "기록이 있다"는 것만큼이나 "기록이 변조되지 않았다"는 것도 중요합니다. 이 문서는 audit_log 무결성을 어떻게 보장하는지, 해시 체인이 무엇인지 설명합니다.

---

## Q1. 왜 그냥 로그만 남기면 안 되나요? 누군가 로그를 바꿀 수 있나요?

일반적인 DB 테이블은 수정이 가능합니다. 관리자(DBA)가 직접 SQL을 실행하면 어떤 row든 바꿀 수 있죠.

감사 로그(audit_log)에서 이것이 문제가 되는 이유를 생각해봅시다.

```
시나리오: 운영자가 실수로 고객 개인정보를 무단 조회했다.
  → audit_log에 기록됨: action='USER_DATA_VIEW', actor_pk=42 (운영자), result='ALLOW'

이 운영자가 DB 접근 권한이 있다면:
  UPDATE audit_log SET actor_pk = 99  -- 다른 사람으로 바꿈
  WHERE pk = 12345;

  또는:
  DELETE FROM audit_log WHERE pk = 12345;  -- 아예 삭제
```

만약 이런 수정이 가능하다면 감사 로그는 법적 증거로 쓸 수 없습니다. ISMS-P, SOC2 같은 보안 인증 심사에서도 "변조 가능한 로그는 로그가 아니다"라고 봅니다.

그래서 두 가지 방어선을 만들었습니다.

```
1차 방어: DB 계정 권한 제거
   → app 계정에서 UPDATE/DELETE 권한 자체를 없앰 (현재 적용 중)

2차 방어: 해시 체인 (hash chain)
   → 설령 root 계정으로 수정해도, 해시가 깨져서 탐지 가능 (P1 미구현)
```

> 💡 **한 줄 요약**: 로그를 DB에 저장하면 DB 접근 권한이 있는 사람이 수정할 수 있어, 권한 제거와 해시 체인이라는 두 방어선이 필요합니다.

---

## Q2. 해시(hash)가 뭔가요? SHA-256이 뭔지 모르는 사람을 위해 설명해주세요.

해시 함수는 **어떤 데이터를 넣어도 항상 같은 길이의 "디지털 지문"을 뽑아내는 함수**입니다.

음식 레시피로 비유하면 이렇습니다. "닭볶음탕 레시피"를 해시 함수에 넣으면 항상 같은 16글자 코드가 나와요. 레시피에서 소금을 1g이라도 바꾸면 완전히 다른 코드가 나옵니다.

```
SHA-256 예시:

입력: "actor_pk=42,action=LOGIN,result=ALLOW"
출력: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
       (항상 64자리 16진수)

입력: "actor_pk=43,action=LOGIN,result=ALLOW"  ← actor_pk만 바뀜
출력: "a948904f2f0f479b8f9564077f40a36d06a3060e8b2fd9b9b45e42af47f6b5c2"
       (완전히 다른 값!)
```

SHA-256의 특성:

| 특성 | 설명 |
|---|---|
| **결정론적** | 같은 입력 → 항상 같은 출력 |
| **단방향** | 출력 → 입력으로 역추적 불가 |
| **눈사태 효과** | 입력 1비트만 바뀌어도 출력이 완전히 달라짐 |
| **충돌 저항** | 다른 두 입력이 같은 출력을 낼 확률 = 천문학적으로 낮음 |

이 특성 덕분에 "해시 값이 같으면 데이터가 변조되지 않았다"고 신뢰할 수 있습니다.

> 💡 **한 줄 요약**: 해시는 데이터의 디지털 지문으로, 데이터가 조금이라도 변경되면 지문이 완전히 달라져서 변조 여부를 즉시 감지할 수 있습니다.

---

## Q3. "해시 체인"이 뭔가요? 블록체인이랑 비슷한 건가요?

맞습니다. 블록체인과 같은 원리입니다. 블록체인은 각 블록이 이전 블록의 해시를 포함하는 구조이고, audit_log 해시 체인도 각 row가 이전 row의 해시를 포함하는 구조입니다.

```
해시 체인 개념:

row #1: row_hash = SHA256(내 데이터) = "abc123..."

row #2: prev_hash = "abc123..."  ← row#1의 row_hash를 가져옴
        row_hash  = SHA256(내 데이터 + prev_hash) = "def456..."

row #3: prev_hash = "def456..."  ← row#2의 row_hash를 가져옴
        row_hash  = SHA256(내 데이터 + prev_hash) = "ghi789..."
```

ASCII 다이어그램으로 표현하면:

```
┌─────────────────────────┐
│ row #1                  │
│ pk=1                    │
│ action=LOGIN            │
│ prev_hash = NULL        │
│ row_hash  = "abc123..." │──┐
└─────────────────────────┘  │
                             │
┌─────────────────────────┐  │
│ row #2                  │  │
│ pk=2                    │  │
│ action=VIEW_USER        │  │
│ prev_hash = "abc123..." │◀─┘  ← row#1의 해시를 가짐
│ row_hash  = "def456..." │──┐
└─────────────────────────┘  │
                             │
┌─────────────────────────┐  │
│ row #3                  │  │
│ pk=3                    │  │
│ action=LOGOUT           │  │
│ prev_hash = "def456..." │◀─┘  ← row#2의 해시를 가짐
│ row_hash  = "ghi789..." │
└─────────────────────────┘
```

이 구조에서 누군가 row #2를 수정하면:

```
공격자가 row #2의 action을 'VIEW_USER' → 'APPROVED_PAYOUT'으로 바꿈

수정 후 row #2의 row_hash를 다시 계산하면 완전히 달라짐
  (데이터가 바뀌었으니 해시도 달라짐)

row #3은 여전히 수정 전 row #2의 해시를 prev_hash로 가지고 있음
  → row #3의 prev_hash ≠ 수정된 row #2의 row_hash
  → 불일치 감지! 변조 탐지!
```

> 💡 **한 줄 요약**: 해시 체인은 각 row가 이전 row의 해시를 포함해 "체인"을 형성하고, 중간 row 하나를 수정하면 이후 row들과 해시가 맞지 않아 변조가 드러나는 구조입니다.

---

## Q4. prev_hash와 row_hash가 audit_log에 왜 필요한가요?

두 컬럼의 역할이 다릅니다.

**`row_hash`** — 이 row 자체가 변조됐는지 감지

```sql
-- row_hash는 이 row의 모든 핵심 필드를 합쳐서 해시를 계산
row_hash = SHA256(
  CONCAT(pk, org_pk, actor_pk, action, resource_type, result, created_at)
)
```

`row_hash`만 있어도 "이 row가 생성 이후 변경됐는지"는 알 수 있습니다. 하지만 row를 **삭제**하면 감지할 수 없어요.

**`prev_hash`** — row 삭제나 순서 조작을 감지

```sql
-- prev_hash는 바로 직전 row의 row_hash
prev_hash = 이전 row의 row_hash 값
```

`prev_hash`가 있으면 중간 row가 삭제돼도 감지할 수 있습니다.

```
row #5의 prev_hash = "xyz789..." (row #4의 row_hash)

누군가 row #4를 DELETE하면:
  row #3이 이제 row #5의 바로 앞 row
  row #3의 row_hash ≠ row #5의 prev_hash
  → 갭 감지! 삭제 탐지!
```

두 컬럼을 함께 쓰면 **수정 탐지 + 삭제 탐지** 모두 커버합니다.

현재 DDL에는 이 컬럼들이 아직 없습니다. [[pipa-consent|PIPA 동의]] 관련 `user_consent_event`에는 설계가 들어가 있고, `audit_log`는 구현 시 추가 예정입니다.

> 💡 **한 줄 요약**: `row_hash`는 row 수정을, `prev_hash`는 row 삭제를 감지합니다. 두 컬럼이 함께 있어야 완전한 변조 탐지가 가능합니다.

---

## Q5. root 사용자가 로그를 지워도 해시 체인이 그걸 탐지할 수 있나요?

탐지는 할 수 있습니다. 하지만 중요한 전제가 있습니다.

**해시 체인은 "사후 탐지" 도구입니다.** root가 로그를 지우는 걸 막지는 못합니다. 다만 지운 뒤 검증을 실행하면 체인이 깨진 걸 발견할 수 있어요.

```
공격 시나리오:
  1. root가 audit_log의 특정 row들을 DELETE
  2. 남은 row들에서 prev_hash 불일치 발생
  3. 월 1회 배치 검증에서 탐지됨
  4. 보안 알림 발송

방어 한계:
  root가 정말 치밀하다면, 삭제 후 남은 row들의 prev_hash/row_hash까지
  모두 재계산해서 다시 UPDATE할 수 있습니다.
  → 이걸 막으려면 외부 WORM 스토리지가 필요합니다 (다음 Q6 참조)
```

이 문제 때문에 architecture.md §2.2에서 "audit `BEFORE UPDATE` 트리거로 불변성" 제안을 거부했습니다. root가 `DROP TRIGGER`를 실행하면 트리거 자체가 사라지거든요. 해시 체인 + 외부 WORM이 정답입니다.

```
방어 레이어 (강도 순):

Level 1: app 계정 UPDATE/DELETE 권한 없음 → 일반 공격 차단
Level 2: 해시 체인 검증 → 특권 사용자의 수정/삭제 탐지
Level 3: 외부 WORM (S3 Object Lock) → root 위협까지 차단

현재: Level 1만 적용 (Level 2: P1 미구현, Level 3: P2 계획)
```

> 💡 **한 줄 요약**: 해시 체인은 root의 변조를 사후에 탐지하지만, root가 해시 값까지 조작하는 걸 막으려면 DB 외부의 WORM 스토리지가 추가로 필요합니다.

---

## Q6. "WORM 스토리지"가 뭔가요? S3 Object Lock은 어떻게 동작하나요?

**WORM**은 "Write Once, Read Many"의 약자입니다. **한 번 쓰면 수정도 삭제도 불가능한 스토리지**를 말합니다.

일상적인 비유로는 공증 문서와 비슷합니다. 한 번 공증이 찍히면 내용을 바꿀 수 없잖아요. WORM 스토리지도 그렇습니다.

**AWS S3 Object Lock**이 대표적인 WORM 구현입니다.

```
S3 Object Lock 작동 방식:

1. audit_log를 S3에 주기적으로 내보내기 (예: 매월 말)
   s3://company-audit-archive/2026/01/audit_log_202601.parquet

2. 이 파일에 Object Lock 적용
   aws s3api put-object-retention \
     --bucket company-audit-archive \
     --key 2026/01/audit_log_202601.parquet \
     --retention '{"Mode":"COMPLIANCE","RetainUntilDate":"2031-01-31"}'

3. 이후 5년간 이 파일은:
   - 삭제 불가 (AWS가 막음)
   - 수정 불가 (덮어쓰기도 안 됨)
   - root 계정도 불가 (COMPLIANCE 모드)
   - 심지어 AWS Support에 요청해도 불가
```

**S3 Object Lock 모드 두 가지:**

| 모드 | 설명 | 용도 |
|---|---|---|
| **GOVERNANCE** | 특수 권한(`s3:BypassGovernanceRetention`)으로 삭제 가능 | 개발/테스트 |
| **COMPLIANCE** | 보존 기간 중 누구도 삭제 불가. 계정 삭제도 안 됨 | 규정 준수 (ISMS-P/GDPR) |

platform_db에서 [[partitioning|파티셔닝]] 기반 WORM 도입 계획:

```
현재: DB 계정 권한 제거만 (Level 1)
P1:   해시 체인 컬럼 추가, 월 1회 배치 검증 (Level 2)
P2/T4: ISMS-P/GDPR 계약 체결 시 S3 Object Lock 적재 (Level 3)
       audit_log → 월별 파티션 → S3 COMPLIANCE 모드 아카이빙
```

> 💡 **한 줄 요약**: WORM 스토리지는 "한 번 쓰면 절대 못 지우는 금고"로, S3 Object Lock의 COMPLIANCE 모드를 쓰면 AWS root 계정도 보존 기간 중 삭제할 수 없습니다.

---

## Q7. 현재 이 기능이 구현되지 않았다고 하는데, 지금은 어떻게 무결성을 보장하나요?

솔직히 말씀드리겠습니다. **현재는 완전한 무결성 보장이 안 됩니다.** 다만 가장 현실적인 공격 경로를 차단하는 1차 방어는 적용되어 있습니다.

**현재 구현 상태 (P0):**

```sql
-- audit_log 테이블에 prev_hash, row_hash 컬럼 없음
-- user_consent_event에는 컬럼은 있으나 NULL로 삽입 중

CREATE TABLE audit_log (
  -- ... 다른 컬럼들 ...
  break_glass BOOLEAN NOT NULL DEFAULT FALSE,
  created_at  DATETIME NOT NULL DEFAULT (NOW()),
  -- prev_hash CHAR(64),  ← 아직 없음 (미구현 예정)
  -- row_hash  CHAR(64),  ← 아직 없음 (미구현 예정)
  PRIMARY KEY (pk, created_at)
);
```

**현재 방어 수단: app 계정 최소 권한**

```sql
-- DB 계정별 권한 (schema-reference.md §M)
GRANT SELECT, INSERT ON platform_db.audit_log TO 'platform_rw'@'%';
-- UPDATE, DELETE 권한 없음 → app이 실수로도 수정 불가

-- audit 전용 계정은 INSERT만
GRANT INSERT ON platform_db.audit_log TO 'audit_append'@'%';
```

이 방어의 한계는 DB 자체에 접근할 수 있는 DBA나 root 계정은 막을 수 없다는 점입니다.

**구현 예정 (P1):**

```
1. audit_log에 prev_hash, row_hash 컬럼 추가
2. INSERT 시 앱 레이어에서 SHA-256 계산 후 저장
   (PostgreSQL이었다면 pgcrypto로 DB 내에서 처리 가능)
3. 월 1회 배치 검증:
   audit_log 전체를 순서대로 읽어 해시 체인 재계산
   불일치 row 발견 시 → 보안 알림 발송
```

**검증 배치 의사코드:**

```typescript
async function verifyAuditHashChain() {
  let prevHash: string | null = null;

  const rows = await db.select()
    .from(auditLog)
    .orderBy(asc(auditLog.pk));  // pk 순서로 순회

  for (const row of rows) {
    // 1. prev_hash 체인 검증
    if (row.prevHash !== prevHash) {
      await alertSecurityTeam({
        message: `audit_log 해시 체인 불일치: pk=${row.pk}`,
        expected: prevHash,
        actual: row.prevHash,
      });
      return;
    }

    // 2. row_hash 자체 검증
    const expectedHash = computeSHA256(row);
    if (row.rowHash !== expectedHash) {
      await alertSecurityTeam({
        message: `audit_log row 변조 감지: pk=${row.pk}`,
      });
      return;
    }

    prevHash = row.rowHash;
  }

  console.log('✅ audit_log 해시 체인 검증 완료 — 이상 없음');
}
```

> 💡 **한 줄 요약**: 현재는 app 계정에서 UPDATE/DELETE를 막는 1차 방어만 작동 중이고, 해시 체인(2차)과 WORM(3차) 방어는 각각 후속·T4 트리거 시점에 구현될 예정입니다.

---

## 부록: break_glass 컬럼은 왜 있나요?

[[break-glass|break-glass]] — `audit_log.break_glass BOOLEAN NOT NULL DEFAULT FALSE` 컬럼이 보이시죠?

이건 [[architecture]] §4 Break-glass 정책과 연관됩니다. 긴급 운영 상황(서비스 장애, 법적 요청)에서 운영자가 평소에는 권한이 없는 작업을 승인을 받고 실행할 때, 그 행위를 명시적으로 표시하는 플래그입니다.

```sql
-- break_glass=TRUE인 감사 로그만 빠르게 조회
SELECT * FROM audit_log
WHERE break_glass = TRUE
ORDER BY created_at DESC;

-- 전용 인덱스가 있어 빠름:
-- INDEX idx_audit_break_glass (break_glass, created_at)
```

SOC2나 ISMS-P 심사 시 "비상 접근이 얼마나 있었고, 어떤 행위를 했나"를 이 컬럼으로 보고합니다. break_glass 행위는 절대 로그 없이 조용히(silent) 처리할 수 없습니다.

---

## 마치며

보안 감사 로그의 무결성은 "로그가 있다"에서 "로그가 변조되지 않았음을 증명할 수 있다"로 발전해야 합니다. platform_db의 방어 계획을 요약하면 다음과 같습니다.

```
방어 레이어    구현 상태    방어 대상
─────────────────────────────────────────────────────────
Level 1        ✅ 현재      app 코드의 실수/의도적 수정
(권한 제거)                 일반 DB 계정의 변조

Level 2        P1 예정     DBA/특권 계정의 조용한 변조
(해시 체인)    미구현    중간 row 삭제

Level 3        P2/T4 조건  DB root의 해시까지 조작
(WORM)         트리거 시   물리적 스토리지 공격
```

지금 당장 완벽하지 않더라도, 각 레이어의 역할과 한계를 이해하고 있으면 보안 개선 계획에 기여할 수 있습니다.

관련 운영 절차는 [[architecture]] §4 감사 무결성 운영 섹션을 참고하세요.

---

## 연결된 개념

- [[partitioning|DB 파티셔닝]] — 파티션 단위 아카이빙과 WORM 스토리지 연계
- [[pipa-consent|PIPA 동의]] — user_consent_event에도 동일하게 적용되는 해시 체인
- [[break-glass|Break-glass 긴급 접근]] — break_glass=true 이벤트를 해시 체인으로 탐지
> 소스 문서
- [[architecture]] — §2.2 보안 D11 (해시 체이닝→WORM), §4 감사 무결성 운영
- [[schema-reference]] — D.8 audit_log DDL (prev_hash/row_hash — P1 미구현 명시)
