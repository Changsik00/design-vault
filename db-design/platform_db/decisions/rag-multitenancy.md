---
type: decision
status: 채택
aliases:
  - RAG 멀티테넌시
  - RAG 격리
  - Qdrant 격리
  - Neo4j 격리
  - shared collection
tags:
  - platform-db
  - decision
  - multitenancy
  - rag
  - qdrant
  - neo4j
---

# RAG 멀티테넌시 — 공유 인프라 + `org_id` 격리

> 상태: 채택 · 영역: RAG 저장소(벡터 **Qdrant** + 그래프 **Neo4j**) 테넌트 격리 · 형식: 비교 → 결정

> 📌 **용어**: 이 문서에서 **RAG = 벡터 검색(Qdrant) + 그래프 검색(Neo4j)** 인프라를 함께 가리킨다. 둘 다 `platform_db`가 아니라 **서비스 도메인** 저장소이지만, 테넌트 격리 철학은 [[multitenancy-pool|Postgres Pool 모델]]과 동일하게 적용한다.

## 결정

RAG의 두 저장소 모두 **테넌트마다 인스턴스·collection을 쪼개지 않고 공유**하되, **`org_id`로 논리 격리**한다. 물리 분리가 아니라 "공유 + 행/포인트/노드 단위 격리" — Postgres가 `org_pk NOT NULL` + RLS로 막는 것과 같은 접근을 벡터·그래프에 옮긴 것이다.

| 저장소 | 격리 키 | 강제 지점 |
|---|---|---|
| **Qdrant**(벡터) | point payload `org_id`·`teacher_pk` | 검색 시 `must` 필터 (payload index) |
| **Neo4j**(그래프) | 노드 속성 `orgId` | Cypher 쿼리 · **멀티홉 경로의 모든 노드**에 `orgId` 강제 |

> 같은 "공유 + 격리" 철학을 3개 저장소(Postgres·Qdrant·Neo4j)에 일관 적용한다. 격리 실패 위험은 "물리 분리"가 아니라 **필수 타입 + 코드 리뷰 + 통합 테스트**로 막는다.

---

## 맥락 — 왜 결정이 필요했나

RAG는 세 격리 요건을 동시에 만족해야 한다: ① org 간 격리(A학원 학생이 B학원 자료 검색 불가) ② 강사 내 격리(학생은 담당 강사 자료만) ③ 원장 cross-강사 분석. "저장소를 어떻게 나누는가"에 따라 학원 수 증가 시 운영 복잡도가 결정된다. 벡터와 그래프는 데이터 모델이 다르므로 격리 강제 방식도 다르지만, **분리하지 않고 공유한다**는 결론은 같다.

---

## 1. 벡터 (Qdrant)

### 비교한 선택지

**선택지 A — collection-per-tenant**: 학원마다 별도 collection(`academy_lectures_org_1`, `_org_2` …).
- **장점**: 완전한 물리 격리 / org 삭제 = collection drop으로 깔끔 / org별 커스텀 스키마 가능
- **단점**: 학원 수만큼 collection 선형 증가(클러스터 관리 복잡도 폭증) / collection 경계 넘는 검색 불가(**원장 cross-강사 분석 불가**) / collection당 인덱스·세그먼트 오버헤드 누적 / Qdrant 권장 최대 collection 수 초과 시 성능 저하

**선택지 B — shared collection + payload 필터 (채택)**: 단일 collection, point마다 `org_id`·`teacher_pk` payload, 검색 시 `must` 필터.

```json
{ "filter": { "must": [
  { "key": "org_id", "match": { "value": 1 } },
  { "key": "teacher_pk", "match": { "value": 10 } }
] } }
```

- **장점**: collection 1개만 운영(관리 단순) / 원장 스코프=`org_id` 필터만 → cross-강사 분석 가능 / payload index로 필터 성능 보장 / org 추가·삭제가 payload 기준(collection 작업 불필요)
- **단점**: 필터 누락 버그 시 org 간 노출 위험 → 코드 리뷰·테스트 필수 / 단일 org가 수백만 chunk면 공유 인덱스 성능 저하 가능

**선택지 C — namespace-per-tenant**: **기각** — Qdrant는 collection 이하 namespace를 공식 지원하지 않음(구현 불가).

