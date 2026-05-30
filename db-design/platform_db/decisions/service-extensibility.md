---
type: decision
status: 채택
aliases:
  - 서비스 확장 결합
  - service CHECK 허용목록
  - capability CHECK 결합
  - EXT 결정
tags:
  - platform-db
  - decision
  - extensibility
  - authorization
  - boundary
---

# 서비스 확장 결합 — service·capability 허용목록을 어디에 둘 것인가

> 상태: 채택 (의도적 트레이드오프, 전환 트리거까지 유지) · 영역: 플랫폼 확장성(EXT) · 형식: 비교 → 결정

## 결정

`org_entitlement.service`·`product.service`와 `delegation_grant.capability`의 허용목록을 **당분간 DB `CHECK`로 유지**한다(Option A). 새 서비스 추가 시 **온라인 CHECK 마이그레이션 1건**을 *명시적·의도적 트레이드오프로* 감수한다. 2번째 서비스가 실재하고 churn이 생기면 코드/데이터 기반(B/C)으로 전환한다.

> [[requirements]] §1 EXT-1·EXT-2가 가리키던 "결정 필요"를 여기서 종결한다. 애매하게 두지 않는 게 핵심이었다.

---

## 맥락 — 목적과의 충돌

platform_db의 목적은 **"새 서비스 추가 한계비용 최소화"**(EXT). 그런데 service·capability를 CHECK로 하드코딩하면 서비스 추가 = platform 마이그레이션 → 목적과 충돌한다. 그럼 지금 끊을 것인가?

**중요한 사실**: D6 결정으로 이 컬럼들은 ENUM이 아니라 **VARCHAR+CHECK**다. CHECK 변경은 **온라인 DDL(테이블 락 없음)** — 즉 서비스 추가 비용은 *"1줄 마이그레이션 작성"*이지 *"다운타임"*이 아니다. "🔴 재앙"이 아니라 "저비용 결합"이 정확한 평가다.

---

## 비교한 선택지

### A — CHECK 허용목록 유지 (채택)

- **장점**: DB 레벨 허용목록(오타·rogue service INSERT 차단 안전 게이트) · 온라인 DDL이라 추가 비용 낮음 · 지금 변경 0
- **단점**: 서비스마다 platform 마이그레이션 1건 · 코어가 서비스 이름/어휘를 "앎"(순수 service-agnostic 아님)

### B — 자유 VARCHAR + 코드/앱 검증

- **장점**: 서비스 추가 시 platform DDL **0**(진짜 service-agnostic) · capability는 이미 [[role-as-code|`ROLE_PERMISSION` 코드 상수]]가 권위라 CHECK는 사실상 **중복 안전망**
- **단점**: DB 레벨 안전 게이트 상실(잘못된 service 값이 INSERT됨 — 단 entitlement 평가에서 어차피 실패) · 검증 책임이 코드로 이동

### C — `service_registry` 테이블 (데이터 기반)

- **장점**: DDL 없이 행 INSERT로 서비스 추가 + DB 레벨 무결성(FK)
- **단점**: 테이블·조인 추가 · 현 규모엔 과설계(YAGNI)

| 기준 | A: CHECK (우리) | B: 코드 검증 | C: 레지스트리 |
|---|---|---|---|
| 서비스 추가 비용 | 온라인 마이그 1건 | **0** | 행 INSERT |
| DB 레벨 안전망 | 있음 | 없음 | 있음(FK) |
| 코어 service-agnostic | △ | ○ | ○ |
| 현 규모 적합 | ○ | ○ | 과설계 |

---

## 왜 A인가 (지금)

1. **비용이 실제로 낮다** — 온라인 CHECK 변경. "재앙"이 아니라 1줄 마이그레이션.
2. **CHECK는 의도적 allowlist 안전장치** — 2번째 서비스가 없는 지금 굳이 제거할 이유가 약하다.
3. **YAGNI** — [[role-as-code]]·도메인 바운디드 컨텍스트 원칙대로, 2번째 서비스가 실재하기 전 레지스트리/추상화는 과설계.
4. **capability는 이미 코드가 진짜 권위** — CHECK는 중복 안전망일 뿐이라 끊어도 손실은 적지만, *지금 끊을 이득도 적다.*

---

## 트레이드오프 — 우리가 받아들인 것

- platform_db 코어가 서비스 이름과 academy capability 어휘를 "안다" — **완전한 service-agnostic은 아니다**(EXT-1·EXT-2를 ✅가 아니라 "의도적 🟡"로 남기는 이유).
- 서비스 추가가 *데이터*가 아니라 *마이그레이션* — 의도적으로 수용.

---

## 전환 조건 — B/C로 갈 때

다음 중 하나라도 충족되면 전환:

1. 2번째 서비스(예: agent)가 실제 출시 + 서비스 추가/변경이 분기당 1회 이상 churn
2. 테넌트 커스텀 capability 실수요 발생
3. per-service 마이그레이션이 릴리스 병목으로 **측정**됨

→ 우선순위: **capability CHECK 먼저 제거(B — 코드가 이미 권위)** → service는 레지스트리(C) 검토.

---

## 곁가지 결정 — Gate B 서비스 기본값 (EXT-5)

`checkGateB(orgPk, service="ACADEMY")`의 academy 기본값은 billing 경로 호환용 잔재다. **2번째 서비스 도입 시 기본값 제거(service 명시 전달 강제)로 중립화**한다. 그 전까지는 저위험이라 유지.

---

## 관련 문서

- [[requirements]] — §1 EXT-1·EXT-2·EXT-5 (이 결정이 종결하는 항목)
- [[role-as-code]] — capability 권위가 코드 상수라는 근거(B의 안전성 토대)
- [[design-asymmetry]] — service-agnostic 코어라는 목적
> 소스 문서
- [[schema-reference]] — §D.6 delegation_grant CHECK, §D.9·§D.12 service CHECK, §K 서비스 확장 방법
