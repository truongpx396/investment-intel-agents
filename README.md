# Investment Intel AI Agents

**Real-time crypto signal alerts & AI-powered daily digests — delivered to Telegram.**

<!-- CI badges — uncomment once GitHub Actions workflows are created (Phase 1)
[![CI — Go](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-go.yml/badge.svg)](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-go.yml)
[![CI — Python](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-python.yml/badge.svg)](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-python.yml)
[![CI — Frontend](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-frontend.yml/badge.svg)](https://github.com/truongpx396/investment-intel-agents/actions/workflows/ci-frontend.yml)
-->

---

## Overview

Investment Intel AI Agents is a POC platform that lets users configure **rule-based signal strategies** for crypto assets (BTC & ETH for POC) and receive **real-time Telegram notifications** when conditions fire (< 60 s SLA). Each alert is enriched with an **LLM-generated anomaly explanation** describing the likely cause. Users also maintain a personal **watchlist** and get a daily **AI-summarised digest** of news and price movements via Telegram, plus periodic **market sentiment pulse** classifications. Strategies can optionally enable **trade execution** — paper trading (virtual balance, always available) or live trading via Binance Spot — with a **portfolio dashboard** tracking positions, P&L, and per-strategy performance. **Strategy templates** by investor profile (Conservative, Moderate, Aggressive, Income/DCA, Growth) lower onboarding friction for new users.

### Key Features

| Feature | Description |
|---------|-------------|
| 🎯 **Signal Strategies** | Create strategies with 6 signal types: price threshold, % price change, RSI, volume spike, MACD crossover, Bollinger Band breach |
| 📋 **Strategy Templates** | 5 curated templates by investor profile (Conservative, Moderate, Aggressive, Income/DCA, Growth) — lower onboarding friction |
| ⚡ **Real-Time Alerts** | Telegram notifications within 60 seconds of signal trigger via 30-second polling |
| 🧠 **Anomaly Explanation** | LLM-generated 1–2 sentence explanation of why each signal fired (news + market context), non-blocking |
| 📰 **Daily Digest** | AI-summarised news & price movements for watchlisted projects, delivered by 09:00 UTC |
| 📈 **Market Sentiment Pulse** | Configurable periodic sentiment classification (bullish/neutral/bearish) via LLM, with optional Telegram push |
| 💹 **Trade Execution** | Paper trading (virtual balance, always available) + live trading via Binance Spot with per-strategy budgets |
| 📊 **Portfolio Dashboard** | Positions, unrealised/realised P&L, trade history, and per-strategy performance across paper and live modes |
| 🔗 **Telegram Linking** | Self-service Telegram account connection — no admin needed |
| 🔔 **Alert History** | Full audit trail of all triggered alerts, filterable by strategy with delivery status badges |
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
                               │  Trade       │          │    Volume / %Chg │
                               │  Executor    │          │  • News + VADER  │
                               │  (Paper+Live)│          │  • LLM Provider  │
                               └──────────────┘          │    (abstraction) │
                                      │                  │  • Anomaly       │
                                      │                  │    Explanation   │
                               ┌──────────────┐          └──────────────────┘
                               │Agent Gateway │  Cron 08:30 UTC
                               │  (Digest     │ ──→ Fetch news
                               │   Agent)     │ ──→ LLM summarise
                               │  (Anomaly    │ ──→ Telegram digest
                               │   Agent)     │  Cron */4h
                               │  (Sentiment  │ ──→ Sentiment pulse
                               │   Agent)     │
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
| **Go Backend — Domain** | Signal evaluation, alert formatting, strategy CRUD, watchlist, market data adapters, trade execution (paper + live), portfolio tracking, exchange adapters, budget enforcement | Import provider SDKs directly, implement auth/billing logic |
| **Python ai-service — Platform** | Compute engine, enrichment pipeline, content provider interfaces, LLM provider abstraction (Anthropic/OpenAI/Gemini), Langfuse observability | Know about crypto indicators or news providers |
| **Python ai-service — Domain** | Technical indicators (pandas-ta), news fetching, sentiment scoring, anomaly explanation generation | Evaluate thresholds, send Telegram, touch NATS |
| **Agent Gateway — Framework** | Cron scheduling, LLM provider, Telegram channel, HTTP fetch | Connect to NATS, dispatch real-time alerts, contain domain logic |
| **Agent Gateway — Domain Skills** | Digest persona, crypto-digest skill, anomaly explanation, sentiment-pulse skill | Bypass framework tools, directly access databases |
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
| **Exchange** | Binance Spot (POC); abstracted behind `ExchangeAdapter` interface for post-POC expansion |
| **News** | CryptoPanic (primary), DuckDuckGo (fallback) |
| **LLM** | Provider-agnostic — Anthropic Claude 3.5 Haiku (default), OpenAI, Google Gemini; switchable via `LLM_PROVIDER` + `LLM_MODEL` env vars |
| **LLM Observability** | Langfuse (traces from ai-service Python SDK + Agent Gateway OTLP) |
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
│   │       ├── strategies/           # Strategy CRUD, SDF validation, import/export, templates
│   │       ├── signals/              # Signal evaluator (30s polling loop)
│   │       ├── alerts/               # Alert persistence, dispatcher, re-driver
│   │       ├── watchlist/            # Watchlist CRUD, digest content
│   │       ├── marketdata/           # Market data adapters (CoinGecko, CryptoCompare)
│   │       ├── trading/              # Trade executor (paper + live), budget manager
│   │       ├── portfolio/            # Portfolio positions, P&L tracking
│   │       ├── exchange/             # Exchange adapter interface + Binance impl
│   │       │   └── binance/          # Binance Spot adapter (POC)
│   │       ├── sentiment/            # Sentiment score CRUD
│   │       └── config/               # Seed config loader (signal types, projects, templates)
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
│   │   │   ├── content/              # ContentProvider ABC, ContentItem dataclass
│   │   │   └── llm/                  # LLM provider abstraction + Langfuse observability
│   │   │       ├── interfaces.py     # LLMProvider ABC, LLMResponse, LLMConfig
│   │   │       ├── factory.py        # Provider factory (anthropic/openai/gemini)
│   │   │       ├── observability.py   # Langfuse @observe decorator
│   │   │       ├── anthropic/        # Anthropic Claude provider
│   │   │       ├── openai/           # OpenAI / Azure OpenAI provider
│   │   │       └── gemini/           # Google Gemini provider
│   │   └── domain/investment/        # Investment-specific
│   │       ├── indicators/           # RSI, MACD, Bollinger, volume spike, pct_change
│   │       ├── news/                 # CryptoPanic + DuckDuckGo adapters
│   │       ├── enrichment/           # Anomaly explanation (POST /explain/{asset})
│   │       └── projects/             # DB-backed project registry
│   └── tests/
│
├── agent-gateway/                     # Pluggable agent gateway (see agent-gateway-abstraction.md)
│   ├── goclaw/                       # GoClaw config (default)
│   │   └── agents/
│   │       ├── digest-agent/         # Daily digest agent + crypto-digest skill
│   │       ├── anomaly-agent/        # Anomaly explanation agent
│   │       └── sentiment-agent/      # Market sentiment pulse agent
│   ├── openclaw/                     # OpenClaw config
│   ├── picoclaw/                     # PicoClaw config
│   ├── nanobot/                      # nanoBot config
│   └── zeroclaw/                     # ZeroClaw config
│
├── frontend/                         # React SPA
│   ├── src/
│   │   ├── platform/                 # Domain-agnostic (auth, billing, settings, admin)
│   │   │   ├── components/           # Design system primitives + Storybook stories
│   │   │   └── pages/                # Auth, billing, settings, admin pages
│   │   ├── domain/investment/        # Investment-specific UI
│   │   │   ├── pages/                # Dashboard, strategies, alerts, watchlist, portfolio
│   │   │   ├── components/           # Template picker, sentiment badge, trade history
│   │   │   └── services/             # TanStack Query hooks
│   │   └── services/                 # Shared TanStack Query hooks
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
│       ├── tasks.md                  # Task list (175 tasks, 14 phases)
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
| Langfuse (LLM Observability) | `http://localhost:3100` |
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
| `LLM_PROVIDER` | LLM provider for ai-service (default: `anthropic`; options: `openai`, `gemini`) |
| `LLM_MODEL` | LLM model name (default: `claude-3-5-haiku-latest`) |
| `LLM_API_KEY` | API key for the configured LLM provider |
| `AGENT_GATEWAY_LLM_API_KEY` | Anthropic API key for Agent Gateway LLM |
| `EXCHANGE_ENCRYPTION_KEY` | AES-256-GCM encryption key for exchange credentials (exactly 32 bytes) |
| `PAPER_TRADING_DEFAULT_BALANCE` | Default paper trading virtual balance (default: 100000) |
| `SENTIMENT_PULSE_CRON` | Cron expression for sentiment pulse (default: `0 */4 * * *`) |
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
  ],
  "execution": {
    "enabled": true,
    "mode": "paper",
    "action": "buy",
    "trade_size_pct": 10,
    "budget_allocation": 5000
  }
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

