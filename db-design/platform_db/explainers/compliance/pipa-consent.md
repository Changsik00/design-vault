---
difficulty: 중급
tags:
  - platform-db
  - explainer
  - p1
  - compliance
  - pipa
  - consent
  - legal
  - korea
aliases:
  - PIPA
  - 개인정보 동의
  - user_consent_event
  - 동의 철회
  - 14세 미만
---

# PIPA 개인정보 동의 요건 설명

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[schema-reference]] §I.1, §I.2 · [[architecture]] §1.5, §4

동의 처리는 "그냥 체크박스 결과 저장하면 되는 거 아닌가?"라고 생각하기 쉽습니다. 하지만 한국 개인정보보호법(PIPA)은 생각보다 구체적인 요건을 요구합니다. 이 문서는 왜 `user_consent_event`가 단순한 boolean 컬럼이 아니라 이벤트 로그 테이블인지, 그 이유를 설명합니다.

---

## Q1. PIPA가 뭔가요? 개발자가 왜 알아야 하나요?

PIPA(Personal Information Protection Act)는 **개인정보보호법**입니다. 한국에서 사용자 데이터를 다루는 서비스라면 반드시 지켜야 합니다.

개발자와 직접 관련된 PIPA 조항들을 정리하면:

| 조항 | 요건 | 위반 시 |
|---|---|---|
| §15 수집·이용 동의 | 목적, 항목, 보유기간 명시 | 과태료 3,000만 원 이하 |
| §17 제3자 제공 | 제공 대상, 목적, 항목, 기간 별도 동의 | 과태료 3,000만 원 이하 |
| §22③ 14세 미만 | 법정대리인 동의 필수 | 형사처벌 가능 |
| §35 열람권 | 사용자 요청 시 동의 이력 제공 | 과태료 |
| §37 철회권 | 동의 철회 즉시 처리 | 과태료 |

"법무팀 일이지 개발자가 왜?"라고 생각할 수 있습니다. 하지만 이 요건들을 만족하는 DB 구조를 만드는 것은 개발자의 몫입니다. 나중에 요건을 충족하는 구조로 바꾸려면 마이그레이션 비용이 훨씬 큽니다.

> 💡 **한 줄 요약**: PIPA는 개인정보 수집·이용·제3자 제공에 대해 사용자 동의를 받고, 그 이력을 증명 가능하게 보관하도록 요구합니다.

---

## Q2. 동의를 그냥 `is_marketing_agreed BOOLEAN` 컬럼 하나로 저장하면 안 되나요?

안 됩니다. 이유가 세 가지입니다.

**첫째, 이력이 사라집니다.** 사용자가 동의 → 철회 → 재동의를 반복했다면 현재 상태만 저장하면 이 과정이 모두 지워집니다. PIPA 분쟁 시 "언제 동의했고 언제 철회했나"를 증명해야 하는데 불가능합니다.

```
boolean 방식의 문제:

2024-01-01: is_marketing_agreed = TRUE   (동의)
2024-03-15: is_marketing_agreed = FALSE  (철회)
2024-06-20: is_marketing_agreed = TRUE   (재동의)

현재 DB 상태: TRUE
과거 이력: 알 수 없음 ← PIPA 분쟁 시 문제
```

**둘째, 버전을 추적할 수 없습니다.** 약관이 2024년 버전과 2025년 버전이 있다면, 사용자가 어느 버전에 동의했는지 알아야 합니다. boolean에는 버전 정보가 없습니다.

**셋째, 동의 유형을 분리할 수 없습니다.** 마케팅 이메일 동의, SMS 동의, 제3자 제공 동의는 각각 별도로 관리해야 합니다. 컬럼을 계속 늘리는 방식(is_email_agreed, is_sms_agreed, is_third_party_agreed...)은 새 동의 유형이 생길 때마다 스키마를 바꿔야 합니다.

```sql
-- 잘못된 방식: 컬럼 기반
ALTER TABLE user ADD COLUMN is_email_agreed BOOLEAN;
ALTER TABLE user ADD COLUMN is_sms_agreed BOOLEAN;
ALTER TABLE user ADD COLUMN is_firebase_agreed BOOLEAN;
-- 새 동의 추가할 때마다 스키마 변경 필요

-- 올바른 방식: 이벤트 테이블
INSERT INTO user_consent_event (consent_type, action, version)
VALUES ('platform.marketing_email', 'GRANTED', '2025-01');
-- 새 동의 추가해도 스키마 변경 불필요
```

