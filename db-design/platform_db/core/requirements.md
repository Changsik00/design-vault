---
type: core
aliases:
  - 요구사항 추적표
  - requirements
  - 플랫폼 목적 요구사항
tags:
  - platform-db
  - core
  - requirements
  - traceability
  - platform-purpose
---

# platform_db — 요구사항 & 검증 (플랫폼 목적 기준)

> 진입점: [[architecture]] · 설계: [[schema-reference]] · 행위 검증: [[bdd-scenarios]]
>
> **이 문서의 관점**: 요구사항을 *"academy를 충족하는가"*가 아니라 ***"platform_db의 목적을 달성하는가"***로 측정한다.  
> academy/market/agent는 **적합성 테스트 케이스**([[bdd-scenarios]])이지 요구사항의 출처가 아니다.
>
> 📄 **이 저장소는 설계 문서(Markdown)이며 구현 코드를 포함하지 않는다.** 따라서 아래 상태는 *구현 여부*가 아니라 **설계 성숙도**를 뜻한다 — 스키마(§)에 DDL·결정이 있으면 "설계 확정"이다.
> **상태 범례**: ✅ 설계 확정 · 🟡 설계 부분(보강 필요) · ⛔ 보류(트리거 미충족) · ❓ 미결정 · 🔴 **미설계 / 목적 부채**

---

## 0. 플랫폼 목적 — 무엇을 위한 DB인가

platform_db는 특정 서비스를 위한 DB가 아니다. **N개 서비스가 꽂히는 service-agnostic 공통 코어**다. 존재 이유는 둘:

1. **서비스 추가 한계비용 최소화** — 두 번째·세 번째 서비스를 붙일 때 *데이터/코드*만으로 되고 *플랫폼 스키마 마이그레이션*은 없어야 한다.
2. **서비스 횡단 불변식 보장** — identity SSOT · 테넌트 격리 · 결제↔권한 원자성 · 감사 · 동의를 모든 서비스에 동일하게 강제한다.

> 따라서 platform_db의 가치 = **"세 번째 서비스를 얼마나 싸게 붙이는가"** + **"횡단 불변식을 구조로 강제하는가"**.  
> 이 두 가지가 1급 요구사항이고, 개별 기능 요구사항(§3)은 그 아래 수단이다.

---

## 1. 확장성 요구사항 (EXT) — *1급, 목적 직결*

> **신설.** 플랫폼의 핵심 가치 명제(서비스 추가 한계비용)를 처음으로 측정 가능한 요구로 끌어올린다.  
> 기존 추적표에서 "미설계 nice-to-have"로 묻혀 있던 결합들이, 목적 렌즈에서는 **P0 부채(🔴)**다.

| ID | 요구사항 | 측정 기준 | 상태 |
|---|---|---|---|
| EXT-1 | 새 서비스 추가가 **platform 스키마 마이그레이션 없이** 가능 | 서비스 N 추가 시 변경되는 platform DDL = 0 | 🟡 **의도적 트레이드오프** — service CHECK는 온라인 1줄 마이그레이션(D6). [[service-extensibility]] Option A 채택 |
| EXT-2 | 플랫폼 코어 인가에 **서비스 고유 어휘 하드코딩 금지** | 코어 테이블에 academy 동사/역할 리터럴 0 | 🟡 **의도적 트레이드오프** — `chk_capability` `ACADEMY.*` 6종(코드 ROLE_PERMISSION과 중복 안전망). [[service-extensibility]] |
| EXT-3 | role→action 매핑은 **코드 상수** (서비스 역할 추가 = 코드 배포, 스키마 무변경) | 새 서비스 역할 권한 추가 시 DB DDL = 0 | ✅ [[role-as-code]] `ROLE_PERMISSION[service]` |
| EXT-4 | 멤버십은 **service 차원만 추가** — `service_membership.role_code` VARCHAR 자유 확장 | 새 서비스 역할 = 행 INSERT(스키마 무변경) | ✅ 설계 확정 |
| EXT-5 | 게이트·기본값에 **특정 서비스 편향 금지** | 플랫폼 API/게이트가 서비스 중립 | 🟡 — `checkGateB(...="ACADEMY")` 기본값. 2번째 서비스 도입 시 중립화 ([[service-extensibility]]) |
| EXT-6 | **신규 서비스 onboarding 절차** 문서화 — 연결 추가만으로 entitlement 동작 | onboarding 체크리스트 존재 + 검증 | 🟡 부분 ([[architecture]] §2.4 단편) |

