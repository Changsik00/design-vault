# ADR-035: Qdrant RAG 멀티테넌시 — shared-collection + payload filter 채택, collection-per-tenant 전환 조건 사전 정의

| 항목 | 값 |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-05-25 |
| **Author** | changsik |
| **Replaces** | (없음 — 신규) |
| **Related** | `apps/academy-api/docs/rag-design.md` |

---

## Context

### 배경

Academy AI RAG는 강의 자료를 벡터 색인하고, 학생·강사·원장이 자연어로 질문하면 관련 강의 청크를 검색해 답변을 생성한다. 이 시스템은 다음 세 가지 격리 요구사항을 동시에 충족해야 한다:

1. **학원(org) 간 데이터 격리** — A 학원의 학생이 B 학원 강의 자료를 검색하면 안 됨
2. **강사 내 격리** — 학생은 담당 강사의 강의 자료만 검색 가능 (타 강사 자료 차단)
3. **원장 크로스 분석** — 원장은 소속 학원 전체 강의 자료를 검색 가능

Qdrant 클러스터 운영 전략으로 **"collection을 어떻게 나눌 것인가"** 를 결정해야 한다. 잘못 결정하면 학원 수 증가에 따라 운영 복잡도가 선형 이상으로 증가한다.

### 검토한 옵션

#### Option A — collection-per-tenant

학원마다 별도 Qdrant collection을 생성한다.

```
academy_lectures_org_1   ← org_1 전용
academy_lectures_org_2   ← org_2 전용
academy_lectures_org_N   ← org_N 전용
```

**장점**:
- 완전한 물리적 격리 — org 간 데이터 접근 가능성 원천 차단
- 컬렉션별 독립 스키마 운영 가능 (향후 org별 커스텀 페이로드 구조 지원)
- 단일 org 삭제 시 collection drop으로 완전 정리 가능

**단점**:
- 학원 수가 늘어날수록 collection 수 선형 증가 → Qdrant 클러스터 관리 복잡도 폭증
- 크로스 org 분석(원장 비교 분석) 불가 — collection 경계를 넘는 검색 미지원
- collection 당 오버헤드(인덱스, 세그먼트) 누적 → 소규모 org가 많을수록 자원 낭비
- Qdrant 권장 최대 collection 수(수백 개)를 초과하면 성능 저하 발생

---

#### Option B — shared-collection + payload filter (채택)

단일 `academy_lectures` collection을 모든 org가 공유하되, 각 point의 payload에 `org_id`와 `teacher_pk`를 저장하고 검색 시 필터로 격리한다.

```
collection: academy_lectures
├── point { org_id: 1, teacher_pk: 10, chunk_text: "..." }
├── point { org_id: 1, teacher_pk: 11, chunk_text: "..." }
├── point { org_id: 2, teacher_pk: 20, chunk_text: "..." }
└── ...
```

검색 시:
```json
{
  "filter": {
    "must": [
      { "key": "org_id", "match": { "value": 1 } },
      { "key": "teacher_pk", "match": { "value": 10 } }
    ]
  }
}
```

**장점**:
- Qdrant 클러스터 관리 단순 — collection 1개만 운영
- 크로스 org 분석 가능 (원장 스코프: org_id 필터만 적용)
- Qdrant payload index 설정으로 필터 성능 보장 (`org_id`, `teacher_pk` 인덱스 생성)
- org 추가/삭제가 collection 단위 작업 없이 payload 기준으로만 처리됨

**단점**:
- 필터 누락 버그 발생 시 org 간 데이터 노출 위험 → 코드 리뷰 + 테스트 필수
- 단일 org가 수백만 개 chunk를 보유하면 collection 공유로 인한 인덱스 성능 저하 가능

---

#### Option C — namespace-per-tenant

Qdrant의 네임스페이스 기능으로 tenant를 분리한다.

**기각 이유**: Qdrant는 collection-level 이하의 namespace를 공식 지원하지 않는다. 이 옵션은 구현 불가.

---

### 3가지 검색 스코프

| 역할 | Qdrant 필터 | 설명 |
|---|---|---|
| 강사 (TEACHER) | `must: [org_id=X, teacher_pk=Y]` | 본인 강의 자료만 검색 |
| 학생 (VIEWER) | `must: [org_id=X, teacher_pk=Y]` | 담당 강사 강의만 검색. `teacher_pk`는 `class_enrollment`로 자동 결정. 결과 top_k < 3이면 org 전체 fallback |
| 원장/관리자 (OWNER/ADMIN) | `must: [org_id=X]` | 학원 전체 강의 검색 (비교 분석 가능) |

