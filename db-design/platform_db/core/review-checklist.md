# platform_db 설계 검토 체크리스트

> 생성: 2026-05-28  
> 기준: Opus Agent A (보안/아키텍처, 72/100) + Agent B (정규화/빌링/운영, 58/100) + 문서 정합성 교차검토  
> 상태 범례: 🔴 미해결 · 🟡 문서결정필요 · ✅ 완료

---

## P0 — 법적 위험 (즉시, 현재 진행형 위반)

| # | 항목 | 상태 | 연관 |
|---|------|------|------|
| P0-1 | **CON-3/4/6 미구현** — 14세 미만 법정대리인 동의·제3자 제공 동의·동의 철회 기능 구현 0% | 🔴 | PIPA §22·§17·§37; requirements A.10 |
| P0-2 | **JWT stale 1h 중 동의 철회 후 마케팅 발송 허용** — 철회 즉시 차단 메커니즘 없음 | 🔴 | PIPA §37; architecture §5.2 |
| P0-3 | **requirements §14 준거표 "✅ P0 완료" ↔ A.10/A.14 "⚠️ phase-17, 0% 구현" 직접 모순** — 감사자 오독 리스크 | ✅ §14 표를 "설계 상태"+"구현 상태" 이중 컬럼으로 분리, ⚠️ phase-17 명기 | architecture.md §14 |

---

## P1 — 설계 구조 결함 (코딩 착수 전 문서 확정 필요)

