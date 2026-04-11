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
Beyond the daily digest, the Agent Gateway runs two additional skills: **Anomaly Explanation**
enriches every fired alert with a short LLM-generated explanation of why the signal triggered
(fetches news + market context at fire time; non-blocking — alert delivers even if the LLM call
fails); **Market Sentiment Pulse** runs on a configurable cron (default every 4 h), classifies
overall market sentiment (bullish / neutral / bearish) from recent news via the LLM, stores the
score, and optionally pushes a short Telegram summary.

### Domain-Decoupled Platform Architecture

> **Architecture Decision:** [architecture-domain-decoupling.md](./architecture-domain-decoupling.md)

The codebase is structured as a **domain-decoupled platform** where cross-cutting capabilities
(authentication, billing, notification, event bus, AI agent orchestration) are isolated from
investment-specific business logic. This enables reuse of the platform layer for entirely
different domains (e.g., e-commerce price monitoring, healthcare alerts) without modifying
auth, billing, or agent code.

**Key structural rules:**
- **`platform/`** packages are domain-agnostic: auth, billing, notification, event bus, health, user profile, admin. They define **provider-agnostic interfaces**; they never import domain code or provider-specific SDKs in their interface/middleware files.
- **`platform/{concern}/{provider}/`** subdirectories contain the **swappable provider implementation** (e.g., `auth/supabase/`, `billing/stripe/`, `notification/telegram/`, `eventbus/nats/`). Each provider subdir imports only its own SDK. Swapping a provider (e.g., Supabase → Casdoor) requires only a new provider subdir + one wiring change in `cmd/server/main.go` — all middleware, handlers, and domain code remain unchanged.
- **`domain/investment/`** packages contain all investment-specific logic: strategies, signals, alerts, watchlist, market data adapters. They implement platform interfaces and register themselves at startup.
- **Agent Gateway** skills are split into reusable platform utilities and domain-specific agent definitions (digest persona, crypto-digest skill).
- **Python ai-service** separates a generic compute/enrichment/content platform from investment-specific indicator implementations and news adapters.
- **React frontend** separates platform pages (auth, billing, settings, admin) from domain pages (dashboard, strategies, alerts, watchlist).
- Platform tables use `platform_` namespace; domain tables use `app_` namespace; Agent Gateway uses `gc_` namespace.
- Swapping the domain requires writing a new `domain/` module and agent skills — zero changes to platform code.
- Swapping an infrastructure provider requires writing a new provider subdir — zero changes to platform interfaces, other providers, or domain code.

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
- **LLM (all AI features)**: Provider-agnostic — configurable via `LLM_PROVIDER` + `LLM_MODEL` env vars; POC default: Anthropic claude-3-5-haiku. ai-service wraps all LLM calls behind an `LLMProvider` interface (`ai-service/src/platform/llm/interfaces.py`); Agent Gateway uses its own pluggable LLM provider config. Swappable to OpenAI, Google Gemini, Mistral, Ollama, or any OpenAI-compatible API by adding a new provider module and changing env vars — no application code changes required
- **Crypto news API**: CryptoPanic free tier (50 req/day per token; DuckDuckGo fallback on quota
  exhaustion). Abstracted behind `NewsProvider` interface in `ai-service/src/news/`.
- **Market data feed**: CoinGecko Free API (no key required, ≤30 req/min) for spot price and
  percentage-change signals; CryptoCompare `histominute` endpoint (free tier, 100k req/month)
  for intraday OHLCV data forwarded to Python ai-service for indicator computation. Go adapter
  abstracted behind `domain/investment/marketdata.Provider`; RSI and OHLCV math happens in Python. All price
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

### Platform vs. Domain Ownership

> **Governing principle:** Platform code (`platform/`) never imports domain code (`domain/`). Domain code depends on platform interfaces only. See [architecture-domain-decoupling.md](./architecture-domain-decoupling.md).

### Ownership by Concern

