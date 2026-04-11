# Implementation Plan: Investment Intel AI Agents — POC

**Branch**: `001-investment-intel-poc` | **Date**: 2026-03-21 | **Spec**: [spec.md](./spec.md)  
**Input**: Feature specification from `/specs/001-investment-intel-poc/spec.md`

## Summary

Build a POC platform where users self-register, configure rule-based signal strategies for assets
in the curated project list (BTC and ETH seeded for POC; additional tokens require only a data insert),
and receive real-time Telegram notifications when conditions fire (< 60 s SLA via 30 s polling).
Users maintain a watchlist of crypto projects and receive a daily AI-summarised Telegram digest
(LLM summarises raw news from a third-party crypto news API) by 09:00 in their configured timezone.
A React web app hosts all configuration UX; alert history is visible per strategy; billing and
subscription management run through Stripe.

The AI agent layer (Agent Gateway) orchestrates the digest pipeline: a scheduled cron agent polls
the news API, invokes the LLM summarisation tool, and dispatches the Telegram digest — replacing
bespoke Python orchestration with the Agent Gateway's built-in cron, HTTP fetch, and message tools.

---

## Technical Context

**Language/Version**:
- Backend API: Go 1.24+ (Golang)
- AI/Data services: Python 3.12+
- Agent Gateway: varies by implementation (GoClaw: Go 1.26+, OpenClaw: Node 24+, PicoClaw: Go 1.25+, nanoBot: Python 3.11+, ZeroClaw: Rust stable)
- Frontend: Node.js 22+ (React 19)

**Primary Dependencies**:
- **API Gateway**: Traefik v3 (routing, TLS termination, rate limiting)
- **Auth / User management**: Supabase (PostgreSQL + Auth + Row Level Security)
- **Billing**: Stripe SDK (Go + React)
- **AI Agent gateway**: Pluggable — GoClaw v1.74+ (default), OpenClaw, PicoClaw, nanoBot, or ZeroClaw (see [Agent Gateway Abstraction](agent-gateway-abstraction.md)); all provide cron, LLM, Telegram channel, and web dashboard; dashboard at `http://localhost:18790`
- **Message queue**: NATS JetStream (signal event fanout, notification queue)
- **Frontend**: ReactJS 19 + TailwindCSS v4 + TanStack Router + TanStack Query
- **Component development & docs**: Storybook 8 (`@storybook/react-vite`) — isolated component development, visual regression baseline, and living design-system documentation
- **LLM (digest summarisation)**: Anthropic Claude (via Agent Gateway provider config, claude-3-5-haiku)
- **Crypto news API**: CryptoPanic free tier (50 req/day per token; DuckDuckGo fallback on quota
  exhaustion). Abstracted behind `NewsProvider` interface in `ai-service/src/news/`.
- **Market data feed**: CoinGecko Free API (no key required, ≤30 req/min) for spot price and
  percentage-change signals; CryptoCompare `histominute` endpoint (free tier, 100k req/month)
  for intraday OHLCV data forwarded to Python ai-service for indicator computation. Go adapter
  abstracted behind `pkg/marketdata.Provider`; RSI and OHLCV math happens in Python. All price
  data is denominated in the asset's `quote_currency` from `app_projects` (default `USD`); the
  adapter passes this value as CoinGecko `vs_currencies` and CryptoCompare `tsym` parameters.
- **Circuit breaker**: `sony/gobreaker` v2 — protects Go evaluator's HTTP calls to ai-service;
  opens after 5 consecutive failures; half-open probe after 30 s; prevents cascading timeouts
  when ai-service is degraded
- **Technical indicators**: `pandas-ta` (RSI, MACD, Bollinger) + `numpy` — owned entirely by
  Python ai-service; Go signal evaluator calls `POST /indicators/{asset}` to get pre-computed values
- **News sentiment**: `vaderSentiment` (POC) — fast, CPU-only; scores news headlines before Agent Gateway
  LLM step; reduces token waste on low-signal or duplicate items
- **Logging**: zerolog (Go), structlog (Python)
- **Monitoring**: Prometheus + Grafana

**Storage**:
- PostgreSQL 18 with pgvector (primary store — users, strategies, signal rules, alerts, watchlists,
  digests; also used by the Agent Gateway for its multi-tenant store)
- Redis 7 (API response cache, rate-limit counters, session tokens, signal evaluation state)

**Testing**:
- Go: `go test` + `testify` + `httptest` for API contract tests
- Python: `pytest` + `pytest-asyncio`
- Frontend: Vitest + React Testing Library + Playwright (E2E) + Storybook (component stories + `@storybook/test` interaction tests)
- CI: GitHub Actions

**Logging**: All services output JSON-structured logs to stdout via zerolog (Go), structlog (Python), and pino (Node.js). Log rotation is handled by the Docker daemon log driver (`json-file` with `max-size: 50m`, `max-file: 3`); no application-level rotation is needed.

**Target Platform**: DigitalOcean Droplets (Linux/amd64) + Cloudflare DNS/CDN/WAF

**Project Type**: Web service (React SPA + Go microservices + Python AI service + pluggable Agent Gateway)

**Performance Goals**:
- Signal-triggered Telegram notification: < 60 s end-to-end (30 s poll cycle + < 30 s delivery)
- REST API endpoints: < 200 ms p95
- React SPA: LCP ≤ 2.5 s, CLS ≤ 0.1 (Cloudflare CDN-served static assets)
- Daily digest delivery: before 09:00 UTC, 100% of days during POC (single-timezone cron at 08:30 UTC; per-user timezone scheduling is post-POC)

**Constraints**:
- POC scale: ≤ 50 registered users — no horizontal scaling required
- Agent Gateway container (GoClaw default: ~25 MB, ~35 MB RAM idle) fits on smallest Droplet alongside other services
- Market data polling at 30 s intervals must stay within free-tier or low-cost API rate limits
- Stripe integration limited to subscription creation and webhook processing (no custom invoicing)

**Scale/Scope**: ≤ 50 users, 2 signal assets seeded for POC (BTC/ETH; extensible via `app_projects` — no schema migration needed to add tokens), curated watchlist of ~20 projects

---

## Constitution Check

*GATE: Must pass before proceeding. Re-checked after Phase 1 design.*

### Principle I — Code Quality

