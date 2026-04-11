# Architecture Decision: Domain-Decoupled Platform Design

> **Status:** Accepted  
> **Created:** 2026-04-11  
> **Decision Maker:** Architecture Review  
> **Scope:** All services (Go backend, Python ai-service, Agent Gateway, React frontend)

---

## 1. Context & Motivation

The current plan tightly couples cross-cutting platform capabilities (authentication, payment processing, AI agent orchestration) with investment-domain business logic. This makes it expensive to reuse these capabilities for a different domain (e.g., e-commerce monitoring, healthcare alerts, logistics tracking).

**Goal:** Restructure the codebase so that:

1. **Authentication & Authorization** — a domain-agnostic platform layer; knows nothing about strategies, alerts, or watchlists.
2. **Payment Processing** — a domain-agnostic billing layer; knows nothing about signal rules or crypto assets.
3. **AI Agent Orchestration** — a domain-agnostic agent framework layer; knows nothing about financial instruments, market data, or digest content.
4. **Domain Business Logic** — all investment-specific concepts (strategies, signals, alerts, watchlists, market data, indicators, news) live in an isolated domain layer that plugs into the platform layers via well-defined interfaces.

In the future, swapping the domain layer (e.g., from "investment intel" to "e-commerce price monitor") should require **zero changes** to auth, billing, or agent orchestration code.

---

## 2. Design Principles

### P1 — Dependency Inversion

Platform layers define **interfaces** (Go interfaces, Python ABCs, TypeScript types). Domain layers provide **implementations**. Platform code never imports domain code directly.