| Concern | Owner | Layer | Rationale |
|---|---|---|---|
| REST API server, router, graceful shutdown | **Go backend** | **Platform** | Generic HTTP server lifecycle; domain modules register their routes at startup |
| JWT validation & user-context injection | **Go backend** | **Platform** | `platform/auth/interfaces.go` defines `AuthProvider`; `platform/auth/supabase/` implements it for POC; swap to Casdoor/Keycloak by adding a new provider dir — middleware and domain code unchanged |
| Stripe checkout, webhook processing, subscription gating | **Go backend** | **Platform** | `platform/billing/interfaces.go` defines `BillingProvider` + `SubscriptionChecker`; `platform/billing/stripe/` implements them for POC; swap to LemonSqueezy/Paddle by adding a new provider dir — gate middleware and domain code unchanged |
| Generic notification dispatch (route to Telegram/email/push) | **Go backend** | **Platform** | `platform/notification/interfaces.go` defines `Sender` + `AccountLinker` + `Dispatcher`; domain formats the `NotificationPayload`; platform routes it to the correct `Sender`; add Discord/Slack/Email by implementing `Sender` in a new provider dir |
| Telegram Bot API client + account linking | **Go backend** | **Platform** | `platform/notification/telegram/` implements `Sender` + `AccountLinker` interfaces; swappable — add `discord/` implementing the same interfaces to support Discord as a notification/linking channel |
| Generic event pub/sub (NATS JetStream abstracted) | **Go backend** | **Platform** | `platform/eventbus/interfaces.go` defines `Publisher` + `Consumer`; `platform/eventbus/nats/` implements them for POC; swap to Kafka/RabbitMQ by adding a new provider dir — domain code unchanged |
| User profile (timezone, settings) | **Go backend** | **Platform** | Domain-agnostic user preferences |
| Admin middleware + user management | **Go backend** | **Platform** | Generic admin operations |
| Health endpoints (`/health`, `/ready`) | **Go backend** | **Platform** | Infrastructure concern |
| PostgreSQL migrations | **Go backend** (`migrations/`) | **Both** | Platform migrations (`001-099`, `platform_*` tables); domain migrations (`100-199`, `app_*` tables); Agent Gateway uses `gc_` prefix |
| Signal evaluation loop (30 s ticker, threshold checks) | **Go backend** | **Domain** | Investment-specific: orchestrates tick, fetches pre-computed indicators, evaluates rules, publishes domain events via platform `EventPublisher` |
| Market data fetching (CoinGecko, CryptoCompare) | **Go backend** | **Domain** | Investment-specific data sources; domain-owned `DataFeedProvider` implementations |
| Strategy CRUD, signal rule validation, SDF | **Go backend** | **Domain** | Investment business logic; registered as domain routes via `DomainModule.RegisterRoutes()` |
| Alert persistence, history, dispatch formatting | **Go backend** | **Domain** | Investment alert records; formats `NotificationPayload` for platform dispatcher — domain creates the message, platform delivers it |
| Watchlist CRUD, digest content provision | **Go backend** | **Domain** | Investment watchlist; provides content for digest via `/internal/` API |
| Seed config loader (signal types, projects) | **Go backend** | **Domain** | Investment-specific configuration loaded at startup from `config/domain/investment/seed.yaml` |
| **Generic computation engine (cache, resample)** | **Python ai-service** | **Platform** | Reusable compute infrastructure; domain registers `ComputeHandler` implementations |
| **Generic content enrichment (sentiment, dedup)** | **Python ai-service** | **Platform** | VADER sentiment works for any text content; deduplication is generic |
| **Generic content provider interface** | **Python ai-service** | **Platform** | `ContentProvider` ABC; domain implements concrete adapters |
| **LLM provider abstraction** | **Python ai-service** | **Platform** | `LLMProvider` ABC in `platform/llm/interfaces.py`; `anthropic/`, `openai/`, `gemini/` provider modules; factory selects provider via `LLM_PROVIDER` env var; all domain code calls `LLMProvider` — never imports vendor SDK directly |
| **LLM observability (Langfuse)** | **Python ai-service + Langfuse** | **Platform** | `@observe` decorator on all `LLMProvider` methods; traces prompt/completion/cost/latency; Langfuse dashboard at `http://localhost:3100` |
| **Technical indicator computation (RSI, MACD, Bollinger, etc.)** | **Python ai-service** | **Domain** | `pandas-ta` / `numpy`; registered as compute handlers; accepts `candle_minutes` parameter to resample 1-min candles; cached at 25 s TTL |
| **News fetching (CryptoPanic, DuckDuckGo)** | **Python ai-service** | **Domain** | Concrete `ContentProvider` implementations; async `httpx`; quota tracking |
| **News enrichment routing** | **Python ai-service** | **Domain** | `POST /enrich/news` endpoint wires platform enrichment pipeline with domain-specific config |
| **Curated project registry** | **Python ai-service** | **Domain** | DB-backed; reads from `app_projects`; cached 5 min; `GET /projects` returns `{slug, display_name, symbol, coingecko_id, quote_currency, is_signal_asset}` |
| Agent Gateway framework (cron, LLM, Telegram, HTTP fetch) | **Agent Gateway** | **Platform** | Domain-agnostic orchestration capabilities; swappable via [agent-gateway-abstraction.md](./agent-gateway-abstraction.md) |
| Daily digest agent persona + crypto-digest skill | **Agent Gateway** | **Domain** | Investment-specific skill files in `agents/digest-agent/`; calls ai-service `GET /news/{slug}`, `POST /enrich/news`; uses LLM for summarisation |
| Anomaly explanation enrichment | **Agent Gateway** | **Domain** | Investment-specific: on signal fire, fetches news + market context via ai-service, invokes LLM to explain the cause, returns explanation text to Go dispatcher for appending to alert; non-blocking (alert delivers without explanation on failure) |
| Market sentiment pulse skill | **Agent Gateway** | **Domain** | Investment-specific skill in `agents/sentiment-agent/`; cron-scheduled (default every 4 h); fetches top N news items via ai-service, classifies sentiment via LLM, stores score via backend `/internal/sentiment` endpoint, optionally sends Telegram push |
| Telegram digest delivery | **Agent Gateway** | **Platform** | Agent Gateway Telegram channel; same bot as Go alert dispatcher (single bot, see §Single Telegram Bot) |
| UI component primitives (Button, Card, Input, etc.) | **React SPA** | **Platform** | Reusable design system with Storybook stories |
| Auth, billing, settings, admin pages | **React SPA** | **Platform** | Domain-agnostic user-facing pages |
| Dashboard, strategy, alert, watchlist pages | **React SPA** | **Domain** | Investment-specific UI |
| Routing, TLS termination, rate limiting | **Traefik** | **Platform** | Infrastructure concern |
| Metrics exposure & tracing | **All services** | **Platform** | Go `/metrics`, Agent Gateway OTLP, ai-service `/metrics` |

### Cross-Service Communication Rules

| From | To | Protocol | Auth |
|---|---|---|---|
| React SPA | Go backend | HTTPS REST (`/api/v1/`) | Supabase JWT (httpOnly cookie) |
| Agent Gateway digest agent | Go backend | HTTPS REST (`/internal/`) | Static bearer token (`AGENT_GATEWAY_INTERNAL_TOKEN`) |
| Agent Gateway digest agent | Python ai-service | HTTP fetch via `GET /news/{slug}`, `POST /enrich/news`, `GET /projects` | None (internal network only) |
| Agent Gateway anomaly agent | Python ai-service | HTTP fetch via `GET /news/{slug}`, `POST /indicators/{asset}` | None (internal network only) |
| Agent Gateway anomaly agent | Go backend | HTTPS REST (`/internal/alerts/:id/explanation`) | Static bearer token (`AGENT_GATEWAY_INTERNAL_TOKEN`) |
| Agent Gateway sentiment agent | Python ai-service | HTTP fetch via `GET /news/{slug}`, `GET /projects` | None (internal network only) |
| Agent Gateway sentiment agent | Go backend | HTTPS REST (`/internal/sentiment`) | Static bearer token (`AGENT_GATEWAY_INTERNAL_TOKEN`) |
| Go backend (signal evaluator) | Python ai-service | HTTP `POST /indicators/{asset}`, `POST /pct_change/{asset}` | None (internal network only) |
| Go backend (alert dispatcher) | Agent Gateway anomaly agent | Internal trigger (NATS → Go calls Agent Gateway HTTP or Agent Gateway subscribes to `signal.triggered.>`) | Internal |
| Go backend | NATS JetStream | NATS protocol | NATS credentials |
| Go backend | Telegram Bot API | HTTPS | `TELEGRAM_BOT_TOKEN` (shared single bot) |
| Agent Gateway | Telegram (digest + sentiment) | Agent Gateway Telegram channel | Same `TELEGRAM_BOT_TOKEN` configured as `AGENT_GATEWAY_TELEGRAM_BOT_TOKEN` |
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

### What Each Service / Layer MUST NOT Do

