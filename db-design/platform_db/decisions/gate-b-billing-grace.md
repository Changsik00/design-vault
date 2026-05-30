---
type: decision
status: 채택
aliases:
  - Gate B 유예 설계 결정
  - validUntil 복합체크
  - status-only 이탈
tags:
  - platform-db
  - decision
  - billing
  - gate-b
  - grace
---

# Gate B 빌링 유예 설계 결정

## 배경

구독 갱신 시 결제 시스템(PG)과 platform_db 사이에는 시간 차가 발생한다.

```
결제 성공 이벤트 → 웹훅/배치 → org_subscription 갱신 → org_entitlement 갱신
                                                                    ↑
                                               Gate B 가 체크하는 것은 여기
```

이 배치 처리가 **지연되거나 실패**하면 정상 결제 고객이 서비스 접근을 잃게 된다.
반대로 배치가 `status=ACTIVE`를 갱신하지 못한 채 방치되면 만료된 구독이 영구 무료 접근 상태가 된다.

Gate B는 이 두 위험을 동시에 다루어야 한다.

---

## 자문 권고 (방법 2 — status-only 체크)

자문은 다음 방식을 권고했다:

```typescript
// 방법 2: status 필드만 체크
function isEntitlementValid(e: OrgEntitlement | null): boolean {
  if (e === null) return false;
  return e.status === "ACTIVE" || e.status === "GRACE";
}
```

**권고 근거**:

- `org_entitlement.status` 는 배치가 갱신하는 **의미론적 진실**이다. validUntil 은 배치가 관리하는 부가 정보.
- 배치가 정상 동작하는 한 status 가 진실이므로, Gate B 는 status 만 신뢰하면 충분하다.
- GRACE 기간은 status 필드로 명시되어 있으므로, Gate B 가 시간 계산을 할 필요가 없다.
- Gate B 에 시간 계산을 넣으면 배치 로직과 게이트 로직이 중복(dual-truth)된다.

---

## 구현 결정 — 부분적 이탈 (validUntil 복합 체크)

본 구현은 status + validUntil 복합 체크를 채택했다:

```typescript
// 실제 구현: status + validUntil 복합 체크
function isEntitlementValid(e: OrgEntitlement | null): boolean {
  if (e === null) return false;
  const statusOk = e.status === "ACTIVE" || e.status === "GRACE";
  const notExpired = e.validUntil === null || e.validUntil > new Date();
  return statusOk && notExpired;
}
```

**이탈 근거 — 불변식 #9 (배치 실패 안전망)**:

배치가 `status=ACTIVE → EXPIRED` 갱신에 실패할 경우:

| 상태 | 방법 2 (status-only) | 본 구현 (복합 체크) |
|------|---------------------|-------------------|
| `status=ACTIVE, validUntil=과거` | ✅ 통과 (영구 무료) | ❌ 차단 (안전망 동작) |
| `status=ACTIVE, validUntil=미래` | ✅ 통과 | ✅ 통과 |
| `status=GRACE, validUntil=과거` | ✅ 통과 | ❌ 차단 |
| `status=EXPIRED, validUntil=미래` | ❌ 차단 | ❌ 차단 |

배치 실패 시 방법 2는 "영구 무료" 취약점이 생긴다. `validUntil` 은 배치가 `status` 를 갱신하지 못했더라도 최소한 만료일 이후 접근을 막아주는 **2차 방어선**이다.

**validUntil=null 처리**:

`null` 은 "무기한(indefinite)"을 의미한다 — 수동 부여된 영구 구독, 파일럿 계정 등.
`null`이면 `notExpired = true` 로 처리 (차단하지 않음).

---

## 트레이드오프

| 항목 | 방법 2 | 본 구현 |
|------|--------|---------|
| 단순성 | status 필드 하나만 신뢰 | status + validUntil 두 필드 |
| 배치 실패 안전망 | 없음 | validUntil 이 2차 방어 |
| 배치-게이트 결합도 | 낮음 | 약간 높음 |
| "영구 무료" 리스크 | 배치 실패 시 발생 | 발생 안 함 |
| GRACE 기간 명확성 | status=GRACE 로 충분 | validUntil 로도 GRACE 만료 체크 |

---

## 향후 조건

아래 조건이 충족되면 방법 2 (status-only)로 단순화를 재검토할 수 있다:

1. 배치(청구 주기 갱신 잡)가 SLA 내 실패 시 알람 + 자동 재시도 보장
2. `status` 와 `valid_until` 의 불일치를 탐지하는 정합성 모니터링 구축
3. 배치 실패가 3개월간 0건이었음을 운영에서 확인

---

## 관련 문서

- [[auth-projection]] — Gate B가 읽는 org_entitlement가 billing의 투영인 이유
- [[subscription-lifecycle]] — 구독 상태와 GRACE 전이
> 소스 문서
- [[schema-reference]] — §D.12 org_entitlement 명세, §E.2 Gate B 구현, idx_entitlement_expiry 인덱스
- [[architecture]] — §3.1 불변식 #9 (status + valid_until 복합 체크)
