# Implementation Plan: Investment Intel AI Agents — POC

**Branch**: `001-investment-intel-poc` | **Date**: 2026-03-21 | **Spec**: [spec.md](./spec.md)  
**Input**: Feature specification from `/specs/001-investment-intel-poc/spec.md`

## Summary

Build a POC platform where users self-register, configure rule-based signal strategies for BTC/ETH,
and receive real-time Telegram notifications when conditions fire (< 60 s SLA via 30 s polling).
Users maintain a watchlist of crypto projects and receive a daily AI-summarised Telegram digest
(LLM summarises raw news from a third-party crypto news API) by 09:00 in their configured timezone.
A React web app hosts all configuration UX; alert history is visible per strategy; billing and
subscription management run through Stripe.

The AI agent layer (GoClaw) orchestrates the digest pipeline: a scheduled GoClaw cron agent polls
the news API, invokes the LLM summarisation tool, and dispatches the Telegram digest — replacing
bespoke Python orchestration with GoClaw's built-in `cron`, `web_fetch`, and `message` tools.

---

## Technical Context

**Language/Version**:
- Backend API: Go 1.24+ (Golang)
- AI/Data services: Python 3.12+
- GoClaw agent gateway: Go 1.26+ (per GoClaw prerequisites)
- Frontend: Node.js 22+ (React 19)

**Primary Dependencies**:
- **API Gateway**: Traefik v3 (routing, TLS termination, rate limiting)
- **Auth / User management**: Supabase (PostgreSQL + Auth + Row Level Security)
- **Billing**: Stripe SDK (Go + React)
- **AI Agent gateway**: GoClaw v1.74+ → current (multi-agent orchestration, Telegram channel, cron, LLM tools); **web UI embedded in binary** since Apr 2026 — no separate web container needed; dashboard at `http://localhost:18790`; permission model now has **6 layers** (added: per-agent grants with setting-level overrides, Apr 2026)
- **Message queue**: NATS JetStream (signal event fanout, notification queue)
- **Frontend**: ReactJS 19 + TailwindCSS v4 + TanStack Router + TanStack Query
- **LLM (digest summarisation)**: Anthropic Claude (via GoClaw provider config, claude-3-5-haiku)
- **Crypto news API**: CryptoPanic free tier (50 req/day per token; DuckDuckGo fallback on quota
  exhaustion). Abstracted behind `NewsProvider` interface in `ai-service/src/news/`.
- **Market data feed**: CoinGecko Free API (no key required, ≤30 req/min) for spot price and
  percentage-change signals; CryptoCompare `histominute` endpoint (free tier, 100k req/month)
  for intraday OHLCV data required by the 14-period RSI calculation. Both abstracted behind
  `pkg/marketdata.Provider` interface.
- **Logging**: zerolog (Go), structlog (Python)
- **Monitoring**: Prometheus + Grafana

**Storage**:
- PostgreSQL 18 with pgvector (primary store — users, strategies, signal rules, alerts, watchlists,
  digests; also used by GoClaw for its multi-tenant store)
- Redis 7 (API response cache, rate-limit counters, session tokens, signal evaluation state)

**Testing**:
- Go: `go test` + `testify` + `httptest` for API contract tests
- Python: `pytest` + `pytest-asyncio`
- Frontend: Vitest + React Testing Library + Playwright (E2E)
- CI: GitHub Actions

**Target Platform**: DigitalOcean Droplets (Linux/amd64) + Cloudflare DNS/CDN/WAF

**Project Type**: Web service (React SPA + Go microservices + Python AI service + GoClaw agent gateway)

**Performance Goals**:
- Signal-triggered Telegram notification: < 60 s end-to-end (30 s poll cycle + < 30 s delivery)
- REST API endpoints: < 200 ms p95
- React SPA: LCP ≤ 2.5 s, CLS ≤ 0.1 (Cloudflare CDN-served static assets)
- Daily digest delivery: before 09:00 user-local time, 100% of days during POC

**Constraints**:
- POC scale: ≤ 50 registered users — no horizontal scaling required
- GoClaw single binary (~25 MB, ~35 MB RAM idle) fits on smallest Droplet alongside other services
- Market data polling at 30 s intervals must stay within free-tier or low-cost API rate limits
- Stripe integration limited to subscription creation and webhook processing (no custom invoicing)

**Scale/Scope**: ≤ 50 users, 2 assets (BTC/ETH) for signals, curated watchlist of ~20 projects

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
| Integration tests for NATS, Supabase, GoClaw, Stripe webhook | ✅ PASS | See Phase 2 foundational tasks |

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
| Background job SLA: digest before 09:00 UTC | ✅ PASS | GoClaw cron expression set to 08:30 UTC (delivers before cutoff) |

**Constitution verdict**: ✅ ALL GATES PASS — no violations to justify.

---

## GoClaw Agent Architecture

GoClaw (`github.com/nextlevelbuilder/goclaw`) replaces bespoke Python orchestration for the AI
agent layer. Key features used in this POC:

| GoClaw Feature | POC Usage |
|----------------|-----------|
| **Cron scheduling** (`cron` tool) | Daily digest trigger at 08:30 UTC |
| **web_fetch tool** | Fetches news items from crypto news API per watchlist project |
| **LLM provider (Anthropic)** | Summarises raw news items via claude-3-5-haiku |
| **Telegram channel** | Dispatches digest messages and (future) alert escalation |
| **Multi-tenant PostgreSQL** | GoClaw shares the same Postgres instance; per-user agent contexts |
| **message tool** | Sends formatted digest to each user's Telegram chat |
| **Agent teams + task board** | Optional future: coordinator agent delegates per-user digest to sub-agents |
| **Subagent `waitAll` + auto-retry** | Parallel `web_fetch` per watchlist project with automatic retry and token tracking (Apr 2026) |
| **Per-agent grants** | Scope digest agent's backend API access to specific `/internal/` endpoints; 6th permission layer (Apr 2026) |
| **Channel health diagnostics** | Actionable Telegram channel health panel with remediation steps — use for T010/T031 debugging (Apr 2026) |
| **Built-in observability (OTLP)** | LLM call tracing feeds Prometheus/Grafana |