| Service / Layer | Must NOT |
|---|---|
| **Go backend — Platform (interfaces)** | Import any `domain/` package; import any provider subpackage directly (e.g., `platform/auth/middleware.go` MUST NOT import `platform/auth/supabase/`); call LLM APIs directly; know about strategies, signals, or market data; compute technical indicators |
| **Go backend — Platform (providers)** | Import another provider's SDK (e.g., `platform/auth/supabase/` MUST NOT import Stripe SDK); import `domain/` packages; contain business logic beyond adapter translation |
| **Go backend — Domain** | Import provider packages directly (use `platform/auth`, `platform/eventbus`, `platform/notification` interfaces only); import NATS/Stripe/Supabase/Telegram SDKs directly; implement auth or billing logic |
| **Python ai-service — Platform** | Know about crypto indicators, news providers, or project registries; contain investment-specific logic; import LLM vendor SDKs outside of `platform/llm/{provider}/` directories |
| **Python ai-service — Domain** | Evaluate signal threshold rules (Go domain logic); touch NATS; send Telegram messages; own user-facing Postgres tables; import LLM vendor SDKs directly (use `LLMProvider` interface via `platform/llm/`) |
| **Agent Gateway — Framework** | Contain domain business logic; connect to NATS; perform real-time alert dispatch; own Postgres migrations |
| **Agent Gateway — Domain Skills** | Bypass the framework's tool system; directly access databases; implement auth or billing |
| **React SPA — Platform** | Import domain components or pages; contain investment-specific business logic |
| **React SPA — Domain** | Implement auth flows (use platform hooks); call market data or news APIs directly; store secrets |
| **Traefik** | Contain application logic; be aware of NATS or Agent Gateway internals |

---

## Agent Gateway Architecture

The Agent Gateway is a **pluggable** component — any framework that satisfies the [Agent Gateway Abstraction](agent-gateway-abstraction.md) contract can be used. The default is GoClaw (`github.com/nextlevelbuilder/goclaw`), which replaces bespoke Python orchestration for the AI agent layer. Supported alternatives: OpenClaw, PicoClaw, nanoBot, ZeroClaw.

Key capabilities required from the gateway:

| Required Capability | POC Usage |
|---------------------|-----------|
| **Cron scheduling** | Daily digest trigger at 08:30 UTC; sentiment pulse every 4 h (configurable via `SENTIMENT_PULSE_CRON`) |
| **HTTP fetch tool** | Fetches news items from crypto news API per watchlist project; fetches market context for anomaly explanation |
| **LLM provider (configurable)** | Summarises raw news items; generates anomaly explanations; classifies market sentiment. Provider set via `LLM_PROVIDER` + `LLM_MODEL` env vars (POC default: Anthropic claude-3-5-haiku) |
| **Telegram channel** | Dispatches digest messages, sentiment pulse summaries, and (future) alert escalation |
| **Shared PostgreSQL** | Gateway shares the same Postgres instance; per-user agent contexts; sentiment scores stored via `/internal/sentiment` |
| **Message tool** | Sends formatted digest / sentiment summary to each user's Telegram chat |
| **Subagent / parallel execution** | Parallel HTTP fetch per watchlist project with automatic retry |
| **Scoped permissions** | Scope agent backend API access to specific `/internal/` endpoints (watchlist, users, sentiment) |
| **Observability (OTLP)** | LLM call tracing feeds both Prometheus/Grafana (generic metrics) and Langfuse (LLM-specific: prompt/completion logging, cost tracking, token usage) via OTLP HTTP endpoint (`/api/public/otel`) |

**Deployment**: The Agent Gateway runs as a **single Docker container** (dashboard at `http://localhost:18790`) alongside backend services, connected to the shared Postgres and Redis instances. NATS is not used by the gateway directly — the Go signal service publishes to NATS; the gateway operates on its own scheduler for digest and sentiment tasks.

### Single Telegram Bot Architecture

The system uses **one Telegram bot** (`TELEGRAM_BOT_TOKEN`) shared between Go backend and the Agent Gateway:

| Responsibility | Owner | Mechanism |
|---|---|---|
| Account linking (`/start?token=`) | **Go backend** | Telegram webhook registered at `https://api.<domain>/telegram/webhook`; Go processes `/start` deep-link updates and linking confirmation messages |
| Real-time alert delivery | **Go backend** | Direct `sendMessage` API call via bot token; latency-critical path (< 60 s SLA) |
| Daily digest delivery | **Agent Gateway** | Agent Gateway Telegram channel configured with the same bot token (`AGENT_GATEWAY_TELEGRAM_BOT_TOKEN = TELEGRAM_BOT_TOKEN`); uses message tool to call `sendMessage` |
| Sentiment pulse push | **Agent Gateway** | Same Telegram channel as digest; sends short sentiment summary when `sentiment_push_enabled = TRUE` for the user |

**Webhook routing**: Telegram delivers all bot updates (messages, commands) to the single registered webhook URL owned by Go backend. The Agent Gateway never receives inbound updates — it only **sends** outbound messages via `sendMessage`. This avoids the Telegram limitation of one webhook per bot. The Go backend MUST filter webhook updates to only process `/start` commands for account linking; all other inbound messages are ignored (no command router needed for POC).

**Configuration**: Both services read the same bot token from the environment. In `docker-compose.yml`, set `TELEGRAM_BOT_TOKEN` once and reference it in both Go backend and Agent Gateway service definitions (`AGENT_GATEWAY_TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}`).

### Anomaly Explanation Architecture

When a signal fires, the Go alert dispatcher enriches the alert with an LLM-generated explanation before sending the Telegram notification. The flow:

1. **Signal fires** → Go evaluator publishes `signal.triggered.{asset}` to NATS (existing flow)
2. **Alert dispatcher** consumes the event, persists the Alert record (existing flow)
3. **Explanation enrichment** (new step, non-blocking):
   - Dispatcher calls ai-service `GET /news/{asset}` to fetch recent news (reuses existing adapter)
   - Dispatcher calls ai-service `POST /indicators/{asset}` (or cached response) for 24 h market context
   - Dispatcher sends news items + market context + signal metadata to the Agent Gateway anomaly agent via an internal HTTP call (`POST /internal/explain`), which invokes the configured LLM provider (≤ 150 tokens) to generate a 1–2 sentence explanation
   - **Alternative (simpler)**: Go dispatcher calls `POST ai-service/explain/{asset}` — a new ai-service endpoint that wraps the LLM call directly via the `LLMProvider` interface, avoiding the Agent Gateway for this latency-sensitive path; the ai-service reads `LLM_PROVIDER`, `LLM_MODEL`, and `LLM_API_KEY` from env vars
