---
page: Cross-Service-Backlog
tags:
  - platform
  - backlog
  - cross-service
last-reviewed: 2026-06-16
---

# Cross-Service Backlog

[[README]] · [[platform/Overview]] · [[platform/Key-Decisions]]

> [!abstract] What this page is
> The authoritative list of **open** cross-service work — obligations created in one service
> that have not yet been implemented in another. Closed items are documented in the owning
> service's own docs (Architecture/API/Config/Interactions) and are not duplicated here.
> Lives in [[ToDo]] — once an item below is done and documented in its proper service doc,
> remove it from this page.

---

## Outstanding feature gaps from the 2026-06-13 code review

> [!todo] Not started — no owning-service doc covers these yet
> From the 2026-06-13 platform code review (source doc since deleted — fully superseded by this
> page plus the owning services' own docs). Re-check the owning service's docs before starting —
> if work has landed since, document it there and delete the line below instead of building it again.

- ~~**alphaKey — CI pipeline.**~~ **Closed 2026-06-16.** alphaKey now has `ci.yml` (lint + test + pact-verify), `nightly.yml`, `security.yml`, `weekly.yml`. Added as part of Pact contract-testing rollout sub-project 2. See [[services/alphaKey/alphaKey]].
- **alphaFrame — resource limits.** Compose services have no CPU/memory limits set (beyond the
  alphaGen Celery Beat container added in the 2026-06-14 wave). Bead from 2026-06-13 review.
- **alphaKey — sync DB calls inside async route handlers.** Confirmed still present in code
  (checked 2026-06-16): `alphakey/api/routers/auth.py` — `register`, `login`, `refresh`, `logout`,
  `logout_all` are all `async def` but call `with get_session()` (sync SQLModel `Session`) and run
  blocking DB I/O inline, no `run_in_threadpool`/`to_thread`. Stalls the event loop under load.
  From theme 4 of the 2026-06-13 review — never fixed, never documented as fixed.
- **alphaKey — first-user-developer registration race.** Confirmed still present in code
  (checked 2026-06-16): `alphakey/api/routers/auth.py:120-124`, own comment admits "COUNT then
  INSERT is a race" — two concurrent registrations on an empty DB can both see `count()==0` and
  both get promoted to `developer`. From theme 5 of the 2026-06-13 review — never fixed.
- **alphaGen — retrain_budget.json read-modify-write race.** Confirmed still present in code
  (checked 2026-06-16): `att/validation/retrain.py` reads the budget JSON, increments, then writes
  atomically via tmp-file + `os.replace` — the write itself is atomic but the read→increment→write
  sequence has no lock, so two concurrent retrain workers can read the same count and one
  increment is lost. From theme 5 of the 2026-06-13 review — never fixed.

---

## 8. alphaTest — dependency model Phase 2: private package registry

> [!todo] Phase 2 (backlog — no registry infra yet)
> **Phase 1** complete (2026-06-15): replaced `sys.path` filesystem coupling with explicit PEP 508
> git-reference declarations in `alphaTest/pyproject.toml`; `repository_dispatch` wiring added to
> alphaTrade, alphaKey, alphaLink CI; `ALPHATEST_DISPATCH_TOKEN` PAT and `SERVICE_TOKEN_SECRET`
> secrets set in all repos. Bead: `alphaTest-nbf`.
>
> **⚠️ PAT renewal required before 2027-01-01.** `ALPHATEST_DISPATCH_TOKEN` expires then.
> See [[alphaTest#CI Secrets & PAT]] for renewal steps.
>
> **Pending e2e verification (bead `alphaTest-7qi`):** full dispatch loop untested — GitHub Actions
> minutes exhausted 2026-06-15. Test when minutes reset: create a PR on alphaTrade or alphaKey,
> confirm `notify-alphatest` job fires and `alphatest-pr-gate` run appears in alphaTest Actions tab.
>
> **Phase 2** (this section) replaces the git refs with versioned wheels from a private package
> registry once that infra is available. Track via bead `alphaTest-nbf`.

### What needs doing (Phase 2)

**Infra:**
- Stand up a private PyPI registry (Artifactory, AWS CodeArtifact, or similar).
- Configure CI in each sibling repo with publish credentials + `pip install --index-url` in alphaTest.

**Per-sibling packaging work** (all three need identical treatment):

| Service | Dist name | Import pkg | Issues |
|---|---|---|---|
| alphaTrade | `alphaTrade` | `alphaTrade` | Clean, consistent |
| alphaKey | `alphakey` | `alphakey` | Dir named `alphaKey`, dist/import lowercase — note casing |
| alphaGen | `alphaGen` | `att` | Import name diverges from dist; stale `algoTradingTrainer.egg-info` to delete |

For each service:
- Bump version off `0.1.0`; adopt semver going forward.
- Add `LICENSE`, `[project].authors`, `[project].urls`, `readme` to `pyproject.toml`.
- Add a `publish.yml` GitHub Actions workflow (`hatch build && twine upload` or `hatch publish`).
- Consider splitting a lightweight `*-schemas` / `*-client` package containing only the types
  alphaTest needs, so alphaTest does not pull `torch`, `ta-lib`, `onnxruntime`, or `optuna`.

**alphaTest side (once registry exists):**
```toml
# pyproject.toml — replace [siblings] git refs with pinned versions
dependencies = [
    "alphatrade-client==1.2.0",
    "alphakey-client==2.1.0",
]
```
```yaml
# CI — add private index
pip install --index-url https://<registry>/simple '.[dev]'
```

### Constraint to preserve

The **pr-gate** regression-guard property must survive Phase 2: the pr-gate workflow must still
test the sibling's PR-HEAD code (not the pinned published version). Keep the git-ref-by-SHA
install path in `pr-gate.yml` even after registry adoption — use the registry pin only for
`post-merge`, `nightly`, `weekly`, and `manual`.

### Acceptance criteria

- [ ] alphaTest `pyproject.toml` `[dependencies]` pins versioned wheels from private registry (no git refs).
- [ ] `pip install -e '.[dev]'` resolves without internet access to GitHub (only registry access).
- [ ] `pr-gate.yml` still overrides with `git+https://...@${CALLER_SHA}` for the dispatching sibling.
- [ ] All 30 tests collect and the contract suite (`make contract`) passes green.
- [ ] `alphaGen`/`att` import-name divergence documented or resolved.

See [[alphaTest]] · Bead: `alphaTest-nbf` · Priority: P3 (no blocking feature).

---

## 9. alphaTest — alphaLink↔alphaTrade Pact contract excludes `fork` (no consumer yet)

> [!todo] Not started — add once alphaLink wires a caller
> While scoping the alphaLink↔alphaTrade Pact contract (sub-project 4 of alphaTest's
> contract-testing rollout), confirmed `POST /api/v1/models/{model_name}/fork`
> (`alphaTrade/alphaTrade/api/routers/models.py:688-763`) has **no caller anywhere in alphaLink** —
> `models/registry/page.tsx` wires promote/demote via `tradeFetch`, but never fork. Pact is
> consumer-driven (the contract is generated by running the consumer's real code), so there's
> nothing real to record for fork — it was excluded from sub-project 4's scope rather than
> written as a hypothetical/fake interaction.

**What needs doing (once alphaLink's UI calls fork):**
- Add a fork button/flow in `alphaLink/src/app/.../models/registry/page.tsx` (or wherever it
  lands) using `tradeFetch('POST', '/models/{name}/fork', ...)`.
- Add a Pact consumer interaction for it in alphaLink's contract test against alphaTrade
  (mirrors the existing promote/demote interactions added in sub-project 4).
- Add the matching provider-state branch in alphaTrade's pact-state setup endpoint (mirrors
  alphaKey's `pact_state.py` pattern from earlier sub-projects).

See [[alphaTest]] · [[services/alphaTrade]] · [[services/alphaLink]] · No bead yet — file one
when picked up.

---

*All other cross-service items from the 2026-06-06 → 2026-06-16 doc-change wave (alphaLink auth UI,
alphaTrade/alphaGen JWT iss/aud, alphaFrame compose plumbing, alphaKey contract publication,
alphaTest/alphaPerf new scenarios, the 8 contract/wiring gaps, and the alphaTrade failure_msg
redaction) are closed and documented in their owning services' own docs.*
