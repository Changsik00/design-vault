---
type: index
aliases:
  - log
  - 변경 이력
tags:
  - platform-db
  - index
  - meta
---

# log — 위키 변경 이력 (append-only)

Ingest/Query/Lint/Maint 기록. **새 항목은 맨 위에 추가**하고, 기존 항목은 수정하지 않는다. 운영 규약은 [WIKI](./WIKI.md).

접두사: `[INGEST]` 소스 반영 · `[QUERY]` 질문→페이지 환원 · `[LINT]` 건강검진 · `[MAINT]` 구조/정리.

---

- **2026-05-31 [MAINT]** LLM Wiki 전환 — 루트 `index.md`(카탈로그)·`WIKI.md`(운영 규약)·`log.md` 신설. 인덱스성 README 4개(db-design·platform_db·explainers·decisions) → `index.md`로 통합·제거. README는 배포 랜딩만 유지. *(PR #22)*
- **2026-05-31 [MAINT]** explainer 토픽 README 6개 제거 — 마스터 인덱스로 일원화(중복·드리프트 제거), CONVENTIONS 규약 정정. *(PR #21)*
- **2026-05-31 [INGEST]** 이슈 #19(Billing/Admin 구현 리포트) → `org_entitlement` UNIQUE `(org_pk, product_code)` → `(org_pk, service)` 채택(entitlement=service당 병합 투영), 불변식 #12를 "FOR UPDATE 또는 조건부 UPDATE"로 확장, billing explainer에 검증 테스트 패턴 반영. **[QUERY→PAGE]** 동시성 제어 explainer 신설(race·락·CAS·SKIP LOCKED, Drizzle 오해 정정). *(PR #20, 이슈 #19 closed)*
- **2026-05-31 [INGEST]** 이슈 #10(구현 audit 갭) → 불변식 #12~#15 신설(FOR UPDATE·outbox 의무·N:M 키·silent-empty 금지), outbox 발행 의무 레지스트리, 운영자 override 액션 계약. 코드 픽스는 monorepo 트랙으로 분리. *(PR #18, 이슈 #10 closed)*
- **2026-05-31 [MAINT]** 요구사항·결정 ID 규약 명문화(번호=고정 참조 핸들, 빈틈 유지) + requirements 표 정규화. *(PR #17)*
- **2026-05-31 [MAINT]** §5 ISO 27001 Annex A 통제 매핑 추가 + 로드맵 P0/P1/P2 블록화. *(PR #16)*
- **2026-05-31 [MAINT]** 토폴로지·3-gate·billing 흐름 ASCII → mermaid, `@aiagent/db-platform`→`@db-platform` 리네임, `.obsidian` UI 상태 gitignore. *(PR #15)*
- **2026-05-31 [MAINT]** 전체 리밸런싱 — 난이도 표기 초/중/고, "사수" 문구 제거, CONVENTIONS 신설 *(PR #11)* → **PostgreSQL 1순위 전환**(전제·DDL·RLS 반전·ENUM 근거·테스트 스택) *(PR #12)* → Obsidian 사용성·mermaid 13·테스트절 21 *(PR #13)* → 루트 README PG 정정 *(PR #14)*.
- **(초기 구축) [QUERY→PAGE]** 공부 질문 → explainer 다수 신설(BOLA·RLS·일관성·머신신원·테스트 전략 등) + 주제 분류·배포용 README. *(PR #6, #8)*
- **(초기 구축) [MAINT]** BDD 커버리지 확장·요구 추적 매트릭스 *(#3)*, black-box E2E 여정 *(#4)*, 예외·생명주기 시나리오 *(#5)*, 스키마 재검수 P0/P1/P2 *(#2)*, platform-db 중복 구조 통합·wikilink 정합 *(#1)*.
