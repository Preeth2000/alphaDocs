---
page: Code-Review-2026-06-13
tags:
  - platform
  - review
last-reviewed: 2026-06-13
---

# Platform Code Review — 2026-06-13

Full-codebase review of all five implemented services. Every finding is filed as a bead in the owning service's tracker (`bd list` in each repo). This page is the index + cross-service themes.

## Bead counts

| Service | Beads filed | P0 |
|---|---|---|
| alphaKey | 28 | 3 — refresh token never returned; KEK rotation no-op; fail_closed query-param bypass |
| alphaGen | 23 | 4 — normalization leakage; backtest same-bar lookahead; manifest/fundamentals contract; unauthenticated API |
| alphaTrade | 21 | 4 — SELL sized by size_pct not position; OCO legs orphaned on signal exit; sync broker calls block event loop; API auth not wired |
| alphaLink | 7 | 1 — cookie-presence-only auth on BFF routes |
| alphaFrame | 11 | 1 — nginx exposes /auth/internal (decrypted secrets) publicly |

## Cross-service themes

1. **Auth chain is broken end-to-end.** alphaKey never returns the refresh token → BFF stores empty cookie → sessions die at access TTL. Meanwhile alphaGen has no auth at all, alphaTrade's JWT dependency exists but isn't wired, and alphaLink's middleware checks only cookie presence. Fixing any one service is insufficient — treat as one epic: alphaKey-7cb → alphaGen-wc7 → alphaTrade auth P0 → alphaGen-ui-a40.
2. **Train/live skew.** Cumulative VWAP differs between alphaGen fetch and alphaTrade warmup window (alphagen-317); manifest feature list omits fundamentals (alphagen-vxw); backtest fills at signal-bar close (alphagen-tlw). Gate metrics currently overstate live performance.
3. **Secrets posture.** Vault exists and is sound at rest (envelope encryption), but: KEK rotation is broken (alphaKey-5fv), nginx leaks the internal plaintext endpoint (alphaFrame-9m3), broker keys also live plaintext in alphaTrade DB (db mode), and the UI writes Polygon keys into alphaGen's .env (alphaGen-ui-q2c).
4. **Async hygiene.** Sync DB/httpx/Redis calls inside async handlers in alphaKey, alphaGen SSE, alphaTrade OCO monitor, alphaLink daily-close path. Event-loop stalls are the platform's main latency risk.
5. **Single-writer assumptions.** MinIO versioning, retrain budget file, mark_stale_runs, first-user-developer all race under concurrency.

## Deliverables from this review

- [[services/alphaTest/Regression-Scenarios|Regression scenario catalogue]] (≈60 scenarios, prioritized; L1/L2/L12/L16 + A1/V2 first — all currently failing).
- [[services/alphaPerf/Performance-Test-Plan|Performance test plan]] (budgets, k6/pytest-benchmark tooling, review-identified hot-spot list).
- Feature gaps filed as beads: MFA, password reset, login rate limiting, session management (alphaKey); drift monitoring + scheduled retraining, data-quality validation (alphaGen); consensus confidence threshold (alphaTrade); alphaKey CI pipeline; DB backups, resource limits, alert delivery (alphaFrame).
