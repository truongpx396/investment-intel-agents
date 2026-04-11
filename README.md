# Investment Intel AI Agents

**Real-time crypto signal alerts & AI-powered daily digests — delivered to Telegram.**

[![CI — Go](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-go.yml/badge.svg)](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-go.yml)
[![CI — Python](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-python.yml/badge.svg)](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-python.yml)
[![CI — Frontend](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-frontend.yml/badge.svg)](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-frontend.yml)

---

## Overview

Investment Intel AI Agents is a POC platform that lets users configure **rule-based signal strategies** for crypto assets (BTC & ETH for POC) and receive **real-time Telegram notifications** when conditions fire (< 60 s SLA). Users also maintain a personal **watchlist** and get a daily **AI-summarised digest** of news and price movements via Telegram.

### Key Features

| Feature | Description |
|---------|-------------|
| 🎯 **Signal Strategies** | Create strategies with 6 signal types: price threshold, % price change, RSI, volume spike, MACD crossover, Bollinger Band breach |
| ⚡ **Real-Time Alerts** | Telegram notifications within 60 seconds of signal trigger via 30-second polling |
| 📰 **Daily Digest** | AI-summarised news & price movements for watchlisted projects, delivered by 09:00 UTC |
| 🔗 **Telegram Linking** | Self-service Telegram account connection — no admin needed |
| 📊 **Alert History** | Full audit trail of all triggered alerts, filterable by strategy |
| 💳 **Billing** | Stripe-powered subscription management |
| 🛡️ **Strategy Definition Format (SDF)** | Portable JSON schema for strategies — extensible for future LLM-generated strategies |

---

## Architecture

```
┌─────────────┐     HTTPS      ┌──────────────┐     NATS      ┌───────────────────┐
│  React SPA  │ ──────────────→│  Go Backend  │ ────────────→ │  Alert Dispatcher │
│  (Vite +    │  TanStack      │  (REST API)  │  JetStream    │  (Telegram Bot)   │
│  React 19)  │  Query         │              │               └───────────────────┘
└─────────────┘                │              │
       │                       │              │  HTTP    ┌──────────────────┐
       │                       │  Signal      │ ───────→ │  Python ai-svc   │
       │                       │  Evaluator   │          │  (FastAPI)       │
       │                       │  (30s tick)  │          │  • RSI / MACD /  │
       └── Cloudflare CDN      │              │          │    Bollinger /   │
                               └──────────────┘          │    Volume / %Chg │
                                      │                  │  • News + VADER  │
                                      │                  └──────────────────┘
                               ┌──────────────┐
                               │Agent Gateway │  Cron 08:30 UTC
                               │  (Digest     │ ──→ Fetch news
                               │   Agent)     │ ──→ LLM summarise
                               │              │ ──→ Telegram digest
                               └──────────────┘
```

### Domain-Decoupled Platform Architecture

The codebase follows a **domain-decoupled platform** design (see [ADR](specs/001-investment-intel-poc/architecture-domain-decoupling.md)) where cross-cutting capabilities (auth, billing, notification, event bus) are isolated from investment-specific business logic. Each platform concern defines **provider-agnostic interfaces**; swapping a provider (e.g., Supabase → Casdoor, Stripe → LemonSqueezy) requires only a new provider directory — zero changes to middleware, domain code, or other providers.

- **`platform/`** — domain-agnostic: auth, billing, notification, event bus, health, user profile, admin
- **`domain/investment/`** — investment-specific: strategies, signals, alerts, watchlist, market data
- **Platform tables** use `platform_` namespace; **domain tables** use `app_` namespace; **Agent Gateway** uses `gc_` namespace

### Service Responsibility

