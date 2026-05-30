---
type: core
aliases:
  - 운영 가능성
  - operability
  - 운영 모델
  - Operability Model
tags:
  - platform-db
  - core
  - operability
  - operations
  - reliability
---

# platform_db — 운영 가능성 (Operability)

> 설계 3종([[architecture]] 왜 · [[schema-reference]] 어떻게 · [[requirements]] 무엇)의 **4번째 축**.  
> 답하려는 질문: ***"이걸 3년 운영하면 무슨 사고가 나나, 그때 platform_db는 무엇을 제공하나."***
>
> **범위 규율**: 여기는 platform_db가 **소유하는 데이터·계약**만 둔다. 콘솔 UI·알림 임계치·DR 런북·Feature Flag rollout은 platform_db가 *enable*만 하고 *소유*는 별도 ops 제품. **단 Permission Debugger는 UI가 아니라 계약으로서 platform_db 책임**.  
> **상태**: 설계는 결정·explainer·스키마(§D.20 usage_snapshot·§D.21 operator)로 대부분 **확정/착수**, 구현(배치·콘솔)은 후속. 이 문서는 그 *계약*을 정의한다.
>
> 🧪 **테스트**: 이 운영 계약·안전장치를 *실제 DB·운영*에서 검증하는 법은 [[testing-strategy]](실제·운영 DB 테스트)·[[orm-testing-drizzle]](ORM).

---

## O1. Operator Plane — 운영자 신원·권한

> 결정: [[operator-plane]] (운영자는 **테넌트 role이 아니라 별도 신원 평면**).

운영자(CS·FINANCE·SUPPORT·SECURITY·SRE·AUDITOR)를 `membership.platform_role`에 넣으면 [[cross-tenant-separation]]이 거부한 Admin-role 안티패턴이 된다. 대신 별도 평면:

```
operator_account   (운영자 신원 — 테넌트 membership과 분리, 별도 인증/MFA)
operator_role      (SUPER_ADMIN/CS/FINANCE/SUPPORT/SECURITY/SRE/AUDITOR — 최소권한 매트릭스)
operator_session   (운영자 세션 — 짧은 TTL, 행위 추적)
```

- 권한 매트릭스(읽기/쓰기 범위)는 [[operator-plane]] 참조. role→action은 코드 상수([[role-as-code]]).
- cross-tenant 도달 = `internal/`·admin 서비스 코드 경로 + [[break-glass]]만. 일반 API 토큰 불가.
- 운영자 행위는 **컴플라이언스 audit 100%**(`actor_type='OPERATOR'`, [[audit-two-lane]]).

**현재**: 🟡 (operator-plane 결정 + operator 테이블 [[schema-reference]] §D.21 설계 / 인증 인프라는 코드 트랙).

---

## O2. Supportability — 운영자가 1초에 답을 얻는 계약

### Permission Debugger *(ROI 최상 · platform_db 책임)*

운영자 질문: *"이 유저가 왜 `PUBLISH_VIDEO`가 안 되죠?"* — 지금은 membership·service_membership·delegation_grant·org_entitlement를 **수동으로 뒤져야** 한다. 그건 설계 실패에 가깝다.

→ platform_db는 **단일 설명 함수**를 제공한다:

```
explainPermission(userPk, orgPk, service, action, resource) → {
  gateA: { pass: true,  reason: "membership ACTIVE" },
  gateB: { pass: false, reason: "entitlement EXPIRED",
           entitlement: { status:"EXPIRED", validUntil:"2026-04-30" } },
  gateC: { pass: null,  reason: "SKIP (B에서 차단)" }
}
```

- 입력 = 권한 판정 입력 그대로, 출력 = **gate별 통과/사유 trace**.
- UI는 ops 제품 소관. 그러나 **이 trace 계약은 platform_db가 제공**한다(`@aiagent/db-platform`의 함수로).

### Entitlement Debugger

"왜 이 org가 차단됐나" → entitlement status·valid_until·grace_until·source + 최근 `billing_event` + 최근 `pg_webhook_event`를 한 번에. (billing 흐름의 어디서 멈췄는지)

### Webhook Replay

`pg_webhook_event`에 수신 webhook이 전부 보존됨 → 운영자가 `FAILED`/`SKIPPED` webhook을 **재처리**(멱등키 `UNIQUE`로 안전, [[idempotency-key]]). "결제 성공했는데 webhook 유실" 복구 경로.

### Manual Recovery (Support Action)

