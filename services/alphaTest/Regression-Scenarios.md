---
service: alphaTest
page: Regression-Scenarios
status: draft
last-reviewed: 2026-06-13
tags:
  - service/alphaTest
  - testing
  - regression
---

# Regression Test Scenarios

> Master scenario catalogue for the alphaTest suite. Derived from the 2026-06-13 platform-wide code review.
> Each scenario lists the bug class it guards against; bead IDs reference the originating issue in each service's tracker.
> Conventions: **[E2E]** full stack via docker compose, **[INT]** two services, **[CON]** contract test, **[UNIT-R]** unit-level regression pinned to a fixed bug.

[[alphaTest]] · [[services/alphaPerf/Performance-Test-Plan|Performance Test Plan]] · [[platform/Overview]]

---

## 1. Authentication & Session Lifecycle (alphaKey + alphaLink BFF)

| # | Scenario | Asserts | Guards |
|---|---|---|---|
| A1 | **[E2E] Login → use → refresh → use → logout** | Login returns `access_token` AND `refresh_token`; BFF sets non-empty httpOnly cookie; refresh after access expiry yields a working new token; logout revokes both instantly. | alphaKey-7cb, alphaGen-ui-3pc |
| A2 | **[INT] Refresh-token rotation + reuse detection** | Old refresh token rejected after rotation; presenting a rotated token revokes ALL sessions for the user (chain compromise). | alphaKey-cqy |
| A3 | **[INT] Instant logout via denylist** | After logout, the still-unexpired access token is rejected by alphaKey, alphaGen and alphaTrade within 1s. With Redis stopped, logout returns an error — never a silent success. | alphaKey-... (denylist add swallow) |
| A4 | **[CON] fail_closed not client-controllable** | Requests with `?fail_closed=false` (and header/body variants) do NOT alter denylist behavior. | alphaKey-9qg |
| A5 | **[INT] Role demotion takes effect immediately** | After admin demotes developer→standard, the user's existing token cannot call developer endpoints (token_version bump or role re-check). | alphaKey-8mo |
| A6 | **[INT] Kill switch / logout-all** | `POST /auth/admin/users/{id}/kill` invalidates all outstanding access+refresh tokens; disabled users can't log in or refresh; re-enable path exists. | alphaKey-0yc |
| A7 | **[UNIT-R] Email case normalization** | `Foo@x.com` and `foo@x.com` are the same account at register and login. | alphaKey duplicate-email bead |
| A8 | **[UNIT-R] Login timing parity** | Response time for unknown email ≈ known email + wrong password (dummy-hash verification). | timing-oracle bead |
| A9 | **[INT] Signing-key rotation** | After `rotate-keys`, old-key tokens verify until `retire_after`, then fail; JWKS stops publishing retired keys without manual intervention; new logins use the new kid. | alphaKey-t8l |
| A10 | **[E2E] Brute-force lockout** | N failed logins → throttled/locked with audit events (once rate limiting lands). | alphaKey-xhx |
| A11 | **[CON] Unauthenticated logout cannot denylist arbitrary jti** | Logout derives jti/exp from the presented token, not the body; junk body values have no effect. | alphaKey-2v2 |

## 2. Vault & Secrets (alphaKey + consumers)

| # | Scenario | Asserts | Guards |
|---|---|---|---|
| V1 | **[INT] Vault round-trip** | PUT secret → masked list shows last4 only → service endpoint returns plaintext with valid X-Service-Token → audit row per read. | — |
| V2 | **[E2E] Master-key (KEK) rotation** | With seeded credentials: rotate KEK → `rotate-master-key` → ALL secrets and signing keys still decrypt; rows actually re-wrapped (key_version changes). **Currently expected to fail.** | alphaKey-5fv |
| V3 | **[CON] /auth/internal not reachable through nginx** | `https://<host>/auth/internal/secrets/<uid>` returns 403/404 from the edge, even with a valid service token. | alphaFrame-9m3 |
| V4 | **[INT] Service-token auth** | Wrong/absent X-Service-Token → 401; access is logged with principal name. | — |
| V5 | **[INT] alphaTrade & alphaGen consume vault keys** | With SECRETS_SOURCE=alphakey, T212/Polygon keys flow from vault; key update in vault hot-reloads the running bot's clients on next tick. | apply_bot_settings path |

## 3. Training Pipeline Integrity (alphaGen)