| Service | Owns | Does NOT |
|---------|------|----------|
| **Go Backend — Platform** | REST API server, auth middleware, billing gate, notification dispatch, event bus, health | Import domain code, call LLM, know about strategies or market data |
| **Go Backend — Domain** | Signal evaluation, alert formatting, strategy CRUD, watchlist, market data adapters | Import provider SDKs directly, implement auth/billing logic |
| **Python ai-service — Platform** | Compute engine, enrichment pipeline, content provider interfaces | Know about crypto indicators or news providers |
| **Python ai-service — Domain** | Technical indicators (pandas-ta), news fetching, sentiment scoring | Evaluate thresholds, send Telegram, touch NATS |
| **Agent Gateway — Framework** | Cron scheduling, LLM provider, Telegram channel, HTTP fetch | Connect to NATS, dispatch real-time alerts, contain domain logic |
| **Agent Gateway — Domain Skills** | Digest persona, crypto-digest skill | Bypass framework tools, directly access databases |
| **React SPA** | Platform pages (auth, billing, settings) + domain pages (dashboard, strategies, alerts) | Call APIs directly, store secrets |

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| **Backend API** | Go 1.24+, Chi router, zerolog |
| **AI / Data Service** | Python 3.12+, FastAPI, pandas-ta, vaderSentiment, structlog |
| **Agent Gateway** | Pluggable — GoClaw v1.74+ (default), OpenClaw, PicoClaw, nanoBot, ZeroClaw |
| **Frontend** | React 19, Vite, TailwindCSS v4, TanStack Router + Query, Storybook 8 |
| **Auth** | Supabase (PostgreSQL + Auth + RLS) |
| **Database** | PostgreSQL 18 + pgvector |
| **Cache** | Redis 7 |
| **Message Queue** | NATS JetStream (signal events, alert queue) |
| **API Gateway** | Traefik v3 (routing, TLS, rate limiting) |
| **Billing** | Stripe SDK |
| **Market Data** | CoinGecko (spot price), CryptoCompare (OHLCV histominute) |
| **News** | CryptoPanic (primary), DuckDuckGo (fallback) |
| **LLM** | Anthropic Claude 3.5 Haiku (via Agent Gateway) |
| **Monitoring** | Prometheus + Grafana, Agent Gateway OTLP traces |
| **CI** | GitHub Actions |
| **Deployment** | DigitalOcean Droplets + Cloudflare DNS/CDN/WAF |
| **Spec Workflow** | [Speckit](https://github.com/nextlevelbuilder/speckit) v0.3.1 (Spec-Driven Development) |
| **Agent Gateway** | Pluggable (see [Agent Gateway Abstraction](specs/001-investment-intel-poc/agent-gateway-abstraction.md)) — default: [GoClaw](https://github.com/nextlevelbuilder/goclaw) v1.74+ |

---

## Signal Types

| Signal | Description | Data Source | Parameters |
|--------|-------------|-------------|------------|
| **Price Threshold** | Fires when spot price crosses above/below a fixed value | CoinGecko spot | `operator`, `threshold` |
| **% Price Change** | Fires on price change exceeding threshold over a time window | CryptoCompare (≤ 240 min) / CoinGecko (24h) | `operator`, `threshold`, `window_minutes` ∈ {5, 15, 60, 240, 1440} |
| **RSI** | Fires when 14-period RSI crosses overbought/oversold level | CryptoCompare → ai-service | `operator`, `threshold`, `candle_minutes` ∈ {5, 15, 60} |
| **Volume Spike** | Fires when volume exceeds % of 20-period SMA | CryptoCompare → ai-service | `volume_threshold_pct`, `candle_minutes` ∈ {5, 15, 60} |
| **MACD Crossover** | Fires on MACD line crossing signal line | CryptoCompare → ai-service | `cross_direction` ∈ {bullish, bearish}, `candle_minutes` |
| **Bollinger Breach** | Fires when close price crosses Bollinger Band | CryptoCompare → ai-service | `band_direction` ∈ {upper, lower}, `candle_minutes` |

---

## Project Structure

```
investment-intel-agents/
│
├── backend/                          # Go API service
│   ├── cmd/server/main.go            # Wires providers + platform + domain; starts server
│   ├── platform/                     # Domain-agnostic layer (reusable across projects)
│   │   ├── auth/                     # Auth interfaces + middleware
│   │   │   ├── interfaces.go         # AuthProvider, TokenValidator
│   │   │   ├── middleware.go          # Auth middleware (provider-agnostic)
│   │   │   └── supabase/             # Swappable: Supabase provider
│   │   ├── billing/                  # Billing interfaces + gate middleware
│   │   │   ├── interfaces.go         # BillingProvider, SubscriptionChecker
│   │   │   └── stripe/               # Swappable: Stripe provider
│   │   ├── notification/             # Notification dispatch (provider-agnostic)
│   │   │   ├── interfaces.go         # Sender, AccountLinker, Dispatcher
│   │   │   └── telegram/             # Swappable: Telegram channel
│   │   ├── eventbus/                 # Event pub/sub (provider-agnostic)
│   │   │   ├── interfaces.go         # Publisher, Consumer
│   │   │   └── nats/                 # Swappable: NATS JetStream
│   │   ├── health/                   # /health, /ready endpoints
│   │   ├── user/                     # User profile (timezone, settings)
│   │   ├── admin/                    # Admin middleware + user management
│   │   └── server/                   # HTTP server setup, router mounting
│   ├── domain/                       # Domain-specific layer (swappable)
│   │   └── investment/               # Investment intelligence domain module
│   │       ├── register.go           # Mounts routes, consumers, workers
│   │       ├── strategies/           # Strategy CRUD, SDF validation, import/export
│   │       ├── signals/              # Signal evaluator (30s polling loop)
│   │       ├── alerts/               # Alert persistence, dispatcher, re-driver
│   │       ├── watchlist/            # Watchlist CRUD, digest content
│   │       ├── marketdata/           # Market data adapters (CoinGecko, CryptoCompare)
│   │       └── config/               # Seed config loader (signal types, projects)
│   ├── pkg/                          # Shared utilities (domain-agnostic)
│   │   ├── httputil/                 # HTTP helpers, error response envelope
│   │   ├── validate/                 # JSON Schema validation helpers
│   │   └── testutil/                 # Test fixtures, DB helpers
│   └── tests/
│       ├── platform/                 # Platform contract + integration tests
│       └── domain/investment/        # Domain-specific tests (contract, integration, unit)
│
├── ai-service/                       # Python AI/data service
│   ├── src/
│   │   ├── platform/                 # Domain-agnostic (reusable)
│   │   │   ├── compute/              # Generic computation engine (cache, resample)
│   │   │   ├── enrichment/           # Generic content enrichment (VADER sentiment, dedup)
│   │   │   └── content/              # ContentProvider ABC, ContentItem dataclass
│   │   └── domain/investment/        # Investment-specific
│   │       ├── indicators/           # RSI, MACD, Bollinger, volume spike, pct_change
│   │       ├── news/                 # CryptoPanic + DuckDuckGo adapters
│   │       └── projects/             # DB-backed project registry
│   └── tests/
│
├── agent-gateway/                     # Pluggable agent gateway (see agent-gateway-abstraction.md)
│   ├── goclaw/                       # GoClaw config (default)
│   ├── openclaw/                     # OpenClaw config
│   ├── picoclaw/                     # PicoClaw config
│   ├── nanobot/                      # nanoBot config
│   └── zeroclaw/                     # ZeroClaw config
│
├── frontend/                         # React SPA
│   ├── src/
│   │   ├── components/               # Design system primitives + stories
│   │   ├── pages/                    # Auth, dashboard, strategies, alerts, etc.
│   │   └── services/                 # TanStack Query hooks
│   ├── .storybook/                   # Storybook config
│   └── tests/
│       ├── unit/                     # Vitest + RTL
│       └── e2e/                      # Playwright
│
├── config/                           # Externalised business configuration
│   ├── seed.yaml                     # Signal types, project seeds, system tunables
│   └── seed.schema.json              # JSON Schema for seed.yaml validation
│
├── contracts/                        # Portable format schemas
│   └── strategy-definition.schema.json  # Strategy Definition Format (SDF)
│
├── infra/
│   ├── docker-compose.yml            # Local dev (all services)
│   ├── docker-compose.prod.yml       # Production overrides
│   ├── traefik/                      # Traefik config
│   └── monitoring/                   # Prometheus + Grafana
│
├── migrations/                       # PostgreSQL migrations (golang-migrate)
│
├── specs/                            # Feature specifications
│   └── 001-investment-intel-poc/
│       ├── spec.md                   # Feature specification
│       ├── plan.md                   # Implementation plan
│       ├── tasks.md                  # Task list (110 tasks, 11 phases)
│       ├── contracts/                # REST API + NATS event contracts
│       └── checklists/               # Requirements checklist
│
├── design-system/                    # Design system documentation
│   └── investment-intel/
│       └── MASTER.md
│
└── .specify/                         # Speckit memory + templates
    └── memory/
        └── constitution.md           # Project constitution v1.0.0
```

---

## Getting Started

### Prerequisites

- **Go** 1.24+
- **Python** 3.12+ with `uv` package manager
- **Node.js** 22+ with npm
- **Docker** & **Docker Compose** v2
- **Telegram Bot Token** — [create one via BotFather](https://core.telegram.org/bots#botfather)
- **Supabase** project (PostgreSQL + Auth)
- **Stripe** account (test mode) with webhook signing secret
- **CryptoPanic** API token (free tier)
- **Anthropic** API key (for Agent Gateway LLM calls)

### Local Development Setup

```bash
# 1. Clone the repository
git clone https://github.com/truongpx396/investment-intel-agents.git
cd investment-intel-agents

# 2. Copy environment variables
cp .env.example .env
# Edit .env with your actual credentials

# 3. Start all services
docker compose up -d

# 4. Run database migrations
docker compose exec backend migrate -path /migrations -database "$DATABASE_URL" up

# 5. Verify services are healthy
curl http://localhost:8080/health        # Go backend
curl http://localhost:8000/health        # Python ai-service
curl http://localhost:18790              # Agent Gateway dashboard
```

### Service URLs (Local Dev)

| Service | URL |
|---------|-----|
| React SPA | `http://localhost:5173` |
| Go Backend API | `http://localhost:8080/api/v1/` |
| Python ai-service | `http://localhost:8000` (internal only) |
| Agent Gateway Dashboard | `http://localhost:18790` |
| Traefik Dashboard | `http://localhost:8090` |
| Prometheus | `http://localhost:9090` |
| Grafana | `http://localhost:3000` |

### Running Tests

```bash
# Go backend
cd backend && go test ./... -race -coverprofile=coverage.out

# Python ai-service
cd ai-service && pytest --cov=src --cov-fail-under=80

# Frontend
cd frontend && npm run test              # Vitest unit tests
cd frontend && npm run test:e2e          # Playwright E2E tests
cd frontend && npm run storybook:build   # Build + test Storybook
```

---

## Configuration

### Environment Variables

All secrets and environment-specific values are set via `.env`:

| Variable | Description |
|----------|-------------|
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key |
| `SUPABASE_JWT_SECRET` | JWT verification secret |
| `TELEGRAM_BOT_TOKEN` | Single Telegram bot token (shared by backend + Agent Gateway) |
| `STRIPE_SECRET_KEY` | Stripe API secret key |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret |
| `CRYPTOPANIC_TOKEN` | CryptoPanic API token |
| `AGENT_GATEWAY_LLM_API_KEY` | Anthropic API key for Agent Gateway LLM |
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis connection string |
| `NATS_URL` | NATS server URL |
| `SIGNAL_COOLDOWN_SECONDS` | Signal cooldown to prevent alert spam (default: 300) |

### Seed Configuration

Business-level configuration is externalised in `config/seed.yaml` — signal type definitions, project seed data, and system tunables. No code changes needed to:

- Adjust allowed `candle_minutes` or `window_minutes` values
- Tune poll interval, cooldown, or cache TTL
- Seed a new project for the watchlist

The seed file is validated against `config/seed.schema.json` at boot. See [plan.md §Seed Configuration](specs/001-investment-intel-poc/plan.md) for the full structure.

### Strategy Definition Format (SDF)

Strategies use a portable JSON format defined by `contracts/strategy-definition.schema.json`. The schema uses a **discriminated union** on `signal_type` so each rule carries only its relevant parameters:

```json
{
  "name": "BTC Breakout",
  "asset": "btc",
  "rules": [
    {
      "signal_type": "price_threshold",
      "operator": "above",
      "threshold": 70000
    },
    {
      "signal_type": "rsi",
      "operator": "above",
      "threshold": 70,
      "candle_minutes": 15
    }
  ]
}
```

The SDF is the wire format for `POST /strategies`, `GET /strategies/:id`, and the import/export endpoints. It's designed to support future LLM-generated strategies: `User text → LLM (with schema) → SDF JSON → POST /strategies/import`.

---

## Docker Networks

The local and production Docker Compose files define two explicit networks for security:

| Network | Services | Purpose |
|---------|----------|---------|
| `public` | Traefik, frontend, backend | Internet-facing |
| `internal` | backend, ai-service, Agent Gateway, NATS, Redis, Postgres | Internal only — ai-service is **never** exposed via Traefik |

---

## Implementation Phases

The project is planned across **11 phases** with **110 tasks**:

| Phase | Focus | Key Deliverables |
|-------|-------|------------------|
| 1 — Setup | Repo, CI, design system, seed config, SDF schema | Monorepo, GitHub Actions, Storybook, `seed.yaml`, SDF JSON Schema |
| 2 — Foundation | Auth, Supabase, NATS, Stripe bootstrap | User registration, JWT middleware, NATS streams |
| 3 — Strategy Config (P1) | Signal strategy CRUD, SDF validation, import/export | Strategy creation, 6 signal types, import/export endpoints |
| 4 — Telegram Linking (P1) | Self-service Telegram account connection | Deep-link flow, broken-link detection |
| 5 — Notifications (P2) | Real-time signal alerts via Telegram | 30s polling, NATS fanout, < 60s delivery SLA |
| 6 — Alert History (P2) | Read-only alert audit trail | Paginated history, strategy filter, status badges |
| 7 — Watchlist & Digest (P3) | Watchlist + Agent Gateway daily digest | CryptoPanic integration, Claude summarisation, 08:30 UTC cron |
| 8 — Billing | Stripe subscription management | Checkout, webhooks, subscription gating |
| 9 — Admin | User management panel | Admin middleware, user list, subscription overrides |
| 10 — Quality Gates | Observability, performance, CI gates | Prometheus, Grafana, Playwright E2E, Lighthouse, k6 |
| 11 — Deploy | Production deployment & handoff | DigitalOcean, Cloudflare, smoke tests, `v0.1.0-poc` tag |

See [tasks.md](specs/001-investment-intel-poc/tasks.md) for the complete task list.

---

## Performance Targets

| Metric | Target |
|--------|--------|
| Signal → Telegram notification | < 60 s (30s poll + < 30s delivery) |
| REST API p95 latency | < 200 ms |
| React SPA LCP | ≤ 2.5 s |
| React SPA CLS | ≤ 0.1 |
| Daily digest delivery | Before 09:00 UTC, 100% of days |
| Signal evaluator memory | ≤ 256 MB peak |
| Agent Gateway digest pipeline | ≤ 5 min for 50 users × 10 projects |

---

## Quality Standards

This project is governed by a [constitution](.specify/memory/constitution.md) with four non-negotiable principles:

1. **Code Quality** — Linters pass with zero warnings; cyclomatic complexity ≤ 10 per function; all public APIs documented
2. **Testing Standards** — TDD (Red-Green-Refactor); unit coverage ≥ 80% globally, ≥ 95% on critical paths (auth, signals, alerts, billing)
3. **UX Consistency** — Design system enforced; loading/empty/error states mandatory; WCAG 2.1 AA compliance
4. **Performance** — Budgets defined at spec time; regressions > 10% block merge

### CI Quality Gates

- Lint + static analysis (golangci-lint, ruff, ESLint) — zero warnings
- Test suite passing with coverage thresholds
- Contract tests updated for any API surface change
- Storybook build + interaction tests
- Performance baseline — no regression > 10%

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Rule-based signals** (no ML) | Deterministic, auditable, and debuggable — ML deferred to post-POC |
| **Go for signal evaluator** | Latency-critical 30s tick; strongly-typed; low memory overhead |
| **Python for indicators** | pandas-ta computes RSI/MACD/Bollinger in one line; no hand-rolled math |
| **Pluggable Agent Gateway** | Built-in cron, LLM provider, Telegram channel — replaces bespoke orchestration; GoClaw default, swappable to OpenClaw / PicoClaw / nanoBot / ZeroClaw |
| **Single Telegram bot** | One bot shared between Go (alerts, linking) and Agent Gateway (digest); Go owns webhook |
| **YAML seed config** | Version-controlled, diff-friendly, reviewable — DB stores user data, seed stores shape |
| **SDF with JSON Schema** | Machine-validatable, LLM-friendly, extensible via `oneOf` discriminated union |
| **NATS JetStream** | At-least-once delivery with durable consumers; `NakWithDelay` for re-drive (no re-publish) |
| **Two Docker networks** | ai-service isolated on `internal` only — never exposed to public internet |
| **Speckit for SDD** | AI-assisted spec workflow ensures spec → plan → tasks → code traceability; constitution enforces quality gates |
| **Agent Gateway over bespoke orchestration** | Single binary/container with cron, LLM, Telegram, and agent tools built in — no custom Python scheduler needed |

---

## Powered By

### Speckit — Spec-Driven Development

[Speckit](https://github.com/nextlevelbuilder/speckit) (`v0.3.1`) powers the entire development workflow for this project. It's an AI-assisted Spec-Driven Development (SDD) toolkit that integrates with GitHub Copilot to guide the process from idea → specification → plan → tasks → implementation.

The workflow stages (each backed by a Copilot agent under `.github/agents/`):

```
/speckit.clarify  →  /speckit.specify  →  /speckit.plan  →  /speckit.tasks  →  /speckit.implement
     ↓                    ↓                    ↓                  ↓                    ↓
  Resolve            spec.md              plan.md            tasks.md            Code + tests
  ambiguity        (requirements)       (architecture)     (110 tasks)         (TDD workflow)
```

Additional commands: `/speckit.analyze` (cross-artifact consistency checks), `/speckit.checklist` (requirements traceability), `/speckit.constitution` (project governance).

All spec artifacts live in `specs/001-investment-intel-poc/`; project memory and templates in `.specify/`.

### Agent Gateway — Pluggable AI Agent Framework

The project uses a **pluggable Agent Gateway** for the daily digest pipeline. The gateway is responsible for cron scheduling, LLM summarisation, and Telegram delivery. It communicates with other services via HTTPS REST only.

**Default: [GoClaw](https://github.com/nextlevelbuilder/goclaw) v1.74+** — chosen for its low footprint, native OTLP, and Markdown-based config.

**Supported alternatives:**

| Framework | Language | Repo | Key Differentiator |
|-----------|----------|------|--------------------|
| [GoClaw](https://github.com/nextlevelbuilder/goclaw) (default) | Go | `nextlevelbuilder/goclaw` | ~35 MB RAM, Markdown config, native OTLP |
| [OpenClaw](https://github.com/openclaw/openclaw) | TypeScript | `openclaw/openclaw` | Largest ecosystem (354k★), multi-channel inbox |
| [PicoClaw](https://github.com/sipeed/picoclaw) | Go | `sipeed/picoclaw` | Ultra-lightweight (~10 MB RAM), hardware support |
| [nanoBot](https://github.com/HKUDS/nanobot) | Python | `HKUDS/nanobot` | Python SDK, ultra-light implementation |
| [ZeroClaw](https://github.com/zeroclaw-labs/zeroclaw) | Rust | `zeroclaw-labs/zeroclaw` | ~5 MB RAM, trait-based architecture, hardware peripherals |

| Required Capability | Usage in This Project |
|---------------------|----------------------|
| **Cron scheduling** | Daily digest trigger at 08:30 UTC |
| **HTTP / web_fetch** | Fetches news from CryptoPanic per watchlist project |
| **LLM provider** (Anthropic) | Summarises news via Claude 3.5 Haiku (≤ 200 tokens/project) |
| **Telegram channel** | Delivers digest messages (shares single bot with Go backend) |
| **Subagent / parallel** | Parallel news fetch across watchlist projects with auto-retry |
| **Scoped permissions** | Scopes digest agent API access to `/internal/` endpoints only |
| **OTLP / observability** | LLM call tracing → Prometheus/Grafana |
| **Web dashboard** | Agent management at `http://localhost:18790` |

The gateway runs as a **single Docker container** and shares Postgres and Redis with the backend but does **not** connect to NATS — all real-time event flow is Go-backend-owned.

See [Agent Gateway Abstraction](specs/001-investment-intel-poc/agent-gateway-abstraction.md) for the full interface contract, switching guide, and framework comparison.

---

## Documentation

| Document | Location | Description |
|----------|----------|-------------|
| Feature Specification | [`specs/001-investment-intel-poc/spec.md`](specs/001-investment-intel-poc/spec.md) | Requirements, user stories, acceptance criteria |
| Implementation Plan | [`specs/001-investment-intel-poc/plan.md`](specs/001-investment-intel-poc/plan.md) | Architecture, service matrix, research notes |
| Task List | [`specs/001-investment-intel-poc/tasks.md`](specs/001-investment-intel-poc/tasks.md) | 110 tasks across 11 phases |
| Agent Gateway Abstraction | [`specs/001-investment-intel-poc/agent-gateway-abstraction.md`](specs/001-investment-intel-poc/agent-gateway-abstraction.md) | Pluggable gateway interface contract, switching guide, framework comparison |
| Domain Decoupling ADR | [`specs/001-investment-intel-poc/architecture-domain-decoupling.md`](specs/001-investment-intel-poc/architecture-domain-decoupling.md) | Platform/domain separation, provider-agnostic interfaces, migration namespaces |
| REST API Contracts | `specs/001-investment-intel-poc/contracts/rest-api.md` | Endpoint signatures, error envelope |
| NATS Events | `specs/001-investment-intel-poc/contracts/nats-events.md` | Stream topology, message schemas |
| Constitution | [`.specify/memory/constitution.md`](.specify/memory/constitution.md) | Quality gates, development workflow |
| Design System | [`design-system/investment-intel/MASTER.md`](design-system/investment-intel/MASTER.md) | Design tokens and component guidelines |

---

## Contributing

1. Read the [constitution](.specify/memory/constitution.md) — it's non-negotiable
2. Follow the [Speckit](https://github.com/nextlevelbuilder/speckit) **spec → plan → tasks** workflow: no code without an approved spec
3. Use **TDD**: tests first, implementation second (Red-Green-Refactor)
4. Branch naming: `###-short-description` (issue number prefix)
5. Commit messages: [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`, etc.)
6. PR size: target < 400 changed lines; decompose larger changes
7. All CI gates must pass before merge

---

## License

This project is licensed under the [MIT License](LICENSE).

> **Third-party notice:** Agent gateway frameworks ([GoClaw](https://github.com/nextlevelbuilder/goclaw),
> [OpenClaw](https://github.com/openclaw/openclaw), [PicoClaw](https://github.com/sipeed/picoclaw),
> [nanoBot](https://github.com/HKUDS/nanobot), [ZeroClaw](https://github.com/zeroclaw-labs/zeroclaw))
> are **not** covered by this license — each is distributed under its own terms.
> See the [LICENSE](LICENSE) file for details.
