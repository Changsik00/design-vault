# platform_db — 요구사항 & 검증

> 작성일: 2026-05-28 · 유지: changsik  
> 진입점: [`architecture.md`](architecture.md) · 설계: [`schema-reference.md`](schema-reference.md)
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
| ARCH-1 | 공통 코어(identity/billing/product)는 단일 `platform_db`, 도메인은 서비스별 DB | R0 | ADR-044 | ✅ |
| ARCH-2 | strong-consistency(결제↔권한)는 단일 DB 트랜잭션, 2PC·Kafka 없음 | R0 | §6 단일 InnoDB 트랜잭션 | ✅ |
| ARCH-3 | cross-DB는 서비스→platform 읽기만 허용, peer 금지, least-privilege DB 계정 | R0 | 불변식 #6 | ✅ |
| ARCH-4 | `@aiagent/db-platform` 패키지로만 platform 접근, Drizzle 직접 참조 금지 | R0 | ADR-032 Option A | ✅ |
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
| RBAC-1 | role 2단: `membership.platform_role`(OWNER/ADMIN/MEMBER/SERVICE) + `service_membership.role_code` | R3 | D1 | ⚠️ phase-17 |
| RBAC-2 | 서비스별 role 어휘 격리 (`academy.director`, `market.seller`…) | R3 | service_membership | ⚠️ phase-17 |
| RBAC-3 | role→action 매핑은 코드 상수(DB 저장 금지) | R0 | ROLE_PERMISSION | ✅ |
| RBAC-4 | 서비스 계정 = `platform_role='SERVICE'` + api_key (글로벌 AGENT role 폐기) | R3 | D4 | ⚠️ phase-17 |
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
| REBAC-2 | `capability_code` 서비스 네임스페이스 `<service>.<action>` | R3 | D2 | ⚠️ phase-17 |
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
| TEN-2 | 모든 조회 `WHERE org_pk` 강제, 타 org는 404 | R1 | ADR-042, BOLA 프레임워크 | ✅ |
| TEN-3 | cross-tenant 조회는 아키텍처 분리(`internal/`) | R1 | ADR-042 | ✅ |
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
| RBAC | 7 | 3 | 1 | 2 | 1 | — |
| ABAC | 7 | 6 | 1 | — | — | — |
| ReBAC | 8 | 5 | 1 | 1 | 1 | — |
| Billing | 10 | 8 | 2 | — | — | — |
| 멀티테넌시 | 7 | 4 | 3 | — | — | — |
| 보안 | 6 | 2 | 3 | 1 | — | — |
| 동의(PIPA) | 10 | — | 1 | 9 | — | — |
| 감사 | 4 | 3 | 1 | — | — | — |
| 비기능 | 5 | 3 | 2 | — | — | — |
| 운영/키관리/경계 | 15 | 2 | 10 | — | 2 | 1 |
| **합계** | **104** | **54(52%)** | **32(31%)** | **13(12%)** | **4(4%)** | **1(1%)** |

> **phase-17 완료 목표**: ✅ 80%+, ⚠️ 0%

---

# B. BDD 시나리오 (platform_db 도메인)

> platform_db 계층에서 직접 검증 가능한 시나리오.  
> academy 서비스 시나리오(F1~F60)는 [`docs/academy/v3/bdd-scenarios.md`](../academy/v3/bdd-scenarios.md) 참조.

---

## Domain 1: Identity — 조직 & 사용자

### P1-01: 신규 학원 onboarding (단일 트랜잭션)