| Gate | Status | Notes |
|------|--------|-------|
| Linter configured (golangci-lint, ruff, ESLint) | ✅ PASS | Must be wired into CI before first PR |
| Single-responsibility functions; cyclomatic complexity ≤ 10 | ✅ PASS | Signal evaluator and digest pipeline are the highest-risk; each step must be its own function |
| Public APIs documented (Go godoc, Python docstrings, JSDoc) | ✅ PASS | All REST handlers and service interfaces require docs |
| Dependencies pinned (go.sum, uv.lock, package-lock.json) | ✅ PASS | Lock files committed; no floating ranges |

### Principle II — Testing Standards

| Gate | Status | Notes |
|------|--------|-------|
| TDD: tests written before implementation | ✅ PASS | Red-Green-Refactor enforced per task phase |
| Unit coverage ≥ 80%; critical paths ≥ 95% | ✅ PASS | Auth, payment webhooks, signal evaluation, alert dispatch = critical paths |
| Contract tests for API changes | ✅ PASS | All REST contracts in `tests/contract/` must precede handler implementation |
| Integration tests for NATS, Supabase, Agent Gateway, Stripe webhook | ✅ PASS | See Phase 2 foundational tasks |

### Principle III — User Experience Consistency

| Gate | Status | Notes |
|------|--------|-------|
| Design system / component library established before UI work | ✅ PASS | TailwindCSS config + component primitives defined in Phase 1 setup |
| Loading, empty, and error states required for every view | ✅ PASS | Mandatory for strategy list, alert history, watchlist, digest status |
| WCAG 2.1 AA keyboard/screen-reader validation | ✅ PASS | Validated via Playwright accessibility checks in CI |

### Principle IV — Performance Requirements

| Gate | Status | Notes |
|------|--------|-------|
| p95 latency budget defined: REST API < 200 ms | ✅ PASS | Defined in Technical Context above |
| Notification SLA defined: < 60 s | ✅ PASS | Defined in spec FR-006; 30 s poll cycle satisfies with headroom |
| LCP ≤ 2.5 s, CLS ≤ 0.1 | ✅ PASS | Cloudflare CDN + code-split React bundles |
| Background job SLA: digest before 09:00 UTC | ✅ PASS | Agent Gateway cron set to 08:30 UTC (delivers before cutoff); per-user timezone scheduling deferred to post-POC |

**Constitution verdict**: ✅ ALL GATES PASS — no violations to justify.

---

## Service Responsibility Matrix

A single authoritative reference for which service owns each concern. When in doubt, consult this table before placing code.

### Ownership by Concern

| Concern | Owner | Rationale |
|---|---|---|
| REST API (auth, strategies, alerts, watchlist, billing, admin) | **Go backend** | Strongly-typed, high-throughput HTTP; all user-facing CRUD and business logic |
| JWT validation & user-context injection | **Go backend** | Supabase public-key verification at the edge of the Go service; no other service touches auth tokens |
| Signal evaluation loop (30 s ticker, price threshold, % change, RSI trigger check) | **Go backend** | Orchestrates the tick; fetches pre-computed indicator values from ai-service; evaluates threshold rules; publishes to NATS — latency-sensitive hot path |
| Market data fetching (CoinGecko spot price, CryptoCompare OHLCV) | **Go backend** (`pkg/marketdata/`) | Raw data fetch stays in Go; OHLCV slices are forwarded to ai-service for indicator computation |
| **Technical indicator computation (RSI, volume SMA, MACD, Bollinger Bands, OHLCV stats, intraday % change)** | **Python ai-service** | `pandas-ta` / `numpy` vectorised math; zero-effort 14-period RSI, 20-period volume SMA ratio, MACD(12,26,9), Bollinger Bands(20,2) vs hand-rolled Go; accepts `candle_minutes` parameter to resample 1-min candles into the user-selected candle size (5/15/60 min) before computing indicators; result cached in Redis at 25 s TTL keyed by `{asset}:{candle_minutes}`; Go evaluator calls `POST /indicators/{asset}` (indicators) and `POST /pct_change/{asset}` (intraday % change — separate cache key by `window_minutes`) per tick |
| **News fetching & `NewsProvider` interface** | **Python ai-service** | CryptoPanic + DuckDuckGo fallback; async `httpx`; quota tracking; `GET /news/{slug}` |
| **News sentiment scoring & deduplication** | **Python ai-service** | `vaderSentiment` for headline sentiment; deduplication before Agent Gateway summarisation; `POST /enrich/news` |
| **Curated project registry** | **Python ai-service** | **DB-backed**: reads from `app_projects` table (ai-service connects to shared Postgres in read-only mode); cached in-memory with 5 min TTL; `GET /projects` returns `{slug, display_name, symbol, coingecko_id, quote_currency, is_signal_asset}`; single source of truth for both React watchlist/strategy form and Agent Gateway skill; `quote_currency` (default `USD`) is consumed by Go market data adapters; adding a new project is a data-only change (visible within 5 min, no code deploy) |
| NATS publish/subscribe (signal events, alert queue, DLQ) | **Go backend** | All real-time event flow is Go-owned; the Agent Gateway and ai-service do NOT touch NATS |
| Alert persistence & Telegram delivery (real-time) | **Go backend** | Direct Telegram Bot API call for < 60 s latency; the Agent Gateway's scheduler is too coarse for real-time |
| Telegram account linking (deep-link, webhook, bot token) | **Go backend** | Stateful linking flow requires Postgres writes; linked to auth middleware |
| Stripe checkout, webhook processing, subscription gating | **Go backend** | Financial operations require transaction safety and webhook signature validation |
| PostgreSQL migrations | **Go backend** (`migrations/`, `golang-migrate`) | Single migration runner owns schema; Agent Gateway uses `gc_` prefix namespace only |
| Daily digest orchestration (cron, fetch, summarise, send) | **Agent Gateway** | Built-in cron, HTTP fetch, LLM provider, and message tools; calls ai-service `GET /news/{slug}` and `POST /enrich/news` |
| LLM summarisation (claude-3-5-haiku) | **Agent Gateway** | LLM provider configured natively in the gateway; enriched + deduplicated news items from ai-service are the input |
| Telegram digest delivery | **Agent Gateway** | Agent Gateway Telegram channel; same bot as Go alert dispatcher (single bot, see §Single Telegram Bot below) |
| Per-user agent context & watchlist retrieval | **Agent Gateway → Go `/internal/` API** | Agent Gateway skill calls Go's `/internal/watchlist` using service token (T057a) |
| UI (all configuration, history, settings, billing pages) | **React SPA** | Single-page app via Cloudflare CDN; all data via TanStack Query → Go REST API |
| Design system primitives & component stories | **React SPA** (Storybook) | Co-located `*.stories.tsx`; visual baseline and living docs |
| Routing, TLS termination, rate limiting | **Traefik** | Handles `api.`, `app.`, `agent.` subdomains; services never terminate TLS directly |
| Metrics exposure & tracing | **Go backend** + **Agent Gateway** (OTLP) + **ai-service** (`/metrics`) | All three expose Prometheus metrics; Agent Gateway emits OTLP traces |