4. **Timeout guard**: the entire explanation step is wrapped in a 10 s context deadline; on timeout, the explanation is `NULL` and the alert proceeds without it
5. **Persistence**: the explanation text is saved to `app_alerts.explanation` (new `TEXT` column, nullable)
6. **Telegram delivery**: the notification message includes the explanation appended below the standard alert fields (or omitted if `NULL`)

**LLM prompt design**: The anomaly explanation prompt includes: signal type + threshold, current value, asset name, 24 h price movement (OHLCV), and up to 5 recent news headlines. The system prompt instructs the LLM to produce exactly 1–2 sentences explaining the likely cause, and to state "no specific news catalyst identified" if no relevant news is provided.

**Metrics**: `anomaly_explanation_latency_seconds` (histogram), `anomaly_explanation_failures_total` (counter by failure reason: timeout, llm_error, news_fetch_error).

### Market Sentiment Pulse Architecture

A lightweight scheduled Agent Gateway skill that provides periodic market sentiment classification.

**Flow**:

1. **Cron trigger**: Agent Gateway fires the sentiment-pulse skill at the configured interval (env `SENTIMENT_PULSE_CRON`, default `0 */4 * * *` = every 4 h)
2. **Per-user processing**: the skill calls `GET /internal/users?has_watchlist=true` to get users with active watchlists
3. **News fetch**: for each user, the skill calls ai-service `GET /news/{slug}` for each watchlisted project (parallelised via subagent `waitAll`), collects the top N items (env `SENTIMENT_NEWS_LIMIT`, default 10)
4. **Sentiment classification**: the skill passes the collected news items to the configured LLM provider (≤ 100 tokens) with a structured prompt requesting JSON output: `{ "score": "bullish|neutral|bearish", "confidence": 0.0-1.0, "summary": "..." }`
5. **Persistence**: the skill calls `POST /internal/sentiment` on the Go backend to store the score in `app_sentiment_scores` (new table: `id`, `user_id`, `score` enum, `confidence` numeric, `summary` text, `news_item_count` int, `scored_at` timestamp); index: `idx_sentiment_user_scored(user_id, scored_at DESC)`
6. **Optional Telegram push**: if `app_users.sentiment_push_enabled = TRUE`, the skill sends a short Telegram message via the message tool (e.g., "🟢 Bullish — ETF momentum + SOL DeFi TVL growth driving positive sentiment")

**Configuration**:
- `SENTIMENT_PULSE_CRON`: cron expression (default `0 */4 * * *`)
- `SENTIMENT_NEWS_LIMIT`: max news items per pulse (default `10`)
- `app_users.sentiment_push_enabled`: per-user opt-in for Telegram push (default `FALSE`)

**Database changes**:
- New table `app_sentiment_scores` (migration `008_sentiment.sql`)
- New column `app_users.sentiment_push_enabled BOOLEAN DEFAULT FALSE` (migration `008a_users_sentiment.sql`)
- RLS enabled on `app_sentiment_scores`, scoped to `user_id`

