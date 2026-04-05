# Feature Specification: Investment Intel AI Agents — POC

**Feature Branch**: `001-investment-intel-poc`  
**Created**: 2026-03-18  
**Status**: Clarified  
**Input**: User description: "investment intel ai agents - POC with React WebApp for configs/strategies on BTC/ETH signals, Telegram notifications per user strategy, and daily digest/news per user watchlist"

## Clarifications

| # | Question | Decision |
|---|----------|----------|
| Q1 | AI agents vs rule-based signal matching? | **Option B** — Rule-based signal engine for all real-time alerts; LLM used only to summarise news items in the daily digest |
| Q2 | Signal evaluation frequency? | **Option B** — Poll market data every 30 seconds; satisfies the 60 s notification SLA with headroom |
| Q3 | User registration model? | **Option A** — Open self-registration with email + password; full auth flow required (verification, password reset) |
| Q4 | Alert history in web app? | **Option A** — Read-only alert history per strategy visible inside the web app |
| Q5 | News data source for digest? | **Option B** — Separate third-party crypto news API, distinct from market data feed; provider selected at planning phase |

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Signal Strategy Configuration (Priority: P1)

A trader visits the web app, creates a new strategy, and selects one or more signal types to watch
for BTC or ETH (e.g., price crossing a threshold, RSI reaching an overbought level, a volume spike
above a defined percentage, a MACD crossover, or a Bollinger Band breach). They save the strategy
and it immediately becomes active.

**Why this priority**: Without the ability to define what to watch for, no notifications or digests
can be meaningfully personalised. This story is the foundation of the entire POC value proposition.

**Independent Test**: A new user can create a strategy with at least one signal rule for BTC or ETH,
save it, and retrieve it back with all settings intact — without any notification or digest feature
being present.

**Acceptance Scenarios**:

1. **Given** a logged-in user with no existing strategies, **When** they create a strategy named
   "BTC Breakout" targeting BTC with a price-threshold signal at $70,000, **Then** the strategy is
   saved and appears in their strategy list with status "Active".
2. **Given** an existing active strategy, **When** the user edits the signal threshold and saves,
   **Then** the updated threshold is persisted and the strategy remains active.
3. **Given** a user attempting to save a strategy with no signal rules, **When** they submit the
   form, **Then** a clear validation message is shown and nothing is saved.
4. **Given** a user, **When** they deactivate a strategy, **Then** it stops generating alerts and
   is shown as "Paused" in the list.

---

### User Story 2 — Real-Time Telegram Notifications per Strategy (Priority: P2)

When a monitored signal condition is met for an asset the user is watching, the user receives a
Telegram message containing the signal name, asset, current value, and the threshold that was
crossed. Each notification is scoped to the user's own active strategies only.

**Why this priority**: Timely notification is the core utility of the product. Without it the
signal strategies have no output.

**Independent Test**: With an active strategy for BTC and a connected Telegram account, a simulated
signal trigger (fired from the admin/test tooling) must result in a Telegram message delivered to
that user within 60 seconds, containing correct signal details.

**Acceptance Scenarios**:

1. **Given** a user with an active BTC price-threshold strategy and a linked Telegram account,
   **When** the BTC price crosses the configured threshold, **Then** a Telegram notification is
   delivered to the user's chat within 60 seconds, showing asset name, current price, and
   strategy name.
2. **Given** a user with a paused strategy, **When** the signal condition is met, **Then** no
   Telegram notification is sent.
3. **Given** two users each with independent strategies on the same asset, **When** the signal
   fires, **Then** each user receives only their own notification; neither sees the other's.
4. **Given** a user whose Telegram account is not yet linked, **When** a signal fires, **Then**
   the notification is queued and the web app shows a prompt to link their Telegram account.

---

### User Story 3 — Watchlist & Daily Digest (Priority: P3)

A user maintains a personal watchlist of crypto projects (beyond BTC/ETH, e.g., SOL, ARB, any
supported token). Each morning they receive a Telegram message summarising overnight news, price
movements, and notable on-chain or social activity for every project on their watchlist.

