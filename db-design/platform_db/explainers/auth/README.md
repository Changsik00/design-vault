---
type: index
aliases:
  - auth 인덱스
tags:
  - platform-db
  - index
---

# 🔐 auth — 인증·인가

3-gate(소속·결제·정책)와 역할·권한 모델을 다룹니다.

| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[gate-abc-flow]] | Gate A/B/C 전체 흐름 |
| 🟢 초 | [[gate-b-entitlement]] | Gate B와 엔타이틀먼트 개념 |
| 🟡 중 | [[bola-object-authz]] | "로그인은 됐는데 남의 데이터가 보이는" 취약점 |
| 🟡 중 | [[casl-ability]] | RBAC·ABAC·ReBAC을 하나의 can()으로 (Gate C 내부) |
| 🟡 중 | [[fail-closed]] | 의심스러우면 막는다 |
| 🟡 중 | [[explainers/auth/gate-b-billing-grace|gate-b-billing-grace]] | Gate B 유예 기간 설계 (status + validUntil) |
| 🟡 중 | [[role-capability]] | role 2단 분리와 capability |
| 🔴 고 | [[machine-identity-apikey]] | 사람만 사용자가 아니다 (api_key) |
| 🔴 고 | [[perm-version-propagation]] | perm_version 전파 |
| 🔴 고 | [[rebac-delegation]] | 관계 기반 권한 위임 |

> 전체 주제 인덱스: [explainers 마스터 인덱스](../README.md) · 작성 규약: [CONVENTIONS](../../CONVENTIONS.md)