**API endpoints** (Go backend):
- `POST /internal/sentiment` — Agent Gateway stores a new score (internal auth)
- `GET /api/v1/sentiment?limit=N` — authenticated user retrieves their sentiment history (default limit 20, max 100)

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
│   ├── cmd/server/main.go          # Wires providers + platform + domain; starts server
│   │
│   ├── platform/                    # DOMAIN-AGNOSTIC (reusable across projects)
│   │   ├── auth/                    # Authentication & authorization (provider-agnostic)
│   │   │   ├── interfaces.go       # AuthProvider, TokenValidator (provider-agnostic contracts)
│   │   │   ├── middleware.go        # Auth middleware (uses AuthProvider interface — no provider knowledge)
│   │   │   ├── handler.go          # /auth/* routes (delegates to AuthProvider)
│   │   │   └── supabase/           # ← SWAPPABLE: Supabase provider (implements AuthProvider)
│   │   │       ├── client.go       # Supabase SDK client
│   │   │       └── config.go       # Supabase-specific config
│   │   │       # Future: casdoor/, keycloak/, auth0/ — same AuthProvider interface
│   │   ├── billing/                 # Subscription & payment (provider-agnostic)
│   │   │   ├── interfaces.go       # BillingProvider, SubscriptionChecker (provider-agnostic contracts)
│   │   │   ├── gate.go             # Subscription gate middleware (uses SubscriptionChecker interface)
│   │   │   └── stripe/             # ← SWAPPABLE: Stripe provider (implements BillingProvider)
│   │   │       ├── client.go       # Stripe SDK client
│   │   │       ├── checkout.go     # Stripe-specific checkout session
│   │   │       ├── webhook.go      # Stripe-specific webhook + signature validation
│   │   │       └── config.go       # Stripe-specific config
│   │   │       # Future: lemonsqueezy/, paddle/ — same BillingProvider interface
│   │   ├── notification/            # Generic notification dispatch (provider-agnostic)
│   │   │   ├── interfaces.go       # NotificationPayload, Sender, AccountLinker, Dispatcher
│   │   │   ├── dispatcher.go       # Route payload → correct Sender by channel
│   │   │   └── telegram/           # ← SWAPPABLE: Telegram channel (implements Sender + AccountLinker)
│   │   │       ├── sender.go       # Telegram Bot API sender
│   │   │       ├── linker.go       # Account linking (deep-link, webhook, confirm)
│   │   │       ├── handler.go      # /telegram/* webhook routes
│   │   │       └── config.go       # Telegram-specific config
│   │   │       # Future: discord/, slack/, email/ — each implements Sender
│   │   ├── eventbus/                # Generic event pub/sub (provider-agnostic)
│   │   │   ├── interfaces.go       # Publisher, Consumer, Handler (provider-agnostic contracts)
│   │   │   └── nats/               # ← SWAPPABLE: NATS JetStream (implements Publisher, Consumer)
│   │   │       ├── client.go       # NATS JetStream implementation
│   │   │       ├── stream.go       # Stream/consumer config helpers
│   │   │       └── config.go       # NATS-specific config
│   │   │       # Future: kafka/, rabbitmq/ — same Publisher/Consumer interface
│   │   ├── health/                  # /health, /ready endpoints
│   │   ├── user/                    # User profile (timezone, settings — domain-agnostic)
│   │   ├── admin/                   # Admin middleware + generic user management
│   │   ├── server/                  # HTTP server setup, graceful shutdown, router mounting
│   │   │   ├── server.go           # Server struct with DomainModule registration
│   │   │   └── router.go           # Platform route registration
│   │   └── config/                  # Platform config (env vars, non-domain settings)
│   │
│   ├── domain/                      # DOMAIN-SPECIFIC (swappable)
│   │   └── investment/              # Investment intelligence domain module
│   │       ├── register.go          # Register(platform) — mounts routes, consumers, workers
│   │       ├── strategies/          # Strategy CRUD, signal rule validation, SDF
│   │       │   ├── service.go
│   │       │   ├── handler.go
│   │       │   ├── validator.go
│   │       │   └── import_handler.go
│   │       ├── signals/             # Signal evaluator (30 s polling loop)
│   │       │   ├── evaluator.go
│   │       │   ├── poller.go
│   │       │   └── cooldown.go
│   │       ├── alerts/              # Alert persistence, history queries, dispatch formatting
│   │       │   ├── handler.go
│   │       │   ├── dispatcher.go    # NATS consumer → format NotificationPayload → platform
│   │       │   └── redriver.go
│   │       ├── watchlist/           # Watchlist CRUD, digest content provision
│   │       │   └── handler.go
│   │       ├── marketdata/          # Market data feed adapter (interface + impl)
│   │       │   ├── interfaces.go    # DataFeedProvider
│   │       │   ├── coingecko.go
│   │       │   └── cryptocompare.go
│   │       └── config/              # Seed config loader (signal types, project seeds)
│   │           └── seed.go
│   │
│   ├── pkg/                         # Shared utilities (domain-agnostic)
│   │   ├── httputil/                # HTTP helpers, error response envelope
│   │   ├── validate/                # JSON Schema validation helpers
│   │   └── testutil/                # Test fixtures, DB helpers
│   │
│   └── tests/
│       ├── platform/                # Platform contract + integration tests
│       │   ├── contract/            # auth, billing, notification contract tests
│       │   └── integration/         # Supabase, Stripe webhook integration tests
│       └── domain/
│           └── investment/          # Domain-specific tests
│               ├── contract/        # strategies, alerts, watchlist contract tests
│               ├── integration/     # signal notification, strategy lifecycle tests
│               └── unit/            # evaluator, validator unit tests
│
├── ai-service/                      # Python AI/data service
│   ├── src/
│   │   ├── main.py                  # FastAPI app factory; mounts platform + domain routers
│   │   ├── config.py                # Settings via pydantic-settings (env vars, .env)
│   │   │
│   │   ├── platform/                # DOMAIN-AGNOSTIC (reusable)
│   │   │   ├── llm/                 # LLM provider abstraction (provider-agnostic)
│   │   │   │   ├── interfaces.py    # LLMProvider ABC, LLMResponse dataclass, LLMConfig
│   │   │   │   ├── factory.py       # create_llm_provider(config) → LLMProvider (reads LLM_PROVIDER env)
│   │   │   │   ├── anthropic/       # ← SWAPPABLE: Anthropic provider (POC default)
│   │   │   │   │   ├── client.py    # AnthropicProvider(LLMProvider) — httpx-based
│   │   │   │   │   └── config.py    # Anthropic-specific config (API key, base URL)
│   │   │   │   ├── openai/          # ← SWAPPABLE: OpenAI / OpenAI-compatible provider
│   │   │   │   │   ├── client.py    # OpenAIProvider(LLMProvider)
│   │   │   │   │   └── config.py
│   │   │   │   └── gemini/          # ← SWAPPABLE: Google Gemini provider
│   │   │   │       ├── client.py    # GeminiProvider(LLMProvider)
│   │   │   │       └── config.py
│   │   │   │       # Future: mistral/, ollama/, bedrock/ — same LLMProvider interface
│   │   │   ├── compute/             # Generic computation engine
│   │   │   │   ├── interfaces.py    # ComputeEngine ABC, ComputeHandler ABC
│   │   │   │   ├── cache.py         # Redis-backed result cache (generic key pattern)
│   │   │   │   └── resample.py      # Generic OHLCV resampling utility
│   │   │   ├── enrichment/          # Generic content enrichment pipeline
│   │   │   │   ├── interfaces.py    # ContentEnricher ABC
│   │   │   │   └── sentiment.py     # VADER sentiment (reusable for any text)
│   │   │   ├── content/             # Generic content provider
│   │   │   │   └── interfaces.py    # ContentProvider ABC, ContentItem dataclass
│   │   │   └── health.py            # GET /health (liveness) + GET /health/ready (readiness)
│   │   │
│   │   └── domain/                  # DOMAIN-SPECIFIC (swappable)
│   │       └── investment/
│   │           ├── register.py      # register(app) — mounts domain routers
│   │           ├── indicators/      # Indicator computation implementations
│   │           │   ├── rsi.py       # RSI compute handler
│   │           │   ├── volume_spike.py
│   │           │   ├── macd.py
│   │           │   ├── bollinger.py
│   │           │   ├── price_stats.py
│   │           │   ├── pct_change.py
│   │           │   ├── router.py    # POST /indicators/{asset}
│   │           │   ├── pct_change_router.py  # POST /pct_change/{asset}
│   │           │   └── cache.py     # Redis-backed indicator cache (TTL = 25 s)
│   │           ├── news/            # News content providers
│   │           │   ├── provider.py  # Abstract NewsProvider + NewsItem dataclass
│   │           │   ├── cryptopanic.py   # CryptoPanic adapter (primary)
│   │           │   ├── duckduckgo.py    # DuckDuckGo fallback adapter
│   │           │   ├── quota.py     # Daily quota counter (Redis-backed)
│   │           │   └── router.py    # GET /news/{project_slug}
│   │           ├── enrichment/      # Investment-specific enrichment
│   │           │   └── router.py    # POST /enrich/news
│   │           └── projects/        # Crypto project registry
│   │               ├── registry.py  # DB-backed slug→{display_name, symbol, ...} map
│   │               └── router.py    # GET /projects
│   └── tests/
│       ├── platform/                # Platform compute/enrichment tests
│       └── domain/
│           └── investment/
│               ├── contract/        # HTTP contract tests (all domain endpoints)
│               ├── integration/     # Redis, CryptoPanic, indicator pipeline tests
│               └── unit/            # Adapter parsing, RSI math, sentiment, quota logic
│
├── agent-gateway/                   # Agent Gateway configuration (pluggable)
│   ├── goclaw/                      # GoClaw-specific config (DEFAULT)
│   │   ├── docker-compose.goclaw.yml
│   │   ├── .env.goclaw
│   │   └── agents/
│   │       ├── _platform/           # DOMAIN-AGNOSTIC agent utilities (reusable)
│   │       │   └── HEARTBEAT.md
│   │       ├── digest-agent/        # DOMAIN-SPECIFIC: investment digest
│   │       │   ├── AGENT.md         # Investment persona + instructions
│   │       │   └── skills/
│   │       │       └── crypto-digest.md
│   │       ├── anomaly-agent/       # DOMAIN-SPECIFIC: anomaly explanation enrichment
│   │       │   ├── AGENT.md         # Persona: explains why a signal fired
│   │       │   └── skills/
│   │       │       └── anomaly-explanation.md
│   │       └── sentiment-agent/     # DOMAIN-SPECIFIC: market sentiment pulse
│   │           ├── AGENT.md         # Persona: classifies market sentiment
│   │           └── skills/
│   │               └── sentiment-pulse.md
│   ├── openclaw/
│   ├── picoclaw/
│   ├── nanobot/
│   └── zeroclaw/
│
├── frontend/                        # React SPA
│   ├── src/
│   │   ├── platform/                # DOMAIN-AGNOSTIC (reusable)
│   │   │   ├── components/          # Design system primitives + shared components
│   │   │   │   └── *.stories.tsx    # Storybook story files
│   │   │   ├── pages/
│   │   │   │   ├── auth/            # SignIn, SignUp, ForgotPassword
│   │   │   │   ├── settings/        # Telegram linking, timezone, account
│   │   │   │   ├── billing/         # Subscription management (Stripe)
│   │   │   │   └── admin/           # Admin: user management
│   │   │   ├── hooks/               # useAuth, useSubscription, useNotification
│   │   │   ├── services/            # Platform API client (TanStack Query hooks)
│   │   │   └── lib/                 # Utilities, design tokens
│   │   │
│   │   └── domain/                  # DOMAIN-SPECIFIC (swappable)
│   │       └── investment/
│   │           ├── pages/
│   │           │   ├── dashboard/   # Strategy list + quick stats
│   │           │   ├── strategies/  # Strategy create/edit, signal rule builder
│   │           │   ├── alerts/      # Alert history view
│   │           │   └── watchlist/   # Watchlist management
│   │           ├── components/      # Domain-specific components
│   │           │   └── *.stories.tsx
│   │           └── services/        # Domain API client hooks
│   ├── .storybook/                  # Storybook config (main.ts, preview.ts)
│   └── tests/
│       ├── unit/                    # Vitest + React Testing Library
│       └── e2e/                     # Playwright
│
├── infra/
│   ├── docker-compose.yml           # Local dev: all services
│   ├── docker-compose.prod.yml      # Production overrides
│   ├── docker-compose.langfuse.yml  # Langfuse LLM observability (opt-in: langfuse-web + langfuse-worker + clickhouse)
│   ├── traefik/                     # Traefik static + dynamic config
│   └── monitoring/                  # Prometheus scrape config, Grafana dashboards
│
├── config/                          # Externalised business configuration
│   ├── platform.env.example         # Platform config template (auth, billing, infra)
│   └── domain/
│       └── investment/
│           ├── seed.yaml            # Signal types, project seed data, system tunables (FR-026)
│           └── seed.schema.json     # JSON Schema for seed.yaml validation
│
├── contracts/                       # Portable format schemas
│   └── strategy-definition.schema.json  # Strategy Definition Format JSON Schema (FR-027)
│
└── migrations/                      # PostgreSQL migrations (golang-migrate)
    ├── 001-099: platform_*          # Platform schema (users, notifications, subscriptions)
    └── 100-199: app_*               # Domain schema (strategies, alerts, watchlist, projects)
