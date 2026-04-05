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
- **Component development & docs**: Storybook 8 (`@storybook/react-vite`) — isolated component development, visual regression baseline, and living design-system documentation
- **LLM (digest summarisation)**: Anthropic Claude (via GoClaw provider config, claude-3-5-haiku)
- **Crypto news API**: CryptoPanic free tier (50 req/day per token; DuckDuckGo fallback on quota
  exhaustion). Abstracted behind `NewsProvider` interface in `ai-service/src/news/`.
- **Market data feed**: CoinGecko Free API (no key required, ≤30 req/min) for spot price and
  percentage-change signals; CryptoCompare `histominute` endpoint (free tier, 100k req/month)
  for intraday OHLCV data forwarded to Python ai-service for indicator computation. Go adapter
  abstracted behind `pkg/marketdata.Provider`; RSI and OHLCV math happens in Python. All price
  data is denominated in the asset's `quote_currency` from `app_projects` (default `USD`); the
  adapter passes this value as CoinGecko `vs_currencies` and CryptoCompare `tsym` parameters.
- **Technical indicators**: `pandas-ta` (RSI, MACD, Bollinger) + `numpy` — owned entirely by
  Python ai-service; Go signal evaluator calls `POST /indicators/{asset}` to get pre-computed values
- **News sentiment**: `vaderSentiment` (POC) — fast, CPU-only; scores news headlines before GoClaw
  LLM step; reduces token waste on low-signal or duplicate items
- **Logging**: zerolog (Go), structlog (Python)
- **Monitoring**: Prometheus + Grafana

**Storage**:
- PostgreSQL 18 with pgvector (primary store — users, strategies, signal rules, alerts, watchlists,
  digests; also used by GoClaw for its multi-tenant store)
- Redis 7 (API response cache, rate-limit counters, session tokens, signal evaluation state)

**Testing**:
- Go: `go test` + `testify` + `httptest` for API contract tests
- Python: `pytest` + `pytest-asyncio`
- Frontend: Vitest + React Testing Library + Playwright (E2E) + Storybook (component stories + `@storybook/test` interaction tests)
- CI: GitHub Actions

**Target Platform**: DigitalOcean Droplets (Linux/amd64) + Cloudflare DNS/CDN/WAF

**Project Type**: Web service (React SPA + Go microservices + Python AI service + GoClaw agent gateway)

**Performance Goals**:
- Signal-triggered Telegram notification: < 60 s end-to-end (30 s poll cycle + < 30 s delivery)
- REST API endpoints: < 200 ms p95
- React SPA: LCP ≤ 2.5 s, CLS ≤ 0.1 (Cloudflare CDN-served static assets)
- Daily digest delivery: before 09:00 UTC, 100% of days during POC (single-timezone cron at 08:30 UTC; per-user timezone scheduling is post-POC)

**Constraints**:
- POC scale: ≤ 50 registered users — no horizontal scaling required
- GoClaw single binary (~25 MB, ~35 MB RAM idle) fits on smallest Droplet alongside other services
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
| Background job SLA: digest before 09:00 UTC | ✅ PASS | GoClaw cron expression set to 08:30 UTC (delivers before cutoff); per-user timezone scheduling deferred to post-POC |

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
| **Technical indicator computation (RSI, volume SMA, MACD, Bollinger Bands, OHLCV stats)** | **Python ai-service** | `pandas-ta` / `numpy` vectorised math; zero-effort 14-period RSI, 20-period volume SMA ratio, MACD(12,26,9), Bollinger Bands(20,2) vs hand-rolled Go; accepts `candle_minutes` parameter to resample 1-min candles into the user-selected candle size (5/15/60 min) before computing indicators; result cached in Redis at 25 s TTL keyed by `{asset}:{candle_minutes}`; Go evaluator calls `POST /indicators/{asset}` per tick (OHLCV slice + `candle_minutes` in request body) |
| **News fetching & `NewsProvider` interface** | **Python ai-service** | CryptoPanic + DuckDuckGo fallback; async `httpx`; quota tracking; `GET /news/{slug}` |
| **News sentiment scoring & deduplication** | **Python ai-service** | `transformers` (FinBERT) or VADER for headline sentiment; deduplication before GoClaw summarisation; `POST /enrich/news` |
| **Curated project registry** | **Python ai-service** | Slug → `{display_name, symbol, coingecko_id, quote_currency}` map; `GET /projects`; single source of truth for both React watchlist/strategy form and GoClaw skill; `quote_currency` (default `USD`) is consumed by Go market data adapters |
| NATS publish/subscribe (signal events, alert queue, DLQ) | **Go backend** | All real-time event flow is Go-owned; GoClaw and ai-service do NOT touch NATS |
| Alert persistence & Telegram delivery (real-time) | **Go backend** | Direct Telegram Bot API call for < 60 s latency; GoClaw's scheduler is too coarse for real-time |
| Telegram account linking (deep-link, webhook, bot token) | **Go backend** | Stateful linking flow requires Postgres writes; linked to auth middleware |
| Stripe checkout, webhook processing, subscription gating | **Go backend** | Financial operations require transaction safety and webhook signature validation |
| PostgreSQL migrations | **Go backend** (`migrations/`, `golang-migrate`) | Single migration runner owns schema; GoClaw uses `gc_` prefix namespace only |
| Daily digest orchestration (cron, fetch, summarise, send) | **GoClaw** | Built-in `cron`, `web_fetch`, LLM provider, and `message` tools; calls ai-service `GET /news/{slug}` and `POST /enrich/news` |
| LLM summarisation (claude-3-5-haiku) | **GoClaw** | LLM provider configured natively in GoClaw; enriched + deduplicated news items from ai-service are the input |
| Telegram digest delivery | **GoClaw** | GoClaw Telegram channel; separate from Go alert dispatcher |
| Per-user agent context & watchlist retrieval | **GoClaw → Go `/internal/` API** | GoClaw skill calls Go's `/internal/watchlist` using service token (T057a) |
| UI (all configuration, history, settings, billing pages) | **React SPA** | Single-page app via Cloudflare CDN; all data via TanStack Query → Go REST API |
| Design system primitives & component stories | **React SPA** (Storybook) | Co-located `*.stories.tsx`; visual baseline and living docs |
| Routing, TLS termination, rate limiting | **Traefik** | Handles `api.`, `app.`, `goclaw.` subdomains; services never terminate TLS directly |
| Metrics exposure & tracing | **Go backend** + **GoClaw** (OTLP) + **ai-service** (`/metrics`) | All three expose Prometheus metrics; GoClaw emits OTLP traces |