스코프 결정 로직 구현 위치: `apps/academy-api/src/domain/rag/rag-scope.resolver.ts`

---

### 트레이드오프 매트릭스

| 기준 | Option A | Option B |
|---|---|---|
| 물리적 격리 수준 | 완전 격리 | 논리적 격리 (필터 기반) |
| 크로스 org 분석 | 불가 | 가능 |
| 운영 복잡도 (학원 증가 시) | 선형 증가 | 일정 유지 |
| Qdrant 클러스터 부하 분산 | 불가 (collection 단위 고정) | 가능 (단일 collection, 균등 분산) |
| 필터 성능 (payload index 설정 시) | 해당 없음 | HNSW 인덱스 + payload 필터 최적화 가능 |
| 전환 복잡도 (B→A) | — | 중간 (collection 분리 + 데이터 마이그레이션) |

---

## Decision

**Option B 채택. 단일 `academy_lectures` collection에 `org_id`, `teacher_pk` payload를 저장하고, 검색 시 payload filter로 격리한다.**

### Qdrant payload index 설정 (필수)

```typescript
// 인덱스 생성 — 서비스 기동 시 또는 마이그레이션에서 1회 실행
await qdrantClient.createPayloadIndex("academy_lectures", {
  field_name: "org_id",
  field_schema: "integer",
});
await qdrantClient.createPayloadIndex("academy_lectures", {
  field_name: "teacher_pk",
  field_schema: "integer",
});
```

payload index 없이 필터 검색 시 전체 스캔 발생 — 반드시 초기 설정 필수.

### 데이터 포인트 구조

```json
{
  "id": "01JWXYZ...",
  "vector": [0.1, 0.2, ...],
  "payload": {
    "org_id": 1,
    "teacher_pk": 10,
    "lecture_pk": 100,
    "chunk_index": 3,
    "text": "삼각함수의 덧셈 공식에서...",
    "subject_tag": "수학"
  }
}
```

---

## Consequences

### 즉각 적용 규칙

- Qdrant 검색 코드에서 `org_id` 필터 누락은 PR 리뷰에서 즉시 기각 (보안 결함)
- `teacher_pk` 필터는 역할에 따라 조건부 적용 — 원장 스코프에서는 생략 가능
- `academy_lectures` collection에 `org_id`, `teacher_pk` payload index 반드시 생성
- 학생 검색 스코프의 `teacher_pk` 자동 결정은 `class_enrollment` 테이블 기반 (`student-teacher-scope.md` 참조)
- RAG_INDEX 스텝에서 Qdrant upsert 시 반드시 `org_id`, `teacher_pk`, `lecture_pk`, `chunk_index`, `subject_tag` payload 포함

### collection-per-tenant 전환 트리거 (세 가지 중 하나 충족 시)

1. 단일 학원(org)의 총 chunk 수가 100만 건 초과
2. 학원별 검색 레이턴시가 P95 기준 SLA(500ms) 지속 위반
3. 규제 요건(ISMS-P, GDPR 등)으로 물리적 데이터 분리 요구

전환 시 절차: 기존 collection 데이터를 org별 신규 collection으로 마이그레이션 후 앱 배포. 단계적 전환(Strangler Fig) 권장.

### 장기 위험

- 필터 코드가 애플리케이션 레이어에 존재하므로 버그 시 org 간 데이터 노출 가능 → 통합 테스트에서 반드시 격리 검증 포함

---

## 참조

| 문서 | 내용 |
|---|---|
| `apps/academy-api/docs/rag-design.md` | RAG 전체 아키텍처 설계 |
| `apps/academy-api/docs/student-teacher-scope.md` | 학생 RAG 검색 스코프 결정 로직 |
| `apps/academy-api/docs/sequences/03-rag-indexing.md` | RAG 인덱싱 시퀀스 |
| `apps/academy-api/docs/sequences/04-lecture-pipeline-detail.md` | 강의 파이프라인 상세 (RAG_INDEX 스텝 포함) |
| ADR-036 | Qdrant + Neo4j 이중 색인 전략 |
