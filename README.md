# 멀티테넌트 SaaS 플랫폼 DB 설계 노트

여러 서비스(academy · agent · market · store …)가 **하나의 공유 코어 DB**를 쓰되, 테넌트는 안전하게 격리한다 — 그 한 줄을 실제 스키마와 코드 계약까지 풀어낸 설계 자료입니다.

요구수렴부터 아키텍처 · 스키마 · 의사결정 · 검증 · **교육 문서**까지 한 흐름으로 정리해서, 강의 · 온보딩 · 컨설팅 레퍼런스로 바로 쓸 수 있게 했어요.

> 모든 내용은 👉 [db-design/platform_db/](./db-design/platform_db/) 안에 있습니다.

---

## 🧭 무엇을 다루나

- **멀티테넌시 격리** — Pool 모델, `org_pk` 행 격리, BOLA 방어
- **3-gate 인가** — 소속 · 결제 · 정책 / RBAC · ABAC · ReBAC
- **결제 ↔ 권한 원자성** — 단일 트랜잭션 + outbox, 강한 vs 결과적 일관성
- **머신 신원** — `api_key`(Firebase JWT 없이), 사람=머신 같은 게이트
- **비밀·암호화** — KMS, bcrypt, 최소권한 GRANT
- **감사·동의** — append-only, PIPA, 보존·파기
- **운영** — Permission Debugger, 관측성/SLO, 사용량
- **검증** — BDD(화이트박스) + e2e(블랙박스) + 개념별 테스트 방법

## 🙋 이런 분께

- 멀티테넌트 SaaS 백엔드·DB를 설계하는 사람
- 플랫폼 팀에 막 합류해 "왜 이렇게 짰는지" 익혀야 하는 분
- 인가·결제·컴플라이언스를 **개념부터** 잡고 싶은 분
- 리뷰·컨설팅에 쓸 레퍼런스가 필요한 팀

## 📦 구성

| 폴더 | 내용 |
|---|---|
| [core/architecture.md](./db-design/platform_db/core/architecture.md) | 아키텍처 & 규율 — **여기서 시작** |
| [core/requirements.md](./db-design/platform_db/core/requirements.md) | 요구사항 추적 + 매트릭스 |
| [core/schema-reference.md](./db-design/platform_db/core/schema-reference.md) | 전체 DDL · ERD · 3-gate · billing |
| [core/operability.md](./db-design/platform_db/core/operability.md) | 운영 가능성(O1~O6) |
| [core/bdd-scenarios.md](./db-design/platform_db/core/bdd-scenarios.md) · [e2e-journeys.md](./db-design/platform_db/core/e2e-journeys.md) | 행위 검증(화이트박스 · 블랙박스) |
| [decisions/](./db-design/platform_db/decisions/) | 의사결정 13편 — "왜 이렇게 했나"(비교+결정) |
| [explainers/](./db-design/platform_db/explainers/) | **학습 문서 35편** — Q&A + 용어 + 테스트, 난이도별 |

## 📚 학습 경로

상세 커리큘럼은 [explainers 학습 인덱스](./db-design/platform_db/explainers/README.md)(주제별·난이도순)를 따라가면 됩니다.

1. **큰 그림** — `architecture` → `requirements`
2. **🟢 기초** — multitenancy-rls · gate-abc-flow · pk-ulid-strategy · delete-patterns
3. **🟡 중급** — role-capability · casl-ability · bola-object-authz · fail-closed · webhook-processing
4. **🔴 고급** — perm-version-propagation · rebac-delegation · consistency-model · api_key · audit-hash-chain
5. **검증** — bdd-scenarios → e2e-journeys

## 🛠️ 활용

- **강의** — explainers를 난이도순으로. 각 문서에 *용어 정리*와 *테스트 방법*(vitest/supertest 예제)이 있어 실습으로 이어집니다.
- **온보딩** — `architecture` → 담당 영역 explainer → 코드.
- **컨설팅** — `decisions/`는 그대로 의사결정 템플릿, `schema-reference`는 레퍼런스 스키마.

## ⚠️ 알아두기

- **이 저장소는 설계 문서(Markdown)입니다 — 실행 코드는 없습니다.** 코드 블록은 *설계 의도·테스트 방법*을 보여주는 예시예요.
- 그래서 상태 표기(✅·🟡·🔴)는 *구현 여부*가 아니라 **설계 성숙도**입니다 — 스키마 DDL·결정이 있으면 "✅ 설계 확정", 더 다듬을 게 있으면 🟡, 아직 설계 안 된 건 🔴.
- 설계가 더 필요한 곳(🔴)은 대부분 **운영(operability)** 영역입니다.
- **MySQL 8 기준**(회사 표준). PostgreSQL이 나은 지점은 이유를 적어뒀습니다.
- 학원·인물은 설명용 **가상 예시**입니다.

## 라이선스

배포 조건은 추후 정의(*TBD*). 외부 공유 전 소유자에게 확인해 주세요.

---

<sub>설계 수렴 기록에서 출발한 저장소입니다. 결정 전 탐색 로그(원본 seed)는 [archive/](./db-design/archive/)에 보존돼 있어요.</sub>