### Cross-Service Communication Rules

| From | To | Protocol | Auth |
|---|---|---|---|
| React SPA | Go backend | HTTPS REST (`/api/v1/`) | Supabase JWT (httpOnly cookie) |
| Agent Gateway digest agent | Go backend | HTTPS REST (`/internal/`) | Static bearer token (`AGENT_GATEWAY_INTERNAL_TOKEN`) |
| Agent Gateway digest agent | Python ai-service | HTTP fetch via `GET /news/{slug}`, `POST /enrich/news`, `GET /projects` | None (internal network only) |
| Go backend (signal evaluator) | Python ai-service | HTTP `POST /indicators/{asset}`, `POST /pct_change/{asset}` | None (internal network only) |
| Go backend | NATS JetStream | NATS protocol | NATS credentials |
| Go backend | Telegram Bot API | HTTPS | `TELEGRAM_BOT_TOKEN` (shared single bot) |
| Agent Gateway | Telegram (digest) | Agent Gateway Telegram channel | Same `TELEGRAM_BOT_TOKEN` configured as `AGENT_GATEWAY_TELEGRAM_BOT_TOKEN` |
| Go backend | Supabase (Auth + DB) | HTTPS + postgres:// | Supabase service role key + RLS |
| Go backend | Stripe | HTTPS | Stripe secret key + webhook signing secret |
| Go backend | CoinGecko / CryptoCompare | HTTPS | No key / free-tier key |
| Python ai-service | CryptoPanic | HTTPS | API token |
| Python ai-service | Redis | redis:// | Password (internal) |

### Standardised Error Response Contract

All Go backend REST API error responses MUST use a consistent JSON envelope. The schema MUST be documented in `contracts/rest-api.md` (T010a) and enforced in all contract tests:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Human-readable description of the error",
    "details": [
      { "field": "window_minutes", "reason": "value 999 is not in allowed set {5, 15, 60, 240, 1440}" }
    ]
  }
}
```

- `code`: Machine-readable error code (e.g., `VALIDATION_FAILED`, `NOT_FOUND`, `UNAUTHORIZED`, `FORBIDDEN`, `RATE_LIMITED`, `INTERNAL_ERROR`, `PAYMENT_REQUIRED`)
- `message`: Human-readable, actionable (per constitution Principle III — no raw stack traces)
- `details`: Optional array of field-level errors for validation failures; omitted for non-validation errors

Python ai-service internal endpoints use the same envelope for consistency, though they are not user-facing.

### What Each Service MUST NOT Do

| Service | Must NOT |
|---|---|
| **Go backend** | Call LLM APIs directly; run digest orchestration; compute technical indicators (delegate to ai-service); write to `gc_` schema tables |
| **Python ai-service** | Evaluate signal threshold rules (that logic stays in Go); touch NATS; send Telegram messages; own any user-facing Postgres tables |
| **Agent Gateway** | Connect to NATS; perform real-time alert dispatch; own Postgres migrations; compute indicators directly |
| **React SPA** | Call market data or news APIs directly; store secrets; implement business logic |
| **Traefik** | Contain application logic; be aware of NATS or Agent Gateway internals |

---

## Agent Gateway Architecture

The Agent Gateway is a **pluggable** component — any framework that satisfies the [Agent Gateway Abstraction](agent-gateway-abstraction.md) contract can be used. The default is GoClaw (`github.com/nextlevelbuilder/goclaw`), which replaces bespoke Python orchestration for the AI agent layer. Supported alternatives: OpenClaw, PicoClaw, nanoBot, ZeroClaw.

Key capabilities required from the gateway:

| Required Capability | POC Usage |
|---------------------|-----------|
| **Cron scheduling** | Daily digest trigger at 08:30 UTC |
| **HTTP fetch tool** | Fetches news items from crypto news API per watchlist project |
| **LLM provider (Anthropic)** | Summarises raw news items via claude-3-5-haiku |
| **Telegram channel** | Dispatches digest messages and (future) alert escalation |
| **Shared PostgreSQL** | Gateway shares the same Postgres instance; per-user agent contexts |
| **Message tool** | Sends formatted digest to each user's Telegram chat |
| **Subagent / parallel execution** | Parallel HTTP fetch per watchlist project with automatic retry |
| **Scoped permissions** | Scope digest agent's backend API access to specific `/internal/` endpoints |
| **Observability (OTLP)** | LLM call tracing feeds Prometheus/Grafana |

**Deployment**: The Agent Gateway runs as a **single Docker container** (dashboard at `http://localhost:18790`) alongside backend services, connected to the shared Postgres and Redis instances. NATS is not used by the gateway directly — the Go signal service publishes to NATS; the gateway operates on its own scheduler for digest tasks.

### Single Telegram Bot Architecture

The system uses **one Telegram bot** (`TELEGRAM_BOT_TOKEN`) shared between Go backend and the Agent Gateway:

| Responsibility | Owner | Mechanism |
|---|---|---|
| Account linking (`/start?token=`) | **Go backend** | Telegram webhook registered at `https://api.<domain>/telegram/webhook`; Go processes `/start` deep-link updates and linking confirmation messages |
| Real-time alert delivery | **Go backend** | Direct `sendMessage` API call via bot token; latency-critical path (< 60 s SLA) |
| Daily digest delivery | **Agent Gateway** | Agent Gateway Telegram channel configured with the same bot token (`AGENT_GATEWAY_TELEGRAM_BOT_TOKEN = TELEGRAM_BOT_TOKEN`); uses message tool to call `sendMessage` |

**Webhook routing**: Telegram delivers all bot updates (messages, commands) to the single registered webhook URL owned by Go backend. The Agent Gateway never receives inbound updates — it only **sends** outbound messages via `sendMessage`. This avoids the Telegram limitation of one webhook per bot. The Go backend MUST filter webhook updates to only process `/start` commands for account linking; all other inbound messages are ignored (no command router needed for POC).

**Configuration**: Both services read the same bot token from the environment. In `docker-compose.yml`, set `TELEGRAM_BOT_TOKEN` once and reference it in both Go backend and Agent Gateway service definitions (`AGENT_GATEWAY_TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}`).

---

## Project Structure

### Documentation (this feature)