### Cross-Service Communication Rules

| From | To | Protocol | Auth |
|---|---|---|---|
| React SPA | Go backend | HTTPS REST (`/api/v1/`) | Supabase JWT (httpOnly cookie) |
| GoClaw digest agent | Go backend | HTTPS REST (`/internal/`) | Static bearer token (`GOCLAW_INTERNAL_TOKEN`) |
| GoClaw digest agent | Python ai-service | `web_fetch` via `GET /news/{slug}`, `POST /enrich/news`, `GET /projects` | None (internal network only) |
| Go backend (signal evaluator) | Python ai-service | HTTP `POST /indicators/{asset}` | None (internal network only) |
| Go backend | NATS JetStream | NATS protocol | NATS credentials |
| Go backend | Telegram Bot API | HTTPS | Bot token |
| GoClaw | Telegram (digest) | GoClaw Telegram channel | GoClaw bot token (`GOCLAW_TELEGRAM_BOT_TOKEN`) |
| Go backend | Supabase (Auth + DB) | HTTPS + postgres:// | Supabase service role key + RLS |
| Go backend | Stripe | HTTPS | Stripe secret key + webhook signing secret |
| Go backend | CoinGecko / CryptoCompare | HTTPS | No key / free-tier key |
| Python ai-service | CryptoPanic | HTTPS | API token |
| Python ai-service | Redis | redis:// | Password (internal) |

### What Each Service MUST NOT Do

| Service | Must NOT |
|---|---|
| **Go backend** | Call LLM APIs directly; run digest orchestration; compute technical indicators (delegate to ai-service); write to `gc_` schema tables |
| **Python ai-service** | Evaluate signal threshold rules (that logic stays in Go); touch NATS; send Telegram messages; own any user-facing Postgres tables |
| **GoClaw** | Connect to NATS; perform real-time alert dispatch; own Postgres migrations; compute indicators directly |
| **React SPA** | Call market data or news APIs directly; store secrets; implement business logic |
| **Traefik** | Contain application logic; be aware of NATS or GoClaw internals |

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
│   │   ├── billing/                 # Stripe subscription + webhook handler
│   │   └── telegram/                # Telegram account linking
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
│   │   │   ├── rsi.py               # 14-period RSI via pandas-ta (vectorised)
│   │   │   ├── volume_spike.py      # Volume spike: current vol vs 20-period SMA ratio via pandas-ta
│   │   │   ├── macd.py              # MACD(12,26,9) crossover detection via pandas-ta
│   │   │   ├── bollinger.py         # Bollinger Bands(20,2) breach detection via pandas-ta
│   │   │   ├── price_stats.py       # 24 h OHLCV summary (open/high/low/close/pct_change)
│   │   │   └── cache.py             # Redis-backed indicator cache (TTL = 25 s, under 30 s poll)
│   │   ├── enrichment/
│   │   │   ├── sentiment.py         # News headline sentiment scoring (transformers/VADER)
│   │   │   └── router.py            # POST /enrich/news — scores + deduplicates news items
│   │   └── projects/
│   │       ├── registry.py          # Static slug→{display_name, symbol, coingecko_id} map
│   │       └── router.py            # GET /projects — curated project list
│   └── tests/
│       ├── contract/                # HTTP contract tests (all endpoints)
│       ├── integration/             # Redis, CryptoPanic, indicator pipeline tests
│       └── unit/                    # Adapter parsing, RSI math, sentiment, quota logic
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
  1-min candles — well within CryptoCompare `histominute` limits (max 2000 per call; may need
  two calls for MACD at 60-min candles or reduce warm-up headroom to 34)
