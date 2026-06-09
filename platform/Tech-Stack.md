---
page: Tech-Stack
tags:
  - platform
  - tech-stack
last-reviewed: 2026-06-06
---

# Tech Stack

[[README]] · [[platform/Overview]] · [[platform/Features]] · [[platform/Key-Decisions]]

Full technology inventory for projectAlpha, per service and shared infrastructure.

---

## alphaFrame — Infrastructure

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| **Container orchestration** | Docker Compose v2 | — | All services on single host |
| **Network** | Docker named bridge `platform` | — | Service discovery by name |
| **Database** | PostgreSQL | 16-alpine | SQL persistence (4 databases) |
| **Cache / broker** | Redis | 7-alpine | Celery broker, pub/sub, token denylist |
| **Object storage** | MinIO | latest | S3-compatible artifact store |
| **ML tracking** | MLflow | v3.13.0 | Experiment tracking + model registry |
| **Reverse proxy** | Nginx | alpine | TLS termination, rate limiting, SSE proxy |
| **Telemetry collector** | OpenTelemetry Collector Contrib | 0.103.0 | OTLP receiver, 3 export pipelines |
| **Metrics storage** | Prometheus | v2.52.0 | 15d retention, alert evaluation |
| **Log storage** | Grafana Loki | 3.1.0 | 7d retention, TSDB v13 |
| **Trace storage** | Grafana Tempo | 2.5.0 | 7d retention, service graphs |
| **Alert routing** | Alertmanager | v0.27.0 | Webhook routing (Slack/PagerDuty-ready) |
| **Dashboards** | Grafana | 11.1.0 | Unified observability UI |
| **Host metrics** | Prometheus Node Exporter | v1.8.1 | CPU, memory, disk |
| **Container metrics** | cAdvisor | v0.49.1 | Per-container resource usage |
| **DB metrics** | postgres_exporter | v0.15.0 | PostgreSQL query stats |
| **Redis metrics** | redis_exporter | v1.61.0 | Redis keyspace stats |
| **Celery monitoring** | Flower | latest | Celery task UI + metrics |

---

## alphaGen — ML Model Generation

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| **Language** | Python | 3.11 | — |
| **API framework** | FastAPI | ≥0.111 | REST + SSE endpoints |
| **ASGI server** | Uvicorn (standard) | ≥0.29 | HTTP server |
| **Task queue** | Celery | ≥5.4 | Async training jobs |
| **ML framework** | PyTorch | ≥2.2 | Model training (MLP, LSTM, CNN, Transformer) |
| **ONNX export** | ONNX | ≥1.16 | Framework-agnostic model format |
| **ONNX runtime** | ONNX Runtime | ≥1.18 | Parity verification |
| **Technical indicators** | TA-Lib | ≥0.4 | RSI, MACD, BBANDS, ATR, EMA, SMA, ADX, Stochastic, OBV |
| **Data manipulation** | Pandas | ≥2.0 | DataFrames |
| **Numerics** | NumPy | ≥1.26 | Arrays |
| **HP optimisation** | Optuna | ≥3.6 | Hyperparameter sweeps |
| **Data provider** | yfinance | ≥0.2 | Free OHLCV data |
| **Data provider** | Polygon API client | ≥1.12 | Premium intraday data |
| **Config validation** | Pydantic | ≥2.6 | RunConfig schema |
| **Settings** | pydantic-settings | ≥2.0 | 12-factor config |
| **CLI** | Typer | ≥0.12 | `att` command |
| **ORM** | SQLModel | ≥0.0.19 | DB models |
| **Migrations** | Alembic | ≥1.13 | Schema versioning |
| **DB driver** | psycopg2-binary | ≥2.9 | PostgreSQL |
| **HTTP client** | httpx | ≥0.27 | External API calls |
| **S3 client** | boto3 | ≥1.34 | MinIO artifact upload |
| **JWT** | PyJWT (crypto) | ≥2.8 | alphaKey token verification |
| **SSE** | sse-starlette | ≥1.6 | Server-sent events |
| **ML tracking** | mlflow-skinny | ≥2.13 | Experiment logging + registry |
| **Telemetry** | opentelemetry-sdk + instrumentation | — | OTLP traces + metrics |
| **Build** | Hatchling | — | Package build |
| **Linting** | Ruff | ≥0.4 | Fast Python linter |
| **Type checking** | Mypy | ≥1.10 | Static types |
| **Testing** | pytest, pytest-cov, testcontainers, hypothesis, mutmut | — | Unit + integration + mutation |

---

