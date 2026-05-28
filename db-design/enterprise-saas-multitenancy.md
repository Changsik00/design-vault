# Enterprise SaaS Multitenancy DB 설계 요구사항

> **컨텍스트**: 엔터프라이즈급 SaaS 플랫폼의 멀티테넌트 DB 설계에 대해 AI agent 와 나눈 요구사항 수렴 기록.
> 인증은 **Firebase Authentication** 또는 **Supabase Auth** 기준으로 설계.
> 학원, 회사, 매장, 프랜차이즈, 병원 등 다양한 조직 유형을 포괄하는 범용 SaaS 구조 상정.

---

## 1. 플랫폼 구조 요구사항

* 하나의 플랫폼에서 여러 서비스 운영 가능해야 함

  * academy (학원/교육기관)
  * market (이커머스/마켓)
  * store (오프라인 매장/POS)
  * crm (고객 관계 관리)
  * hrm (인사/근태 관리)
  * agent (AI 자동화)
  * future services

* 모든 서비스는 공통 사용자 시스템을 공유해야 함

* 서비스별로 독립적인 도메인/기능/권한 구조 필요

* 멀티 테넌트(organization 기반) 구조 필요

---

## 2. 사용자 / 조직 구조 요구사항

* 사용자(User)는 여러 조직(Organization)에 소속될 수 있어야 함

* 조직 타입 다양하게 존재

  | 타입 | 예시 |
  |------|------|
  | `academy` | 학원, 교육원, 온라인 강의 플랫폼 |
  | `company` | 일반 기업, 스타트업, 팀 단위 조직 |
  | `store` | 단독 매장, 편의점, 음식점 |
  | `franchise` | 프랜차이즈 본사/지점 구조 |
  | `hospital` | 병원, 의원, 클리닉 |
  | `team` | 소규모 팀, 프리랜서 그룹 |

* 조직 내부 계층 관계 필요 (조직 타입별 예시)

  * academy: 원장 → 부원장 → 강사 → 조교
  * company: CEO → 팀장 → 팀원
  * store: 점주 → 매니저 → 직원 → 아르바이트
  * franchise: 본사 관리자 → 지점장 → 직원
  * hospital: 원장 → 과장 → 의사 → 간호사 → 직원

* 조직 내부 관리/감독 관계 모델링 필요

* 조직별 사용자 초대 기능 필요

* 조직별 권한 상속/위임 가능해야 함

* 다중 조직 소속 시 각 조직에서의 역할 독립적으로 관리 필요

---

## 3. 권한 시스템 요구사항

* RBAC (Role Based Access Control) 필요

  * OWNER
  * DIRECTOR / MANAGER
  * STAFF (강사, 직원, 의사 등 역할 담당자)
  * MEMBER / VIEWER
  * etc (조직 타입별 커스텀 역할 확장 가능해야 함)

* ABAC (Attribute Based Access Control) 필요

  * 특정 조건 기반 권한
  * 카테고리/부서 제한
  * 상태 기반 제한 (활성/정지/휴면)
  * 정책 기반 제한 (시간, 위치, 디바이스 등)

* ReBAC (Relationship Based Access Control) 필요

  * 조직 소속 관계
  * 상하 관계 (계층 기반)
  * 리소스 ownership 관계
  * 위임 관계

* `canXXX()` 형태의 centralized permission evaluation 구조 필요

* 모든 API 요청에서 runtime permission check 필요

* delegated permission (권한 위임) 필요

  * 예: 원장이 특정 강사에게 업로드 권한 부여
  * 예: 점주가 매니저에게 가격 수정 권한 임시 위임

* ownership 기반 권한 필요

  * 내가 생성한 리소스만 수정 가능

* 서비스별 권한 분리 필요

  * academy 권한
  * store/market 권한
  * hrm 권한
  * agent 권한

---

## 4. Subscription / Product 요구사항

* Product 개념 필요 (서비스 유형별 예시)

  | Product | 대상 조직 타입 |
  |---------|--------------|
  | Academy Pro | academy |
  | Store Basic / Store Pro | store, franchise |
  | HRM Essential | company, franchise |
  | CRM Plus | company, store, hospital |
  | Agent Pro | 전체 |
  | Market Seller | store, company |

* Product 별 서비스 접근 권한 필요

