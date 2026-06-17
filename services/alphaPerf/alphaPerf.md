---
service: alphaPerf
status: planned
stack: TBD
owner: ""
last-reviewed: 2026-06-06
tags:
  - service/alphaPerf
  - status/planned
---

# alphaPerf

> Performance testing suite — load tests, latency benchmarks, and throughput profiling for the alphaPlatform services.

**Status:** ⬜ Planned — not yet implemented  
**Repo path:** `projectAlpha/alphaPerf/` (empty directory)

---

## Intended Scope

Based on platform context, alphaPerf is expected to cover:

- **API load testing**: Stress [[alphaDocs/services/alphaGen/API|alphaGen]] and [[alphaDocs/services/alphaTrade/API|alphaTrade]] endpoints under concurrent load (k6, Locust, or similar).
- **Inference latency benchmarking**: Measure ONNX model inference time per tick across different model architectures (MLP vs LSTM vs Transformer) and window sizes.
- **Scheduler tick overhead**: Profile end-to-end latency from bar close to order submission under different model counts.
- **Database throughput**: Benchmark Postgres write throughput under peak trading activity (signals, orders, positions, P&L updates simultaneously).
- **SSE stream capacity**: Test concurrent SSE subscriber count without dropped events.

---

## Planned Integrations

| Service | Purpose |
|---|---|
| [[alphaGen\|alphaGen]] | Benchmark training job submission + log streaming |
| [[alphaTrade\|alphaTrade]] | Benchmark inference tick latency + API throughput |
| [[alphaFrame\|alphaFrame]] | Prometheus metrics for baseline comparison |

---


## Test Catalogue

> Performance budgets + scenario matrix (2026-06-13 review): [[Performance-Test-Plan|Performance Test Plan]]

---

> [!warning] Not implemented
> No code, no endpoints, no configuration exists yet. This page is a placeholder for planned functionality.

---

*Update this page when implementation begins. See [[_templates/service-template|service-template]] for the standard structure.*
