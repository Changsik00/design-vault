# Phase 로드맵

> 작성일: 2026-05-23
> v1 (12주 MVP) / v2 (16주 MVP) 모두 과스코프. **v0.1 → v0.5 → v1.0 → v2.0 으로 4단계 분할**.

---

## 0. 단계 모델

```
v0.1 MVP (8주)        — 핵심 파이프라인 단대단 + RAG 인프라
   │ 1~2 학원 파일럿
   ▼
v0.5 (8주)            — 운영 대시보드 + 학원장 RAG Q&A + 면담 prep
   │ 5~10 학원 파일럿
   ▼
v1.0 (12주)           — 학생 RAG 채팅 + 강사 평가 + 학부모 PWA β
   │ 30 학원 파일럿
   ▼
v2.0 (16주)           — Android 자동 녹음 + 글로벌 RAG + 본사 통합
   │ 100+ 학원 상용
   ▼
v3.0+                 — Enterprise / 광고 수익 / 인접 시장
```

각 단계의 졸업 조건은 **숫자로 못 박는다**. 미달 시 다음 단계 보류.

---

## 1. v0.1 — 핵심 파이프라인 (8주)

### 가치
> "강사가 음성 올리면 학원 YouTube 채널에 영상이 자동 publish 된다."

### Deliverable
- 강사 음성 업로드 (Vite SPA)
- Google STT + Claude 청킹 + Google TTS + HyperFrames + YouTube
- RAG 인프라 (Qdrant + Neo4j + 인덱싱)
- 학원장 검수 화면 (수동 default)
- 알림톡 + 이메일 + SMS fallback
- 강사 자료 옵션 업로드 (RAG seed)
- Firebase Auth + CASL + trust_relationship
- **`student` 테이블** — 알림 대상(F20) + PIPA 동의(F25) 필수 (v0.1 포함)
- **`usage_log` 테이블** — Claude/STT 비용 cap 집계(F24) 필수 (v0.1 포함)

### 졸업 조건
- 파이프라인 단대단 동작
- 영상 자동 업로드 성공률 ≥ 90%
- RAG 인덱싱 누락 0%
- 파일럿 학원 2곳 × 4주 × 30편 publish

---

## 2. v0.5 — 학원장 운영 가치 (8주)

### 가치
> "학원장이 자연어로 학원 데이터를 묻고, 학생 면담 prep 을 자동으로 받는다."

### Deliverable
- 학원장 자연어 질의 채팅 (RAG-aug)
- 학생 면담 prep 자동 생성 (출결 + 평가 + RAG 검색)
- 학원장 운영 대시보드 5 P0 화면 (오늘 수업 · 출결 · 알림 · 데이터 누적 · X2 Dashboard)
- 출결·메모 입력
- OCR 보드 스냅샷 (Google Vision) → RAG 인덱싱 확장
- Neo4j 시각화 차트 1개 (학원 전체 개념 맵)
- 강사 자료 자동 매칭 가이드 (UX)

### 졸업 조건
- 학원장 일평균 사용 ≥ 5분
- RAG Q&A 응답 p95 ≤ 8초
- 5 P0 화면 사용성 테스트 통과 (5학원)
- 5~10 학원 파일럿 30일 연속

---

## 3. v1.0 — 학생·강사·학부모 가치 (12주)

### 가치
> "학생이 RAG 채팅으로 학원 자산을 묻고, 강사는 5축 평가로 자가 진단하며, 학부모는 PWA 로 자녀 영상을 본다."

### Deliverable
- 학생 RAG 채팅 UI (ChatGPT 스타일)
- YouTube 모달 시청 + PDF/SRT 다운로드
- 강사 평가 5축 (평점 · 업로드 이행률 · 수업 완주율 · RAG 활용도 · 조회수)
- `teacher_review` (학생 익명 평점) + 모더레이션
- 학부모 PWA β (자녀 영상 큐레이션)
- Neo4j 시각화 (강사 자가 진단 · 학생 학습 경로)
- 강사별 RAG namespace 강화

### 졸업 조건
- 학생 RAG 채팅 주간 활성 비율 ≥ 50%
- 학원 채널 평균 월 구독자 +30명 (Pro 등급)
- 영상 attribution 신규 등록률 ≥ 15%
- 30 학원 파일럿 6개월 운영

---

## 4. v2.0 — 자동화 + 확장 (16주)

### 가치
> "강사가 칠판 앞에 앉기만 하면 수업이 자동 녹음되어 영상이 된다."

### Deliverable
- **Android 앱 (별도 프로젝트)** — 1-tap 수업 시작, 자동 녹음·판서 캡처·업로드
- 글로벌 RAG (k-anonymity ≥ 5, 학원 간 익명 집계)
- 본사 통합 채널 (다지점 자동 큐레이션)
- 영상 자동 분석 (강사 자가 진단 강화)
- 자녀 등장 동의 + 모자이크 파이프라인
- YouTube 댓글 모더레이션

### 졸업 조건
- Android 앱 자동 업로드 성공률 ≥ 95%
- 학원 100개 + Enterprise 본사 5개사
- ARR 10억원 + Net Retention ≥ 110%

---

## 5. v3.0+ — Enterprise + 인접 시장

- Operation Policy Agent (학원법 룰 엔진)
- Enterprise Plan (SSO/SAML, SLA 99.95%)
- 학원 간 익명 벤치마크 상품화
- YouTube Partner 광고 수익화
- IoT 자동 캡처 (교실 카메라/마이크)
- 인접 시장 (공부방, 직무교육)

졸업 기준: **ARR 100억 / SOM 5% / Net Retention 130%+**.

---

## 6. 단계 간 의존성

```
v0.1 → v0.5 : RAG 인프라 (Layer 0) 가 v0.1 에 있어야 v0.5 의 학원장 Q&A 가능
v0.5 → v1.0 : 운영 데이터 (출결·메모·OCR) 축적 후 강사 평가/학생 RAG 의미
v1.0 → v2.0 : 학원 30+ 파일럿 검증 후 Android 별도 프로젝트 자원 투입
```

v0.1 의 RAG 가 v0.5/v1.0 모든 가치의 **기반 인프라**. RAG 없으면 그 위의 모든 UI 가치 X.

---

## 7. 미정 항목 (단계별 결정)

| 항목 | 결정 시점 |
|---|---|
| HyperFrames 단가 협상 | v0.1 W4 PoC 결과 + v0.5 진입 전 |
| 자체 TTS (Kokoro 등) 이행 | v1.0 진입 시 비용 평가 |
| Neo4j Aura 마이그레이션 | v1.0 운영 부담 평가 시점 |
| Korean OAuth (Kakao/Naver) | v0.5 학원 가입 마찰 측정 후 |
| LangGraph.js 도입 | v2.0 multi-step 에이전트 필요 시 |
| Python LangGraph (최후 대안) | v3.0+ 복잡 에이전트 한계 도달 시 |
