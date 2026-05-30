# platform_db 개념 설명 문서 (Explainers)

> 대상: DB 지식이 많지 않은 개발자  
> 형식: Q&A — 실제로 궁금한 질문을 던지고, 왜 그렇게 설계했는지 설명  
> 참고 원문: [[architecture]] · [[schema-reference]] · [[decisions/gate-b-billing-grace|gate-b-billing-grace (설계 결정)]]

---

## 문서 목록

### P0 — 코드 짜기 전에 반드시 읽어야 하는 것

| # | 파일 | 주제 | 핵심 질문 | 상태 |
|---|------|------|-----------|------|
| 1 | [[gate-abc-flow]] | Gate A/B/C 전체 흐름 | API 요청 하나가 DB에서 어떤 3단계를 거치나요? | ✅ |
| 2 | [[gate-b-entitlement]] | Gate B & 엔타이틀먼트 개념 | 결제 상태랑 서비스 접근이 왜 다른 테이블인가요? | ✅ |
| 3 | [[role-capability]] | role 2단 분리 + capability | platform_role이 뭐고 service_role이 뭔가요? 왜 둘이 있나요? | ✅ |
| 4 | [[pk-ulid-strategy]] | BIGINT pk + ULID public_id | 테이블마다 pk랑 public_id가 두 개인데, 뭘 쓰면 되나요? | ✅ |
| 5 | [[multitenancy-rls]] | Pool 모델 + RLS 개념 | MySQL은 RLS가 없는데 다른 org 데이터를 어떻게 못 보게 막나요? | ✅ |

---

### P1 — 기능 구현 전에 읽으면 좋은 것

| # | 파일 | 주제 | 핵심 질문 | 상태 |
|---|------|------|-----------|------|
| 6 | [[explainers/gate-b-billing-grace\|gate-b-billing-grace]] | Gate B + 유예 기간 설계 결정 | status만 보면 안 되고 validUntil도 같이 봐야 하는 이유가 뭔가요? | ✅ |
| 7 | [[enum-vs-varchar-check]] | ENUM vs VARCHAR+CHECK (D6) | ENUM 쓰면 편한데 왜 VARCHAR+CHECK를 쓰나요? | ✅ |
| 8 | [[subscription-lifecycle]] | 구독 상태 머신 | TRIALING→ACTIVE→CANCELED→EXPIRED 각 상태에서 무슨 일이 생기나요? | ✅ |
| 9 | [[feature-limits]] | feature_limits 3중 정의 우선순위 | product_feature, plan_definition, org_entitlement 중 뭘 읽으면 되나요? | ✅ |
| 10 | [[idempotency-key]] | 멱등성 키 (payment_ledger) | 결제 요청을 두 번 보내면 어떻게 되나요? 멱등성이 왜 필요하죠? | ✅ |
| 11 | [[outbox-pattern]] | Outbox 패턴 | 이벤트를 그냥 Kafka/메시지큐로 바로 보내면 안 되나요? | ✅ |
| 12 | [[webhook-processing]] | PG 웹훅 수신·처리 흐름 | Toss/Stripe 결제 결과가 DB에 어떻게 반영되나요? | ✅ |
| 13 | [[index-design]] | 인덱스 설계 원리 | 인덱스가 없으면 뭐가 문제인가요? 복합 인덱스는 어떻게 작동하나요? | ✅ |
| 14 | [[pipa-consent]] | PIPA 동의 요건 | 개인정보 동의가 왜 테이블이 따로 있고, 철회를 즉시 반영해야 하나요? | ✅ |

---

### P2 — 운영·확장할 때 필요한 것

| # | 파일 | 주제 | 핵심 질문 | 상태 |
|---|------|------|-----------|------|
| 15 | [[break-glass]] | break_glass 긴급 접근 | 긴급 상황에 운영팀이 직접 DB를 건드릴 때 왜 특별한 컬럼이 필요한가요? | ✅ |
| 16 | [[partitioning]] | DB 파티셔닝 (audit_log) | 파티션이 뭔가요? 왜 audit_log만 파티션을 쓰나요? | ✅ |
| 17 | [[online-ddl-migration]] | 온라인 DDL & 마이그레이션 위험 | 운영 중인 대형 테이블에 컬럼 추가하면 왜 서비스가 멈출 수 있나요? | ✅ |
| 18 | [[audit-hash-chain]] | audit_log 해시 체인 무결성 | 로그 파일에 해시값을 왜 저장하나요? 그냥 로그 남기면 안 되나요? | ✅ |

---

## 진행 현황

| 우선순위 | 전체 | 완료 | 잔여 |
|---------|------|------|------|
| P0 | 5 | 5 | 0 |
| P1 | 9 | 9 | 0 |
| P2 | 4 | 4 | 0 |
| **합계** | **18** | **18** | **0** |

---

## 읽는 순서 추천

처음 합류한 개발자라면:

```
1번 → 2번 → 4번 → 5번 → 3번   (Gate 흐름 + 식별자 + 멀티테넌시 먼저)
↓
6번 → 8번 → 9번               (빌링 흐름)
↓
11번 → 12번                   (이벤트 패턴)
↓
나머지는 해당 기능 구현할 때
```