```
┌─────────────────────────────────────────────────────────┐
│                    React Frontend                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Platform UI  │  │  Domain UI   │  │  Platform UI  │  │
│  │  (Auth pages, │  │ (Strategies, │  │  (Billing     │  │
│  │   Settings)   │  │  Alerts,     │  │   pages)      │  │
│  └──────────────┘  │  Watchlist)   │  └──────────────┘  │
│                    └──────────────┘                      │
└─────────────────────────────────────────────────────────┘
                          │
                     HTTP REST API
                          │
┌─────────────────────────────────────────────────────────┐
│                    Go Backend                           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Platform Layer (domain-agnostic)    │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │    │
│  │  │  Auth     │  │ Billing  │  │ Notification │  │    │
│  │  │ (Supabase │  │ (Stripe  │  │ (Telegram    │  │    │
│  │  │  JWT,     │  │  webhook,│  │  Bot API,    │  │    │
│  │  │  RLS,     │  │  checkout│  │  retry,      │  │    │
│  │  │  session) │  │  gate)   │  │  queuing)    │  │    │
│  │  └──────────┘  └──────────┘  └──────────────┘  │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │    │
│  │  │ EventBus │  │ Health   │  │ User Profile │  │    │
│  │  │ (NATS    │  │ (liveness│  │ (timezone,   │  │    │
│  │  │  publish, │  │  /ready) │  │  settings,   │  │    │
│  │  │  consume) │  │          │  │  linking)    │  │    │
│  │  └──────────┘  └──────────┘  └──────────────┘  │    │
│  └──────────────────────┬──────────────────────────┘    │
│                         │ interfaces                     │
│  ┌──────────────────────┴──────────────────────────┐    │
│  │           Domain Layer (investment-specific)     │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │    │
│  │  │Strategies│  │ Signals  │  │   Alerts     │  │    │
│  │  │(CRUD,    │  │(evaluator│  │ (dispatch,   │  │    │
│  │  │ validate,│  │ poller,  │  │  history,    │  │    │
│  │  │ SDF)     │  │ cooldown)│  │  re-drive)   │  │    │
│  │  └──────────┘  └──────────┘  └──────────────┘  │    │
│  │  ┌──────────┐  ┌──────────┐                     │    │
│  │  │Watchlist │  │ Market   │                     │    │
│  │  │(CRUD,    │  │ Data     │                     │    │
│  │  │ digest   │  │(CoinGecko│                     │    │
│  │  │ trigger) │  │ Crypto-  │                     │    │
│  │  │          │  │ Compare) │                     │    │
│  │  └──────────┘  └──────────┘                     │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                          │
                   internal HTTP
                          │
┌─────────────────────────────────────────────────────────┐
│                 Python AI Service                       │
│  ┌──────────────────────────────────────────────────┐   │
│  │         Platform Layer (domain-agnostic)          │   │
│  │  ┌───────────┐  ┌───────────┐  ┌─────────────┐  │   │
│  │  │ Compute   │  │ Enrichment│  │ Provider    │  │   │
│  │  │ Engine    │  │ Pipeline  │  │ Registry    │  │   │
│  │  │ (cache,   │  │ (sentiment│  │ (abstract   │  │   │
│  │  │  resample,│  │  dedup,   │  │  interfaces)│  │   │
│  │  │  health)  │  │  score)   │  │             │  │   │
│  │  └───────────┘  └───────────┘  └─────────────┘  │   │
│  └──────────────────────┬───────────────────────────┘   │
│                         │ interfaces                     │
│  ┌──────────────────────┴───────────────────────────┐   │
│  │         Domain Layer (investment-specific)         │   │
│  │  ┌───────────┐  ┌───────────┐  ┌─────────────┐  │   │
│  │  │ RSI, MACD │  │CryptoPanic│  │ Project     │  │   │
│  │  │ Bollinger │  │ DuckDuckGo│  │ Registry    │  │   │
│  │  │ Volume    │  │ (news     │  │ (crypto     │  │   │
│  │  │ PctChange │  │  adapters)│  │  assets)    │  │   │
│  │  └───────────┘  └───────────┘  └─────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          │
                    Agent Gateway
                          │
┌─────────────────────────────────────────────────────────┐
│               Agent Gateway (pluggable)                 │
│  ┌──────────────────────────────────────────────────┐   │
│  │        Framework Layer (domain-agnostic)          │   │
│  │  Cron, LLM provider, Telegram channel, HTTP      │   │
│  │  fetch, subagent, permissions, observability      │   │
│  └──────────────────────┬───────────────────────────┘   │
│                         │ skill files                    │
│  ┌──────────────────────┴───────────────────────────┐   │
│  │        Domain Skills (investment-specific)         │   │
│  │  crypto-digest.md, AGENT.md persona               │   │
│  │  (references /news/, /enrich/, /projects/)        │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### P2 — Interface Contracts at Boundaries

Each boundary between platform and domain is a Go `interface`, Python `ABC`, or TypeScript `type`:

| Boundary | Interface | Platform Side | Domain Implementation |
|---|---|---|---|
| Subscription gating | `SubscriptionGatable` | Billing middleware checks `HasActiveSubscription(userID)` | Domain decides which routes are gated |
| Notification dispatch | `NotificationPayload` | Telegram sender accepts generic `{recipient, subject, body, metadata}` | Domain formats investment-specific alert message |
| Event publishing | `EventPublisher` | NATS helper publishes generic `{subject, payload}` | Domain defines `signal.triggered.*` subjects and payload schemas |
| Digest content | `DigestContentProvider` | Agent Gateway skill calls generic `/internal/digest/content` | Domain returns investment news + price summaries |
| Indicator computation | `ComputeEngine` | ai-service accepts generic `{data_points[], compute_type, params}` | Domain registers RSI, MACD, Bollinger, etc. as compute types |
| News fetching | `ContentProvider` | ai-service fetches from generic provider interface | Domain implements CryptoPanic, DuckDuckGo adapters |
| Market data | `DataFeedProvider` | Go backend fetches via generic provider interface | Domain implements CoinGecko, CryptoCompare adapters |

### P3 — Domain Registration Pattern

The domain layer **registers** its routes, event handlers, validation rules, and compute types with the platform at startup — the platform never hard-imports domain packages.

```go
// backend/cmd/server/main.go
func main() {
    platform := platform.New(cfg)          // auth, billing, notification, health, NATS
    domain := investmentdomain.New(cfg)    // strategies, signals, alerts, watchlist, market data
    domain.Register(platform)              // domain registers its HTTP routes, NATS consumers, event handlers
    platform.Start()                       // platform starts HTTP server, NATS connections, background workers
}
```

To swap domains:
```go
// Future: e-commerce domain
func main() {
    platform := platform.New(cfg)
    domain := ecommercedomain.New(cfg)     // products, price monitors, deal alerts
    domain.Register(platform)
    platform.Start()
}
```

### P4 — Separate Database Namespaces

| Namespace | Owner | Domain-Agnostic? |
|---|---|---|
| `platform_` | Platform layer | ✅ Yes — `platform_users`, `platform_notification_queue`, `platform_subscription_status` |
| `app_` | Domain layer | ❌ No — `app_strategies`, `app_signal_rules`, `app_alerts`, `app_watchlist_entries`, `app_projects` |
| `gc_` | Agent Gateway | ✅ Yes — gateway's internal tables |

Platform tables (`platform_*`) contain only domain-agnostic data: user identity, subscription status, notification queue, Telegram link status. Domain tables (`app_*`) contain all business-specific data and reference `platform_users.id` as a foreign key.

> **Migration note for POC:** Renaming existing `app_users` → `platform_users` and splitting domain-specific columns is a Phase 1 schema design task. For POC velocity, we can keep the existing `app_users` table but **organize Go code** into platform vs. domain packages from the start. The table rename can happen as a follow-up migration without code-structure changes.

---

## 3. Revised Project Structure

### Go Backend — Platform vs. Domain packages

```
backend/
├── cmd/server/main.go                    # Wires platform + domain; starts server
│
├── platform/                             # ← DOMAIN-AGNOSTIC (reusable across projects)
│   ├── auth/                             # JWT validation, Supabase integration, session mgmt
│   │   ├── middleware.go                 # Auth middleware (extracts user from JWT)
│   │   ├── supabase.go                  # Supabase client
│   │   └── handler.go                   # /auth/* routes (register, login, logout, etc.)
│   ├── billing/                          # Stripe integration, subscription gating
│   │   ├── checkout.go                  # Checkout session creation
│   │   ├── webhook.go                   # Stripe webhook handler
│   │   ├── gate.go                      # Subscription gate middleware
│   │   └── interfaces.go               # SubscriptionGatable, BillingEventHandler
│   ├── notification/                     # Generic notification dispatch (Telegram, future: email, push)
│   │   ├── interfaces.go               # NotificationPayload, NotificationSender, NotificationChannel
│   │   ├── telegram/                    # Telegram-specific sender
│   │   │   ├── bot.go                   # Telegram Bot API client
│   │   │   ├── link_service.go          # Account linking (deep-link, webhook, confirm)
│   │   │   └── handler.go              # /telegram/* routes
│   │   └── dispatcher.go               # Generic dispatch: accept payload → route to channel
│   ├── eventbus/                         # Generic event pub/sub (NATS JetStream abstracted)
│   │   ├── interfaces.go               # EventPublisher, EventConsumer, EventHandler
│   │   ├── nats.go                      # NATS JetStream implementation
│   │   └── stream.go                   # Stream/consumer config helpers
│   ├── health/                           # /health, /ready endpoints
│   ├── user/                             # User profile (timezone, settings — not domain-specific)
│   │   ├── model.go                     # User struct (id, email, timezone, telegram_chat_id, subscription_status)
│   │   └── handler.go                  # /users/me routes
│   ├── admin/                            # Admin middleware + generic user management
│   ├── server/                           # HTTP server setup, graceful shutdown, router mounting
│   │   ├── server.go                    # Server struct, Start(), Shutdown()
│   │   └── router.go                   # Platform route registration
│   └── config/                           # Platform config (env vars, non-domain settings)
│
├── domain/                               # ← INVESTMENT-SPECIFIC (swappable)
│   └── investment/                       # Domain module: investment intelligence
│       ├── register.go                  # Register(platform) — mounts domain routes, consumers, workers
│       ├── strategies/                  # Strategy CRUD, signal rule validation, SDF
│       │   ├── service.go
│       │   ├── handler.go
│       │   ├── validator.go
│       │   └── import_handler.go
│       ├── signals/                     # Signal evaluation loop, market data orchestration
│       │   ├── evaluator.go
│       │   ├── poller.go
│       │   └── cooldown.go
│       ├── alerts/                      # Alert persistence, dispatch formatting, re-drive
│       │   ├── handler.go              # /alerts routes
│       │   ├── dispatcher.go           # NATS consumer → format NotificationPayload → platform.notification
│       │   └── redriver.go
│       ├── watchlist/                   # Watchlist CRUD, digest content provision
│       │   └── handler.go
│       ├── marketdata/                  # Market data feed adapters (CoinGecko, CryptoCompare)
│       │   ├── interfaces.go           # DataFeedProvider (domain-specific: OHLCV, spot price)
│       │   ├── coingecko.go
│       │   └── cryptocompare.go
│       └── config/                      # Seed config loader (signal types, project seeds)
│           ├── seed.go
│           └── seed_test.go
│
├── pkg/                                  # Shared utilities (domain-agnostic)
│   ├── httputil/                         # HTTP helpers, error response envelope
│   ├── validate/                         # JSON Schema validation helpers
│   └── testutil/                         # Test fixtures, DB helpers
│
└── tests/
    ├── contract/                         # HTTP contract tests
    ├── integration/                      # Integration tests
    └── unit/                             # Unit tests
