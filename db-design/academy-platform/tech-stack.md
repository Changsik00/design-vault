# 기술 스택 (v0.1 MVP)

> 작성일: 2026-05-23
> 모든 결정은 최종 확정 상태. 변경 시 본 문서 + 관련 v3/ 문서 동시 갱신.

---

## 1. 핵심 결정 요약

| 영역 | 결정 | 비고 |
|---|---|---|
| **Frontend** | Vite + React + React Router (SPA) | server-side 일체 미사용 (디버깅 단순화) |
| **API + Worker** | NestJS 단일 앱 — `apps/academy-api` | HTTP + BullMQ 같이. 부하 늘면 분리 |
| **Language** | TypeScript only | Polyglot 운영 부담 회피 |
| **LLM Orchestration** | Claude / Google / OpenAI SDK 직접 | LangGraph 차후 (필요 시 Node 우선) |
| **RDB** | MySQL 8.0 | content-api 와 일관 |
| **Vector DB** | Qdrant Community (Docker) | Cloud 옵션 차후 |
| **Graph DB** | Neo4j Community (Docker) | Aura 옵션 차후 |
| **Embedding** | Upstage Solar (1024-dim, 한국어) | |
| **STT** | Google Cloud Speech-to-Text V2 | content-api 와 같은 GCP 프로젝트 |
| **OCR** | Google Vision | (MVP 미사용, v0.5 도입) |
| **TTS** | Google Cloud TTS Neural2 ko-KR | 무료 1M char/월 → 초과 시 $16/1M |
| **Video 합성** | HyperFrames (HTML + GSAP) | 청킹 병렬 렌더로 latency 단축 |
| **Upload** | YouTube Data API v3 | OAuth refresh_token + KMS |
| **Notification** | 카카오 알림톡 (NHN) + 이메일 (SES) + SMS (SOLAPI) | fallback chain |
| **Queue** | BullMQ + Redis | |
| **Storage** | AWS S3 | content-api 와 동일 |
| **Auth (Layer 1+2)** | Firebase Auth + `packages/identity` (회사 표준) | identity_user + membership tuple. [`identity-policy.md`](identity-policy.md) |
| **Auth (Layer 3)** | @casl/ability + trust_relationship | academy-api 의 비즈니스 룰. [`auth-and-policy.md`](auth-and-policy.md) |
| **Settings** | content-api 의 settings facade 패턴 (env + config + zod) | |

---

## 2. monorepo 위치

```
aiagent-monorepo/
├── apps/
│   ├── content-api/       (기존 — STT/OCR/Vision 패턴 참고)
│   ├── content-monitor/   (기존)
│   ├── academy-api/       ← NEW  · NestJS (HTTP + BullMQ Worker)
│   └── academy-web/       ← NEW  · Vite + React SPA
└── packages/              (기존 + 차후 추출)
    ├── identity/          ← NEW  · 회사 표준 identity (Layer 1+2)
    │                         Firebase Auth verifier + identity_user/membership
    │                         + Custom Claims sync. 신규 service 의 표준 import
    ├── aws/               (S3, 재사용)
    ├── gcs/               (Google Cloud Storage, 재사용)
    ├── logger/            (재사용)
    ├── config-*           (4종 재사용)
    └── (차후) @aiagent/stt, @aiagent/ocr  ← content-api 안정화 후 추출
```

**`packages/identity` 가 핵심**:
- 회사 전 product 가 공유하는 표준 인증·membership 패키지
- academy-api 가 첫 reference 구현
- 신규 서비스는 본 패키지 import 만으로 인증·권한 표준 채택
- 상세 정책: [`identity-policy.md`](identity-policy.md)

content-api 는 aiagent-api 도메인(SP `SSP_AGENT_CONTENT_UPDATE`, 회원 idx 등)에 묶여있어 academy-api 가 직접 호출하지 않는다. STT/OCR 로직은 패턴 차용 → 차후 packages 추출. content-api 도 표준화 시 `packages/identity` 채택 가능.

---

## 3. Docker 환경

### 로컬 개발 (`docker-compose.yml`)