**Why this priority**: The digest provides ambient awareness and complements the real-time alerts.
It is lower priority because real-time alerts are the primary POC value, but it rounds out the
daily usage loop.

**Independent Test**: A user with a watchlist containing at least two projects receives a single
consolidated Telegram digest before 09:00 local time every day. The digest must contain at least
one item per watchlisted project and must not include projects not on the watchlist.

**Acceptance Scenarios**:

1. **Given** a user with SOL and ARB on their watchlist, **When** the daily digest runs at the
   scheduled time, **Then** the user receives a single Telegram message with a section for SOL
   and a section for ARB, each containing at least one news item or market summary from the past
   24 hours.
2. **Given** a user with an empty watchlist, **When** the digest runs, **Then** no message is
   sent and the user sees an empty-state prompt in the web app to add projects.
3. **Given** a user who adds a new project to their watchlist at 14:00, **When** the next daily
   digest runs, **Then** that project is included in the digest.
4. **Given** a user, **When** they remove a project from their watchlist, **Then** the next
   digest does not include that project.

---

### User Story 4 — Telegram Account Linking (Priority: P1)

A user connects their Telegram account to the web app so that notifications and digests can be
delivered. The linking flow is self-service and requires no admin intervention.

**Why this priority**: Telegram delivery depends entirely on this link being established. It is a
prerequisite for stories 2 and 3 and must be part of the initial setup flow.

**Independent Test**: A new user follows the in-app linking instructions and completes the
connection. After completion, a test message is delivered to confirm the link works — end-to-end,
without any backend intervention.

**Acceptance Scenarios**:

1. **Given** a logged-in user with no Telegram account linked, **When** they follow the in-app
   linking instructions and complete the authorisation step in Telegram, **Then** their account
   is marked as linked and a confirmation message appears both in the web app and in Telegram.
2. **Given** a user who has already linked their Telegram, **When** they visit the account
   settings, **Then** the linked Telegram username is displayed and an option to unlink is
   present.
3. **Given** a user who abandons the linking flow halfway, **When** they return to the app,
   **Then** their account remains unlinked and they can restart the process.

---

### User Story 5 — Alert History (Priority: P2)

A user opens the web app and views a chronological list of every alert that has fired across all
their strategies. Each entry shows the strategy name, asset, signal type, trigger value, and
timestamp. This gives users a permanent audit trail independent of their Telegram chat history.

**Why this priority**: Pairs directly with US2 (notifications) and adds accountability and
debuggability for the signal engine with minimal additional implementation cost — the `Alert`
entity is already part of the data model.

**Independent Test**: With at least one strategy that has previously triggered, a user can
open the alert history view, see all past alerts in reverse-chronological order, and filter by
strategy — without any notification or digest feature needing to be present.

**Acceptance Scenarios**:

1. **Given** a user with a strategy that has triggered 3 times, **When** they open the alert
   history page, **Then** all 3 alerts are listed in reverse-chronological order showing
   strategy name, asset, trigger value, threshold, and timestamp.
2. **Given** a user with multiple strategies, **When** they filter the history by a specific
   strategy, **Then** only alerts for that strategy are shown.
3. **Given** a user with no triggered alerts, **When** they visit the alert history, **Then**
   a clear empty-state message is shown.
4. **Given** a new alert fires, **When** the user refreshes the alert history page, **Then**
   the new alert appears at the top of the list.

---

### Edge Cases

- What happens when a signal data source is temporarily unavailable? The system must queue
  undelivered notifications and attempt re-delivery; the user must be informed of any delay
  exceeding 5 minutes.
- What happens when a user has more than one strategy that triggers on the same asset at the same
  time? Each matching strategy produces its own distinct notification; they are not merged.
- What happens if the daily digest contains no fresh news for a watchlisted project? A brief
  "no significant updates" placeholder is included for that project rather than silently omitting it.
- What happens when a user's Telegram chat is blocked or the bot is removed? The system marks
  the link as broken, queues the notifications, and surfaces a re-link prompt in the web app.
