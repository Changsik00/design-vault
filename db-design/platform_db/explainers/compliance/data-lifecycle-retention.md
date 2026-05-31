---
difficulty: 중
tags:
  - platform-db
  - explainer
  - p1
  - operability
  - retention
  - lifecycle
  - pipa
  - partitioning
aliases:
  - 데이터 생명주기
  - 보존
  - 파기
  - retention
  - purge
  - anonymize
  - 보존 매트릭스
---

# 데이터 생명주기와 보존·파기 (retention)

> **대상**: DB 지식이 많지 않은 개발자
> **연관 문서**: [[operability|operability.md O3]] · [[requirements|requirements.md RETN-1]] · [[schema-reference|schema-reference.md §N.2·§D.8·§I]] · [[delete-patterns]]

"이 데이터는 언제 지워야 하나요?"라고 물으면 많은 사람이 "안 지워도 되지 않나요? 디스크 싸잖아요"라고 답합니다. 이건 두 가지를 동시에 놓친 답입니다. 첫째, **법적으로 일정 기간은 *반드시 남겨야* 하는** 데이터가 있습니다(보존 의무). 둘째, **법적으로 일정 기간이 지나면 *반드시 지워야* 하는** 데이터가 있습니다(파기 의무). 이 문서는 `platform_db`가 데이터를 어떻게 태어나게 하고, 숨기고, 익명화하고, 마지막에 파기하는지 — 그 전체 생명주기를 설명합니다.

soft-delete의 세 패턴(status / deleted_at / append-only) 자체는 [[delete-patterns]]에서 이미 다뤘습니다. 이 문서는 그 다음 단계 — "soft-delete된 데이터가 *결국 어떻게 사라지는가*"에 집중합니다.

---

## Q1. 데이터를 그냥 영원히 쌓아두면 안 되나요? 디스크는 싸잖아요.

안 됩니다. 두 가지 의무가 **동시에** 걸려 있기 때문입니다.

**비유 — 회사 서류 캐비닛**을 떠올려 보세요. 회계 장부는 세무서가 "7년은 보관하라"고 요구합니다(보존 의무). 반면 채용 탈락자 이력서는 개인정보보호법이 "목적 끝나면 파쇄하라"고 요구합니다(파기 의무). 그래서 캐비닛 관리자는 두 가지 일을 모두 합니다 — 어떤 서류는 7년 동안 절대 못 버리게 잠가두고, 어떤 서류는 1년 뒤 파쇄기에 넣습니다.

만약 "디스크 싸니까 다 쌓아두자"고 하면:

```
무한정 쌓을 때 터지는 곳 3가지

① 비용·성능   audit_log가 5년치 = 수억 행 → 인덱스 비대, 백업 시간 폭증
② 유출 위험   탈퇴한 회원의 이메일·전화번호가 3년 뒤에도 그대로 → 유출 시 책임
③ 법 위반     PIPA "보유기간 경과 시 지체 없이 파기"(§21) 위반 → 과징금
```

특히 ③번이 핵심입니다. **"안 지운 것" 자체가 위법**이 될 수 있습니다. 개인정보는 "처리 목적이 끝나면 지워야" 하는데, 목적이 끝났는데도 들고 있으면 그것만으로 파기 의무 위반입니다.

> 💡 **한 줄 요약**: 데이터에는 "남겨야 할 의무(보존)"와 "지워야 할 의무(파기)"가 동시에 걸려 있어서, 무한정 쌓으면 비용·유출·법 위반 세 가지가 한꺼번에 터집니다.

---

## Q2. 데이터 하나가 태어나서 사라지기까지 어떤 단계를 거치나요?

`platform_db`의 데이터는 다음 6단계를 거칩니다. `identity_user`(회원)를 예로 들면 이렇습니다.

```
① 생성       INSERT → status='ACTIVE', deleted_at=NULL
② 활성        평소 사용 중. 모든 컬럼 정상
③ soft-delete 탈퇴 요청 → status='DELETED', deleted_at=NOW()
                ↑ 행은 그대로 있고 "지워진 것으로 표시"만 함 ([[delete-patterns]])
④ hard anonymize  탈퇴 30일 뒤 배치 → email/phone/firebase_uid = NULL
                ↑ 행은 남기되 "개인 식별 컬럼만 NULL"로 만듦
⑤ 보존기간     익명화된 행이 보존 의무 기간 동안 유지 (테이블마다 다름)
⑥ 파기        보존기간 경과 → 파티션 DROP 또는 행 삭제로 물리 제거
```

