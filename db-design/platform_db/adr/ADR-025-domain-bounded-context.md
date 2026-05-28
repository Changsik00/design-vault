# ADR-025: domain bounded context 정책 (STT / OCR 공존)

| 항목         | 값                                                                                       |
| ------------ | ---------------------------------------------------------------------------------------- |
| **Status**   | Accepted                                                                                 |
| **Date**     | 2026-05-16                                                                               |
| **Phase**    | phase-04 (OCR) — spec-04-01                                                              |
| **Author**   | dennis + Agent                                                                           |
| **Replaces** | (없음 — 신규)                                                                            |
| **Related**  | ADR-024 (Layered Clean within NestJS), `apps/content-api/docs/ARCHITECTURE.md` §3.3, §13 |

---

## Context

phase-04 (OCR) 진입으로 `apps/content-api` 가 STT + OCR 두 feature 를 호스트하게 됨. ARCHITECTURE.md §3.3 의 `domain/` layer 가 다음 구조로 진화 중:

```
src/domain/
├── content/      ← STATUS / FAIL_REASON / Job / Readiness (cross-feature shared)
├── stt/          ← extractTranscript (STT 한정 — V2 response 파싱)
└── upload/       ← SourceType + S3 key + sanitize (cross-feature entry)
```

phase-04 진입 시점에 필요한 결정:

1. **SourceType 의 위치** — `domain/upload/` 유지 vs `domain/content/` 이동
2. **OCR 한정 domain 함수 위치** — `domain/ocr/` 신설 시점
3. **image vs pdf 의 SOURCE_TYPE 분리 여부** — DB ENUM 신규 추가 필요성
4. **shared/ 의 경계** — domain 적 개념이 shared/ 로 새는 것 방지

spec-04-01 의 작업 단위에서 위 4개 동시 결정 — 후속 spec (04-02 ~ 04-10) 의 prerequisite.

## Decision

### 1. `domain/upload/` 유지 (cross-feature entry 의 단일 위치)

upload 는 STT/OCR 양쪽 _entry point_ 의 공통 영역. SourceType / S3 key 조립 / 파일명 sanitize 가 양쪽에서 동일 호출.

- `domain/content/` 로 이동 X — content/ 는 _runtime state_ (STATUS / FAIL*REASON / Job / Readiness) 영역. upload 는 \_entry validation* 영역 → 서로 concern 다름
- `shared/` 로 이동 X — feature 의존 (AGT_SRC token 자체가 비즈니스) 이므로 §13.2 위반

### 2. `domain/ocr/` 는 04-03 (Vision adapter) 도래 시점에 신설

- 본 spec-04-01 에선 `domain/ocr/` 디렉토리 생성 X — YAGNI
- 04-03 의 `GoogleVisionClient` 구현 시 response 파싱 domain 함수 (예: `extractOcrText` / `extractOcrPages`) 가 첫 사용처 → 그때 신설
- 신설 시 `domain/stt/` 와 대칭 (각 feature 의 SDK response 파싱 함수 위치)

### 3. image vs pdf 의 SOURCE_TYPE 통합 (모두 `AGT_SRC:DOCUMENT`)

- 신규 ENUM 값 추가 X — DDL default 가 `AGT_SRC:DOCUMENT` 이므로 기존 ENUM 재사용
- 사유:
  - DB 팀 협의 비용 회피 (사용자 결정 2026-05-16)
  - OCR pipeline 입장에선 image / pdf 구분 무의미 (Google Vision 이 ext / mime 으로 분기)
  - SOURCE*TYPE 의 본 spec 영역에선 \_pipeline dispatch 신호* 만
- **향후 image vs pdf 구분 필요시**:
  - 별 SourceType 추가 (DB 팀 협의) — 권장 X (비용 ↑, 효익 ↓)
  - 기존 `CONTENT_TYPE` 컬럼 활용 (`AGT_CON:IMAGE_RAW` / `AGT_CON:PDF_RAW` 같은 sub-type) — 권장
  - 별 spec 으로 진행 — 본 ADR 영역 외

### 4. dispatch 매트릭스 함수의 위치

- `sourceTypePipeline(t: SourceType): 'STT' | 'OCR'` — `domain/upload/source-type.ts` 안에 위치
- 사유: dispatch 신호 (SourceType) 의 단일 출처 — switch / map / route 가 한 파일에 결집
- `Pipeline = 'STT' | 'OCR'` type 도 동일 파일 — 이후 feature 추가 시 (예: `'TRANSLATE'`) 단일 지점 수정

### 5. `shared/` 엄격 (재확인)

ARCHITECTURE.md §13.2 의 `shared/` 정책 본 ADR 로 강화:

- `SourceType` 같은 domain 개념은 `domain/` 만. `shared/` 거부.
- `Pipeline` type 도 domain 개념 — `domain/upload/` 에 위치 (cross-feature entry 의 일부).
- `shared/` 는 _순수 cross-cutting 기반_ (zod schema / DI token / env contract) 만.

## Consequences

### Positive

- **SourceType 단일 출처**: dispatch 결정이 한 파일에 결집 → 새 feature 추가 시 영향 분석 trivial
- **YAGNI 준수**: `domain/ocr/` 신설을 04-03 으로 미뤄 빈 디렉토리 + dead code 회피
- **DB 비용 0**: 기존 ENUM 재사용 — DB 팀 요청 없이 phase-04 진입 가능
- **확장성**: 새 feature 추가 (예: `'TRANSLATE'`) 시 `Pipeline` type 확장 + `sourceTypePipeline` exhaustive switch 가 compile error 로 영향 범위 알림

### Negative

- **image vs pdf 구분 손실**: DB 레벨에서 두 종류 동일 SOURCE_TYPE — 통계 / 운영 분석 시 ext 추적 별도 (CONTENT_TYPE 또는 ORIGINAL_FILE ext 파싱)
- **`domain/upload/` 범용성 부담**: SourceType / Pipeline 외에 future cross-feature entry 함수가 추가되며 비대화 risk → §13.2 의 _feature 의존 검사_ 가 review checklist

### Neutral

- `domain/stt/` 와 `domain/ocr/` 대칭 — STT 처럼 OCR 도 response 파싱 함수만 가짐 (작은 surface). domain 의 비대화 risk 낮음.

## Alternatives Considered

### Alt 1: SourceType 을 `domain/content/` 로 이동

- 거부 사유: `domain/content/` 의 concern 은 runtime state. SourceType 은 entry validation — concern 혼재 risk

### Alt 2: `domain/ocr/` 본 spec 에서 신설

- 거부 사유: 첫 사용처 (04-03 Vision response 파싱) 가 아직 없음. 빈 디렉토리 + placeholder file 은 YAGNI

### Alt 3: `AGT_SRC:IMAGE` 신규 ENUM 추가

- 거부 사유: 사용자 결정 (DB 요청 최소화 2026-05-16). 기존 DOCUMENT 재사용으로 충분

### Alt 4: `Pipeline` type 을 `shared/` 로

- 거부 사유: feature 의존 (STT/OCR 자체가 비즈니스 분류) → §13.2 위반

## References

- ARCHITECTURE.md §3.3 (domain/), §13 (Layer Import Rules + shared/ Strict)
- ADR-024 Layered Clean within NestJS
- spec-04-01 (본 ADR 의 도입 spec)
- HANDOVER §2.6 (SST_AGENT_CONTENTS DDL — SOURCE_TYPE default = AGT_SRC:DOCUMENT)