```text
specs/001-investment-intel-poc/
├── plan.md              ← this file
├── research.md          ← Agent Gateway integration patterns, market data feed evaluation
├── quickstart.md        ← local dev setup
├── contracts/
│   ├── rest-api.md      ← REST endpoint contracts (auth, strategies, alerts, watchlist, digest)
│   └── nats-events.md   ← NATS message schemas (signal.triggered.{asset}, alert.pending)
└── tasks.md             ← implementation task list
```

### Source Code (repository root)

```text
investment-intel/
│
├── backend/                         # Go API service
│   ├── cmd/server/main.go
│   ├── internal/
│   │   ├── auth/                    # JWT validation, Supabase integration
│   │   ├── strategies/              # Strategy CRUD, signal rule validation
│   │   ├── signals/                 # Signal evaluator (30 s polling loop)
│   │   ├── alerts/                  # Alert persistence, history queries
│   │   ├── watchlist/               # Watchlist CRUD
│   │   ├── telegram/                # Telegram account linking
│   │   └── billing/                 # Stripe subscription + webhook handler
│   ├── pkg/
│   │   ├── marketdata/              # Market data feed adapter (interface + impl)
│   │   └── nats/                    # NATS publisher/subscriber helpers
│   └── tests/
│       ├── contract/                # HTTP contract tests (per endpoint)
│       ├── integration/             # Supabase, NATS, Stripe webhook integration tests
│       └── unit/                    # Pure unit tests
│
├── ai-service/                      # Python AI/data service
│   ├── src/
│   │   ├── main.py                  # FastAPI app factory; mounts routers; configures logging
│   │   ├── config.py                # Settings via pydantic-settings (env vars, .env)
│   │   ├── health.py                # GET /health (liveness) + GET /health/ready (readiness)
│   │   ├── news/
│   │   │   ├── provider.py          # Abstract NewsProvider + NewsItem dataclass
│   │   │   ├── cryptopanic.py       # CryptoPanic adapter (primary)
│   │   │   ├── duckduckgo.py        # DuckDuckGo fallback adapter
│   │   │   ├── router.py            # GET /news/{project_slug}
│   │   │   └── quota.py             # Daily quota counter (Redis-backed)
│   │   ├── indicators/
│   │   │   ├── router.py            # POST /indicators/{asset} — accepts OHLCV slice, returns RSI + volume spike + MACD + Bollinger + price stats
│   │   │   ├── pct_change_router.py # POST /pct_change/{asset} — accepts OHLCV slice + window_minutes, returns pct_change (separate cache key)
│   │   │   ├── rsi.py               # 14-period RSI via pandas-ta (vectorised)
│   │   │   ├── volume_spike.py      # Volume spike: current vol vs 20-period SMA ratio via pandas-ta
│   │   │   ├── macd.py              # MACD(12,26,9) crossover detection via pandas-ta
│   │   │   ├── bollinger.py         # Bollinger Bands(20,2) breach detection via pandas-ta
│   │   │   ├── price_stats.py       # 24 h OHLCV summary (open/high/low/close/pct_change)
│   │   │   └── cache.py             # Redis-backed indicator cache (TTL = 25 s, under 30 s poll)
│   │   ├── enrichment/
│   │   │   ├── sentiment.py         # News headline sentiment scoring (VADER)
│   │   │   └── router.py            # POST /enrich/news — scores + deduplicates news items
│   │   └── projects/
│   │       ├── registry.py          # DB-backed slug→{display_name, symbol, coingecko_id, quote_currency} map
│   │       └── router.py            # GET /projects — curated project list
│   └── tests/
│       ├── contract/                # HTTP contract tests (all endpoints)
│       ├── integration/             # Redis, CryptoPanic, indicator pipeline tests
│       └── unit/                    # Adapter parsing, RSI math, sentiment, quota logic
│
├── agent-gateway/                      # Agent Gateway configuration
│   ├── agents/
│   │   └── digest-agent/
│   │       ├── AGENT.md             # Agent persona + instructions
│   │       ├── HEARTBEAT.md         # Health check checklist
│   │       └── skills/
│   │           └── crypto-digest.md # Digest generation skill
│   └── docker-compose.agent-gateway.yml
│
├── frontend/                        # React SPA
│   ├── src/
│   │   ├── components/              # Design system primitives + feature components
│   │   │   └── *.stories.tsx        # Storybook story files co-located with components
│   │   ├── pages/
│   │   │   ├── auth/                # SignIn, SignUp, ForgotPassword
│   │   │   ├── dashboard/           # Strategy list + quick stats
│   │   │   ├── strategies/          # Strategy create/edit, signal rule builder
│   │   │   ├── alerts/              # Alert history view
│   │   │   ├── watchlist/           # Watchlist management
│   │   │   ├── settings/            # Telegram linking, timezone, account
│   │   │   ├── billing/             # Subscription management (Stripe)
│   │   │   └── admin/               # Admin: user management, subscription config
│   │   ├── services/                # API client (TanStack Query hooks)
│   │   └── lib/                     # Utilities, design tokens
│   ├── .storybook/                  # Storybook config (main.ts, preview.ts)
│   └── tests/
│       ├── unit/                    # Vitest + React Testing Library
│       └── e2e/                     # Playwright
│
├── infra/
│   ├── docker-compose.yml           # Local dev: all services
│   ├── docker-compose.prod.yml      # Production overrides
│   ├── traefik/                     # Traefik static + dynamic config
│   └── monitoring/                  # Prometheus scrape config, Grafana dashboards
│
├── config/                          # Externalised business configuration
│   ├── seed.yaml                    # Signal types, project seed data, system tunables (FR-026)
│   └── seed.schema.json             # JSON Schema for seed.yaml validation
│
├── contracts/                       # Portable format schemas
│   └── strategy-definition.schema.json  # Strategy Definition Format JSON Schema (FR-027)
│
└── migrations/                      # PostgreSQL migrations (golang-migrate)
```

**Structure Decision**: Web application layout (backend + AI service + frontend + agent gateway).
Chosen over single-project because Go, Python, and React are distinct runtimes with independent
build/deploy pipelines. The Agent Gateway runs as a fourth service using its own Docker image.

---

## Seed Configuration (FR-026)

All business-level configuration is externalised into `config/seed.yaml`, validated against `config/seed.schema.json` at Go backend startup (fail-fast on invalid config). The seed file is NOT a database migration — it is a declarative description of the domain's configurable surface. The Go backend loads it once at startup and caches the parsed result in memory.

### Seed file structure (illustrative excerpt)