```gherkin
Scenario: 신규 학원장 가입 시 platform_db 원자적 생성
  Given identity_user에 firebase_uid='fb_new_001'인 row가 없다
  And organization에 slug='hanwool'인 row가 없다

  When 가입 API가 단일 트랜잭션으로 실행된다

  Then 다음 row들이 동일 트랜잭션 내 INSERT된다:
    | 테이블           | 주요 값                                      |
    | identity_user    | firebase_uid='fb_new_001', type='HUMAN', status='ACTIVE' |
    | user_profile     | display_name='김지영', locale='ko'          |
    | organization     | slug='hanwool', type='ACADEMY', status='ACTIVE' |
    | membership       | role='OWNER', status='ACTIVE'               |
    | org_entitlement  | service='ACADEMY', status='ACTIVE', source='FREE' |
  And organization.perm_version=1
  And identity_user.perm_version=1
```

### P1-02: 멀티 워크스페이스 — 1 user, N org

```gherkin
Scenario: 기존 사용자가 두 번째 학원에 초대 수락
  Given identity_user.pk=10 (최정훈)이 org_pk=1의 TEACHER다
  And membership_invite에 token='abc123', org_pk=2, status='PENDING'이 존재한다
  And expires_at이 아직 유효하다

  When acceptMembershipInvite(token='abc123', userPk=10)가 호출된다

  Then membership에 (user_pk=10, org_pk=2, role='TEACHER', status='ACTIVE') INSERT
  And membership_invite.status='ACCEPTED' UPDATE
  And identity_user에 새 row가 생성되지 않는다 (기존 user_pk 재사용)
```

### P1-03: 만료된 초대 토큰 차단

```gherkin
Scenario: expires_at 지난 토큰으로 수락 시도
  Given membership_invite.expires_at이 1시간 전에 지났다

  When acceptMembershipInvite(token)가 호출된다

  Then ForbiddenException 발생
  And membership_invite.status가 'EXPIRED'로 UPDATE된다
  And membership에 새 row가 INSERT되지 않는다
```

---

## Domain 2: Membership & 권한 동기화

### P2-01: 멤버십 회수 → perm_version bump

```gherkin
Scenario: DIRECTOR가 TEACHER 멤버십을 정지시킨다
  Given membership (user_pk=20, org_pk=1, status='ACTIVE')가 존재한다
  And organization.perm_version=5다

  When membership.status='SUSPENDED' UPDATE + bumpPermVersion(orgPk=1)이 호출된다

  Then membership.status='SUSPENDED'
  And organization.perm_version=6
  And audit_log에 (actor_pk=director_pk, action='revoke_membership', result='ALLOW') INSERT
```

### P2-02: Gate A — 정지된 멤버십 차단

```gherkin
Scenario: SUSPENDED 멤버의 API 호출
  Given membership (user_pk=20, org_pk=1, status='SUSPENDED')

  When checkGateA(userPk=20, orgPk=1)가 호출된다

  Then false 반환 (Gate A 실패)
  And 이 결과로 API 계층에서 403 Forbidden 응답
```

### P2-03: 역할 변경 → perm_version bump

```gherkin
Scenario: TEACHER → DIRECTOR 역할 승격
  Given membership (user_pk=30, org_pk=1, role='TEACHER')
  And organization.perm_version=7

  When membership.role='DIRECTOR' UPDATE + bumpPermVersion(orgPk=1) 단일 트랜잭션

  Then membership.role='DIRECTOR'
  And organization.perm_version=8
```

---

## Domain 3: Delegation Grant (ReBAC)

### P3-01: capability 위임 — 허용값 CHECK constraint

```gherkin
Scenario: 허용된 capability로 delegation_grant INSERT
  Given DIRECTOR (grantor_pk=10)가 TEACHER (grantee_pk=20)에게 권한 위임

  When delegation_grant INSERT:
    capability='PUBLISH_VIDEO', org_pk=1, status='ACTIVE'

  Then INSERT 성공
  And delegation_grant.status='ACTIVE'

Scenario: 허용되지 않은 capability 거부
  When delegation_grant INSERT: capability='ADMIN_ALL'

  Then MySQL CHECK constraint 위반으로 INSERT 실패
  And delegation_grant row가 생성되지 않는다
```