| # | Scenario | Asserts | Guards |
|---|---|---|---|
| T1 | **[UNIT-R] No normalization leakage** | NormStats persisted in the manifest are computed from the training fold only; recomputing stats on val data and diffing must show a difference (proves split-fitting). | alphagen-bl6 |
| T2 | **[UNIT-R] Backtest execution timing** | Backtest fills occur on bar t+1 relative to the signal bar; golden test with a synthetic series where same-bar fills would inflate return. | alphagen-tlw |
| T3 | **[CON] Manifest = inference contract** | For a run with fundamentals + VWAP enabled: `len(feature_names) == n_features == ONNX input width`; alphaTrade adapter builds input from manifest without shape errors. | alphagen-vxw |
| T4 | **[UNIT-R] Embargo ≥ label horizon** | Config with `horizon_bars > embargo_bars` is rejected at validation. | alphagen-usk |
| T5 | **[UNIT-R] Determinism** | Same config + seed + cached data → identical labels, splits, and (CPU) near-identical metrics across two runs. | sweep seed bead |
| T6 | **[UNIT-R] Earnings features present** | With fundamentals enabled and stub yfinance data (naive AND tz-aware indices), eps columns appear in the feature frame. | alphagen-7pa |
| T7 | **[UNIT-R] VWAP train/live parity** | Same OHLCV slice through alphaGen fetch-VWAP and alphaTrade `_precompute_passthrough_features` yields identical VWAP values. | alphagen-317 |
| T8 | **[INT] Validation gate paths** | Gate-fail blocks registration + cleans checkpoints; force-save registers with `staging` alias only after explicit endpoint call; gate-pass publishes `model.ready`. | alphagen-2rs |
| T9 | **[E2E] Train → publish → consume** | POST /runs → SSE log stream completes with `done` → artifacts in MinIO under correct user/account prefix → MLflow version aliased → alphaTrade model_sync downloads, parity-checks and loads it. | alphagen-2yl chain |
| T10 | **[INT] Concurrent publish versioning** | Two simultaneous publishes of the same run_name get distinct versions; `latest` pointer is consistent. | alphagen-2yl |
| T11 | **[INT] Run lifecycle edge cases** | Cancel during queued and during running; worker restart marks only orphaned runs failed; SSE log replay after completion returns full file + terminal status. | alphagen-9rp, 7s8 |
| T12 | **[CON] Ownership enforcement** | User B cannot GET/cancel/publish user A's run once auth lands; unauthenticated requests rejected. | alphagen-wc7, alphaGen-ui-mk8 |

## 4. Live Trading Engine (alphaTrade)

Highest-value suite — every scenario simulates the broker (mock T212 HTTP server).

| # | Scenario | Asserts | Guards |
|---|---|---|---|
| L1 | **[INT] SELL closes the actual position** | Open 10 shares; signal SELL with size_pct that would compute 3 (or 17) shares → market order quantity == 10; journal quantity == 10; no short ever created. | alphaTrade SELL-sizing P0 |
| L2 | **[INT] OCO cancelled on signal exit** | BUY fill places SL+TP; later SELL signal → both legs cancelled at broker before/with market close; monitor task ends; no orphaned orders remain. | OCO-orphan P0 |
| L3 | **[INT] OCO leg fill** | SL (or TP) fills → other leg cancelled, position closed with cooldown, journal exit_reason OCO_SL/OCO_TP, retirement stats recorded. | — |
| L4 | **[INT] Partial OCO placement** | Stop placed, limit placement fails → defined recovery (retry/cancel survivor/monitor single leg) — never a silent one-sided state. | partial-OCO bead |
| L5 | **[INT] Pyramiding accumulation** | With pyramiding allowed, second BUY adds quantity and recomputes weighted avg_entry (not overwrite). | BUY-overwrite bead |
| L6 | **[UNIT-R] max_positions ignores cooldown rows** | Positions with quantity=0 (cooldown placeholders) don't count toward max_positions. | gate-count bead |
| L7 | **[INT] Daily loss halt** | Equity drop past threshold → new orders blocked, single edge-triggered alert; recovers next day; halt never blocks SELL/exits if that's the design (pin the decision). | halt semantics |
| L8 | **[INT] Kill switch** | Engaged → signals logged, zero orders; resume works; bot starts halted by default. | — |
| L9 | **[INT] Reconciliation on restart** | Local DB positions diverge from broker → startup reconcile converges (removes stale, adopts unknown); OCO monitors re-attached with the ORIGINAL sl/tp and model_id. | OCO re-attach bead |
| L10 | **[INT] Order idempotency & dedup** | Same (model, ticker, bar, side) never produces two broker orders across: duplicate ticks, queue re-enqueue, process restart mid-queue. | cid + _seen logic |
| L11 | **[INT] Stale order drop** | Order older than stale window when dequeued → dropped, metric incremented, no broker call. | — |
| L12 | **[INT] Broker resilience** | 429 with Retry-After honored; 5xx retried with backoff; 401 not retried; event loop stays responsive during a slow OCO poll (tick jitter < 1s while broker latency 30s). | monitor_oco blocking P0 |
| L13 | **[UNIT-R] Equity fallback** | Broker summary missing totalValue with open positions → tick skipped (no trading on cash-only equity). | equity-fallback bead |
| L14 | **[INT] Consensus attribution** | Two models, same ticker, conflicting overrides → documented attribution rule applied (orders use defined run_name/size); both models' logits journaled. | attribution bead |
| L15 | **[INT] Model retirement** | Losing streak crosses retirement thresholds → model excluded from trading, alert sent, can be un-retired via API. | — |
| L16 | **[CON] API auth enforced** | With auth configured, every /api/v1 route 401s without credentials — including kill_switch, settings, orders. With env key UNSET, service refuses to start in production mode (or all routes deny). | alphaTrade auth P0 |
| L17 | **[INT] Settings hot-reload** | Changing account/keys/provider in UI takes effect next tick without restart; secrets never echoed back in GET. | apply_bot_settings |
| L18 | **[INT] Bar scheduling** | 1d tick fires after NYSE close (not UTC midnight) incl. half-days; intraday ticks suppressed when market closed unless extended_hours. | calendar fallback bead |