```yaml
# config/seed.yaml — business configuration, NOT secrets
version: "1"  # schema version; increment when breaking changes are made

system:
  poll_interval_seconds: 30
  signal_cooldown_seconds: 300
  indicator_cache_ttl_seconds: 25
  ai_service_readiness_timeout_seconds: 60

signal_types:
  price_threshold:
    display_name: "Price Threshold"
    description: "Fires when spot price crosses above or below a fixed value"
    parameters:
      operator: { type: enum, values: ["above", "below"], required: true }
      threshold: { type: decimal, required: true, currency_denominated: true }
    data_source: coingecko_spot

  pct_change:
    display_name: "% Price Change"
    description: "Fires when price changes by more than a threshold % over a time window"
    parameters:
      operator: { type: enum, values: ["above", "below"], required: true }
      threshold: { type: decimal, required: true, unit: "percent" }
      window_minutes: { type: enum, values: [5, 15, 60, 240, 1440], required: true }
    data_source_routing:
      - { window_minutes: [5, 15, 60, 240], source: cryptocompare_histominute, compute: ai_service_pct_change }
      - { window_minutes: [1440], source: coingecko_24h }

  rsi:
    display_name: "RSI"
    description: "Fires when 14-period RSI crosses overbought or oversold level"
    parameters:
      operator: { type: enum, values: ["above", "below"], required: true }
      threshold: { type: decimal, required: true, default: 70, unit: "dimensionless" }
      candle_minutes: { type: enum, values: [5, 15, 60], required: false, default: 15 }
    data_source: cryptocompare_histominute
    compute: ai_service_indicators
    candle_multiplier: 14   # fetch 14 × candle_minutes 1-min candles

  volume_spike:
    display_name: "Volume Spike"
    parameters:
      operator: { type: enum, values: ["above"], required: true }
      volume_threshold_pct: { type: decimal, required: false, default: 200, unit: "percent" }
      candle_minutes: { type: enum, values: [5, 15, 60], required: false, default: 15 }
    data_source: cryptocompare_histominute
    compute: ai_service_indicators
    candle_multiplier: 20

  macd_crossover:
    display_name: "MACD Crossover"
    parameters:
      cross_direction: { type: enum, values: ["bullish", "bearish"], required: true }
      candle_minutes: { type: enum, values: [5, 15, 60], required: false, default: 15 }
    data_source: cryptocompare_histominute
    compute: ai_service_indicators
    candle_multiplier: 35

  bollinger_breach:
    display_name: "Bollinger Band Breach"
    parameters:
      band_direction: { type: enum, values: ["upper", "lower"], required: true }
      candle_minutes: { type: enum, values: [5, 15, 60], required: false, default: 15 }
    data_source: cryptocompare_histominute
    compute: ai_service_indicators
    candle_multiplier: 20

projects_seed:   # seeded into app_projects on first run (idempotent upsert)
  - slug: btc
    display_name: Bitcoin
    symbol: BTC
    coingecko_id: bitcoin
    quote_currency: USD
    is_signal_asset: true
  - slug: eth
    display_name: Ethereum
    symbol: ETH
    coingecko_id: ethereum
    quote_currency: USD
    is_signal_asset: true
  - slug: sol
    display_name: Solana
    symbol: SOL
    coingecko_id: solana
    quote_currency: USD
    is_signal_asset: false   # watchlist only for POC
  # ... remaining ~17 watchlist-only projects
```

### Design decisions