### P3-02: 위임 회수 → DB 재검증에서 차단

```gherkin
Scenario: REVOKE 후 stale JWT로 sensitive write 시도
  Given delegation_grant (grantee_pk=20, capability='PUBLISH_VIDEO', status='ACTIVE')였다
  And 방금 status='REVOKED'로 UPDATE됐다
  And grantee의 JWT claims는 아직 stale(TTL 내)

  When grantee가 sensitive write API(@VerifyOnDb)를 호출한다

  Then DB에서 delegation_grant.status='REVOKED' 재확인
  And 403 Forbidden 반환
  And audit_log에 (result='DENY', action='publish_video') INSERT
```

### P3-03: expires_at 지난 grant 자동 무력화

```gherkin
Scenario: 만료 기간 지난 delegation_grant
  Given delegation_grant.expires_at=어제, status='ACTIVE'

  When getDelegationGrants(granteePk=20, orgPk=1)가 호출된다

  Then expires_at < NOW() 인 grant는 결과에서 제외된다
  And 비즈니스 로직에서 해당 capability가 없는 것으로 평가된다
```

---

## Domain 4: Billing — 결제 & 권한

### P4-01: 구독 결제 → 단일 트랜잭션 완결

```gherkin
Scenario: TOSS 결제 완료 webhook 처리 → org_entitlement 활성화
  Given org_subscription.status='TRIALING', org_entitlement.status='ACTIVE' (FREE)
  And pg_webhook_event에 (pg_provider='TOSS', event_id='evt_001')이 없다

  When 결제 성공 처리 트랜잭션이 실행된다

  Then 단일 트랜잭션 내에서:
    | 처리                  | 내용 |
    | payment_ledger INSERT | type='CHARGE', amount_minor=990000, idempotency_key='inv_001', status='SUCCEEDED' |
    | org_subscription UPDATE | status='ACTIVE', current_period_end=+30일 |
    | org_entitlement UPSERT | status='ACTIVE', source='SUBSCRIPTION', feature_limits={...} |
    | organization UPDATE   | perm_version=perm_version+1 |
    | outbox_event INSERT   | event_type='subscription.activated' |
  And pg_webhook_event INSERT: (pg_provider='TOSS', event_id='evt_001', status='PROCESSED')
```

### P4-02: 중복 webhook 멱등 처리

```gherkin
Scenario: 동일 webhook event_id 재수신
  Given pg_webhook_event에 (pg_provider='TOSS', event_id='evt_001', status='PROCESSED')가 이미 존재

  When 동일 event_id의 webhook이 다시 수신된다

  Then pg_webhook_event INSERT가 UNIQUE constraint 위반으로 실패 (또는 SKIPPED 처리)
  And payment_ledger에 중복 INSERT 없음
  And org_entitlement 중복 변경 없음
```

### P4-03: Gate B — EXPIRED entitlement 차단

```gherkin
Scenario: 만료된 org_entitlement로 서비스 접근 시도
  Given org_entitlement (org_pk=1, service='ACADEMY', status='EXPIRED')

  When checkGateB(orgPk=1, service='ACADEMY')가 호출된다

  Then false 반환 (Gate B 실패)
  And 402 Payment Required 반환
```

### P4-04: GRACE 기간 중 서비스 유지

```gherkin
Scenario: 결제 실패 후 grace 기간 내 접근
  Given org_entitlement (status='GRACE', grace_until=내일)

  When checkGateB(orgPk=1, service='ACADEMY')가 호출된다

  Then true 반환 (Gate B 통과 — GRACE도 허용)
  And 학원장 화면에 "결제 실패 배너" 표시 (앱 레이어 처리)
```

### P4-05: 환불 — append-only 원장