```

**Structure Decision**: Domain-decoupled platform architecture with provider abstraction (backend + AI service + frontend + agent gateway).
Chosen because the platform capabilities (auth, billing, notification, event bus, AI agents) are
domain-agnostic and must be reusable for non-investment domains in the future. Additionally, each
platform concern separates **provider-agnostic interfaces** from **swappable provider implementations**
(Supabase, Stripe, Telegram, NATS, Anthropic are POC providers — each can be swapped to Casdoor, LemonSqueezy,
Discord, Kafka, OpenAI/Gemini/Mistral, etc. by adding a new provider subdirectory and changing one wiring line in `main.go` or env vars).
The investment-specific business logic is isolated in `domain/investment/` directories across all
services, connected to the platform via well-defined interfaces. See
[architecture-domain-decoupling.md](./architecture-domain-decoupling.md) for the full architecture
decision record with interface definitions, provider abstraction patterns, and migration rules.

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
- LLM provider configured via `AGENT_GATEWAY_LLM_API_KEY` env var + `AGENT_GATEWAY_LLM_PROVIDER` (default `anthropic`) + `AGENT_GATEWAY_LLM_MODEL` (default `claude-3-5-haiku`); model set per-agent in AGENT.md
- The Agent Gateway's built-in Telegram channel handles bot token + webhook; set `AGENT_GATEWAY_TELEGRAM_BOT_TOKEN`

### Signal Evaluation
- 30 s polling loop implemented as a Go ticker in `domain/investment/signals/`; one goroutine per active
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
  - Both streams initialised explicitly on backend startup via `platform/eventbus/` — do NOT rely on NATS auto-create defaults
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
- Abstracted behind `domain/investment/marketdata.Provider` interface — swap without changing signal evaluator
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
- `httpx` — async external API calls (LLM providers, news APIs)
- `pandas` + `pandas-ta` — indicator computation
- `numpy` — OHLCV aggregation
- `vaderSentiment` — headline sentiment scoring (fast, CPU-only, no GPU needed; sufficient for POC)
- `redis[asyncio]` — async Redis client
- `pydantic-settings` — config management
- `structlog` — JSON structured logging
- `langfuse` — LLM observability (trace all LLM calls with `@observe` decorator; prompt/completion logging, per-user cost tracking, prompt versioning)
- `anthropic` — Anthropic SDK (POC default LLM provider; loaded dynamically by `platform/llm/anthropic/`)
- `openai` — OpenAI SDK (optional; loaded dynamically by `platform/llm/openai/` when `LLM_PROVIDER=openai`)
- `google-genai` — Google Gemini SDK (optional; loaded dynamically by `platform/llm/gemini/` when `LLM_PROVIDER=gemini`)
- `pytest` + `pytest-asyncio` + `httpx` (test client) — testing

### Crypto News API
- **Selected for POC**: CryptoPanic free tier (50 req/day per token, returns news by currency filter)
- Agent Gateway digest agent calls HTTP fetch with CryptoPanic API endpoint per watchlist project,
  passes results to the configured LLM for summarisation (≤ 200 tokens per project)
- Fallback: if CryptoPanic quota exceeded, the Agent Gateway uses web search (DuckDuckGo) as fallback

### Supabase Auth
- Supabase Auth handles JWT issuance; Go backend validates JWTs using Supabase public key
- Row Level Security (RLS) on all tables ensures users can only read/write their own rows
- Email verification and password reset are built into Supabase Auth — no custom flow required
  (FR-020 is satisfied by delegating these flows to Supabase; `platform/auth/` only needs JWT
  validation middleware and user-context injection)

### Stripe
- POC uses a single subscription product with one price (monthly flat fee)
- Stripe webhook events consumed by `platform/billing/`: `customer.subscription.created`,
  `customer.subscription.deleted`, `invoice.payment_failed`
- Subscription status stored on `users` table; middleware checks status on protected routes

### LLM Provider Abstraction

The project follows the same provider-abstraction pattern used for auth (Supabase→Casdoor), billing (Stripe→LemonSqueezy), notification (Telegram→Discord), and event bus (NATS→Kafka). All LLM interactions go through a **provider-agnostic `LLMProvider` interface** — no service directly imports a vendor SDK.

**Interface** (`ai-service/src/platform/llm/interfaces.py`):
```python
class LLMProvider(ABC):
    async def complete(self, *, system: str, user: str, max_tokens: int, temperature: float = 0.3,
                       response_format: str | None = None) -> LLMResponse: ...
    async def complete_json(self, *, system: str, user: str, max_tokens: int,
                            json_schema: dict) -> dict: ...
    @property
    def model_name(self) -> str: ...
    @property
    def provider_name(self) -> str: ...
