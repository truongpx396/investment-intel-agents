# Tasks: Investment Intel AI Agents — POC

**Input**: `specs/001-investment-intel-poc/plan.md`, `spec.md`  
**Prerequisites**: `plan.md` ✅, `spec.md` ✅

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel with other tasks in the same phase
- **[Story]**: US1 = Signal Strategy Config, US2 = Telegram Notifications, US3 = Daily Digest,
  US4 = Telegram Linking, US5 = Alert History, US0 = Cross-cutting / Auth / Billing / Admin

---

## Phase 1: Project Setup & Shared Infrastructure

**Purpose**: Establish repo structure, CI, design system, and all shared config before any
feature work begins. No user story can be built without this foundation.

- [ ] T001 Initialise monorepo directory structure per `plan.md` (`backend/`, `ai-service/`, `goclaw/`, `frontend/`, `infra/`, `migrations/`)
- [ ] T002 [P] Bootstrap Go module in `backend/` (`go.mod`, `go.sum`, `golangci-lint` config, `Makefile`)
- [ ] T003 [P] Bootstrap Python project in `ai-service/` (`pyproject.toml`, `uv.lock`, `ruff` config)
- [ ] T004 [P] Bootstrap React project in `frontend/` (Vite + React 19, TailwindCSS v4, TanStack Router, TanStack Query, Vitest, Playwright, Storybook 8 with `@storybook/react-vite`); configure `.storybook/main.ts` with Vite builder and TailwindCSS; verify `npm run storybook` starts successfully
- [ ] T005 [P] Configure GitHub Actions CI pipeline — create the following workflow files under `.github/workflows/`:

  **`ci-go.yml`** — triggers on `pull_request` and `push` to `main`; jobs:
  - `lint`: `golangci-lint run ./...` (use `golangci-lint-action`)
  - `test`: `go test ./... -race -coverprofile=coverage.out`; fail if coverage < 80% globally or < 95% on `internal/signals/`, `internal/alerts/`, `internal/auth/`, `internal/billing/`
  - `build`: `go build ./cmd/server/` to catch compile errors on every PR

  **`ci-python.yml`** — triggers on `pull_request` and `push` to `main`; jobs:
  - `lint`: `ruff check .` and `ruff format --check .`
  - `test`: `pytest --cov=src --cov-fail-under=80` inside `ai-service/`

  **`ci-frontend.yml`** — triggers on `pull_request` and `push` to `main`; jobs:
  - `lint`: `eslint . --max-warnings 0` + `tsc --noEmit`
  - `test`: `vitest run --coverage` inside `frontend/`
  - `storybook`: `npm run build-storybook` + `@storybook/test-runner` interaction tests (see T008a); archive Storybook build as a CI artefact
  - `e2e` (Phase 10 gate, skipped until T073): `playwright test` — kept as a separate job so it can be enabled without touching other jobs

  **`ci-migrations.yml`** — triggers on `pull_request` when files under `migrations/` change; jobs:
  - `validate`: spin up Postgres 18 via service container, run `golang-migrate up` against it, assert exit 0

  All workflows MUST use pinned action versions (e.g., `actions/checkout@v4`) and cache Go modules, pip, and npm dependencies for faster runs