```

### Python ai-service — Platform vs. Domain packages

```
ai-service/
├── src/
│   ├── main.py                           # FastAPI app factory; mounts platform + domain routers
│   ├── config.py                         # Settings
│   │
│   ├── platform/                         # ← DOMAIN-AGNOSTIC (reusable)
│   │   ├── compute/                     # Generic computation engine
│   │   │   ├── interfaces.py            # ComputeEngine ABC: register(compute_type, handler)
│   │   │   ├── cache.py                 # Redis-backed result cache (generic key pattern)
│   │   │   └── resample.py             # Generic OHLCV resampling utility
│   │   ├── enrichment/                  # Generic content enrichment pipeline
│   │   │   ├── interfaces.py            # ContentEnricher ABC: enrich(items) → enriched_items
│   │   │   └── sentiment.py            # VADER sentiment (reusable for any text content)
│   │   ├── content/                     # Generic content provider
│   │   │   └── interfaces.py           # ContentProvider ABC: fetch(slug) → list[ContentItem]
│   │   └── health.py                    # /health, /health/ready
│   │
│   └── domain/                           # ← INVESTMENT-SPECIFIC (swappable)
│       └── investment/
│           ├── register.py              # register(app) — mounts domain routers
│           ├── indicators/              # Indicator computation implementations
│           │   ├── rsi.py               # RSI compute handler (registered as compute_type="rsi")
│           │   ├── volume_spike.py
│           │   ├── macd.py
│           │   ├── bollinger.py
│           │   ├── price_stats.py
│           │   ├── pct_change.py
│           │   └── router.py           # POST /indicators/{asset}, POST /pct_change/{asset}
│           ├── news/                    # News content providers
│           │   ├── cryptopanic.py       # CryptoPanic adapter (implements ContentProvider)
│           │   ├── duckduckgo.py        # DuckDuckGo fallback adapter
│           │   ├── quota.py             # CryptoPanic quota tracking
│           │   └── router.py           # GET /news/{slug}, POST /enrich/news
│           └── projects/               # Crypto project registry
│               ├── registry.py          # DB-backed project registry
│               └── router.py           # GET /projects
```

### Agent Gateway — Framework vs. Domain skills

```
agent-gateway/
├── README.md
├── docker-compose.gateway.yml
│
├── goclaw/                               # GoClaw-specific config (DEFAULT)
│   ├── docker-compose.goclaw.yml
│   ├── .env.goclaw
│   └── agents/
│       ├── _platform/                   # ← DOMAIN-AGNOSTIC agent utilities (reusable)
│       │   ├── HEARTBEAT.md             # Generic health check skill
│       │   └── tools/                   # Generic tools (if any custom tools added)
│       └── digest-agent/                # ← DOMAIN-SPECIFIC agent
│           ├── AGENT.md                 # Investment-specific persona + instructions
│           └── skills/
│               └── crypto-digest.md     # Investment-specific digest skill
│
├── openclaw/                             # ... same pattern
├── picoclaw/
├── nanobot/
└── zeroclaw/
```

### React Frontend — Platform vs. Domain pages

```
frontend/src/
├── platform/                             # ← DOMAIN-AGNOSTIC (reusable)
│   ├── components/                      # Shared UI primitives (Button, Input, Card, etc.)
│   │   └── *.stories.tsx
│   ├── pages/
│   │   ├── auth/                        # SignIn, SignUp, ForgotPassword, ResetPassword
│   │   ├── settings/                    # Telegram linking, timezone, account settings
│   │   ├── billing/                     # Subscription management (Stripe)
│   │   └── admin/                       # Admin user management
│   ├── hooks/                           # useAuth, useSubscription, useNotification
│   ├── services/                        # Platform API client hooks (TanStack Query)
│   └── lib/                             # Design tokens, utilities
│
├── domain/                               # ← INVESTMENT-SPECIFIC (swappable)
│   └── investment/
│       ├── pages/
│       │   ├── dashboard/               # Strategy list + quick stats
│       │   ├── strategies/              # Strategy create/edit, signal rule builder
│       │   ├── alerts/                  # Alert history view
│       │   └── watchlist/               # Watchlist management + digest status
│       ├── components/                  # Domain-specific components
│       │   └── *.stories.tsx
│       └── services/                    # Domain API client hooks
│
└── App.tsx                               # Composes platform + domain routes
```

---

## 4. Interface Definitions (Key Examples)

### 4.1 Go — Notification Interface (Platform)

```go
// platform/notification/interfaces.go
package notification

