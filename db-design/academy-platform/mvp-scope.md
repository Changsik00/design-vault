# MVP (v0.1) 스코프

> 작성일: 2026-05-23

---

## 1. 가치 명제

> **강사가 수업 오디오를 업로드하면, RAG 가 분석해서 슬라이드 영상이 만들어지고, 학원장의 검수를 거쳐 학원 YouTube 채널에 자동 업로드되며, 학부모에게 알림톡이 발송된다.**

이 한 줄이 v0.1 의 전부. 그 외 모든 가치 (학생 RAG 채팅, 운영 대시보드, Android 앱 등) 는 후속 단계로 분리.

---

## 2. In (MVP 포함)

### 2.1 파이프라인
- 강사 음성 업로드 (Vite SPA, drag&drop)
- Google STT V2 → transcript with timestamps
- Claude 토픽 청킹 + 스크립트 정리 → 청크별 SRT
- Google TTS Neural2 → 청크별 MP3 (한국어 나레이션)
- Claude HTML 생성 (HyperFrames 포맷, GSAP clip 동기) — RAG-augmented
- HyperFrames API 병렬 렌더 → 청크별 MP4
- 학원장 검수 화면 (수동 검수 default)
- YouTube Data API v3 자동 업로드
- 학부모 알림톡 (영상 링크) + 이메일 + SMS fallback

### 2.2 RAG 인프라 (Layer 0)
- Qdrant 인덱싱 (org_id + teacher_id payload)
- Neo4j 개념 그래프 (8 노드 타입 + org_id 격리)
- Upstage Solar Embedding (1024-dim, 한국어)
- 인덱싱 파이프라인 (강의 publish 시 자동)
- 강사 RAG seed 자료 옵션 업로드 (`lecture.type='material'`)

### 2.3 인증·권한
- Firebase Auth (이메일/비밀번호)
- CASL ability 레이어 (RBAC + ABAC 확장 대비)
- `trust_relationship` 테이블 (directorGrant)
- 멀티테넌트 `org_id` 강제 (NestJS Interceptor)

### 2.4 데이터
- MySQL: identity_db 8테이블 + academy_db 10테이블 (usage_log 파티셔닝 포함)
- S3 파일 저장 (audio, mp3, mp4, thumbnail)
- BullMQ + Redis 큐
- AWS KMS (YouTube refresh_token 암호화)

### 2.5 UI 화면 (최소)
- 로그인 (Firebase)
- 강사 업로드 화면 (audio + 메타: 학원, 반, 날짜)
- 강사 자료 업로드 화면 (옵션 RAG seed)
- 학원장 검수 화면 (영상 미리보기 + 승인/반려)
- 학원장 YouTube OAuth 연결 화면 (최초 1회)
- 발송 결과 / 파이프라인 상태 조회

---

## 3. Out (MVP 제외, 후속 단계로)

| 항목 | 위치 |
|---|---|
| Android 앱 (자동 녹음·업로드) | v2.0 (별도 프로젝트) |
| 학생 RAG 채팅 UI | v1.0 |
| 학원장 자연어 질의 (Q&A 채팅) | v0.5 |
| 학생 면담 prep 자동 생성 | v0.5 |
| 강사 평가 5축 (평점·이행률·조회수·완주율·RAG) | v1.0 |
| 학원장 운영 대시보드 (5 P0 화면 — 오늘 수업/출결/알림 등) | v0.5 |
| 출결·메모 입력 화면 | v0.5 |
| OCR 보드 스냅샷 (Google Vision) | v0.5 |
| Neo4j 개념 그래프 시각화 (D3.js / Cytoscape) | v0.5 (학원장 1개 차트), v1.0 (강사/학생) |
| 학부모 PWA β | v1.0 |
| 글로벌 RAG (k-anonymity ≥ 5) | v2.0 |
| Enterprise 본사 통합 채널 | v3.0 |
| 광고 수익화 (YouTube Partner) | v3.0+ |

---

## 4. 단순화 가정 (MVP 시점)

| 가정 | 비고 |
|---|---|
| 한국어 only | 다국어는 v3.0+ 검토 |
| KST timezone 고정 | 멀티 지역은 차후 |
| 학원당 YouTube 채널 1개 | 1:N 채널 (강사별) 은 v0.5+ |
| 강의 길이 ≤ 60분 | 60분 초과는 chunk-splitter 보완 필요 |
| 강사 업로드 = 웹 drag&drop | 모바일 자동 녹음 = Android v2.0 |
| 검수 default = 수동 | 자동 publish = 강사별 trust_relationship 부여 시만 |
| 학생/학부모 직접 접근 없음 | 학부모는 알림톡 + YouTube 링크로만 |
| 본인 학원 데이터만 격리 | 학원 간 cross 는 v2.0 글로벌 RAG |

---

## 5. v0.1 졸업 조건 (Definition of Done)

다음 모두 충족 시 v0.5 진입:

- [ ] 파이프라인 단대단 동작 (음성 업로드 → YouTube publish)
- [ ] 영상 생성 SLA: 50분 강의 → 90분 이내 publish (p95)
- [ ] 영상 자동 업로드 성공률 ≥ 90%
- [ ] HyperFrames render 실패율 ≤ 5%
- [ ] STT 실패율 ≤ 5%
- [ ] RAG 인덱싱 누락률 = 0% (모든 publish 영상 indexed)
- [ ] 멀티테넌트 격리 100% 통과 (E2E 학원 A → B 데이터 절대 노출 X)
- [ ] 비용 cap (학원당 일 처리 한도) 정상 작동
- [ ] 파일럿 학원 2곳 동행 운영 4주 + 실 강의 30편 publish

미달 시 v0.1 내 보강.

---

## 6. 8주 일정 (예시)

| 주차 | 작업 |
|---|---|
| W1 | 인프라 (docker-compose, monorepo 셋업, Firebase Auth, MySQL 마이그레이션 첫 셋) |
| W2 | 강사 업로드 + S3 + class_session API + BullMQ 큐 |
| W3 | STT + 청킹 + SRT 생성 (Claude) |
| W4 | TTS + HyperFrames HTML 생성 + 첫 렌더 PoC |
| W5 | RAG 인프라 (Qdrant + Neo4j + 인덱싱 파이프라인) |
| W6 | YouTube OAuth + 업로드 + 학원장 검수 화면 |
| W7 | 알림톡 + 이메일 + 학원 가입/온보딩 |
| W8 | 파일럿 학원 2곳 동행 + 보완 |

일정은 가이드. 실제 진행은 spec 단위 PR.

---

## 7. MVP 가 아닌 것 — 명시적 배제

> **"학원의 모든 운영"** 을 자동화하려 하지 않는다. 본 MVP 의 가치는 **수업 → 영상 → 채널** 의 단대단 자동화 한 가지.

출결, 결제, 학생 관리, 학부모 응대 등은 v0.5 이후 단계적 추가 (또는 영원히 미포함). 학원장이 기존 도구(클래스업, 엑셀, 카톡)를 그대로 쓰면서 본 제품을 **얹어 쓰는 형태** 가 MVP 시점의 기본 가정.