여기서 ③→④가 헷갈리기 쉽습니다. **soft-delete는 "표시"일 뿐 데이터는 그대로 있습니다.** 탈퇴 직후 30일은 변심·분쟁·오삭제 복구를 위해 데이터를 살려둡니다. 30일이 지나야 비로소 개인 식별 컬럼을 NULL로 밀어버립니다(익명화). schema-reference §D.1에 그대로 명시돼 있습니다:

```
탈퇴 처리 시: email=NULL, phone_e164=NULL, firebase_uid=NULL,
              email_verified=FALSE  (hard anonymize, 30일 배치)
```

`identity_user`의 행 자체(pk)는 남습니다. 왜냐하면 `audit_log`에 `actor_pk=10`으로 남은 과거 행위 기록과 join 가능해야 하고(누가 한 일인지 추적), 그러나 "그게 누구인지(이메일·전화)"는 더 이상 알 수 없게 만들기 때문입니다. **"행위는 남기되 사람은 지운다"** — 이것이 익명화의 핵심입니다.

> 💡 **한 줄 요약**: 생성 → 활성 → soft-delete(표시) → 30일 뒤 hard anonymize(식별 컬럼 NULL) → 보존기간 → 파기. soft-delete는 아직 진짜 삭제가 아니라 복구 가능한 "표시"입니다.

---

## Q3. retain / purge / anonymize — 비슷해 보이는데 정확히 뭐가 다른가요?

세 단어가 자주 섞여 쓰여서 혼란스럽지만, 하는 일이 완전히 다릅니다.

| 동작 | 한국어 | 행이 남나? | 식별정보가 남나? | 언제 |
|---|---|---|---|---|
| **retain** | 보존 | 남음 | 남음 (의도적) | 보존 의무 기간 동안 |
| **anonymize** | 익명화 | 남음 | **지워짐** (컬럼 NULL) | 목적 종료(탈퇴 30일) |
| **purge** | 파기 | **지워짐** | 지워짐 | 보존기간 경과 후 |

```
anonymize = 사람을 지운다 (행은 남기고 식별 컬럼만 NULL)
            "이게 누구의 데이터인지" 알 수 없게 → 더는 개인정보 아님

purge     = 데이터를 지운다 (행 자체를 물리 삭제)
            파티션 DROP, 또는 보존기간 끝난 데이터 완전 제거
```

가장 중요한 통찰: **익명화된 데이터는 더 이상 "개인정보"가 아닙니다.** 누구인지 알 수 없으니까요. 그래서 PIPA의 파기 의무도 적용되지 않고, 통계·감사 목적으로 더 오래 보존할 수 있습니다. 즉 익명화는 "파기 의무를 회피하면서 데이터의 유용성(집계·감사)은 남기는" 절충안입니다.

```sql
-- anonymize: 행은 남기고 사람만 지움 (30일 배치)
UPDATE identity_user
SET email=NULL, phone_e164=NULL, firebase_uid=NULL, email_verified=FALSE
WHERE status='DELETED' AND deleted_at < NOW() - INTERVAL 30 DAY;
-- pk=10 행은 여전히 존재 → audit_log.actor_pk=10 join 가능
-- 그러나 "pk=10이 누구인지"는 영영 알 수 없음

-- purge: 행 자체를 제거 (파티션 DROP — Q5에서 상세)
ALTER TABLE audit_log DROP PARTITION p202101;  -- 5년 지난 월 통째로 삭제
```

> 💡 **한 줄 요약**: retain은 의도적으로 남기는 것, anonymize는 행은 남기고 사람만 지우는 것(더는 개인정보 아님), purge는 행 자체를 물리 삭제하는 것입니다.

---

## Q4. 테이블마다 보존기간이 다르다는데, 왜 다르고 무슨 기준인가요?