// NotificationPayload is the domain-agnostic envelope for any notification.
// The domain layer populates this; the platform layer delivers it.
type NotificationPayload struct {
    RecipientUserID string            // Platform resolves this to telegram_chat_id, email, etc.
    Channel         string            // "telegram", "email", "push" (extensible)
    Subject         string            // Short title (used by some channels)
    Body            string            // Formatted message body
    Metadata        map[string]string // Domain-specific key-value pairs (for logging/tracing)
}

// Sender delivers a notification via a specific channel.
type Sender interface {
    Send(ctx context.Context, payload NotificationPayload) error
    Channel() string
}

// Dispatcher routes payloads to the correct Sender based on Channel.
type Dispatcher interface {
    Dispatch(ctx context.Context, payload NotificationPayload) error
    RegisterSender(sender Sender)
}
```

### 4.2 Go — Event Bus Interface (Platform)

```go
// platform/eventbus/interfaces.go
package eventbus

// Event is a domain-agnostic event envelope.
type Event struct {
    Subject       string // e.g., "signal.triggered.btc" — domain defines subjects
    Payload       []byte // JSON-encoded domain payload
    SchemaVersion string // Forward compatibility
}

// Publisher publishes events to the bus.
type Publisher interface {
    Publish(ctx context.Context, event Event) error
}

