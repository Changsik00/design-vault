# RAG 전략 (v0.1 MVP)

> 작성일: 2026-05-23
> RAG 는 v0.1 의 **Layer 0 인프라**. UI 가치가 아닌 분석·생성·질의의 공통 기반.

---

## 1. RAG 가 v0.1 에 있는 이유

v2 문서는 RAG 를 "학생 채팅 기능"으로 본 반면, 실제 RAG 는 **강의 영상 분석 자체의 입력 컨텍스트** 다.

```
                  ┌─────────────────────────────────────┐
                  │   사용자 가치 (Phase 별 단계 도입)   │
                  ├─────────────────────────────────────┤
                  │ v0.1: 강의 HTML 생성 (RAG-aug)      │
                  │ v0.5: 학원장 자연어 질의·면담 prep  │
                  │ v1.0: 학생 RAG 채팅·강사 자가 분석  │
                  │ v2.0: 글로벌 RAG (학원 간 익명 집계)│
                  └─────────────────┬───────────────────┘
                                    │
                                    ▼
   ┌────────────────────────────────────────────────────────┐
   │       RAG 인프라 (Layer 0, v0.1 부터 항상 켜져 있음)    │
   │                                                        │
   │   Qdrant + Neo4j + Upstage Solar Embedding             │
   │   org_id + teacher_id namespace                    │
   │   강의 publish 직후 자동 인덱싱                          │
   └────────────────────────────────────────────────────────┘
```

→ v0.1 에서 인프라 없으면 위에 얹을 v0.5/v1.0 가치 X.

---

## 2. Namespace 전략

### 2.1 격리 수준

```
학원 (org_id = identity_db.organization.pk)
  └─ 강사 (teacher_id = identity_db.identity_user.pk)
       └─ 강의 (lecture_id = academy_db.lecture.pk)
            └─ 청크 (chunk_id = academy_db.lecture_chunk.pk)
```

### 2.2 필터 패턴

| 사용처 | Qdrant filter | 의미 |
|---|---|---|
| **강사 RAG (자가 분석)** | `org_id == X AND teacher_id == Y` | 본인 강의·자료만 |
| **학원 RAG (학원장)** | `org_id == X` | 학원 전체 (강사 통합) |
| **학원 RAG (학생, v1.0+)** | `org_id == X AND visibility != 'private'` | 학원 전체에서 공개·미등록 |
| **학원 RAG (학부모, v1.0+)** | `org_id == X AND student.enrolled_classes` | 자녀 수강 수업만 |
| **글로벌 RAG (v2.0+)** | 별도 collection, 익명 집계 (k≥5) | 학원 간 비교 |

### 2.3 학원이 강사를 취해서 통합

학원 단위 질의는 **모든 강사 청크를 mix** 해서 retrieve. 결과 다양성 ↑.
강사 단위 질의는 본인 청크만. 결과 일관성 ↑.

---

## 3. 인덱싱 파이프라인

### 3.1 트리거

- 강의 publish 직후 (`youtube_video.status='published'`)
- 강사 자료 업로드 직후 (`lecture.type='material'`)

### 3.2 흐름

```
lecture_chunk.srt_content
       ↓ chunk 단위로 그대로 (or sub-chunk if > 500 tokens)
[Upstage Solar Embedding API]
       ↓ 1024-dim vector
[Qdrant upsert]
   point_id = UUID
   payload = {
     org_id, teacher_id, lecture_id, chunk_id,
     chunk_type ('lecture'|'material'),
     subject, grade_level,
     title, snippet_text (앞 200자),
     visibility (lecture 의 youtube_video visibility 따름),
     indexed_at, embedding_model
   }
       ↓
lecture_chunk.qdrant_point_ids = [...] UPDATE
lecture_chunk.rag_indexed_at = NOW UPDATE
lecture_chunk.embedding_model = "upstage-solar-1024" UPDATE
```

### 3.3 동시에 Neo4j

```
[Claude] chunk.srt_content + lecture context → 개념·관계 JSON 추출
       ↓
[Cypher MERGE] Subject / Chapter / Concept / Definition / Example
                + 관계 (CONTAINS, PREREQUISITE_OF, RELATED_TO, ...)
                + org_id, teacher_id 속성 강제
       ↓
(Lecture)-[:COVERS]→(Concept) 엣지 추가
```