```gherkin
Scenario: 결제 취소 → REFUND 기록
  Given payment_ledger에 (type='CHARGE', amount_minor=990000, status='SUCCEEDED') row 존재

  When 환불 처리가 실행된다

  Then payment_ledger에 신규 row INSERT:
    type='REFUND', amount_minor=-990000, idempotency_key='ref_001', status='SUCCEEDED'
  And 기존 CHARGE row는 수정되지 않는다 (append-only)
```

---

## Domain 5: 감사 로그

### P5-01: 모든 권한 결정 기록

```gherkin
Scenario: ALLOW 결정 감사 기록
  Given Gate A/B/C 모두 통과한 API 호출

  Then audit_log에 INSERT:
    actor_type='HUMAN', actor_pk=user_pk, org_pk=1,
    action='publish_video', result='ALLOW', ip=<클라이언트 IP>

Scenario: DENY 결정 감사 기록
  Given Gate A 실패(membership SUSPENDED)

  Then audit_log에 INSERT:
    result='DENY', action='publish_video', ip=<클라이언트 IP>
  And 403 Forbidden 반환
```

### P5-02: audit_log 불변성 — 수정 불가

```gherkin
Scenario: audit_log row UPDATE 시도
  Given audit_log.pk=9999 row 존재

  When UPDATE audit_log SET result='ALLOW' WHERE pk=9999 실행

  Then 앱 레이어에서 UPDATE 차단 (DB 계정 권한 없음)
  And row가 변경되지 않는다
```

### P5-03: 월 파티셔닝 — 조회 성능

```gherkin
Scenario: 특정 월 감사 로그 조회
  Given audit_log에 2026-05 파티션에 1만 건 데이터 존재

  When SELECT * FROM audit_log WHERE org_pk=1 AND created_at BETWEEN '2026-05-01' AND '2026-05-31'

  Then p202605 파티션만 스캔 (Explain: partitions=p202605)
  And 전체 테이블 스캔 없음
```

---

## Domain 6: Consent — PIPA 동의 (phase-17)

> ⚠️ 아래 시나리오는 phase-17 구현 후 통합 테스트로 전환.

### P6-01: 가입 시 필수 동의 누락 차단

```gherkin
Scenario: PIPA 필수 동의 누락
  Given 김지영이 가입 페이지에서 개인정보 처리방침을 동의하지 않는다

  When 가입 API가 호출된다

  Then 400 Bad Request 반환 ("필수 약관에 동의해 주세요")
  And identity_user row INSERT 없음
  And user_consent_event row INSERT 없음
```

### P6-02: 정상 가입 시 동의 이벤트 기록

```gherkin
Scenario: 모든 필수 약관 동의 후 가입
  When 모든 필수 약관 동의 + 가입 API 호출

  Then user_consent_event에 다음 row들 INSERT:
    | consent_type                    | action  | version    |
    | platform.terms_of_service       | GRANTED | 2026-05-01 |
    | platform.privacy_policy         | GRANTED | 2026-05-01 |
    | platform.third_party_firebase   | GRANTED | 2026-05-01 |
  And 모든 row에 ip, user_agent 기록
  And row는 이후 UPDATE·DELETE 불가 (append-only)
```

### P6-03: 마케팅 동의 철회 — append-only

```gherkin
Scenario: 마케팅 수신 동의 철회
  Given user_consent_event에 (user_pk=10, consent_type='MARKETING', action='GRANTED') 존재

  When 철회 API가 호출된다

  Then user_consent_event에 신규 row INSERT:
    (user_pk=10, consent_type='platform.marketing_email', action='REVOKED')
  And 기존 GRANTED row는 수정되지 않는다 (append-only)
  And 가장 최신 action='REVOKED' → 현재 마케팅 미동의 상태로 판단
```

### P6-04: PIPA §17 제3자 제공 — Firebase Auth 국외이전