* Product 별 capability/feature 제한 가능해야 함

  * academy: 최대 강사 수, 최대 수강생 수, 강의 콘텐츠 업로드 용량
  * store: 최대 상품 수, 포스 단말기 수, 재고 관리 기능 여부
  * company/hrm: 최대 직원 수, 근태 리포트 기능 여부
  * agent: 최대 agent 수, 자동화 실행 횟수
  * 공통: 기능 활성화 여부, API 호출 quota

* Subscription 개념 필요

* Product 와 Subscription 은 분리 가능해야 함

* 하나의 Subscription 이 여러 Product 를 포함 가능해야 함

* 무료 Product / 프로모션 Product 지원 가능해야 함

* 결제 없이 Product Access 부여 가능해야 함 (어드민 수동 부여, 파트너십 등)

* Product 활성/비활성 상태 필요

* organization 단위 Product ownership 필요

---

## 5. 결제 (Billing) 요구사항

* 결제 시스템(PG) 연동 필요

* 국내/국외 PG 분리 가능해야 함

  * 국내: Toss Payments, KG이니시스, NHN KCP
  * 국외: Stripe, PayPal, Adyen

* PG 별 Adapter 구조 필요 가능성

* 결제 상태와 실제 권한 상태 분리 필요

* payment ledger 저장 필요

  * 결제 이력
  * 환불
  * 영수증
  * 정산 (프랜차이즈 본사/지점 정산 포함)

* webhook 기반 결제 처리 필요 가능성

* PG 중복 webhook 대응 필요

* idempotency 처리 필요

* 결제 완료 후 access/license 활성화 필요

* 환불/만료/grace period 처리 가능해야 함

---

## 6. Runtime Access 요구사항

* 실제 runtime access state 필요

* `canXXX()` 는 runtime access state 기준으로 판단해야 함

* 권한 체크는 billing ledger 직접 조회하면 안 됨

* organization 기반 access state 필요

* product 기반 access state 필요

* subscription 만료 시 즉시 권한 차단 가능해야 함

* 권한 변경 즉시 API 반영 가능해야 함

* organization 단위 access control 필요

---

## 7. 데이터베이스 요구사항

* MySQL 기반 구조 고려 (또는 PostgreSQL — Supabase 선택 시 Postgres 기본)

* 하나의 DB 서버 내 multiple database/schema 사용 고려

  * `identity_db` — 사용자, 조직, 인증
  * `billing_db` — 구독, 결제, 라이선스
  * `academy_db` — 강의, 수강생, 출결
  * `store_db` — 상품, 재고, POS
  * `hrm_db` — 직원, 근태, 급여
  * `agent_db` — AI 자동화 작업
  * etc

* User DB 분리 여부 고민 필요

* Billing DB 분리 여부 고민 필요

* FK 전략 고민 필요 (cross-db FK 미지원 시 application-level 보정)

* cross database transaction 전략 필요

* transaction isolation level 고려 필요

* dirty read 방지 필요

* runtime authoritative state 필요

* versioning / soft-delete 전략 필요 가능성

---

## 8. 인증 (Authentication) 요구사항

* **Firebase Authentication** 또는 **Supabase Auth** 사용

  | | Firebase Auth | Supabase Auth |
  |--|--------------|--------------|
  | Custom Token | 지원 | JWT 커스텀 클레임 |
  | Social Login | Google, Apple, 등 | Google, Apple, GitHub 등 |
  | Email/PW | 지원 | 지원 |
  | Phone OTP | 지원 | 지원 |
  | DB 연동 | 별도 구성 필요 | PostgreSQL 내장 |

* 인증 제공자를 인증 전용으로 사용 (authorization 은 내부 시스템)

* Custom Token / JWT 기반 세션 사용 예정

* 내부 User ID 와 Auth Provider UID 분리 필요

  * `firebase_uid` 또는 `supabase_uid` 를 PK 로 사용하지 않음

* 내부 PK 전략 필요

  * ULID (시간순 정렬 가능, UUID 대안)
  * BIGINT AUTO_INCREMENT
  * UUID v7 (최신 표준)

* email unique 처리 고민 필요

* external auth provider 종속 최소화 필요

* authorization 과 authentication 분리 필요

---

## 9. Frontend 권한 동기화 요구사항

