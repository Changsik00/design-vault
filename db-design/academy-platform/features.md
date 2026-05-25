# 핵심 기능 카탈로그

> 작성일: 2026-05-23
> 사용자 가치 단위로 본 6 핵심 기능. 각 기능의 Phase 매핑 + 의존성 + UI 흐름.

---

## 0. 기능 맵

```
[v0.1 MVP]
  1. YouTube 자동 파이프라인 ⭐ 핵심 차별점
       └ 강사 자료 업로드 (RAG seed)
       └ 학원장 검수
       └ 학부모 알림톡

[v0.5]
  2. 학원장 자연어 질의 (운영 Q&A)
  3. 학생 면담 prep RAG
  4. 운영 대시보드 5 P0
  5. 출결 / 메모 / OCR 보드 인덱싱

[v1.0]
  6. 학생 RAG 채팅 + YouTube 모달
  7. 강사 평가 5축
  8. 학부모 PWA β

[v2.0]
  9. Android 자동 녹음 (별도 프로젝트)
  10. Neo4j 시각화 (강사 자가 진단 / 학생 학습 경로)
  11. 글로벌 RAG (학원 간 익명 집계)
```

각 기능은 **RAG 인프라 (Layer 0)** 위에 얹는다. 인프라는 v0.1 부터 항상 켜져 있음.

---

## 1. YouTube 자동 파이프라인 ⭐ (v0.1)

> **한 줄**: 강사가 수업을 시작하기만 하면, 약 90분 안에 학원 YouTube 채널에 강의 영상이 올라간다.

### 1.1 사용자 시나리오 (Happy Path)

```
17:55  강사 박민준, A학원 강의실 도착, 강의 시작
       (MVP: 수업 후 음성 파일을 강사 화면에서 업로드)
       (v2.0: Android 앱이 자동 녹음 + 판서 캡처)

18:00~18:50  수업 진행

18:51  수업 종료. 강사 음성 파일 업로드
       → S3 PUT → lecture row INSERT → BullMQ 큐

18:51~19:01  Google STT (50분 → ~10분)
19:01~19:04  Claude 청킹 + SRT 생성
19:04~19:06  병렬: Google TTS (청크별 MP3) + Claude HTML 생성
19:06~19:25  병렬: HyperFrames 렌더 (청크 3개)
19:25  학원장 검수 대기 (수동 default)
19:30  학원장 1-tap 승인 → YouTube 업로드 (~5분)
19:35  publish 완료 → 알림 발송

20:00  학부모 알림톡: "[A학원] 김민서 학생 오늘 수업 영상 📺 [링크]"
       학생 PWA push: "오늘 수업 정리됨"
       강사 push: "영상 업로드 완료"

전체 소요: 약 100분 (목표 SLA 90분 이내, 검수 자동화 시)
```

### 1.2 학원장 설정 (최초 1회)

**채널 연결:**
```
설정 → 외부 연동 → "YouTube 채널 연결"
   ↓ Google OAuth (scope: youtube.upload + youtube.readonly + youtube)
   ↓ 학원 채널 선택
   ↓ youtube_channel 레코드 생성 + refresh_token (KMS 암호화)
```

**자동 업로드 정책:**
- `review_required` (default, MVP 권장)
- `auto` (전체 자동, 신뢰 학원만)
- `per_teacher` (강사별 trust_relationship 토글)

**기본 공개 범위:**
- `public` (SEO 노출 + 채널 페이지 자동 게시)
- `unlisted` (링크 보유자만, 기본 권장)
- `private` (학원 내부만)

### 1.3 강사 인터페이스

**업로드 화면 (MVP):**
```
[강의 업로드]
오디오 파일 [drag & drop]
학원 / 반 [드롭다운]
수업 일시 [날짜 선택]
과목 / 학년 [선택]
[업로드]
```

**검수 요청 (수동 default):**
- 영상 생성 완료 → 학원장 큐 등재
- 강사가 trust_relationship `auto_publish_own` 보유 시 → 자동 publish

**본인 영상 통계 (v1.0):**
- 누적 영상 수 / 총 조회수 / 평균 시청 시간
- RAG 참조 횟수 (학생이 본인 영상 N회 참조)
- 평점 평균 (학생 익명 평가)

### 1.4 학원장 인터페이스

**영상 검수 화면:**
```
검수 대기 영상 N건

▶ [중3 수학] 이차함수 정점
  강사: 박민준 · 5/21 18:00~18:50
  
[썸네일 후보 3종]
[제목 편집] [설명 편집]
공개 범위: [unlisted ▾]

[승인] [반려: 사유 입력]
```

### 1.5 강사 자료 사전 업로드 (RAG seed, 옵션)