- Market data adapter must return the full rolling window, not just spot price
- **Signal cooldown (anti-duplicate)**: before publishing to NATS, poller checks Redis key `cooldown:{signal_rule_id}` (default TTL 5 min, configurable via `SIGNAL_COOLDOWN_SECONDS`); if key present, suppress publish for this tick — prevents duplicate events when a condition stays true across multiple consecutive ticks and protects against alert spam
- **NATS stream topology** (two streams — incompatible retention policies require separation):
  - `SIGNALS` stream: `signal.triggered.>`, `WorkQueuePolicy` (auto-delete on ACK), `MaxAge 1h`, `FileStorage`, `Replicas 1`; consumer `alerts-dispatcher` with `MaxAckPending: 100` (flow-control ceiling)
  - `ALERT_QUEUE` stream: `alert.pending` + `alert.expired`, `LimitsPolicy`, `MaxAge 25h`, `FileStorage`, `Replicas 1`; consumer `alerts-redriver` with `MaxDeliver: 48`, `AckWait: 60s`
  - Both streams initialised explicitly on backend startup via `pkg/nats/` — do NOT rely on NATS auto-create defaults
  - `FileStorage` requires a named Docker volume (`nats-data:/data`) in both local and production compose files
- On signal fire: publish `signal.triggered.{asset}` to `SIGNALS` stream (subject is asset-keyed; user isolation enforced by `user_id` in payload); `alerts-dispatcher` persists Alert and calls Telegram Bot API directly (not via GoClaw — GoClaw handles digest only; real-time alerts go direct for latency); if user has no linked Telegram, dispatcher sets `telegram_status=Pending` and publishes `alert.pending` to `ALERT_QUEUE`
- **Re-drive pattern (FR-025)**: `alerts-redriver` consumes `alert.pending`; if still unlinked calls `NakWithDelay(backoff)` — NATS redelivers the *same* message (no re-publish, no stream bloat); when `MaxDeliver` is exhausted NATS forwards message to `alert.expired` subject; lightweight DLQ handler sets `telegram_status=Expired`; 24 h expiry enforced by stream `MaxAge` as the authoritative safety net

### Market Data Feed
- **Selected for POC**: CoinGecko Free API (no key required, 30 req/min — sufficient for 2 assets
  at 30 s poll interval = 4 req/min; adding more tokens scales linearly — each new asset adds
  ~2 req/min; free-tier headroom supports up to ~7 assets at 30 s polling)
- RSI requires `/coins/{id}/ohlc` endpoint (returns up to 30 days of OHLC at 4h resolution);
  for intraday RSI, use CryptoCompare `histominute` endpoint (free tier: 100k req/month)
- Abstracted behind `pkg/marketdata.Provider` interface — swap without changing signal evaluator
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
- `transformers` (FinBERT) or `vaderSentiment` for headline sentiment scoring is 3 lines in Python
- Deduplication (URL hash + title fuzzy-match) prevents GoClaw LLM from wasting tokens on repeated headlines
- `POST /enrich/news` input: `list[NewsItem]` → output: `list[EnrichedNewsItem]` with `sentiment_score`, `is_duplicate` flag
- GoClaw receives already-enriched, deduplicated items — LLM prompt is shorter, output quality is higher

**Indicator cache design:**
- Redis key: `indicators:{asset}:{candle_minutes}:{YYYY-MM-DD-HH-MM}` rounded to nearest 25 s, TTL = 25 s; different `candle_minutes` values produce separate cache entries
- Go evaluator: on cache hit → use cached value; on cache miss → ai-service fetches OHLCV, computes, stores, returns
- Prevents duplicate CryptoCompare API calls when multiple strategies evaluate in the same tick
- The cached response includes all indicator values computed for that candle size: `{ rsi, volume_spike_ratio, macd_line, macd_signal, macd_histogram, bb_upper, bb_mid, bb_lower, pct_change, price_stats, quote_currency }` — a single cache entry serves RSI, volume-spike, MACD, Bollinger, and intraday % change rules sharing the same asset + candle size; price-denominated fields (`bb_upper`, `bb_mid`, `bb_lower`, `price_stats.*`) are in the asset's `quote_currency`; `pct_change` is dimensionless (percentage)

**Python dependency stack:**
- `fastapi` + `uvicorn` — async HTTP
- `httpx` — async external API calls
- `pandas` + `pandas-ta` — indicator computation
- `numpy` — OHLCV aggregation
- `transformers` + `torch` (CPU-only, small model) OR `vaderSentiment` — sentiment; choose VADER for POC (no GPU needed, fast, good enough for headlines)
- `redis[asyncio]` — async Redis client
- `pydantic-settings` — config management
- `structlog` — JSON structured logging
- `pytest` + `pytest-asyncio` + `httpx` (test client) — testing

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
