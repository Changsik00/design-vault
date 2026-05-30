---
type: core
aliases:
  - 요구사항 추적표
  - requirements
  - BDD 시나리오
tags:
  - platform-db
  - core
  - requirements
  - bdd
  - traceability
---

# platform_db — 요구사항 & 검증

> 작성일: 2026-05-28
> 진입점: [[architecture]] · 설계: [[schema-reference]]
>
> **이 문서**: "이 설계가 요구를 충족하는가"를 두 각도로 검증한다.  
> **A. 요구사항 추적표** — 8라운드 자문 총망라 (우리 구현 상태 기준 현실 평가)  
> **B. BDD 시나리오** — platform_db 6도메인 행위 시나리오 (아래로 내려갈수록 미구현)
>
> **상태 범례**  
> ✅ 구현완료 · 🟡 부분구현/P1 · ⚠️ phase-17 대상 · ⛔ 보류(YAGNI/트리거 미충족) · ❓ 미결정

---

# A. 요구사항 추적표

## A.1 아키텍처 / 토폴로지

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| ARCH-1 | 공통 코어(identity/billing/product)는 단일 `platform_db`, 도메인은 서비스별 DB | R0 | [[design-asymmetry]] | ✅ |
| ARCH-2 | strong-consistency(결제↔권한)는 단일 DB 트랜잭션, 2PC·Kafka 없음 | R0 | §6 단일 InnoDB 트랜잭션 | ✅ |
| ARCH-3 | cross-DB는 서비스→platform 읽기만 허용, peer 금지, least-privilege DB 계정 | R0 | 불변식 #6 | ✅ |
| ARCH-4 | `@aiagent/db-platform` 패키지로만 platform 접근, Drizzle 직접 참조 금지 | R0 | [[identity-billing-access]] (A) | ✅ |
| ARCH-5 | cross-schema FK 하드 금지(독립 백업/복원 보장) | R0 | 마이그레이션 규약 | ✅ |
| ARCH-6 | async 부수효과는 `outbox_event` | R0 | §6.2 | ✅ |

## A.2 Identity & User

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| USR-1 | 1 firebase_uid = 전 서비스 동일 사용자(SSOT) | R0 | identity_user 중앙화 | ✅ |
| USR-2 | 내부 PK BIGINT, 외부 노출 ULID(`public_id`) | R0 | 불변식 #2 | ✅ |
| USR-3 | HUMAN/SERVICE/SYSTEM 동일 모델 + type 구분 | R0 | identity_user.type | ✅ |
| USR-4 | 표시이름 SSOT는 `user_profile`, 서비스별 프로필은 서비스 DB | R0 | §3.1 | ✅ |
| USR-5 | email은 ACTIVE 중 unique(탈퇴 후 재사용 허용) | R2 | unique index (DDL 규약) | 🟡 P0 |
| USR-6 | 상태 ACTIVE/SUSPENDED/DELETED, soft→hard delete, anonymize | R0 | identity_user.status + deleted_at | ✅ |
| USR-7 | 1 user = N org (멀티 워크스페이스) | R0 | membership 복합 PK | ✅ |
| USR-8 | 소셜(Kakao/Naver) custom token + 다중 provider 계정연결 | R2 | custom token ✅ / 연결 모델 | 🟡 v1.0 |
| USR-10 | `email_verified` — Firebase JWT 동기화. 이메일 인증 필수 기능(결제·초대) 차단 근거 | 신규 | identity_user.email_verified + email_verified_at | 🟡 P1 |
| USR-11 | `phone_verified` — SMS OTP 또는 Firebase Phone Auth. 전화번호 수집 시 인증 여부 추적 | 신규 | identity_user.phone_verified + phone_verified_at | 🟡 P1 |
| USR-12 | 데이터 이식성 — 강사 본인 데이터(identity·profile·멤버십) admin 처리로 이전 가능. `platform.content_ownership` 동의 시 본인 생성 콘텐츠도 이전 가능. 학생·수업기록·결제이력은 org 소유로 이전 불가 | 신규 | user_consent_event(platform.content_ownership·platform.data_transfer) | 🟡 P1 |