`lecture.type='material'` 로 별도 업로드:
- 강의안 PDF
- 보드 사진
- 예제 문제 / 정의 카드

→ Qdrant 인덱싱 → 후속 강의의 HTML 생성 시 RAG 컨텍스트로 자동 활용.
가이드 텍스트 + UX 는 [`rag-strategy.md`](rag-strategy.md) §6 참조.

### 1.6 메타데이터 자동 생성

**제목 템플릿:**
```
[<학년> <과목>] <단원명> — <부제 30자 이내>
예: [중3 수학] 이차함수 정점 — y=ax²+bx+c 변환 5문제
```

**설명 템플릿:**
```
{class_date} {teacher_name} 강사 · {academy_name}

이번 수업 내용:
- {bullet_1}
- {bullet_2}

타임라인:
{srt_timestamps}

PDF: {pdf_url}
자막: {srt_url}

#{tag1} #{tag2} #{tag3}

* 본 영상은 AI 보조로 생성되었습니다.
```

**썸네일:**
- Claude로 문구 후보 3종 → 학원 브랜딩 합성 → PNG 3장
- 기본값 #1 자동 선택 (강사·학원장 변경 가능)

### 1.7 검수 / 거버넌스

**LLM-as-judge 자동 검열** (publish 직전):
- 부적절 발언 (욕설/혐오)
- 학생 개인정보 노출 (이름·전화·주소)
- 자녀 등장 미동의 학생 음성/이미지 (Phase 2 카메라 모드 시)

검출 시 → `status='pending_review'` 강제 + 학원장 알림.

**사후 처리:**
- publish 후 부적절 발견 → 학원장 1-tap "비공개 전환"
- YouTube API `videos.update(privacyStatus=private)` 즉시 호출

### 1.8 실패 처리

| 단계 | 실패 시 |
|---|---|
| 업로드 | 강사 재업로드 (브라우저 진행률) |
| STT | 3회 재시도 → 강사 알림 + 수동 텍스트 입력 옵션 |
| Claude (청킹/HTML) | 2회 재시도 → 강사·학원장 알림 |
| TTS | 3회 재시도 |
| HyperFrames | 1회 재시도 → 강사·학원장 알림 + 수동 처리 큐 |
| YouTube upload | 3회 재시도 (Quota 초과 시 1시간 후) |

### 1.9 KPI

| 지표 | 목표 (v0.1 졸업) |
|---|---|
| 영상 생성 SLA (p95) | < 90분 |
| 영상 자동 업로드 성공률 | > 90% |
| HyperFrames render 실패율 | < 5% |
| 평균 검수 소요 시간 (수동) | < 2분/영상 |
| 학원 채널 월 신규 영상 (Pro) | > 120편 (v1.0 목표) |
| 학부모 영상 링크 클릭률 | > 50% (v1.0 목표) |

---

## 2. 학원장 자연어 질의 (v0.5)

> **한 줄**: 학원장이 "이번 달 학생들이 가장 어려워한 단원?" 같은 질문을 자연어로 던지면, RAG 가 학원 자산을 검색해서 답한다.

### 2.1 시나리오

```
[학원장 PWA]
  Q: "이번 달 학생들이 가장 어려워한 단원?"
       ↓
  RAG: rag_query_log + Neo4j 분석
       ↓
  A: "5월 RAG 질의 패턴 분석:
      1. 이차함수 정점 (42 질의, 학생 18명)
      2. 판별식 (28 질의, 12명)
      3. ...
      → 콘텐츠 보강 추천: 김지영 강사의 5/21 이차함수 강의 활용"
```

### 2.2 다른 질의 예

- "박민준 강사 이번 주 수업 핵심 정리해줘"
- "지난 학기와 이번 학기 강의 비교"
- "학생 김민서가 어려워한 개념?"
- "이번 달 신규 영상 attribution?"

### 2.3 의존성

- RAG 인프라 (v0.1 부터)
- 출결/메모 인덱싱 (v0.5 신규)
- `rag_query_log` 누적 (v0.5 신규)

---

## 3. 학생 면담 prep RAG (v0.5)

> **한 줄**: 학원장이 "학생 김민서 면담 준비해줘" 라고 하면, 출결·평가·RAG 질의 패턴·강의 참여를 종합 분석한 prep 문서 자동 생성.

### 3.1 시나리오

