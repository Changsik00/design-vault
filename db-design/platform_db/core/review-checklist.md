---
type: core
aliases:
  - 설계 검토 체크리스트
  - review checklist
tags:
  - platform-db
  - core
  - review
  - checklist
---

# platform_db 설계 검토 체크리스트

> 생성: 2026-05-28  
> 기준: Opus Agent A (보안/아키텍처, 72/100) + Agent B (정규화/빌링/운영, 58/100) + 문서 정합성 교차검토  
> 상태 범례: 🔴 미해결 · 🟡 문서결정필요 · ✅ 완료

---

## P0 — 법적 위험 (즉시, 현재 진행형 위반)

| # | 항목 | 상태 | 연관 |
|---|------|------|------|
| P0-1 | **CON-3/4/6 미구현** — 14세 미만 법정대리인 동의·제3자 제공 동의·동의 철회 기능 구현 0% | 🔴 | PIPA §22·§17·§37; [[requirements]] A.10 |
| P0-2 | **JWT stale 1h 중 동의 철회 후 마케팅 발송 허용** — 철회 즉시 차단 메커니즘 없음 | 🔴 | PIPA §37; [[architecture]] §5.2 |
| P0-3 | **requirements §14 준거표 "✅ P0 완료" ↔ A.10/A.14 "⚠️ phase-17, 0% 구현" 직접 모순** — 감사자 오독 리스크 | ✅ §14 표를 "설계 상태"+"구현 상태" 이중 컬럼으로 분리, ⚠️ phase-17 명기 | [[architecture]] §14 |

---

## P1 — 설계 구조 결함 (코딩 착수 전 문서 확정 필요)

