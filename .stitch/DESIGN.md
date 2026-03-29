# Investment Intel — Stitch Design Source of Truth

> Generated from `ui-ux-pro-max` skill on 2026-03-28.
> All screen generation MUST reference this file.
> Per-page overrides live in `.stitch/pages/[page].md`.

---

## Project

- **Name:** Investment Intel
- **Stitch Project ID:** `17004846820291981820`
- **Domain:** Crypto fintech — real-time signal alerts, strategy configuration, Telegram notifications, daily digest
- **Device:** DESKTOP primary (1440px design width); all screens must be responsive to 375px / 768px / 1024px / 1440px
- **Framework:** React 19 + TailwindCSS v4 (screens rendered as Tailwind HTML for visual reference)

---

## Design System (from ui-ux-pro-max)

### Style
- **Name:** Dark Mode (OLED)
- **Vibe:** Dark, professional, trustworthy crypto trading platform. OLED-optimised, eye-strain prevention.
- **Key Effects:** Minimal amber glow (`text-shadow: 0 0 10px #F59E0B`), dark-to-light transitions, high readability, visible focus rings

### Color Palette

| Role | Hex | Tailwind Custom Name | Usage |
|------|-----|----------------------|-------|
| Primary (Gold) | `#F59E0B` | `primary` | Trust, financial strength — CTAs, active states, badges |
| Secondary (Amber) | `#FBBF24` | `secondary` | Highlights, hover tones |
| Accent (Purple) | `#8B5CF6` | `accent` | Tech actions — primary buttons, links, Stripe CTA |
| Background | `#0F172A` | `bg-app` | App background (OLED deep navy-black) |
| Surface | `#1E293B` | `surface` | Cards, sidebars, input backgrounds |
| Surface Raised | `#273549` | `surface-raised` | Elevated cards, dropdown menus |
| Border | `#334155` | `border-muted` | Dividers, input borders |
| Text Primary | `#F8FAFC` | `text-primary` | Headings, body text |
| Text Muted | `#94A3B8` | `text-muted` | Labels, timestamps, placeholder text |
| Success (Emerald) | `#10B981` | `success` | Active status, positive signals, "Paid" badges |
| Warning (Amber) | `#F59E0B` | `warning` | Pending/queued states |
| Error (Red) | `#EF4444` | `error` | Failures, delivery errors, destructive actions |
| Info (Blue) | `#3B82F6` | `info` | Informational badges, Telegram linking |

### Typography

**Primary recommendation (Crypto/Web3):**
- **Heading Font:** Orbitron (400, 500, 600, 700) — futuristic, crypto-native
- **Body Font:** Exo 2 (300, 400, 500, 600, 700) — readable, technical

**Alternative (Financial Trust):**
- IBM Plex Sans — use for dense data tables and legal/billing copy where readability trumps branding

**CSS Import:**
```css
@import url('https://fonts.googleapis.com/css2?family=Exo+2:wght@300;400;500;600;700&family=Orbitron:wght@400;500;600;700&display=swap');
```

**Tailwind Config:**
```js
fontFamily: {
  display: ['Orbitron', 'sans-serif'],
  body: ['Exo 2', 'sans-serif'],
}
```

### Spacing Scale
`4px / 8px / 16px / 24px / 32px / 48px / 64px`

### Border Radius
- Cards: `rounded-xl` (12px)
- Buttons: `rounded-lg` (8px)
- Inputs: `rounded-lg` (8px)
- Badges: `rounded-full` (9999px)

### Shadows
- Cards: `shadow-md` → `shadow-lg` on hover
- Modals: `shadow-xl`
- Glow (gold): `shadow-[0_0_20px_rgba(245,158,11,0.3)]`
- Glow (purple): `shadow-[0_0_20px_rgba(139,92,246,0.3)]`

---

## Icons
Use **Lucide React** icons exclusively. No emojis as UI icons.
Consistent sizing: `w-5 h-5` (body), `w-4 h-4` (inline/badge)

---

## Layout System

