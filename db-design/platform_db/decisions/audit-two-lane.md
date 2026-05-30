---
type: decision
status: 채택
aliases:
  - audit 2-lane
  - 감사 로그 분리
  - 컴플라이언스 vs 텔레메트리
  - audit 샘플링
tags:
  - platform-db
  - decision
  - audit
  - observability
  - compliance
---

# 감사 로그 2-lane — 컴플라이언스 audit vs access 텔레메트리

> 상태: 채택 · 영역: 운영 가능성(Operability) · 형식: 비교 → 결정

## 결정

권한 이벤트를 **2개 lane**으로 분리한다.

1. **컴플라이언스 `audit_log`** — 보안유의 이벤트만 **100%**, append-only · WORM · 5년. (DENY · ERROR · 민감 리소스 ALLOW · admin/break-glass · consent · 운영자 행위)
2. **access 텔레메트리** — 일상 read ALLOW 대량 → 관측 파이프라인(OLAP, 예: ClickHouse), **샘플링 허용**, 컴플라이언스 로그 아님.

> 즉 *"audit_log를 샘플링"*이 아니라 *"무엇이 audit이고 무엇이 telemetry인가"*를 가른다.

---

## 맥락 — 볼륨 폭발

현재 AUD-2 = "모든 권한 결정(ALLOW/DENY/ERROR) 기록". 1000 rps × gate 5회 ≈ **5000 audit/s ≈ 4.3억/일** → `audit_log` 폭발(파티션·스토리지·쿼리 모두 위협).

순진한 해법 "ALLOW 샘플링"은 **잘못된 대상을 샘플링**한다 — 그 로그는 컴플라이언스 증거다.

---

## 비교한 선택지

### A — 단일 audit_log에 전량 (현재)

- **장점**: 단순, 완전
- **단점**: 볼륨 폭발(4.3억/일), 컴플라이언스 증거가 일상 텔레메트리에 묻힘

### B — 단일 로그 + ALLOW 샘플링

- **장점**: 볼륨↓
- **단점**: **컴플라이언스 완전성 위반** — PIPA/ISMS-P/SOC2에서 "누가 무엇에 접근했는지 증명 불가". 샘플링된 audit는 감사 증거로 부적격.
- **기각**.

### C — 2-lane 분리 (채택)

- **컴플라이언스 lane**: 보안유의 이벤트만 100% (대폭 축소된 볼륨), WORM
- **텔레메트리 lane**: 대량 read ALLOW는 관측용으로, 샘플링·단기보존·OLAP
- **장점**: 컴플라이언스 완전성 유지 + 볼륨 통제 + 관측성 확보
- **단점**: "무엇이 보안유의인가" 분류 규칙 필요, 파이프라인 1개 추가

| 기준 | A: 전량 | B: 샘플링 | C: 2-lane (우리) |
|---|---|---|---|
| 컴플라이언스 완전성 | ✅ | ❌ | ✅ |
| 볼륨 통제 | ❌ | ✅ | ✅ |
| 관측성 | △ | △ | ✅ |

---

## 컴플라이언스 lane에 들어가는 것 (계약)

- 모든 **DENY · ERROR**
- 민감 리소스 **ALLOW**: 결제·환불·publish·삭제·권한/role 변경·consent 변경·export
- **admin / break-glass / 운영자 support-action** 전건 ([[operator-plane]])
- 일상 read ALLOW(목록·단순 조회)는 ❌ → 텔레메트리

> **AUD-2 수정 필요**: "모든 권한 결정 기록" → "**보안유의 이벤트** 기록". 일상 read는 audit 아님.

---

## 트레이드오프

- 분류 규칙 유지보수 + 텔레메트리 파이프라인. 대신 `audit_log`가 작고 신뢰 가능해진다.

## 전환 조건

- 텔레메트리 lane은 관측 요구가 생길 때 도입. **지금 당장 할 일**은 컴플라이언스 lane 정의 + *일상 ALLOW를 audit_log에 넣지 않기*.

---

## 관련 문서

- [[audit-hash-chain]] — 컴플라이언스 lane의 무결성(해시 체인)
- [[partitioning]] — audit_log 파티셔닝(축소된 볼륨엔 더 여유)
- [[operator-plane]] — 운영자 행위가 컴플라이언스 lane에 들어가는 이유
- [[requirements]] — §6 운영 가능성 (AUD-2 범위 수정)
> 소스 문서
- [[schema-reference]] — §D.8 audit_log, §H.2 감사 불변성