| # | 항목 | 상태 | 연관 |
|---|------|------|------|
| P1-1 | **billing 모델 구조 모순** — org_subscription.sku_pk(단일 FK) vs subscription_item(복수 N:M) — 복합 상품 과금 불가 | ✅ subscription_item N:M 확정, sku_pk 제거 | [[schema-reference]] §C; [[requirements]] §8 |
| P1-2 | **feature limits 3중 정의, 우선순위 없음** — product_feature / plan_definition.default_limits / org_entitlement.feature_limits | ✅ org_entitlement 최종 권위 확정(불변식 #10) | [[schema-reference]] §B, §C, §E |
| P1-3 | **GRACE 이중 관리 모순** — org_subscription.grace_until ↔ org_entitlement.grace_until, Gate B 판정 기준 미정의 | ✅ org_entitlement.grace_until이 Gate B 기준 | [[architecture]] §5.4; [[schema-reference]] E.2 |
| P1-4 | **Gate B vs valid_until 관계 미정의** — status만 보면 배치 실패 시 영구 무료, valid_until 실시간 검사하면 배치 불필요 | ✅ status + valid_until 복합 체크 확정(불변식 #9) | [[architecture]] §5.4; [[schema-reference]] E.2 |
| P1-5 | **audit_log PK DDL 결함** — AUTO_INCREMENT 없음, 파티션 복합 PK 없음 → INSERT 불가 | ✅ AUTO_INCREMENT 추가, PRIMARY KEY(pk, created_at) | [[schema-reference]] §F |
| P1-6 | **`pg_provider`·`billing_event.event_type` ENUM — D6 원칙 미일관 적용** — service를 VARCHAR+CHECK로 바꾼 동일 논리를 billing 4개 테이블에 미적용. PG 추가 시 대형 테이블 MODIFY COLUMN 잠금 발생 (R8 자문) | 🔴 phase-17+ 마이그레이션 대상 추가 | [[architecture]] §4 D6 노트; [[schema-reference]] D.16/17/18 |

---

## P2 — 운영 자동화 부재 (출시 전 필수)

| # | 항목 | 상태 | 연관 |
|---|------|------|------|
| P2-1 | **TRIALING→EXPIRED 자동 배치 미설계** — requirements §12.11 비어있음 → 무한 TRIALING 가능 | ✅ §12.11 테이블에 TRIALING→EXPIRED 전환 행 추가 | [[architecture]] §12.11 |
| P2-2 | **org_entitlement 만료 인덱스 없음** — WHERE valid_until < NOW() AND status='ACTIVE' 풀스캔 | ✅ `INDEX idx_entitlement_expiry (valid_until, status)` 추가 | [[schema-reference]] §E DDL |
| P2-3 | **audit_log 파티션 자동 추가 없음** — p_future 단일 파티션 3개월 후 과부하 | 🔴 | [[schema-reference]] §F |
| P2-4 | **§12.5 "월 1회 해시 재계산" 플레이북 ↔ audit_log hash 컬럼 부재 모순** — 실행 불가한 절차 기재 | ✅ §12.5에 "P1 미구현" 경고 추가; user_consent_event에 prev_hash/row_hash 컬럼 추가(DDL) | [[architecture]] §12.5; [[schema-reference]] §I.2 |
| P2-5 | **`org_subscription.external_sub_id` 인덱스 없음** — PG webhook이 external_sub_id로 구독 조회 시 풀스캔 (R8 자문) | ✅ `INDEX idx_org_subscription_external_sub_id (external_sub_id)` 추가 | [[schema-reference]] D.13 |
| P2-6 | **`pg_webhook_event.status` 인덱스 없음** — 재처리 워커 WHERE status='FAILED' 풀스캔. outbox_event와 비대칭 (R8 자문) | ✅ `INDEX idx_pg_webhook_status (status, created_at)` 추가 | [[schema-reference]] D.18 |
| P2-7 | **Gate B 핫패스 인덱스 valid_until 누락** — 모든 요청마다 도는 Gate B 지연에 직접 영향. E.2에 "추가 필요" 명시됐으나 DDL 미수정 (R8 자문 확인) | ✅ `idx_org_service_status (org_pk, service, status, valid_until)` DDL 수정 | [[schema-reference]] D.12, E.2 |
| P2-8 | **PG webhook 완전 소실 시 복구 수단 없음** — webhook이 배포 중 유실되거나 PG 재시도가 모두 소진되면 `pg_webhook_event`에 기록 자체가 없어 재처리 워커도 무력. PG API 대사 조회(reconciliation) 잡 미설계 | 🔴 `GET /subscriptions/{id}` 주기적 폴링으로 DB 상태와 대조하는 잡 필요 | [[architecture]] §5; [[schema-reference]] D.13 D.18 |
| P2-9 | **배포 중 in-flight webhook 유실 위험** — K8s rolling update 시 `terminationGracePeriodSeconds` 부족하면 처리 중 트랜잭션이 강제 종료됨. 트랜잭션 롤백 → PG non-200 수신 → PG 재시도로 대부분 복구되지만, LB 해제 타이밍 갭에서 수신 후 응답 전 kill 시 PG가 성공으로 오인하고 재시도 안 할 수 있음. graceful shutdown 설정 전략 미문서화 | 🔴 K8s `preStop` + `terminationGracePeriodSeconds` 설정 명문화, 근본 해결은 MQ 도입 검토 (P2-10 참조) | [[architecture]] §17 배포 SLA |
| P2-10 | **outbox_event · pg_webhook_event 데이터 무한 누적** — PROCESSED/SKIPPED 레코드 삭제 전략 없음. outbox_event는 모든 트랜잭션마다 1행씩 생성되어 가장 빠르게 증가 | 🔴 sweeper 잡 설계 필요 (예: PROCESSED + 30일 경과 → DELETE) | [[schema-reference]] D.19 D.18 |

---

## P3 — 보안 갭 (출시 전 필수)

| # | 항목 | 상태 | 연관 |
|---|------|------|------|
| P3-1 | **OWNER lockout 가드 완전 부재** — 유일 OWNER 탈퇴 시 org 좀비화, BDD만 있고 DB constraint/앱 가드 없음 | 🔴 | [[requirements]] P9-04 |
| P3-2 | **break_glass 컬럼 없음** — meta_json 풀스캔, 자기승인 차단 불가 | ✅ audit_log DDL에 `break_glass BOOLEAN NOT NULL DEFAULT FALSE` + 인덱스 추가 | [[schema-reference]] §D.8 |
| P3-3 | **Qdrant org_id(ULID) vs MySQL org_pk(BIGINT) 식별자 불일치** — 매핑 검증 없음, 다른 org 데이터 노출 가능 | 🔴 | [[architecture]] §2; [[design-asymmetry]] |
| P3-4 | **delegation_grant capability CHECK 6종 하드코딩** — 두 번째 서비스 추가 시 즉시 마이그레이션 블로킹 (G2 aspirational) | 🔴 | [[architecture]] §3.2 G2 |
| P3-5 | **user_consent_event DDL에 PIPA §17 4요건 컬럼 없음** — meta_json, prev_hash, row_hash 미정의, 동의 무결성 증명 불가 | ✅ DDL에 `meta_json JSON`, `prev_hash CHAR(64)`, `row_hash CHAR(64)` 추가 (P1 활성화 예정 명기) | [[schema-reference]] §I.2 |

---

## P4 — 사업 확장 필수 (MVP 이후 1분기)

| # | 항목 | 상태 |
|---|------|------|
| P4-1 | price_krw → multi-currency (price_minor + currency 분리) | 🔴 |
| P4-2 | VAT/세금 컬럼·테이블 없음 → 한국 B2B 전자세금계산서 불가 | 🔴 |
| P4-3 | coupon/promotion 테이블 없음 (source='PROMO' 태깅만으로 추적 불가) | 🔴 |
| P4-4 | invoice/invoice_line 테이블 없음 | 🔴 |
| P4-5 | proration 단가 컬럼 없음 | 🔴 |
| P4-6 | payment_ledger 부분환불 self-referencing FK 없음 | 🔴 |
| P4-7 | subscription CANCELED → 기간 만료까지 서비스 유지 전이 미설계 | 🔴 |
| P4-8 | billing 도메인 FK 전무 — **의도적 설계 결정으로 명시 완료** (고write·append-only 패턴, FK 잠금 회피). D.13/16/17/18 설계 포인트에 명기 (R8 자문 반영) | 🟡 의도 명시됨 |
| P4-9 | **`user_consent_event` 파티셔닝 미설계** — 5년 보존 후 파기 요건. audit_log 동일 월별 RANGE 파티션 패턴 적용 검토 (R8 자문) | 🔴 P2 검토 대상 |

---

## P5 — 문서 정합성 (코드 무관, 문서 편집만)

| # | 항목 | 상태 | 대상 파일 |
|---|------|------|-----------|
| P5-1 | requirements.md 불변식 번호 일괄 불일치 — §3 재구성으로 #4→#3, #5→#4, #8→#6 등 | ✅ | [[requirements]] |
| P5-2 | schema-reference.md D번호 갭 — D.16→D.19, D.17/D.18 없음 | ✅ | [[schema-reference]] |
| P5-3 | consent_type namespace 불일치 — BDD I.3/P6은 `TERMS_OF_SERVICE`, §I.2는 `platform.terms_of_service` | ✅ | [[requirements]], [[schema-reference]] |
| P5-4 | A.14 scorecard 합산 오류 + 합계 불일치 | ✅ | [[requirements]] |
| P5-5 | SEC-6 ID 중복 — A.9와 A.13에 동일 ID 중복 기재 | ✅ A.13 → OPS-4 변경 | [[requirements]] |
| P5-6 | checkGateB 설명 stale — arch §5.4, schema E.2에 service 파라미터 반영 안 됨 | ✅ | [[architecture]], [[schema-reference]] |
| P5-7 | D-table 구현 상태 컬럼 없음 — D1/D2/D5가 "결정"으로 단정 기재, 실제 aspirational | ✅ 구현상태 컬럼 추가 | [[architecture]] |
| P5-8 | **`membership` PK 선언 오류** — `UNIQUE KEY pk_membership` 대신 `PRIMARY KEY` 선언이 의미상 올바름 (R8 자문) | ✅ `PRIMARY KEY (user_pk, org_pk)`로 변경 | [[schema-reference]] D.4 |

---

## 진행 현황

| 등급 | 전체 | 완료 | 잔여 |
|------|------|------|------|
| P0 | 3 | **1** | 2 |
| P1 | 6 | **5** | 1 |
| P2 | 11 | **6** | 5 |
| P3 | 5 | **2** | 3 |
| P4 | 9 | **1** | 8 |
| P5 | 8 | **8** | 0 |
| **합계** | **42** | **23** | **19** |

> R8 자문(2026-05-28) 반영: +6개 항목 추가, 5개 즉시 완료(DDL 수정), 1개 진행 중(P1-6 ENUM 마이그레이션)

---

## 참고: 에이전트 점수

| 에이전트 | 평가 영역 | 점수 |
|----------|-----------|------|
| Agent A (Opus) | 보안·격리·아키텍처 | 72/100 |
| Agent B (Opus) | 정규화·빌링·운영 | 58/100 |
| 통합 평가 | 전체 | **65/100** |