> **결정 완료**: EXT-1/EXT-2/EXT-5의 service·capability 결합은 [[service-extensibility]]에서 종결 — **Option A(CHECK 유지) 채택**(온라인 저비용 + YAGNI, 코어가 어휘를 아는 건 의도적 수용). 2번째 서비스 churn 시 코드/데이터 기반(B/C)으로 전환(트리거 명시). 그래서 🔴(미관리 부채)가 아니라 **🟡(의도적·시한부 트레이드오프)**.

---

## 2. 목적 pillar 지도 — 기존 요구사항을 무엇으로 보는가

기존 기능 요구사항(§3, 104건)은 아래 6개 **목적 pillar**를 떠받치는 수단이다. "출처(R라운드)"는 연혁일 뿐, *왜 필요한가*는 pillar로 읽는다.

| Pillar | 목적 | 떠받치는 요구사항 군 | 핵심 불변식 |
|---|---|---|---|
| **P1 Identity SSOT** | 전 서비스 단일 사용자 | USR, AUTHN | #1 Firebase=인증, #2 PK/ULID |
| **P2 Service-agnostic 인가** | 서비스 무관 권한 모델 | RBAC, ABAC, ReBAC | #5 service=VARCHAR+CHECK, role=코드 |
| **P3 Billing↔Auth 원자성** | 결제=권한, 보편 게이트 | BILL | #4 canXXX=entitlement만, #7 단일 트랜잭션 |
| **P4 테넌트 격리** | org 횡단 격리 | TEN | #3 org_pk NOT NULL |
| **P5 감사·동의 컴플라이언스** | 법적 증거·불변 기록 | AUD, CON | append-only, WORM |
| **P6 머신·B2B 신원** | 사람=머신 동일 3-gate | api_key·SERVICE 계정 (AUTHN-5, SEC-5/7, RBAC-4) | type=SERVICE |

> ⚠️ **우선순위 비판**: P6(머신 신원, `api_key`)은 **agent 서비스의 존재 전제**다. 스키마(§J)는 설계 확정이나 *우선순위*에서 계속 밀린다 — academy 역할 설계가 무르익는 동안 P6는 거의 다뤄지지 않았다("손에 쥔 서비스 > 플랫폼 명제"). 두 번째 서비스가 agent라면 P6가 academy 잔여보다 앞서야 한다.

---

## 3. 기능 요구사항 추적 (pillar 그룹)

### P1 — Identity SSOT

| ID      | 요구사항                                                     | 출처  | 적용 설계                                               | 상태      |
| ------- | -------------------------------------------------------- | --- | --------------------------------------------------- | ------- |
| USR-1   | 1 firebase_uid = 전 서비스 동일 사용자(SSOT)                      | R0  | identity_user 중앙화                                   | ✅       |
| USR-2   | 내부 PK BIGINT, 외부 노출 ULID(`public_id`)                    | R0  | 불변식 #2                                              | ✅       |
| USR-3   | HUMAN/SERVICE/SYSTEM 동일 모델 + type 구분                     | R0  | identity_user.type                                  | ✅       |
| USR-4   | 표시이름 SSOT는 `user_profile`, 서비스별 프로필은 서비스 DB              | R0  | §2.1                                                | ✅       |
| USR-5   | email은 ACTIVE 중 unique(탈퇴 후 재사용 허용)                      | R2  | unique index (DDL 규약)                               | 🟡 P0   |
| USR-6   | 상태 ACTIVE/SUSPENDED/DELETED, soft→hard delete, anonymize | R0  | identity_user.status + deleted_at                   | ✅       |
| USR-7   | 1 user = N org (멀티 워크스페이스)                               | R0  | membership 복합 PK                                    | ✅       |
| USR-8   | 소셜(Kakao/Naver) custom token + 다중 provider 계정연결          | R2  | custom token ✅ / 연결 모델                              | 🟡 v1.0 |
| USR-10  | `email_verified` — Firebase JWT 동기화. 필수 기능(결제·초대) 차단 근거  | 신규  | identity_user.email_verified                        | 🟡 P1   |
| USR-11  | `phone_verified` — SMS OTP/Firebase Phone. 전화 수집 시 인증 추적 | 신규  | identity_user.phone_verified                        | 🟡 P1   |
| USR-12  | 데이터 이식성 — 본인 데이터 admin 이전(동의 기반), org 소유 데이터는 이전 불가      | 신규  | user_consent_event(content_ownership·data_transfer) | 🟡 P1   |
| AUTHN-1 | Firebase Auth = 인증 SSOT, firebase_uid는 조회 키(PK/FK 아님)    | R0  | 불변식 #1                                              | ✅       |
| AUTHN-2 | 비밀번호 강도·이메일 인증·재설정·토큰 TTL 1h                             | R0  | Firebase 정책                                         | ✅       |
| AUTHN-3 | 만료/위조 토큰 Guard 401 빠른 차단                                 | R2  | FirebaseJwtGuard                                    | ✅       |
| AUTHN-4 | MFA (OWNER/관리자, v0.5)                                    | R0  | Firebase MFA + platform_role                        | 🟡      |
| AUTHN-6 | 계정 정지/삭제 시 전 서비스 즉시 차단                                   | R0  | identity_user.status                                | ✅       |
| AUTHN-7 | 다중 디바이스 허용, 이상 로그인 감지                                    | R0  | Firebase                                            | ✅       |
| AUTHN-8 | JWT claim 신뢰 경계 — 필수 claim 누락 fail-closed + DB fallback  | R6  | FirebaseJwtGuard 규약                                 | 🟡      |