- [ ] T006 [P] Set up `docker-compose.yml` for local dev: Postgres 18 + pgvector, Redis 7, NATS JetStream, Traefik, GoClaw (`latest` image — web UI embedded, dashboard at `http://localhost:18790`), backend, ai-service, frontend, `prometheus-nats-exporter` — **no separate `goclaw-web` container** (Apr 2026: web UI is compiled into the Go binary); NATS service MUST mount a named volume (`nats-data:/data`) and enable JetStream with `FileStorage` configured (`--js --sd /data` flags or equivalent NATS config file) so that stream state persists across `docker compose restart`
- [ ] T007 [P] Create initial PostgreSQL migration framework (`golang-migrate`); add `migrations/` directory with `001_init.sql` creating base schema namespaces (`app_`, `gc_`)
- [ ] T008 [P] Define TailwindCSS design tokens (colours, spacing, typography) and create shared component primitives: `Button`, `Input`, `Card`, `Badge`, `Spinner`, `EmptyState`, `ErrorMessage`; each primitive MUST have a co-located `*.stories.tsx` file covering all variants (size, state, disabled, loading); Storybook stories serve as the living design-system documentation and visual baseline — **no primitive ships without a story**
- [ ] T008a [P] Add `storybook:build` step to GitHub Actions CI: build static Storybook (`npm run build-storybook`) and run `@storybook/test-runner` to execute interaction tests; fail CI if any story throws or any interaction test fails; Storybook build artefact archived as a CI artefact for visual review
- [ ] T009 [P] Configure Traefik static config: TLS termination (Cloudflare origin cert), routing rules for `api.`, `app.`, `goclaw.` subdomains, rate-limit middleware
- [ ] T010 [P] Configure GoClaw: `.env` with `GOCLAW_ANTHROPIC_API_KEY`, `GOCLAW_TELEGRAM_BOT_TOKEN`, `GOCLAW_DB_*`; `docker-compose.goclaw.yml`; verify `GET /health` returns 200 — use the `latest` image (web UI + API in one binary, dashboard at `http://localhost:18790`); use **channel health diagnostics panel** in the GoClaw dashboard to verify Telegram bot token and channel connectivity before writing any custom diagnostic code (Apr 2026)
- [ ] T010a [P] Create `specs/001-investment-intel-poc/contracts/rest-api.md`: draft endpoint signatures and response schemas for auth, strategies, alerts, watchlist, billing, and internal service-to-service routes; prerequisite for all contract tests in Phases 3–8
- [ ] T010b [P] Create `specs/001-investment-intel-poc/contracts/nats-events.md`: define the complete NATS JetStream topology — stream configs, subject schemas, consumer configs, and payload contracts; prerequisite for T014, T041, T042, and T042a. Document the following:
  **Stream topology** (two streams — different retention semantics require separate streams):
  | Stream | Subjects | Retention | MaxAge | Storage | Replicas |
  |---|---|---|---|---|---|
  | `SIGNALS` | `signal.triggered.>` | `WorkQueuePolicy` (auto-delete on ACK) | 1 h (safety net for unACKed messages) | `FileStorage` | 1 (POC single-node) |
  | `ALERT_QUEUE` | `alert.pending`, `alert.expired` | `LimitsPolicy` | 25 h (covers FR-025 24 h window + buffer) | `FileStorage` | 1 |
  **Consumer configs** (document alongside each stream):
  - `alerts-dispatcher` on `SIGNALS`: `DeliverAll`, `AckExplicit`, `MaxAckPending: 100` (flow-control ceiling; prevents dispatcher goroutine overload if Telegram API degrades), `AckWait: 30s`
  - `alerts-redriver` on `ALERT_QUEUE` subject `alert.pending`: `DeliverAll`, `AckExplicit`, `MaxDeliver: 48` (≈ 48 × 30 min backoff = 24 h coverage), `AckWait: 60s`; messages that exceed `MaxDeliver` are auto-forwarded by NATS to `alert.expired` subject (Dead Letter) — no application re-publish required
  **Message schemas** (include `schema_version: "v1"` header on all messages for forward compatibility):
  - `signal.triggered.{asset}` — `{ schema_version, user_id, strategy_id, signal_rule_id, asset, trigger_value, threshold, triggered_at }`; subject is asset-keyed; user isolation enforced by `user_id` in payload, **not** by subject
  - `alert.pending` — `{ schema_version, alert_id, user_id, triggered_at }`; produced by `alerts-dispatcher` when `telegram_chat_id IS NULL`
  - `alert.expired` — `{ schema_version, alert_id, user_id, triggered_at, expired_at }`; written by NATS Dead Letter forwarding when `MaxDeliver` is exhausted; consumed by a lightweight handler that sets `telegram_status=Expired` in Postgres
  - ~~`digest.requested`~~ — **removed**: GoClaw digest is triggered by its own internal cron scheduler; no NATS producer or consumer exists for this subject
- [ ] T010c [P] Create `specs/001-investment-intel-poc/research.md` with initial structure: GoClaw integration notes, market data feed evaluation, crypto news API evaluation, performance test results placeholder (filled during Phase 10)

**Checkpoint**: All services start locally via `docker compose up`. CI runs and is green on an empty test suite.

---

## Phase 2: Foundation — Auth, Supabase, NATS, Stripe Bootstrap

**Purpose**: Core infrastructure all user stories depend on. No feature work starts until this phase is complete.

**⚠️ CRITICAL**: Stories US1–US5 and billing MUST NOT be started until this phase passes its checkpoint.