보존기간은 디스크 사정이 아니라 **각 데이터에 걸린 법·분쟁·운영 사유**로 결정됩니다. `platform_db`의 보존 매트릭스(operability O3)는 다음과 같습니다.

| 테이블 | 보존 | 파기 방법 | 왜 이 기간? |
|---|---|---|---|
| `audit_log` | 5년 | 월별 파티션 DROP | ISMS-P 보안 감사 요건 |
| `user_consent_event` | 5년 | append-only, 5년 후 파기 | **PIPA 동의 증거** — "동의받았다"를 입증할 기간 |
| `payment_ledger` | 5~7년 | append-only | 세법·전자상거래법(거래 기록 보존 의무) |
| `pg_webhook_event` | 90일 | sweeper DELETE | 운영 데이터(재처리 끝나면 불필요) |
| `outbox_event` | PROCESSED 후 30일 | sweeper DELETE | 운영 데이터(발행 끝나면 불필요) |
| access 텔레메트리 | 30~90일 | OLAP TTL | 운영(트렌드만 보면 됨) |
| `identity_user`(탈퇴) | DELETED 후 30일 → 익명화 | hard anonymize 배치 | PIPA(목적 종료 시 파기) |
| chat/usage 이벤트 | 1년 → S3 아카이브 | 서비스 DB 소관 | 비용(핫 스토리지에서 내림) |

보존 사유는 크게 **세 부류**로 나뉩니다.

```
법적 보존 의무   audit_log(ISMS-P 5년), consent(PIPA 5년), payment(세법 7년)
                → 짧게 잡으면 위법. 함부로 못 줄임.

분쟁 대응        consent_event(누가 언제 동의/철회했나)
                → 마케팅 수신 분쟁, 14세 미만 동의 분쟁 등의 증거

운영 편의        outbox/webhook(처리 끝난 이벤트), 텔레메트리
                → 법 무관. 빨리 지울수록 좋음(무한 누적 방지)
```

여기서 핵심 구분: **`user_consent_event`(consent)는 보존 대상이지만 `outbox_event`는 보존 대상이 아닙니다.** 동의 이력은 법적 증거라 5년을 채워야 하고, outbox는 "이벤트를 발행했다"는 처리 흔적일 뿐이라 30일이면 충분합니다. 같은 "이벤트 테이블"처럼 보여도 성격이 완전히 다릅니다.

> 💡 **한 줄 요약**: 보존기간은 디스크가 아니라 법(감사·세법·PIPA)·분쟁·운영 사유로 정해집니다. consent·audit·payment는 법적 증거라 5~7년, outbox·webhook은 운영 흔적이라 30~90일입니다.

---

## Q5. 5년 지난 audit_log를 어떻게 파기하나요? `DELETE FROM ...` 하면 되나요?

안 됩니다. 두 가지 이유로 행 단위 `DELETE`는 막혀 있습니다.

**이유 1 — 권한이 없습니다.** `platform_db`의 앱 계정은 append-only 테이블에 대해 DELETE 권한 자체가 없습니다([[least-privilege-grant]]). `audit_log`는 `audit_append` 계정으로 INSERT만 가능하고, 누구도 UPDATE·DELETE를 칠 수 없습니다(WORM 원칙). 코드가 실수로 `DELETE`를 쳐도 DB가 거부합니다.

**이유 2 — 느리고 위험합니다.** 수억 행 테이블에서 `DELETE FROM audit_log WHERE created_at < ...`는 수백만 행을 한 줄씩 지우며 락·로그·인덱스 재구성을 유발합니다. 운영 중 DB를 마비시킬 수 있습니다.

그래서 **파티션 DROP**을 씁니다. `audit_log`는 월별 RANGE 파티셔닝되어 있습니다([[partitioning]], schema-reference §D.8):

```sql
CREATE TABLE audit_log (...)
PARTITION BY RANGE COLUMNS(created_at) (
  PARTITION p202601 VALUES LESS THAN ('2026-02-01 00:00:00'),
  PARTITION p202602 VALUES LESS THAN ('2026-03-01 00:00:00'),
  ...
  PARTITION p_future VALUES LESS THAN (MAXVALUE)
);
```

5년 지난 달을 통째로 떨어내는 건 **메타데이터 한 줄 조작**입니다 — 거의 즉시 끝나고 락도 없습니다.