### P2 — Service-agnostic 인가 (RBAC / ABAC / ReBAC)

> ⚠️ **목적 정렬**: RBAC-2·REBAC-2는 *기능적으로 ✅*지만 **EXT-2(§1)** 관점에선 **service 결합 부채**다. "academy 어휘 격리"를 넘어 "코어에 어휘 없음"까지 가야 완성.

| ID      | 요구사항                                                                            | 출처  | 적용 설계                          | 상태                                                               |
| ------- | ------------------------------------------------------------------------------- | --- | ------------------------------ | ---------------------------------------------------------------- |
| RBAC-1  | role 2단: `platform_role`(OWNER/MEMBER/SERVICE) + `service_membership.role_code` | R3  | D1                             | ✅                                                                |
| RBAC-2  | 서비스별 role 어휘 격리 (`academy.director`, `market.seller`…)                          | R3  | service_membership             | ✅ (단 EXT-4 충족, EXT-2는 별개)                                        |
| RBAC-3  | role→action 매핑은 코드 상수(DB 저장 금지)                                                 | R0  | ROLE_PERMISSION                | ✅                                                                |
| RBAC-4  | 서비스 계정 = `platform_role='SERVICE'` + api_key                                    | R3  | D4                             | 🟡 SERVICE 확정 / api_key 코드 트랙 별도                                 |
| RBAC-5  | 역할 변경 즉시 반영(perm_version)                                                       | R0  | bumpPermVersion()              | ✅                                                                |
| RBAC-6  | 마지막 OWNER lockout 방지                                                            | R2  | 앱 트랜잭션 가드                      | 🟡                                                               |
| RBAC-7  | DB role 레지스트리는 테넌트 커스텀롤 트리거 시                                                   | R3  | P2 보류                          | ⛔                                                                |
| RBAC-8  | ROLE_PERMISSION 변경 배포 SLA 명시 — hot-fix vs weekly                                | R6  | §5.3 열린 결정                     | ❓                                                                |
| ABAC-1  | 소유권(`owner_pk == principal`) 기반 통제                                              | R0  | CASL ability                   | ✅                                                                |
| ABAC-2  | 테넌트 속성(`org_pk` 일치) 모든 도메인 강제                                                   | R0  | 불변식 #3                         | ✅                                                                |
| ABAC-3  | feature_limit 한도 평가(entitlement), 카운터는 서비스측                                     | R2  | org_entitlement.feature_limits | ✅                                                                |
| ABAC-4  | entitlement 상태 게이트(ACTIVE/GRACE/SUSPENDED/EXPIRED)                              | R0  | Gate B, 불변식 #4                 | ✅                                                                |
| ABAC-5  | 리소스 visibility 속성 필터                                                            | R0  | CASL                           | ✅                                                                |
| ABAC-6  | NIST 환경속성 — api_key `allowed_ip_cidr` + Gateway/WAF IP                          | R4  | D8 (P1)                        | 🟡                                                               |
| ABAC-7  | 최종 `can()` 결정 캐싱 금지, 입력 블록만 TTL 60s                                             | R0  | Redis 전략                       | ✅                                                                |
| REBAC-1 | capability 위임(grantor→grantee, scoped, expiry, revoke)                          | R0  | delegation_grant               | ✅                                                                |
| REBAC-2 | `capability` 서비스 네임스페이스 `<service>.<action>`(`ACADEMY.*`)                       | R3  | D2                             | ✅ (🟡 EXT-2: CHECK 하드코딩은 의도적 트레이드오프 → [[service-extensibility]]) |
| REBAC-3 | 임퍼소네이션 금지, 위임 행사를 감사 기록                                                         | R0  | audit_log                      | ✅                                                                |
| REBAC-4 | org 계층(HQ_BRANCH/HOLDING)                                                       | R0  | org_relation                   | ✅                                                                |
| REBAC-5 | 계층은 권한 근거 아님, 명시적 membership만                                                   | R2  | 불변식                            | ✅                                                                |
| REBAC-6 | 자기참조 차단(DB CHECK constraint)                                                    | R2  | chk_no_self_ref                | ✅                                                                |
| REBAC-7 | Zanzibar full relation_tuple은 보류                                                | R0  | ⛔ 트리거 시                        | ⛔                                                                |
| REBAC-8 | `delegation_grant`(platform) ↔ 서비스 `trust_relationship`(도메인) 경계 규칙              | R6  | 아키텍처 규약                        | ✅(규칙)/🟡(구현)                                                     |