## alphaTrade — Trading Executor

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| **Language** | Python | 3.11 | — |
| **API framework** | FastAPI | ≥0.111 | REST + SSE + health endpoints |
| **ASGI server** | Uvicorn (standard) | ≥0.29 | HTTP server |
| **ONNX Runtime** | onnxruntime | ≥1.17 | ONNX model inference (no PyTorch) |
| **Technical indicators** | TA-Lib | ≥0.4.28 | Feature replication from alphaGen |
| **Data manipulation** | Pandas | ≥2.2 | DataFrames |
| **Numerics** | NumPy | ≥1.26 | Arrays |
| **Data provider** | yfinance | ≥0.2.40 | Default OHLCV source |
| **Data provider** | polygon-api-client | ≥1.13 | Premium data |
| **Scheduling** | APScheduler | ≥3.10 | Backtest cron scheduler |
| **Exchange calendar** | exchange-calendars | ≥4.5 | NYSE bar-close timing |
| **HTTP client** | httpx, aiohttp | ≥0.27, ≥3.9 | T212 API calls |
| **Retry logic** | tenacity | ≥8.2 | Exponential backoff for T212 |
| **ORM** | SQLModel | ≥0.0.19 | 23 table models |
| **Migrations** | Alembic | ≥1.13 | Schema versioning (23 migrations) |
| **DB driver** | psycopg2-binary | ≥2.9 | PostgreSQL (SQLite fallback) |
| **Config validation** | Pydantic + pydantic-settings | ≥2.7, ≥2.3 | Settings + overrides |
| **S3 client** | boto3 | ≥1.34 | MinIO model download |
| **Redis** | redis[asyncio] | ≥5.0 | Token denylist cache |
| **ML registry** | mlflow-skinny | ≥2.13 | Model version management |
| **Metrics** | prometheus-client | ≥0.20 | Prometheus exposition |
| **Telemetry** | opentelemetry-sdk + instrumentation | — | OTLP traces + metrics |
| **Config file** | pyyaml | ≥6.0 | `overrides.yaml` parsing |
| **CLI** | Typer + Rich | ≥0.12, ≥13 | Management commands |
| **Logging** | python-json-logger | ≥3.2 | Structured JSON logs |
| **Testing** | pytest, pytest-asyncio, respx, freezegun | — | Unit + integration |
| **Security scanning** | bandit, pip-audit | — | Dependency + code security |

---

## alphaLink — Frontend

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| **Framework** | Next.js (App Router) | 16.2 | SSR + BFF API routes |
| **UI library** | React | 18 | Component tree |
| **Language** | TypeScript | 5 | Type safety |
| **Styling** | Tailwind CSS | 3.4 | Utility classes |
| **UI components** | shadcn/ui (Radix UI) | — | Accessible component primitives |
| **Icons** | Lucide React | — | Icon set |
| **Toast notifications** | Sonner | — | Non-blocking alerts |
| **State management** | Zustand | 5 | Auth store + session config store |
| **Server data** | React Query (TanStack) | 5 | Caching + invalidation |
| **Data fetching** | SWR | 2.4 | Alternative fetch hook |
| **Charts** | Recharts | 3.8 | Backtest equity, P&L charts |
| **Charts** | lightweight-charts | 5.2 | Candlestick / OHLCV charts |
| **Config serialisation** | js-yaml | 2.8 | RunConfig → YAML |
| **Validation** | Zod | 4.4 | Schema validation |
| **CSS utilities** | clsx, class-variance-authority | — | Conditional class composition |
| **Unit testing** | Jest | — | Component tests |
| **E2E testing** | Playwright | — | Browser automation |
| **HTTP mocking** | MSW (Mock Service Worker) | — | Test mocking |
| **Build** | SWC | — | Fast TypeScript compilation |
| **Linting** | ESLint | — | JavaScript/TypeScript linting |
| **Secret scanning** | Gitleaks | — | Pre-commit secret detection |

---

## alphaKey — Auth Service

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| **Language** | Python | 3.11 | — |
| **API framework** | FastAPI | — | REST auth endpoints |
| **ASGI server** | Uvicorn | — | HTTP server |
| **ORM** | SQLModel (SQLAlchemy) | — | 6 table models |
| **Migrations** | Alembic | — | Schema versioning |
| **DB driver** | psycopg2-binary | — | PostgreSQL |
| **JWT** | PyJWT (crypto) | ≥2.8 | ES256 token issuance + verification |
| **Cryptography** | cryptography (hazmat) | — | ECDSA key generation, Fernet encryption |
| **Password hashing** | argon2-cffi | — | Argon2 password hash |
| **Async Redis** | redis[asyncio] | ≥5.0 | JTI denylist |
| **HTTP client** | httpx | — | Credential verification calls |
| **Email validation** | pydantic[email] | — | Email format validation |
| **Settings** | pydantic-settings | — | 12-factor config |
| **Metrics** | prometheus-client | — | Prometheus exposition |
| **Telemetry** | opentelemetry-sdk | — | Traces + metrics |
| **Logging** | python-json-logger | — | Structured JSON |
| **Testing** | pytest, respx, freezegun | — | Unit + integration |
| **Build** | Hatchling | — | Package build |

---

## Shared / Cross-Cutting

| Category | Technology | Where |
|---|---|---|
| **State management** | GasTownHall Beads (`.beads/`) | All services (issue tracking / state mgmt) |
| **Secret management** | `.env` + alphaKey vault | All services |
| **Package management** | pip / pyproject.toml | Python services |
| **Package management** | npm / package-lock.json | alphaLink |
| **Containerisation** | Docker | All services |
| **Networking** | Docker Compose bridge `platform` | All services |
| **CI** | GitHub Actions (`.github/workflows/`) | alphaGen, alphaTrade, alphaLink |
| **Python version** | pyenv `.python-version` | alphaGen, alphaTrade, alphaKey |