```sql
-- ✅ 2021년 1월 파티션 통째 DROP — 즉시, 락 없음
ALTER TABLE audit_log DROP PARTITION p202101;

-- ❌ 같은 일을 DELETE로 하면 — 권한 없음 + 수백만 행 락
DELETE FROM audit_log WHERE created_at < '2021-02-01';
```

**비유**: DELETE는 캐비닛 서랍을 열어 종이를 한 장씩 파쇄기에 넣는 것이고, 파티션 DROP은 "2021년 1월" 라벨이 붙은 서랍을 통째로 들어내 버리는 것입니다. 후자가 비교가 안 되게 빠릅니다.

`user_consent_event`도 같은 패턴을 검토 중입니다(append-only + 5년 보존이라 `audit_log`와 동일 — schema-reference §I.2 파티셔닝 검토). 파티션이 없는 테이블을 파기해야 할 땐, 외부 WORM 스토리지로 **EXPORT한 뒤** 파티션 DROP(또는 아카이브 후 제거)하는 것이 원칙입니다(schema-reference §H, 파티션 DROP 금지 → EXPORT 후 DROP).

> 💡 **한 줄 요약**: 행 단위 DELETE는 권한도 없고(append-only WORM) 느려서, 5년 지난 데이터는 월별 파티션을 통째로 DROP합니다 — 메타데이터 한 줄 조작이라 즉시 끝나고 락이 없습니다.

---

## Q6. outbox_event·pg_webhook_event는 무한히 쌓인다던데, 누가 정리하나요? (sweeper)

이게 **현재 미구현 갭**입니다(schema-reference §N.2, operability O3 — 둘 다 🔴).

`outbox_event`와 `pg_webhook_event`는 "처리하고 나면 더는 필요 없는" 운영 데이터입니다. 그런데 처리가 끝나도 행이 자동으로 사라지진 않습니다 — `status`만 PENDING→SENT(outbox) 또는 RECEIVED→PROCESSED(webhook)로 바뀔 뿐입니다.

```sql
-- outbox_event: 발행 끝나면 status='SENT', sent_at 기록 — 행은 그대로
-- pg_webhook_event: 처리 끝나면 status='PROCESSED', processed_at 기록 — 행은 그대로
```

정리하는 주체가 없으면 이 행들이 **무한 누적**됩니다. billing이 활발할수록 빠르게 불어나고, 결국 인덱스 비대·디스크 부족으로 이어집니다. 이걸 정리하는 배치가 **sweeper**입니다.

```sql
-- sweeper: 처리 끝난 지 오래된 운영 이벤트만 청소 (일 1회 배치)

-- outbox: SENT/PROCESSED 된 지 30일 지난 것만 DELETE
DELETE FROM outbox_event
WHERE status = 'SENT' AND sent_at < NOW() - INTERVAL 30 DAY;

-- webhook: PROCESSED 된 지 90일 지난 것만 DELETE
DELETE FROM pg_webhook_event
WHERE status = 'PROCESSED' AND created_at < NOW() - INTERVAL 90 DAY;
```

sweeper의 두 가지 핵심 안전 규칙:

```
① 보존 대상 아님   outbox/webhook은 법적 증거가 아니라 운영 흔적
                  → 영구 보존 불필요. 처리 끝나면 청소해도 됨
② 처리된 것만 청소  PENDING/RECEIVED/FAILED는 절대 건드리면 안 됨
                  → 아직 처리 안 끝났거나 재처리해야 할 데이터
                  → 이걸 지우면 결제 성공했는데 권한 안 열리는 사고 발생
```

②번이 특히 중요합니다. `FAILED` webhook은 운영자가 나중에 재처리(Webhook Replay, operability O2)해야 하므로 sweeper가 절대 건드리면 안 됩니다. **"처리 완료(SENT/PROCESSED) + 충분히 오래됨"** 두 조건을 모두 만족해야만 지웁니다.

> 💡 **한 줄 요약**: outbox·webhook은 처리가 끝나도 행이 안 사라져 무한 누적되므로, sweeper 배치가 PROCESSED/SENT된 지 30/90일 지난 것만 골라 지웁니다 — PENDING·FAILED는 절대 건드리지 않습니다. (현재 미구현 갭)