The project is planned across **14 phases** with **175 tasks**:

| Phase | Focus | Key Deliverables |
|-------|-------|------------------|
| 1 — Setup | Repo, CI, design system, seed config, SDF schema | Monorepo, GitHub Actions, Storybook, `seed.yaml`, SDF JSON Schema |
| 1b — LLM + Langfuse | LLM provider abstraction & observability | `LLMProvider` interface (Anthropic/OpenAI/Gemini), factory, Langfuse tracing |
| 2 — Foundation | Auth, Supabase, NATS, Stripe bootstrap | User registration, JWT middleware, NATS streams |
| 3 — Strategy Config (P1) | Signal strategy CRUD, SDF validation, import/export, templates | Strategy creation, 6 signal types, import/export, 5 investor-profile templates |
| 4 — Telegram Linking (P1) | Self-service Telegram account connection | Deep-link flow, broken-link detection |
| 5 — Notifications (P2) | Real-time signal alerts via Telegram | 30s polling, NATS fanout, < 60s delivery SLA |
| 6 — Alert History (P2) | Read-only alert audit trail | Paginated history, strategy filter, status badges |
| 7 — Watchlist & Digest (P3) | Watchlist + Agent Gateway daily digest | CryptoPanic integration, LLM summarisation, 08:30 UTC cron |
| 7a — Anomaly + Sentiment | Anomaly explanation (US6) + Market sentiment (US7) | LLM-enriched alerts, 4h sentiment pulse, sentiment history |
| 7b — Trade Execution + Portfolio | Trade execution (US8) + Portfolio management (US9) | Paper/live trading, Binance adapter, portfolio dashboard, budget enforcement |
| 8 — Billing | Stripe subscription management | Checkout, webhooks, subscription gating, idempotent webhooks |
| 9 — Admin | User management panel | Admin middleware, user list, subscription overrides |
| 10 — Quality Gates | Observability, performance, CI gates | Prometheus, Grafana, Langfuse, Playwright E2E, Lighthouse, k6 |
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
| Anomaly explanation latency | ≤ 10 s (non-blocking — alert delivers without explanation on timeout) |
| Sentiment pulse duration | ≤ 3 min p95 |
| Scalability threshold | 500 active strategies × 10 signal assets (documented in research.md) |

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
| **LLM provider abstraction** | `LLMProvider` interface with Anthropic/OpenAI/Gemini implementations — switch via env vars, zero code changes |
| **Langfuse for LLM observability** | Unified tracing from ai-service (Python SDK) and Agent Gateway (OTLP) — prompt logs, token costs, latency |
| **Paper trading by default** | Every user starts with virtual balance; live trading requires explicit exchange connection + confirmation |
| **Strategy templates** | Seed-config-driven, asset-agnostic SDF templates by investor profile — no DB storage, read-only |
| **Exchange adapter pattern** | `ExchangeAdapter` interface with Binance Spot impl; future DEX support via `Capabilities()` branching |
| **AES-256-GCM credential encryption** | Exchange API keys encrypted at rest; decrypted only in-memory at point of use |
| **Speckit for SDD** | AI-assisted spec workflow ensures spec → plan → tasks → code traceability; constitution enforces quality gates |
| **Agent Gateway over bespoke orchestration** | Single binary/container with cron, LLM, Telegram, and agent tools built in — no custom Python scheduler needed |