**Deployment**: GoClaw runs as a **single Docker container** (one binary — web UI + API embedded, dashboard at `http://localhost:18790`) alongside backend services, connected to the shared Postgres and Redis instances. **No separate `goclaw-web` container is needed** (web UI was embedded into the Go binary in Apr 2026). NATS is not used by GoClaw directly — the Go signal service publishes to NATS; GoClaw operates on its own scheduler for digest tasks.

---

## Project Structure

### Documentation (this feature)

```text
specs/001-investment-intel-poc/
├── plan.md              ← this file
├── research.md          ← GoClaw integration patterns, market data feed evaluation
├── data-model.md        ← entity schemas, state machines
├── quickstart.md        ← local dev setup
├── contracts/
│   ├── rest-api.md      ← REST endpoint contracts (auth, strategies, alerts, watchlist, digest)
│   └── nats-events.md   ← NATS message schemas (signal.triggered, digest.requested)
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
│   │   ├── billing/                 # Stripe subscription + webhook handler
│   │   └── digest/                  # Digest schedule coordination
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
│   │   ├── news/                    # Crypto news API adapter (interface + impl)
│   │   └── health.py                # Health endpoint
│   └── tests/
│       ├── contract/
│       ├── integration/
│       └── unit/
│
├── goclaw/                          # GoClaw agent configuration
│   ├── agents/
│   │   └── digest-agent/
│   │       ├── AGENT.md             # Agent persona + instructions
│   │       ├── HEARTBEAT.md         # Health check checklist
│   │       └── skills/
│   │           └── crypto-digest.md # Digest generation skill
│   └── docker-compose.goclaw.yml
│
├── frontend/                        # React SPA
│   ├── src/
│   │   ├── components/              # Design system primitives + feature components
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
└── migrations/                      # PostgreSQL migrations (golang-migrate)
```

**Structure Decision**: Web application layout (backend + AI service + frontend + agent gateway).
Chosen over single-project because Go, Python, and React are distinct runtimes with independent
build/deploy pipelines. GoClaw runs as a fourth service using its own Docker image.

---

## Complexity Tracking

No constitution violations — no entries required.

---

## Research Notes (Phase 0 Summary)

Key findings to carry into implementation:

### GoClaw Integration
- GoClaw v1.74+ requires Go 1.26, PostgreSQL 18 + pgvector, Redis (optional but recommended)
- Digest agent uses `cron` tool (cron expression `30 8 * * *` = 08:30 UTC daily)
- **Apr 2026 updates** (apply during T006/T010/T055–T057a):
  - Web UI is now embedded in the Go binary — single `latest` Docker image, dashboard at `http://localhost:18790`; no `goclaw-web` service in docker-compose
  - Cron timezone handling is stable for all schedule kinds (`cron`, `at`, `every`); "Run Now" button works for manual testing
  - Subagents support `waitAll` + auto-retry + token tracking — use for parallel `web_fetch` per watchlist project
  - Per-agent grants (6th permission layer) — scope digest agent's API access to specific `/internal/` endpoints
  - KG sharing is configured separately from workspace sharing — configure KG access independently if needed
  - Channel health diagnostics panel in dashboard — use to verify Telegram connectivity before custom code
  - SSRF re-validation triggers automatically when a provider type is changed
- Per-user digest: `message` tool with user's Telegram chat ID (stored in Postgres by backend, read
  by GoClaw agent context or via a custom skill that queries backend API)
- GoClaw shares Postgres with the backend service; schema namespacing required (prefix `gc_` for
  GoClaw tables vs `app_` for application tables)
- LLM provider configured via `GOCLAW_ANTHROPIC_API_KEY` env var; model set per-agent in AGENT.md
- GoClaw's built-in Telegram channel handles bot token + webhook; set `GOCLAW_TELEGRAM_BOT_TOKEN`

### Signal Evaluation
- 30 s polling loop implemented as a Go ticker in `internal/signals/`; one goroutine per active
  strategy is wasteful at scale — evaluate all active strategies in a single pass per tick
- RSI calculation requires at minimum 14 candles of OHLCV data; market data adapter must return
  rolling window, not just spot price
- On signal fire: publish `signal.triggered` event to NATS JetStream; separate consumer in
  `internal/alerts/` persists Alert record and dispatches Telegram message via Telegram Bot API
  directly (not via GoClaw — GoClaw handles digest only; real-time alerts go direct for latency)

### Market Data Feed
- **Selected for POC**: CoinGecko Free API (no key required, 30 req/min — sufficient for 2 assets
  at 30 s poll interval = 4 req/min)
- RSI requires `/coins/{id}/ohlc` endpoint (returns up to 30 days of OHLC at 4h resolution);
  for intraday RSI, use CryptoCompare `histominute` endpoint (free tier: 100k req/month)
- Abstracted behind `pkg/marketdata.Provider` interface — swap without changing signal evaluator

### Crypto News API
- **Selected for POC**: CryptoPanic free tier (50 req/day per token, returns news by currency filter)
- GoClaw digest agent calls `web_fetch` with CryptoPanic API endpoint per watchlist project,
  passes results to Anthropic claude-3-5-haiku for summarisation (≤ 200 tokens per project)
- Fallback: if CryptoPanic quota exceeded, GoClaw uses `web_search` (DuckDuckGo) as fallback

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