---

## Q7. "지워야 할 의무"와 "남겨야 할 의무"가 충돌하면요? 탈퇴했는데 동의 이력을 남기면 불법 아닌가요?

좋은 질문입니다. 그리고 이게 보존·파기 설계에서 가장 까다로운 부분입니다.

**상황**: 회원이 탈퇴하면 PIPA는 "개인정보를 지체 없이 파기하라"(파기 의무)고 합니다. 그런데 동시에, "이 회원이 마케팅 수신에 동의했었다"는 기록(`user_consent_event`)은 **분쟁 대비 5년 보존**해야 합니다(보존 의무). 둘이 충돌하는 것처럼 보입니다.

해결책은 Q3의 익명화입니다. **"기록은 남기되, 그게 누구인지는 지운다."**

```
탈퇴 시:
  identity_user        → status='DELETED', 30일 뒤 식별 컬럼 익명화
  user_consent_event   → 행 그대로 보존 (5년) — 단, user_pk만 남고
                          그 user_pk가 누구인지(이메일 등)는 익명화됨
```

핵심은 `user_consent_event`의 행 자체는 **지우지 않는다**는 것입니다. append-only라 UPDATE·DELETE 권한도 없습니다([[delete-patterns]] Q4). 대신 그 행이 가리키는 `identity_user`의 식별 컬럼이 NULL이 되면서, "동의는 있었으나 누구였는지는 알 수 없는" 상태가 됩니다.

```sql
-- 탈퇴 fan-out: 전체 동의 일괄 철회 마커 INSERT (감사 마커, schema-reference §I.2)
INSERT INTO user_consent_event (user_pk, consent_type, action, version, created_at)
VALUES (10, 'ALL', 'REVOKED', '2026-05-01', NOW());
-- ↑ 기존 GRANTED 행은 그대로 남음 (증거). 철회 사실만 새 행으로 추가

-- 그런데 user_pk=10이 누구인지는?
-- → identity_user(pk=10)의 email/phone이 30일 뒤 NULL → 식별 불가
```

이렇게 하면 두 의무가 모두 충족됩니다:

```
파기 의무  ✅  "누구인지"는 익명화로 지워짐 (개인 식별 불가)
보존 의무  ✅  "동의/철회가 있었다"는 사실은 5년 보존 (분쟁 증거)
```

**비유로 돌아가면**: 탈퇴자의 이름·연락처는 파쇄하되, "2026년 1월에 마케팅 동의서에 서명이 있었다"는 사실 자체는 익명 처리한 채 5년 캐비닛에 둡니다. 나중에 "동의 없이 마케팅 보냈잖아"라는 분쟁이 오면, 익명화된 그 기록이 "그 시점엔 동의가 있었다"를 입증합니다.

> 💡 **한 줄 요약**: 충돌은 익명화로 풉니다 — 동의 이력 행은 5년 보존하되(보존 의무), 그게 누구의 것인지는 식별 컬럼 익명화로 지웁니다(파기 의무). "사실은 남기고 사람은 지운다"가 두 의무를 동시에 만족시킵니다.

---

## 용어 정리

| 용어 | 뜻 | platform_db에서 |
|---|---|---|
| **retention(보존)** | 데이터를 의도적으로 일정 기간 유지 | 보존 매트릭스(audit 5y·consent 5y·payment 7y) |
| **purge(파기)** | 행 자체를 물리적으로 제거 | 파티션 DROP, 또는 보존기간 경과 데이터 삭제 |
| **anonymize(익명화)** | 행은 남기고 개인 식별 컬럼만 NULL | 탈퇴 30일 배치(`email/phone/firebase_uid=NULL`) |
| **soft-delete** | "지워진 것으로 표시"(복구 가능, 행은 존재) | `status='DELETED'` + `deleted_at` ([[delete-patterns]]) |
| **hard delete** | 행을 진짜로 삭제 (`DELETE FROM`) | append-only 테이블엔 권한 없음([[least-privilege-grant]]) |
| **hard anonymize** | soft-delete 후 식별 컬럼을 영구 NULL 처리 | 탈퇴 30일 경과 배치 (복구 불가) |
| **sweeper** | 처리 끝난 운영 데이터를 주기 청소하는 배치 | outbox/webhook PROCESSED 30/90일 후 DELETE(미구현) |
| **파티션 DROP** | 월별 파티션을 통째로 떨어내 파기 | `ALTER TABLE ... DROP PARTITION`(즉시, 락 없음) |
| **WORM** | Write Once Read Many — 한 번 쓰면 못 고침 | append-only 4종(audit·consent·payment·billing) |
| **PIPA 보존 의무** | 일정 기간 *반드시 남겨야* 함 | consent 5년(동의 증거) |
| **PIPA 파기 의무** | 목적 종료 시 *반드시 지워야* 함(§21) | 탈퇴 시 식별정보 익명화 |