### 3.4 청킹 전략 (MVP 기본값)

- 청크 단위 = lecture_chunk 의 자연 토픽 (보통 5~15분)
- 토픽 chunk 가 500 토큰 초과 시 sub-chunk (sentence boundary)
- embedding 은 sub-chunk 단위, payload 의 `chunk_id` 는 동일
- 한 토픽이 Qdrant 에 N 개 point 로 분할되어 들어감

차후 (v1.0+):
- Hybrid search (vector + keyword BM25)
- Reranking (cross-encoder)
- Dense + Sparse retrieval

---

## 4. 검색 패턴

### 4.1 v0.1 — HTML 생성용 retrieval

```
query: 현재 청크의 SRT 요약 (Claude 가 미리 작성한 1~2 문장)
filter: org_id + teacher_id
top_k: 8
   ├─ 강사 보유 자료 (top 5)
   └─ 같은 강사 과거 강의 청크 (top 3)
```

### 4.2 v0.5 — 학원장 자연어 질의

```
query: 학원장의 자연어 ("이번 달 학생들이 가장 어려워한 단원?")
filter: org_id
top_k: 10
   + Neo4j 보강 (개념 그래프에서 관련 개념·강사 추출)
   + Claude (synthesis)
```

### 4.3 v0.5 — 학생 면담 prep

```
query: 학생 ID, 면담 목적 (예: "수학 약점 분석")
filter: org_id + student.enrolled_classes
top_k: 10
   + 출결 / 평가 / 과거 상담 메모 cross-reference
   + Claude (prep 문서 생성)
```

### 4.4 v1.0 — 학생 RAG 채팅

```
query: 학생의 자연어 ("오늘 수학 뭐 배웠어?")
filter: org_id + visibility != 'private' + student.enrolled_classes
top_k: 8
   + Neo4j 보강 (개념 prerequisite)
   + Claude (학생 친화 답변 + 영상 카드)
```

---

## 5. 프롬프트 schema (HTML 생성)

Claude 입력은 **구조화된 섹션** 으로 묶어 일관성·토큰 효율 ↑.

```
<role>
당신은 학원 강의 슬라이드 작성자입니다. 한국어 교육 콘텐츠 전문가.
</role>

<lecture_context>
  학원: {academy_name}
  강사: {teacher_name}
  과목: {subject} · 학년: {grade}
  단원: {chapter}
</lecture_context>

<chunk_transcript>
  {srt_content_with_timestamps}
</chunk_transcript>

<retrieved_materials>
  [강사 보유 자료 — 참고용]
  [material_1] {title}
    {chunk_text_200chars}
  [material_2] ...
</retrieved_materials>

<retrieved_past_lectures>
  [같은 강사의 과거 관련 강의]
  [lecture_3 / 2026-04-12] {summary}
  ...
</retrieved_past_lectures>

<concept_graph>
  [Neo4j 추출]
  핵심 개념: [이차함수, 정점]
  선수 개념: [이차식, 함수의 그래프]
  관련 개념: [판별식, 최댓값]
</concept_graph>

<output_requirements>
  - HyperFrames HTML 컨벤션 준수:
    * <html><body>...</body></html> 완전 형태
    * window.__timelines 등록 (GSAP timeline)
    * 청크는 class="clip" data-start data-duration data-track-index
    * <audio> 는 외부 src (presigned S3 URL) 참조, muted 속성 X
    * Math.random() 금지, seeded PRNG 사용
  - 슬라이드 텍스트는 요점만 (transcript 그대로 X)
  - 시각 요소: 공식 박스, 예제 카드, 강조 텍스트
  - 한국어 가독성 우선 (긴 영어 단어 회피)
</output_requirements>
```

각 섹션 누락 시 LLM 이 안전 fallback (예: retrieved_materials 가 없으면 transcript 만으로 생성).

---

## 6. 강사 자료 업로드 UX 가이드

`lecture.type='material'` 로 업로드. 강사 자료 화면에서 안내 텍스트:

```
[강사 자료 업로드 — RAG 컨텍스트 보강]

이 자료는 강의 영상 생성 시 자동으로 참고됩니다.
어떤 자료가 효과적일까요?

📚 효과적인 자료
  ✓ 강의안 PDF (개념 + 예제가 정돈)
  ✓ 교과서 발췌 (정의·공식 카드)
  ✓ 예제 풀이 노트 (단계별 풀이)
  ✓ 자주 묻는 질문 모음

❌ 효과 적은 자료
  ✗ 긴 텍스트 위주 (요약·도식 없음)
  ✗ 타인 강의 영상 (저작권 충돌)
  ✗ 학생 개인정보 포함 자료 (이름·전화 등)

자료의 정보를 알려주세요 (RAG 검색 정확도 ↑):
  과목 *
  학년 *
  단원
  자료 형식 [PDF / 이미지 / 텍스트]
```

업로드된 자료는 OCR (Vision, v0.5+) 또는 PDF text extraction (v0.1) 으로 텍스트화 → `lecture_chunk` 로 분할 → 인덱싱.

---

## 7. Re-indexing / Lifecycle

### 7.1 청크 텍스트 변경 시 (드뭄)
- `lecture_chunk.srt_content` UPDATE → trigger `reindex-chunk` job
- 기존 qdrant points hard delete (by `qdrant_point_ids`)
- 새 embedding → upsert
- `qdrant_point_ids`, `rag_indexed_at` 갱신

### 7.2 Embedding 모델 변경 시
- 전체 chunk 대상 batch re-embed (별도 spec)
- 무중단 (새 collection 에 indexing → traffic switch)

### 7.3 강의 삭제 시
- `lecture.deleted_at` soft delete
- Qdrant points hard delete
- Neo4j Lecture 노드 detach (Concept 노드는 유지 — 다른 lecture 참조 가능)

### 7.4 학원 폐업 시
- 학원 데이터 export → 학원장에게 전달 (PIPA)
- 30일 후 hard delete (MySQL + Qdrant + Neo4j + S3)

---

## 8. 비용 통제

### 8.1 Embedding 비용

| 항목 | 비용 |
|---|---|
| Upstage Solar Embedding | ~$0.10 / 1M tokens |
| 강의 50분 transcript | ~10K tokens → $0.001 |
| 일 6편 × 30일 = 180편 | ~$0.20/월/학원 |

→ 사실상 무시 가능.

### 8.2 Claude (HTML 생성) — 가장 비싼 단계

| 항목 | 비용 |
|---|---|
| Sonnet 입력 (60K) + 출력 (4K) | ~$0.30/청크 |
| 청크 3개 평균 → 강의 1편 | ~$0.90 |
| 일 6편 × 30일 | ~$160/월/학원 |

→ 비용 cap (`org_entitlement.feature_limits.claude_monthly_usd`) 으로 통제. 집계는 `usage_log` 월별 파티션 SUM.

### 8.3 Neo4j 추출 (Haiku 사용)

| 항목 | 비용 |
|---|---|
| Haiku 입력+출력 | ~$0.02/청크 |
| 강의 1편 | ~$0.06 |
| 일 6편 × 30일 | ~$11/월/학원 |

→ Sonnet 대신 Haiku 로 충분 (구조화된 JSON 추출).

---

## 9. 평가 / 품질 측정

MVP 시점 (자동화된 평가):

| 메트릭 | 측정 방법 | 목표 |
|---|---|---|
| RAG retrieval recall | 강의 publish 후 strong/weak query 셋 측정 | ≥ 70% (top-5 안에 정답) |
| 인덱싱 latency | publish → indexed | < 1분/청크 |
| 인덱싱 누락률 | publish 영상 대비 indexed chunk 수 | 0% |
| Embedding cost / 영상 | accounting | < $0.01 |
| HTML 생성 cost / 영상 | accounting | < $2 (cap) |

v1.0+: 학생 thumbs up/down 으로 user feedback 측정.

---

## 10. 미결 / Phase 별 결정

| 항목 | Phase |
|---|---|
| Qdrant collection 단일 vs 학원별 분리 | 학원 1000+ 도달 시 재검토 |
| Hybrid search 도입 | v1.0 학생 채팅 정확도 측정 후 |
| Reranking 도입 | v1.0 학생 채팅 정확도 측정 후 |
| Cross-encoder ko 모델 평가 | v1.0+ |
| 자체 임베딩 호스팅 (BGE-M3 등) | Phase 3+ 비용 평가 |
| 글로벌 RAG (k-anonymity) | v2.0 |