### Authenticated App Shell
- **Left Sidebar:** 240px fixed, `bg-surface` (#1E293B), border-right `border-muted`
- **Top Bar:** 64px fixed, `bg-app/80 backdrop-blur`, border-bottom
- **Main Content:** `ml-64 pt-16`, padding `p-6 md:p-8`, `bg-app`
- **Sidebar Nav Items:** icon + label, `py-3 px-4`, hover `bg-surface-raised`, active: `bg-accent/10 text-accent border-l-2 border-accent`
- **Sidebar Footer:** user avatar + name + plan badge + logout button

### Unauthenticated (Auth Pages)
- Split layout: left hero panel (gradient `from-app via-surface to-accent/20`) + right form panel (`bg-surface`)
- Max width: `max-w-md` for form container

---

## Component Specs

### Buttons
```
Primary (Accent): bg-accent text-white font-semibold px-6 py-3 rounded-lg hover:brightness-110 transition-all duration-200 cursor-pointer
Secondary (Outline): border-2 border-primary text-primary bg-transparent hover:bg-primary/10 px-6 py-3 rounded-lg transition-all duration-200 cursor-pointer
Destructive: bg-error/10 text-error border border-error/30 hover:bg-error/20 px-6 py-3 rounded-lg
Ghost: text-text-muted hover:text-text-primary hover:bg-surface-raised px-4 py-2 rounded-lg
```

### Cards
```
bg-surface border border-border-muted rounded-xl p-6 shadow-md
hover: shadow-lg transform -translate-y-0.5 transition-all duration-200 cursor-pointer
```

### Inputs
```
bg-surface-raised border border-border-muted rounded-lg px-4 py-3 text-text-primary placeholder-text-muted
focus: border-primary outline-none ring-2 ring-primary/20
error: border-error ring-2 ring-error/20
label: text-sm font-medium text-text-muted mb-1 block
```

### Badges / Status Pills
```
Active:  bg-success/10 text-success border border-success/30 text-xs font-semibold px-2.5 py-0.5 rounded-full
Paused:  bg-text-muted/10 text-text-muted border border-text-muted/30 text-xs font-semibold px-2.5 py-0.5 rounded-full
Error:   bg-error/10 text-error border border-error/30 text-xs font-semibold px-2.5 py-0.5 rounded-full
Pending: bg-warning/10 text-warning border border-warning/30 text-xs font-semibold px-2.5 py-0.5 rounded-full
```

### Loading States (UX guideline: skeleton for ops > 300ms)
```
Skeleton: animate-pulse bg-surface-raised rounded-lg (replace real content)
Spinner:  animate-spin w-5 h-5 border-2 border-accent border-t-transparent rounded-full
```

### Error States
```
Inline field error: <p role="alert" class="text-error text-sm mt-1">message</p>
Page-level error: bg-error/10 border border-error/30 rounded-lg p-4 text-error with Lucide AlertCircle icon
Empty state: centered, Lucide icon (w-12 h-12 text-text-muted), heading, subtext, optional CTA button
```

### Charts (from ui-ux-pro-max chart search)
- **Sparklines / trend:** Recharts `AreaChart` with `fill: #F59E0B20`, `stroke: #F59E0B`
- **OHLC / candlestick:** TradingView Lightweight Charts (bullish `#10B981`, bearish `#EF4444`)
- **Streaming area:** opacity-fading area, dark grid lines
- **Gauge / quota:** Custom SVG donut ring or ApexCharts

---

## Pages Inventory (from plan.md + tasks.md)

| Route | Task | US | Screen File |
|-------|------|----|-------------|
| `/sign-in` | T017 | US0 | `.stitch/designs/sign-in.html` |
| `/sign-up` | T017 | US0 | `.stitch/designs/sign-up.html` |
| `/forgot-password` | T017 | FR-020 | `.stitch/designs/forgot-password.html` |
| `/reset-password` | T017 | FR-020 | `.stitch/designs/reset-password.html` |
| `/dashboard` | T024, T043a | US1, US2 | `.stitch/designs/dashboard.html` |
| `/strategies/new` | T025 | US1 | `.stitch/designs/strategy-create.html` |
| `/strategies/:id/edit` | T025 | US1 | `.stitch/designs/strategy-edit.html` |
| `/alerts` | T046 | US5 | `.stitch/designs/alert-history.html` |
| `/watchlist` | T053, T058 | US3 | `.stitch/designs/watchlist.html` |
| `/settings` | T033, T058a | US4 | `.stitch/designs/settings.html` |
| `/billing` | T064 | US0 | `.stitch/designs/billing.html` |
| `/admin` | T070 | US0 | `.stitch/designs/admin.html` |

---

## Critical Spec Requirements Per Screen

### Auth Screens (T017, FR-020)
- Email + password form with `<label for=...>` on all inputs (accessibility)
- `role="alert"` on inline validation errors (aria-live)
- Loading state on submit button: spinner replaces text, disabled
- Sign Up: email verification notice banner after submission ("Check your inbox")
- Forgot Password: success state ("Reset link sent")
- No placeholder-only inputs

### Dashboard (T024, T043a — US1, US2)
- Strategy cards grid with **Active** / **Paused** status badges
- **Delivery-failure indicator:** red badge on card if any alert has `telegram_status=Failed` (FR-010)
- Clicking failure badge → navigates to `/alerts?strategy_id=...`
- Empty state when no strategies: "Create your first strategy" CTA
- Loading skeleton (3 placeholder cards with animate-pulse)

### Strategy Create/Edit (T025, FR-003)
- Asset selector: BTC | ETH (radio or segmented control)
- Signal type selector with 3 options (FR-003):
  1. **Price Threshold** — operator (above/below) + value (USD)
  2. **% Price Change** — direction (up/down) + percentage + time window (minutes)
  3. **RSI** — period (fixed 14, labelled), threshold value (default 70 overbought / 30 oversold), operator (above/below), both thresholds user-configurable
- Client-side validation: at least one signal rule before save; no contradictory rules
- Error message when no rules added: "Add at least one signal rule to activate this strategy"

### Alert History (T046, FR-022 — US5)
- **Filter by Strategy** dropdown (primary filter per spec) — NOT just asset/channel
- Pagination: default 20/page, limit/offset, prev/next
- Delivery-failure badge per row (red indicator if `telegram_status=Failed`)
- Empty state: "No alerts yet — activate a strategy to start receiving signals"
- Columns: Strategy Name | Asset | Signal Type | Trigger Value | Threshold | Timestamp | Delivery Status

### Watchlist (T053, T058 — US3)
- **Purpose: digest subscription**, NOT a price tracker
- Curated project grid: add/remove toggle per project tile
- Projects shown: BTC, ETH, SOL, ARB, LINK, MATIC, DOT, AVAX, etc. (curated list per A-003)
- Empty state: "Add projects to receive your daily digest"
- Digest status section: last digest timestamp + status (Sent/Pending/Failed)
- Re-link prompt if Telegram not linked: "Link your Telegram account to receive digests"

### Settings (T033, T058a — US4)
- **Telegram Linking section (most important):**
  - State A (unlinked): bot deep-link button + step-by-step instructions (1. Click link → 2. Opens Telegram → 3. Send /start → 4. Return here)
  - State B (linked): green "Connected" badge + Telegram username + "Unlink" button + "Send test message" button
  - State C (broken link detected): orange warning banner "Your Telegram connection was broken" + "Re-link" CTA (FR-019)
- **Timezone selector (T058a):** IANA timezone dropdown, hint: "Your daily digest will be sent at 09:00 [selected timezone]"
- **Account info:** email, display name, change password link

### Billing (T064 — US0 / Stripe)
- Current subscription status card
- **"Subscribe" CTA → Stripe Checkout redirect** (not a generic pricing page)
- **"Manage subscription" → Stripe Customer Portal link**
- If `subscription_status = cancelled/past_due`: warning banner with re-subscribe CTA
- If `subscription_status = active`: manage/cancel options
- Loading state while fetching subscription status
- **402 gate:** if user hits gated feature, redirect here with explanation banner

### Admin (T070 — US0)
- Users table: Email | Subscription Status | Telegram Linked | Last Active | Actions
- Actions: override subscription status (active/cancelled/past_due)
- Admin-only route — "Access Denied" (403) shown to non-admins
- Filter by subscription status

---

## Anti-Patterns (DO NOT USE)
- ❌ Light backgrounds anywhere in the app
- ❌ Emojis as UI icons
- ❌ Missing `cursor-pointer` on interactive elements
- ❌ Layout-shifting hover transforms
- ❌ Placeholder-only form inputs (always pair with `<label>`)
- ❌ No feedback after form submit
- ❌ Visual-only error indicators (must use `role="alert"` or `aria-live`)
- ❌ Content hidden behind fixed navbars
- ❌ Watchlist as price tracker (it's a digest subscription list)
- ❌ Billing page without Stripe-specific CTAs
- ❌ Alert history without strategy-name filter