### P3 — Billing↔Auth 원자성 & Entitlement 게이트

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| BILL-1 | cross-service 단일 상품 카탈로그(product/product_sku) | R0 | §2.3 | ✅ |
| BILL-2 | `org_entitlement` = 런타임 권위 (Gate B만, ledger 직접 조회 금지) | R0 | 불변식 #4 | ✅ |
| BILL-3 | 결제→entitlement 단일 트랜잭션 + perm_version + outbox | R0 | §1.3 | ✅ |
| BILL-4 | 멱등(webhook `event_id` UNIQUE, payment `idempotency_key` UNIQUE) | R0 | pg_webhook_event / payment_ledger | ✅ |
| BILL-5 | 환불/chargeback append-only, 금액은 정수 minor(float 금지) | R0 | payment_ledger | ✅ |
| BILL-6 | GRACE 유예기간, grace_until | R0 | org_entitlement.grace_until | ✅ |
| BILL-7 | 업그레이드(즉시)/다운그레이드(예약) | R2 | 🟡 P1 | 🟡 |
| BILL-8 | PG adapter 추상화(Toss/Stripe/PayPal/Manual) | R0 | pg_provider | ✅ |
| BILL-9 | PG webhook 서명 검증(handler, INSERT 전) + replay tolerance(±5분) | R6 | signature_ok 컬럼 | 🟡 코드 규약 |
| BILL-10 | source=FREE/MANUAL/PROMO로 결제 없이 access 부여 | R0 | org_entitlement.source | ✅ |

### P4 — 테넌트 격리 불변식

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| TEN-1 | 모든 도메인 테이블 `org_pk NOT NULL`(예외 3부류: 전역 카탈로그·플랫폼 이벤트·audit SYSTEM) | R0 | 불변식 #3 | ✅ |
| TEN-2 | 모든 조회 `WHERE org_pk` 강제, 타 org는 404 | R1 | [[cross-tenant-separation]], BOLA | ✅ |
| TEN-3 | cross-tenant 조회는 아키텍처 분리(`internal/`) | R1 | [[cross-tenant-separation]] | ✅ |
| TEN-4 | PG RLS로 테넌트 격리 강제 + 앱 `org_pk`·CI 린트(defense-in-depth) | R1 | 🟡 린트 미도입 | 🟡 |
| TEN-5 | Qdrant payload 필터 + `org_id` 인덱스 | R1 | [[rag-multitenancy]] | 🟡 |
| TEN-6 | Neo4j `org_id` 속성 + 멀티홉 경로 전체 강제 | R1 | [[rag-multitenancy]] | 🟡 |
| TEN-7 | 분리 트리거 T1~T4 사전 정의 | R0 | [[multitenancy-pool]] | ✅ |
| TEN-9 | rate-limit 정책 위치 — org=`feature_limits` / 머신=`api_key.rate_limit_tier`, 강제는 Gateway | R6 | §3.1 결정 | 🟡 |