- **YAML over DB-only config**: signal type parameter schemas are structural (they define what fields exist, their types, and allowed values). Keeping them in YAML makes them version-controlled, diff-friendly, and reviewable in PRs. The DB stores user data (`app_strategies`, `app_signal_rules`); the seed file stores the *shape* of that data.
- **JSON Schema validation at boot**: `config/seed.schema.json` prevents typos and structural drift from silently breaking the evaluator at runtime. The schema is also reusable by CI linting.
- **No secrets in seed**: environment-specific values (`TELEGRAM_BOT_TOKEN`, API keys, DSNs) remain in `.env` / env vars. The seed file is safe to commit.
- **Idempotent project seeding**: `projects_seed` entries are upserted into `app_projects` on startup (match by `slug`). This means the seed file is the source of truth for the initial project list, but admin-added projects in the DB are not overwritten.
- **Extensibility**: to add a new signal type (e.g., `funding_rate`), a developer adds a new entry under `signal_types` in the seed file, updates the JSON Schema, adds the computation logic in ai-service, and the evaluator/validator automatically pick it up — no changes to the strategy CRUD layer or DB schema (the `signal_type` column uses a CHECK constraint derived from the seed file's keys at migration time, or a VARCHAR with application-level validation).

---

## Strategy Definition Format — SDF (FR-027)

A portable, self-describing JSON structure that fully represents a strategy and its signal rules. The SDF is the canonical wire format for all strategy CRUD operations and the target format for future LLM-generated strategies.

### JSON Schema location

`contracts/strategy-definition.schema.json` — authoritative; all validation (backend, frontend, LLM pipeline) references this single schema.

### SDF structure (example instance)

```json
{
  "$schema": "../contracts/strategy-definition.schema.json",
  "name": "BTC Breakout",
  "asset": "btc",
  "status": "active",
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
    },
    {
      "signal_type": "pct_change",
      "operator": "above",
      "threshold": 5.0,
      "window_minutes": 60
    },
    {
      "signal_type": "volume_spike",
      "operator": "above",
      "volume_threshold_pct": 300,
      "candle_minutes": 15
    },
    {
      "signal_type": "macd_crossover",
      "cross_direction": "bullish",
      "candle_minutes": 60
    },
    {
      "signal_type": "bollinger_breach",
      "band_direction": "upper",
      "candle_minutes": 15
    }
  ]
}
```

### JSON Schema design (discriminated union)

The `rules` array uses a discriminated union on `signal_type`:

```json
{
  "rules": {
    "type": "array",
    "minItems": 1,
    "items": {
      "oneOf": [
        { "$ref": "#/$defs/price_threshold_rule" },
        { "$ref": "#/$defs/pct_change_rule" },
        { "$ref": "#/$defs/rsi_rule" },
        { "$ref": "#/$defs/volume_spike_rule" },
        { "$ref": "#/$defs/macd_crossover_rule" },
        { "$ref": "#/$defs/bollinger_breach_rule" }
      ],
      "discriminator": { "propertyName": "signal_type" }
    }
  }
}
```

Each `$def` specifies exactly the parameters valid for that signal type (e.g., `pct_change_rule` requires `window_minutes` from `{5,15,60,240,1440}` but forbids `candle_minutes`). This means:
- **Frontend** can auto-generate the form from the schema (show/hide fields per signal type)
- **Backend** validates incoming requests against the schema before business-rule checks
- **LLM** (post-POC) receives the schema as context and produces valid SDF documents from natural language

### Wire format alignment

| Endpoint | Request body | Response body |
|---|---|---|
| `POST /strategies` | Single SDF document (without `status` — defaults to `active`) | Created strategy with `id` + `status` |
| `GET /strategies/:id` | — | Full SDF document with `id`, `status`, `created_at` |
| `PUT /strategies/:id` | Full SDF document (without `id`) | Updated strategy |
| `POST /strategies/import` | `{ "strategies": [ SDF, SDF, ... ] }` | Array of created strategies with IDs |
| `GET /strategies/:id/export` | — | Pure SDF document (no server-side metadata except `id`) |

### LLM strategy generation pathway (post-POC extension point)

The SDF is intentionally designed to be the bridge between natural language and the system:

```
User text prompt
    → LLM (with SDF JSON Schema as context)
    → SDF JSON document(s)
    → POST /strategies/import (validation + persist)
    → Active strategies
```

For POC, the import endpoint and schema are implemented and tested. The LLM-to-SDF translation layer is deferred to post-POC but the contract is already stable.

---

## Database Index Strategy

All indexes are created in the same migration as their parent table. Each index MUST include a comment explaining the query pattern it optimises.

| Table | Index | Columns | Purpose |
|---|---|---|---|
| `app_strategies` | `idx_strategies_user_id` | `user_id` | Strategy list page: `WHERE user_id = $1` |
| `app_strategies` | `idx_strategies_user_status` | `user_id, status` | Evaluator: `WHERE user_id = $1 AND status = 'Active'` |
| `app_signal_rules` | `idx_signal_rules_strategy_id` | `strategy_id` | Join on strategy load |
| `app_alerts` | `idx_alerts_user_created` | `user_id, triggered_at DESC` | Alert history page: reverse-chronological per user |
| `app_alerts` | `idx_alerts_user_strategy` | `user_id, strategy_id, triggered_at DESC` | Alert history filtered by strategy |
| `app_alerts` | `idx_alerts_idempotency` | `signal_rule_id, triggered_at` | **UNIQUE** — idempotency guard (NFR-REL-002); prevents duplicate Alert records |
| `app_alerts` | `idx_alerts_pending_status` | `telegram_status` WHERE `telegram_status IN ('Pending', 'Failed')` | Re-drive worker and dashboard delay banner queries |
| `app_watchlist_entries` | `idx_watchlist_user_id` | `user_id` | Watchlist page: `WHERE user_id = $1` |
| `app_users` | `idx_users_email` | `email` | **UNIQUE** — enforced by Supabase Auth, but explicit index ensures fast lookup |
| `app_users` | `idx_users_telegram_chat_id` | `telegram_chat_id` WHERE `telegram_chat_id IS NOT NULL` | Partial index for linking lookup |

---

## Complexity Tracking

No constitution violations — no entries required.

---

## Research Notes (Phase 0 Summary)

Key findings to carry into implementation:

### Agent Gateway Integration (GoClaw default)
- GoClaw v1.74+ requires Go 1.26, PostgreSQL 18 + pgvector, Redis (optional but recommended)
- Digest agent uses `cron` tool (cron expression `30 8 * * *` = 08:30 UTC daily)
- **Apr 2026 updates** (apply during T006/T010/T055–T057a):
  - Web UI is now embedded in the Go binary — single `latest` Docker image, dashboard at `http://localhost:18790`; no `agent-gateway-web` service in docker-compose
  - Cron timezone handling is stable for all schedule kinds (`cron`, `at`, `every`); "Run Now" button works for manual testing
  - Subagents support `waitAll` + auto-retry + token tracking — use for parallel HTTP fetch per watchlist project
  - Per-agent grants (6th permission layer) — scope digest agent's API access to specific `/internal/` endpoints
  - KG sharing is configured separately from workspace sharing — configure KG access independently if needed
  - Channel health diagnostics panel in dashboard — use to verify Telegram connectivity before custom code
  - SSRF re-validation triggers automatically when a provider type is changed
- Per-user digest: `message` tool with user's Telegram chat ID (stored in Postgres by backend, read
  by Agent Gateway agent context or via a custom skill that queries backend API)
- The Agent Gateway shares Postgres with the backend service; schema namespacing required (prefix `gc_` for
  gateway tables vs `app_` for application tables)
- LLM provider configured via `AGENT_GATEWAY_ANTHROPIC_API_KEY` env var; model set per-agent in AGENT.md
- The Agent Gateway's built-in Telegram channel handles bot token + webhook; set `AGENT_GATEWAY_TELEGRAM_BOT_TOKEN`

### Signal Evaluation
- 30 s polling loop implemented as a Go ticker in `internal/signals/`; one goroutine per active
  strategy is wasteful at scale — evaluate all active strategies in a single pass per tick
- RSI calculation requires `14 × candle_minutes` 1-minute candles from CryptoCompare (e.g.,
  `candle_minutes=15` → 210 candles, `candle_minutes=60` → 840 candles); ai-service resamples
  into 14 candles of the chosen size before computing RSI-14
- Volume spike requires `20 × candle_minutes` 1-minute candles (e.g., `candle_minutes=15` → 300
  candles); ai-service resamples into 20 candles and computes volume SMA + ratio
- MACD crossover requires `35 × candle_minutes` 1-minute candles (e.g., `candle_minutes=15` →
  525 candles); 35 candles covers slow EMA(26) + signal(9) warm-up; ai-service resamples and
  computes MACD via `pandas-ta.macd()`
- Bollinger Band breach requires `20 × candle_minutes` 1-minute candles (e.g.,
  `candle_minutes=15` → 300 candles); ai-service resamples and computes Bollinger Bands via
  `pandas-ta.bbands()`
- The maximum candle fetch for any signal type is MACD at `candle_minutes=60`: 35 × 60 = 2100
  1-min candles — exceeds CryptoCompare `histominute` limit of 2000 per call. The adapter MUST
  use two paginated calls for this case (first call: limit=2000, second call: remaining 100
  candles with `toTs` set to the earliest timestamp from the first batch). Reducing warm-up
  to 33 candles is NOT acceptable — the MACD signal line (period 9) requires all 35 candles
  (26 slow EMA + 9 signal) for numerically stable values. All other signal type × candle size
  combinations fit within a single 2000-candle call