운영자 override(entitlement 강제 부여·Trial 연장·구독 정지)는 **Support Action 경로**로만 — 별도 권한(FINANCE 등) + `who/when/why` 컴플라이언스 audit 필수. "결제했는데 권한 안 열림"의 운영자 복구.

### 수동 진단·복구 절차 (Debugger 미작성 시 임시)

**권한이 갑자기 안 먹힐 때**:
```
1. identity_user.status, membership.status 확인
2. org_entitlement.status / valid_until → GRACE/EXPIRED 전환 여부
3. org.perm_version vs 클라이언트 X-Perm-Version 비교
4. Redis 캐시 flush: DEL "perm:{user_pk}:{org_pk}"
5. audit_log.result='DENY' 최근 30분 조회
```

**결제는 됐는데 권한이 안 풀릴 때**(Webhook Replay 전 수동):
```
1. payment_ledger에서 idempotency_key로 SUCCEEDED 확인
2. outbox_event.status='PENDING'인 subscription.activated 조회
3. outbox 워커 재시도 또는 수동 org_entitlement UPDATE + perm_version 갱신
```

**현재**: 🟡 (debug 계약 [[permission-debugger]]·support_action §D.8 설계 / 구현은 후속 — 그 전까진 수동).

---

## O3. Data Lifecycle — 보존·파기

테이블별 보존정책이 없으면 audit/chat/usage가 무한 누적 → 나중에 무조건 터진다.

| 데이터 | 보존 | 파기 방법 | 근거 |
|---|---|---|---|
| `audit_log` (컴플라이언스) | 5년 | 월별 파티션 DROP([[partitioning]]) | ISMS-P |
| `user_consent_event` | 5년 | append-only, 5년 후 파기 | PIPA |
| `payment_ledger` | 5~7년(회계 검토) | append-only | 세법/전자상거래법 |
| `pg_webhook_event` | 90일 | sweeper DELETE | 운영 |
| `outbox_event` | PROCESSED 후 30일 | sweeper DELETE([[outbox-pattern]]) | 운영 |
| access 텔레메트리 | 30~90일 | OLAP TTL([[audit-two-lane]]) | 운영 |
| `identity_user` (탈퇴) | DELETED 후 30일 → 익명화 | hard anonymize 배치 | PIPA |
| chat/usage 이벤트 | 1년 → S3 아카이브 | 서비스 DB 소관 | 비용 |

- **파기 트리거**: 월별 파티션 배치 · sweeper job(PROCESSED 정리) · 탈퇴 30일 배치.
- **현재 🟡**: 보존 정책·매트릭스는 설계 완료([[data-lifecycle-retention]]), sweeper·retention·파티션 자동 추가 *배치*만 미작성([[schema-reference]] §D.8). → 이 표가 그 응집점.

### 정기 배치 (platform_db 위에서 도는 운영 작업)

| 작업 | 주기 | 방법 |
|---|---|---|
| TRIALING→EXPIRED 전환 | 일 1회 | `UPDATE org_entitlement SET status='EXPIRED' WHERE valid_until<NOW() AND status='TRIALING'` |
| 만료 구독 정리 | 일 1회 | `... status='EXPIRED' WHERE valid_until<NOW() AND status='ACTIVE'` |
| GRACE 전환 | 일 1회 | `valid_until<NOW() AND grace_until>NOW() → status='GRACE'` |
| 만료 초대 토큰 정리 | 일 1회 | `membership_invite SET status='EXPIRED' WHERE expires_at<NOW() AND status='PENDING'` |
| audit_log 파티션 추가 | 분기 1회(→월별 자동화 필요) | `ALTER TABLE audit_log REORGANIZE PARTITION p_future ...` |
| outbox/webhook sweeper | 일 1회 | PROCESSED + 30/90일 경과 DELETE |
| 해시 사슬 검증 | 월 1회 | audit_log + user_consent_event 재계산(해시 컬럼 활성화 후) |

> ⚠️ 위 배치 다수가 **현재 미작성/수동**. 미작성 시 무한 TRIALING·파티션 과부하·로그 누적이 발생.

---

## O4. Usage & Metering — 현재 얼마 썼나

`feature_limits`로 한도는 정의되는데 ***현재 사용량을 모른다*** — "학생 500명 제한"인데 지금 498인지 5인지 백오피스가 알 수 없다. 운영 치명상.

**원칙**: 이벤트 전량 = 서비스 DB / **집계 = platform `usage_snapshot`**.

```
usage_snapshot (org_pk, service, metric, period, used, limit, source_ts)
  - 서비스가 주기적으로 집계 push (또는 platform이 pull)
  - feature_limits 대비 used/limit 즉시 가시화 (498/500)
```