```

**Factory** (`ai-service/src/platform/llm/factory.py`): reads `LLM_PROVIDER` env var, dynamically imports the matching provider module, returns an `LLMProvider` instance. Providers are registered via a simple dict mapping `{"anthropic": "platform.llm.anthropic.client:AnthropicProvider", "openai": "platform.llm.openai.client:OpenAIProvider", "gemini": "platform.llm.gemini.client:GeminiProvider"}`.

**Environment variables** (ai-service):
| Variable | Purpose | Default |
|----------|---------|---------|
| `LLM_PROVIDER` | Provider name (anthropic, openai, gemini) | `anthropic` |
| `LLM_MODEL` | Model identifier | `claude-3-5-haiku-latest` |
| `LLM_API_KEY` | Provider API key | (required) |
| `LLM_BASE_URL` | Override base URL (for local proxies, Azure OpenAI, etc.) | Provider default |
| `LLM_MAX_RETRIES` | Max retry count for transient errors | `2` |
| `LLM_TIMEOUT_SECONDS` | Per-request timeout | `30` |

**Agent Gateway LLM config** (mapped per gateway):
| Variable | Purpose | Default |
|----------|---------|---------|
| `AGENT_GATEWAY_LLM_PROVIDER` | Gateway LLM provider | `anthropic` |
| `AGENT_GATEWAY_LLM_MODEL` | Gateway LLM model | `claude-3-5-haiku-latest` |
| `AGENT_GATEWAY_LLM_API_KEY` | Gateway LLM API key | (required) |

**Design decisions:**
- Provider SDKs (`anthropic`, `openai`, `google-genai`) are **optional dependencies** — only the configured provider's SDK needs to be installed; the factory raises a clear error if the SDK is missing
- The `LLMProvider` interface includes both `complete()` (free-text) and `complete_json()` (structured output) methods — sentiment classification uses `complete_json()` for reliable JSON parsing
- All LLM calls are instrumented with Langfuse `@observe` decorator for cost/latency/quality tracking regardless of provider
- Temperature, max_tokens, and response format are passed per-call (not fixed at provider level) because different skills have different requirements (digest = creative, sentiment = deterministic)

### Langfuse — LLM Observability

**Selected for POC**: Langfuse (self-hosted, MIT license) — purpose-built LLM observability platform that complements the existing generic OTLP→Prometheus→Grafana telemetry stack.

**Why Langfuse in addition to Prometheus/Grafana?**
The existing Agent Gateway OTLP pipeline provides generic request/latency/error metrics. Langfuse fills LLM-specific observability gaps:

| Capability | OTLP → Prometheus | Langfuse |
|---|---|---|
| Request latency / error rate | ✅ | ✅ |
| Full prompt + completion logging | ❌ | ✅ |
| Per-user / per-skill cost tracking | ❌ | ✅ (auto-calculated by model) |
| Prompt version management + A/B eval | ❌ | ✅ |
| Nested agent trace visualisation | ❌ | ✅ |
| Quality scoring (manual + automated) | ❌ | ✅ |
| LLM playground (test prompts live) | ❌ | ✅ |
| Token usage breakdown | ❌ | ✅ |

**Architecture** (self-hosted via Docker Compose):
- `langfuse-web` — Next.js web UI + API server (dashboard at `http://localhost:3100`)
- `langfuse-worker` — async background worker for trace processing
- `clickhouse` — OLAP storage for traces and observations (high-volume, columnar)
- Shared Postgres — reuses the project's existing Postgres instance (Langfuse schema in `langfuse_` prefix, separate from `app_` / `gc_` / `platform_` prefixes)
- Redis — reuses the project's existing Redis instance (Langfuse uses a dedicated key prefix)
- S3/MinIO — optional blob storage for large payloads (not required for POC scale)

**Integration points** (dual-path architecture — all LLM calls from both services appear in the same Langfuse dashboard):
1. **ai-service** (Python — first-class SDK path): `langfuse` Python SDK v3 with `@observe` decorator on all `LLMProvider` methods — traces every LLM call with model, tokens, cost, latency, user_id, skill_name. Full feature set: prompt versioning, playground, scoring, nested trace visualisation
2. **Agent Gateway** (any language — OTLP path): Agent Gateway (GoClaw, OpenClaw, ZeroClaw, etc.) exports OTLP traces to Langfuse via the OTLP-compatible HTTP endpoint at `http://langfuse-web:3100/api/public/otel`. Langfuse auto-maps `gen_ai.*` semantic convention attributes (model name, token usage, cost) into its LLM-specific data model. This works for **any OTLP-capable agent framework regardless of language** (Go, Rust, TypeScript, Python). Two sub-options:
   - **Direct export**: configure the gateway's OTLP exporter to point at Langfuse's `/api/public/otel` endpoint (simplest; requires gateway to support OTLP over HTTP)
   - **OTel Collector fan-out**: gateway exports OTLP to an OpenTelemetry Collector, which fans out to both Prometheus (generic metrics) and Langfuse (LLM traces) — recommended when you also want Prometheus metrics from the same spans
3. **Quality scoring**: manual scoring via Langfuse UI during POC; automated scoring (e.g., checking sentiment JSON schema validity) as a post-POC enhancement

