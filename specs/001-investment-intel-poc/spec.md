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
for BTC or ETH (e.g., price crossing a moving-average threshold, RSI reaching an overbought level,
or a volume spike above a defined percentage). They save the strategy and it immediately becomes
active.

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
  from the web app.
- **FR-002**: Each strategy MUST be scoped to a single asset (BTC or ETH for POC) and MUST
  contain at least one signal rule before it can be saved as active.
- **FR-003**: Supported signal types for POC MUST include: price threshold (above/below a fixed
  value), percentage price change over a configurable time window, and RSI crossing an overbought
  or oversold level. All signal matching MUST be deterministic rule-based logic; no AI/ML
  inference is used for signal evaluation.
- **FR-004**: The system MUST validate signal rules for logical consistency before saving and
  display actionable error messages when validation fails.
- **FR-005**: Users MUST be able to view the current status (Active / Paused) of all their
  strategies on a single dashboard screen.

**Telegram Notifications**

- **FR-006**: The system MUST deliver a Telegram notification to a user within 60 seconds of a
  monitored signal condition being confirmed. The signal evaluation engine MUST poll market data
  at a minimum frequency of every 30 seconds to satisfy this SLA.
- **FR-007**: Each notification MUST include: asset name, signal type, current value at trigger
  time, configured threshold, and strategy name.
- **FR-008**: Notifications MUST only be sent for strategies that are in Active status.
- **FR-009**: Notifications from different users' strategies MUST be fully isolated; no user
  receives another user's alert.
- **FR-010**: If Telegram delivery fails, the system MUST retry at least 3 times with
  exponential back-off and surface a delivery-failure indicator in the web app.

**Watchlist & Daily Digest**

- **FR-011**: Users MUST be able to add and remove projects from their personal watchlist via
  the web app; the watchlist MUST support any project the system has data coverage for.
- **FR-012**: The system MUST send a consolidated daily Telegram digest to each user who has at
  least one project on their watchlist.
- **FR-013**: The digest MUST be delivered before 09:00 in the user's configured timezone
  (defaulting to UTC+0 if not set).
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
  detect this and surface a re-link prompt in the web app.

**General**

- **FR-020**: The system MUST support open self-registration with email + password. Registration
  MUST include email verification. Users MUST be able to reset their password via an email link.
  All protected pages MUST redirect unauthenticated users to the login screen.
- **FR-021**: All user data (strategies, watchlists, Telegram link status) MUST persist across
  sessions.
- **FR-022**: The web app MUST provide a read-only alert history view listing all past triggered
  alerts for the authenticated user, sortable by date and filterable by strategy.
- **FR-023**: The daily digest content MUST be generated by passing raw news items from the news
  API through an LLM summarisation step; the summarised output (not raw feed content) is what is
  sent via Telegram.
- **FR-024**: The system MUST integrate with a third-party crypto news API (provider determined
  at planning) to retrieve news items for watchlisted projects; the integration MUST be
  abstracted behind an interface so the provider can be swapped without changing digest logic.

### Key Entities

- **User**: Represents a registered account; holds authentication credentials, timezone
  preference, and Telegram link status.
- **Strategy**: A named, user-owned configuration that targets a single asset and contains one
  or more signal rules; has an Active/Paused lifecycle status.
- **Signal Rule**: A single condition within a strategy (e.g., "BTC RSI > 70"); belongs to
  exactly one strategy; defines the signal type, asset, operator, and threshold value.
- **Alert**: An event record created when a signal rule's condition is confirmed; carries the
  trigger timestamp, asset value at trigger, delivery status to Telegram, and is displayed in
  the user's alert history view.
- **Watchlist Entry**: An association between a User and a tracked project; drives which projects
  appear in that user's daily digest.
- **News Item**: A raw article or headline retrieved from the third-party news API for a specific
  project; serves as input to the LLM summarisation step for digest generation.
- **Digest**: A daily generated summary for a user, containing one section per watchlist entry;
  carries delivery status and the scheduled send time.

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
- **SC-007**: All pages and key interactions load and respond within 2 seconds under the
  expected POC user load (≤ 50 concurrent users).

### Assumptions

- **A-001**: Asset price and signal data (RSI, volume, price) are sourced from a third-party
  market data feed that supports polling at ≤ 30-second intervals; the specific provider is
  selected at planning. Data acquisition implementation is out of scope for this spec.
- **A-002**: The Telegram bot is registered and operational; bot setup is a deployment
  prerequisite, not a feature to be built.
- **A-003**: "Projects" available for the watchlist are limited to a curated list maintained by
  the team for the POC; a full asset search/discovery flow is out of scope.
- **A-004**: User timezone defaults to UTC+0 if not explicitly set; timezone configuration UI
  is in scope but an exhaustive timezone selector is not required (common timezones sufficient).
- **A-005**: POC target scale is ≤ 50 registered users; no high-availability or horizontal
  scaling architecture is required at this stage.
- **A-006**: News content for the daily digest is sourced from a separate third-party crypto
  news API (distinct from the market data feed in A-001); the specific provider (e.g.,
  CryptoPanic, Messari, CoinGecko News) is selected at planning.
- **A-007**: An LLM API (provider TBD at planning, e.g., OpenAI, Anthropic) is available for
  digest summarisation; LLM call costs are acceptable within POC budget constraints.