### P5 — 감사 & 동의 컴플라이언스

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| AUD-1 | `audit_log` append-only, 월 파티셔닝, actor 통합 | R0 | §D.8 | ✅ |
| AUD-2 | **보안유의 이벤트** 기록(DENY·ERROR·민감 ALLOW·운영자) — 일상 read ALLOW는 텔레메트리(샘플링), audit 아님 | R0 | [[audit-two-lane]] | 🟡 범위 재정의(2-lane 설계됨·스키마 부분) |
| AUD-3 | trace_id + audit_event_id(분산 추적) | R4 | 🟡 P1 | 🟡 |
| AUD-4 | 사람·머신 활동 통계 분리(type 필터) | R2 | identity_user.type | ✅ |
| CON-1 | `user_consent_event` append-only 이벤트 | R5 | D7 | ✅ 설계 확정 |
| CON-2 | consent_type 네임스페이스(`platform.*`/`pg.*`) | R5 | user_consent_event 설계 | ✅ 설계 확정 |
| CON-3 | 14세 미만 법정대리인 동의(PIPA §22) | R5 | 🟡 P0 법적 필수 | ✅ 설계 확정 |
| CON-4 | 제3자 정보제공 4요건(PIPA §17) | R5 | 🟡 P0 법적 필수 | ✅ 설계 확정 |
| CON-5 | 마케팅 수신/거부(정보통신망법 §50) | R5 | 🟡 | ✅ 설계 확정 |
| CON-6 | 동의 철회권(PIPA §37) — REVOKED 이벤트 | R5 | 🟡 | ✅ 설계 확정 |
| CON-7 | 약관 버전 관리 + 재동의 인터셉터 | R5 | 🟡 P1 | 🟡 |
| CON-8 | `platform.content_ownership` — 콘텐츠 소유권 약관 동의(전자서명법 §3) | 신규 | user_consent_event | ✅ 설계 확정 |
| CON-9 | `platform.data_transfer` — 이전 처리 전 본인 동의 필수 | 신규 | user_consent_event | ✅ 설계 확정 |
| CON-10 | `platform.withdrawal` — 탈퇴 최종 확인 동의 | 신규 | user_consent_event | ✅ 설계 확정 |
| CON-12 | fan-out anonymize — 탈퇴 시 outbox `user.deleted` → 각 서비스 anonymize | R6 | §4 | 🟡 P1 |
| CON-13 | 동의 `meta_json` canonical(RFC 8785 JCS) + JSON Schema 검증 | R6 | user_consent_event | 🟡 P1 |

### P6 — 머신·B2B 신원 (api_key)

> agent·B2B 통합의 전제. 사람과 동일 3-gate를 통과하는 머신 신원. **스키마(§J)는 설계 확정, 코드 트랙은 별도(이 저장소 범위 밖) — 두 번째 서비스가 agent라면 구현 최우선 후보.**

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| AUTHN-5 | B2B = `api_key`(prefix+secret_hash, scopes, IP, rotation, 즉시 revoke) | R0 | api_key 테이블 설계 확정 | ✅ 설계 확정 |
| SEC-5 | api_key 하드닝(allowed_ip_cidr·rotated_at·revoked_reason) | R4 | D8 | ✅ 설계 확정 |
| SEC-7 | api_key 보강 — `last_used_ip`·`created_by_user_pk`·`rate_limit_tier`·`environment` | R7 | api_key 설계 | 🟡 P1 |

### 횡단 — 아키텍처 · 보안 · 비기능 · 운영