---

## Architecture Strengths

A summary of the properties that make this system well-structured, maintainable, and extensible — even at POC scale.

### 🔌 Swap Anything Without Touching Business Logic

The architecture is built on **two layers of abstraction** that compound:

1. **Platform / Domain split** — cross-cutting concerns (auth, billing, notification, event bus) live in `platform/`; all investment logic lives in `domain/investment/`. The platform never imports domain code. Swapping the entire domain (e.g., to e-commerce price monitoring) requires zero changes to auth, billing, or agent code.

2. **Provider abstraction within platform** — each platform concern defines a provider-agnostic interface (`AuthProvider`, `BillingProvider`, `Sender`, `Publisher`) with a swappable implementation directory underneath. Changing Supabase → Casdoor, Stripe → LemonSqueezy, Telegram → Discord, or NATS → Kafka means adding one new directory and changing one wiring line in `main.go` — all middleware, handlers, and domain code remain untouched.

This means a provider swap and a domain swap are **independent operations**, composable without coordination.

### 🧩 Right Language for Each Job

Rather than forcing one language to do everything, each service plays to its language's strengths:

| Concern | Language | Why |
|---------|----------|-----|
| Latency-critical 30 s signal evaluation, low-memory poller, type-safe REST API | **Go** | Goroutines, sub-ms GC, compiled binary, strong typing |
| Technical indicators (RSI, MACD, Bollinger), data resampling, sentiment scoring | **Python** | pandas-ta computes any indicator in one line; VADER for CPU-only NLP; fastest path to correct math |
| LLM orchestration, cron scheduling, multi-tool agents | **Agent Gateway** (pluggable) | Purpose-built for agent workflows; no hand-rolled scheduler or LLM piping |
| Responsive UI with real-time state management | **React + TanStack** | TanStack Query handles cache/polling; TanStack Router for type-safe routing |

