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
- [ ] T001a [P] Create `CHANGELOG.md` stub at repository root with an `[Unreleased]` section per constitution Quality Gates §6; all feature PRs MUST append entries incrementally (not batch at release)
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
- [ ] T009a [P] Configure and integration-test Traefik rate-limit middleware per NFR-SEC-004: authenticated users ≤ 120 req/min, unauthenticated paths (login, register) ≤ 20 req/min; integration test MUST assert that requests exceeding the limit receive HTTP 429
- [ ] T010 [P] Configure GoClaw: `.env` with `GOCLAW_ANTHROPIC_API_KEY`, `GOCLAW_TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}` (same single bot — see plan.md §Single Telegram Bot Architecture), `GOCLAW_DB_*`; `docker-compose.goclaw.yml`; verify `GET /health` returns 200 — use the `latest` image (web UI + API in one binary, dashboard at `http://localhost:18790`); use **channel health diagnostics panel** in the GoClaw dashboard to verify Telegram bot token and channel connectivity before writing any custom diagnostic code (Apr 2026)
- [ ] T010a [P] Create `specs/001-investment-intel-poc/contracts/rest-api.md`: draft endpoint signatures and response schemas for auth, strategies, alerts, watchlist, billing, and internal service-to-service routes; **MUST include the standardised error response envelope** (see plan.md §Standardised Error Response Contract): `{ error: { code, message, details? } }` — all contract tests MUST validate error responses against this schema; prerequisite for all contract tests in Phases 3–8
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

**Goal**: Logged-in users can create, edit, activate, pause, and delete signal strategies for any
asset in the curated project list (BTC and ETH seeded for POC) with price-threshold, % price-change,
and RSI signal rules. Dashboard shows all strategies.

**Independent Test**: Create a BTC price-threshold strategy at $70k → save → retrieve → appears
as Active. Edit threshold → verify persisted. Delete → gone. No notifications needed.

### Tests — US1 (write first, must fail before implementation)

- [ ] T019 [P] [US1] Contract tests: `POST /strategies`, `GET /strategies`, `GET /strategies/:id`, `PUT /strategies/:id`, `DELETE /strategies/:id`, `PATCH /strategies/:id/status` in `backend/tests/contract/strategies_test.go`
- [ ] T020 [P] [US1] Unit tests for signal rule validator in `backend/internal/strategies/validator_test.go`: empty rules, contradictory rules (price above X AND below X), missing required fields, `window_minutes` required for percentage-change signal type, `window_minutes` value outside allowed set (5, 15, 60, 240, 1440) rejected, `window_minutes` ignored/disallowed for price-threshold and RSI signal types, `candle_minutes` required for RSI signal type, `candle_minutes` value outside allowed set (5, 15, 60) rejected, `candle_minutes` defaults to 15 when omitted for RSI, `candle_minutes` ignored/disallowed for price-threshold and percentage-change signal types, `candle_minutes` required for volume-spike / MACD-crossover / Bollinger-Band-breach signal types (defaults to 15 when omitted), `volume_threshold_pct` required for volume-spike (defaults to 200 when omitted; must be > 0), `cross_direction` required for MACD-crossover (only `bullish` or `bearish` accepted), `band_direction` required for Bollinger-Band-breach (only `upper` or `lower` accepted), unknown signal type rejected

### Implementation — US1