> 💡 **한 줄 요약**: boolean 컬럼은 현재 상태만 저장하므로 이력이 사라집니다. PIPA는 "언제 어떤 버전에 동의/철회했나"를 증명할 수 있어야 합니다.

---

## Q3. user_consent_event 테이블이 왜 append-only(UPDATE 금지)인가요?

`user_consent_event`의 각 행은 **법적 증거**입니다. 전자서명법 §3에 따르면 약관 체크박스 동의는 서면 서명과 동일한 법적 효력이 있고, 그 기록이 바로 이 테이블의 각 행입니다.

UPDATE나 DELETE가 가능하다면 이 증거를 조작할 수 있게 됩니다. "나는 그 약관에 동의한 적 없다"는 사용자 주장에 "2024-01-15에 동의하셨습니다"라고 반박하려면, 그 행이 변조되지 않았음을 보장해야 합니다.

```
append-only 원칙:

동의 시:   INSERT (action='GRANTED')   ← 추가만 가능
철회 시:   INSERT (action='REVOKED')   ← 추가만 가능 (기존 행 변경 ❌)
재동의:    INSERT (action='GRANTED')   ← 추가만 가능

타임라인:
  2024-01-15 14:23 | GRANTED | v2024-01
  2024-03-20 09:17 | REVOKED | v2024-01
  2024-06-01 11:45 | GRANTED | v2025-06  ← 약관 갱신 후 재동의
```

이 테이블은 DB 계정 권한 설계에서도 반영됩니다. 애플리케이션 계정(`platform_rw`)에는 INSERT 권한만 부여하고 UPDATE/DELETE 권한을 아예 없애는 것이 목표입니다. 설령 서버가 해킹당하더라도 과거 동의 이력을 조작할 수 없게 됩니다.

미래에는 `prev_hash`/`row_hash` 컬럼으로 **[[audit-hash-chain|해시 체이닝]]**도 도입됩니다. 각 행이 이전 행의 해시를 포함하게 만들어, 중간 행을 삭제하거나 수정하면 해시 불일치로 조작을 감지할 수 있습니다. (현재는 P1 미구현 상태입니다.)

> 💡 **한 줄 요약**: 동의 이벤트 행은 법적 증거이므로 변조되면 안 됩니다. append-only + DB 권한 제한 + 해시 체이닝으로 무결성을 보장합니다.

---

## Q4. 동의 "철회"는 어떻게 처리하나요? 기존 레코드를 false로 바꾸면 안 되나요?

바꾸면 안 됩니다. 철회도 **새로운 이벤트로 INSERT**합니다.

```sql
-- 마케팅 이메일 동의 철회
INSERT INTO user_consent_event (
  user_pk, consent_type, action, version, ip, user_agent, created_at
) VALUES (
  12345,
  'platform.marketing_email',
  'REVOKED',              -- ← 새 이벤트: 철회
  '2025-06',
  X'7F000001',
  'Mozilla/5.0...',
  NOW()
);
-- 기존 GRANTED 행은 그대로 남아있음
```

현재 동의 상태를 조회할 때는 가장 최신 이벤트를 읽습니다:

```sql
-- 현재 마케팅 이메일 동의 상태 조회
SELECT action
FROM user_consent_event
WHERE user_pk = 12345
  AND consent_type = 'platform.marketing_email'
ORDER BY created_at DESC
LIMIT 1;

-- 결과: 'REVOKED' → 현재 철회 상태
-- 결과: 'GRANTED' → 현재 동의 상태
-- 결과: 없음 → 아직 동의한 적 없음
```

철회가 발생하면 단순히 DB에 INSERT하는 것 외에 다음 처리도 함께 됩니다:

```
REVOKED 이벤트 INSERT
  → organization.perm_version 또는 user.perm_version bump
    (클라이언트 캐시 무효화)
  → 마케팅 발송 큐에서 해당 사용자 즉시 제거
  → 다음 민감 요청 시 DB 재검증으로 차단 보장
```

> 💡 **한 줄 요약**: 철회도 REVOKED 이벤트를 새로 INSERT합니다. 기존 행을 변경하거나 삭제하지 않습니다.