---

## 테스트 방법
> 🧪 *실제 DB·ORM·운영에서 돌리는 법*: [[testing-strategy]] · [[orm-testing-drizzle]]


보존·파기는 "안 일어나야 할 일이 안 일어남"을 검증하는 음성(negative) 테스트가 많아서 특히 꼼꼼해야 합니다. 시간이 지나야 발동하는 배치라 **시간을 조작**(`deleted_at`을 과거로 INSERT)하는 게 핵심 기법입니다.

**테스트 1 — 30일 경과 후 hard anonymize가 식별 컬럼을 NULL로 만드는가**

```sql
-- given: 31일 전 탈퇴한 회원
INSERT INTO identity_user (firebase_uid, email, phone_e164, status, deleted_at, ...)
VALUES ('uid-x', 'gone@test.com', '+821011112222', 'DELETED',
        NOW() - INTERVAL 31 DAY, ...);

-- when: anonymize 배치 실행

-- then: 식별 컬럼은 NULL, 행(pk)은 존재
SELECT email, phone_e164, firebase_uid, email_verified
FROM identity_user WHERE pk = @pk;
-- 기대: NULL, NULL, NULL, FALSE   ← 사람은 지워졌으나 행은 남음
```

**테스트 2 — 보존기간 안 지난 데이터는 파기되지 않는가 (음성 테스트)**

```sql
-- given: 29일 전 탈퇴 (아직 30일 안 됨)
INSERT INTO identity_user (..., status='DELETED', deleted_at=NOW() - INTERVAL 29 DAY);

-- when: anonymize 배치 실행

-- then: 아직 익명화되면 안 됨 — 복구 가능 기간
SELECT email FROM identity_user WHERE pk = @pk;
-- 기대: 'gone@test.com' (그대로) ← 29일은 건드리면 안 됨
```

**테스트 3 — sweeper가 PROCESSED만 지우고 PENDING/FAILED는 남기는가**

```sql
-- given: 세 가지 상태의 오래된 outbox 행
INSERT INTO outbox_event (status, sent_at, created_at) VALUES
  ('SENT',    NOW() - INTERVAL 40 DAY, NOW() - INTERVAL 40 DAY),  -- 청소 대상
  ('PENDING', NULL,                    NOW() - INTERVAL 40 DAY),  -- 남아야 함
  ('FAILED',  NULL,                    NOW() - INTERVAL 40 DAY);  -- 남아야 함

-- when: sweeper 실행 (SENT + 30일 경과만)

-- then:
SELECT status, COUNT(*) FROM outbox_event GROUP BY status;
-- 기대: SENT 0건(삭제됨), PENDING 1건, FAILED 1건 (보존)
--       ↑ PENDING/FAILED를 지우면 결제 사고 → 절대 안 됨
```

**테스트 4 — 파티션 DROP 후 해당 기간 데이터가 조회되지 않는가**

```sql
-- given: 2021-01 파티션에 audit_log 행 존재
SELECT COUNT(*) FROM audit_log WHERE created_at < '2021-02-01';  -- > 0

-- when:
ALTER TABLE audit_log DROP PARTITION p202101;

-- then: 해당 기간 조회 시 0건, 그러나 최신 데이터는 정상
SELECT COUNT(*) FROM audit_log WHERE created_at < '2021-02-01';  -- 기대: 0
SELECT COUNT(*) FROM audit_log WHERE created_at >= '2026-01-01'; -- 기대: 정상
```

**테스트 5 — 충돌 해소: 탈퇴 후에도 동의 이력은 보존되는가**

