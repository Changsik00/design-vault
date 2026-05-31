---
type: index
aliases:
  - WIKI
  - 위키 운영 규약
  - llm wiki
tags:
  - platform-db
  - index
  - meta
---

# WIKI — 이 저장소의 운영 규약 (LLM Wiki)

이 vault는 [Karpathy의 LLM Wiki 패턴](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)을 따른다: **사람은 소스를 큐레이팅하고 질문하고, LLM(에이전트)이 위키를 갱신·유지한다.** 위키는 질문·소스가 쌓일수록 풍부해지는 **누적 산출물**이다.

> 문서 *작성* 규약(파일명·frontmatter·wikilink·ID·다이어그램)은 [[CONVENTIONS]]. 이 문서는 위키를 *운영*하는 규약이다.

## 🧱 3계층

| 계층 | 여기서는 | 규칙 |
|---|---|---|
| **Raw Sources** (불변 원천) | 구현 리포트 이슈(GitHub #10·#19 …), 설계 논의, 외부 표준/스펙 | 읽기만. 위키가 소스를 수정하지 않는다. seed 탐색 로그는 `db-design/archive/`. |
| **Wiki** (LLM 생성·유지) | `db-design/platform_db/` 의 `core/` · `decisions/` · `explainers/` | 모든 정본 설계가 여기. 카테고리별 md. |
| **Schema** (구조·규약·워크플로) | 이 `WIKI.md` + [[CONVENTIONS]] | 에이전트가 챗봇이 아니라 *규율 있는 유지자*로 동작하게 하는 설정. |

## 🧭 네비게이션 파일 (루트)

- **[index](./index.md)** — 카테고리별 카탈로그(core·decisions·explainers·entities). 무엇이 어디 있는지의 **단일 정본**. 디렉토리마다 README를 두지 않는다(중복·드리프트 방지).
- **[log](./log.md)** — append-only 시간순 기록(Ingest/Query/Lint). 새 항목은 **맨 위에** 추가, 기존 항목 수정 금지.
- **[[glossary]]** — 약어·용어 사전(A–Z 빠른 찾기 + 카테고리별, 정본 문서 링크). 새 용어 정의 시 여기에 등재.
- **[README](./README.md)** — 사람용 소개·강의·배포 랜딩(GitHub 진입점).

## 🔄 3 연산

### 1. Ingest — 새 소스를 위키에 녹인다
구현 리포트·설계 논의·외부 표준이 들어오면:
1. 소스를 읽고 **영향받는 페이지를 한 번에** 갱신(core 불변식·decisions·관련 explainer·schema DDL). 한 곳만 고치지 말 것.
2. **교차참조**(wikilink)를 양방향으로 잇는다.
3. **모순을 플래그**한다 — 설계 vs 구현이 어긋나면(예: 이슈 #19의 entitlement 키) 어느 쪽을 채택할지 결정하고 *반대편 서술을 정정*한다(드리프트 금지).
4. **코드 vs 설계 경계**를 지킨다 — 이 vault는 설계만. 코드 픽스는 monorepo 트랙으로 명시 분리.
5. `index`에 새 페이지 등재, `log`에 한 줄 기록, 원 이슈에 반영 요약 코멘트.

### 2. Query — 질문에 답하고, 답을 환원한다
질문이 오면 관련 페이지를 찾아 답한다. 그 답이 **재사용 가치**가 있으면(개념 설명·결정 근거) **새 explainer/섹션으로 환원**해 위키를 키운다.
- 예: "FOR UPDATE/조건부 UPDATE 차이?" 질문 → [[concurrency-control]] explainer 신설.

### 3. Lint — 주기적 건강검진
- `python3 /tmp/verify_links.py` → 미해결·모호 wikilink 0 (코드펜스·인라인코드 무시).
- **상태 표기 일관성**: ✅/🟡/🔴/⛔/❓ 의미가 [[CONVENTIONS]]와 일치.
- **불변식 번호 드리프트**: 번호는 고정 핸들(삭제 시 빈틈 유지). 외부(monorepo)는 번호 대신 텍스트로 참조.
- **고아 페이지**: `index`에 없는 문서, **모순**: 설계 vs 구현, **stale**: 더 이상 맞지 않는 주장.
- **엔진 일관성**: 콜아웃 밖 MySQL 마커(InnoDB·AUTO_INCREMENT·ON DUPLICATE …) 0.

## ✅ 유지 원칙

- **DRY 네비게이션**: 인덱스는 `index.md` 하나. 디렉토리별 README 금지(`decisions/`도 index에 흡수). 사람용 랜딩만 `README.md`.
- **정본은 한 곳**: 한 사실은 한 페이지에. 다른 곳은 wikilink로 가리킨다.
- **추가 시 3종 갱신**: 새 문서 → ① 해당 디렉토리 ② `index` 등재 ③ `log` 기록.
- 산출물은 한국어, PostgreSQL 1순위, 커밋 `docs(...)` 소문자·One Task=One Commit.
