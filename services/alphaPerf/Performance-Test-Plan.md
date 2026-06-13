---
service: alphaPerf
page: Performance-Test-Plan
status: draft
last-reviewed: 2026-06-13
tags:
  - service/alphaPerf
  - testing
  - performance
---

# Performance Test Plan

> Performance scenarios and budgets for alphaPerf, derived from the 2026-06-13 code review.
> Known hot spots found in review are listed per service — these are where load tests should bite first.

[[alphaPerf]] · [[services/alphaTest/Regression-Scenarios|Regression Scenarios]] · [[platform/Overview]]

---

## Budgets (initial proposals — tune after first baseline run)

| Path | Budget | Rationale |
|---|---|---|
| Signal→order submission latency (alphaTrade, mock broker) | p99 < 2s from bar close | stale-drop window is interval×0.5; 1m bars leave 30s |
| Tick wall time, 25 models / 10 tickers | < 15s (1m interval headroom) | inference + fetch + gates serial today |
| Event-loop blocking (alphaTrade) | max single stall < 250ms | OCO polls / sync calls must not freeze ticks |
| alphaKey token verify (downstream services, cached JWKS) | p99 < 10ms local | runs on every API call |
| alphaKey login (argon2) | p95 < 500ms @ 20 rps | argon2 cost vs UX |
| BFF proxy overhead (alphaLink) | p95 < 30ms added | thin proxy |
| Training run (golden 1d config, CPU) | within ±20% of baseline | regression fence, not absolute |
| SSE fan-out | 200 concurrent streams, < 5% missed events, stable RSS | threadpool exhaustion risk found in review |
| MinIO publish (3 artifacts ~10MB) | p95 < 5s | publish path |

## Tooling

- **k6** (or Locust) for HTTP/SSE load against nginx.
- **pytest-benchmark** for in-process micro-benchmarks (feature pipeline, labelers, window building) — alphaGen already has `tests/perf/test_benchmarks.py`; extend and export to Prometheus pushgateway for trend tracking.
- Mock T212 server with injectable latency/429 profiles (shared with alphaTest).
- Grafana dashboard "alphaPerf baselines" fed from CI runs; fail CI on >20% regression vs rolling baseline.

---

## Scenario Matrix

### P1 — Trading hot path (alphaTrade)
1. **Tick scaling**: 1→100 models across 1→25 tickers, mock data provider with fixed latency. Measure tick wall time, event-loop max stall, per-model inference latency histogram. *Known issues to expose:* per-tick `registry.refresh` + BotSettings reload (alphaTrade per-tick-overhead bead), `EquityRepo.today_open()` full-table scan (degrades as equity rows grow — load 1M rows first), sync OCO polling stalls (monitor_oco P0).
2. **Order burst**: 50 signals in one tick → queue drain rate under throttle config; verify priority eviction behaves and no order exceeds stale window.
3. **OCO monitor scaling**: 100 open positions with active OCO → broker poll rate vs throttle, connection reuse (T212Client per-request connections bead), event-loop health.
4. **Long-run soak**: 48h simulated ticks at 1m cadence → memory growth (positions, _seen sets, OCO tasks), SQLite/Postgres table growth (equity_curve!), log volume.

### P2 — Auth path (alphaKey)
1. **Steady-state verify**: 500 rps mixed GET across alphaGen/alphaTrade with JWT auth on → JWKS caching effectiveness, denylist Redis round-trip (per-call connection bead), DB token_version lookup cost.
2. **Login storm**: 20 rps logins (argon2 CPU); find the knee; verify event loop not starved (sync-in-async bead).
3. **Vault read fan-out**: alphaTrade+alphaGen secrets refresh under tick load; N+1 audit-commit cost.

### P3 — Training throughput (alphaGen)
1. **Feature pipeline micro-bench**: 1M-row intraday frame through indicators → windows. *Known:* `make_windows` np.stack copy (memory bead), triple-barrier O(n·h) loop (bead) — benchmark before/after fixes.
2. **Worker throughput**: N queued runs, concurrency=1 vs 2 — wall time, peak RSS (no max-tasks-per-child bead), Redis log pub rate.
3. **SSE log streaming**: 100 clients on one running job; threadpool usage (sync pubsub bead), missed-line rate.
4. **Publish/MinIO**: parallel publishes, version-race behavior under contention.

### P4 — UI/BFF (alphaLink)
1. **Dashboard load**: 50 concurrent users on trade dashboard (REST + 2 SSE streams each) → BFF overhead, upstream connection counts (SSE leak bead).
2. **Next.js route handlers**: jobs/runs proxy at 100 rps.

### P5 — Infra (alphaFrame)
1. **Contention test**: training run at full CPU while tick load runs → alphaTrade latency budget still met? (resource-limits bead — expected to fail until limits added).
2. **Observability overhead**: OTel on vs off → request latency delta < 5%.
3. **nginx**: rate-limit zone behavior at burst; SSE through proxy for 1h (buffering bead).

---

## Review-Identified Hot Spots (fix-then-measure list)

| Service | Issue (bead) | Expected gain |
|---|---|---|
| alphaTrade | today_open full scan | O(1) day-open lookup; flat tick time as history grows |
| alphaTrade | monitor_oco sync calls in loop | stall p99 from seconds → <50ms |
| alphaTrade | per-tick registry/settings reload | tick setup cost ~constant |
| alphaTrade | httpx connection per request | broker call p50 −TLS handshake (~100ms+) |
| alphaKey | sync DB/httpx in async handlers | throughput ceiling lifted; no loop stalls |
| alphaKey | Redis connection per denylist call | verify path −1 conn setup per request |
| alphaGen | make_windows copy + triple-barrier loop | large-config train prep minutes → seconds |
| alphaGen | sync pubsub SSE threads | supports >40 concurrent log streams |
| alphaLink | SSE upstream leak | stable connection count over churn |
