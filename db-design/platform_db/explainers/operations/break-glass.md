---
difficulty: 중
tags:
  - platform-db
  - explainer
  - p2
  - security
  - audit
  - ops
  - compliance
aliases:
  - break-glass
  - 긴급 접근
  - break_glass 플래그
  - 비상 접근
---

# Break-glass 긴급 접근 설명

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[architecture]] §2.2 보안, §4 · [[schema-reference]] §D.8 audit_log

운영 중에 예기치 못한 장애가 발생하면 "DB를 직접 수정해야 한다"는 압박을 받는 순간이 옵니다. 이 문서는 그 상황을 어떻게 안전하게 다루는지, 왜 아무나 아무 때나 DB를 건드리면 안 되는지를 설명합니다.

---

## Q1. "break-glass"가 뭔가요? 어디서 온 단어인가요?

비상구 옆에 붙어있는 빨간 소화기 박스를 본 적 있으시죠? 유리를 깨야(break glass) 꺼낼 수 있습니다. 평소에는 봉인되어 있지만, 진짜 비상상황에는 봉인을 깨고 사용할 수 있습니다.

```
┌──────────────────────────────────────┐
│   🔴  BREAK GLASS IN EMERGENCY       │
│                                      │
│   ┌────────────────────────┐         │
│   │  긴급 DB 접근 권한     │         │
│   │  (평소에는 잠겨있음)   │         │
│   └────────────────────────┘         │
│                                      │
│  사용 조건: 장애 + 승인자 결재       │
│  사용 후:   전체 이력 감사           │
└──────────────────────────────────────┘
```

보안 업계에서 break-glass는 **일반적인 접근 통제를 우회하는 승인된 예외 경로**를 의미합니다. 핵심 단어는 "**승인된**"입니다. 아무도 모르게 몰래 접근하는 것이 아니라, 정해진 절차를 밟고, 모든 행위가 기록되며, 사후에 검토됩니다.

ISMS-P(정보보호 관리체계), SOC2, ISO 27001 같은 보안 인증에서는 이런 긴급 접근 절차가 있는지, 그리고 그 이력이 감사 가능한지를 중요하게 봅니다.

> 💡 **한 줄 요약**: Break-glass는 비상 상황에 한해 정해진 절차로 허가되는 긴급 운영 접근입니다. 봉인을 깨는 행위 자체가 기록으로 남습니다.

---

## Q2. 운영팀이 장애 시 DB를 직접 수정해야 할 때가 있나요? 그냥 하면 안 되나요?

장애 상황에서는 종종 DB를 직접 수정하고 싶은 충동이 생깁니다. 예를 들어:

- 결제는 됐는데 코드 버그로 `org_entitlement.status`가 `ACTIVE`로 안 바뀐 경우
- 배치 오류로 `EXPIRED`가 돼야 할 구독이 `ACTIVE`로 남아있는 경우
- 잘못된 데이터가 들어가서 특정 사용자가 서비스를 이용하지 못하는 경우

"그냥 DB에 들어가서 한 줄만 고치면 되는 거 아닌가?"라는 생각이 드는 순간입니다. 하지만 이렇게 하면 여러 문제가 생깁니다:

**1. 추적 불가**: 누가, 언제, 왜 수정했는지 기록이 없습니다. 나중에 "이 데이터가 왜 이렇게 됐지?"라는 의문이 생겨도 답을 알 수 없습니다.

**2. 감사 실패**: ISMS-P, SOC2 감사에서 "권한 없는 직원이 production DB를 직접 수정했다"는 기록은 심각한 결함으로 분류됩니다.

**3. 실수 확대**: 황급히 수정하다가 WHERE 절을 빠뜨린 `UPDATE`가 수만 건을 한 번에 바꾸는 참사가 실제로 발생합니다.

