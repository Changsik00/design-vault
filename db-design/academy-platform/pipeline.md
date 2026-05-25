# 강의 파이프라인 (v0.1)

> 작성일: 2026-05-23
> SRT 를 중간 산출로 도입해 TTS와 HTML 의 타이밍 정렬을 단일 source 로 보장.

---

## 1. 단대단 흐름

```
[강사 audio 업로드] (Vite SPA → presigned S3 PUT)
       │
       ▼
[academy-api]
   ├─ S3 저장 확인
   ├─ MySQL: lecture INSERT (status=queued)
   └─ BullMQ enqueue: process-lecture(lectureId)
       │
       ▼  비동기
[academy-worker] (BullMQ jobs)
   ┌─────────────────────────────────────────────────────┐
   │ 1. transcribe                                        │
   │    Google STT V2 batchRecognize (multi-URI 청크 분할)│
   │    output: lecture.transcript (timestamps 포함)      │
   ├─────────────────────────────────────────────────────┤
   │ 2. chunk-by-topic + script                           │
   │    Claude (RAG-augmented)                            │
   │    - 토픽 경계 자동 식별                              │
   │    - 청크당 SRT 생성 (시작/종료 timestamp 포함)       │
   │    - 청크당 시각 슬라이드 keyword/구조                 │
   │    output: lecture_chunk N rows                      │
   ├─────────────────────────────────────────────────────┤
   │ 3a. tts-narration  (각 청크 병렬)                     │
   │    Google TTS Neural2 ko-KR                          │
   │    input: chunk.srt_content (텍스트만)                │
   │    output: video_asset(type=audio_mp3) per chunk      │
   ├─────────────────────────────────────────────────────┤
   │ 3b. generate-html  (각 청크 병렬, 3a 와 동시)         │
   │    Claude (RAG-augmented)                            │
   │    input: chunk.srt + RAG context (materials,        │
   │           past lectures, Neo4j concepts)             │
   │    output: video_asset(type=hyperframes_html)        │
   │           <audio src="..."> 가 3a 결과 가리킴          │
   ├─────────────────────────────────────────────────────┤
   │ 4. render-video  (3a + 3b 완료 후, 각 청크 병렬)      │
   │    HyperFrames API (webhook 대기)                    │
   │    output: video_asset(type=video_mp4) per chunk     │
   ├─────────────────────────────────────────────────────┤
   │ 5. await-review                                      │
   │    lecture.status = pending_review                   │
   │    학원장 알림톡 + 이메일 발송 (검수 대기 1건)         │
   │    (수동 검수 default — auto_publish 권한 trust 시 skip)│
   ├─────────────────────────────────────────────────────┤
   │ 6. upload-youtube  (검수 통과 후)                     │
   │    YouTube Data API v3 resumable upload              │
   │    + thumbnails.set                                  │
   │    output: youtube_video.status=published            │
   ├─────────────────────────────────────────────────────┤
   │ 7. notify                                            │
   │    학부모 알림톡 (영상 링크) → SMS/이메일 fallback     │
   │    강사 알림 ("영상 publish 완료")                   │
   └─────────────────────────────────────────────────────┘

   ─ 인덱싱 단계 (publish 직후 별도 큐) ─
   ┌─────────────────────────────────────────────────────┐
   │ 8. index-rag                                         │
   │    각 chunk SRT → Upstage embedding                  │
   │    Qdrant upsert (payload: org_id + teacher_id +  │
   │                          lecture_id + chunk_id + ...) │
   │    lecture_chunk.qdrant_point_ids 저장                │
   ├─────────────────────────────────────────────────────┤
   │ 9. extract-neo4j-concepts                            │
   │    Claude 개념·관계 추출                              │
   │    Cypher MERGE (Subject, Chapter, Concept,          │
   │                  Definition, Example, ...)            │
   │    학원·강사 ID 속성 강제                              │
   └─────────────────────────────────────────────────────┘
```

→ `lecture_chunk.rag_indexed_at` 업데이트로 인덱싱 완료 마킹.

---

## 2. BullMQ Job 정의

| Job 이름 | 멱등 키 | 재시도 | 비고 |
|---|---|---|---|
| `transcribe` | `lectureId` | 3회 (지수 백오프) | Google STT 실패는 드뭄 |
| `chunk-by-topic` | `lectureId` | 2회 | Claude 호출 비용 ↑ |
| `generate-tts` | `chunkId` | 3회 | Google TTS 호출, 청크 단위 |
| `generate-html` | `chunkId` | 2회 | Claude 호출 |
| `render-video` | `chunkId` | 1회 | HyperFrames 비용 ↑, 중복 호출 회피 |
| `upload-youtube` | `videoId` | 3회 (네트워크 에러만) | Quota 한계 시 즉시 abort |
| `notify` | `(lectureId, channel)` | 3회 | 알림톡 → SMS → 이메일 |
| `index-rag` | `chunkId` | 무한 | 낮은 비용, eventual consistency OK |
| `extract-neo4j` | `chunkId` | 무한 | 동일 |

각 job 은 멱등키로 중복 처리 차단. 청크 단위 작업은 lecture 의 chunk[] 를 fan-out 으로 처리.

---

## 3. SRT 가 단일 source 인 이유

SRT 는 **타이밍 정보 + 텍스트** 를 동시에 담는다. 이를 TTS / HTML 양쪽이 동일 source 로 쓰면:

| 사용처 | 활용 |
|---|---|
| Google TTS | 텍스트 → 오디오 (자연스러운 발화) |
| HyperFrames HTML | `class="clip" data-start data-duration` 에 SRT 타임라인 매핑 |

→ TTS 오디오의 발화 속도와 슬라이드 전환 timing 이 자동 정렬. FFmpeg lip-sync 같은 보정 불필요.

청크별 SRT 길이는 약 5~15분. 50분 강의 → 3~5 청크.

---

## 4. SLA 목표

| 단계 | 목표 시간 |
|---|---|
| 전체 (수업 종료 → publish) | < 90분 (p95) |
| transcribe (50분 audio) | < 10분 |
| chunk-by-topic + script | < 3분 |
| generate-tts (청크 1개) | < 1분 |
| generate-html (청크 1개) | < 2분 |
| render-video (청크 1개) | < 20분 |
| upload-youtube | < 5분 |
| index-rag | < 1분/청크 |

병렬 처리로 transcribe 후 단계 시간 단축. render-video 가 가장 긴 단계 → **청크 N개 병렬 = 총 시간 1/N**.

---

## 5. 실패 처리

### 5.1 자동 재시도

각 job 은 BullMQ 의 지수 백오프 재시도. 최대 재시도 횟수 초과 시 DLQ 진입.

### 5.2 강사·학원장 알림

| 시점 | 알림 |
|---|---|
| `transcribe` 최종 실패 | 강사: "STT 실패, 음성 품질 확인 요청" |
| `render-video` 최종 실패 | 강사·학원장: "영상 생성 실패, 수동 처리 필요" |
| `upload-youtube` 최종 실패 (Quota) | 학원장: "YouTube quota 초과, 1시간 후 재시도" |

### 5.3 부분 publish

청크 N개 중 1개 실패 시 **전체 lecture 실패** (v0.1 단순화).
- v0.5+ : 성공한 청크만 publish + 실패 청크 별도 재시도 검토.

---

## 6. 비용 cap (학원당)

`settings/limits.ts` 에 정의 (env override 가능):

| 항목 | 기본값 (학원당) |
|---|---|
| 일 처리 영상 수 | 6편 |
| 청크 최대 길이 | 15분 |
| 강의 최대 길이 | 60분 |
| Claude 호출 (lecture 1개) | $2 한도 |
| HyperFrames 청크 수 | 6개 |
| YouTube 업로드 (일) | 6편 |

한도 초과 시:
- 큐에서 **대기** (다음 날 자동 처리) — 강사·학원장 알림
- 또는 **명시적 거부** (학원장 설정)

---

## 7. HyperFrames 통합 상세

### 7.1 호출

```http
POST https://api.hyperframes.app/v1/renders
Authorization: Bearer {HYPERFRAMES_API_KEY}
Content-Type: application/json

{
  "html": "<!DOCTYPE html>...<audio src='https://s3.../chunk-3.mp3' />...",
  "duration_seconds": 300,
  "webhook_url": "https://api.academy-mvp.../webhooks/hyperframes"
}
```

응답: `{ render_id, status: queued }`

### 7.2 Webhook 수신

```http
POST /webhooks/hyperframes
{ render_id, status: ready|failed, mp4_url, duration_seconds }
```

서명 검증 → `video_asset(type=video_mp4)` 등록 → 다음 job 트리거.

### 7.3 HTML 생성 가이드 (Claude 프롬프트)

HyperFrames 의 HTML 컨벤션 (필수):
- `window.__timelines` 등록 (GSAP timeline)
- `<audio>` 는 `muted` 속성 없이 외부 src 참조
- 청크 요소는 `class="clip" data-start="..." data-duration="..." data-track-index="..."`
- `Math.random()` 금지 → seeded PRNG
- GSAP timeline 동기 구축

이 규칙들을 Claude 시스템 프롬프트에 명시해서 일관된 HTML 생성.

---

## 8. RAG 통합 (HTML 생성 단계)

`generate-html` job 의 입력:

```
<lecture_context>     ← 학원·강사·과목·학년
<chunk_transcript>    ← SRT 본문
<retrieved_materials> ← 강사 보유 자료 (Qdrant top-5)
<retrieved_past>      ← 같은 강사 과거 강의 (Qdrant top-3)
<concept_graph>       ← Neo4j 핵심/선수/관련 개념
<output_requirements> ← HyperFrames HTML 컨벤션
```

상세는 [`rag-strategy.md`](rag-strategy.md) 참조.

---

## 9. 멱등성 / 동시성

- 같은 lectureId 로 process-lecture 2번 호출 시 BullMQ 의 `jobId` 멱등키로 차단
- worker 재시작 시 in-progress job 은 visibility timeout 후 자동 재배정
- chunk 청크별 fan-out 은 `Promise.all` 또는 별도 큐 + barrier
- HyperFrames webhook 은 idempotent (같은 render_id 중복 수신 시 무시)

---

## 10. 관측성 (MVP 최소)

| 메트릭 | 출처 |
|---|---|
| 파이프라인 단계별 latency | BullMQ job duration |
| 각 외부 API 호출 latency / 실패율 | adapter 단위 로깅 |
| 학원별 일 처리량 | DB 집계 (lecture status 별) |
| 비용 (학원·일) | settings/limits.ts 의 임계값 대비 추적 |

로깅 표준: `@aiagent/logger` (pino 기반, content-api 재사용).
모니터링 스택 (Loki/Prometheus/Grafana) 은 v0.5+ Phase에서 도입.