* 권한 변경 즉시 Front 반영 필요

* Product/Subscription 변경 즉시 Front 반영 필요

* Front permission cache 필요 가능성

* Front permission invalidation 전략 필요

* stale permission refresh 전략 필요

* self-healing permission refresh 전략 가능성

  * 403 응답 기반 refresh
  * permission version 기반 refresh

* realtime UX 필요 가능성

* websocket / SSE 사용 가능성

* multi-user organization 환경에서 권한 동기화 필요

---

## 10. 이벤트 / 메시지큐 요구사항

* MQ (Message Queue) 도입 가능성

* Event Driven Architecture 고려 가능성

* 권한 상태 변경 이벤트 필요 가능성

* Product 활성화 이벤트 필요 가능성

* Subscription 변경 이벤트 필요 가능성

* Transactional Outbox Pattern 고려 가능성

* sync boundary / async boundary 분리 필요

* Strong Consistency 와 Eventual Consistency 분리 필요

  * Runtime Access State → strong consistency
  * notification / analytics → async 처리 가능

---

## 11. 실시간 UX 요구사항

* 권한 변경 후 즉시 UI 반영 필요

* subscription 만료 즉시 UX 반영 필요

* organization access 변경 즉시 반영 필요

* API 기준 authoritative permission 유지 필요

* frontend 는 cache/projection 역할 가능성

* multi-tab / multi-device 동기화 고려 가능성

* permission snapshot API 필요 가능성

---

## 12. 아키텍처 요구사항

* Enterprise SaaS 수준 authorization architecture 필요

* ERP/HRM 수준 organization modeling 필요

* Modular Monolith vs MSA 고민 필요

* API Gateway 가능성 고민

* Authorization Core 필요 가능성

* Runtime Permission Evaluation Layer 필요

* Product / License / Billing / Auth 분리 설계 필요

* 확장 가능한 permission system 필요

* 확장 가능한 organization graph 필요 (프랜차이즈 계층, 병원 다중 지점 등)

---

## 13. B2B API / Server-to-Server 요구사항

* B2B API 서비스 제공 가능해야 함

* 일부 고객은 직접 API 호출 가능해야 함

* Server-to-Server (S2S) 연동 지원 필요

* 외부 시스템 자동화 연동 가능해야 함

* API Key 기반 인증 필요 가능성

* Service Account 개념 필요 가능성

* Human User 와 Machine Identity 분리 필요

* Organization 단위 API Access 관리 필요

* Product 별 API Access 제한 가능해야 함

* API Scope 기반 권한 필요 가능성

  * `read`, `write`, `admin`, `webhook`, `automation`

* API Token 발급 / 회수 / 만료 필요

* Secret Rotation 가능해야 함

* Multi API Key 지원 가능해야 함

* API Key 별 권한 제한 필요 가능성

* API Key 별 접근 서비스 제한 가능해야 함

  * academy api, store api, hrm api, agent api, market api

* API 사용량 제한 (rate limit / quota) 필요 가능성

* Product 별 API quota 제한 가능해야 함

* API 사용량 billing 가능성 고려 필요

* Webhook 제공 가능성

* Webhook Signature Verification 필요

* 외부 시스템 callback 처리 필요 가능성

* Audit Log 필요

  * 어떤 API Key 가 어떤 요청을 수행했는지
  * 조직 타입별 감사 로그 요구사항 상이할 수 있음 (병원 HIPAA, 금융 등)

* API Access Revocation 즉시 반영 필요

* Organization subscription 만료 시 API access 차단 필요

* `canXXX()` 구조가 Human User 뿐 아니라 API Client 도 지원 가능해야 함

* Runtime authorization system 이 machine identity 도 지원해야 함

* OAuth Client Credentials Flow 가능성 고려 필요

* Internal API 와 External Public API 분리 필요 가능성

* API Versioning 전략 필요

* Tenant Isolation 필요

* Organization boundary 기반 API authorization 필요

* API Gateway 필요 가능성 증가

* Rate limiting / throttling 전략 필요

* Idempotency 지원 필요 가능성 (특히 payment/write API)

* Long-running async job API 필요 가능성

* Async processing status polling / webhook 구조 필요 가능성

* SDK / OpenAPI 제공 가능성 고려 필요

* External developer platform 가능성 고려 필요