```sql
-- 이런 실수가 운영 DB에서 일어남
-- 의도: 특정 1명의 상태를 바꾸려 했으나
UPDATE org_entitlement SET status = 'ACTIVE';  -- WHERE 빠짐!
-- → 전체 org의 entitlement가 ACTIVE로 바뀜
```

그래서 break-glass 절차가 있습니다. "장애 시 DB를 수정해야 한다"는 현실을 인정하되, 그 과정을 안전하게 만드는 것입니다.

> 💡 **한 줄 요약**: DB 직접 수정은 추적 불가, 감사 실패, 실수 확대의 위험이 있습니다. Break-glass는 이를 "해도 되지만 절차대로"로 관리합니다.

---

## Q3. break_glass 플래그가 audit_log에 있는 이유가 뭔가요?

[[audit-hash-chain|audit_log]] 테이블에는 모든 시스템 이벤트가 기록됩니다. 그 중에서 break-glass를 통해 이뤄진 행위는 특별히 표시해야 합니다.

```sql
-- audit_log 테이블의 break_glass 컬럼
break_glass BOOLEAN NOT NULL DEFAULT FALSE,
-- 인덱스는 테이블 밖에서 별도 생성 (PG)
CREATE INDEX idx_audit_break_glass ON audit_log (break_glass, created_at);
```

대부분의 감사 로그는 `break_glass=FALSE`입니다. `TRUE`인 로그만 빠르게 조회할 수 있도록 전용 인덱스를 만들었습니다.

이 설계가 필요한 이유를 구체적으로 설명하면:

```
일반 감사 로그 (하루 수십만 건):
  break_glass=FALSE: 정상적인 API 요청, 시스템 자동화

Break-glass 감사 로그 (월 수 건 이하):
  break_glass=TRUE: 긴급 접근 행위

보안팀이 "지난 6개월 break-glass 이력 모두 보여줘"를 요청하면:

-- 인덱스 없는 경우: 수백만 건 풀스캔
SELECT * FROM audit_log
WHERE break_glass = TRUE
ORDER BY created_at DESC;

-- 인덱스 있는 경우: idx_audit_break_glass로 즉시 조회
-- break_glass=TRUE인 건수가 적으므로 인덱스 효율 매우 높음
```

모든 break-glass 행위는 다음 정보와 함께 기록됩니다:

```sql
INSERT INTO audit_log (
  actor_type,   -- 'OPERATOR' (운영자 평면 — schema-reference §D.8)
  actor_pk,     -- 운영자 identity_user.pk
  action,       -- 수행한 작업 (예: 'MANUAL_ENTITLEMENT_UPDATE')
  resource_type,-- 수정한 테이블 (예: 'org_entitlement')
  resource_pk,  -- 수정한 행 PK
  result,       -- 'ALLOW'
  meta_json,    -- { reason: '결제 정상인데 entitlement 미반영', approver: 'cto@...' }
  break_glass,  -- TRUE ← 이것이 핵심
  created_at
) VALUES (...);
```

> 💡 **한 줄 요약**: `break_glass=TRUE` 행은 일반 로그와 구별해 빠르게 조회하기 위해 전용 인덱스가 있고, 이 로그들이 사후 감사의 핵심 증거가 됩니다.

---

## Q4. break-glass 접근 절차는 어떻게 되나요?

`architecture.md §4`에 명시된 절차를 단계별로 설명합니다:

```
Step 1: 요청 (운영자)
  ├── 사유 작성: "PG 웹훅 처리 실패로 50건 entitlement 미반영"
  ├── 대상 특정: "org_pk IN (1234, 5678, ...) — 영향받은 org 목록"
  ├── 만료 시간: "2시간 후 자동 회수"
  └── 요청 채널: Slack #incident 채널 + 티켓 시스템

Step 2: 승인 (승인자)
  ├── 사유 검토: "실제 장애인가? 다른 방법은 없나?"
  ├── 범위 검토: "왜 50건인가? 더 많거나 적은 건 아닌가?"
  └── 결재: 승인 또는 반려 (이유 명시)

Step 3: 접근 (운영자)
  ├── delegation_grant(capability='platform.break_glass') 발급
  │   또는 break_glass_session 임시 발급
  ├── 정해진 범위 안에서만 작업 수행
  └── 모든 쿼리/행위 자동으로 audit_log(break_glass=TRUE) 기록

Step 4: 회수 (자동)
  ├── 지정한 만료 시간에 자동으로 권한 회수
  └── "작업 다 못 했어도 시간 되면 회수됨"

Step 5: 사후 리뷰 (팀 전체)
  ├── 어떤 작업을 했는지 audit_log에서 확인
  ├── 왜 이런 장애가 났는지 근본 원인 분석
  └── 재발 방지 대책 수립 (자동화, 코드 수정 등)
```

중요한 원칙은 **"절대 silent 금지"** 입니다. 아무도 모르게 DB를 수정하는 것은 어떤 경우에도 허용되지 않습니다. 작업이 아무리 사소해 보여도, break-glass로 한 모든 행위는 기록됩니다.

> 💡 **한 줄 요약**: 요청 → 승인 → 제한된 접근 → 자동 회수 → 사후 리뷰의 5단계. 모든 단계에서 기록이 남습니다.

---

## Q5. break-glass와 임퍼소네이션(impersonation)이 어떻게 다른가요?

이 둘을 혼동하면 안 됩니다. `architecture.md`에도 **"임퍼소네이션(위임)과 다름"**이라고 명시되어 있습니다.

| 구분 | Break-glass | 임퍼소네이션 |
|---|---|---|
| 의미 | 운영자가 자신의 신분으로 긴급 접근 | A가 B인 척 행동 |
| 행위 주체 | 운영자 본인 | 다른 사람을 가장함 |
| 허용 여부 | 절차를 따르면 허용 | **우리 시스템에서 금지** |
| audit_log actor | 운영자 본인 pk | (금지이므로 없음) |
| 사용 상황 | 장애 대응, 데이터 정정 | 고객 지원 "사용자 화면 보기" (금지) |

임퍼소네이션이 위험한 이유([[pipa-consent|PIPA]] §3, §59 위반):

```
임퍼소네이션 시나리오 (우리 시스템에서 금지):

운영자 A가 사용자 김지영(user_pk=999)인 척 로그인
  → 김지영의 개인 데이터 전부 볼 수 있음
  → 김지영 명의로 행위 가능
  → audit_log에 "김지영이 했다"고 기록됨 ← 행위자 위조
  → 나중에 "내가 그 행위를 했다"고 김지영에게 책임 전가 가능

이것은 PIPA §3(개인정보 처리 원칙), §59(금지행위) 위반
```

break-glass는 다릅니다:

```
Break-glass 시나리오 (올바른 방식):

운영자 A가 자신의 계정으로 break-glass 접근
  → A 본인 권한으로 김지영의 entitlement 수정
  → audit_log: actor_pk=A, break_glass=TRUE, resource=org_entitlement
  → "A가 김지영의 entitlement를 수정했다"는 투명한 기록
```

고객 지원에서 "사용자 화면을 직접 보고 싶다"는 요구가 있을 수 있습니다. 이를 위해서는 임퍼소네이션이 아닌, **운영자 전용 read-only 뷰**나 **사용자가 명시적으로 허용하는 지원 세션**을 별도로 만들어야 합니다.

> 💡 **한 줄 요약**: Break-glass는 "운영자가 자기 신분으로 긴급 접근", 임퍼소네이션은 "남인 척 행동". 우리 시스템에서 임퍼소네이션은 금지입니다.

---

## Q6. 운영자가 break-glass를 남용하면 어떻게 탐지하나요?

break-glass는 비상용입니다. 월에 수백 번 쓰인다면 그건 비상이 아니라 일상이 된 것이고, 심각한 보안 문제입니다.

탐지 방법은 크게 세 가지입니다:

**1. 빈도 알림**: break-glass 이벤트가 발생하면 즉시 알림

