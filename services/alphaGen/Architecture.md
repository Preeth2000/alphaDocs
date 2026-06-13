---
service: alphaGen
page: Architecture
tags:
  - service/alphaGen
  - architecture
---

# alphaGen — Architecture

[[services/alphaGen/alphaGen|alphaGen]] · [[services/alphaGen/Interactions|Interactions]] · [[services/alphaGen/API|API]] · [[services/alphaGen/Data|Data]] · [[services/alphaGen/Config|Config]]

---

## Purpose

alphaGen trains multi-class (BUY/SELL/HOLD) classifiers on OHLCV price data, validates their quality through a configurable gate, runs walk-forward backtests, exports to ONNX, and publishes to MinIO so [[services/alphaTrade/alphaTrade|alphaTrade]] can consume them. Exposes a FastAPI + Celery platform for async job management, and retains a CLI (`att`) for local dev and CI.

---

## Internal Modules (`src/att/`)

| Module | Path | Responsibility |
|---|---|---|
| `config` | `att/config.py` | Pydantic RunConfig — full YAML → Python config schema |
| `cli` | `att/cli.py` | Typer CLI: `att train`, `att backtest`, `att validate`, `att verify` |
| `api` | `att/api/` | FastAPI app, routers, Celery app + worker, DB models |
| `data` | `att/data/` | OHLCV fetch (yfinance / Polygon), parquet cache, fundamentals |
| `features` | `att/features/` | TA-Lib indicator computation, normalization, windowing, fundamentals join; `feature_names` derived from fitted DataFrame columns (not config) |
| `labels` | `att/labels/` | Label generation strategies (ForwardReturn, TripleBarrier) |
| `models` | `att/models/` | NN architectures: MLP, LSTM, CNN, Transformer |
| `train` | `att/train/` | Walk-forward training loop, dataset splits, metrics |
| `validate` | `att/validate/` | Validation gate, retrain manager, bar/config checks |
| `backtest` | `att/backtest/` | Bar-by-bar backtest engine + report writer |
| `export` | `att/export/` | ONNX export, parity verification, manifest writer |
| `publish` | `att/publish/` | MinIO artifact upload (model.onnx, manifest.json, backtest.json) |
| `sweep` | `att/sweep/` | Optuna hyperparameter sweep (optuna_runner) |
| `security` | `att/security/` | alphaKey JWT offline verification (ES256 JWKS + Redis denylist) |
| `secrets` | `att/secrets/` | alphaKey vault client — fetch per-user secrets |
| `telemetry` | `att/telemetry.py` | OpenTelemetry setup (OTLP exporter) |
| `mlflow_utils` | `att/mlflow_utils.py` | MLflow logging + model registration |

---

## Execution Surfaces

Two independent surfaces share the `att` library:

| Surface | Entry point | Who uses it | Infra needed |
|---|---|---|---|
| **FastAPI + Celery** | `att/api/app.py` | Platform, alphaLink | Postgres, Redis, MinIO, MLflow |
| **CLI** (`att` command) | `att/cli.py` | Local dev, CI | None (direct library calls) |

The CLI **does not touch the API** — it calls att.* library functions directly.

---

## Training Pipeline (Celery task: `att.train_and_export`)

```mermaid
sequenceDiagram
    participant API as FastAPI POST /runs
    participant RD as Redis (db0)
    participant W as Celery Worker
    participant ATT as att library
    participant PG as PostgreSQL
    participant ML as MLflow
    participant MN as MinIO

    API->>PG: INSERT run (status=queued)
    API->>RD: enqueue train_and_export task
    W->>PG: UPDATE run (status=running)
    W->>RD: publish run:{id}:log (status=running)
    W->>ATT: fetch OHLCV data
    W->>ATT: compute features + labels
    W->>ATT: walk-forward train (N folds)
    W->>RD: stream log lines → run:{id}:log
    W->>ATT: export ONNX
    W->>ATT: write manifest.json
    W->>ATT: run backtest
    alt force_save=false
        W->>ATT: validation gate check
        alt gate passes
            W->>ML: log_and_register_run
            W->>MN: publish artifacts
            W->>PG: UPDATE run (status=complete, minio_version, artifact_prefix)
            W->>RD: publish model.ready
        else gate fails
            W->>PG: UPDATE run (status=failed, error=gate:category:decision:reason)
            W->>RD: publish status=gate_failed
        end
    else force_save=true
        W->>ML: log_and_register_run
        W->>MN: publish artifacts
        W->>PG: UPDATE run (status=complete)
        W->>RD: publish model.ready
    end
```

---

## Stale Job Detection

On worker startup (`worker_ready` signal):
- Runs `running` for > 30 minutes → marked `failed`
- Runs `queued` for > 5 minutes → marked `failed`

---

## Job State Machine

```mermaid
stateDiagram-v2
    [*] --> queued: POST /runs
    queued --> running: Celery worker picks up
    running --> complete: Gate passed (or force_save)
    running --> failed: Gate failed or exception
    running --> cancelled: DELETE /runs/{id}
    failed --> queued: POST /runs/{id}/force-save (new run, source_run_id set)
    complete --> published: POST /runs/{id}/publish (re-publish to MinIO)
```

---

## Multi-Tenancy

- Each run has `user_id` (set from JWT `sub` claim at creation) and `visibility` (private | public)
- All `/runs` endpoints require Bearer JWT via `att.security.alphakey_auth.require_auth`; endpoints that operate on a specific run (`GET`, `DELETE`, `force-save`, `publish`, `log`) enforce `run.user_id == claims.sub` and return `403` on mismatch
- MinIO paths namespaced: `{minio_user}/{minio_account}/{run_name}/{version}/`
- alphaKey vault provides per-user `minio_user`, `minio_account`, `POLYGON_API_KEY` (when `SECRETS_SOURCE=alphakey`)
- Public runs visible to all users via `GET /models` with visibility filter

---

## Key Design Decisions

- **Single Celery worker (concurrency=1)**: Prevents GPU/CPU contention during training. Jobs queue.
- **SSE from Redis pub/sub** (not DB polling): Log lines stream in real-time from worker → Redis → browser. Terminal runs replay from log file.
- **ONNX as deployment artifact**: Framework-agnostic — alphaTrade uses `onnxruntime`, no PyTorch needed.
- **Manifest is the contract**: Everything alphaTrade needs (feature_names, window, norm_stats, input_shape, model_hash) is in `manifest.json` — no dependency on training code. `feature_names` is written from `FeaturePipeline._fitted_columns` (actual fitted DataFrame columns post-`fit_transform`), not reconstructed from config — so fundamentals columns (eps, sector_code, etc.) and optional OHLCV extras (VWAP, Transactions) are always included when present.
- **MLflow = registry, MinIO = artifacts**: MLflow tracks experiments + aliases; MinIO is the actual binary storage. MLflow links to MinIO paths.
