---
type: index
aliases:
  - billing 인덱스
tags:
  - platform-db
  - index
---

# 💳 billing — 결제·구독

결제 원장→권한 투영, 멱등·outbox·일관성을 다룹니다.

| 난이도 | 문서 | 한 줄 |
|---|---|---|
| 🟢 초 | [[feature-limits]] | feature_limits 3중 정의 우선순위 |
| 🟢 초 | [[subscription-lifecycle]] | 구독 상태 머신 (TRIALING→ACTIVE→…) |
| 🟡 중 | [[idempotency-key]] | 멱등성 키 (payment_ledger) |
| 🟡 중 | [[outbox-pattern]] | Outbox 패턴 |
| 🟡 중 | [[usage-metering]] | usage_snapshot 사용량 계량 |
| 🟡 중 | [[webhook-processing]] | PG(결제대행사) 웹훅 수신·처리 |
| 🔴 고 | [[consistency-model]] | 강한 일관성 vs 결과적 일관성 |

> 전체 주제 인덱스: [explainers 마스터 인덱스](../README.md) · 작성 규약: [CONVENTIONS](../../CONVENTIONS.md)