```gherkin
Scenario: Firebase Auth 국외이전 동의 기록
  Given 가입 시 Firebase Auth 국외이전 동의 항목이 필수로 표시된다

  When 가입이 완료된다

  Then user_consent_event에 INSERT:
    consent_type='THIRD_PARTY_FIREBASE_AUTH',
    action='GRANTED',
    meta_json={
      "recipient": "Google Firebase",
      "purpose": "이메일/비밀번호 인증",
      "items": ["email", "password_hash"],
      "retention": "서비스 탈퇴 후 30일"
    }
```

---

## Domain 7: 멀티테넌시 & BOLA 방어

### P7-01: BOLA — 타 테넌트 리소스 접근 차단

```gherkin
Scenario: 다른 org의 리소스 ID로 접근 시도
  Given identity_user.pk=10 (김지영)이 org_pk=1 소속이다
  And org_pk=2에 lecture.pk=999 (다른 학원 강의)가 존재한다

  When org_pk=1 컨텍스트에서 lecture.pk=999 조회를 시도한다

  Then 쿼리에서 WHERE org_pk=1 AND pk=999 → row 없음
  And 404 NotFoundException 반환 (존재 여부 비공개)
  And audit_log에 (result='DENY', action='get_lecture') INSERT
```

### P7-02: 멀티테넌시 격리 — Qdrant org_id 필터 강제

```gherkin
Scenario: Qdrant 벡터 검색에서 org 격리
  Given Qdrant collection 'academy_lectures'에 org_id='org_A' 포인트 1000개, org_id='org_B' 포인트 500개가 혼재한다

  When org_A 컨텍스트에서 RAG 검색을 수행한다

  Then 검색 요청에 filter.must=[{key:'org_id', match:{value:'org_A'}}] 가 포함된다
  And 반환된 결과에 org_id='org_B' 포인트가 없다
  And org_B 포인트는 org_A 검색 결과에 절대 포함되지 않는다
```

### P7-03: Neo4j 멀티홉 경로 전체 org_id 강제

```gherkin
Scenario: 멀티홉 그래프 쿼리에서 경로 전체 org_id 강제
  Given Neo4j에 org_id='org_A' 노드와 org_id='org_B' 노드가 관계로 연결된 케이스가 있다 (비정상적 데이터)

  When org_A 컨텍스트에서 개념 그래프를 조회한다

  Then Cypher 쿼리에 MATCH (c:Concept {orgId: $orgId}) ... MATCH (c2:Concept {orgId: $orgId}) 가 포함된다
  And 양 끝 노드 모두 org_id='org_A' 인 경우만 반환된다
  And org_B 노드를 통과하는 경로는 결과에서 제외된다
```

---

## Domain 8: perm_version 전파 & Break-glass

### P8-01: perm_version 불일치 → 클라이언트 캐시 무효화

```gherkin
Scenario: 권한 변경 후 클라이언트 캐시 갱신 강제
  Given 클라이언트가 perm_version=5로 캐시된 권한 스냅샷을 가지고 있다
  And 서버에서 membership.role이 변경되어 organization.perm_version=6으로 bump됐다

  When 클라이언트가 다음 API를 호출한다

  Then API 응답 헤더에 X-Perm-Version: 6이 포함된다
  And 클라이언트가 캐시의 perm_version=5 와 헤더의 6이 다름을 감지한다
  And 클라이언트가 GET /me/permissions를 재요청한다
  And 새 스냅샷의 perm_version=6으로 캐시가 갱신된다
```

### P8-02: Break-glass 긴급 접근 — 감사 기록 필수

```gherkin
Scenario: 운영자 긴급 접근 (Break-glass)
  Given 운영자(admin)가 platform.break_glass capability를 부여받았다
  And 승인자(approver_pk)가 break_glass 요청을 승인했다

  When 운영자가 긴급 접근으로 platform_db 조회를 수행한다

  Then 모든 행위가 audit_log에 (actor_type='HUMAN', action='break_glass_access', meta_json={reason:'서비스장애', approver_pk:...}) INSERT된다
  And audit_log row에 silent 처리가 없다 (break_glass=true 표시 필수)
  And delegation_grant.expires_at 도달 시 자동 회수된다
  And 이후 사후 리뷰(post-review) 대상으로 표기된다
```