| ID | 요구사항 | 출처 | 적용 설계 | 상태 |
|---|---|---|---|---|
| ARCH-1 | 공통 코어(identity/billing/product) 단일 `platform_db`, 도메인은 서비스별 DB | R0 | [[design-asymmetry]] | ✅ |
| ARCH-2 | strong-consistency(결제↔권한)는 단일 DB 트랜잭션, 2PC·Kafka 없음 | R0 | [[payment-atomicity]] | ✅ |
| ARCH-3 | cross-DB는 서비스→platform 읽기만, peer 금지, least-privilege 계정 | R0 | 불변식 #6 | ✅ |
| ARCH-4 | `@platform-db` 패키지로만 platform 접근 | R0 | [[identity-billing-access]] (A) | ✅ |
| ARCH-5 | cross-schema FK 하드 금지(독립 백업/복원) | R0 | [[fk-strategy]] | ✅ |
| ARCH-6 | async 부수효과는 `outbox_event` | R0 | [[outbox-pattern]] | ✅ |
| SEC-1 | BOLA 방어 — org_pk 질의 강제 프레임워크화 | R4 | Drizzle base repo | ✅ |
| SEC-2 | NIST 환경속성 — api_key IP/컨텍스트 | R4 | D8 (P1) | 🟡 |
| SEC-3 | 감사 불변성 — UPDATE/DELETE 권한 제거 + 해시 체이닝 + WORM | R4 | append-only ✅ / 해시 🟡 | 🟡 |
| SEC-4 | secret 관리 — KMS, 평문 로그 금지 | R0 | 정책 | ✅ |
| SEC-6 | secret rotation cadence — api_key 90/365d, Firebase Admin SDK 180d | R7 | 운영 플레이북 | 🟡 |
| SEC-8 | 데이터 분류(PII/민감/미성년/결제) + 선별 app-level 암호화(secret·guardian) | R7 | §2.2 | 🟡 분류 P0·암호화 P1 |
| NFR-1 | routine read 권한평가 DB hit 0(JWT claims), sensitive write만 DB 재검증 | R0 | @VerifyOnDb | ✅ |
| NFR-2 | perm_version 무효화 폭 분리(user/org) | R2 | bumpPermVersion | ✅ |
| NFR-3 | 확장 로드맵 — read replica → 테이블 분리 → DB-per-tenant | R0 | [[multitenancy-pool]] | ✅ |
| NFR-4 | permission_snapshot 프론트 read-model | R3 | GET /me/permissions | 🟡 |
| NFR-6 | perm_version 전파 — `/me/permissions {user_pv, org_pv}`, X-PV 헤더, 멀티인스턴스 sync | R6 | §1.2 | ✅(설계)/🟡(P1) |
| OPS-1 | platform_db 비대화 방어 — 논리 바운디드 컨텍스트(모듈·오너·절단선) | R7 | §2.4 | ✅(원칙)/🟡(구현) |
| OPS-2 | Break-glass 긴급 운영 접근 — 승인·사유·만료·전건 감사 | R7 | §4 / [[break-glass]] | 🟡 P1 |
| OPS-3 | service-to-service trust — HTTP 분리 시 mTLS/internal JWT, `internal/*` 내부망 전용 | R7 | Option B 전환 시 | ⛔ |
| OPS-4 | secret rotation cadence — KMS DEK 연 1회 등 (→ SEC-6) | R6 | §4 | 🟡 P1 |
| USR-9 | 휴면(DORMANT) — 유효기간제 2023 폐지로 법 의무 아님, 선택적 제품정책 | R6 | §2.4 YAGNI | ⛔→🟡 제품 결정 |

---

## 4. 운영 가능성 (Operability) — *신규 식별 갭*

> 설계는 충실하나 **"3년 운영하면 무슨 사고가 나나"**에 대한 요구가 약하다. 아래는 platform_db가 **소유**해야 할 운영 요구 — decisions·explainers·스키마로 **설계는 대부분 확정/착수**, 구현(배치·콘솔)은 후속이다. **상세 모델은 [[operability]]**(O1~O6), 여기는 추적표.  
> **범위 규율**: 콘솔 UI·알림 임계치·DR 런북은 platform_db가 *enable*만 하고 *소유*는 별도 ops 제품/문서. 단 **Permission Debugger는 UI가 아니라 *계약*으로서 platform_db 책임**(DBG-1).