- **한도 실시간 enforcement 카운터는 서비스측**(ABAC-3) — 실시간 차단은 빠른 서비스 카운터, *가시성·과금*은 platform.
- **과금형(metered billing)**: 정산용 `usage_event`를 platform이 받음(정확성 요구↑). usage_snapshot과 동일 경로.

**현재**: 🟡 (snapshot 테이블 [[schema-reference]] §D.20 설계 / 집계 push 배치 미작성).

---

## O5. Reliability Contract — 의존성이 죽으면

장애는 "정상 동작"이 아니라 "무엇이 어떻게 죽고 무엇이 버티나"다. 의존성별 platform 계약:

| 죽으면 | 영향 | 계약 (버티는 법) |
|---|---|---|
| **MySQL(platform_db)** | 전 서비스 auth 불가 (SPOF) | read replica로 read 지속(NFR-3 트리거 T2). RTO/RPO·백업은 ops 소관이나 **단일 인스턴스=SPOF임을 명시**. |
| **Redis** (권한 캐시) | 캐시 미스 → DB 직격(느려짐) | **fail-open 아님** — 캐시는 가속용, 권위 아님. 없으면 DB 조회로 *느려도 정상 동작*. |
| **Firebase** | 신규 로그인 불가 | 기존 JWT(TTL 1h)는 유효 → 진행 세션 영향 적음. `firebase_uid`는 조회 키라 vendor 교체 가능([[firebase-boundary]]). |
| **PG webhook 유실/지연** | entitlement 갱신 지연 | Gate B `validUntil` 복합체크가 2차 방어([[decisions/gate-b-billing-grace|gate-b-billing-grace]]). + reconciliation 폴링(🟡 배치 미작성) + Webhook Replay(O2). |
| **Qdrant/Neo4j** | RAG 검색 불가 | platform auth와 무관(서비스 도메인). org 격리만 보장([[rag-multitenancy]]). |

- **entitlement 가용성(AVAIL-1)**: entitlement는 billing 장애와 **구조적으로 격리**됨([[auth-projection]]) — billing/PG가 죽어도 *기존 entitlement read는 영향 없음*. "결제 시스템 장애 시 기존 권한 최소 N시간 유지"의 근거가 이미 설계에 있다(명문화만 필요).

**현재**: 🟡 (auth-projection·validUntil ✅; reconciliation 폴링은 배치 미작성, 단일 MySQL SPOF는 트리거 시 분리).

---

## O6. Observability — 무엇을 봐야 하나

3-gate를 만들었으면 반드시 관측해야 할 KPI. (지표 *방출*은 앱/관측 파이프라인, *무엇을 보나*는 platform_db 설계가 정의.)

| 축 | 지표 |
|---|---|
| Gate 건전성 | GateA/B/C **실패율** (서비스별) · DENY 급증 |
| 레이턴시 | permission 평가 · **entitlement 조회** · membership 조회 P99 |
| audit | 컴플라이언스 audit volume · 텔레메트리 volume([[audit-two-lane]]) |
| billing | **webhook delay**(수신→PROCESSED) · webhook 실패율 · 결제 실패율 |
| 이상 | api_key 이상 사용 · 특정 org 과금 폭증 · GateB 실패 급증 |

- 이 지표가 알림(Alert) 임계치로 이어지나, **임계치·알림 메커니즘은 ops 제품 소관**. platform_db는 *측정 가능하도록 데이터를 노출*할 책임.

**현재**: 🟡 (KPI 정의 [[observability-slo]] 설계 / 노출·대시보드는 후속).

---

## 종합 — 운영 성숙도

| 관점 | 현재 | 이 문서로 |
|---|---|---|
| 개발자 | 90 | 90 |
| 아키텍트 | 80 | 85 |
| **운영자** | **60** | **85 (목표)** |

가장 부족한 건 스키마가 아니라 **운영 모델**이었다. O1~O6은 그 모델의 계약을 정의한다 — 구현 우선순위는 [[requirements]] §4 운영 가능성 표 참조.

## 관련 문서

- [[operator-plane]] · [[audit-two-lane]] — O1·O2·O6의 핵심 결정
- [[requirements]] — §4 운영 가능성 추적표(OPER/SUPP/DBG/RETN/AVAIL/OWN/USAGE)
- [[auth-projection]] · [[decisions/gate-b-billing-grace|gate-b-billing-grace]] — O5 신뢰성 계약의 설계 근거
