# Design System Master File

> **LOGIC:** When building a specific page, first check `design-system/pages/[page-name].md`.
> If that file exists, its rules **override** this Master file.
> If not, strictly follow the rules below.

---

**Project:** Investment Intel
**Generated:** 2026-03-28 18:23:06
**Category:** Fintech/Crypto

---

## Global Rules

### Color Palette

| Role | Hex | CSS Variable |
|------|-----|--------------|
| Primary | `#F59E0B` | `--color-primary` |
| Secondary | `#FBBF24` | `--color-secondary` |
| CTA/Accent | `#8B5CF6` | `--color-cta` |
| Background | `#0F172A` | `--color-background` |
| Text | `#F8FAFC` | `--color-text` |

**Color Notes:** Gold trust + purple tech

### Typography

- **Heading Font:** Orbitron
- **Body Font:** Exo 2
- **Mood:** crypto, web3, futuristic, tech, blockchain, digital
- **Google Fonts:** [Orbitron + Exo 2](https://fonts.google.com/share?selection.family=Exo+2:wght@300;400;500;600;700|Orbitron:wght@400;500;600;700)

**CSS Import:**
```css
@import url('https://fonts.googleapis.com/css2?family=Exo+2:wght@300;400;500;600;700&family=Orbitron:wght@400;500;600;700&display=swap');
```

### Spacing Variables

| Token | Value | Usage |
|-------|-------|-------|
| `--space-xs` | `4px` / `0.25rem` | Tight gaps |
| `--space-sm` | `8px` / `0.5rem` | Icon gaps, inline spacing |
| `--space-md` | `16px` / `1rem` | Standard padding |
| `--space-lg` | `24px` / `1.5rem` | Section padding |
| `--space-xl` | `32px` / `2rem` | Large gaps |
| `--space-2xl` | `48px` / `3rem` | Section margins |
| `--space-3xl` | `64px` / `4rem` | Hero padding |

### Shadow Depths

| Level | Value | Usage |
|-------|-------|-------|
| `--shadow-sm` | `0 1px 2px rgba(0,0,0,0.05)` | Subtle lift |
| `--shadow-md` | `0 4px 6px rgba(0,0,0,0.1)` | Cards, buttons |
| `--shadow-lg` | `0 10px 15px rgba(0,0,0,0.1)` | Modals, dropdowns |
| `--shadow-xl` | `0 20px 25px rgba(0,0,0,0.15)` | Hero images, featured cards |

---

## Component Specs

### Buttons

```css
/* Primary Button */
.btn-primary {
  background: #8B5CF6;
  color: white;
  padding: 12px 24px;
  border-radius: 8px;
  font-weight: 600;
  transition: all 200ms ease;
  cursor: pointer;
}

.btn-primary:hover {
  opacity: 0.9;
  transform: translateY(-1px);
}

/* Secondary Button */
.btn-secondary {
  background: transparent;
  color: #F59E0B;
  border: 2px solid #F59E0B;
  padding: 12px 24px;
  border-radius: 8px;
  font-weight: 600;
  transition: all 200ms ease;
  cursor: pointer;
}
```

### Cards

```css
.card {
  background: #0F172A;
  border-radius: 12px;
  padding: 24px;
  box-shadow: var(--shadow-md);
  transition: all 200ms ease;
  cursor: pointer;
}

.card:hover {
  box-shadow: var(--shadow-lg);
  transform: translateY(-2px);
}
```

### Inputs

```css
.input {
  padding: 12px 16px;
  border: 1px solid #E2E8F0;
  border-radius: 8px;
  font-size: 16px;
  transition: border-color 200ms ease;
}

.input:focus {
  border-color: #F59E0B;
  outline: none;
  box-shadow: 0 0 0 3px #F59E0B20;
}
```

### Modals

```css
.modal-overlay {
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(4px);
}

.modal {
  background: white;
  border-radius: 16px;
  padding: 32px;
  box-shadow: var(--shadow-xl);
  max-width: 500px;
  width: 90%;
}
```

---

## Style Guidelines

**Style:** Dark Mode (OLED)

**Keywords:** Dark theme, low light, high contrast, deep black, midnight blue, eye-friendly, OLED, night mode, power efficient

**Best For:** Night-mode apps, coding platforms, entertainment, eye-strain prevention, OLED devices, low-light

**Key Effects:** Minimal glow (text-shadow: 0 0 10px), dark-to-light transitions, low white emission, high readability, visible focus

### Page Pattern

**Pattern Name:** Horizontal Scroll Journey

- **Conversion Strategy:** Immersive product discovery. High engagement. Keep navigation visible.
28,Bento Grid Showcase,bento,  grid,  features,  modular,  apple-style,  showcase", 1. Hero, 2. Bento Grid (Key Features), 3. Detail Cards, 4. Tech Specs, 5. CTA, Floating Action Button or Bottom of Grid, Card backgrounds: #F5F5F7 or Glass. Icons: Vibrant brand colors. Text: Dark., Hover card scale (1.02), video inside cards, tilt effect, staggered reveal, Scannable value props. High information density without clutter. Mobile stack.
29,Interactive 3D Configurator,3d,  configurator,  customizer,  interactive,  product", 1. Hero (Configurator), 2. Feature Highlight (synced), 3. Price/Specs, 4. Purchase, Inside Configurator UI + Sticky Bottom Bar, Neutral studio background. Product: Realistic materials. UI: Minimal overlay., Real-time rendering, material swap animation, camera rotate/zoom, light reflection, Increases ownership feeling. 360 view reduces return rates. Direct add-to-cart.
30,AI-Driven Dynamic Landing,ai,  dynamic,  personalized,  adaptive,  generative", 1. Prompt/Input Hero, 2. Generated Result Preview, 3. How it Works, 4. Value Prop, Input Field (Hero) + 'Try it' Buttons, Adaptive to user input. Dark mode for compute feel. Neon accents., Typing text effects, shimmering generation loaders, morphing layouts, Immediate value demonstration. 'Show, don't tell'. Low friction start.
- **CTA Placement:** Floating Sticky CTA or End of Horizontal Track
- **Section Order:** 1. Intro (Vertical), 2. The Journey (Horizontal Track), 3. Detail Reveal, 4. Vertical Footer

---

## Anti-Patterns (Do NOT Use)

- ❌ Light backgrounds
- ❌ No security indicators

### Additional Forbidden Patterns

- ❌ **Emojis as icons** — Use SVG icons (Heroicons, Lucide, Simple Icons)
- ❌ **Missing cursor:pointer** — All clickable elements must have cursor:pointer
- ❌ **Layout-shifting hovers** — Avoid scale transforms that shift layout
- ❌ **Low contrast text** — Maintain 4.5:1 minimum contrast ratio
- ❌ **Instant state changes** — Always use transitions (150-300ms)
- ❌ **Invisible focus states** — Focus states must be visible for a11y

---

## Responsive Layout

### Approach

**Mobile-first** — base styles target 375 px; wider layouts added progressively.

### Breakpoints (Tailwind mapping)

| Token | Width | Tailwind | Behaviour |
|-------|-------|----------|-----------|
| `mobile` | 375 px | _(default)_ | Single column; stacked cards; bottom nav; hamburger menu |
| `tablet` | 768 px | `md:` | Two-column where appropriate; side nav or top nav; data tables visible |
| `laptop` | 1024 px | `lg:` | Full sidebar nav; multi-column dashboards; side-by-side panels |
| `desktop` | 1440 px | `xl:` | Max-width container (1280 px); comfortable whitespace |

### Touch Targets

- All interactive elements ≥ **44 × 44 px** on viewports < 768 px (WCAG 2.5.5 Level AAA)
- Spacing between adjacent touch targets ≥ 8 px

### Per-Page Responsive Behaviour

| Page | Mobile (375 px) | Tablet (768 px) | Desktop (1024 px+) |
|------|-----------------|-----------------|--------------------|
| **Dashboard** (`/dashboard`) | Single-column strategy cards; quick stats stacked above | Two-column card grid; stats row | Three-column grid; sidebar quick stats |
| **Strategy Create/Edit** | Single-column form; signal rule builder stacked; execution config below rules | Two-column: rules left, preview/execution right | Same as tablet with wider inputs |
| **Template Picker** | Single-column scrollable card list | Two-column card grid | Three-column card grid |
| **Alert History** (`/alerts`) | Stacked cards (no table); each card shows signal, time, status | Data table with horizontal scroll if needed | Full data table; filters in sidebar |
| **Watchlist** (`/watchlist`) | Vertical list with swipe-to-remove | Two-column grid | Three-column grid with inline actions |
| **Portfolio Dashboard** (`/portfolio`) | Stacked sections: summary → positions (cards) → trade history (cards) | Two-column: summary + positions left, trade history right | Full data tables; summary bar at top |
| **Trade History** | Stacked cards per trade | Data table | Full data table with filters sidebar |
| **Settings / Telegram** | Single-column | Single-column centred (max 600 px) | Same as tablet |
| **Auth pages** | Centred card (90% width) | Centred card (max 420 px) | Same as tablet |
| **Billing** | Single-column plan cards | Two-column plan cards | Three-column plan cards |
| **Admin** | Stacked user cards | Data table | Full data table with search sidebar |

### Data Table → Card Reflow Pattern

On viewports < 768 px, all data tables (alert history, trade history, portfolio positions, admin user list) MUST reflow to **stacked cards**:

```
/* Desktop: <table> */        /* Mobile: stacked cards */
┌──────┬──────┬──────┐        ┌──────────────────────┐
│ Col1 │ Col2 │ Col3 │        │ Label1: Value1       │
├──────┼──────┼──────┤   →    │ Label2: Value2       │
│  A   │  B   │  C   │        │ Label3: Value3       │
└──────┴──────┴──────┘        └──────────────────────┘
```

Implement via a shared `<ResponsiveTable>` component that renders `<table>` on `md:` and card list on mobile. Each column definition includes a `label` used as the card field label.

### Navigation

- **Mobile (< 768 px)**: Bottom tab bar (Dashboard, Alerts, Watchlist, Portfolio, More) + hamburger for secondary pages (Settings, Billing, Admin)
- **Tablet (768 px+)**: Collapsible sidebar nav or top nav bar
- **Desktop (1024 px+)**: Persistent sidebar nav

### Component-Level Guidelines

| Component | Mobile adaptation |
|-----------|-------------------|
| `Button` | Full-width (`w-full`) in forms; icon-only variant in toolbars |
| `Card` | Full-width with reduced padding (`p-3` vs `p-6`) |
| `Input` | `font-size: 16px` minimum (prevents iOS zoom on focus) |
| `Modal` | Full-screen sheet (slides up from bottom) |
| `Badge` | Same size; ensure touch target met when clickable |
| `EmptyState` | Illustration scales down; text stays readable |

---

## Pre-Delivery Checklist

Before delivering any UI code, verify:

- [ ] No emojis used as icons (use SVG instead)
- [ ] All icons from consistent icon set (Heroicons/Lucide)
- [ ] `cursor-pointer` on all clickable elements
- [ ] Hover states with smooth transitions (150-300ms)
- [ ] Light mode: text contrast 4.5:1 minimum
- [ ] Focus states visible for keyboard navigation
- [ ] `prefers-reduced-motion` respected
- [ ] Responsive: 375px, 768px, 1024px, 1440px — mobile-first (base → `md:` → `lg:` → `xl:`)
- [ ] Touch targets ≥ 44×44 px on viewports < 768px
- [ ] Data tables reflow to stacked cards on mobile (use `<ResponsiveTable>`)
- [ ] `font-size: 16px` minimum on inputs (prevents iOS zoom)
- [ ] Modals render as full-screen bottom sheets on mobile
- [ ] No content hidden behind fixed navbars
- [ ] No horizontal scroll on mobile
- [ ] Navigation pattern matches breakpoint (bottom tabs / sidebar / persistent sidebar)