- Market data adapter must return the full rolling window, not just spot price
- **Signal cooldown (anti-duplicate)**: before publishing to NATS, poller checks Redis key `cooldown:{signal_rule_id}` (default TTL 5 min, configurable via `SIGNAL_COOLDOWN_SECONDS`); if key present, suppress publish for this tick — prevents duplicate events when a condition stays true across multiple consecutive ticks and protects against alert spam
- **NATS stream topology** (two streams — incompatible retention policies require separation):
  - `SIGNALS` stream: `signal.triggered.>`, `WorkQueuePolicy` (auto-delete on ACK), `MaxAge 1h`, `FileStorage`, `Replicas 1`; consumer `alerts-dispatcher` with `MaxAckPending: 100` (flow-control ceiling)
  - `ALERT_QUEUE` stream: `alert.pending` + `alert.expired`, `LimitsPolicy`, `MaxAge 25h`, `FileStorage`, `Replicas 1`; consumer `alerts-redriver` with `MaxDeliver: 48`, `AckWait: 60s`
  - Both streams initialised explicitly on backend startup via `pkg/nats/` — do NOT rely on NATS auto-create defaults
  - `FileStorage` requires a named Docker volume (`nats-data:/data`) in both local and production compose files
- Docker compose defines two explicit networks: `public` (Traefik, frontend, backend) and `internal` (backend, ai-service, Agent Gateway, NATS, Redis, Postgres); ai-service is on `internal` only — never exposed via Traefik
- On signal fire: publish `signal.triggered.{asset}` to `SIGNALS` stream (subject is asset-keyed; user isolation enforced by `user_id` in payload); `alerts-dispatcher` persists Alert and calls Telegram Bot API directly (not via the Agent Gateway — the gateway handles digest only; real-time alerts go direct for latency); if user has no linked Telegram, dispatcher sets `telegram_status=Pending` and publishes `alert.pending` to `ALERT_QUEUE`
- **Re-drive pattern (FR-025)**: `alerts-redriver` consumes `alert.pending`; if still unlinked calls `NakWithDelay(backoff)` — NATS redelivers the *same* message (no re-publish, no stream bloat); when `MaxDeliver` is exhausted NATS forwards message to `alert.expired` subject; lightweight DLQ handler sets `telegram_status=Expired`; 24 h expiry enforced by stream `MaxAge` as the authoritative safety net
- **Circuit breaker on ai-service** (`sony/gobreaker`): the Go evaluator wraps all `POST /indicators/{asset}` and `POST /pct_change/{asset}` calls in a circuit breaker; after 5 consecutive failures (timeout or 5xx), the circuit opens for 30 s — all ai-service calls during this window return immediately with a sentinel error (tick is skipped, logged as warning, no alert fired); after 30 s the circuit half-opens and allows one probe request; if it succeeds, circuit closes and normal evaluation resumes; this prevents cascading timeouts when ai-service is degraded and preserves the 30 s tick budget for non-indicator signals (price threshold via CoinGecko still evaluates normally)
- **Graceful shutdown**: on `SIGTERM` / `SIGINT` the backend MUST: (1) stop the 30 s evaluation ticker (no new ticks start), (2) drain NATS connections — finish in-flight message ACKs, then close consumers and publisher, (3) close Redis connection pool, (4) close Postgres connection pool, (5) stop HTTP server with `http.Server.Shutdown(ctx)` (deadline = 15 s); log each shutdown step; if any step exceeds deadline, force-close and log warning; this ensures zero-downtime deploys and no lost NATS ACKs
- **ai-service readiness gate**: on startup the Go evaluator MUST poll `GET ai-service:8000/health/ready` with 2 s interval and 60 s overall timeout before starting the 30 s evaluation ticker; if ai-service is not ready within 60 s, the backend starts but logs a CRITICAL warning and skips indicator-based signal evaluation (price threshold + CoinGecko 24h % change still evaluate normally); the readiness check is also used by docker-compose `depends_on.condition: service_healthy`

### Market Data Feed
- **Selected for POC**: CoinGecko Free API (no key required, 30 req/min — sufficient for 2 assets
  at 30 s poll interval = 4 req/min; adding more tokens scales linearly — each new asset adds
  ~2 req/min; free-tier headroom supports up to ~7 assets at 30 s polling)
- RSI requires `/coins/{id}/ohlc` endpoint (returns up to 30 days of OHLC at 4h resolution);
  for intraday RSI, use CryptoCompare `histominute` endpoint (free tier: 100k req/month)
- Abstracted behind `pkg/marketdata.Provider` interface — swap without changing signal evaluator
- **CryptoCompare request budget analysis** (POC worst-case per 30 s tick):
  - Distinct `{asset, candle_minutes}` combos for indicators: 2 assets × 3 candle sizes = **6 calls**
  - Distinct `{asset, window_minutes}` combos for pct_change: 2 assets × 4 windows (5,15,60,240) = **8 calls**
  - MACD at `candle_minutes=60` requires 2 paginated calls per asset = **+2 extra calls**
  - **Total worst-case per tick: 16 calls** → 16 × 2/min = **32 req/min** → **~1.38M req/month**
  - CryptoCompare free tier: 100k req/month → **exceeds free tier if all combos are active simultaneously**
  - **Mitigation**: (a) indicator cache (25 s TTL) means most ticks are cache hits, reducing actual calls to ~1-2 per tick for a small user base; (b) most POC users will use default `candle_minutes=15` — realistically 2-4 distinct combos, not 14; (c) upgrade to CryptoCompare paid tier ($0/month for 500k req) if needed; (d) batch multiple assets into one CryptoCompare call where API supports it (TODO: verify `histominute` multi-asset support)
  - **Monitoring**: Track `cryptocompare_requests_total` Prometheus counter per tick; alert if approaching 80% of monthly budget