## A.3 인증 (AuthN)

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| AUTHN-1 | Firebase Auth = 인증 SSOT, firebase_uid는 조회 키(PK/FK 아님) | R0 | 불변식 #1 | ✅ |
| AUTHN-2 | 비밀번호 강도·이메일 인증·재설정·토큰 TTL 1h | R0 | Firebase 정책 | ✅ |
| AUTHN-3 | 만료/위조 토큰 Guard 401 빠른 차단 | R2 | FirebaseAuthGuard | ✅ |
| AUTHN-4 | MFA (OWNER/관리자, v0.5) | R0 | Firebase MFA + platform_role 판단 | 🟡 |
| AUTHN-5 | B2B = `api_key`(prefix+secret_hash, scopes, IP, rotation, 즉시 revoke) | R0 | api_key 테이블 설계 확정 | ⚠️ phase-17 |
| AUTHN-6 | 계정 정지/삭제 시 전 서비스 즉시 차단 | R0 | identity_user.status | ✅ |
| AUTHN-7 | 다중 디바이스 허용, 이상 로그인 감지 | R0 | Firebase | ✅ |
| AUTHN-8 | JWT claim 신뢰 경계 — 필수 claim 누락 fail-closed + DB fallback | R6 | FirebaseAuthGuard 규약 | 🟡 |

## A.4 인가 — RBAC

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| RBAC-1 | role 2단: `membership.platform_role`(OWNER/MEMBER/SERVICE) + `service_membership.role_code` | R3 | D1, 0008 | ✅ |
| RBAC-2 | 서비스별 role 어휘 격리 (`academy.director`, `market.seller`…) | R3 | service_membership, 0008 | ✅ |
| RBAC-3 | role→action 매핑은 코드 상수(DB 저장 금지) | R0 | ROLE_PERMISSION | ✅ |
| RBAC-4 | 서비스 계정 = `platform_role='SERVICE'` + api_key (글로벌 AGENT role 폐기) | R3 | D4, 0008 | 🟡 platform_role='SERVICE' ✅ / api_key phase-17 |
| RBAC-5 | 역할 변경 즉시 반영(perm_version) | R0 | bumpPermVersion() | ✅ |
| RBAC-6 | 마지막 OWNER lockout 방지 | R2 | 앱 트랜잭션 가드 | 🟡 |
| RBAC-7 | DB role 레지스트리는 테넌트 커스텀롤 트리거 시 | R3 | P2 보류 | ⛔ |

## A.5 인가 — ABAC

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| ABAC-1 | 소유권(`owner_pk == principal`) 기반 통제 | R0 | CASL ability | ✅ |
| ABAC-2 | 테넌트 속성(`org_pk` 일치) 모든 도메인 강제 | R0 | 불변식 #3 | ✅ |
| ABAC-3 | feature_limit 한도 평가(entitlement), 카운터는 서비스측 | R2 | org_entitlement.feature_limits | ✅ |
| ABAC-4 | entitlement 상태 게이트(ACTIVE/GRACE/SUSPENDED/EXPIRED) | R0 | Gate B, 불변식 #4 | ✅ |
| ABAC-5 | 리소스 visibility 속성 필터 | R0 | CASL | ✅ |
| ABAC-6 | NIST 환경속성 — api_key `allowed_ip_cidr` + Gateway/WAF IP | R4 | D8 (P1) | 🟡 |
| ABAC-7 | 최종 `can()` 결정 캐싱 금지, 입력 블록만 TTL 60s | R0 | Redis 전략 | ✅ |