// Handler processes a received event.
type Handler interface {
    Handle(ctx context.Context, event Event) error
}

// Consumer subscribes to events and routes them to Handlers.
type Consumer interface {
    Subscribe(subject string, handler Handler) error
    Start(ctx context.Context) error
    Drain(ctx context.Context) error
}
```

### 4.3 Go — Domain Registration Interface

```go
// platform/server/interfaces.go
package server

// DomainModule is implemented by each domain to register itself with the platform.
type DomainModule interface {
    // RegisterRoutes mounts domain HTTP routes onto the platform router.
    RegisterRoutes(router chi.Router, auth middleware.AuthMiddleware, gate middleware.SubscriptionGate)

    // RegisterEventHandlers registers domain NATS event handlers with the event bus.
    RegisterEventHandlers(consumer eventbus.Consumer)

    // RegisterBackgroundWorkers starts domain-specific background goroutines (e.g., signal poller).
    RegisterBackgroundWorkers(ctx context.Context) error

    // Shutdown gracefully stops domain workers.
    Shutdown(ctx context.Context) error
}
```

### 4.4 Go — Billing Gate Interface (Platform)

```go
// platform/billing/interfaces.go
package billing

// SubscriptionChecker is used by the gate middleware to check subscription status.
// The platform provides the implementation (Stripe-backed).
type SubscriptionChecker interface {
    HasActiveSubscription(ctx context.Context, userID string) (bool, error)
}

// BillingEventHandler allows the domain to react to billing events (optional).
type BillingEventHandler interface {
    OnSubscriptionCreated(ctx context.Context, userID string) error
    OnSubscriptionCancelled(ctx context.Context, userID string) error
    OnPaymentFailed(ctx context.Context, userID string) error
}
```

### 4.5 Python — Compute Engine Interface (Platform)

```python
# ai-service/src/platform/compute/interfaces.py
from abc import ABC, abstractmethod
from typing import Any

class ComputeHandler(ABC):
    """Domain registers concrete handlers for each compute type."""

    @abstractmethod
    async def compute(self, data_points: list[dict], params: dict) -> dict:
        """Accept raw data points + params, return computed result."""
        ...

