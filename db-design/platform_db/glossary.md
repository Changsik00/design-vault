---
type: index
aliases:
  - 용어집
  - glossary
  - terms
  - 약어
  - dictionary
tags:
  - platform-db
  - index
  - meta
---

# 📖 용어집 (glossary)

이 vault에서 쓰는 약어·용어 사전. **각 용어를 클릭하면 그 용어를 다루는 문서로 이동**한다(지식그래프에도 연결로 나타남).

> 🔎 모르는 약어는 `Ctrl/Cmd+F`로 검색. 더 깊은 설명은 각 문서의 `## 용어 정리` 절.
> 📄 설계 문서(코드 없음) · 🐘 DDL은 PostgreSQL 1순위.

## 🔤 약어 (A–Z)

- **[[role-capability|ABAC]]** (Attribute-Based Access Control) — 속성(소유·한도·테넌트·IP)으로 인가
- **[[consistency-model|ACID]]** (Atomicity·Consistency·Isolation·Durability) — DB 트랜잭션 4대 보장
- **[[bola-object-authz|BFLA]]** (Broken Function Level Authorization) — 기능/엔드포인트 단위 인가 누락 (OWASP API#5)
- **[[bola-object-authz|BOLA]]** (Broken Object Level Authorization) — 객체(1건) 단위 인가 누락 (OWASP API#1)
- **[[consistency-model|CAP]]** (CAP theorem) — 분산 시 일관성·가용성·분단내성 중 둘만
- **[[concurrency-control|CAS]]** (Compare-And-Set) — 전제를 WHERE에 박은 조건부 UPDATE로 race 방지
- **[[casl-ability|CASL]]** — JS 권한 라이브러리 (ability·`can()`)
- **[[machine-identity-apikey|CIDR]]** — IP 대역 표기 (`203.0.113.0/24`)
- **[[least-privilege-grant|DDL]]** (Data Definition Language) — CREATE/ALTER/DROP (migrator만 보유)
- **[[secret-encryption|DEK · KEK]]** (Data / Key Encryption Key) — 봉투 암호화의 2겹 키
- **[[json-column|GIN]]** (Generalized Inverted Index) — JSONB `@>` 포함 검색 인덱스
- **[[machine-identity-apikey|HMAC]]** — 공유비밀 서명 검증 (webhook 수신용; api_key 인증엔 미사용)
- **[[bola-object-authz|IDOR]]** (Insecure Direct Object Reference) — BOLA의 옛 이름·부분집합 ("DB 키 노출 해킹")
- **[[break-glass|ISMS-P]]** — 정보보호 관리체계 인증(KR) — audit 5년·긴급접근 감사 요구
- **[[architecture|ISO 27001]]** (ISO/IEC 27001:2022) — 정보보안 경영시스템 — Annex A 기술통제 근거
- **[[perm-version-propagation|JWT]]** (JSON Web Token) — claims 담은 토큰 (Firebase TTL ~1h)
- **[[secret-encryption|KMS]]** (Key Management Service) — 마스터키(KEK) 보관·관리
- **[[orm-testing-drizzle|N+1]]** — 부모1+자식N 쿼리 폭발 함정
- **[[secret-encryption|PII]]** (Personally Identifiable Information) — 개인식별정보 (PIPA 보호 대상)
- **[[pipa-consent|PIPA]]** (Personal Information Protection Act) — 한국 개인정보보호법
- **[[role-capability|RBAC]]** (Role-Based Access Control) — 역할(`role_code`→`ROLE_PERMISSION`)로 인가
- **[[rebac-delegation|ReBAC]]** (Relationship-Based Access Control) — 관계·위임(`delegation_grant`)으로 인가
- **[[multitenancy-rls|RLS]]** (Row-Level Security) — DB가 행 단위 테넌트 격리를 강제
- **[[observability-slo|SLI · SLO · SLA]]** — 측정값 / 내부목표 / 외부계약
- **[[break-glass|SOC2]]** — 보안 운영 통제 인증 (break-glass 점검)
- **[[observability-slo|SPOF]]** (Single Point of Failure) — platform_db = 인가 단일 장애점
- **[[architecture|SSOT]]** (Single Source of Truth) — 단일 진실 원천
- **[[perm-version-propagation|TTL]]** (Time To Live) — 캐시·토큰 유효 수명
- **[[pk-ulid-strategy|ULID]]** (Universally Unique Lexicographically Sortable ID) — 시간순 정렬 26자 외부 식별자
- **[[audit-hash-chain|WORM]]** (Write Once Read Many) — 한 번 쓰면 수정·삭제 불가 (append-only)
- **[[consistency-model|2PC]]** (Two-Phase Commit) — 분산 트랜잭션 프로토콜 (단일 DB라 미사용)

## 🔐 인증·인가

- **[[bola-object-authz|인증 (authN)]]** — "너 누구냐", 신원 확인(Firebase)
- **[[bola-object-authz|인가 (authZ)]]** — "너 이거 해도 되냐", 권한 확인(3-gate)
- **[[gate-abc-flow|3-gate]]** — A(소속)·B(이용권)·C(정책) 순차 단락 인가
- **[[gate-abc-flow|Gate A · 소속]]** — `(user,org)` ACTIVE membership 검증 (실패 403)
- **[[gate-abc-flow|Gate B · 이용권]]** — `org_entitlement` status·valid_until 검증 (실패 402)
- **[[gate-abc-flow|Gate C · 정책]]** — CASL로 RBAC·ABAC·ReBAC 조합 판단 (실패 403)
- **[[casl-ability|ability / can()]]** — 규칙 모은 객체 / 허용여부 질의(boolean)
- **[[role-capability|ROLE_PERMISSION]]** — 역할→동사 매핑 코드 상수 (DB 저장 금지)
- **[[role-capability|platform_role]]** — 테넌트 소속 권위 (OWNER/MEMBER/SERVICE)
- **[[role-capability|role_code]]** — 서비스 내 도메인 역할 (DIRECTOR/TEACHER…)
- **[[role-capability|capability]]** — 행동 하나를 1:1 위임 (`<service>.<action>`)
- **[[rebac-delegation|위임 (delegation)]]** — 권한 일부를 한시적으로 빌려줌 (grantor→grantee)
- **[[rebac-delegation|임퍼소네이션]]** (impersonation) — 그 사람이 *되는* 것 — 행위자 위조, 명시적 금지
- **[[rebac-delegation|Zanzibar]]** — 구글 범용 ReBAC(relation tuple), 현재 보류
- **[[fail-closed|fail-closed]]** — 불확실·장애 시 "차단" 쪽으로 (인가 기본값)
- **[[fail-closed|deny-by-default]]** — 명시적 allow 없으면 거부
- **[[fail-closed|allow-list / deny-list]]** — 통과 목록 / 차단 목록 (인가엔 allow-list)
- **[[machine-identity-apikey|머신 신원 / api_key]]** — 프로그램 신원(`type='SERVICE'`) / 발급 비밀키
- **[[machine-identity-apikey|Bearer 토큰]]** — `Authorization: Bearer <키>` 인증 방식
- **[[machine-identity-apikey|key_prefix / secret_hash]]** — 키 식별 평문 / bcrypt 해시(평문 1회 노출)
- **[[machine-identity-apikey|key rotation / revoke]]** — 정기 교체(90일) / 즉시 폐기(`revoked_at`)
- **[[machine-identity-apikey|Confused Deputy]]** — 권한 있는 정직한 시스템을 속여 권한 차용
- **[[perm-version-propagation|perm_version]]** — 권한 변경 카운터 — bump 시 기존 토큰 stale
- **[[perm-version-propagation|@VerifyOnDb]]** — 민감 쓰기 직전 DB 재검증 (stale 토큰 즉시 차단)
- **[[perm-version-propagation|X-Perm-Version / forceRefresh]]** — 현재 버전 헤더 / 토큰 강제 재발급
- **[[perm-version-propagation|thundering herd]]** — 동시 갱신 몰림 부하 (user_pv/org_pv 분리로 완화)

## 🏢 멀티테넌시·격리

- **[[multitenancy-rls|멀티테넌트 / Pool 모델]]** — 공유 DB + `org_pk` 행 격리
- **[[multitenancy-rls|org_pk]]** — 테넌트 격리 키, 모든 도메인 테이블 NOT NULL (불변식 #3)
- **[[multitenancy-rls|SET LOCAL app.org_pk]]** — 서버가 트랜잭션마다 주입하는 RLS 세션 변수
- **[[bola-object-authz|테넌트 (tenant)]]** — 분리 단위 = `organization`
- **[[casl-ability|소유권 (ownership)]]** — `owner_pk == principal` (ABAC의 한 형태)
- **[[bola-object-authz|defense-in-depth]]** — 심층 방어, 한 겹 뚫려도 다음 겹이 막음
- **[[bola-object-authz|enumeration]]** — id를 1씩 늘려 긁는 공격 (ULID로 방어)
- **[[rag-multitenancy|RAG 격리]]** — 벡터(Qdrant)·그래프(Neo4j) 공유 + `org_id` 격리

## 💳 결제·구독·일관성

- **[[gate-b-entitlement|entitlement]]** (org_entitlement) — org 런타임 접근 권한·한도 SSOT, Gate B가 읽음. unique `(org_pk, service)`
- **[[feature-limits|feature_limits]]** — org 기능 한도(JSONB), 런타임 최종 권위 (불변식 #10)
- **[[subscription-lifecycle|org_subscription]]** — PG 결제 계약 상태 (TRIALING/ACTIVE/PAST_DUE/CANCELED/EXPIRED)
- **[[explainers/auth/gate-b-billing-grace|GRACE / grace_until]]** — 결제 실패 후 서비스 유지 유예
- **[[explainers/auth/gate-b-billing-grace|valid_until]]** — 만료 시각 안전망, status만 보면 영구 무료 위험 (불변식 #9)
- **[[idempotency-key|멱등성 / idempotency_key]]** — 여러 번 호출해도 결과 동일 / payment_ledger UNIQUE
- **[[webhook-processing|웹훅 / 서명 검증]]** — PG가 결과를 우리 URL로 POST / 진위 확인(HMAC)
- **[[outbox-pattern|Outbox 패턴]]** (outbox_event) — 부수효과를 같은 트랜잭션에 기록 후 비동기 발송
- **[[outbox-pattern|aggregate_pk]]** — outbox 이벤트가 *관한 엔티티*의 pk (환불→`payment_ledger.pk`, org_pk 아님)
- **[[consistency-model|강한 일관성]]** — 쓴 즉시 최신값(read-after-write) — 결제→권한
- **[[consistency-model|결과적 일관성]]** — 잠깐 불일치해도 수렴 — 알림·집계

## 🗄️ 데이터 모델·동시성

- **[[pk-ulid-strategy|BIGINT pk / public_id]]** — 내부 JOIN 전용 정수 PK / 외부 노출 ULID
- **[[delete-patterns|soft delete]]** (deleted_at) — 행 보존 + 삭제 시각 표시(복원 가능)
- **[[delete-patterns|append-only]]** — UPDATE/DELETE 없이 새 row로만 이력 적재
- **[[enum-vs-varchar-check|ENUM vs VARCHAR+CHECK]]** — 전역 타입 대신 테이블 로컬 CHECK (확장 안전, D6)
- **[[json-column|JSONB / GIN]]** — 이진 JSON 컬럼 / `@>` 검색 인덱스
- **[[fk-strategy|FK (외래키)]]** — 참조 존재 보장(고아 방지; cross-schema는 끊음)
- **[[index-design|인덱스 (B-Tree) / 커버링]]** — 정렬 구조로 풀스캔 회피 / INCLUDE로 본체 접근 생략
- **[[partitioning|파티셔닝 / 프루닝]]** — 선언적 RANGE + PARTITION OF / 해당 파티션만 스캔
- **[[partitioning|복합 PK `(pk, created_at)`]]** — 파티션 테이블은 PK·UNIQUE에 파티션 키 포함 필수
- **[[online-ddl-migration|online DDL / CONCURRENTLY]]** — 무중단 스키마 변경 / 비차단 인덱스 생성
- **[[concurrency-control|race condition / lost update]]** — 동시 수정 충돌 / read-then-write 덮어쓰기
- **[[concurrency-control|비관적 락 (FOR UPDATE)]]** — 충돌 가정, 행 미리 잠금(대기)
- **[[concurrency-control|낙관적 락 (version)]]** — 충돌 드묾 가정, 저장 시 version 확인
- **[[concurrency-control|SKIP LOCKED]]** — 잠긴 행 건너뛰기 — 워커 작업 분배(outbox)
- **[[concurrency-control|데드락 (deadlock)]]** — 상호 락 대기 — PG가 감지해 한쪽 abort

## 🛡️ 보안·암호화

- **[[least-privilege-grant|least privilege (최소권한)]]** — 꼭 필요한 권한만 부여
- **[[least-privilege-grant|GRANT / REVOKE]]** — DB 권한 부여 / 회수
- **[[least-privilege-grant|INSERT-only 계정]]** — INSERT만 가진 전용 계정 (`audit_append` 등)
- **[[least-privilege-grant|separation of duties]]** — 상충 권한(추가 vs 수정)을 한 계정에 안 몲
- **[[least-privilege-grant|blast radius]]** — 계정 탈취 시 피해 범위 (최소권한이 축소)
- **[[secret-encryption|해시 / 암호화]]** — 단방향(복원 불가) / 양방향(복호 가능)
- **[[secret-encryption|bcrypt / salt]]** — salt 내장 단방향 해싱 / rainbow table 방어 무작위값
- **[[secret-encryption|봉투 암호화 (envelope)]]** — DEK로 데이터, KEK로 DEK를 감싸는 2겹
- **[[secret-encryption|at-rest / in-transit]]** — 저장 중(KMS·bcrypt) / 전송 중(TLS) 암호화
- **[[schema-reference|BYTEA]]** — 이진 컬럼 표준 타입 (MySQL은 VARBINARY)

## ⚖️ 컴플라이언스·감사

- **[[audit-hash-chain|해시 체인]]** (tamper-evident) — 각 행이 직전 해시 포함 → 변조·삭제 사후 탐지
- **[[audit-hash-chain|prev_hash / row_hash]]** — 삭제·순서조작 감지 / 행 자체 변조 감지
- **[[audit-hash-chain|pgcrypto]]** — PG 내장 암호화 확장 (SHA-256 DB 네이티브)
- **[[audit-hash-chain|S3 Object Lock]]** — root도 못 지우는 외부 WORM (3차 방어)
- **[[pipa-consent|전자서명법]]** — 약관 체크박스 동의 = 서면 서명 효력 (§3)
- **[[pipa-consent|consent 이벤트]]** (user_consent_event) — append-only 동의/철회/재동의 이력
- **[[pipa-consent|제3자 제공 동의]]** (§17) — recipient/purpose/items/retention 4요건
- **[[pipa-consent|14세 미만 동의]]** (§22) — 법정대리인 동의 법적 필수
- **[[data-lifecycle-retention|retention / purge / anonymize]]** — 보존 / 파기(파티션 DROP) / 식별 컬럼 NULL화
- **[[data-lifecycle-retention|sweeper]]** — 처리 끝난 운영 데이터 주기 청소 배치

## 🛠️ 운영·관측·테스트

- **[[operator-plane|operator-plane]]** — 운영자는 테넌트 role이 아닌 별도 신원 평면
- **[[break-glass|break-glass]]** — 절차로만 허가되는 긴급 접근 (silent 금지·전건 audit)
- **[[schema-reference|support_action]]** — 운영자 override를 표식하는 audit 컬럼 (SUPP-1)
- **[[observability-slo|audit 2-lane]]** — 컴플라이언스 audit(영구·WORM) vs access 텔레메트리(샘플·단기)
- **[[observability-slo|observability]]** (metrics/logs/traces) — 외부 신호로 내부 상태 추론
- **[[observability-slo|error budget]]** — SLO 뒤집은 허용 실패량 (소진 시 배포 동결)
- **[[observability-slo|P99]]** — 100건 중 99건 응답시간 (꼬리 지연)
- **[[observability-slo|trace_id]]** — 한 요청을 GateA→B→C로 묶는 추적 키 (AUD-3)
- **[[permission-debugger|Permission Debugger]]** (DBG-1) — 거부 요청에 Gate별 결과+사유 1초 trace
- **[[orm-testing-drizzle|ORM / Drizzle / drizzle-kit]]** — 타입안전 DB 계층 / 그 CLI(generate·migrate·push)
- **[[orm-testing-drizzle|drift]]** (migration drift) — 스키마 코드 ≠ 실제 DB → 거짓 통과
- **[[orm-testing-drizzle|base repo]]** — org_pk 자동 주입으로 BOLA 방어 프레임워크화
- **[[orm-testing-drizzle|Testcontainers]]** — 일회용 실 Postgres 컨테이너(테스트)
- **[[testing-strategy|per-test rollback]]** — 테스트를 트랜잭션으로 감싸 끝에 rollback 격리
- **[[testing-strategy|synthetic 테넌트]]** — 운영 DB 내 전용 `org_pk` 테스트 조직
- **[[testing-strategy|shadow / canary]]** — 병렬 비교 배포 / 소수 트래픽 점진 배포
- **[[rate-limiting|rate limiting / quota]]** — 빈도 제한(Gateway 강제) / 누적 총량(entitlement)
- **[[usage-metering|usage_snapshot]]** — 사용량 집계 스냅샷 (가시성용, 실시간 아님)

## 🧩 공통·스키마·거버넌스

- **[[architecture|SSOT]]** (Single Source of Truth) — 단일 진실 원천(entitlement·firebase_uid·display_name)
- **[[architecture|불변식 (invariant)]]** — 코드 검증·위반 PR 반려하는 설계 규칙(15개)
- **[[CONVENTIONS|설계 성숙도 (✅🟡🔴)]]** — 구현 여부 아닌 설계 진척 — 확정/부분/미설계
- **[[pk-ulid-strategy|firebase_uid]]** — Firebase 발급 사용자 식별자 (조회 키, PK/FK 아님)
- **[[schema-reference|TIMESTAMPTZ]]** — 시각 컬럼 표준 (UTC, `DEFAULT now()`)
- **[[schema-reference|GENERATED ALWAYS AS IDENTITY]]** — 내부 PK 표준(BIGINT)
- **[[CONVENTIONS|ID 규약 `<분류>-<번호>`]]** — 접두사=의미, 번호=고정 참조 핸들(빈틈 유지)

---

> 새 용어를 정의했으면 여기에 한 줄(용어에 링크) 추가하고 [log](../../log.md)에 기록한다 (운영 규약: [WIKI](../../WIKI.md)).