## A.6 인가 — ReBAC / 위임

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| REBAC-1 | capability 위임(grantor→grantee, scoped, expiry, revoke) | R0 | delegation_grant | ✅ |
| REBAC-2 | `capability` 서비스 네임스페이스 `<service>.<action>`(`ACADEMY.*`) | R3 | D2, 0008 | ✅ |
| REBAC-3 | 임퍼소네이션 금지, 위임 행사를 감사 기록 | R0 | audit_log | ✅ |
| REBAC-4 | org 계층(HQ_BRANCH/HOLDING) | R0 | org_relation | ✅ |
| REBAC-5 | 계층은 권한 근거 아님, 명시적 membership만 | R2 | 불변식 | ✅ |
| REBAC-6 | 자기참조 차단(DB CHECK constraint) | R2 | chk_no_self_ref | ✅ |
| REBAC-7 | Zanzibar full relation_tuple은 보류 | R0 | ⛔ 트리거 시 | ⛔ |
| REBAC-8 | delegation_grant(platform capability) ↔ 서비스 trust_relationship 경계 규칙 | R6 | academy_db 정리 필요 | 🟡 |

## A.7 Product / Billing / Entitlement

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| BILL-1 | cross-service 단일 상품 카탈로그(product/product_sku) | R0 | §3.2 | ✅ |
| BILL-2 | `org_entitlement` = 런타임 권위 (Gate B만, ledger 직접 조회 금지) | R0 | 불변식 #4 | ✅ |
| BILL-3 | 결제→entitlement 단일 트랜잭션 + perm_version + outbox | R0 | §6.1 | ✅ |
| BILL-4 | 멱등(webhook `event_id` UNIQUE, payment `idempotency_key` UNIQUE) | R0 | pg_webhook_event / payment_ledger | ✅ |
| BILL-5 | 환불/chargeback append-only, 금액은 정수 minor(float 금지) | R0 | payment_ledger | ✅ |
| BILL-6 | GRACE 유예기간, grace_until | R0 | org_entitlement.grace_until | ✅ |
| BILL-7 | 업그레이드(즉시)/다운그레이드(예약) | R2 | 🟡 P1 | 🟡 |
| BILL-8 | PG adapter 추상화(Toss/Stripe/PayPal/Manual) | R0 | pg_provider enum | ✅ |
| BILL-9 | PG webhook 서명 검증(handler, INSERT 전) + replay tolerance | R6 | signature_ok 컬럼 | 🟡 |
| BILL-10 | source=FREE/MANUAL/PROMO로 결제 없이 access 부여 | R0 | org_entitlement.source | ✅ |

## A.8 멀티테넌시 & 격리

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| TEN-1 | 모든 도메인 테이블 `org_pk NOT NULL` | R0 | 불변식 #3 | ✅ |
| TEN-2 | 모든 조회 `WHERE org_pk` 강제, 타 org는 404 | R1 | [[cross-tenant-separation]], BOLA 프레임워크 | ✅ |
| TEN-3 | cross-tenant 조회는 아키텍처 분리(`internal/`) | R1 | [[cross-tenant-separation]] | ✅ |
| TEN-4 | MySQL RLS 부재 → CI 린트 보강 | R1 | 🟡 린트 미도입 | 🟡 |
| TEN-5 | Qdrant payload 필터 + `org_id` 인덱스 | R1 | 🟡 | 🟡 |
| TEN-6 | Neo4j `org_id` 속성 + 멀티홉 경로 전체 강제 | R1 | 🟡 | 🟡 |
| TEN-7 | 분리 트리거 T1~T4 사전 정의 | R0 | §7 | ✅ |

## A.9 보안 표준

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| SEC-1 | BOLA 방어 — org_pk 질의 강제 프레임워크화 | R4 | Drizzle base repo | ✅ |
| SEC-2 | NIST 환경속성 — api_key IP/컨텍스트 | R4 | D8 (P1) | 🟡 |
| SEC-3 | 감사 불변성 — UPDATE/DELETE 권한 제거 + 해시 체이닝 + WORM | R4 | audit_log append-only ✅ / 해시 🟡 | 🟡 |
| SEC-4 | secret 관리 — KMS, 평문 로그 금지 | R0 | 정책 | ✅ |
| SEC-5 | api_key 하드닝(allowed_ip_cidr·rotated_at·revoked_reason) | R4 | D8 (phase-17) | ⚠️ |
| SEC-6 | secret rotation cadence — api_key 90/365d, Firebase Admin SDK 180d | R7 | 운영 플레이북 | 🟡 |

