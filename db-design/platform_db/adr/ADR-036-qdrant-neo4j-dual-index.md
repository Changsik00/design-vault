# ADR-036: RAG 색인 이중 구조 — Qdrant(dense vector) + Neo4j(concept graph) 채택

| 항목 | 값 |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-05-25 |
| **Author** | changsik |
| **Replaces** | (없음 — 신규) |
| **Related** | `apps/academy-api/docs/rag-design.md`, `apps/academy-api/src/shared/jobs/pipeline-jobs.ts` |

---

## Context

### 배경

학원 강의 RAG에서 학생이 "삼각함수 덧셈 공식 알려줘"라고 질문하면, 해당 chunk만 반환해서는 답변 품질이 충분하지 않다. 교육 콘텐츠 특성상 **관련 개념**(사인 법칙, 코사인 법칙, 단위원 등)이 함께 제공되어야 학습 효과가 높아진다.

순수 벡터 검색(cosine similarity)은 의미적 유사도를 기반으로 동작하므로, 명시적인 "개념 연결" 지식 그래프 없이는 다음 한계가 있다:

- 동의어/유사 표현 기반 검색에는 강함 ("덧셈 공식" ≈ "합차 공식")
- 명시적 개념 계층(선수 개념, 연관 개념) 탐색에는 약함
- 같은 강의에 자주 함께 등장하는 개념 간의 연결을 자동으로 발견하지 못함

이 한계를 보완하기 위해 **벡터 검색 + 개념 그래프 검색 이중 구조** 채택 여부를 결정한다.

### 검토한 옵션

#### Option A — Qdrant 단독 벡터 검색

```
질문 → embedding → Qdrant cosine search (top-k) → LLM 답변 생성
```

**장점**:
- 구현 단순 — 인프라 1개 (Qdrant)
- 운영 복잡도 최소

**단점**:
- 개념 확장 없이 cosine similarity 상위 k개만 반환
- 교육 콘텐츠 특성상 "삼각함수를 이해하려면 단위원을 알아야 한다"는 선수 개념 연결 불가
- 같은 강의에서 함께 등장하는 개념 간 관계 발견 불가
- 검색 결과가 질문과 표면적으로 유사한 chunk에 편중 → 맥락 부족 답변 발생 가능성

---

#### Option B — Qdrant + Neo4j 이중 색인 (채택)

```
질문 → embedding → Qdrant dense search (seed chunks)
                          ↓
              Neo4j APPEARS_IN 역탐색 (seed chunk → 관련 개념)
                          ↓
              Neo4j RELATED_TO 1-hop 확장 (관련 개념 → 추가 chunk)
                          ↓
              중복 제거 → LLM 답변 생성
```

**장점**:
- 교육 도메인의 개념 연결 강점 극대화 — 선수 개념, 연관 개념 자동 확장
- Qdrant가 "무엇을 찾는가"를 결정하고, Neo4j가 "무엇과 연결되는가"를 결정 — 역할 분리 명확
- 개념 그래프는 강의가 추가될수록 자동으로 풍부해짐 (공동 출현 기반)
- 크로스 강의 개념 연결 가능 — 2개 강의에서 공통 개념 등장 시 자동 연결

**단점**:
- Neo4j 인프라 운영 필요 (docker-compose.dev.yml 추가, 프로덕션 클러스터 관리)
- RAG_INDEX 스텝에서 Qdrant + Neo4j 동시 기록 → 실패 시 부분 색인 정합성 관리 필요
- 개념 추출 품질이 Neo4j 그래프 품질을 결정 — LLM 개념 추출 프롬프트 설계 중요

---

#### Option C — Qdrant + BM25 Hybrid 검색

```
질문 → [dense vector (Qdrant) + sparse BM25] → RRF(Reciprocal Rank Fusion) → LLM 답변 생성
```

**장점**:
- 키워드 정밀도 향상 — "삼각함수"라는 단어가 명확히 등장하는 chunk 우선 반환
- Qdrant 자체 hybrid search 지원 — 추가 인프라 불필요

**단점**:
- 개념 확장보다 키워드 정밀도 향상에 초점 — 교육 콘텐츠의 핵심 요구사항(개념 연결)에 부합하지 않음
- BM25는 형태소 분석 필요 (한국어 처리) → 추가 설정 복잡도
- "개념 간 관계 탐색"이 아닌 "질문과 유사한 표현 찾기"에 특화 → 선수 개념 발견 불가

**기각** — 개념 연결이 핵심 요구사항인 교육 도메인에서 BM25 hybrid는 Option B보다 적합하지 않다고 판단.

---

### 트레이드오프 매트릭스

| 기준 | Option A | Option B | Option C |
|---|---|---|---|
| 개념 연결 탐색 | 불가 | 가능 (그래프 탐색) | 불가 |
| 키워드 정밀도 | 보통 | 보통 | 높음 |
| 인프라 복잡도 | 낮음 | 중간 | 낮음 |
| 검색 결과 다양성 | 낮음 | 높음 | 보통 |
| 구현 비용 | 낮음 | 중간 | 낮음 |
| 교육 도메인 적합성 | 보통 | 높음 | 보통 |

---

## Decision

**Option B 채택. Qdrant(dense vector) + Neo4j(concept graph) 이중 색인으로 RAG 검색 품질을 높인다.**