- [ ] T021 [US1] Migration `003_strategies.sql`: `app_strategies` (`id`, `user_id`, `name`, `asset VARCHAR NOT NULL REFERENCES app_projects(slug)`, `status` enum Active/Paused, `created_at`); `app_signal_rules` (`id`, `strategy_id`, `signal_type` enum {`price_threshold`, `pct_change`, `rsi`, `volume_spike`, `macd_crossover`, `bollinger_breach`}, `operator`, `threshold` NUMERIC — denominated in the asset's `quote_currency` for price-based signals (`price_threshold`, `pct_change`) or dimensionless for indicator signals, `window_minutes`, `candle_minutes`, `volume_threshold_pct`, `cross_direction`, `band_direction`); `window_minutes` MUST have a CHECK constraint restricting values to `{5, 15, 60, 240, 1440}` for `pct_change` signal type and MUST be NULL for all others; `candle_minutes` MUST have a CHECK constraint restricting values to `{5, 15, 60}` for `rsi`, `volume_spike`, `macd_crossover`, `bollinger_breach` signal types with DEFAULT 15, and MUST be NULL for `price_threshold` and `pct_change`; `volume_threshold_pct` MUST have a CHECK > 0 and DEFAULT 200 for `volume_spike`, NULL for all others; `cross_direction` MUST be CHECK `{bullish, bearish}` for `macd_crossover`, NULL for all others; `band_direction` MUST be CHECK `{upper, lower}` for `bollinger_breach`, NULL for all others; **indexes** (per plan.md §Database Index Strategy): `idx_strategies_user_id(user_id)`, `idx_strategies_user_status(user_id, status)`, `idx_signal_rules_strategy_id(strategy_id)`
- [ ] T022 [P] [US1] Implement strategy service in `backend/internal/strategies/service.go`: CRUD, status toggle, signal rule validation (cyclomatic complexity ≤ 10 per function)
- [ ] T023 [P] [US1] Implement strategy HTTP handlers in `backend/internal/strategies/handler.go`; wire into router; RLS enforced via Supabase JWT user ID
- [ ] T024 [US1] React — Strategy list page (`/dashboard`): fetches `GET /strategies`, shows Active/Paused badges, empty-state, loading skeleton, error state
- [ ] T025 [P] [US1] React — Strategy create/edit form (`/strategies/new`, `/strategies/:id/edit`): asset selector (populated from `GET /projects` — curated project list; not a hardcoded BTC/ETH dropdown), signal type selector (options: Price Threshold, % Price Change, RSI, Volume Spike, MACD Crossover, Bollinger Band Breach), operator + threshold inputs (threshold field MUST show the asset's `quote_currency` symbol — e.g., `$` for USD — as a prefix label; the currency symbol is derived from the selected asset's `quote_currency` in the `GET /projects` response), **time-window dropdown** (visible only when signal type = percentage change; options: 5 min, 15 min, 1 h, 4 h, 24 h mapping to `window_minutes` values 5, 15, 60, 240, 1440), **candle-size dropdown** (visible when signal type ∈ {RSI, Volume Spike, MACD Crossover, Bollinger Band Breach}; options: 5 min, 15 min, 1 h mapping to `candle_minutes` values 5, 15, 60; default = 15 min; label: "Candle Timeframe"), **volume threshold input** (visible only when signal type = Volume Spike; numeric input for `volume_threshold_pct`; default 200; label: "Volume Spike % of SMA"), **cross direction selector** (visible only when signal type = MACD Crossover; options: Bullish / Bearish), **band direction selector** (visible only when signal type = Bollinger Band Breach; options: Upper / Lower), client-side validation matching server rules
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
- [ ] T030 [P] [US4] Implement Telegram Bot API client in `backend/internal/telegram/bot.go`: `SendMessage(chatID, text)`, `GetWebhookUpdate()`, token-based deep-link generation; bot token read from `TELEGRAM_BOT_TOKEN` env var (single bot shared with GoClaw — see plan.md §Single Telegram Bot Architecture); register webhook at `https://api.<domain>/telegram/webhook`; webhook handler MUST filter updates to only process `/start` commands for account linking (all other inbound messages are ignored for POC)
- [ ] T031 [P] [US4] Implement linking service in `backend/internal/telegram/link_service.go`: generate one-time link token, handle bot `/start?token=` deep-link webhook, confirm link, send confirmation message, detect broken link (Telegram 403/blocked → set `telegram_chat_id = NULL`, emit web-app prompt)
- [ ] T032 [US4] Implement link/unlink HTTP handlers; register Telegram webhook endpoint with Telegram Bot API
- [ ] T033 [P] [US4] React — Settings page (`/settings/telegram`): shows link status, displays bot deep-link + instructions when unlinked, shows linked username + unlink button when linked, re-link prompt when broken
- [ ] T034 [US4] Integration test: link flow, unlink, broken-link detection in `backend/tests/integration/telegram_test.go`; MUST explicitly assert the Telegram 403/blocked → `telegram_chat_id = NULL` state transition and that a re-link prompt flag is set (per FR-019)

**Checkpoint**: US4 fully functional. User can link and unlink. CI green.

---

## Phase 5: User Story 2 — Real-Time Telegram Notifications (P2)

**Goal**: When an active strategy's signal condition fires, user receives a Telegram notification within 60 s.

**Independent Test**: With active BTC strategy + linked Telegram, simulate signal trigger → Telegram message delivered < 60 s with correct content. Paused strategy → no message. Two users → isolated.

### Tests — US2 (write first)

- [ ] T035 [P] [US2] Unit tests for signal evaluator in `backend/internal/signals/evaluator_test.go`: price threshold above/below, % change across multiple `window_minutes` values (5 min and 1440 min at minimum — 5 min verifies CryptoCompare → ai-service `pct_change` path; 1440 min verifies CoinGecko `price_change_percentage_24h` path), RSI overbought/oversold across multiple `candle_minutes` values (5 min and 60 min at minimum — verifies correct candle count is fetched: 70 vs. 840 1-min candles), volume spike above threshold (current vol / 20-period SMA > `volume_threshold_pct` / 100) and below threshold across `candle_minutes` values (verifies 300 vs. 1200 1-min candles fetched for 15 vs. 60), MACD crossover bullish and bearish (verifies MACD line vs. signal line cross detection, 525 vs. 2100 1-min candles for 15 vs. 60), Bollinger Band upper breach and lower breach across `candle_minutes` values (verifies close > upper band and close < lower band, 300 vs. 1200 1-min candles for 15 vs. 60); deterministic with mocked market data; MUST include a panic-recovery test (per NFR-REL-001): inject a panic in one evaluation tick and assert that the goroutine survives and the next tick executes successfully
- [ ] T036 [P] [US2] Contract test: `POST /signals/test-trigger` (admin-only test endpoint for CI) in `backend/tests/contract/signals_test.go`
- [ ] T037 [P] [US2] Unit tests for alert dispatcher in `backend/internal/alerts/dispatcher_test.go`: correct message format, paused strategy = no send, retry logic (3× exponential back-off), idempotency guard — duplicate `signal_rule_id + triggered_at` MUST NOT create a second Alert record (per NFR-REL-002)

### Implementation — US2

- [ ] T038 [US2] Migration `005_alerts.sql`: `app_alerts` (`id`, `strategy_id`, `user_id`, `signal_rule_id`, `asset`, `trigger_value`, `threshold`, `quote_currency VARCHAR(3) NOT NULL DEFAULT 'USD'`, `triggered_at`, `telegram_status` enum Pending/Sent/Failed, `retry_count`) — `quote_currency` is snapshotted from `app_projects` at alert creation time so historical alerts remain accurate even if the project's quote currency is changed later; **indexes** (per plan.md §Database Index Strategy): `idx_alerts_user_created(user_id, triggered_at DESC)`, `idx_alerts_user_strategy(user_id, strategy_id, triggered_at DESC)`, **UNIQUE** `idx_alerts_idempotency(signal_rule_id, triggered_at)` — enforces NFR-REL-002, `idx_alerts_pending_status(telegram_status) WHERE telegram_status IN ('Pending', 'Failed')` — partial index for re-drive worker and dashboard queries
- [ ] T039 [US2] Implement market data adapter interface `pkg/marketdata/provider.go` and CoinGecko implementation `pkg/marketdata/coingecko.go`; CoinGecko serves two purposes: (a) spot price for price-threshold signals, (b) `price_change_percentage_24h` for 1440-min % change signals; the adapter interface MUST accept `window_minutes` so the evaluator can route to the correct provider; the CoinGecko adapter MUST resolve the strategy asset slug (e.g., `btc`) to a CoinGecko coin ID (e.g., `bitcoin`) and quote currency (e.g., `usd`) via `app_projects.coingecko_id` and `app_projects.quote_currency` — load the slug→{coin-ID, quote_currency} map on startup and refresh every 5 min (or expose via ai-service `GET /projects`); CoinGecko calls MUST pass `vs_currencies={quote_currency}` (default `usd`); unit tests with mocked HTTP responses
- [ ] T039a [P] [US2] Implement CryptoCompare `histominute` adapter in `pkg/marketdata/cryptocompare.go` for intraday OHLCV data; MUST pass `tsym={quote_currency}` (from `app_projects.quote_currency`, default `USD`) on all API calls; serves multiple purposes: (a) `14 × candle_minutes` 1-min candles for RSI, (b) `20 × candle_minutes` 1-min candles for volume spike and Bollinger Bands, (c) `35 × candle_minutes` 1-min candles for MACD (**note: `candle_minutes=60` → 2100 candles, which exceeds CryptoCompare's 2000/call limit — adapter MUST use two paginated calls**: first call `limit=2000`, second call for remaining 100 candles using `toTs` set to earliest timestamp from first batch; reducing warm-up to 33 candles is NOT acceptable — MACD signal line requires all 35 candles for numerical stability), (d) `window_minutes` 1-min candles for intraday % price change signals (`window_minutes` ∈ {5, 15, 60, 240}) — Go fetches the OHLCV slice but does NOT compute % change; the slice is forwarded to ai-service which returns the computed `pct_change` value; the adapter accepts a generic `candle_count` parameter so the evaluator can request the right number per signal type; Go does NOT compute any indicators or OHLCV-derived math directly; unit tests with mocked HTTP covering RSI, volume spike, MACD, Bollinger Band, and % change paths at different candle sizes; **MUST include a test for MACD at `candle_minutes=60` verifying two paginated API calls are made and the resulting 2100 candles are correctly concatenated**
- [ ] T040 [P] [US2] Implement technical indicators endpoint in `ai-service/src/indicators/`: `rsi.py` (14-period RSI using `pandas-ta`; accepts `candle_minutes` parameter — resamples incoming 1-min OHLCV slice into candles of the requested size via `pandas.DataFrame.resample()` before computing RSI-14), `volume_spike.py` (20-period volume SMA via `pandas-ta.sma()`; resamples 1-min candles into requested candle size; returns `volume_spike_ratio` = current volume / SMA; threshold comparison happens in Go evaluator), `macd.py` (MACD(12,26,9) via `pandas-ta.macd()`; resamples 1-min candles; returns `macd_line`, `macd_signal`, `macd_histogram` for the latest candle; cross detection happens in Go evaluator by comparing current and previous candle values), `bollinger.py` (Bollinger Bands(20,2) via `pandas-ta.bbands()`; resamples 1-min candles; returns `bb_upper`, `bb_mid`, `bb_lower` and latest `close` for the most recent candle; breach detection happens in Go evaluator), `price_stats.py` (24 h open/high/low/close/pct_change), `pct_change.py` (intraday % price change: accepts the OHLCV slice, extracts first close and last close, computes `(last − first) / first × 100`; used for `window_minutes` ∈ {5, 15, 60, 240} — keeps all OHLCV math in Python; Go evaluator compares the returned `pct_change` against the user's threshold), `cache.py` (two cache key patterns — **indicator cache**: `indicators:{asset}:{candle_minutes}` TTL = 25 s for RSI/volume/MACD/Bollinger; **pct_change cache**: `pct_change:{asset}:{window_minutes}` TTL = 25 s for intraday % change — keyed by `window_minutes` because OHLCV slice sizes differ from indicator slices), `router.py` (`POST /indicators/{asset}` — accepts OHLCV slice + `candle_minutes` query param, returns `{ rsi, volume_spike_ratio, macd_line, macd_signal, macd_histogram, bb_upper, bb_mid, bb_lower, price_stats, quote_currency }`, checks indicator cache first; **`POST /pct_change/{asset}`** — accepts OHLCV slice + `window_minutes` query param, returns `{ pct_change, quote_currency }`, checks pct_change cache first — separate endpoint to avoid cache key collision with indicator cache); unit tests covering: RSI overbought (>70) / oversold (<30) / neutral, volume spike ratio above and below threshold, MACD bullish and bearish cross (compare consecutive candles), Bollinger upper and lower breach, intraday pct_change positive and negative across different window sizes, cache-hit path, and correct resampling for `candle_minutes` 5 vs. 60
- [ ] T040a [P] [US2] Update Go signal evaluator (`internal/signals/evaluator.go`) for indicator-based and OHLCV-derived signal types: read `candle_minutes` (for indicator signals) or `window_minutes` (for intraday % change) from the signal rule; determine candle count by signal type (RSI: `14 × candle_minutes`, volume spike / Bollinger: `20 × candle_minutes`, MACD: `35 × candle_minutes`, intraday % change: `window_minutes` 1-min candles); fetch the required 1-min candles via CryptoCompare adapter; **indicator signals** (RSI, volume spike, MACD, Bollinger): `POST` the slice to `ai-service/indicators/{asset}?candle_minutes={candle_minutes}`; **intraday % change** (`window_minutes` ≤ 240): `POST` the slice to `ai-service/pct_change/{asset}?window_minutes={window_minutes}` (separate endpoint and cache key — see plan.md §Indicator cache design); parse the response and evaluate the relevant field per signal type: RSI → compare `rsi` against overbought/oversold threshold; volume spike → compare `volume_spike_ratio` against `volume_threshold_pct / 100`; MACD crossover → detect `macd_line` crossing `macd_signal` in the configured `cross_direction` (requires comparing current and previous candle values from the response); Bollinger Band breach → compare latest `close` against `bb_upper` or `bb_lower` per `band_direction`; **intraday % change** → compare `pct_change` from ai-service response against the user's threshold (Go does NOT compute the percentage itself — ai-service owns all OHLCV math); **circuit breaker** (`sony/gobreaker`): wrap all ai-service HTTP calls in a circuit breaker (open after 5 consecutive failures, half-open probe after 30 s); when circuit is open, skip indicator/pct_change evaluation for that tick (log warning, do NOT fire alert on error); price-threshold and CoinGecko 24h % change signals are NOT affected by the circuit breaker (they don't call ai-service); cache indicator responses in Redis at `indicators:{asset}:{candle_minutes}` and pct_change responses at `pct_change:{asset}:{window_minutes}`, both with TTL = 25 s; unit tests MUST cover: RSI signal path with `candle_minutes=15` and `candle_minutes=60`, volume spike fire and no-fire, MACD bullish cross and bearish cross, Bollinger upper breach and lower breach, **intraday pct_change fire and no-fire for `window_minutes=15` and `window_minutes=240`**, indicator cache-hit skips HTTP call, **pct_change cache-hit skips HTTP call (separate cache key from indicators)**, different `candle_minutes` values use separate cache keys, **circuit breaker open → tick skipped with warning log**, **circuit breaker half-open → probe succeeds → circuit closes**, ai-service timeout falls back to skipping the tick (log + continue, do NOT fire alert on error)
- [ ] T041 [US2] Implement signal evaluation loop in `backend/internal/signals/poller.go`: 30 s Go ticker, single-pass evaluation across all active strategies per tick, publish `signal.triggered.{asset}` to NATS JetStream on match; payload MUST include `user_id`, `strategy_id`, `signal_rule_id`, `asset`, `trigger_value`, `threshold`, `triggered_at`; subject is asset-keyed (`signal.triggered.btc` / `signal.triggered.eth`) — user isolation is enforced in the dispatcher by `user_id` in the payload, not by subject; **signal cooldown**: before publishing, check Redis key `cooldown:{signal_rule_id}` (TTL = configurable, default 5 min via `SIGNAL_COOLDOWN_SECONDS` env var); if key exists, skip publish for this tick (condition is still true but already fired recently); if key absent, publish and set the key with TTL — this prevents duplicate NATS events when a condition remains true across multiple consecutive ticks and protects against alert spam; **ai-service readiness gate**: on startup, poll `GET ai-service:8000/health/ready` with 2 s interval and 60 s overall timeout before starting the 30 s ticker; if ai-service is not ready within 60 s, start the ticker but log CRITICAL warning and skip all indicator-based + pct_change evaluations (price-threshold + CoinGecko 24h % change still evaluate normally); **graceful shutdown**: on `SIGTERM` / `SIGINT`, (1) stop the 30 s ticker (no new ticks), (2) drain NATS connections (finish in-flight ACKs, close consumers + publisher), (3) close Redis pool, (4) close Postgres pool, (5) `http.Server.Shutdown(ctx)` with 15 s deadline; log each step; force-close on deadline exceeded; unit tests MUST cover: fires on first match, suppressed on second tick within cooldown, fires again after cooldown expires, **graceful shutdown stops ticker and drains NATS before closing pools**
- [ ] T042 [US2] Implement NATS consumer + alert dispatcher in `backend/internal/alerts/dispatcher.go`: consume `signal.triggered.{asset}` (durable consumer group `alerts-dispatcher`), persist Alert record scoped to `user_id`, call `telegram.bot.SendMessage`, update `telegram_status`; if user has no linked Telegram (`telegram_chat_id IS NULL`), set `telegram_status=Pending` and publish `alert.pending` to NATS (payload: `alert_id`, `user_id`, `triggered_at`) instead of attempting delivery; 3× retry with exponential back-off on Telegram API failure; surface `telegram_status=Failed` for web-app indicator (per FR-010, FR-025)
- [ ] T042a [P] [US2] Implement NATS consumer + re-drive worker in `backend/internal/alerts/redriver.go`: consume `alert.pending` from the `ALERT_QUEUE` stream (durable consumer `alerts-redriver`); on receipt, check if user has now linked Telegram; if linked, attempt `telegram.bot.SendMessage` and update `telegram_status=Sent`, then ACK the message; if still unlinked, call `msg.NakWithDelay(backoff)` — **do NOT re-publish a new message** (re-publishing creates stream bloat, duplicate-message risk, and false Prometheus pending-count alerts); backoff schedule: 1 min, 5 min, 15 min, 30 min, then 30 min for all remaining retries up to `MaxDeliver: 48`; when NATS exhausts `MaxDeliver` it auto-forwards the message to `alert.expired` subject — a separate lightweight handler in `redriver.go` consumes `alert.expired` and sets `telegram_status=Expired` in Postgres (no Telegram call); idempotency: check `telegram_status` in Postgres before attempting delivery — if already `Sent`, ACK without sending; unit tests in `backend/internal/alerts/redriver_test.go` MUST cover: deliver-on-link, NakWithDelay-when-unlinked, expired-DLQ-handler sets correct DB status, idempotent double-delivery
- [ ] T043 [P] [US2] Integration test: end-to-end signal trigger → NATS → alert persist → Telegram send (with mocked Telegram Bot API) in `backend/tests/integration/signal_notification_test.go`; MUST include the following cases: (a) happy path — linked user receives message; (b) durable consumer restart — message is redelivered and idempotency guard (`signal_rule_id + triggered_at`) prevents duplicate `Alert` record; (c) unlinked user — `alert.pending` is published and no Telegram call is made; (d) re-drive after linking — re-drive worker delivers queued alert within SLA
- [ ] T043a [P] [US2] React — Delivery-failure indicator on strategy dashboard card (`/dashboard`): when any alert for a strategy has `telegram_status=Failed`, show a red badge on the card; clicking it navigates to alert history filtered by that strategy (per FR-010); Vitest unit test for the badge component
- [ ] T043b [P] [US2] React — Delay-warning banner on dashboard: when any alert has `telegram_status=Pending` and `triggered_at` is older than 5 minutes, display an amber warning banner per FR-025; auto-dismiss when all pending alerts resolve; Vitest unit test

**Checkpoint**: Signal fires → Telegram notification < 60 s. Paused strategy = silent. Two-user isolation verified. CI green. ≥ 95% coverage on `internal/signals/` and `internal/alerts/`.

---

## Phase 6: User Story 5 — Alert History (P2)

**Goal**: Read-only alert history per user in the web app, filterable by strategy, reverse-chronological.

**Independent Test**: Strategy with 3 past alerts → open history page → all 3 listed with correct metadata → filter by strategy → only matching alerts shown → empty state when none.

### Tests — US5 (write first)

- [ ] T044 [P] [US5] Contract tests: `GET /alerts` (paginated, filter by `strategy_id`), `GET /alerts/:id` in `backend/tests/contract/alerts_test.go`

### Implementation — US5

- [ ] T045 [US5] Implement alert query handler in `backend/internal/alerts/handler.go`: `GET /alerts` with `strategy_id` filter, pagination (limit/offset), RLS scoped to authenticated user; enforce `limit` max 100 per FR-022 — clamp values > 100 to 100; default page size = 20
- [ ] T046 [P] [US5] React — Alert history page (`/alerts`): reverse-chronological list, strategy filter dropdown, pagination, loading skeleton, empty state, delivery-failure badge; `GET /alerts` via TanStack Query
- [ ] T046a [P] [US5] React — Per-row delivery-failure badge in alert history: each alert row MUST show a coloured status badge (`Sent` = green, `Pending` = amber, `Failed` = red, `Expired` = grey) per FR-010; Vitest unit test for badge states
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

- [ ] T051 [US3] Migration `006_watchlist.sql`: `app_watchlist_entries` (`id`, `user_id`, `project_slug`, `project_name`, `added_at`); **indexes**: `idx_watchlist_user_id(user_id)`; seed migration for curated project list (`app_projects` table with columns `slug VARCHAR PRIMARY KEY`, `display_name`, `symbol`, `coingecko_id`, `quote_currency VARCHAR(3) NOT NULL DEFAULT 'USD'`, `is_signal_asset BOOLEAN DEFAULT FALSE`; seed BTC and ETH with `is_signal_asset=TRUE` and `quote_currency='USD'`; this table is also the FK target for `app_strategies.asset` — **CRITICAL**: this migration MUST be numbered `002a_projects.sql` and placed in Phase 2 immediately after `002_users.sql`, NOT in Phase 7; the `app_projects` DDL + seed data MUST run before `003_strategies.sql` (Phase 3) because `app_strategies.asset` has a FK constraint on `app_projects(slug)`; the watchlist DDL (`app_watchlist_entries`) remains in `006_watchlist.sql`)
- [ ] T052 [P] [US3] Implement watchlist CRUD handlers in `backend/internal/watchlist/handler.go`; contract: add/remove/list entries; RLS scoped to user
- [ ] T053 [P] [US3] React — Watchlist page (`/watchlist`): grid of curated projects with add/remove toggle, loading, empty-state ("add projects to receive your daily digest"), error state
- [ ] T053a [P] [US3] Scaffold Python ai-service app in `ai-service/src/`: create `main.py` (FastAPI app factory, structlog JSON logging, CORS restricted to internal network), `config.py` (pydantic-settings loading `CRYPTOPANIC_TOKEN`, `REDIS_URL`, `ALLOWED_HOSTS`); expose `GET /health` (liveness) and `GET /health/ready` (readiness — checks Redis connectivity and CryptoPanic reachability); add to `docker-compose.yml` with `--network internal` so it is unreachable from the public internet (Traefik does NOT expose ai-service externally — only GoClaw and Go backend call it internally)
- [ ] T053b [P] [US3] Implement Redis-backed daily quota counter in `ai-service/src/news/quota.py`: `QuotaTracker` class with `increment(token) -> bool` (returns `False` when limit reached); key `cryptopanic:quota:{token}:{YYYY-MM-DD}`, TTL = 25 h; unit tests covering: under-limit returns True, at-limit returns False, key expires next day
- [ ] T053c [P] [US3] Implement curated project registry in `ai-service/src/projects/registry.py`: static mapping of `project_slug → { display_name, symbol, coingecko_id, quote_currency }` (all POC projects use `quote_currency: "USD"`); expose `GET /projects` endpoint returning the full list (response MUST include `quote_currency` per project so the React asset selector can display the correct currency symbol and the Go evaluator can pass it to market data adapters); used by React watchlist page, strategy create/edit form, and GoClaw skill to resolve slugs; unit test the registry mapping completeness (all ~20 POC projects present)
- [ ] T054 [US3] Implement CryptoPanic news adapter in `ai-service/src/news/cryptopanic.py`: fetch news by currency slug, parse items (title, url, published_at), DuckDuckGo fallback on quota exceeded; expose `GET /news/{project_slug}` endpoint; wire `QuotaTracker` from T053b to gate CryptoPanic calls
- [ ] T054a [P] [US3] Define abstract `NewsProvider` interface in `ai-service/src/news/provider.py`: `fetch_news(project_slug: str) -> list[NewsItem]` with `NewsItem` dataclass (title, url, published_at, source); `CryptoPanicAdapter` in T054 MUST implement this interface; unit tests asserting interface contract; interface contract also documented in `contracts/rest-api.md`
- [ ] T054b [P] [US3] Implement news enrichment endpoint in `ai-service/src/enrichment/`: `sentiment.py` (VADER sentiment scoring on news headlines — `compound` score per item), `router.py` (`POST /enrich/news` — accepts `list[NewsItem]`, returns `list[EnrichedNewsItem]` with `sentiment_score` and `is_duplicate` flag based on URL hash deduplication); GoClaw digest skill calls this before passing items to the LLM — shorter prompt, higher summary quality; unit tests for sentiment scoring, deduplication logic, and empty-list handling
- [ ] T055 [US3] Create GoClaw digest agent in `goclaw/agents/digest-agent/`: `AGENT.md` (persona: "Investment Intel Digest Bot"; instructions to fetch each user's watchlist via backend API, call news endpoint per project, summarise with LLM, send via `message` tool); `HEARTBEAT.md` (confirm digest sent) — **note (Apr 2026)**: Knowledge Graph sharing is configured separately from workspace sharing in the GoClaw agent setup UI; if KG access is required for the digest agent, enable KG sharing independently
- [ ] T056 [P] [US3] Create GoClaw `crypto-digest` skill in `goclaw/agents/digest-agent/skills/crypto-digest.md`: skill steps — (1) `GET /watchlist` for user via internal service token (see T057a), (2) `web_fetch` CryptoPanic per project, (3) LLM summarise (claude-3-5-haiku, ≤ 200 tokens/project), (4) `message` to Telegram chat ID; skill MUST explicitly handle the no-news case: if `web_fetch` returns zero items for a project, insert a section reading "No significant updates in the last 24 hours" (per FR-015) — **note (Apr 2026)**: parallel `web_fetch` across watchlist projects is reliable; use subagent `waitAll` for concurrent news fetch to reduce total digest latency; auto-retry and token tracking are built in
- [ ] T056a [P] [US3] Wire price-movement data into GoClaw digest skill (FR-014): the digest agent MUST call `POST /indicators/{asset}` (or a dedicated `GET /price_stats/{asset}` endpoint on ai-service) per watchlisted project to retrieve 24 h OHLCV summary (open, high, low, close, pct_change); include the price-movement section alongside news in each project’s digest section; unit test the skill template output includes both news and price data
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
- [ ] T073 [P] Add Playwright E2E test: full happy-path flow (register → link Telegram mock → create strategy → view history → add watchlist item) in `frontend/tests/e2e/happy_path.spec.ts`; MUST run at both desktop (1280×720) and mobile (375×667) viewports per SC-006
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
| 1 – Setup | — | T001–T010c (incl. T001a, T009a) | Blocks everything |
| 2 – Foundation | US0 (auth) | T011–T018 | Blocks US1–US5 |
| 3 – Strategy Config | US1 (P1) | T019–T027 | MVP gate |
| 4 – Telegram Linking | US4 (P1) | T028–T034 | Blocks US2 + US3 delivery |
| 5 – Notifications | US2 (P2) | T035–T043b (incl. T042a, T043a, T043b) | Core value delivery |
| 6 – Alert History | US5 (P2) | T044–T047 (incl. T046a) | Pairs with US2 |
| 7 – Watchlist + Digest | US3 (P3) | T048–T059 (incl. T056a) | GoClaw + AI layer |
| 8 – Billing | US0 | T060–T066 | Revenue gate |
| 9 – Admin | US0 | T067–T070 | Ops tooling |
| 10 – Quality Gates | — | T071–T078a | Constitution compliance |
| 11 – Deploy | — | T079–T083 | POC handoff |

**Total tasks**: 102 (96 prior + 6 added by analysis remediation: T001a, T009a, T043b, T046a, T056a; T034/T035/T037/T040/T045/T073 expanded in-place; T020/T021/T025/T035/T039a/T040/T040a further expanded for volume-spike, MACD-crossover, and Bollinger-Band-breach signal types; T021/T025/T039/T051 updated for asset extensibility — `app_strategies.asset` changed from enum to VARCHAR FK to `app_projects`; adding new tokens is a data-only change, no schema migration required; T021/T025/T038/T039/T039a/T051/T053c updated for explicit USD quote-currency denomination — `app_projects.quote_currency` column added, market data adapters pass it as CoinGecko `vs_currencies` / CryptoCompare `tsym`, UI shows currency symbol on threshold inputs, alerts snapshot quote_currency at creation time)

**Architecture review changes applied** (no new tasks; existing tasks expanded):
- **M2 — pct_change cache key fix**: T040 + T040a split `POST /pct_change/{asset}` into a separate endpoint with cache key `pct_change:{asset}:{window_minutes}` (avoids collision with indicator cache keyed by `candle_minutes`)
- **M4 — Circuit breaker**: T040a adds `sony/gobreaker` wrapping all ai-service HTTP calls; circuit opens after 5 failures, half-open probe after 30 s
- **M5 — Single Telegram bot**: T010, T030 clarified that `GOCLAW_TELEGRAM_BOT_TOKEN = TELEGRAM_BOT_TOKEN` (one bot, Go owns webhook, GoClaw only sends outbound); plan.md §Single Telegram Bot Architecture added
- **M6 — Graceful shutdown**: T041 adds `SIGTERM`/`SIGINT` handler: stop ticker → drain NATS → close Redis → close Postgres → shutdown HTTP (15 s deadline)
- **M7 — ai-service readiness gate**: T041 adds startup poll of `GET /health/ready` (2 s interval, 60 s timeout) before starting evaluation ticker
- **M3 — MACD pagination**: T039a mandates two paginated CryptoCompare calls for `candle_minutes=60` (2100 candles); 'accept 33 candles' alternative removed
- **M1 — Request budget**: plan.md §Market Data Feed now includes CryptoCompare request budget analysis with mitigation strategies
- **L2 — DB indexes**: T021, T038, T051 now include explicit index definitions per plan.md §Database Index Strategy
- **L3 — Migration ordering**: T051 makes `002a_projects.sql` placement in Phase 2 a CRITICAL requirement, not a note
- **L4 — Error response contract**: T010a requires standardised error envelope `{ error: { code, message, details? } }` per plan.md §Standardised Error Response Contract