### P8-03: webhook 서명 검증 — 서명 실패 시 처리 차단

```gherkin
Scenario: PG webhook 서명 검증 실패
  Given TOSS webhook이 수신됐다
  And HMAC-SHA256 서명이 일치하지 않는다

  When webhook handler가 서명을 검증한다

  Then pg_webhook_event INSERT: (signature_ok=false, status='FAILED')
  And 결제 처리 로직이 실행되지 않는다 (payment_ledger INSERT 없음)
  And 보안 audit_log에 (action='webhook_signature_failure', result='DENY') INSERT
```

### P8-04: fan-out anonymize — 탈퇴 시 서비스 DB 정리

```gherkin
Scenario: 회원 탈퇴 → 전 서비스 개인정보 익명화
  Given identity_user.pk=10 (김지영)이 DELETED 상태로 전환됐다
  And 30일이 경과했다

  When 탈퇴 정리 배치가 실행된다

  Then outbox_event에 (event_type='user.deleted', aggregate_pk=10) INSERT
  And outbox 워커가 academy-api에 user.deleted 이벤트 전달
  And academy_db에서 student.display_name, parent_phone 등이 익명화 처리된다
  And platform_db.user_consent_event는 5년 보존 후 파기 대상으로 표기된다
  And identity_user.email, phone_e164가 NULL 처리된다
```

---

---

## Domain 9: 데이터 이전 — 강사 이전 & 회원탈퇴

> 격리 구조(org_pk NOT NULL) 때문에 이전 가능/불가 경계가 명확하다. admin 페이지에서 처리.

### P9-01: 강사 멤버십 이전 (consent 기반)

```gherkin
Scenario: 강사가 consent 후 admin을 통해 org B로 이전
  Given teacher (identity_user.pk=20)이 org_pk=1(A학원)의 TEACHER 멤버십 보유
  And user_consent_event에 (user_pk=20, consent_type='platform.content_ownership', action='GRANTED') 존재
  And user_consent_event에 (user_pk=20, consent_type='platform.data_transfer', action='GRANTED') 존재
  And org_pk=2(B학원)에서 membership_invite (token='xfer_abc', email=teacher.email) 발송됨

  When admin이 강사 이전 처리를 실행한다

  Then 단일 트랜잭션 내에서:
    | 처리 | 내용 |
    | membership(user_pk=20, org_pk=1) | status='SUSPENDED' |
    | delegation_grant(org_pk=1, grantee_pk=20) 전체 | status='REVOKED' |
    | membership_invite(token='xfer_abc') | status='ACCEPTED' |
    | membership(user_pk=20, org_pk=2, role='TEACHER') | INSERT |
    | outbox_event | event_type='teacher.transferred', payload={from_org:1, to_org:2, user_pk:20} |
  And identity_user, user_profile은 변경 없음 (강사 본인 데이터 — 이전의 대상이 아님)
  And audit_log에 (actor_type='SYSTEM', action='teacher_transfer', org_pk=1, result='ALLOW',
                   meta_json={to_org_pk:2, consent_verified:true}) INSERT
```

### P9-02: consent 없는 이전 시도 차단

```gherkin
Scenario: platform.data_transfer 동의 없이 admin 이전 시도
  Given user_consent_event에 (consent_type='platform.data_transfer') row가 없다

  When admin이 강사 이전 처리를 실행한다

  Then 403 Forbidden ("강사 본인의 데이터 이전 동의가 필요합니다")
  And membership 변경 없음
  And audit_log에 (action='teacher_transfer', result='DENY') INSERT
```

### P9-03: 이전 불가 데이터 경계 확인