- What happens when a user sets contradictory signal rules (e.g., "price above $X" AND "price
  below $X" simultaneously)? The form must validate and reject logically impossible combinations
  before saving.

---

## Requirements *(mandatory)*

### Functional Requirements

**Strategy & Signal Configuration**

- **FR-001**: Users MUST be able to create, edit, activate, pause, and delete signal strategies
  from the web app. Deleting a strategy MUST cascade-delete its `app_signal_rules` rows
  (`ON DELETE CASCADE`). Historical `app_alerts` rows MUST be retained with `strategy_id` set
  to NULL (`ON DELETE SET NULL`) so the alert history audit trail is preserved.
- **FR-002**: Each strategy MUST be scoped to a single asset chosen from the curated project
  list (BTC and ETH seeded for POC; additional tokens can be added to `app_projects` without
  schema changes) and MUST contain at least one signal rule before it can be saved as active.
  Strategy creation MUST reject any asset where `app_projects.is_signal_asset = FALSE`; this
  prevents users from creating signal strategies for watchlist-only projects that lack market
  data feed configuration.
  All price-denominated values (thresholds, trigger values, price-change calculations) MUST be
  quoted in the asset's configured `quote_currency` (default **USD** for all POC assets). The
  `quote_currency` is stored per project in `app_projects` and passed to market data adapters
  (CoinGecko `vs_currencies`, CryptoCompare `tsym`). Multi-currency support is an extensibility
  point; for POC every project uses USD.
- **FR-003**: Supported signal types for POC MUST include: price threshold (above/below a fixed
  value), percentage price change over a configurable time window, RSI crossing an overbought
  or oversold level, volume spike (current volume exceeds a percentage of the 20-period SMA),
  MACD crossover (MACD line crosses the signal line in a configured direction), and Bollinger
  Band breach (close price crosses outside an upper or lower Bollinger Band). All signal
  matching MUST be deterministic rule-based logic; no AI/ML inference is used for signal
  evaluation.
  
  **RSI signals** MUST be calculated over a 14-period window with a user-selectable candle size
  (`candle_minutes`). POC-allowed `candle_minutes` values: **5, 15, 60** (representing 5-min,
  15-min, and 1-hour candles; default = **15**). The evaluator fetches `14 × candle_minutes`
  1-minute candles from CryptoCompare `histominute` and resamples them into the chosen candle
  size before computing RSI (e.g., `candle_minutes=60` → fetch 840 1-min candles → resample to
  14 hourly candles → compute RSI-14). All candle data is sourced from CryptoCompare
  `histominute`; no additional API is required. The default overbought threshold is 70 and the
  default oversold threshold is 30; both thresholds MUST be user-configurable per signal rule.
  
  **Percentage price change** signals MUST specify a `window_minutes` value from the following
  POC-allowed set: **5, 15, 60, 240, 1440** (representing 5 min, 15 min, 1 h, 4 h, 24 h). The
  system MUST reject any value outside this set. For windows ≤ 240 min the Go evaluator
  fetches the required OHLCV slice from CryptoCompare `histominute` and forwards it to the
  Python ai-service, which computes `pct_change` = `(current_close − close_at_T−window) /
  close_at_T−window × 100`; the Go evaluator then compares the returned value against the
  user's threshold. For the 1440 min (24 h) window the evaluator uses CoinGecko's
  pre-computed `price_change_percentage_24h` field directly (no ai-service call needed).
  
  **Volume spike** signals fire when the current candle's volume exceeds a user-configured
  percentage (`volume_threshold_pct`, default 200%) of the 20-period simple moving average of
  volume. The candle size is controlled by `candle_minutes` (same POC-allowed set as RSI:
  **5, 15, 60**; default = **15**). The evaluator fetches `20 × candle_minutes` 1-minute candles
  from CryptoCompare `histominute`, resamples them, and computes the volume SMA and current
  volume ratio. No additional API is required.
  
  **MACD crossover** signals fire when the MACD line crosses the signal line in a user-configured
  direction (`cross_direction`: **bullish** = MACD crosses above signal, **bearish** = MACD
  crosses below signal). MACD is computed with standard parameters (fast = 12, slow = 26,
  signal = 9) over candles sized by `candle_minutes` (same POC-allowed set: **5, 15, 60**;
  default = **15**). The evaluator fetches `35 × candle_minutes` 1-minute candles from
  CryptoCompare `histominute` (35 candles covers the 26-period slow EMA + 9-period signal
  with warm-up headroom), resamples them, and computes MACD via `pandas-ta`. No additional
  API is required.
  
  **Bollinger Band breach** signals fire when the current close price crosses outside a
  Bollinger Band in a user-configured direction (`band_direction`: **upper** = close > upper
  band, **lower** = close < lower band). Bollinger Bands are computed with standard parameters
  (period = 20, standard deviations = 2.0) over candles sized by `candle_minutes` (same
  POC-allowed set: **5, 15, 60**; default = **15**). The evaluator fetches `20 × candle_minutes`
  1-minute candles from CryptoCompare `histominute`, resamples them, and computes Bollinger
  Bands via `pandas-ta`. No additional API is required.
  
  **Price-threshold** signals do not use `window_minutes` or `candle_minutes` (always evaluated
  against the current spot price). This deterministic-only constraint applies exclusively to the
  signal evaluation engine; ML/LLM usage for news summarisation and sentiment scoring is
  permitted and governed by FR-023.
- **FR-004**: The system MUST validate signal rules for logical consistency before saving and
  display actionable error messages when validation fails.
- **FR-005**: Users MUST be able to view the current status (Active / Paused) of all their
  strategies on a single dashboard screen.

**Telegram Notifications**

- **FR-006**: The system MUST deliver a Telegram notification to a user within 60 seconds of a
  monitored signal condition being confirmed. The signal evaluation engine MUST poll market data
  at a minimum frequency of every 30 seconds to satisfy this SLA.
- **FR-007**: Each notification MUST include: asset name, signal type, current value at trigger
  time (with quote-currency symbol, e.g., "$70,123.45"), configured threshold (with the same
  currency symbol), and strategy name.
- **FR-008**: Notifications MUST only be sent for strategies that are in Active status.
- **FR-009**: Notifications from different users' strategies MUST be fully isolated; no user
  receives another user's alert.
- **FR-010**: If Telegram delivery fails, the system MUST retry at least 3 times with
  exponential back-off and surface a delivery-failure indicator in the web app. The delivery-failure
  indicator MUST be visible on both the strategy dashboard card and in the alert history row;
  it MUST persist until the alert is successfully delivered. There is no explicit user
  "acknowledge" action for POC; the indicator auto-clears when `telegram_status` transitions
  to `Sent`. Failed alerts older than 24 hours transition to `Expired` and the badge is removed.

**Watchlist & Daily Digest**

- **FR-011**: Users MUST be able to add and remove projects from their personal watchlist via
  the web app; the watchlist MUST support any project from the system's curated project list
  (see A-003); a full asset search or discovery flow is out of scope for POC.
- **FR-012**: The system MUST send a consolidated daily Telegram digest to each user who has at
  least one project on their watchlist.
- **FR-013**: The digest MUST be delivered before 09:00 UTC (defaulting to UTC+0 if the user
  has not set a timezone). For the POC, the GoClaw cron runs at a single fixed time (08:30 UTC);
  per-user timezone-aware scheduling is deferred to post-POC. Users in timezones west of UTC
  may receive the digest before their local 09:00; users east of UTC will receive it after.
- **FR-014**: Each digest MUST contain a section per watchlisted project covering: notable news
  items from the past 24 hours and a summary price movement (open, high, low, close over 24 h).
- **FR-015**: Projects with no fresh news MUST still appear in the digest with an explicit
  "no significant updates" note.

**Telegram Account Linking**

- **FR-016**: Users MUST be able to link their Telegram account from the web app settings using
  a self-service flow requiring no admin assistance.
- **FR-017**: Upon successful linking the system MUST send a confirmation message to the user's
  Telegram chat.
- **FR-018**: Users MUST be able to unlink their Telegram account at any time; unlinking MUST
  immediately halt all outbound Telegram messages for that user.
- **FR-019**: If a Telegram link becomes broken (bot removed, chat blocked), the system MUST
  detect this and surface a re-link prompt in the web app. Detection is **reactive only** for
  POC: a Telegram 403/blocked response during message delivery triggers the broken-link state.
  Proactive periodic health-checking of linked Telegram accounts is deferred to post-POC.

**General**

- **FR-020**: The system MUST support open self-registration with email + password. Registration
  MUST include email verification. Users MUST be able to reset their password via an email link.
  All protected pages MUST redirect unauthenticated users to the login screen.
- **FR-021**: All user data (strategies, watchlists, Telegram link status) MUST persist across
  sessions.
- **FR-022**: The web app MUST provide a read-only alert history view listing all past triggered
  alerts for the authenticated user, sortable by date and filterable by strategy. Alert history
  MUST be paginated; the default page size is 20 records; the API MUST accept `limit` (max 100)
  and `offset` query parameters.
- **FR-023**: The daily digest content MUST be generated by passing raw news items from the news
  API through an LLM summarisation step; the summarised output (not raw feed content) is what is
  sent via Telegram.
- **FR-024**: The system MUST integrate with a third-party crypto news API (provider determined
  at planning) to retrieve news items for watchlisted projects; the integration MUST be
  abstracted behind an interface so the provider can be swapped without changing digest logic.
  The interface contract (method signatures, return types, error handling) MUST be defined in
  `specs/001-investment-intel-poc/contracts/rest-api.md` before implementation begins.

- **FR-025**: The system MUST queue undelivered notifications when Telegram delivery fails or
  when the user's Telegram account is not yet linked. If delivery has not succeeded within
  5 minutes of the original trigger, the web app MUST display a delay warning visible on the
  dashboard. Queued notifications for an unlinked account MUST be reattempted for up to 24 hours
  after the user completes Telegram linking; notifications older than 24 hours MAY be discarded.

**Seed Configuration & Strategy Definition Format**

- **FR-026**: All business-level configuration — signal type definitions (with parameter schemas,
  allowed values, and defaults), POC project seed data, and system tunables (poll interval,
  cooldown, cache TTL, allowed `candle_minutes` / `window_minutes` sets) — MUST be externalised
  into a declarative YAML seed file (`config/seed.yaml`) that is loaded at application startup.
  The Go backend MUST validate the seed file against a JSON Schema (`config/seed.schema.json`)
  on boot and refuse to start if validation fails. Adding a new signal type parameter, adjusting
  allowed value sets, or seeding a new project MUST NOT require code changes — only a seed file
  update (and, for new projects, a corresponding `app_projects` DB insert). The seed file MUST
  NOT contain secrets, credentials, or environment-specific values (those stay in env vars).
- **FR-027**: The system MUST define a **Strategy Definition Format (SDF)** — a portable,
  self-describing JSON structure that fully represents a strategy and its signal rules. The SDF
  MUST be documented as a JSON Schema (`contracts/strategy-definition.schema.json`) and serve as
  the canonical wire format for `POST /strategies` request bodies and `GET /strategies/:id`
  response bodies. The format MUST use a discriminated-union pattern (discriminator field:
  `signal_type`) so that each signal type carries only its relevant parameters and new signal
  types can be added by extending the schema's `oneOf` array without breaking existing clients.
  The SDF MUST be self-validatable: the JSON Schema alone is sufficient for a consumer (human,
  frontend form, or LLM) to produce a valid strategy without reading application code. This is
  an **extensibility foundation** for post-POC LLM-generated strategies ("text → SDF → save");
  the LLM integration itself is out of scope for POC but the format MUST be designed to support
  it.
- **FR-028**: The system MUST expose a `POST /strategies/import` endpoint that accepts one or
  more strategies in SDF format and creates them for the authenticated user after full validation
  (same rules as `POST /strategies`). This enables bulk creation and is the entry point for
  future LLM-generated strategy pipelines. A corresponding `GET /strategies/:id/export` endpoint
  MUST return the strategy in SDF format. Both endpoints are part of the REST API and require
  authentication.

### Key Entities

- **User**: Represents a registered account; holds authentication credentials, timezone
  preference, and Telegram link status.
- **Strategy**: A named, user-owned configuration that targets a single asset and contains one
  or more signal rules; has an Active/Paused lifecycle status.
- **Signal Rule**: A single condition within a strategy (e.g., "BTC RSI > 70"); belongs to
  exactly one strategy; defines the signal type, asset, operator, threshold value (denominated
  in the asset's `quote_currency` for price-based signals; dimensionless for RSI/volume/MACD/
  Bollinger), optional
  `window_minutes` (required for percentage-change signals; POC-allowed values: 5, 15, 60, 240,
  1440 minutes), optional `candle_minutes` (required for RSI, volume-spike, MACD-crossover,
  and Bollinger-Band-breach signals; POC-allowed values: 5, 15, 60; default 15), optional
  `volume_threshold_pct` (required for volume-spike; default 200), and optional
  `cross_direction` (required for MACD-crossover: bullish/bearish) and `band_direction`
  (required for Bollinger-Band-breach: upper/lower).
- **Alert**: An event record created when a signal rule's condition is confirmed; carries the
  trigger timestamp, asset value at trigger, delivery status to Telegram, and is displayed in
  the user's alert history view.
- **Watchlist Entry**: An association between a User and a tracked project; drives which projects
  appear in that user's daily digest.
- **News Item**: A raw article or headline retrieved from the third-party news API for a specific
  project; serves as input to the LLM summarisation step for digest generation.
- **Digest**: A daily generated summary for a user, containing one section per watchlist entry;
  carries delivery status and the scheduled send time.
- **Strategy Definition (SDF)**: A portable JSON document conforming to
  `contracts/strategy-definition.schema.json` that fully describes a strategy and its signal
  rules in a machine-readable, self-validatable format. Used as the wire format for strategy
  CRUD and import/export; designed as the future target format for LLM-generated strategies.

---

## Non-Functional Requirements

### Security

- **NFR-SEC-001**: All protected API routes MUST require a valid Supabase-issued JWT; unauthenticated
  requests MUST receive a 401 response.
- **NFR-SEC-002**: Row Level Security (RLS) MUST be enabled on all user-scoped application
  tables (`app_strategies`, `app_signal_rules`, `app_alerts`, `app_watchlist_entries`,
  `app_users`); queries MUST be scoped to the authenticated user's `user_id`. Global reference
  tables (`app_projects`) are public-read and do not require RLS.
- **NFR-SEC-003**: Stripe webhook endpoints MUST validate the `Stripe-Signature` header; requests
  failing signature verification MUST be rejected with a 400 response.
- **NFR-SEC-004**: API rate limiting MUST be enforced via Traefik middleware; authenticated users
  MUST be limited to 120 requests/minute; unauthenticated paths (login, register) MUST be limited
  to 20 requests/minute.

### Performance

- **NFR-PERF-001**: REST API endpoints MUST respond within 200 ms p95 under the POC baseline
  load of ≤ 50 registered users generating typical interactive traffic (estimated peak ≤ 10
  concurrent requests; each user averaging ≤ 2 req/s during active sessions).
- **NFR-PERF-002**: React SPA pages MUST achieve LCP ≤ 2.5 s and CLS ≤ 0.1 on a median device
  profile when assets are served via Cloudflare CDN.
- **NFR-PERF-003**: The signal evaluation loop MUST process all active strategies within each 30 s
  tick with peak memory consumption not exceeding 256 MB.
- **NFR-PERF-004**: The GoClaw digest agent MUST complete per-user digest generation and delivery
  within 5 minutes of the scheduled cron trigger; peak memory consumption MUST NOT exceed 512 MB.
  Throughput SLA: the pipeline MUST complete all digests for ≤ 50 users × ≤ 10 watchlist projects
  each within the 5-minute window. GoClaw is a third-party binary; measurement MUST use GoClaw's
  built-in OTLP traces and Prometheus metrics (LLM call duration, `message` tool latency) rather
  than Go pprof. Memory is measured via container runtime metrics (`docker stats` or cAdvisor)
  during Phase 10 performance validation.
- **NFR-PERF-005**: Performance regressions exceeding 10% against the established baseline MUST
  block merging until resolved, per constitution Principle IV.

### Reliability

- **NFR-REL-001**: The signal evaluation loop MUST be self-recovering; a panic or unhandled error
  in a single evaluation tick MUST NOT crash the service — it MUST log the error, skip that tick,
  and resume on the next interval.
- **NFR-REL-002**: NATS JetStream consumers MUST use durable subscriptions with at-least-once
  delivery semantics; duplicate alert dispatch MUST be prevented via idempotency checks on
  `signal_rule_id + triggered_at` before persisting an `Alert` record.
- **NFR-REL-003**: The Go evaluator's HTTP calls to ai-service MUST be protected by a circuit
  breaker. After 5 consecutive failures (timeout or 5xx), the circuit opens for 30 seconds;
  during this window, indicator-based and pct_change evaluations are skipped (logged as warning,
  no alert fired). Price-threshold and CoinGecko 24h % change signals are NOT affected. After
  30 s the circuit half-opens; a successful probe closes the circuit and resumes normal evaluation.
- **NFR-REL-004**: On `SIGTERM` / `SIGINT` the Go backend MUST perform graceful shutdown: stop
  the evaluation ticker, drain NATS connections (finish in-flight ACKs), close Redis and Postgres
  pools, and stop the HTTP server with a 15-second deadline. This ensures zero-downtime deploys
  and no lost NATS acknowledgements.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A first-time user can complete account registration, link their Telegram account,
  create their first strategy, and add projects to their watchlist in under 5 minutes.
- **SC-002**: Signal-triggered Telegram notifications are delivered within 60 seconds of the
  condition being confirmed in 95% of cases under normal operating conditions.
- **SC-003**: Daily digest messages are delivered before the user's configured 09:00 deadline on
  100% of days during the POC evaluation period.
- **SC-004**: 90% of POC users are able to create a valid, active strategy on their first
  attempt without requiring support assistance.
- **SC-005**: The system correctly isolates each user's notifications — zero cross-user alert
  leakage across all test scenarios.
- **SC-006**: The web app is usable on both desktop and mobile browsers without layout breakage
  or lost functionality.
- **SC-007**: REST API endpoints respond within 200 ms p95; React SPA pages achieve LCP ≤ 2.5 s
  and CLS ≤ 0.1 on a median device profile; all measurements taken under the expected POC load
  of ≤ 50 concurrent users.

### Assumptions

- **A-001**: Asset price and signal data (RSI, volume, price) are sourced from a third-party
  market data feed that supports polling at ≤ 30-second intervals; the specific provider is
  selected at planning. Data acquisition implementation is out of scope for this spec.
- **A-002**: A **single** Telegram bot is registered and operational; bot setup is a deployment
  prerequisite, not a feature to be built. The same bot token is used by Go backend (real-time
  alerts + account linking webhook) and GoClaw (daily digest delivery via `sendMessage`).
  Telegram delivers all inbound updates to Go backend's webhook endpoint; GoClaw only sends
  outbound messages.
- **A-003**: "Projects" available for the watchlist are limited to a curated list maintained by
  the team for the POC; a full asset search/discovery flow is out of scope.
- **A-004**: User timezone defaults to UTC+0 if not explicitly set; timezone configuration UI
  is in scope but an exhaustive timezone selector is not required (common timezones sufficient).
- **A-005**: POC target scale is ≤ 50 registered users; no high-availability or horizontal
  scaling architecture is required at this stage.
- **A-006**: News content for the daily digest is sourced from a separate third-party crypto
  news API (distinct from the market data feed in A-001); the provider selected for POC is
  CryptoPanic (free tier, 50 req/day per token) with DuckDuckGo as a fallback on quota
  exhaustion. Provider selection is confirmed in `plan.md` Research Notes.
- **A-007**: An LLM API is available for digest summarisation; the provider confirmed at
  planning is **Anthropic claude-3-5-haiku** via GoClaw provider config. LLM call costs are
  acceptable within POC budget constraints.
- **A-008**: Alert records, digest records, and raw news items are retained indefinitely during
  the POC evaluation period; no automated purge or archival policy is implemented at this stage.
  Data retention policy will be revisited and formalised post-POC.

---

## Glossary

| Term | Definition |
|------|------------|
| **Strategy** | A named, user-owned configuration targeting a single asset from the curated project list (BTC and ETH for POC) that contains one or more signal rules and has an Active/Paused lifecycle. |
| **Signal Rule** | A single condition within a strategy (e.g., "BTC RSI > 70") defining signal type, operator, threshold, optional `window_minutes` (required for % change signals; allowed POC values: 5, 15, 60, 240, 1440), optional `candle_minutes` (required for RSI, volume-spike, MACD-crossover, and Bollinger-Band-breach signals; allowed POC values: 5, 15, 60; default 15), optional `volume_threshold_pct` (required for volume-spike; default 200), optional `cross_direction` (required for MACD-crossover: bullish/bearish), and optional `band_direction` (required for Bollinger-Band-breach: upper/lower). |
| **Signal** | An event produced when a signal rule's condition evaluates to true during a polling tick. Signals are deterministic and rule-based (no ML). |
| **Alert** | A persistent record created when a signal fires; carries trigger metadata, delivery status, and is displayed in the alert history view. |
| **Notification** | The Telegram message delivered to a user when an alert is created. "Alert" = the data record; "Notification" = the delivery act. |
| **Watchlist Entry** | An association between a User and a tracked crypto project; drives which projects appear in the daily digest. |
| **Digest** | A daily AI-summarised Telegram message containing news and price summaries for each project on a user's watchlist. |
| **News Item** | A raw article or headline from the third-party news API, before LLM summarisation. |
| **Enriched News Item** | A news item after sentiment scoring and deduplication by the Python ai-service. |
| **Curated Project** | A crypto project (e.g., SOL, ARB) from the team-maintained registry available for watchlist selection. |
| **Curated Project List** | The full set of `app_projects` entries returned by `GET /projects`; includes both signal assets and watchlist-only projects. |
| **Signal Asset** | A curated project with `app_projects.is_signal_asset = TRUE`, meaning it has a configured market data feed (CoinGecko + CryptoCompare) and can be used as a strategy target. BTC and ETH are signal assets for POC. Projects with `is_signal_asset = FALSE` are available for the watchlist but cannot be used in signal strategies. |
| **Quote Currency** | The fiat currency in which asset prices and thresholds are denominated. Stored per project in `app_projects.quote_currency`; defaults to `USD` for all POC assets. Market data adapters use this value (CoinGecko `vs_currencies`, CryptoCompare `tsym`). |
| **Linking** | The process of connecting a user's web-app account to their Telegram chat via a bot deep-link. |
| **Strategy Definition Format (SDF)** | A portable JSON structure (with JSON Schema) that fully represents a strategy and its signal rules. Used as the wire format for `POST /strategies` and `GET /strategies/:id`, and the target format for future LLM-generated strategies ("text → SDF → save"). |
| **Seed Configuration** | A declarative YAML file (`config/seed.yaml`) that externalises all business-level configuration: signal type definitions with parameter schemas, POC project seed data, and system tunables. Validated against `config/seed.schema.json` at boot. |
| **Strategy Import** | Bulk creation of strategies via `POST /strategies/import` accepting an array of SDF documents; validates each against the JSON Schema and the same business rules as single creation. |