- [ ] T011 Write contract tests for `POST /auth/register`, `POST /auth/login`, `POST /auth/logout`, `POST /auth/forgot-password`, `POST /auth/reset-password` in `backend/tests/contract/auth_test.go`
- [ ] T012 Implement Supabase Auth integration in `backend/internal/auth/`: JWT validation middleware using Supabase public key; user context injection; email verification + password-reset flows are **delegated to Supabase Auth** — no custom implementation required (FR-020 is satisfied by Supabase's built-in flows)
- [ ] T013 [P] Create `app_users` migration (`002_users.sql`): `id`, `email`, `timezone` (default `UTC`), `telegram_chat_id`, `telegram_linked_at`, `subscription_status`, `stripe_customer_id`, `created_at`
- [ ] T014 [P] Implement NATS JetStream connection helper in `backend/pkg/nats/`: publisher, durable consumer setup, reconnect logic; on service startup the helper MUST call `js.AddStream()` (or `js.UpdateStream()` if already exists) for **both** streams defined in T010b — `SIGNALS` and `ALERT_QUEUE` — with the exact `RetentionPolicy`, `MaxAge`, `Storage`, and `Replicas` values from the contract; do NOT rely on NATS auto-create defaults (wrong retention policy will be silently applied); integration test with local NATS must assert that both streams exist with correct config after `Connect()`
- [ ] T015 [P] Bootstrap Stripe integration in `backend/internal/billing/`: Stripe Go SDK init, webhook signature validation middleware, stub handlers for `customer.subscription.created`, `customer.subscription.deleted`, `invoice.payment_failed`; contract tests for `POST /billing/webhook`
- [ ] T016 [P] Implement `GET /health` and `GET /ready` endpoints in backend; wire into Traefik health checks
- [ ] T017 [P] Implement React auth flow pages: `SignIn`, `SignUp` (with email verification notice), `ForgotPassword`, `ResetPassword`; use TanStack Query mutation hooks; loading + error states required
- [ ] T018 [P] Implement auth route guard in React: redirect unauthenticated users to `/sign-in`; persist JWT in httpOnly cookie via backend proxy

**Checkpoint**: User can register, verify email, log in, and access a blank authenticated dashboard. CI green.

---

## Phase 3: User Story 1 — Signal Strategy Configuration (P1) 🎯 MVP

**Goal**: Logged-in users can create, edit, activate, pause, and delete BTC/ETH signal strategies
with price-threshold, % price-change, and RSI signal rules. Dashboard shows all strategies.

**Independent Test**: Create a BTC price-threshold strategy at $70k → save → retrieve → appears
as Active. Edit threshold → verify persisted. Delete → gone. No notifications needed.

### Tests — US1 (write first, must fail before implementation)

- [ ] T019 [P] [US1] Contract tests: `POST /strategies`, `GET /strategies`, `GET /strategies/:id`, `PUT /strategies/:id`, `DELETE /strategies/:id`, `PATCH /strategies/:id/status` in `backend/tests/contract/strategies_test.go`
- [ ] T020 [P] [US1] Unit tests for signal rule validator in `backend/internal/strategies/validator_test.go`: empty rules, contradictory rules (price above X AND below X), missing required fields

### Implementation — US1

- [ ] T021 [US1] Migration `003_strategies.sql`: `app_strategies` (`id`, `user_id`, `name`, `asset` enum BTC/ETH, `status` enum Active/Paused, `created_at`); `app_signal_rules` (`id`, `strategy_id`, `signal_type` enum, `operator`, `threshold`, `window_minutes`)
- [ ] T022 [P] [US1] Implement strategy service in `backend/internal/strategies/service.go`: CRUD, status toggle, signal rule validation (cyclomatic complexity ≤ 10 per function)
- [ ] T023 [P] [US1] Implement strategy HTTP handlers in `backend/internal/strategies/handler.go`; wire into router; RLS enforced via Supabase JWT user ID
- [ ] T024 [US1] React — Strategy list page (`/dashboard`): fetches `GET /strategies`, shows Active/Paused badges, empty-state, loading skeleton, error state
- [ ] T025 [P] [US1] React — Strategy create/edit form (`/strategies/new`, `/strategies/:id/edit`): asset selector, signal type selector, operator + threshold inputs, client-side validation matching server rules
- [ ] T026 [P] [US1] React — Strategy status toggle (Activate/Pause button) with optimistic update + rollback on error
- [ ] T027 [US1] Integration test: full strategy lifecycle (create → activate → edit → pause → delete) against live Postgres in `backend/tests/integration/strategies_test.go`

**Checkpoint**: US1 fully functional and independently testable. CI green. ≥ 80% coverage on `internal/strategies/`.

---

## Phase 4: User Story 4 — Telegram Account Linking (P1)

**Goal**: User links their Telegram account via self-service flow; confirmation message sent; can unlink; broken links detected.

**Independent Test**: Follow in-app instructions → complete Telegram bot `/start` → account marked linked → confirmation Telegram message received.

### Tests — US4 (write first)

- [ ] T028 [P] [US4] Contract tests: `POST /telegram/link/initiate`, `POST /telegram/link/confirm`, `DELETE /telegram/link`, `GET /telegram/link/status` in `backend/tests/contract/telegram_test.go`

### Implementation — US4

- [ ] T029 [US4] Migration `004_telegram.sql`: add `telegram_link_token` (short-lived UUID), `telegram_link_token_expires_at` to `app_users`
- [ ] T030 [P] [US4] Implement Telegram Bot API client in `backend/internal/telegram/bot.go`: `SendMessage(chatID, text)`, `GetWebhookUpdate()`, token-based deep-link generation
- [ ] T031 [P] [US4] Implement linking service in `backend/internal/telegram/link_service.go`: generate one-time link token, handle bot `/start?token=` deep-link webhook, confirm link, send confirmation message, detect broken link (Telegram 403/blocked → set `telegram_chat_id = NULL`, emit web-app prompt)
- [ ] T032 [US4] Implement link/unlink HTTP handlers; register Telegram webhook endpoint with Telegram Bot API
- [ ] T033 [P] [US4] React — Settings page (`/settings/telegram`): shows link status, displays bot deep-link + instructions when unlinked, shows linked username + unlink button when linked, re-link prompt when broken
- [ ] T034 [US4] Integration test: link flow, unlink, broken-link detection in `backend/tests/integration/telegram_test.go`

**Checkpoint**: US4 fully functional. User can link and unlink. CI green.

---

## Phase 5: User Story 2 — Real-Time Telegram Notifications (P2)

**Goal**: When an active strategy's signal condition fires, user receives a Telegram notification within 60 s.

**Independent Test**: With active BTC strategy + linked Telegram, simulate signal trigger → Telegram message delivered < 60 s with correct content. Paused strategy → no message. Two users → isolated.

### Tests — US2 (write first)

- [ ] T035 [P] [US2] Unit tests for signal evaluator in `backend/internal/signals/evaluator_test.go`: price threshold above/below, % change, RSI overbought/oversold; deterministic with mocked market data
- [ ] T036 [P] [US2] Contract test: `POST /signals/test-trigger` (admin-only test endpoint for CI) in `backend/tests/contract/signals_test.go`
- [ ] T037 [P] [US2] Unit tests for alert dispatcher in `backend/internal/alerts/dispatcher_test.go`: correct message format, paused strategy = no send, retry logic (3× exponential back-off)

### Implementation — US2

- [ ] T038 [US2] Migration `005_alerts.sql`: `app_alerts` (`id`, `strategy_id`, `user_id`, `signal_rule_id`, `asset`, `trigger_value`, `threshold`, `triggered_at`, `telegram_status` enum Pending/Sent/Failed, `retry_count`)
- [ ] T039 [US2] Implement market data adapter interface `pkg/marketdata/provider.go` and CoinGecko implementation `pkg/marketdata/coingecko.go`; unit tests with mocked HTTP responses
- [ ] T039a [P] [US2] Implement CryptoCompare `histominute` adapter in `pkg/marketdata/cryptocompare.go` for intraday OHLCV data required by the 14-period RSI calculation; unit tests with mocked HTTP; wire as the data source for `internal/signals/rsi.go`; CoinGecko adapter handles spot price + % change signals only
- [ ] T040 [P] [US2] Implement RSI calculator in `backend/internal/signals/rsi.go` (14-period, from OHLC slice); unit tests covering overbought (>70), oversold (<30), neutral
- [ ] T041 [US2] Implement signal evaluation loop in `backend/internal/signals/poller.go`: 30 s Go ticker, single-pass evaluation across all active strategies per tick, publish `signal.triggered.{asset}` to NATS JetStream on match; payload MUST include `user_id`, `strategy_id`, `signal_rule_id`, `asset`, `trigger_value`, `threshold`, `triggered_at`; subject is asset-keyed (`signal.triggered.btc` / `signal.triggered.eth`) — user isolation is enforced in the dispatcher by `user_id` in the payload, not by subject; **signal cooldown**: before publishing, check Redis key `cooldown:{signal_rule_id}` (TTL = configurable, default 5 min via `SIGNAL_COOLDOWN_SECONDS` env var); if key exists, skip publish for this tick (condition is still true but already fired recently); if key absent, publish and set the key with TTL — this prevents duplicate NATS events when a condition remains true across multiple consecutive ticks and protects against alert spam; unit tests MUST cover: fires on first match, suppressed on second tick within cooldown, fires again after cooldown expires
- [ ] T042 [US2] Implement NATS consumer + alert dispatcher in `backend/internal/alerts/dispatcher.go`: consume `signal.triggered.{asset}` (durable consumer group `alerts-dispatcher`), persist Alert record scoped to `user_id`, call `telegram.bot.SendMessage`, update `telegram_status`; if user has no linked Telegram (`telegram_chat_id IS NULL`), set `telegram_status=Pending` and publish `alert.pending` to NATS (payload: `alert_id`, `user_id`, `triggered_at`) instead of attempting delivery; 3× retry with exponential back-off on Telegram API failure; surface `telegram_status=Failed` for web-app indicator (per FR-010, FR-025)
- [ ] T042a [P] [US2] Implement NATS consumer + re-drive worker in `backend/internal/alerts/redriver.go`: consume `alert.pending` from the `ALERT_QUEUE` stream (durable consumer `alerts-redriver`); on receipt, check if user has now linked Telegram; if linked, attempt `telegram.bot.SendMessage` and update `telegram_status=Sent`, then ACK the message; if still unlinked, call `msg.NakWithDelay(backoff)` — **do NOT re-publish a new message** (re-publishing creates stream bloat, duplicate-message risk, and false Prometheus pending-count alerts); backoff schedule: 1 min, 5 min, 15 min, 30 min, then 30 min for all remaining retries up to `MaxDeliver: 48`; when NATS exhausts `MaxDeliver` it auto-forwards the message to `alert.expired` subject — a separate lightweight handler in `redriver.go` consumes `alert.expired` and sets `telegram_status=Expired` in Postgres (no Telegram call); idempotency: check `telegram_status` in Postgres before attempting delivery — if already `Sent`, ACK without sending; unit tests in `backend/internal/alerts/redriver_test.go` MUST cover: deliver-on-link, NakWithDelay-when-unlinked, expired-DLQ-handler sets correct DB status, idempotent double-delivery
- [ ] T043 [P] [US2] Integration test: end-to-end signal trigger → NATS → alert persist → Telegram send (with mocked Telegram Bot API) in `backend/tests/integration/signal_notification_test.go`; MUST include the following cases: (a) happy path — linked user receives message; (b) durable consumer restart — message is redelivered and idempotency guard (`signal_rule_id + triggered_at`) prevents duplicate `Alert` record; (c) unlinked user — `alert.pending` is published and no Telegram call is made; (d) re-drive after linking — re-drive worker delivers queued alert within SLA
- [ ] T043a [P] [US2] React — Delivery-failure indicator on strategy dashboard card (`/dashboard`): when any alert for a strategy has `telegram_status=Failed`, show a red badge on the card; clicking it navigates to alert history filtered by that strategy (per FR-010); Vitest unit test for the badge component

**Checkpoint**: Signal fires → Telegram notification < 60 s. Paused strategy = silent. Two-user isolation verified. CI green. ≥ 95% coverage on `internal/signals/` and `internal/alerts/`.

---

## Phase 6: User Story 5 — Alert History (P2)

**Goal**: Read-only alert history per user in the web app, filterable by strategy, reverse-chronological.

**Independent Test**: Strategy with 3 past alerts → open history page → all 3 listed with correct metadata → filter by strategy → only matching alerts shown → empty state when none.

### Tests — US5 (write first)

- [ ] T044 [P] [US5] Contract tests: `GET /alerts` (paginated, filter by `strategy_id`), `GET /alerts/:id` in `backend/tests/contract/alerts_test.go`

### Implementation — US5

- [ ] T045 [US5] Implement alert query handler in `backend/internal/alerts/handler.go`: `GET /alerts` with `strategy_id` filter, pagination (limit/offset), RLS scoped to authenticated user
- [ ] T046 [P] [US5] React — Alert history page (`/alerts`): reverse-chronological list, strategy filter dropdown, pagination, loading skeleton, empty state, delivery-failure badge; `GET /alerts` via TanStack Query
- [ ] T047 [US5] Integration test: alert list + filter in `backend/tests/integration/alerts_test.go`

**Checkpoint**: US5 functional. History visible after US2 generates alerts. CI green.

---

## Phase 7: User Story 3 — Watchlist & Daily Digest (P3)

**Goal**: User manages a watchlist; GoClaw digest agent runs daily at 08:30 UTC, fetches news via CryptoPanic, summarises with Claude (claude-3-5-haiku), sends one Telegram message per user with per-project sections.

**Independent Test**: Two projects on watchlist → next digest run → single Telegram message with one section per project. Empty watchlist → no message. New project added before run → included.

### Tests — US3 (write first)

- [ ] T048 [P] [US3] Contract tests: `POST /watchlist`, `GET /watchlist`, `DELETE /watchlist/:project_id` in `backend/tests/contract/watchlist_test.go`
- [ ] T049 [P] [US3] Unit tests for CryptoPanic adapter in `ai-service/tests/unit/test_news_adapter.py`: response parsing, quota-exceeded fallback, empty results
- [ ] T050 [P] [US3] GoClaw digest skill test: validate `crypto-digest.md` skill prompt produces correctly structured output with mocked `web_fetch` response (manual verification + prompt regression test)

### Implementation — US3

- [ ] T051 [US3] Migration `006_watchlist.sql`: `app_watchlist_entries` (`id`, `user_id`, `project_slug`, `project_name`, `added_at`); seed migration for curated project list (`app_projects`)
- [ ] T052 [P] [US3] Implement watchlist CRUD handlers in `backend/internal/watchlist/handler.go`; contract: add/remove/list entries; RLS scoped to user
- [ ] T053 [P] [US3] React — Watchlist page (`/watchlist`): grid of curated projects with add/remove toggle, loading, empty-state ("add projects to receive your daily digest"), error state
- [ ] T054 [US3] Implement CryptoPanic news adapter in `ai-service/src/news/cryptopanic.py`: fetch news by currency slug, parse items (title, url, published_at), DuckDuckGo fallback on quota exceeded; expose `GET /news/{project_slug}` endpoint
- [ ] T054a [P] [US3] Define abstract `NewsProvider` interface in `ai-service/src/news/provider.py`: `fetch_news(project_slug: str) -> list[NewsItem]` with `NewsItem` dataclass (title, url, published_at, source); `CryptoPanicAdapter` in T054 MUST implement this interface; unit tests asserting interface contract; interface contract also documented in `contracts/rest-api.md`
- [ ] T055 [US3] Create GoClaw digest agent in `goclaw/agents/digest-agent/`: `AGENT.md` (persona: "Investment Intel Digest Bot"; instructions to fetch each user's watchlist via backend API, call news endpoint per project, summarise with LLM, send via `message` tool); `HEARTBEAT.md` (confirm digest sent) — **note (Apr 2026)**: Knowledge Graph sharing is configured separately from workspace sharing in the GoClaw agent setup UI; if KG access is required for the digest agent, enable KG sharing independently
- [ ] T056 [P] [US3] Create GoClaw `crypto-digest` skill in `goclaw/agents/digest-agent/skills/crypto-digest.md`: skill steps — (1) `GET /watchlist` for user via internal service token (see T057a), (2) `web_fetch` CryptoPanic per project, (3) LLM summarise (claude-3-5-haiku, ≤ 200 tokens/project), (4) `message` to Telegram chat ID; skill MUST explicitly handle the no-news case: if `web_fetch` returns zero items for a project, insert a section reading "No significant updates in the last 24 hours" (per FR-015) — **note (Apr 2026)**: parallel `web_fetch` across watchlist projects is reliable; use subagent `waitAll` for concurrent news fetch to reduce total digest latency; auto-retry and token tracking are built in
- [ ] T057 [US3] Configure GoClaw cron: add cron entry in agent config `cron: "30 8 * * *"` (08:30 UTC); verify trigger via GoClaw web dashboard — **note (Apr 2026)**: timezone handling is now stable for all schedule kinds (`cron`, `at`, `every`); use the **"Run Now"** button in the dashboard to manually trigger the digest agent during development and verification (previously broken, fixed Apr 2026)
- [ ] T057a [P] [US3] Define and implement service-to-service auth for GoClaw → backend API calls: `GOCLAW_INTERNAL_TOKEN` env var; add `/internal/` route prefix in Traefik config validated by a static-bearer middleware (bypasses Supabase JWT); document token scheme in `contracts/rest-api.md`; unit test the middleware — **note (Apr 2026)**: additionally configure **per-agent grants** in GoClaw (new feature) to scope the digest agent's access to only the `/internal/watchlist` and `/internal/users` endpoints; per-agent grants provide a finer-grained permission layer on top of the static bearer token; if the provider type is changed, GoClaw will automatically re-validate for SSRF
- [ ] T058 [P] [US3] React — Digest history section on watchlist page: last digest timestamp + status (Sent/Pending/Failed); re-link prompt if Telegram not linked
- [ ] T058a [P] [US3] React — Timezone selector in `/settings/account`: dropdown of common IANA timezones; persisted to `app_users.timezone` via `PATCH /users/me`; display hint "Your daily digest is scheduled for 09:00 [timezone]"; include UI notice that digest is currently sent at 08:30 UTC (POC limitation — per-timezone scheduling is post-POC)
- [ ] T059 [US3] Integration test: watchlist CRUD in `backend/tests/integration/watchlist_test.go`; news adapter in `ai-service/tests/integration/test_news_adapter.py`

**Checkpoint**: Watchlist CRUD works. GoClaw digest cron fires at 08:30 UTC and delivers Telegram messages. Empty watchlist = no message. CI green.

---

## Phase 8: Billing & Subscription (US0)

**Goal**: Users subscribe via Stripe from the web app; subscription status gates access to signal strategies and digest.

- [ ] T060 Write contract tests for `POST /billing/create-checkout-session`, `GET /billing/subscription-status`, `POST /billing/webhook` in `backend/tests/contract/billing_test.go`
- [ ] T061 [P] Implement Stripe checkout session creation in `backend/internal/billing/checkout.go`: create Stripe customer + checkout session, return session URL to frontend
- [ ] T062 [P] Implement Stripe webhook handler in `backend/internal/billing/webhook.go`: `subscription.created` → set `subscription_status=active`; `subscription.deleted` → set `subscription_status=cancelled`; `invoice.payment_failed` → set `subscription_status=past_due`
- [ ] T063 [P] Implement subscription gate middleware in backend: routes under `/strategies`, `/watchlist`, `/alerts` return 402 if `subscription_status != active`
- [ ] T064 [P] React — Billing page (`/billing`): current plan status, "Subscribe" CTA (redirects to Stripe Checkout), "Manage subscription" link (Stripe Customer Portal); loading + error states
- [ ] T065 React — Subscription gate in frontend: redirect to `/billing` with explanation banner when API returns 402
- [ ] T066 Integration test: Stripe webhook lifecycle (created → cancelled → payment_failed) using Stripe CLI event fixtures

**Checkpoint**: Subscription required to access features. Stripe webhook correctly updates status. CI green.

---

## Phase 9: Admin Panel (US0)

**Goal**: Admin users can manage all users, view subscription statuses, and configure Stripe products.

- [ ] T067 Migration `007_admin.sql`: add `is_admin` boolean to `app_users`; seed one admin user for POC
- [ ] T068 [P] Implement admin middleware in backend: validate `is_admin = true` from JWT claims; return 403 otherwise
- [ ] T069 [P] Implement admin handlers: `GET /admin/users` (list all users + subscription status), `PATCH /admin/users/:id/subscription` (manually override status)
- [ ] T070 [P] React — Admin page (`/admin`): user table (email, status, Telegram linked, last active), subscription override controls; guarded by admin role check

**Checkpoint**: Admin can view and manage all users. Non-admins receive 403. CI green.

---

## Phase 10: Observability, Performance & Quality Gates

**Purpose**: Wire monitoring, validate performance budgets, finalise CI gates.

- [ ] T071 [P] Configure Prometheus scrape targets: backend `/metrics` (Go runtime + custom `signal_evaluations_total`, `alerts_dispatched_total`, `alerts_pending_total`, `telegram_delivery_latency_seconds`), GoClaw OTLP → Prometheus, **`prometheus-nats-exporter`** (official NATS exporter — expose `nats_consumer_pending_messages`, `nats_publish_errors_total`, `nats_consumer_redelivered_messages_total` per consumer group); add `prometheus-nats-exporter` service to `docker-compose.yml` alongside NATS
- [ ] T072 [P] Import Grafana dashboards: Go API latency (p50/p95/p99), NATS throughput, GoClaw LLM call latency + cost, signal evaluation cycle time
- [ ] T073 [P] Add Playwright E2E test: full happy-path flow (register → link Telegram mock → create strategy → view history → add watchlist item) in `frontend/tests/e2e/happy_path.spec.ts`
- [ ] T074 [P] Add Playwright accessibility audit (axe-core) on all pages; assert zero WCAG 2.1 AA violations in `frontend/tests/e2e/a11y.spec.ts`
- [ ] T075 Run Lighthouse CI on React SPA: assert LCP ≤ 2.5 s, CLS ≤ 0.1; add as CI step
- [ ] T076 [P] Verify all API endpoints respond < 200 ms p95 under 50-user simulated load using `k6` (or `hey`); add results to `specs/001-investment-intel-poc/research.md`
- [ ] T077 [P] Final coverage sweep: assert `go test -cover` ≥ 80% globally; ≥ 95% on `internal/signals/`, `internal/alerts/`, `internal/auth/`, `internal/billing/`; fail CI if thresholds not met
- [ ] T078 [P] Final lint sweep: `golangci-lint run ./...`, `ruff check .`, `eslint .` — zero warnings; fix all or add justified inline suppressions
- [ ] T078a [P] Configure Prometheus alert rules in `infra/monitoring/alerts.yaml`: (1) `digest_delivery_failures_total > 0` within 1 h → page operator; (2) `signal_evaluation_cycle_seconds p95 > 25` (>83% of 30 s tick) → warn; (3) `telegram_delivery_latency_seconds p95 > 30` → warn; (4) `nats_consumer_pending_messages{consumer="alerts-dispatcher"} > 50` for 5 min → warn (stuck consumer); (5) `nats_consumer_pending_messages{consumer="alerts-redriver"} > 100` for 5 min → warn (re-drive backlog growing); (6) `nats_publish_errors_total > 0` → warn; wire all rules into Grafana alert notification channel

**Checkpoint**: All constitution quality gates pass in CI. Prometheus + Grafana operational. Performance budgets met.

---

## Phase 11: Deployment & POC Handoff

- [ ] T079 Write `infra/docker-compose.prod.yml` with production overrides: image tags pinned, env vars from secrets, resource limits — GoClaw service uses the `latest` (or pinned `vX.Y.Z`) image only; **no separate `goclaw-web` container** (web UI is embedded in the binary since Apr 2026); use `latest-otel` variant if OpenTelemetry export is required; **NATS persistent volume**: NATS is configured with `FileStorage` (per T010b stream configs) — the production compose MUST mount a named volume for the NATS data directory (e.g., `nats-data:/data`) so that unprocessed `alert.pending` messages in the `ALERT_QUEUE` stream survive container restarts; verify this is also present in `docker-compose.yml` (local dev) to catch issues early
- [ ] T080 [P] Write `specs/001-investment-intel-poc/quickstart.md`: local dev setup (clone → `cp .env.example .env` → `docker compose up`) and production deploy steps
- [ ] T081 [P] Configure Cloudflare DNS + WAF rules; verify TLS termination via Traefik
- [ ] T082 [P] Smoke test production deployment: register a user, link Telegram, create strategy, add watchlist item, confirm digest arrives next morning
- [ ] T083 [P] Tag release `v0.1.0-poc` in Git; update `CHANGELOG.md` with all user-visible changes per constitution

**Checkpoint**: POC running on DigitalOcean. Smoke test passed. Handoff complete.

---

## Summary

| Phase | Stories | Tasks | Critical Path |
|-------|---------|-------|---------------|
| 1 – Setup | — | T001–T010c | Blocks everything |
| 2 – Foundation | US0 (auth) | T011–T018 | Blocks US1–US5 |
| 3 – Strategy Config | US1 (P1) | T019–T027 | MVP gate |
| 4 – Telegram Linking | US4 (P1) | T028–T034 | Blocks US2 + US3 delivery |
| 5 – Notifications | US2 (P2) | T035–T043a (incl. T042a) | Core value delivery |
| 6 – Alert History | US5 (P2) | T044–T047 | Pairs with US2 |
| 7 – Watchlist + Digest | US3 (P3) | T048–T059 | GoClaw + AI layer |
| 8 – Billing | US0 | T060–T066 | Revenue gate |
| 9 – Admin | US0 | T067–T070 | Ops tooling |
| 10 – Quality Gates | — | T071–T078a | Constitution compliance |
| 11 – Deploy | — | T079–T083 | POC handoff |

**Total tasks**: 96 (83 original + 11 added by analysis remediation: T010a–T010c, T031a, T039a, T042a, T043a, T054a, T057a, T058a, T078a + 2 added for Storybook: T008a; T004/T008 expanded in-place)