```gherkin
Scenario: 강사 이전 후 org A 데이터는 org A에 남아있음
  Given P9-01 이전이 완료됐다

  Then academy_db 내 (org_pk=1, teacher_pk=20)으로 태그된 기존 강의는 org A에 남아있다
  And 학생 등록 기록 (org_pk=1)은 변경되지 않는다
  And payment_ledger (org_pk=1)은 변경되지 않는다
  And teacher_pk=20은 org_pk=2에서 새 강의를 생성할 수 있다 (identity 이전됐으므로)

  Note: 기존 강의 콘텐츠 이전을 원하면 별도 admin 작업 필요
        (academy_db lecture.org_pk 업데이트 — ToS 콘텐츠 소유권 조항 근거)
```

### P9-04: 회원탈퇴 — platform_db 즉시 처리

```gherkin
Scenario: 회원 탈퇴 요청 → 즉시 처리 + 30일 배치
  Given identity_user.pk=10 (김지영)이 탈퇴 요청
  And user_consent_event에 (consent_type='platform.withdrawal', action='GRANTED') INSERT됨

  When 탈퇴 API가 호출된다 (단일 트랜잭션)

  Then identity_user.status='DELETED', deleted_at=NOW()
  And membership(user_pk=10) 전체 status='SUSPENDED'
  And delegation_grant(grantee_pk=10 또는 grantor_pk=10) 전체 status='REVOKED'
  And outbox_event INSERT: event_type='user.deleted', aggregate_pk=10
  And 전 서비스 즉시 차단 (Gate A: status='DELETED' → false)

  When 30일 후 hard anonymize 배치가 실행된다

  Then identity_user.email=NULL, phone_e164=NULL, firebase_uid=NULL UPDATE
  And identity_user.email_verified=FALSE, phone_verified=FALSE
  And user_profile.avatar_url=NULL UPDATE
  And user_consent_event rows는 5년 보존 (user_pk는 익명화 안 함 — PIPA 증거)
  And audit_log에 (action='user_hard_anonymize', actor_type='SYSTEM') INSERT
```

### P9-05: 탈퇴 후 동일 이메일 재가입 허용

```gherkin
Scenario: 탈퇴한 이메일로 재가입
  Given identity_user.pk=10이 탈퇴 후 30일 hard anonymize 완료 (email=NULL)

  When 동일 email='kim@example.com'으로 신규 가입 시도

  Then UNIQUE KEY uq_identity_user_email이 email=NULL이라 충돌 없음
  And 신규 identity_user row INSERT 성공 (새 pk, 새 firebase_uid)
  And 이전 user_pk=10과 연관 없음
```

---

## Domain 10: Email & Phone 인증

### P10-01: 이메일 미인증 → 민감 기능 차단

```gherkin
Scenario: 이메일 미인증 사용자 결제 시도
  Given identity_user.email_verified=false
  And JWT claims에도 email_verified=false

  When 결제 API를 호출한다

  Then 403 Forbidden ("이메일 인증 후 이용 가능합니다")
  And payment_ledger에 row INSERT 없음
  And audit_log에 (result='DENY', action='payment_blocked_unverified_email') INSERT

Scenario: 이메일 미인증 사용자 멤버 초대 시도
  Given identity_user.email_verified=false

  When 멤버 초대 API를 호출한다

  Then 403 Forbidden ("이메일 인증 후 초대 가능합니다")
  And membership_invite에 row INSERT 없음
```

### P10-02: Firebase 이메일 인증 완료 → DB 동기화

```gherkin
Scenario: 사용자가 Firebase 인증 이메일 링크 클릭 후 API 재호출
  Given identity_user.email_verified=false
  And 사용자가 Firebase 인증 이메일 링크를 클릭했다
  And Firebase가 JWT email_verified=true로 갱신했다

  When 사용자가 다음 API를 호출한다 (FirebaseAuthGuard 통과)

  Then FirebaseAuthGuard가 JWT의 email_verified=true를 감지한다
  And identity_user.email_verified=true, email_verified_at=NOW() UPDATE
  And 이후 이메일 인증 필요 기능 접근 허용
```