## A.10 정책 & 동의 (PIPA)

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| CON-1 | `user_consent_event` append-only 이벤트 | R5 | D7 | ⚠️ phase-17 |
| CON-2 | consent_type 네임스페이스(`platform.*`/`pg.*`) | R5 | user_consent_event 설계 | ⚠️ phase-17 |
| CON-3 | 14세 미만 법정대리인 동의(PIPA §22) | R5 | 🟡 P0 법적 필수 | ⚠️ phase-17 |
| CON-4 | 제3자 정보제공 4요건(PIPA §17) | R5 | 🟡 P0 법적 필수 | ⚠️ phase-17 |
| CON-5 | 마케팅 수신/거부(정보통신망법 §50) | R5 | 🟡 | ⚠️ phase-17 |
| CON-6 | 동의 철회권(PIPA §37) — REVOKED 이벤트 | R5 | 🟡 | ⚠️ phase-17 |
| CON-7 | 약관 버전 관리 + 재동의 인터셉터 | R5 | 🟡 P1 | 🟡 |
| CON-8 | `platform.content_ownership` — TEACHER role 취득 시 콘텐츠 소유권 약관 체크박스 동의 (전자서명법 §3 효력). 미동의 시 이전 불가 안내 | 신규 | user_consent_event | ⚠️ phase-17 |
| CON-9 | `platform.data_transfer` — 강사 이전 처리 전 강사 본인 동의 필수. admin이 동의 없이 임의 이전 금지 | 신규 | user_consent_event | ⚠️ phase-17 |
| CON-10 | `platform.withdrawal` — 탈퇴 최종 확인 동의 기록. 철회 불가 처리 전 명시적 동의 | 신규 | user_consent_event | ⚠️ phase-17 |

## A.11 감사 / 관측

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| AUD-1 | `audit_log` append-only, 월 파티셔닝, actor 통합 | R0 | §D.8 | ✅ |
| AUD-2 | 모든 권한 결정(ALLOW/DENY/ERROR) 기록 | R0 | audit_log.result | ✅ |
| AUD-3 | trace_id + audit_event_id(분산 추적) | R4 | 🟡 P1 | 🟡 |
| AUD-4 | 사람·머신 활동 통계 분리(type 필터) | R2 | identity_user.type | ✅ |

## A.12 비기능

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| NFR-1 | routine read 권한평가 DB hit 0(JWT claims), sensitive write만 DB 재검증 | R0 | VerifyOnDb 데코레이터 | ✅ |
| NFR-2 | perm_version 무효화 폭 분리(user/org) | R2 | bumpPermVersion | ✅ |
| NFR-3 | 확장 로드맵 — read replica → 테이블 분리 → DB-per-tenant | R0 | 분리 트리거 T1~T4 | ✅ |
| NFR-4 | permission_snapshot 프론트 read-model | R3 | GET /me/permissions | 🟡 |
| NFR-6 | perm_version 전파 — X-PV 헤더, 멀티인스턴스 sync | R6 | 🟡 P1 | 🟡 |