| # | 항목 | 상태 | 연관 |
|---|------|------|------|
| P1-1 | **billing 모델 구조 모순** — org_subscription.sku_pk(단일 FK) vs subscription_item(복수 N:M) — 복합 상품 과금 불가 | ✅ subscription_item N:M 확정, sku_pk 제거 | schema-reference §C; requirements §8 |
| P1-2 | **feature limits 3중 정의, 우선순위 없음** — product_feature / plan_definition.default_limits / org_entitlement.feature_limits | ✅ org_entitlement 최종 권위 확정(불변식 #10) | schema-reference §B, §C, §E |
| P1-3 | **GRACE 이중 관리 모순** — org_subscription.grace_until ↔ org_entitlement.grace_until, Gate B 판정 기준 미정의 | ✅ org_entitlement.grace_until이 Gate B 기준 | architecture §5.4; schema-reference E.2 |
| P1-4 | **Gate B vs valid_until 관계 미정의** — status만 보면 배치 실패 시 영구 무료, valid_until 실시간 검사하면 배치 불필요 | ✅ status + valid_until 복합 체크 확정(불변식 #9) | architecture §5.4; schema-reference E.2 |
| P1-5 | **audit_log PK DDL 결함** — AUTO_INCREMENT 없음, 파티션 복합 PK 없음 → INSERT 불가 | ✅ AUTO_INCREMENT 추가, PRIMARY KEY(pk, created_at) | schema-reference §F |

---

## P2 — 운영 자동화 부재 (출시 전 필수)

| # | 항목 | 상태 | 연관 |
|---|------|------|------|
| P2-1 | **TRIALING→EXPIRED 자동 배치 미설계** — requirements §12.11 비어있음 → 무한 TRIALING 가능 | ✅ §12.11 테이블에 TRIALING→EXPIRED 전환 행 추가 | architecture.md §12.11 |
| P2-2 | **org_entitlement 만료 인덱스 없음** — WHERE valid_until < NOW() AND status='ACTIVE' 풀스캔 | ✅ `INDEX idx_entitlement_expiry (valid_until, status)` 추가 | schema-reference §E DDL |
| P2-3 | **audit_log 파티션 자동 추가 없음** — p_future 단일 파티션 3개월 후 과부하 | 🔴 | schema-reference §F |
| P2-4 | **§12.5 "월 1회 해시 재계산" 플레이북 ↔ audit_log hash 컬럼 부재 모순** — 실행 불가한 절차 기재 | ✅ §12.5에 "P1 미구현" 경고 추가; user_consent_event에 prev_hash/row_hash 컬럼 추가(DDL) | architecture.md §12.5; schema-reference §I.2 |

---

## P3 — 보안 갭 (출시 전 필수)

| # | 항목 | 상태 | 연관 |
|---|------|------|------|
| P3-1 | **OWNER lockout 가드 완전 부재** — 유일 OWNER 탈퇴 시 org 좀비화, BDD만 있고 DB constraint/앱 가드 없음 | 🔴 | requirements P9-04 |
| P3-2 | **break_glass 컬럼 없음** — meta_json 풀스캔, 자기승인 차단 불가 | ✅ audit_log DDL에 `break_glass BOOLEAN NOT NULL DEFAULT FALSE` + 인덱스 추가 | schema-reference §D.8 |
| P3-3 | **Qdrant org_id(ULID) vs MySQL org_pk(BIGINT) 식별자 불일치** — 매핑 검증 없음, 다른 org 데이터 노출 가능 | 🔴 | architecture §2 ADR-044 |
| P3-4 | **delegation_grant capability CHECK 6종 하드코딩** — 두 번째 서비스 추가 시 즉시 마이그레이션 블로킹 (G2 aspirational) | 🔴 | architecture §3.2 G2 |
| P3-5 | **user_consent_event DDL에 PIPA §17 4요건 컬럼 없음** — meta_json, prev_hash, row_hash 미정의, 동의 무결성 증명 불가 | ✅ DDL에 `meta_json JSON`, `prev_hash CHAR(64)`, `row_hash CHAR(64)` 추가 (P1 활성화 예정 명기) | schema-reference §I.2 |

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
| P4-8 | billing 도메인 FK 전무 (schema-reference C.3 표 vs 실제 DDL 불일치) | 🔴 |

---

## P5 — 문서 정합성 (코드 무관, 문서 편집만)

| # | 항목 | 상태 | 대상 파일 |
|---|------|------|-----------|
| P5-1 | requirements.md 불변식 번호 일괄 불일치 — §3 재구성으로 #4→#3, #5→#4, #8→#6 등 | ✅ | requirements.md |
| P5-2 | schema-reference.md D번호 갭 — D.16→D.19, D.17/D.18 없음 | ✅ | schema-reference.md |
| P5-3 | consent_type namespace 불일치 — BDD I.3/P6은 `TERMS_OF_SERVICE`, §I.2는 `platform.terms_of_service` | ✅ | requirements.md, schema-reference.md |
| P5-4 | A.14 scorecard 합산 오류 + 합계 불일치 | ✅ | requirements.md |
| P5-5 | SEC-6 ID 중복 — A.9와 A.13에 동일 ID 중복 기재 | ✅ A.13 → OPS-4 변경 | requirements.md |
| P5-6 | checkGateB 설명 stale — arch §5.4, schema E.2에 service 파라미터 반영 안 됨 | ✅ | architecture.md, schema-reference.md |
| P5-7 | D-table 구현 상태 컬럼 없음 — D1/D2/D5가 "결정"으로 단정 기재, 실제 aspirational | ✅ 구현상태 컬럼 추가 | architecture.md |

---

## 진행 현황

| 등급 | 전체 | 완료 | 잔여 |
|------|------|------|------|
| P0 | 3 | **1** | 2 |
| P1 | 5 | **5** | 0 |
| P2 | 4 | **3** | 1 |
| P3 | 5 | **2** | 3 |
| P4 | 8 | 0 | 8 |
| P5 | 7 | **7** | 0 |
| **합계** | **32** | **18** | **14** |

---

## 참고: 에이전트 점수

| 에이전트 | 평가 영역 | 점수 |
|----------|-----------|------|
| Agent A (Opus) | 보안·격리·아키텍처 | 72/100 |
| Agent B (Opus) | 정규화·빌링·운영 | 58/100 |
| 통합 평가 | 전체 | **65/100** |