| ID | 요구사항 | 결정/계약 | 상태 |
|---|---|---|---|
| OPER-1 | 운영자 신원 평면 — 테넌트 role 아님, 별도 인증·MFA, 전건 감사 | [[operator-plane]] · §D.21 | 🟡 operator 테이블·OPERATOR_PERMISSION 설계 / 인증 인프라 코드 트랙 |
| OPER-2 | 운영자 역할 매트릭스 — CS=R / FINANCE=환불 / AUDITOR=audit R-only … 최소권한 | [[operator-plane]] | 🟡 역할 매트릭스 코드 상수 설계 |
| SUPP-1 | Support Action — 운영자 override(entitlement 강제부여·Trial 연장 등)에 who/when/why 감사 + 별도 권한 | [[operator-plane]] · [[audit-two-lane]] | 🟡 audit `support_action` §D.8 설계 / 매트릭스 코드 상수 |
| DBG-1 | **Permission Debugger 계약** — 거부 요청에 Gate A/B/C 결과+사유를 1초에 반환하는 trace (예: `A=PASS, B=FAIL(entitlement expired), C=SKIP`). **platform_db 책임**(4개 테이블 직접 뒤지는 건 설계 실패) | [[operability]] O2 · [[permission-debugger]] | ✅ 설계 확정(계약) / 구현 미착수 · ROI 최상 |
| RETN-1 | 데이터 보존·생명주기 정책 — 테이블별 보존기간(audit 5y·consent 5y·payment 7y·chat 1y…) + 파기 트리거 | [[operability]] O3 · [[data-lifecycle-retention]] | ✅ 설계 확정(정책·매트릭스) / 배치 미작성 |
| AVAIL-1 | entitlement 가용성 계약 — billing/PG 장애 시 기존 entitlement read가 영향 없이 최소 N시간 유지 | [[operability]] O5 · [[auth-projection]] | 🟡 명문화 필요 |
| OWN-1 | 마지막 OWNER 보호 — 앱 트랜잭션 가드 + **DB 레벨 방어** 검토 (현재 앱만, RBAC-6 강화) | [[operability]] O2 | 🟡 |
| USAGE-1 | 사용량 가시성 — `usage_snapshot`(집계는 platform, 이벤트 전량은 서비스 DB). feature_limits 대비 *현재 얼마 썼나*(498/500인지 5/500인지). 과금형도 동일 경로 | [[operability]] O4 | 🟡 (스키마 §D.20 설계·집계 배치 미작성) |

> **enable만 (별도 ops 제품 소관 — 여기선 트리거만)**: Admin Console UI · Alert 임계치 · DR 런북/RTO·RPO · Feature Flag rollout · 샤딩 실행. → 트리거: 2번째 서비스 / org 규모 T1~T4([[multitenancy-pool]]) / 컴플라이언스. (관측 지표·신뢰성 계약 자체는 O5·O6에서 platform_db 책임으로 정의.)

---

## 5. 목적 달성 스코어카드

### 5.1 확장성(EXT) — *플랫폼 가치 명제*

| EXT | 충족 |
|---|---|
| EXT-1 마이그레이션 없는 서비스 추가 | 🟡 의도적 트레이드오프 ([[service-extensibility]]) |
| EXT-2 코어에 서비스 어휘 없음 | 🟡 의도적 트레이드오프 ([[service-extensibility]]) |
| EXT-3 role→action 코드 상수 | ✅ |
| EXT-4 role_code 자유 확장 | ✅ |
| EXT-5 게이트 서비스 중립 | 🟡 (Gate B default, 2번째 서비스 시 중립화) |
| EXT-6 onboarding 절차 문서화 | 🟡 |

> **목적 부채 요약**: service·capability CHECK 결합 = 서비스 추가 시 platform 마이그레이션 필요. [[service-extensibility]]에서 **Option A(저비용 온라인 CHECK 유지) 의도적 채택**으로 종결 + 전환 트리거 명시. 남은 1급 과제는 **P6(머신 신원) 구현 우선순위** — 설계(§J·[[machine-identity-apikey]])는 확정, 코드 트랙만 남음.

### 5.2 기능 요구사항 (pillar별, **설계 성숙도** 기준)

| Pillar / 군 | 항목 | ✅ 설계확정 | 🟡 부분 | ⛔ 보류 | ❓ |
|---|---|---|---|---|---|
| P1 Identity (USR+AUTHN) | 18 | 11 | 7 | — | — |
| P2 인가 (RBAC+ABAC+ReBAC) | 23 | 16 | 4 | 2 | 1 |
| P3 Billing | 10 | 8 | 2 | — | — |
| P4 테넌트 격리 | 8 | 4 | 4 | — | — |
| P5 감사·동의 | 16 | 12 | 4 | — | — |
| P6 머신 신원 (api_key) | 3 | 2 | 1 | — | — |
| 횡단 (ARCH/SEC/NFR/OPS) | 22 | 11 | 10 | 1 | — |
| **합계** | **100** | **64(64%)** | **32(32%)** | **3(3%)** | **1(1%)** |