```yaml
services:
  mysql:
    image: mysql:8.0
    ports: ["3306:3306"]
    volumes: [mysql_data:/var/lib/mysql]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  qdrant:
    image: qdrant/qdrant:latest
    ports: ["6333:6333", "6334:6334"]
    volumes: [qdrant_data:/qdrant/storage]

  neo4j:
    image: neo4j:5-community
    ports: ["7474:7474", "7687:7687"]
    environment:
      NEO4J_AUTH: neo4j/dev_password
    volumes: [neo4j_data:/data]

  academy-api:
    build: ./apps/academy-api
    depends_on: [mysql, redis, qdrant, neo4j]
    environment:
      DATABASE_URL: mysql://...
      REDIS_URL: redis://redis:6379
      QDRANT_URL: http://qdrant:6333
      NEO4J_URL: bolt://neo4j:7687
      GOOGLE_APPLICATION_CREDENTIALS: /run/secrets/gcp.json
    # 민감 키는 .env (gitignored) 또는 K8s Secret 으로 주입
    # - HYPERFRAMES_API_KEY
    # - ANTHROPIC_API_KEY
    # - FIREBASE_ADMIN_KEY
    # - NHN_API_KEY, SOLAPI_KEY, UPSTAGE_API_KEY ...
    env_file: [.env.local]
    ports: ["3000:3000"]

  # academy-web 은 Vite 개발 서버 — 보통 호스트에서 직접 실행 (HMR 빠름)

volumes:
  mysql_data:
  qdrant_data:
  neo4j_data:
```

### 프로덕션

| 컴포넌트 | 배포 |
|---|---|
| `academy-web` (Vite SPA static) | S3 + CloudFront |
| `academy-api` (HTTP + Worker) | Docker image → ECR → EKS |
| MySQL | AWS RDS |
| Redis | AWS ElastiCache |
| Qdrant | EC2 자체 호스팅 (Phase 2 에서 Qdrant Cloud 검토) |
| Neo4j | EC2 자체 호스팅 (Phase 2 에서 Aura 검토) |
| S3 | AWS S3 |

content-api 와 같은 EKS 클러스터 사용. namespace 분리.

---

## 4. 외부 서비스 의존성

| 서비스 | 용도 | SLA 의존도 | Key 보관 |
|---|---|---|---|
| Firebase Auth | 사용자 인증 | 매우 높음 | env (FIREBASE_ADMIN_KEY) |
| Claude API | LLM (청킹, HTML, 개념 추출) | 매우 높음 | env (ANTHROPIC_API_KEY) |
| Google Cloud STT V2 | STT | 매우 높음 | service account JSON |
| Google Cloud Vision | OCR (v0.5+) | 높음 | 동일 service account |
| Google Cloud TTS Neural2 | TTS 나레이션 | 매우 높음 | 동일 service account |
| Upstage Solar Embedding | RAG 인덱싱 | 중간 (캐시 가능) | env (UPSTAGE_API_KEY) |
| HyperFrames API | 비디오 생성 | **매우 높음 (v0.1 핵심)** | env (HYPERFRAMES_API_KEY) |
| YouTube Data API v3 | 채널 업로드 | **매우 높음 (v0.1 핵심)** | 학원별 OAuth refresh_token (KMS 암호화) |
| NHN Cloud 알림톡 | 학부모 알림 | 높음 | env (NHN_API_KEY) |
| AWS SES | 이메일 | 중간 | IAM role |
| SOLAPI | SMS 폴백 | 낮음 | env (SOLAPI_KEY) |
| AWS S3 | 파일 저장 | 매우 높음 | IAM role |
| AWS KMS | refresh_token 암호화 | 매우 높음 | IAM role |

---

## 5. v1·v2 결정 폐기 내역

| 폐기 | 사유 |
|---|---|
| PostgreSQL (v1 architecture 일부 기재) | v1 내부 모순 (tech-decisions는 MySQL). MySQL 로 통일 |
| Clova Speech / Clova OCR | content-api 의 Google 패턴 재사용. 운영 경험 보존 |
| ElevenLabs | 비용 (학원 100개 시 월 3만 USD 추정), Google TTS 무료 한도로 MVP 충분 |
| PWA (Progressive Web App) | "PWA" 표현 부정확. SSR/Server Components 디버깅 부담 — 순수 SPA 로 단순화 |
| FastAPI + LangGraph (Python) | TS monorepo polyglot 부담. MVP~v1.0 까지 LangGraph 가치 받지 못함. Node 우선 검토 |
| Android 앱 MVP 포함 | 별도 프로젝트 + 16주 동기 개발 부담. MVP 는 강사 수동 업로드, v2.0 부터 Android |

---

## 6. 차후 검토 (Phase 2~)

| 항목 | 검토 시점 |
|---|---|
| LangGraph 도입 (Node 우선) | 복잡 multi-step 에이전트 필요 시 (v2.0~) |
| 자체 TTS (Kokoro/Chatterbox/MeloTTS) | 학원 50+ 도달 시 비용 ROI 평가 |
| Qdrant Cloud / Neo4j Aura | 운영 부담 ↑ 시점 |
| `@aiagent/stt`, `@aiagent/ocr` 패키지 추출 | content-api · academy-api 양쪽 안정화 후 |
| Korean OAuth (Kakao/Naver) | 학원 가입 마찰 줄이기 위해 v0.5+ 검토 |