```typescript
// audit_log에 break_glass=TRUE가 INSERT될 때마다
// 보안팀 Slack 채널에 알림 발송
async function onAuditLogInsert(log: AuditLog) {
  if (log.breakGlass === true) {
    await slack.notify({
      channel: '#security-alerts',
      text: `🚨 Break-glass 접근 감지
운영자: ${log.actorPk}
작업: ${log.action}
대상: ${log.resourceType} #${log.resourcePk}
시각: ${log.createdAt}
사유: ${log.metaJson?.reason}`,
    });
  }
}
```

**2. 정기 리뷰**: 월 1회 break-glass 이력 전체 검토

```sql
-- 월간 break-glass 리뷰 쿼리
SELECT
  actor_pk,
  COUNT(*) AS action_count,
  MIN(created_at) AS first_action,
  MAX(created_at) AS last_action,
  jsonb_agg(DISTINCT action) AS actions
FROM audit_log
WHERE break_glass = TRUE
  AND created_at >= now() - INTERVAL '1 month'
GROUP BY actor_pk
ORDER BY action_count DESC;
```

**3. 패턴 이상 감지**: 정상 break-glass와 다른 패턴 탐지

```
의심스러운 패턴:
  ❌ 승인 기록 없이 break_glass=TRUE 이벤트 발생
  ❌ 같은 운영자가 짧은 시간에 다수의 다른 org 데이터 접근
  ❌ 업무 시간 외(새벽, 주말) 빈번한 접근
  ❌ 접근 이유(meta_json.reason)가 구체적이지 않음
  ❌ 승인된 범위(특정 org_pk)를 벗어난 접근

정상 break-glass 패턴:
  ✅ 티켓/슬랙에 승인 기록이 있음
  ✅ 접근 범위가 명시된 장애 대상으로 한정됨
  ✅ 작업 후 짧은 시간에 완료
  ✅ 사후 리뷰 문서 작성됨
```

**SOC2 / ISMS-P 감사 시 실제로 확인하는 것**들입니다:
- break-glass 접근 이력이 모두 승인 티켓과 매칭되는가
- 자동 회수가 실제로 동작했는가
- 사후 리뷰가 수행됐는가
- 재발 방지 조치가 있었는가

> 💡 **한 줄 요약**: 발생 즉시 알림 + 월간 리뷰 + 패턴 이상 감지로 남용을 탐지합니다. SOC2/ISMS-P 감사에서 이 이력이 핵심 증거입니다.

---

## 마치며

Break-glass는 "통제된 비상구"입니다. 없으면 장애 대응이 불가능하고, 있어도 통제 없이 쓰면 더 큰 보안 사고가 됩니다.

`platform_db`의 break-glass 설계는 세 가지를 보장합니다:

1. **접근 가능**: 진짜 장애 상황에서 막히지 않는다
2. **추적 가능**: 무슨 일이 있었는지 나중에 정확히 알 수 있다
3. **남용 방지**: 정상 절차를 벗어난 사용을 감지하고 차단한다

현재 break-glass는 P1 구현 대상입니다. 서비스 규모가 커지고 운영팀이 생기면, 이 절차를 가장 먼저 정비해야 합니다. ISMS-P 심사에서 빠지지 않고 점검하는 항목이기도 합니다.

가장 중요한 원칙 하나만 기억하세요: **"절대 silent 금지"**. 모든 break-glass 행위는 기록으로 남습니다.

---

## 연결된 개념

- [[audit-hash-chain|audit_log 해시 체인]] — break_glass=true 이벤트의 무결성 보장
- [[pipa-consent|PIPA 동의]] — 동의 없는 비상 접근이 가져오는 법적 리스크
> 소스 문서
- [[architecture]] — §2.2 보안, §4 Break-glass 절차
- [[schema-reference]] — D.8 audit_log DDL (break_glass 컬럼, idx_audit_break_glass 인덱스)