- **Adapter routing by signal type and time window** (per FR-003):
  | Signal Type | Config field | Allowed values | Data Source | Method |
  |---|---|---|---|---|
  | Price threshold | — | N/A (spot) | CoinGecko `/simple/price?vs_currencies={quote_currency}` | Current spot price vs. threshold (quote currency from `app_projects.quote_currency`, default `usd`) |
  | % price change | `window_minutes` | 5, 15, 60, 240 | CryptoCompare `histominute` (`tsym={quote_currency}`) | Fetch `window_minutes` 1-min candles → forward OHLCV slice to ai-service `pct_change` computation → Go evaluator compares returned % against threshold |
  | % price change | `window_minutes` | 1440 (24 h) | CoinGecko `/simple/price?vs_currencies={quote_currency}` | `price_change_percentage_24h` field (currency-agnostic %; spot price in quote currency) |
  | RSI | `candle_minutes` | 5, 15, 60 (default 15) | CryptoCompare `histominute` | Fetch `14 × candle_minutes` 1-min candles → resample → ai-service RSI |
  | Volume spike | `candle_minutes` | 5, 15, 60 (default 15) | CryptoCompare `histominute` | Fetch `20 × candle_minutes` 1-min candles → resample → ai-service volume SMA ratio |
  | MACD crossover | `candle_minutes` | 5, 15, 60 (default 15) | CryptoCompare `histominute` | Fetch `35 × candle_minutes` 1-min candles → resample → ai-service MACD |
  | Bollinger Band breach | `candle_minutes` | 5, 15, 60 (default 15) | CryptoCompare `histominute` | Fetch `20 × candle_minutes` 1-min candles → resample → ai-service Bollinger Bands |
  Go signal evaluator selects adapter based on `signal_type` + `window_minutes` / `candle_minutes` from each rule.
  The evaluator resolves the strategy's `asset` slug to a CoinGecko coin ID (e.g., `btc` → `bitcoin`) via `app_projects.coingecko_id`; this lookup is cached in-memory on startup and refreshed every 5 min, so adding a new token to `app_projects` requires no code changes — only a data insert + evaluator cache refresh

### Python ai-service Design

**Why Python owns indicator computation:**
- `pandas-ta` computes 14-period RSI in one line on a DataFrame — no hand-rolled Wilder smoothing needed; `pandas` `resample()` converts 1-min candles to user-selected candle size (5/15/60 min) before indicator computation
- Volume spike detection: `pandas-ta.sma(volume, length=20)` gives the baseline; current volume / SMA ratio computed in one expression
- MACD(12,26,9): `pandas-ta.macd()` returns MACD line, signal line, and histogram in one call
- Bollinger Bands(20,2): `pandas-ta.bbands()` returns upper, mid, and lower bands in one call
- `numpy` vectorises OHLCV aggregation (24 h open/high/low/close/pct_change) trivially
- **Intraday % price change** (`window_minutes` ≤ 240): ai-service receives the OHLCV slice, extracts `close` at index 0 (T − window) and `close` at the last index (current), computes `(current − old) / old × 100`; this keeps all OHLCV-derived math in Python and the Go evaluator simply compares the returned `pct_change` value against the user's threshold — same pattern as RSI/MACD/Bollinger
- Result is cached in Redis at TTL = 25 s (safely under the 30 s poll cycle); Go evaluator calls `POST /indicators/{asset}` (OHLCV slice in request body) and receives a pre-computed struct — no OHLCV slice management in Go
- All seven computations (RSI, volume spike, MACD, Bollinger Bands, price stats, intraday % change, 24 h stats) use the same `histominute` candle data and share the resample-then-compute pattern; Go never performs arithmetic on OHLCV data

**Why Python owns news enrichment:**
- `vaderSentiment` for headline sentiment scoring is 3 lines in Python
- Deduplication (URL hash + title fuzzy-match) prevents the Agent Gateway LLM from wasting tokens on repeated headlines
- `POST /enrich/news` input: `list[NewsItem]` → output: `list[EnrichedNewsItem]` with `sentiment_score`, `is_duplicate` flag
- The Agent Gateway receives already-enriched, deduplicated items — LLM prompt is shorter, output quality is higher

**Indicator cache design:**
- **Indicator cache** (RSI, volume spike, MACD, Bollinger, price stats): Redis key `indicators:{asset}:{candle_minutes}:{YYYY-MM-DD-HH-MM}` rounded to nearest 25 s, TTL = 25 s; different `candle_minutes` values produce separate cache entries; cached response: `{ rsi, volume_spike_ratio, macd_line, macd_signal, macd_histogram, bb_upper, bb_mid, bb_lower, price_stats, quote_currency }`; a single cache entry serves RSI, volume-spike, MACD, and Bollinger rules sharing the same asset + candle size; price-denominated fields (`bb_upper`, `bb_mid`, `bb_lower`, `price_stats.*`) are in the asset's `quote_currency`
- **Pct-change cache** (intraday % price change): Redis key `pct_change:{asset}:{window_minutes}:{YYYY-MM-DD-HH-MM}` rounded to nearest 25 s, TTL = 25 s; keyed by `window_minutes` (not `candle_minutes`) because the OHLCV slice size varies by window (e.g., 60 candles for 1 h vs. 840 candles for RSI at `candle_minutes=60`); cached response: `{ pct_change, quote_currency }`; separate endpoint `POST /pct_change/{asset}?window_minutes={wm}` to avoid overloading the indicator cache with incompatible slice sizes
- Go evaluator: on cache hit → use cached value; on cache miss → ai-service fetches OHLCV, computes, stores, returns
- Prevents duplicate CryptoCompare API calls when multiple strategies evaluate in the same tick

**Python dependency stack:**
- `fastapi` + `uvicorn` — async HTTP
- `httpx` — async external API calls
- `pandas` + `pandas-ta` — indicator computation
- `numpy` — OHLCV aggregation
- `vaderSentiment` — headline sentiment scoring (fast, CPU-only, no GPU needed; sufficient for POC)
- `redis[asyncio]` — async Redis client
- `pydantic-settings` — config management
- `structlog` — JSON structured logging
- `pytest` + `pytest-asyncio` + `httpx` (test client) — testing

### Crypto News API
- **Selected for POC**: CryptoPanic free tier (50 req/day per token, returns news by currency filter)
- Agent Gateway digest agent calls HTTP fetch with CryptoPanic API endpoint per watchlist project,
  passes results to Anthropic claude-3-5-haiku for summarisation (≤ 200 tokens per project)
- Fallback: if CryptoPanic quota exceeded, the Agent Gateway uses web search (DuckDuckGo) as fallback

### Supabase Auth
- Supabase Auth handles JWT issuance; Go backend validates JWTs using Supabase public key
- Row Level Security (RLS) on all tables ensures users can only read/write their own rows
- Email verification and password reset are built into Supabase Auth — no custom flow required
  (FR-020 is satisfied by delegating these flows to Supabase; `internal/auth/` only needs JWT
  validation middleware and user-context injection)

### Stripe
- POC uses a single subscription product with one price (monthly flat fee)
- Stripe webhook events consumed by `internal/billing/`: `customer.subscription.created`,
  `customer.subscription.deleted`, `invoice.payment_failed`
- Subscription status stored on `users` table; middleware checks status on protected routes