### P10-03: SMS OTP 전화번호 인증

```gherkin
Scenario: 전화번호 OTP 인증 성공
  Given identity_user.phone_e164='+821012345678', phone_verified=false
  And OTP SMS가 발송됐다 (TTL 5분)

  When 사용자가 올바른 OTP를 입력한다

  Then identity_user.phone_verified=true, phone_verified_at=NOW() UPDATE
  And audit_log에 (action='phone_verified', result='ALLOW') INSERT

Scenario: 전화번호 OTP 만료 후 시도
  Given OTP TTL 5분이 경과했다

  When 사용자가 OTP를 입력한다

  Then 400 Bad Request ("인증번호가 만료됐습니다. 재발송하세요")
  And phone_verified=false 유지
```

### P10-04: OTP 브루트포스 차단 (rate-limit)

```gherkin
Scenario: OTP 5회 연속 실패 → 차단
  Given 동일 phone_e164로 OTP 검증이 5회 연속 실패했다

  When 6번째 OTP 시도

  Then 429 Too Many Requests ("잠시 후 다시 시도하세요")
  And OTP 무효화 (재발송 필요)
  And audit_log에 (action='phone_otp_brute_force', result='DENY') INSERT

  Note: rate-limit은 Gateway 레이어에서 강제 (TEN-9, api_key.rate_limit_tier)
```

---

# C. academy BDD 수용성 크로스체크

> 상세 시나리오: [`docs/academy/v3/bdd-scenarios.md`](../academy/v3/bdd-scenarios.md)  
> 이 섹션은 platform_db 설계가 academy BDD를 수용하는지 갭만 기록.

| academy BDD | platform_db 설계 수용 여부 | 갭/조치 |
|---|---|---|
| F1-01 신규 학원장 가입 | ✅ | identity_user + org + membership + entitlement 단일 트랜잭션 설계 완료 |
| F1-02 약관 동의 | ⚠️ | user_consent_event 미구현 (phase-17). 현재 앱 레이어에서만 처리 |
| F3-01~04 강사 초대/수락/만료/제거 | ✅ | membership_invite + membership 설계 완료 |
| F4-01~04 멀티 워크스페이스 | ✅ | membership 복합 PK(user_pk, org_pk) 설계 완료 |
| F5-01~05 Trust Grant | 🟡 | delegation_grant 구현 완료. capability_code 네임스페이스는 phase-17 |
| F6-01~04 Agent 인증 | 🟡 | identity_user.type='SERVICE' ✅. api_key 테이블은 phase-17 |
| F7-01~05 권한 거부 | ✅ | 3-gate(Gate A/B/C) + VerifyOnDb 설계 완료 |
| F11~F16 파이프라인 | ✅ (academy_db) | platform_db와 직접 무관. entitlement.feature_limits로 cap 확인 |
| F43 구독 만료 | ✅ | org_entitlement.status=EXPIRED 전환 설계 완료 |
| F47~F48 결제 흐름 | ✅ | payment_ledger + pg_webhook + outbox 설계 완료 |

**식별된 갭**:

| 갭 | 심각도 | 조치 |
|---|---|---|
| C-1 `user_consent_event` 미구현 | PIPA P0 법적 | phase-17 spec-17-xx |
| C-2 `api_key` 미구현 | B2B 머신 인증 P1 | phase-17 spec-17-xx |
| C-3 `service_membership` 미구현 | cross-service 권한 P1 | phase-17 spec-17-xx |
| C-4 `capability_code` 네임스페이스 미적용 | academy 어휘 platform 고착 P1 | phase-17 spec-17-xx |
| C-5 youtube_channel의 `org_pk` 컬럼 확인 필요 | 멀티테넌시 불변식 | spec-16-xx 확인 |