class ComputeEngine(ABC):
    """Platform provides the engine; domain registers handlers."""

    @abstractmethod
    def register(self, compute_type: str, handler: ComputeHandler) -> None: ...

    @abstractmethod
    async def execute(self, compute_type: str, data_points: list[dict], params: dict) -> dict: ...
```

### 4.6 Python — Content Provider Interface (Platform)

```python
# ai-service/src/platform/content/interfaces.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class ContentItem:
    """Domain-agnostic content item."""
    title: str
    url: str
    published_at: str
    source: str
    metadata: dict  # Domain-specific fields

class ContentProvider(ABC):
    """Domain implements concrete providers (CryptoPanic, DuckDuckGo, etc.)."""

    @abstractmethod
    async def fetch(self, slug: str) -> list[ContentItem]: ...
```

---

## 5. Migration Rules

### What Moves to `platform/`

| Current Location | New Location | Reason |
|---|---|---|
| `internal/auth/` | `platform/auth/` | Auth is domain-agnostic |
| `internal/billing/` | `platform/billing/` | Billing is domain-agnostic |
| `internal/telegram/bot.go` | `platform/notification/telegram/bot.go` | Telegram delivery is a channel, not domain logic |
| `internal/telegram/link_service.go` | `platform/notification/telegram/link_service.go` | Linking is user-level, not domain-specific |
| `pkg/nats/` | `platform/eventbus/` | Event bus is a platform concern |
| Health endpoints | `platform/health/` | Infrastructure concern |

### What Stays in `domain/investment/`

| Current Location | New Location | Reason |
|---|---|---|
| `internal/strategies/` | `domain/investment/strategies/` | Investment-specific business logic |
| `internal/signals/` | `domain/investment/signals/` | Investment signal evaluation |
| `internal/alerts/` | `domain/investment/alerts/` | Investment alert formatting + dispatch logic |
| `internal/watchlist/` | `domain/investment/watchlist/` | Investment watchlist |
| `pkg/marketdata/` | `domain/investment/marketdata/` | Financial market data adapters |

### How Domain Uses Platform

```go
// domain/investment/alerts/dispatcher.go
package alerts

import (
    "backend/platform/notification"
    "backend/platform/eventbus"
)

type AlertDispatcher struct {
    notifier notification.Dispatcher  // Platform interface — no Telegram knowledge here
    store    AlertStore               // Domain-specific persistence
}

func (d *AlertDispatcher) Handle(ctx context.Context, event eventbus.Event) error {
    // Parse domain-specific payload
    var signal SignalTriggered
    json.Unmarshal(event.Payload, &signal)

    // Persist domain-specific alert record
    alert, err := d.store.Create(ctx, signal)

    // Format domain-specific message, then hand off to platform
    payload := notification.NotificationPayload{
        RecipientUserID: signal.UserID,
        Channel:         "telegram",
        Subject:         fmt.Sprintf("🚨 %s Alert", signal.Asset),
        Body:            formatAlertMessage(alert), // Domain-specific formatting
        Metadata:        map[string]string{"strategy_id": signal.StrategyID},
    }
    return d.notifier.Dispatch(ctx, payload)
}
```

---

## 6. Impact on Existing Tasks

This is a **structural reorganization**, not a feature change. All existing task descriptions remain valid — only the package/directory paths change. The key impact:

| Task | Impact |
|---|---|
| T001 (monorepo structure) | Directory structure updated to include `platform/` and `domain/investment/` |
| T002 (Go bootstrap) | Go module structure accounts for `platform/` and `domain/` top-level packages |
| T012 (Supabase auth) | Code lives in `platform/auth/` instead of `internal/auth/` |
| T015 (Stripe) | Code lives in `platform/billing/` instead of `internal/billing/` |
| T014 (NATS helper) | Code lives in `platform/eventbus/` instead of `pkg/nats/` |
| T022-T023 (strategies) | Code lives in `domain/investment/strategies/` |
| T030-T032 (Telegram) | Bot + linking lives in `platform/notification/telegram/` |
| T041-T042 (signals, alerts) | Code lives in `domain/investment/signals/` and `domain/investment/alerts/` |
| T052 (watchlist) | Code lives in `domain/investment/watchlist/` |
| T055-T056 (Agent Gateway) | Skills live in `agent-gateway/goclaw/agents/digest-agent/` (domain-specific) |

### No Functional Changes

- All REST API endpoints, NATS subjects, database schemas, and CI pipelines remain identical.
- The Agent Gateway abstraction layer (already documented) is unaffected — it's already domain-agnostic by design.
- The SDF (Strategy Definition Format) is a domain concept and stays in the domain layer.

---

## 7. Future Domain Swap Example

To create an "E-Commerce Price Monitor" using the same platform:

```
backend/
├── platform/          # ← Reused as-is (zero changes)
├── domain/
│   └── ecommerce/     # ← New domain module
│       ├── register.go
│       ├── products/          # Product CRUD (like strategies)
│       ├── price_monitors/    # Price check loop (like signal evaluator)
│       ├── deal_alerts/       # Deal notification formatting
│       └── retailers/         # Amazon, eBay adapters (like CoinGecko)

