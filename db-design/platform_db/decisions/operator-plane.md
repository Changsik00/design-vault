---
type: decision
status: 채택
aliases:
  - 운영자 신원 평면
  - operator plane
  - 백오피스 권한 모델
  - operator role
tags:
  - platform-db
  - decision
  - operations
  - identity
  - security
---

# 운영자 신원 평면 — 백오피스 권한을 어디에 둘 것인가

> 상태: 채택 · 영역: 운영 가능성(Operability) · 형식: 비교 → 결정

## 결정

운영자(CS·FINANCE·SUPPORT·SECURITY·SRE·AUDITOR)는 **테넌트 `membership.platform_role`이 아니라 별도 신원 평면**(operator plane)으로 모델링한다. cross-tenant 도달은 [[cross-tenant-separation]]의 아키텍처 분리(`internal/`·admin 서비스) + [[break-glass]]로만 한다. **role flag로 권한을 우회하지 않는다.**

---

## 맥락 — 가장 큰 운영 구멍, 그리고 함정

지금 권한 모델엔 고객(OWNER/MEMBER/SERVICE)만 있다. 운영자가 없으니 현업에선 *"운영자 = OWNER"* 또는 *"직접 SQL"*로 수렴한다 — 가장 큰 운영 구멍. 그런데 운영자를 **어디에** 두느냐가 함정이다.

---

## 비교한 선택지

### A — `membership.platform_role`에 SUPER_ADMIN/CS/FINANCE 추가

- **장점**: 기존 3-gate 재사용, 구현 빠름
- **단점**: [[cross-tenant-separation]]이 **명시적으로 거부한 Admin-role 안티패턴 재도입** — 단일 장애점(탈취 시 전 테넌트 노출) · 비즈니스 로직에 보안 우회 혼재 · role 남용 확산 · 격리 불변식이 런타임 조건에 의존
- **기각** — 우리 자신의 결정을 깨뜨린다.

### B — 별도 운영자 신원 평면 (채택)

- 운영자 신원은 테넌트 `membership`과 **분리**(org에 안 묶임). **별도 인증**(전용 IdP / MFA 강제).
- `operator_role`(SUPER_ADMIN/CS/FINANCE/SUPPORT/SECURITY/SRE/AUDITOR) — 읽기/쓰기 범위 매트릭스. role→action은 코드 상수([[role-as-code]] 동일 원칙).
- cross-tenant 조회·override는 `internal/`·admin 서비스 **코드 경로에서만**(아키텍처 분리). 일반 API 토큰으론 도달 불가.
- 모든 운영자 행위 = `audit_log`에 `actor_type='OPERATOR'` + `support_action`/`break_glass` 플래그 **100%** 기록.
- **장점**: 고객 인증과 운영자 인증 분리(공격 표면 분리) · 격리는 코드 위치로 강제 · 최소권한 매트릭스
- **단점**: 신원 평면 1개 추가(`operator_account`/`operator_role`) · 별도 인증 인프라

### C — 외부 IAM/SSO 위임 + operator_role

- 운영자 *인증*은 회사 IdP(Okta 등), *인가*는 우리 `operator_role`. B의 전환형(컴플라이언스·운영자 증가 시).

| 기준 | A: tenant role | B: operator plane (우리) | C: IAM 위임 |
|---|---|---|---|
| cross-tenant-separation 준수 | ❌ 위반 | ✅ | ✅ |
| 공격 표면 | 단일 장애점 | 분리 | 분리 |
| 구현 비용 | 낮음 | 중간 | 높음 |
| 최소권한 | 어려움 | 매트릭스 | 매트릭스 |

---

## 왜 B인가

운영자 권한은 본질적으로 **cross-tenant**(전 고객 조회)다. 그걸 tenant role에 넣는 순간 격리 불변식이 무너진다. operator plane은 ① 인증 분리 ② 코드 경로 분리 ③ 전건 감사 ④ 최소권한 매트릭스로, 우리가 이미 선택한 [[cross-tenant-separation]]·[[break-glass]] 원칙과 정합한다.

### 운영자 역할 매트릭스 (계약)

| operator_role | 읽기 | 쓰기 | 비고 |
|---|---|---|---|
| SUPER_ADMIN | 전체 | 제한적(break-glass) | 2인 승인 |
| CS | org/user 조회 | ❌ | 상담 |
| FINANCE | billing 조회 | 환불 · entitlement 부여 | membership 수정 ❌ |
| SUPPORT | org/user 조회 | Support Action(제한) | |
| SECURITY | audit · api_key 조회 | 키 revoke | |
| SRE | 운영지표 · 헬스 | 일부 운영 | PII 최소 노출 |
| AUDITOR | audit 조회만 | ❌ | 불변 |