```
[학원장 PWA] → 학생 카드 클릭 → "면담 prep 자동 생성"

[Claude + RAG]
  - 출결 패턴 (지난 3개월)
  - RAG 질의 이력 (자주 물은 개념 = 약점 단원)
  - 강사 코멘트 (memo)
  - YouTube 시청 이력 (어느 영상 자주 봤나)
       ↓
  prep 문서:
  ## 김민서 학생 면담 prep (2026-06-15)
  
  ### 출결 & 참여
  - 출석률 88%, 결석 4회 (전월 대비 ↑)
  - 수업 후 RAG 채팅 18회 (학원 평균 ↑)
  
  ### 약점 단원
  - 이차함수 정점: 7회 질의 (집중 약점)
  - 판별식: 3회
  
  ### 강사 관찰
  - 박민준: "응용 문제에서 막힘"
  - 김지영: "기초 개념은 안정적"
  
  ### 추천 액션
  - 이차함수 보강 강의 영상 시청 권장
  - 다음 주 보충 수업 권유
```

### 3.2 의존성

- `rag_query_log` (v0.5)
- `attendance` (v0.5)
- `memo` (v0.5)

---

## 4. 운영 대시보드 5 P0 (v0.5)

> **한 줄**: 학원장이 5분 안에 학원 상태 파악 가능한 통합 화면 5종.

| 화면 | 내용 |
|---|---|
| S-01 오늘 수업 | 수업 목록 + 출결/업로드 상태 |
| S-02 출결 현황 | 학원 전체 출결률 + 결석 학생 |
| S-06 데이터 누적 | 영상 / RAG / 운영 데이터 축적률 |
| S-07 운영 알림 | 운영 흐름 끊김 통합 (검수 대기·업로드 실패·미입력 출결 등) |
| S-08 X2 토글 | 학원장 view ↔ 강사 view 컨테이너 |

상세 PRD 는 originals/02-prd-screen-design.md 참고. v0.5 진입 시 정식 spec 작성.

---

## 5. OCR / 출결 / 메모 (v0.5)

> 운영 데이터 수집 확장. RAG 인덱싱 범위 ↑.

- **OCR 보드 스냅샷**: Google Vision (이미 monorepo 안에 있는 패턴) → 텍스트 추출 → `lecture_chunk` 인덱싱
- **출결**: 1-tap 입력 (강사 모바일)
- **메모**: 학생별 상담 메모 + tsvector 검색

→ 모두 Qdrant + Neo4j 에 인덱싱 → 학원장 질의 / 학생 RAG 의 컨텍스트 풍부 ↑.

---

## 6. 학생 RAG 채팅 (v1.0)

> **한 줄**: 학생이 "오늘 수학 뭐 배웠어?" 를 물으면, 학원 자산 기반으로 답하고 YouTube 영상 카드를 보여준다.

### 6.1 UX 흐름

```
[학생 PWA]
  메인 화면
    ↓
  채팅 입력 + 빠른 질문 카드 (오늘 뭐 배웠어 / 시험 범위 / 이번 주 숙제 / ...)
    ↓
  학생: "오늘 수학 뭐 배웠어?"
    ↓
  AI: "5/21 김지영 강사 중3 수학반 이차함수 정점 ...
       y = ax²+bx+c → y = a(x-p)²+q 변환 5문제"
  
  📺 [영상 카드] 이차함수 정점 변환
  📄 [PDF] 강의노트
  📝 [SRT] 자막
       ↓
  카드 탭 → 모달 (YouTube embed, 앱 떠나지 않음)
```

### 6.2 학원 RAG vs 강사 RAG (학생 화면)

| 모드 | 필터 |
|---|---|
| 기본 (학원 RAG) | `org_id` + `visibility != private` + `student.enrolled_classes` |
| 강사 한정 ("김지영 강사 강의만") | + `teacher_id = 김지영` |

### 6.3 안전장치

- 일 30 질의 / 월 500 질의 (학원장 조정 가능)
- LLM-as-judge 부적절 질문 거부
- 답안 직접 제공 정책상 제한 (학원 인가 시만)
- 다른 학원 데이터 절대 노출 X (E2E 테스트 100%)
- 만 14세 미만 법정대리인 동의 필수

상세는 [`rag-strategy.md`](rag-strategy.md).

---

## 7. 강사 평가 5축 (v1.0)

> **한 줄**: 강사를 평점 + 업로드 이행률 + 완주율 + RAG 참조 + 조회수의 5축으로 평가하고, 학원장이 스케줄을 픽스한다.

### 7.1 5축 정의

| 축 | 가중치 (기본) | 데이터 소스 |
|---|---|---|
| 학생 평점 | 30% | `teacher_review` (익명, 월 1회) |
| 업로드 이행률 | 25% | `class` × `youtube_video.published_at` 72h |
| 수업 완주율 | 20% | `class.actual_end_at` vs `scheduled_end_at` |
| RAG 활용도 | 15% | `rag_query_log.referenced_videos` |
| YouTube 조회수 | 10% | YouTube Data API v3 |