Go does **not** compute indicators or call LLMs. Python does **not** evaluate threshold rules or touch NATS. The Agent Gateway does **not** dispatch real-time alerts. Each service has a clear "MUST NOT" list that prevents responsibility creep.

### 🛡️ Defence in Depth — Not Just Perimeter Security

Security is layered across every boundary, not concentrated at a single gateway:

- **Network isolation**: two Docker networks (`public` / `internal`); ai-service is never exposed to the internet
- **Auth at every hop**: Supabase JWT for user-facing API; static bearer token for internal service-to-service; per-agent scoped grants in Agent Gateway
- **Encryption at rest**: exchange credentials AES-256-GCM encrypted; decrypted only in-memory at point of use
- **Rate limiting at multiple layers**: Traefik (global), per-user exchange rate limiter (Redis sliding window), signal cooldown (prevents alert/trade spam)
- **Race condition handling**: advisory locks (`pg_advisory_xact_lock`) + per-user mutex for exchange disconnect during in-flight trades
- **Secrets never in code**: `.env` + `.env.example` with placeholders; `.gitignore` enforced; production roadmap to HashiCorp Vault

### ⚡ Non-Blocking by Design

Every expensive or failure-prone operation is designed to degrade gracefully without blocking the critical path:

- **Anomaly explanation** (LLM call) fails → alert still delivers within the 60 s SLA; explanation simply missing
- **Exchange API** times out → alert still delivers; trade logged as `Failed` with reason
- **ai-service** is down → circuit breaker (`sony/gobreaker`) opens after 5 failures; price-threshold + CoinGecko 24h signals still evaluate; indicator-based signals skip gracefully
- **Telegram delivery** fails → 3× exponential retry; if user unlinked → `NakWithDelay` re-drive for 24 h (no re-publish, no stream bloat); auto-expire via NATS dead letter
- **Trade execution** runs in a goroutine; budget validation and exchange calls never block alert dispatch

The pattern: **always deliver the core value (the alert), enrich it when possible, log what failed, retry later**.

### 📐 Externalised Configuration — Code Ships Shape, Not Data

Business rules live in declarative config, not application code:

- **`config/seed.yaml`** — signal type definitions (parameters, allowed values, defaults), project seeds (BTC, ETH, 18 watchlist tokens), system tunables (poll interval, cooldown, cache TTL), and strategy templates (5 investor profiles). Validated against JSON Schema at boot; invalid config = hard startup failure.
- **`contracts/strategy-definition.schema.json`** (SDF) — the portable format for strategies. Machine-validatable, LLM-friendly (a future LLM can generate valid SDF from natural language). Adding a new signal type = extend the schema's `oneOf` array + add a seed entry; no Go/Python code changes.
- **Environment variables** — all secrets, provider selection (`LLM_PROVIDER`, `LLM_MODEL`), and operational tunables (`SIGNAL_COOLDOWN_SECONDS`, `SENTIMENT_PULSE_CRON`).

Consequence: adding a new watchlist token is a database insert. Tuning RSI thresholds is a YAML edit. Switching the LLM from Claude to GPT-4o is an env var change. None of these require a code deploy.

### 📊 Observability as a First-Class Citizen

Every service emits structured data for monitoring, not just logs:

- **Prometheus metrics** — `signal_evaluations_total`, `alerts_dispatched_total`, `trade_execution_total{mode,status}`, `exchange_circuit_breaker_state`, `anomaly_explanation_failures_total`, `cryptopanic_quota_remaining`, and 13+ alert rules
- **Langfuse LLM tracing** — unified view of all LLM calls from both ai-service (Python SDK) and Agent Gateway (OTLP); prompt/completion logs, per-call token cost, latency histograms
- **Structured JSON logging** — zerolog (Go), structlog (Python), pino (Node.js); machine-parseable from day one
- **Grafana dashboards** — API latency (p50/p95/p99), NATS throughput, LLM call cost, signal cycle time
- **NATS consumer health** — `prometheus-nats-exporter` tracks pending messages, redelivery counts per consumer group

This isn't monitoring bolted on at the end — the task list includes observability tasks in Phase 1 (CI), Phase 10 (Prometheus/Grafana), and per-feature (every dispatcher/executor emits counters).

### 🔄 Spec-Driven Development — Traceability from Idea to Test

Every line of code traces back to a requirement:

```
User request → spec.md (user stories + acceptance criteria)
            → plan.md (architecture + service matrix + research)
            → tasks.md (175 tasks, phased, with dependencies)
            → code + tests (TDD: test first, implement second)
```

The [constitution](.specify/memory/constitution.md) enforces quality gates at each stage. Cross-artifact consistency is checked via `/speckit.analyze`. Requirements traceability is maintained via `/speckit.checklist`. The result: no orphan code (every function traces to a task), no orphan tasks (every task traces to a user story), no orphan stories (every story traces to a clarified decision).

---

## Powered By

### Speckit — Spec-Driven Development

[Speckit](https://github.com/nextlevelbuilder/speckit) (`v0.3.1`) powers the entire development workflow for this project. It's an AI-assisted Spec-Driven Development (SDD) toolkit that integrates with GitHub Copilot to guide the process from idea → specification → plan → tasks → implementation.

The workflow stages (each backed by a Copilot agent under `.github/agents/`):

```
/speckit.clarify  →  /speckit.specify  →  /speckit.plan  →  /speckit.tasks  →  /speckit.implement
     ↓                    ↓                    ↓                  ↓                    ↓
  Resolve            spec.md              plan.md            tasks.md            Code + tests
  ambiguity        (requirements)       (architecture)     (175 tasks)         (TDD workflow)
```

Additional commands: `/speckit.analyze` (cross-artifact consistency checks), `/speckit.checklist` (requirements traceability), `/speckit.constitution` (project governance).

All spec artifacts live in `specs/001-investment-intel-poc/`; project memory and templates in `.specify/`.

### Agent Gateway — Pluggable AI Agent Framework

The project uses a **pluggable Agent Gateway** for the AI agent layer. The gateway runs three domain-specific agents: a **daily digest agent** (news summarisation), an **anomaly explanation agent** (enriches fired alerts with LLM-generated cause analysis), and a **market sentiment pulse agent** (periodic sentiment classification). It communicates with other services via HTTPS REST only.

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
| **Cron scheduling** | Daily digest at 08:30 UTC; sentiment pulse every 4 h (configurable) |
| **HTTP / web_fetch** | Fetches news from CryptoPanic per watchlist project; market context for anomaly explanation |
| **LLM provider** (configurable) | Summarises news, generates anomaly explanations, classifies market sentiment (≤ 200 tokens/project) |
| **Telegram channel** | Delivers digest + sentiment push messages (shares single bot with Go backend) |
| **Subagent / parallel** | Parallel news fetch across watchlist projects with auto-retry |
| **Scoped permissions** | Per-agent grants: digest → `/internal/watchlist`, anomaly → `/internal/alerts/:id/explanation`, sentiment → `/internal/sentiment` |
| **OTLP / observability** | LLM call tracing → Langfuse + Prometheus/Grafana |
| **Web dashboard** | Agent management at `http://localhost:18790` |

The gateway runs as a **single Docker container** and shares Postgres and Redis with the backend but does **not** connect to NATS — all real-time event flow is Go-backend-owned.

See [Agent Gateway Abstraction](specs/001-investment-intel-poc/agent-gateway-abstraction.md) for the full interface contract, switching guide, and framework comparison.

---

## Documentation

| Document | Location | Description |
|----------|----------|-------------|
| Feature Specification | [`specs/001-investment-intel-poc/spec.md`](specs/001-investment-intel-poc/spec.md) | Requirements, user stories, acceptance criteria |
| Implementation Plan | [`specs/001-investment-intel-poc/plan.md`](specs/001-investment-intel-poc/plan.md) | Architecture, service matrix, research notes |
| Task List | [`specs/001-investment-intel-poc/tasks.md`](specs/001-investment-intel-poc/tasks.md) | 175 tasks across 14 phases |
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