ai-service/
├── src/
│   ├── platform/      # ← Reused as-is
│   └── domain/
│       └── ecommerce/
│           ├── scrapers/      # Price scraping compute handlers
│           └── deals/         # Deal content providers

agent-gateway/
├── goclaw/agents/
│   ├── _platform/     # ← Reused as-is
│   └── deals-digest-agent/   # ← New domain-specific agent
│       ├── AGENT.md
│       └── skills/
│           └── daily-deals.md

frontend/src/
├── platform/          # ← Reused as-is (auth, billing, settings pages)
├── domain/
│   └── ecommerce/
│       ├── pages/     # Product list, price history, deal alerts
│       └── services/  # Domain API hooks
```

Only `domain/` directories change. Platform code, infrastructure, CI/CD — all reused.

---

## 8. Implementation Guidelines

### Rule 1: No Domain Imports in Platform

```go
// ❌ FORBIDDEN — platform importing domain
import "backend/domain/investment/strategies"

// ✅ CORRECT — platform defines interface, domain implements
import "backend/platform/server" // DomainModule interface
```

### Rule 2: Domain Depends on Platform Interfaces, Not Implementations

```go
// ❌ FORBIDDEN — domain importing NATS directly
import "github.com/nats-io/nats.go"

// ✅ CORRECT — domain uses platform's EventPublisher interface
import "backend/platform/eventbus"
```

### Rule 3: Database Migrations Follow Namespace Convention

```sql
-- Platform migrations: 001_platform_users.sql, 002_platform_notifications.sql
-- Domain migrations:   100_investment_strategies.sql, 101_investment_alerts.sql
-- Number ranges: 001-099 = platform, 100-199 = investment domain, 200-299 = next domain
```

### Rule 4: Config Separation

```
config/
├── platform.env.example      # Auth, billing, notification, infra config
└── domain/
    └── investment/
        ├── seed.yaml          # Investment-specific seed config
        └── seed.schema.json   # Investment-specific schema
```

### Rule 5: Test Organization Mirrors Source

```
tests/
├── platform/                  # Platform contract + integration tests
│   ├── auth_test.go
│   ├── billing_test.go
│   └── notification_test.go
└── domain/
    └── investment/            # Domain-specific tests
        ├── strategies_test.go
        ├── signals_test.go
        └── alerts_test.go
```

---

## 9. Decision Summary

| Aspect | Before | After |
|---|---|---|
| Auth code location | `internal/auth/` (mixed with domain) | `platform/auth/` (isolated, reusable) |
| Billing code location | `internal/billing/` (mixed with domain) | `platform/billing/` (isolated, reusable) |
| Notification code | Telegram-specific in `internal/telegram/` | Generic `platform/notification/` with Telegram as a channel |
| Event bus | `pkg/nats/` (NATS-specific) | `platform/eventbus/` with NATS as an implementation |
| Domain code | `internal/strategies/`, `internal/signals/`, etc. | `domain/investment/strategies/`, `domain/investment/signals/`, etc. |
| AI agents | Agent Gateway already domain-agnostic | Skills organized into `_platform/` (reusable) and domain-specific agent dirs |
| Swapping domains | Requires rewriting most of `internal/` | Requires writing only a new `domain/` module + skill files |

**Trade-off accepted:** Slightly more indirection (interfaces at boundaries) in exchange for full domain portability and platform reusability.