학원장이 ±10%p 조정 가능.

### 7.2 종합 점수

```
composite_score =
    0.30 * normalize(student_rating, 1, 5)
  + 0.25 * upload_rate
  + 0.20 * class_completion_rate
  + 0.15 * normalize(rag_reference, p10, p90)
  + 0.10 * normalize(youtube_views, p10, p90)
(0 ~ 100 스케일)
```

### 7.3 스케줄 픽스 워크플로

```
강사: "매주 화목 19:00 추가 요청"
   ↓ teacher_schedule_request INSERT
학원장: 알림톡 → 1-tap → 평가 점수 표시
   - composite_score 78 → 노란불
   - 충돌 검사 OK
   - [승인] [반려] [협상]
   ↓
승인 → teacher_availability UPDATE + 강사 알림
```

---

## 8. Android 자동 녹음 (v2.0, 별도 프로젝트)

> **한 줄**: 강사가 칠판 앞에 앉기만 하면 수업이 자동 녹음되고 영상이 된다.

### 8.1 별도 프로젝트 명시

- 이 monorepo 에는 **명세만** 보관
- 별도 Git repo + 별도 팀 + 12주 일정 (백엔드 v2.0 과 동기)
- 기술 스택: Kotlin + Jetpack Compose + WorkManager + Room + Retrofit + mTLS + FCM

### 8.2 핵심 기능

- 수업 시작 1-tap (오늘 스케줄 카드 → 탭)
- MediaRecorder 자동 녹음 (ForegroundService)
- CameraX 판서 스냅샷 (60초 주기 또는 수동 탭)
- WorkManager 자동 업로드 (Wi-Fi 감지 시)
- 출결 1-tap
- 영상 검수 (모바일)
- 진행도 표시 (FCM push)

### 8.3 백엔드 계약 (`apps/academy-api`)

```
POST /v1/devices/register
GET  /v1/teachers/me/today-schedule
POST /v1/class-sessions
POST /v1/class-sessions/:id/finalize
GET  /v1/class-sessions/:id/pipeline-status
```

상세는 historical v2/05-features/02-android-app-spec.md 참조 (Phase 2 시점에 v3 spec 으로 정식 작성).

---

## 9. Neo4j 시각화 (v1.0+)

> 학원의 지식 자산을 그래프로 본다.

- 학원 전체 개념 맵 (학원장)
- 강사 본인 강의 coverage (강사 자가 진단)
- 학생 학습 경로 (학생 PWA)

D3.js 또는 Cytoscape.js. 노드 타입별 색상 (Subject=red, Chapter=orange, Concept=blue, Definition=gray).
온톨로지 상세는 [`data-model.md`](data-model.md) §4.

---

## 10. 글로벌 RAG (v2.0+)

> 학원 간 익명 집계.

- k-anonymity ≥ 5 보장
- 별도 collection (글로벌)
- 학원장 동의 시 참여
- 사용처: "다른 학원 강사 평균 평점", "유사 학원의 인기 단원"

---

## 11. 기능 의존성 그래프

```
[v0.1 인프라]
  ├─ Firebase Auth
  ├─ MySQL 9 테이블 + Qdrant + Neo4j
  ├─ BullMQ 워커 + S3
  └─ 알림톡 / SES / SMS

[v0.1 핵심]
  └─ YouTube 자동 파이프라인
      ├─ 강사 자료 (RAG seed)
      ├─ 학원장 검수
      └─ 학부모 알림톡

[v0.5]
  ├─ 학원장 자연어 질의 ──┐
  ├─ 학생 면담 prep ──────┤── 의존: rag_query_log + memo + 출결
  ├─ 운영 대시보드 5 P0 ───┘
  └─ OCR / 출결 / 메모 (인덱싱 확장)

[v1.0]
  ├─ 학생 RAG 채팅 ───── 의존: 학원 RAG 누적
  ├─ 강사 평가 5축 ───── 의존: teacher_review + rag_query_log + YouTube 조회수
  └─ 학부모 PWA β ────── 의존: 학생 RAG namespace

[v2.0]
  ├─ Android 자동 녹음 ── 별도 프로젝트
  ├─ Neo4j 시각화 ────── 의존: 충분한 그래프 누적
  └─ 글로벌 RAG ──────── 의존: 학원 50+

[v3.0+]
  ├─ Enterprise (본사 통합)
  ├─ YouTube Partner (광고 수익)
  └─ IoT 자동 캡처
```

상세 Phase 일정은 [`phase-roadmap.md`](phase-roadmap.md).