**OTLP → Langfuse attribute mapping** (Agent Gateway must set these span attributes for full Langfuse feature support):
| OTel Attribute | Langfuse Field | Required? |
|---|---|---|
| `gen_ai.request.model` | Generation model name | ✅ Yes |
| `gen_ai.usage.input_tokens` | Input token count | ✅ Yes |
| `gen_ai.usage.output_tokens` | Output token count | ✅ Yes |
| `gen_ai.usage.cost` | Cost in USD | Optional (Langfuse auto-calculates from model) |
| `gen_ai.prompt` | Prompt / input text | Recommended |
| `gen_ai.completion` | Completion / output text | Recommended |
| `gen_ai.system` | Provider name (e.g., `anthropic`) | Recommended |
| `langfuse.user.id` | Per-user attribution | Recommended |
| `langfuse.session.id` | Session grouping | Optional |
| `langfuse.trace.metadata.skill_name` | Skill name (digest, sentiment) | Recommended |

**Langfuse OTLP limitations** (vs. Python/JS SDK):
- Prompt management / playground: available via SDK only; OTLP path provides observability but not prompt CRUD
- Quality scoring: OTLP traces can be scored post-hoc via Langfuse REST API; SDK provides inline scoring helpers
- Supports OTLP over HTTP only (`HTTP/JSON` and `HTTP/protobuf`); gRPC not yet supported

**Environment variables** (ai-service + docker-compose):
| Variable | Purpose | Default |
|----------|---------|---------|
| `LANGFUSE_PUBLIC_KEY` | Langfuse project public key | (required) |
| `LANGFUSE_SECRET_KEY` | Langfuse project secret key | (required) |
| `LANGFUSE_HOST` | Langfuse server URL | `http://langfuse-web:3100` |
| `LANGFUSE_ENABLED` | Enable/disable tracing (disable for tests) | `true` |

**Agent Gateway → Langfuse OTLP config** (added to each gateway `.env` file):
| Variable | Purpose | Default |
|----------|---------|--------|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Langfuse OTLP endpoint (or OTel Collector) | `http://langfuse-web:3100/api/public/otel` |
| `OTEL_EXPORTER_OTLP_HEADERS` | Basic Auth header (`Authorization=Basic <base64(pk:sk)>`) | (required) |

For OTel Collector fan-out, the gateway exports to the collector at `http://otel-collector:4318` and the collector config includes both `otlphttp/langfuse` and `prometheusremotewrite` exporters.

**Cost tracking**: Langfuse automatically calculates cost per trace based on model pricing tables (built-in for Anthropic, OpenAI, Gemini, Mistral). Per-user cost attribution is achieved by passing `user_id` to `langfuse.trace()` (Python SDK) or setting `langfuse.user.id` span attribute (OTLP path). Dashboard shows: total cost by model, cost per skill (digest, anomaly, sentiment), cost per user, cost trend over time — unified across both ai-service (Python SDK) and Agent Gateway (OTLP) traces.

### Anomaly Explanation
- **LLM model**: Configurable via `LLM_PROVIDER` + `LLM_MODEL` env vars (POC default: Anthropic claude-3-5-haiku); ai-service calls LLM via the `LLMProvider` interface, not directly via Anthropic SDK — latency-sensitive path calls ai-service directly (avoids Agent Gateway scheduling overhead)
- **Token budget**: ≤ 150 output tokens; input = signal metadata (~50 tokens) + up to 5 news headlines (~200 tokens) + 24 h OHLCV summary (~50 tokens) = ~300 input tokens → total ~450 tokens per explanation
- **Cost estimate** (POC default model: claude-3-5-haiku): ~450 tokens × $0.25/MTok (haiku input) + 150 × $1.25/MTok (haiku output) ≈ $0.0003/explanation; at 100 alerts/day = ~$0.03/day. Costs vary by LLM provider; tracked via Langfuse per-model cost dashboards
- **Latency budget**: 10 s hard timeout; typical haiku response in 1–3 s; news fetch cached from recent digest/sentiment runs reduces cold-start
- **Fallback strategy**: timeout → deliver alert without explanation; news fetch failure → use market-context-only prompt; LLM error → deliver alert without explanation; all failures logged + metriced
- **ai-service endpoint**: `POST /explain/{asset}` — new endpoint in `ai-service/src/enrichment/explanation.py`; accepts `{ signal_type, threshold, current_value, news_items[], price_stats }`, calls LLM via `LLMProvider` interface, returns `{ explanation: string }`; cached at `explanation:{asset}:{signal_rule_id}:{minute}` with TTL = 55 s (prevents duplicate LLM calls if the same rule fires on consecutive ticks within cooldown edge cases)

### Market Sentiment Pulse
- **LLM model**: Configurable via Agent Gateway LLM provider config (POC default: Anthropic claude-3-5-haiku); same provider abstraction as digest — not latency-sensitive, so gateway scheduling overhead is acceptable
- **Token budget**: ≤ 100 output tokens; input = up to 10 news headlines (~400 tokens) + structured prompt (~100 tokens) = ~500 input tokens → total ~600 tokens per pulse per user
- **Cost estimate** (POC default model: claude-3-5-haiku): ~600 tokens × $0.25/MTok (haiku input) + 100 × $1.25/MTok (haiku output) ≈ $0.0003/pulse/user; at 50 users × 6 pulses/day = ~$0.09/day. Costs vary by LLM provider; tracked via Langfuse per-model cost dashboards
- **CryptoPanic quota impact**: sentiment pulse reuses the same news adapter as digest; at 6 pulses/day × 50 users × ~2 projects avg = 600 CryptoPanic calls/day — exceeds 50 req/day free tier; **mitigation**: (a) cache news responses at ai-service level with 30 min TTL (sentiment pulse doesn't need real-time freshness); (b) share cached news across users with overlapping watchlists; (c) DuckDuckGo fallback on quota exhaustion (existing adapter); (d) consider a single "global" sentiment pulse that evaluates market-wide sentiment once and stores it for all users, rather than per-user evaluation — reduces LLM + news API calls from O(users) to O(1)
- **Structured output**: LLM prompt requests JSON `{ "score": "bullish|neutral|bearish", "confidence": 0.0-1.0, "summary": "..." }`; Agent Gateway skill parses the response and rejects malformed output (retries once); if both attempts fail, score is recorded as `neutral` with confidence `0.0`
- **Sentiment trend**: over time, stored sentiment scores enable a trend line visible on the dashboard (post-POC UI enhancement); the API `GET /sentiment?limit=N` returns the most recent scores for chart rendering