### Neo4j 그래프 구조

```cypher
// 노드 정의
(Concept { name: String, orgId: Long, teacherPk: Long })
(LectureChunk { chunkId: String, orgId: Long, teacherPk: Long, lecturePk: Long })

// 관계 정의
(Concept)-[:APPEARS_IN]->(LectureChunk)   // 개념이 특정 chunk에 등장
(Concept)-[:RELATED_TO]->(Concept)        // 동일 chunk 내 공동 출현 → 개념 연결
```

**격리 원칙**: `Concept`과 `LectureChunk` 모든 노드에 `orgId`, `teacherPk` 프로퍼티 저장. 조회 시 반드시 `WHERE orgId = $orgId` 조건 포함.

### 검색 흐름 상세

```
1. Qdrant dense search (top 10)
   → filter: { org_id: orgId, teacher_pk: teacherPk }
   → seed_chunks = [chunk_A, chunk_B, ...]

2. Neo4j APPEARS_IN 역탐색
   MATCH (c:Concept)-[:APPEARS_IN]->(lc:LectureChunk)
   WHERE lc.chunkId IN $seedChunkIds AND lc.orgId = $orgId
   RETURN c (관련 개념어 목록)

3. Neo4j RELATED_TO 1-hop 확장
   MATCH (c:Concept)-[:RELATED_TO]->(related:Concept)-[:APPEARS_IN]->(lc:LectureChunk)
   WHERE c.name IN $concepts AND lc.orgId = $orgId
   RETURN lc.chunkId AS expandedChunkIds

4. 추가 chunk 수집 (Qdrant 또는 MySQL에서 텍스트 조회)

5. 중복 제거 후 최종 context 구성

6. OpenAI GPT-4o → 답변 생성
```

### RAG_INDEX 스텝에서 Neo4j 기록

```cypher
// chunk 색인 시 실행
MERGE (lc:LectureChunk { chunkId: $chunkId, orgId: $orgId, teacherPk: $teacherPk })
SET lc.lecturePk = $lecturePk

FOREACH (concept IN $concepts |
  MERGE (c:Concept { name: concept, orgId: $orgId, teacherPk: $teacherPk })
  MERGE (c)-[:APPEARS_IN]->(lc)
)

// 동일 chunk 내 개념 간 RELATED_TO 관계 (공동 출현 기반)
WITH lc, $concepts AS concepts
UNWIND concepts AS c1
UNWIND concepts AS c2
WITH lc, c1, c2 WHERE c1 < c2
MATCH (n1:Concept { name: c1, orgId: $orgId }), (n2:Concept { name: c2, orgId: $orgId })
MERGE (n1)-[:RELATED_TO]->(n2)
MERGE (n2)-[:RELATED_TO]->(n1)
```

---

## Consequences

### 즉각 적용 규칙

- Neo4j를 `docker-compose.dev.yml`에 포함 — 로컬 개발 환경에서 Qdrant와 함께 기동
- RAG_INDEX 스텝: Qdrant upsert와 Neo4j MERGE를 순차 실행 (Qdrant 실패 시 Neo4j 기록 스킵, 재시도 시 멱등성 보장)
- 개념 추출은 AI_SUMMARY 스텝에서 GPT-4o로 수행 — `concepts: string[]` 필드를 JSON 출력에 포함 (ADR-033 참조)
- Neo4j 조회 시 반드시 `WHERE orgId = $orgId` 조건 포함 — 미포함 시 PR 기각
- `LectureChunk` Neo4j 노드의 `chunkId`는 MySQL `lecture_chunk.public_id` (ULID)와 1:1 대응

### 부분 색인 정합성 처리

Qdrant 성공 + Neo4j 실패 시: `lecture_chunk.neo4j_id = NULL` 상태로 남음. RAG 검색은 Qdrant 단독으로 동작(개념 확장 없이). 재색인 job으로 복구 가능하도록 `indexed_at = NULL` 체크 배치 설계 권장.

### 향후 전환 트리거 (Neo4j 제거 검토 조건)

1. Qdrant 단독 검색 품질이 Neo4j 확장 대비 통계적으로 유의미한 차이가 없음을 A/B 테스트로 확인
2. Neo4j 운영 비용이 RAG 검색 품질 향상 대비 과도하다고 판단

### 장기 위험

- Neo4j RELATED_TO 그래프가 시간이 지날수록 조밀해짐 → 1-hop 탐색 결과 폭발 가능성. 향후 관계 강도(weight) 도입 또는 탐색 깊이 제한 검토 필요

---

## 참조

| 문서 | 내용 |
|---|---|
| `apps/academy-api/docs/rag-design.md` | RAG 전체 아키텍처 설계 |
| `apps/academy-api/docs/sequences/03-rag-indexing.md` | RAG 인덱싱 시퀀스 |
| `apps/academy-api/docs/sequences/04-lecture-pipeline-detail.md` | 강의 파이프라인 상세 (RAG_INDEX 스텝 포함) |
| `apps/academy-api/docs/persona-system.md` | AI_SUMMARY 출력 스키마 (concepts 필드 정의) |
| ADR-033 | Academy AI Agent LLM 전략 (GPT-4o 개념 추출) |
| ADR-035 | Qdrant RAG 멀티테넌시 전략 (shared-collection 구조) |
