---
type: decision
status: 채택
aliases:
  - RAG 멀티테넌시
  - Qdrant 격리
  - shared collection
tags:
  - platform-db
  - decision
  - multitenancy
  - rag
  - qdrant
---

# RAG 멀티테넌시 — shared collection + payload 필터

> 상태: 채택 · 영역: 벡터 저장소(Qdrant) 테넌트 격리 · 형식: 비교 → 결정

## 결정

RAG 벡터는 **단일 `academy_lectures` collection을 모든 org가 공유**하되, 각 point의 payload에 `org_id`·`teacher_pk`를 저장하고 **검색 시 payload 필터로 격리**한다. collection을 테넌트마다 쪼개지 않는다.

> [[multitenancy-pool|MySQL Pool 모델]]과 같은 철학 — "물리 분리"가 아니라 "공유 + 행/payload 단위 격리".

---

## 맥락 — 왜 결정이 필요했나

RAG는 세 격리 요건을 동시에 만족해야 한다: ① org 간 격리(A학원 학생이 B학원 자료 검색 불가) ② 강사 내 격리(학생은 담당 강사 자료만) ③ 원장 cross-강사 분석. "collection을 어떻게 나누는가"에 따라 학원 수 증가 시 운영 복잡도가 결정된다.

---

## 비교한 선택지

### 선택지 A — collection-per-tenant

학원마다 별도 collection(`academy_lectures_org_1`, `_org_2` …).

- **장점**: 완전한 물리 격리 / org 삭제 = collection drop으로 깔끔 / org별 커스텀 스키마 가능
- **단점**:
  - 학원 수만큼 collection 선형 증가 → 클러스터 관리 복잡도 폭증
  - collection 경계를 넘는 검색 불가 → **원장 cross-강사 분석 불가**
  - collection당 인덱스·세그먼트 오버헤드 누적(소규모 org 많을수록 낭비)
  - Qdrant 권장 최대 collection 수 초과 시 성능 저하

### 선택지 B — shared collection + payload 필터 (채택)

단일 collection, point마다 `org_id`·`teacher_pk` payload, 검색 시 `must` 필터.

```json
{ "filter": { "must": [
  { "key": "org_id", "match": { "value": 1 } },
  { "key": "teacher_pk", "match": { "value": 10 } }
] } }
```

- **장점**: collection 1개만 운영(관리 단순) / 원장 스코프=org_id 필터만 → cross-강사 분석 가능 / payload index로 필터 성능 보장 / org 추가·삭제가 payload 기준(collection 작업 불필요)
- **단점**: 필터 누락 버그 시 org 간 노출 위험 → 코드 리뷰·테스트 필수 / 단일 org가 수백만 chunk면 공유 인덱스 성능 저하 가능

### 선택지 C — namespace-per-tenant

- **기각** — Qdrant는 collection 이하 namespace를 공식 지원하지 않음(구현 불가).

| 기준 | A: per-tenant | B: shared+필터 (우리) |
|---|---|---|
| 물리 격리 | 완전 | 논리(필터 기반) |
| cross-org 분석 | 불가 | 가능 |
| 운영 복잡도(학원 증가) | 선형 증가 | 일정 유지 |
| 클러스터 부하 분산 | 고정 | 균등 분산 |
| 격리 실패 위험 | 없음 | 필터 누락 시(코드로 방어) |

검색 스코프 3종: 강사=`[org_id, teacher_pk]` / 학생=`[org_id, teacher_pk]`(담당 강사 자동 결정, 결과 부족 시 org fallback) / 원장=`[org_id]`.

---

## 왜 B인가

학원 수가 수십~수천으로 늘어도 운영 복잡도가 일정하고, 원장 cross-강사 분석을 막지 않는다. 격리 실패 위험(필터 누락)은 "물리 분리"가 아니라 "필수 타입 + 코드 리뷰 + 통합 테스트"로 막는다 — MySQL Pool 모델에서 `org_pk NOT NULL`로 막는 것과 동일한 접근.

필수 안전장치:
- `org_id`·`teacher_pk` **payload index 생성**(없으면 필터 시 전체 스캔).
- 검색 코드에서 `org_id` 필터 누락은 PR 즉시 기각(보안 결함).
- 어댑터 인터페이스에서 `org_id`는 필수 타입(optional 금지). → [[cross-tenant-separation]]

---

## 트레이드오프 — 우리가 받아들인 것

- 격리가 DB 네이티브가 아니라 애플리케이션 필터에 의존 → 필터 누락이 곧 데이터 노출. 통합 테스트에 격리 검증을 반드시 포함한다.
- 단일 거대 org가 인덱스를 압박할 수 있다(아래 전환 조건으로 관리).

## 전환 조건 — collection-per-tenant로 갈 때

세 가지 중 하나 충족 시(Strangler Fig로 단계 전환):
1. 단일 org chunk 수 100만 건 초과
2. 학원별 검색 레이턴시 P95 SLA(500ms) 지속 위반
3. ISMS-P·GDPR 등 물리 분리 규제 요건

---

## 관련 문서

- [[multitenancy-pool]] — MySQL 행 격리(같은 "공유+격리" 철학)
- [[cross-tenant-separation]] — 필터를 의도적으로 벗어나는 cross-org 집계 처리
> 소스 문서
- [[schema-reference]] — §G 저장소별 격리 구현 현황