```sql
-- given: 동의 이력이 있는 회원이 탈퇴 + 익명화 완료
-- when: anonymize 배치 후

-- then: consent 행은 5년 보존 (행 살아있음), 단 user는 식별 불가
SELECT COUNT(*) FROM user_consent_event WHERE user_pk = @pk;  -- 기대: > 0 (보존)
SELECT email FROM identity_user WHERE pk = @pk;               -- 기대: NULL (익명화)
```

**자가진단 체크리스트**:

```
□ 새 테이블을 만들 때 "이건 몇 년 보존인가?"를 보존 매트릭스에 추가했나?
□ 그 보존기간의 근거가 법(감사/세법/PIPA)인가, 분쟁 대비인가, 운영인가?
□ 파기 방법이 정해졌나? (파티션 DROP / sweeper DELETE / 익명화)
□ append-only 테이블이면 DELETE 권한이 없음을 확인했나? (파티션 DROP만 가능)
□ sweeper 대상이면 "처리 완료 상태"만 지우고 PENDING/FAILED는 보존하나?
□ 개인 식별 정보가 있으면 탈퇴 시 익명화 경로가 있나?
□ 보존 의무와 파기 의무가 충돌하는가? 충돌하면 익명화로 풀 수 있나?
```

---

## 마치며

데이터 생명주기는 결국 두 개의 시계를 동시에 보는 일입니다. 하나는 **"최소 이만큼은 남겨라"**(보존 의무)를 가리키고, 다른 하나는 **"이때까지는 지워라"**(파기 의무)를 가리킵니다. 둘 다 법이 정한 시계라 마음대로 못 돌립니다.

`platform_db`가 이걸 다루는 방식을 한 문장으로 요약하면: **"법적 증거는 append-only로 5년 쌓고 파티션으로 파기, 운영 흔적은 sweeper로 청소, 개인정보는 익명화로 '사람만' 지운다."**

지금 가장 큰 위험은 기술이 아니라 **미구현입니다.** sweeper·retention 배치·파티션 자동 추가가 전부 🔴(schema-reference §N.2)입니다. 즉 보존 매트릭스는 문서로만 존재하고, 실제로는:

- audit_log가 5년 넘어도 파기되지 않고(파기 의무 위반 위험),
- 탈퇴 30일 지나도 익명화 배치가 없어 식별정보가 남아 있고(PIPA 위반 위험),
- outbox/webhook이 무한 누적되고 있을 수 있습니다.

새 테이블을 만들거나 새 서비스를 붙일 때는 "이 데이터, 언제 어떻게 사라지나?"를 반드시 보존 매트릭스에 등록하세요. 등록되지 않은 데이터는 영원히 쌓이는 데이터이고, 그건 비용·유출·법 위반의 시한폭탄입니다.

---

## 연결된 개념

- [[delete-patterns]] — soft-delete 3종(status/deleted_at/append-only). 생명주기 ③단계의 메커니즘
- [[partitioning]] — 월별 RANGE 파티셔닝. 5년 파기를 "파티션 DROP 한 줄"로 만드는 기반
- [[least-privilege-grant]] — append-only 계정엔 DELETE 권한 없음 → 행 단위 파기 불가, 파티션 DROP만 가능
- [[outbox-pattern]] — sweeper가 청소하는 PROCESSED outbox_event의 발행 메커니즘
- [[pipa-consent]] — user_consent_event 5년 보존(동의 증거)과 탈퇴 시 익명화의 법적 맥락
- [[audit-hash-chain]] — audit_log append-only + 해시 체이닝. 5년 보존 데이터의 무결성
> 소스 문서
- [[operability]] — O3 Data Lifecycle 보존 매트릭스 + 정기 배치 표
- [[requirements]] — RETN-1 데이터 보존·생명주기 정책 (🔴 미설계)
- [[schema-reference]] — §D.1 identity_user(탈퇴 30일 익명화), §D.8 audit_log 월 파티션, §D.18/§D.19 webhook·outbox, §I.2 user_consent_event 5년, §N.2 sweeper·보존 갭
- [[architecture]] — §2.1 불변식(soft-delete·append-only), §1.5 정책·동의