---

## Q5. 14세 미만 가입자 처리가 왜 특별한가요?

PIPA §22③은 **만 14세 미만 아동의 개인정보는 법정대리인(부모)의 동의를 받아야 한다**고 규정합니다. 이는 벌금이 아니라 형사처벌도 가능한 강행규정입니다.

단순히 생년월일 확인 후 가입을 막는 것이 아니라, 법정대리인 동의 흐름을 설계해야 합니다:

```
14세 미만 가입 흐름:

1. 가입 시 생년월일 입력
2. 14세 미만 감지
3. "보호자(법정대리인) 동의가 필요합니다" 안내
4. 보호자 연락처(이메일/SMS) 입력
5. 보호자에게 동의 요청 발송
6. 보호자가 동의 완료
7. user_consent_event에 두 행 INSERT:
   - (consent_type='platform.under14_guardian', action='GRANTED',
      meta_json={guardian_relation:'parent', verified_by:'sms_otp'})
   - 14세 미만 본인 기본 동의들
8. 가입 완료
```

`meta_json`에는 법정대리인 정보가 저장됩니다:

```json
{
  "guardian_relation": "parent",
  "guardian_contact": "010-****-5678",
  "verified_by": "sms_otp",
  "verification_timestamp": "2025-06-01T10:23:45Z"
}
```

> ⚠️ 현재 guardian 동의 로직은 구현 대상입니다. 법적 필수 사항이므로 서비스 오픈 전에 반드시 구현해야 합니다.

14세 미만 확인 강도(SMS 본인인증 vs PASS 인증)는 아직 열린 결정이지만, 최소한 동의 이력은 반드시 `user_consent_event`에 기록해야 합니다.

> 💡 **한 줄 요약**: 14세 미만 가입자는 법정대리인 동의가 법적 필수입니다. meta_json에 보호자 정보를 저장하고 `platform.under14_guardian` 이벤트를 기록합니다.

---

## Q6. 제3자 제공 동의(PIPA §17)가 뭔가요? 왜 meta_json에 4가지 정보가 필요한가요?

PIPA §17은 사용자 데이터를 제3자(다른 회사)에게 제공할 때 **별도 동의**를 받아야 하고, 동의 내용에 반드시 4가지를 포함해야 한다고 규정합니다:

| 필수 항목 | 의미 | 예시 |
|---|---|---|
| recipient (제공받는 자) | 누구에게 주나 | "Toss Payments 주식회사" |
| purpose (제공 목적) | 왜 주나 | "결제 처리 및 사기 방지" |
| items (제공 항목) | 무엇을 주나 | "이름, 이메일, 결제금액" |
| retention (보유기간) | 얼마나 보관하나 | "결제 완료 후 5년" |

이 4가지 정보를 `meta_json`에 저장합니다:

```sql
-- Toss 결제 시 제3자 제공 동의
INSERT INTO user_consent_event (
  user_pk,
  consent_type,
  action,
  version,
  meta_json,
  created_at
) VALUES (
  12345,
  'pg.toss_third_party',
  'GRANTED',
  '2025-06',
  JSON_OBJECT(
    'recipient', 'Toss Payments 주식회사',
    'purpose', '결제 처리, 사기 방지, 전자금융거래법 준수',
    'items', JSON_ARRAY('이름', '이메일', '결제금액', '결제수단'),
    'retention', '결제 완료 후 5년 (전자금융거래법 §22)'
  ),
  NOW()
);
```

Firebase 인증도 제3자 제공에 해당합니다. Firebase는 구글(미국)의 서비스이므로 **국외 이전**에 대한 동의도 필요합니다:

```
가입 시 필수 동의 3가지:

1. platform.terms_of_service      — 이용약관
2. platform.privacy_policy        — 개인정보 처리방침
3. platform.third_party_firebase  — Firebase(구글, 미국) 국외 이전
                                    meta_json: {
                                      recipient: 'Google LLC',
                                      purpose: '사용자 인증 서비스',
                                      items: ['이메일', 'UID', '로그인 이력'],
                                      retention: '계정 삭제 후 90일'
                                    }
```

> 💡 **한 줄 요약**: PIPA §17은 제3자 제공 시 recipient/purpose/items/retention 4가지를 명시한 별도 동의를 요구합니다. meta_json에 이 정보를 구조화해 저장합니다.