| 기준 | A: per-tenant | B: shared+필터 (우리) |
|---|---|---|
| 물리 격리 | 완전 | 논리(필터 기반) |
| cross-org 분석 | 불가 | 가능 |
| 운영 복잡도(학원 증가) | 선형 증가 | 일정 유지 |
| 클러스터 부하 분산 | 고정 | 균등 분산 |
| 격리 실패 위험 | 없음 | 필터 누락 시(코드로 방어) |

검색 스코프 3종: 강사=`[org_id, teacher_pk]` / 학생=`[org_id, teacher_pk]`(담당 강사 자동 결정, 결과 부족 시 org fallback) / 원장=`[org_id]`.

---

## 2. 그래프 (Neo4j)

벡터와 달리 그래프는 **경로(path)**가 격리 단위다. 단일 노드 필터로는 부족하다 — 멀티홉 경로가 **다른 org 노드를 한 번이라도 거치면** 그 자체로 누출이다.

**결정 — 공유 그래프 + 노드 `orgId` 속성 + 경로 전체 강제**:
- 모든 노드에 `orgId` 속성을 부여한다(생성 시 필수).
- Cypher 쿼리는 매칭되는 **모든 바인딩 노드**에 `orgId = $orgId`를 강제한다. 멀티홉(`(a)-[*]->(b)`)이면 경로상 중간 노드까지 전부 같은 org여야 한다.
- **APOC 등 절차적 우회**는 타입 강제를 건너뛸 수 있으므로 **쓰기측 APOC를 차단**한다(P1).

```cypher
// org 격리: 경로상 모든 노드가 같은 org여야 함
MATCH p = (c:Concept {orgId: $orgId})-[:RELATED*1..3]->(t:Concept)
WHERE all(n IN nodes(p) WHERE n.orgId = $orgId)
RETURN p
```

> 왜 collection-per-tenant(Qdrant)에 대응하는 "graph-per-tenant"를 안 쓰나: 그래프는 org 경계를 넘는 개념 연결이 분석 가치를 가질 수 있고(원장 스코프), 인스턴스 분리는 멀티홉 쿼리·운영 복잡도를 키운다. 벡터와 같은 이유로 **공유 + `orgId` 경로 강제**를 택한다.

---

## 왜 공유 + 격리인가 (공통)

학원 수가 수십~수천으로 늘어도 운영 복잡도가 일정하고, 원장 cross-강사 분석을 막지 않는다. 격리 실패 위험(Qdrant 필터 누락 / Neo4j 경로 누락)은 "물리 분리"가 아니라 **필수 타입 + 코드 리뷰 + 통합 테스트**로 막는다 — Postgres Pool 모델에서 RLS + `org_pk NOT NULL`로 막는 것과 동일한 접근.

필수 안전장치:
- Qdrant: `org_id`·`teacher_pk` **payload index 생성**(없으면 필터 시 전체 스캔). 검색 코드의 `org_id` 필터 누락은 PR 즉시 기각.
- Neo4j: 노드 생성 시 `orgId` 필수, 경로 쿼리는 `all(n IN nodes(p) WHERE n.orgId = $orgId)` 강제. 쓰기측 APOC 차단(P1).
- 벡터·그래프 어댑터 인터페이스에서 `org_id`는 **필수 타입**(optional 금지). → [[cross-tenant-separation]]

---

## 트레이드오프 — 우리가 받아들인 것

- 격리가 DB 네이티브가 아니라 애플리케이션 필터/쿼리에 의존 → 누락이 곧 데이터 노출. 통합 테스트에 격리 검증(Qdrant 필터·Neo4j 경로)을 반드시 포함한다.
- 단일 거대 org가 공유 인덱스/그래프를 압박할 수 있다(아래 전환 조건으로 관리).

## 전환 조건 — per-tenant 물리 분리로 갈 때

세 가지 중 하나 충족 시(Strangler Fig로 단계 전환):
1. 단일 org chunk/노드 수 100만 건 초과
2. 학원별 검색 레이턴시 P95 SLA(500ms) 지속 위반
3. ISMS-P·GDPR 등 물리 분리 규제 요건

---

## 관련 문서

- [[multitenancy-pool]] — Postgres 행 격리(같은 "공유+격리" 철학의 원형)
- [[cross-tenant-separation]] — 필터/경로를 의도적으로 벗어나는 cross-org 집계 처리
- [[schema-reference]] — §G.2 Qdrant · §G.3 Neo4j 격리 구현 현황