> 매트릭스 값은 코드 상수(`OPERATOR_PERMISSION[role]`)가 권위 — 테넌트 role과 동일하게 DB 레지스트리 회피([[role-as-code]]).

### 운영자 override 액션 계약

운영자/관리자 override 연산(`adminForceSubscriptionStatus`·`adminForceEntitlementStatus`·`adminSuspendUser`·`adminReactivateUser`·`adminRevokeApiKey`·`grantAdminAccess` 등)은 테넌트 우회로 상태를 강제하므로 **아래 계약을 모두 지킨다**:

1. **신원**: operator-plane 신원(`actor_type='OPERATOR'`)으로만 — 테넌트 `platform_role` 아님. 호출 가능 역할은 역할 매트릭스를 따른다(예: entitlement 강제=FINANCE, api_key revoke=SECURITY).
2. **race 방지**: 조건부 UPDATE(`WHERE 기대상태` + 영향행 검증) 또는 `SELECT … FOR UPDATE` 중 하나로 동시 override·사용자 작업과의 race를 막는다(불변식 #12, [[concurrency-control]]). *단, override는 "현재 상태 무시 강제"가 의도라 상태 전제 가드는 생략할 수 있다 — 대신 존재 확인 + 같은 행만 잠그면 충분.*
3. **스코프**: 단일 행을 결정하는 키로만 변경(불변식 #14). `adminForceEntitlementStatus`는 unique 키 `(org_pk, service)`로 — org는 서비스당 entitlement 1개라 한 행만 갱신된다(이슈 H4: 이전 `(org_pk, product_code)` 가정에서 발생하던 over-update가 키 정합으로 해소).
4. **outbox**: 같은 트랜잭션에서 `outbox_event` 발행(불변식 #13, [[outbox-pattern]] 레지스트리) — admin override도 검색 색인·캐시 무효화·알림 등 부수효과가 billing 경로와 동일하다(이슈 H1: admin 함수군 누락).
5. **감사**: `audit_log`에 `support_action=TRUE` + who(operator)/when/why(`meta_json`) 기록 — 컴플라이언스 필수([[audit-two-lane]] · [[break-glass]] 원칙). silent override 금지.
6. **멱등/재실행 안전**: 같은 override 재시도가 중복 부수효과를 만들지 않도록(이벤트 멱등키).

| 의무 | 근거 |
|---|---|
| operator-plane 신원으로만 호출 | actor_type='OPERATOR', 역할 매트릭스 |
| 조건부 UPDATE 또는 `FOR UPDATE` | 불변식 #12 · [[concurrency-control]] (race 방지) |
| 단일행 결정 키로만 변경 | 불변식 #14 (entitlement=`(org_pk, service)`, 이슈 H4) |
| 같은 트랜잭션 `outbox_event` 발행 | 불변식 #13, [[outbox-pattern]] (이슈 H1) |
| `audit_log` who/when/why 기록 | [[audit-two-lane]] · [[break-glass]], silent override 금지 |
| 멱등키로 재실행 안전 | 중복 부수효과 방지 |

**grantAdminAccess regrant 의미**(이슈 M4): 만료·취소된 접근을 재부여할 때 `updated_at`을 갱신하고 `created_at`은 최초 부여 시점으로 보존한다(또는 append-only 이력 모델이면 새 행 추가) — "언제 처음/다시 부여됐나"가 감사에서 추적되도록 한다. 빈 갱신(no-op) 금지.

**테스트 요지**: override 함수마다 ① 같은 트랜잭션 `outbox_event` INSERT 검증 ② 대상 부재 시 `NOT_FOUND` ③ `support_action=TRUE` 감사 row 기록을 확인한다. **🟡 알려진 갭**: override **멱등성(같은 상태로 두 번 force → outbox 1회)**은 아직 미검증 — consumer 멱등성으로 보강 필요. (실제 DB 검증법: [[testing-strategy]] · [[orm-testing-drizzle]])

---

## 트레이드오프

- 신원 평면이 둘(tenant + operator) → 약간의 복잡도. 대신 격리·감사·최소권한을 얻는다.
- 운영자 인증 인프라(MFA/IdP)가 필요하다.

## 전환 조건

- 운영자 수↑ / SOC2 등 컴플라이언스 → C(외부 IAM SSO로 인증 위임).

---

## 관련 문서

- [[cross-tenant-separation]] — 운영자 cross-tenant 도달이 따르는 아키텍처 분리
- [[break-glass]] — 운영자 긴급 쓰기(override) 경로
- [[role-as-code]] — operator_role→action 매핑도 코드 상수
- [[requirements]] — §6 운영 가능성 OPER-1·OPER-2·SUPP-1
> 소스 문서
- [[schema-reference]] — §H 보안, §M DB 계정 최소권한 (operator 계정 분리)