## A.13 운영 / 키관리 / 데이터 보호 / 경계

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| OPS-1 | **platform_db 비대화 방어** — 논리 바운디드 컨텍스트(모듈·마이그레이션 오너·절단선 FK 최소화) | R7 | §12.1 | ✅(원칙)/🟡(모듈 분리 구현) |
| OPS-2 | **Break-glass** 긴급 운영 접근 — 승인·사유·만료·전건 감사(임퍼소네이션과 별개) | R7 | §12.4 | 🟡 P1 |
| OPS-3 | service-to-service trust — HTTP 분리 시 mTLS/internal JWT, `internal/*` 내부망 전용 | R7 | Option B 전환 트리거 시 | ⛔ |
| OPS-4 | **secret rotation cadence** — api_key 90/365d, KMS DEK 연 1회, Firebase Admin SDK 키 180d 회전 (→ A.9 SEC-6 참조) | R6 | §12.3 | 🟡 정책 명시 P1 |
| SEC-7 | api_key 보강 — `last_used_ip`·`created_by_user_pk`·`rate_limit_tier`·`environment` | R7 | api_key 설계 | 🟡 P1 |
| SEC-8 | **데이터 분류**(PII/민감/미성년/결제) + **선별 app-level 암호화**(secret·guardian). phone/email 전면암호화 거부 | R7 | §9 | 🟡 분류 P0·암호화 P1 |
| BILL-12 | PG webhook **서명 검증**(handler, INSERT 전) + replay skew tolerance(±5분) | R6 | pg_webhook_event + 앱 규약 | 🟡 코드 규약 |
| TEN-9 | rate-limit 정책 위치 — org=`feature_limits` / 머신=`api_key.rate_limit_tier`, 강제는 Gateway | R6 | §4 결정 | 🟡 |
| USR-9 | 휴면(DORMANT) — **유효기간제 2023 폐지로 법적 의무 아님**, 선택적 제품정책 | R6 | §11 YAGNI | ⛔→🟡 제품 결정 |
| NFR-6 | perm_version **전파 프로토콜** — `/me/permissions {user_pv, org_pv}`, X-PV 헤더, 멀티인스턴스 sync | R6 | §5.4 설계 / 🟡 멀티인스턴스 P1 | ✅(설계)/🟡(P1) |
| RBAC-8 | ROLE_PERMISSION 변경 **배포 SLA 명시** — hot-fix vs weekly 정책 | R6 | §15 열린 결정 | ❓ |
| REBAC-8 | `delegation_grant`(platform capability) ↔ 서비스 `trust_relationship`(도메인) **경계 규칙** | R6 | 아키텍처 규약 | ✅(규칙)/🟡(구현) |
| AUTHN-8 | JWT claim 신뢰 경계 — 필수 claim 누락 **fail-closed** + DB fallback | R6 | FirebaseAuthGuard 규약 | 🟡 |
| CON-12 | **fan-out anonymize** — 탈퇴 시 outbox `user.deleted` → 각 서비스 dangling anonymize | R6 | §12.6 | 🟡 P1 |
| CON-13 | 동의 `meta_json` **canonical(RFC 8785 JCS) + JSON Schema 검증** | R6 | user_consent_event 설계 | 🟡 P1 |

## A.14 충족 스코어카드 (우리 현재 기준)

| 도메인 | 항목 수 | ✅ | 🟡 | ⚠️ | ⛔ | ❓ |
|---|---|---|---|---|---|---|
| 아키텍처 | 6 | 6 | — | — | — | — |
| Identity & User | 11 | 6 | 5 | — | — | — |
| 인증 | 8 | 6 | 2 | — | — | — |
| RBAC | 7 | 4 | 2 | 0 | 1 | — |
| ABAC | 7 | 6 | 1 | — | — | — |
| ReBAC | 8 | 6 | 1 | 0 | 1 | — |
| Billing | 10 | 8 | 2 | — | — | — |
| 멀티테넌시 | 7 | 4 | 3 | — | — | — |
| 보안 | 6 | 2 | 3 | 1 | — | — |
| 동의(PIPA) | 10 | — | 1 | 9 | — | — |
| 감사 | 4 | 3 | 1 | — | — | — |
| 비기능 | 5 | 3 | 2 | — | — | — |
| 운영/키관리/경계 | 15 | 2 | 10 | — | 2 | 1 |
| **합계** | **104** | **56(54%)** | **33(32%)** | **10(10%)** | **4(4%)** | **1(1%)** |

> **2026-05-30 갱신**: 0008 마이그레이션(RBAC-1/2, REBAC-2 구현) 반영 — ⚠️ 13→10.
> **phase-17 완료 목표**: ✅ 80%+, ⚠️ 0%

---

# B. BDD 시나리오

→ [[bdd-scenarios]] 로 분리됨 (2026-05-30).
platform_db 10개 도메인 행위 시나리오(Identity·Membership·Delegation·Billing·감사·Consent·멀티테넌시·perm_version/Break-glass·데이터 이전·인증) + academy 수용성 크로스체크를 담는다.

