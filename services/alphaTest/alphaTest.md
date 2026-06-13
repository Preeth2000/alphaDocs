---
service: alphaTest
status: planned
stack: TBD
owner: ""
last-reviewed: 2026-06-06
tags:
  - service/alphaTest
  - status/planned
---

# alphaTest

> Regression testing suite — validates that new model deployments do not introduce regressions in trading behaviour, signal quality, or API contract compatibility.

**Status:** ⬜ Planned — not yet implemented  
**Repo path:** `projectAlpha/alphaTest/` (empty directory)

---

## Intended Scope

Based on platform context, alphaTest is expected to cover:

- **Model regression testing**: Run new ONNX models against a fixed historical dataset and compare signal distributions, hit rate, and Sharpe against a baseline — catch degradations before promotion to Production in MLflow.
- **API contract testing**: Verify that [[services/alphaGen/API|alphaGen API]] and [[services/alphaTrade/API|alphaTrade API]] endpoints conform to expected request/response schemas across deployments.
- **Integration smoke tests**: Post-deploy checks that all services are healthy and the end-to-end flow (train → publish → sync → inference) works.
- **Backtest reproducibility**: Assert that re-running a training config produces equivalent backtest metrics (within tolerance) — catch non-determinism.

---

## Planned Integrations

| Service | How alphaTest would interact |
|---|---|
| [[services/alphaGen/alphaGen\|alphaGen]] | Submit training jobs, poll for completion, assert gate metrics |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | POST `/backtest/trigger`, assert backtest results |
| [[services/alphaFrame/alphaFrame\|alphaFrame]] | Connect to Postgres + MinIO for fixture setup/teardown |
| [[services/alphaKey/alphaKey\|alphaKey]] | Service token auth for API calls |

---


## Test Catalogue

> Regression scenario catalogue (2026-06-13 review): [[services/alphaTest/Regression-Scenarios|Regression Scenarios]]

---

> [!warning] Not implemented
> No code, no endpoints, no configuration exists yet. This page is a placeholder for planned functionality.

---

*Update this page when implementation begins. See [[_templates/service-template|service-template]] for the standard structure.*