> 기존 104건은 구 A.13(R6/R7 보강)에 NFR-6·AUTHN-8·REBAC-8·BILL-12 등 **상위 항목과 중복 기재**가 있었음 → 목적 재구성 시 dedup하여 100건. (기능 합계 외 EXT 6건은 §4.1에서 별도 관리.)
>
> **2026-05-31 — 상태 축 정정**: 이 저장소는 *설계 문서*이므로 점수는 "구현률"이 아니라 **설계 성숙도**다. 스키마 DDL이 있는 consent(§I)·api_key(§J)는 "미구현"이 아니라 **설계 확정(✅)**으로 재분류(과거 ⚠️ 11건 폐기, §4 운영도 operator 평면(§D.21)·usage_snapshot(§D.20) 설계로 🔴를 닫아 *미설계* 0 — 남은 건 구현(배치·콘솔)뿐). 실행 코드는 이 저장소 범위 밖.  
> **읽는 법**: 설계는 64% 확정·32% 보강이지만, "플랫폼 목적" 관점의 핵심은 **service 결합(의도적 채택 → [[service-extensibility]])**과 **P6(머신 신원) 구현 우선순위**다(설계는 §J로 됨).

---

## 6. BDD 시나리오

→ [[bdd-scenarios]] (화이트박스 통합 — Part A/B 19개 도메인 + 요구 추적 매트릭스) · [[e2e-journeys]] (블랙박스 E2E 사용자 여정 C-01~07 — API 경계에서만 관찰).

---

## 7. 범위 경계 — enterprise 원본에서 좁혀낸 것

> 출처: [[enterprise-saas-multitenancy]](아카이브 — 결정 전 요구수렴 기록). platform_db는 그 넓은 그림에서 **결정·구현된 부분만** 다루고, 나머지는 아래처럼 *명시적으로* 범위를 긋는다. (원본 대부분은 §3 기능 요구로 흡수됨.)

### 7.1 의도적으로 좁힌 결정 (원본의 "고민 필요" → 우리 결정)

| 원본 | 우리 결정 |
|---|---|
| 인증: Firebase **또는** Supabase | **Firebase 확정** ([[firebase-boundary]]) |
| DB: MySQL **또는** Postgres | **PostgreSQL 권장(1순위)**. 회사 인프라가 MySQL이면 각 절의 "🐬 MySQL이라면" 노트를 따른다 |
| identity_db/billing_db 분리 **고민** | **통합 platform_db** ([[design-asymmetry]]) |
| 조직 타입 academy/store/franchise/hospital | **generic `org.type`** + `entitlement.service` ([[service-extensibility]]) |
| 내부 PK: ULID/BIGINT/UUID v7 | **BIGINT + ULID public_id** (불변식 #2) |

### 7.2 미타겟 갭 (원본엔 있으나 우리 미반영 — 추적용)

| ID | 갭 | 출처 | 상태 |
|---|---|---|---|
| SETTLE-1 | **프랜차이즈/다지점 정산**(본사↔지점 매출 분배) | 원본 §4·§5 | ⛔ franchise 미타겟 — franchise 테넌트 도입 시 |
| WEBHOOK-OUT-1 | **아웃바운드 고객 webhook**(우리→고객 시스템) | 원본 §13 | ⛔ 범위 밖 — 인바운드 PG webhook만 설계. B2B 고객 트리거 시 |
| ORGGRAPH-1 | **다단계 조직 그래프**(프랜차이즈 계층·병원 다지점) | 원본 §12 | 🟡 `org_relation`은 HQ_BRANCH/HOLDING 2단 — 깊은 계층 수요 시 |
| B2BAPI-1 | B2B API 심화: OAuth Client Credentials · API versioning · scope-per-product | 원본 §13 | ⛔ — api_key 기본(P6) 후 B2B 정식화 시 |
| AUDIT-REG-1 | 조직 타입별 감사 요건 상이(병원 HIPAA·금융) | 원본 §13 | ⛔ 범위 밖 — 규제 산업 테넌트 없음 |

> 대부분의 ⛔는 franchise/hospital/B2B-정식화 등 **아직 안 겨냥한 시나리오**다(현재 플랫폼이 깨진 게 아님). 단 **WEBHOOK-OUT-1·SETTLE-1**은 academy/market만 커져도 닿을 수 있어 ⛔(트리거 시) 경계에 둔다.