## 5. Model Distribution (alphaGen → MinIO/MLflow → alphaTrade)

| # | Scenario | Asserts | Guards |
|---|---|---|---|
| D1 | **[INT] model_sync promote** | New Production alias in MLflow → daemon downloads, validates manifest+onnx parity, atomically swaps models_dir entry, registry refreshes, no tick uses a half-written model. | — |
| D2 | **[INT] model_sync poison model** | Corrupt onnx / missing manifest → permanent-fail record, deployment marked failed, daemon doesn't retry-loop, alphaGen retrain manager sees the failure category. | — |
| D3 | **[CON] model.ready schema** | Single versioned payload shape from both publishers; consumers tolerate unknown fields. | alphagen-2x3 |
| D4 | **[INT] Visibility & adoption** | Private model invisible to other users in UI and registry list; public model adoptable; adopted model trades under adopter's scoping rules. | multi-tenant chain |

## 6. UI / BFF (alphaLink)

| # | Scenario | Asserts | Guards |
|---|---|---|---|
| U1 | **[E2E] Forged session cookie rejected** | Hand-set `alphakey_session=1` without valid JWT → BFF API routes (jobs, fs, trade, polygon-key) return 401, pages redirect to login. | alphaGen-ui-a40 |
| U2 | **[E2E] Session expiry UX** | Access token expires mid-session → silent refresh retries once → on refresh failure user lands on /login?reason=session_expired with return path. | trade-fetch logic |
| U3 | **[CON] /api/fs confined** | Path traversal attempts (`/etc`, `~/..`, absolute) outside the allowlisted root → 400. | alphaGen-ui-a16 |
| U4 | **[E2E] Train wizard → run detail** | Config wizard YAML matches schema golden file; submit → job appears; SSE log renders; failure surfaces gate reason; force-save flow works. | config-to-yaml tests exist — extend e2e |
| U5 | **[E2E] SSE proxy hygiene** | Closing the browser tab aborts the upstream SSE connection within seconds (no connection leak across 100 open/close cycles). | alphaGen-ui-9nz |
| U6 | **[E2E] Trade dashboard live flow** | signal_fired / order_enqueued / tick_complete events render; kill-switch toggle round-trips. | — |

## 7. Infrastructure (alphaFrame)

| # | Scenario | Asserts | Guards |
|---|---|---|---|
| I1 | **[E2E] Cold start ordering** | `make up` from nothing → all healthchecks green; services tolerate slow Postgres/Redis (no crash-loop before healthy). | — |
| I2 | **[E2E] Restart resilience** | `docker kill` each service in turn → it returns (restart policy) and the platform re-converges; Redis restart does not lose denylist/queue (persistence). | alphaFrame-gmq, w31 |
| I3 | **[CON] Network exposure audit** | From outside the compose network, only nginx 80/443 (+ UI if kept) answer; scripted nmap-style check in CI. | alphaFrame-w76 |
| I4 | **[INT] Alert delivery** | Stop a service → ServiceDown fires → receiver (Slack stub) gets the alert within target SLA. | alphaFrame-8pr |
| I5 | **[E2E] Backup/restore drill** | pg_dump + restore into a fresh stack → users, vault secrets (decryptable), trade journal intact. | alphaFrame-8kh |

---

## Baseline / Golden Artifacts Needed

- Frozen OHLCV parquet fixtures (1d + 5m, one ticker each) committed to alphaTest.
- One golden manifest + ONNX model per arch (mlp, lstm) with known logits for fixed input.
- Mock T212 server (record/replay of order, fill, position, 429 flows) — single source of truth for L-suite.
- Schema snapshots: manifest.json, model.ready event, alphaKey token claims, BFF auth responses (for CON tests).

## Priority Order for Implementation

1. **L1, L2, L12, L16** — money-losing bugs (all currently failing).
2. **A1, V2** — session + key rotation (currently failing).
3. **T1–T3, T7** — silent model-quality corruption.
4. **U1, V3, I3** — exposure scenarios.
5. Remainder.