---

## Q7. 동의 철회가 [[gate-abc-flow|Gate 흐름]]의 JWT stale 문제와 어떻게 연결되나요?

JWT(JSON Web Token)는 서버가 발급한 인증 토큰입니다. 기본 유효 시간이 1시간이라면, 발급 후 1시간 동안은 DB를 조회하지 않고도 유효한 것으로 처리합니다. 여기서 문제가 생깁니다.

```
시나리오: 마케팅 동의 철회 후 이메일이 여전히 발송되는 문제

10:00 - 사용자가 JWT 발급 (유효: 10:00 ~ 11:00)
10:30 - 마케팅 이메일 동의 철회
         → user_consent_event에 REVOKED INSERT
         → perm_version bump
         → 발송 큐에서 즉시 제거

11:05 - JWT 만료 → 다음 요청에서 새 JWT 발급
         → 새 JWT에는 updated 동의 상태 반영됨

문제 없는 구간: 10:30 ~ 11:05 (35분)
  → 발송 큐에서 즉시 제거했으므로 실제 발송 차단됨
  → JWT가 stale해도 민감 쓰기(결제, 데이터 변경 등)는 @VerifyOnDb로 DB 재검증
```

민감한 작업(결제, 개인정보 변경, 콘텐츠 발행 등)은 `@VerifyOnDb` 데코레이터로 JWT가 stale해도 DB에서 최신 상태를 확인합니다:

```typescript
// 민감 쓰기 핸들러 (예시)
@Post('/publish')
@VerifyOnDb()  // ← JWT stale 방어: DB에서 최신 동의 상태·권한 재확인
async publishLecture(@User() user: AuthUser, ...) {
  // 이 지점에 도달했다는 것은 DB 재검증 통과
  // 사용자가 동의를 철회했거나 구독이 만료됐다면 이미 차단됨
}
```

철회 시 `perm_version`을 bump하면, 프론트엔드는 `X-Perm-Version` 헤더 불일치를 감지해서 권한 스냅샷을 자동으로 다시 요청합니다. 그 순간부터 클라이언트도 최신 동의 상태를 알게 됩니다.

```
철회 발생 시 전파 흐름:

user_consent_event INSERT (action='REVOKED')
  └─▶ perm_version bump (즉시)
        └─▶ 다음 API 응답에 X-Perm-Version 헤더 포함
              └─▶ 클라이언트가 불일치 감지
                    └─▶ /permission-snapshot 재요청
                          └─▶ 최신 동의 상태 로드
```

> 💡 **한 줄 요약**: JWT stale 구간에도 perm_version bump + 발송 큐 즉시 차단 + 민감 쓰기 DB 재검증으로 동의 철회가 지연 없이 반영됩니다.

---

## 마치며

`user_consent_event`의 설계 원칙을 한 줄씩 정리하면:

- **append-only**: 모든 행은 법적 증거, 변조 불가
- **consent_type 네임스페이스**: `platform.*` / `pg.*` / `[service].*`로 분리해 관리
- **terms_version**: 어느 버전에 동의했는지 추적
- **meta_json**: 제3자 제공 4요건 등 구조화된 부가 정보
- **hash chain (P1)**: `prev_hash`/`row_hash`로 위변조 감지 예정

이 구조 덕분에 사용자가 "나는 그 약관에 동의한 적 없다"고 주장해도, "2025-06-01 14:23에 v2025-06 약관에 동의하셨습니다"라고 정확히 응답할 수 있습니다. 그게 바로 이 테이블이 존재하는 이유입니다.

---

## 연결된 개념

- [[audit-hash-chain|audit_log 해시 체인]] — user_consent_event의 prev_hash/row_hash 무결성 보호
- [[break-glass|Break-glass 긴급 접근]] — 동의 없는 비상 접근의 법적 리스크
- [[multitenancy-rls|Pool 모델 + RLS]] — 테넌트별 동의 데이터 격리
- [[gate-abc-flow|Gate A/B/C 전체 흐름]] — 동의 철회 후 JWT stale 문제와 @VerifyOnDb의 관계
> 소스 문서
- [[architecture]] — §1.5 정책 & 동의 전체, §4 정보주체 권리 운영
- [[schema-reference]] — I.1-I.2 user_consent_event DDL
