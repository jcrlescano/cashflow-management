# UI Design

Part of the Phase 0 design set: [plan](cashflow-app-plan.md) · [architecture](architecture.md) · [database](database-design.md) · [API](api-design.md) · **UI** · [engineering standards](engineering-standards.md)

The screens themselves are drawn on the **Cashflow App Design** canvas (Phase 0 artboards, light and dark), and the tokens, type, components and usage rules are published as the **Cashflow Ledger** design system: https://claude.ai/artifact/Ph8RQ5op81hiVPE9bEMu1Y. This document defines the visual system, information architecture and front-end architecture the canvas and the code follow.

---

## 1. Design principles

1. **Payday first.** The most frequent task (confirm pay → adjust → transfer → tick) is one tap from anywhere and works on a phone.
2. **Every number explains itself.** Any computed figure opens an "explain" popover with its formula trace from the engine.
3. **Defaults, never locks.** Every rule screen shows the default (e.g., "Philippines — TRAIN law") with *Use as is*, *Clone & edit*, or *Create your own*. Overridden values carry an "Edited" chip and *Reset to default*.
4. **Spreadsheet speed where it matters.** The planner behaves like a grid (keyboard, fill-right, paste), everything else is forms and cards.
5. **Calm, legible money.** Tabular figures, right-aligned, consistent sign and color rules, no decoration that competes with numbers.

---

## 2. Visual system ("Ledger")

### 2.1 Color tokens

| Token | Light | Dark | Use |
|---|---|---|---|
| `--bg` | `#F6F7F9` | `#0E1116` | App background |
| `--surface` | `#FFFFFF` | `#161B22` | Cards, panels |
| `--surface-2` | `#F0F2F5` | `#1E242C` | Table headers, inputs |
| `--border` | `#DDE1E6` | `#2D333B` | Dividers |
| `--ink` | `#111827` | `#E6EDF3` | Primary text |
| `--ink-2` | `#4B5563` | `#9DA7B3` | Secondary text (≥ 4.5:1 on surface) |
| `--primary` | `#0F5F59` | `#3FB8AC` | Primary actions, focus ring, links |
| `--needs` | `#1D4ED8` | `#6C9BFF` | Needs bucket |
| `--savings` | `#0F766E` | `#34C3B5` | Savings bucket |
| `--wants` | `#C2410C` | `#FF8A4C` | Wants bucket |
| `--positive` | `#166534` | `#4ADE80` | Surplus, on track |
| `--warning` | `#92400E` | `#FBBF24` | Short of target, due soon |
| `--negative` | `#B91C1C` | `#F87171` | Over budget, negative unallocated |

Bucket colors differ in lightness as well as hue (blue / teal / orange) so they stay distinguishable for color-blind users; status is never shown by color alone (icon + text too). User-added buckets pick from an accessible palette.

### 2.2 Typography

- **Inter** (Google Fonts), with `font-feature-settings: "tnum" 1, "cv11" 1` on all numbers.
- Scale: 12 (caption) · 14 (body, tables) · 16 (inputs on mobile) · 20 (section) · 24 (page title) · 32 (hero figures).
- Weights: 400 body, 500 labels/table headers, 600 headings and key figures.

### 2.3 Spacing, shape, elevation

- 4-px grid; common steps 8 / 12 / 16 / 24 / 32.
- Radius 8 (inputs, buttons), 12 (cards); 1-px borders instead of heavy shadows; one subtle shadow for popovers/dialogs.
- Touch targets ≥ 44 px; table rows 40 px (desktop), 48 px (mobile).

### 2.4 Money formatting

- Format by workspace locale/currency: `₱41,100.00`; negative `−₱1,500.25` in `--negative`.
- Amounts right-aligned; totals in weight 600 with a top border.
- Overridden planner cells: small dot marker + tooltip "Overrides schedule (₱600.00)".

### 2.5 Iconography

Lucide icons (stroke 1.75) via an icon component; icons always paired with a label or `aria-label`.

---

## 3. Information architecture

```
App shell
├─ Dashboard
├─ Payday                (current period; period picker)
├─ Planner               (year grid; item drawer)
├─ Income & Tax
│  ├─ Salary profiles    (history, editor)
│  ├─ Calculator         (what-if)
│  ├─ Payslips           (expected vs actual)
│  └─ Tax summary        (YTD, annualization)
├─ Goals & Funds
├─ Debts
├─ Accounts              (balances, register, interest)
├─ Net Worth
├─ Reports
└─ Settings
   ├─ Workspace          (currency, locale, time zone, rounding)
   ├─ Plan years         (create, rollover)
   ├─ Pay schedule
   ├─ Buckets & targets
   ├─ Categories
   ├─ Accounts & routes
   ├─ Tax tables         (template / clone / custom, versions, brackets, preview)
   ├─ Contributions      (schemes, methods, versions, preview)
   ├─ Earnings & deductions
   ├─ Members & invites
   ├─ Import / Export
   ├─ My templates       (personal tax tables, contribution sets, category sets — reusable in any workspace)
   └─ Security & recovery (passkeys, recovery key, trusted contact, export)
```

Desktop: left navigation rail (collapsible) + top bar (plan year switcher, workspace switcher, notifications, user). Mobile: bottom tab bar (Dashboard · Payday · Planner · More) and full-screen sheets.

---

## 4. Screen inventory (canvas artboards)

| # | Screen | Key content |
|---|---|---|
| 0 | Foundations | Tokens, type, components, states |
| 1 | Sign in | Google sign-in button, invite acceptance, "no access" state |
| 1b | Unlock & protect your data | First-run: create passkey or vault passphrase, save recovery key (must confirm); daily: unlock with passkey; recover with recovery key; privacy explainer "what whoever runs the server can't see" |
| 2 | Onboarding | Workspace setup: currency, pay schedule, tax table choice, import or start fresh |
| 3 | Dashboard | Next payday, 50/30/20 meters, income vs outflow chart, goals, debts, net worth |
| 4 | Payday | 4-step flow: confirm pay · adjust · transfer tree · pay items · close |
| 5 | Planner | 24-period grid with buckets/categories, overrides, unallocated row, item drawer |
| 6 | Item editor (drawer) | Schedule rules timeline, split mode, active months, funding account, links |
| 7 | Salary profiles & calculator | Profile editor + live payslip with explanation |
| 8 | Payslips & tax summary | Expected vs actual, YTD tax, year-end projection |
| 9 | Tax tables | Template vs own copies, versions, bracket editor, preview |
| 10 | Contributions | Schemes list, method editor (percentage / bracket table / fixed), preview |
| 11 | Earnings & deductions, buckets | Component types with flags; bucket targets editor |
| 12 | Goals & funds | Phase progress, sinking items, ETA |
| 13 | Debts | Balances, payoff timeline |
| 14 | Accounts & interest | Account list, register, ADB interest |
| 15 | Net worth | Assets/liabilities, trend |
| 16 | Reports | Cash flow, categories, variance |
| 17 | Import | Upload → mapping summary → per-period comparison → commit |
| 18 | Settings & members | Workspace prefs, pay schedule, invites |
| 19 | Mobile: Dashboard & Payday | 390 × 844 phone layouts |

---

## 5. Front-end architecture (Blazor WebAssembly)

| Concern | Approach |
|---|---|
| Styling | **Tailwind CSS v4**, compiled by the Tailwind standalone CLI from an MSBuild target in `Cashflow.Web` (no Node project needed; `dotnet watch` runs it in watch mode). Tokens from §2 are defined once in `app.css` with `@theme` (`--color-needs`, `--color-surface`, …) so classes read `bg-surface text-ink border-border`. Dark mode via the `dark:` variant driven by a `data-theme` attribute (system preference by default). Class scanning covers `**/*.razor` and `**/*.razor.cs`. |
| Components | In-house component library in `Cashflow.Web/Components/UI` (Button, Input, MoneyInput, Select, Checkbox, Switch, Dialog, Drawer, Menu, Tabs, Toast, Table, Badge, Card, Tooltip, Popover) — Tailwind classes inside, accessible behavior (focus trap, Esc to close, ARIA roles) implemented once and tested with bUnit. Long class lists are composed with a small `Cx()` helper; no `@apply` except for base element styles. Custom `PlannerGrid` (virtualized rows/columns, frozen first column, keyboard navigation). |
| Charts | Lightweight SVG chart components (bars, stacked bars, line, donut/meter) rendered by Blazor — no JS chart library for the MVP; ApexCharts.Blazor only if richer charts are needed later. |
| Structure | `Pages/<Feature>/…` + `Components/<Feature>/…` + `Shared/` (money display, explain popover, edited chip, empty states) |
| State | Per-page state in components; a small `AppState` (user, workspace, plan year, current period) as a scoped service with change notifications; no global store library |
| Data access | Typed API client over `Cashflow.Contracts`; `ETag`s kept per resource for `If-Match` |
| Forms | `EditForm` + FluentValidation (same validators as the API via the shared Contracts rules where possible) |
| Errors | Problem Details → field errors in forms, toast for others; error boundary per page |
| Loading | Skeletons for first load; optimistic update for ticks/toggles with rollback on failure |
| Offline | PWA service worker caches the shell and last-viewed dashboard/payday read-only; writes require connection |
| Routing | `/`, `/payday/{periodId?}`, `/planner/{year}`, `/income/…`, `/goals`, `/debts`, `/accounts/{id?}`, `/net-worth`, `/reports/{name}`, `/settings/…` |
| i18n | Resource files; English first; all formatting via the workspace `CultureInfo` |

### Reusable components

`MoneyText`, `MoneyInput` (locale-aware, accepts `1,250.5`), `PercentInput`, `BucketMeter`, `ExplainPopover`, `EditedChip` + `ResetToDefault`, `EffectiveDatePicker`, `VersionTimeline`, `RuleMethodPicker`, `TransferTree`, `PeriodPicker`, `StatusBadge`, `EmptyState`, `ConfirmDialog`.

---

## 6. Accessibility

- WCAG 2.2 AA: contrast ≥ 4.5:1 for text, visible focus ring (`--primary`, 2 px), full keyboard support (planner: arrows, Enter, Esc, Tab).
- Real buttons/links/labels; icon buttons have `aria-label`; tables use proper headers; live regions announce totals after edits.
- Respect `prefers-reduced-motion` and `prefers-color-scheme`; text resizes to 200% without loss.
- Charts have a data-table alternative.

---

## 7. Responsive rules

| Width | Layout |
|---|---|
| ≥ 1280 | Rail + content up to 1360 px; planner shows 12–24 period columns |
| 768–1279 | Collapsed rail; planner shows 6 months, horizontal scroll |
| < 768 | Bottom tabs; Payday as vertical steps; planner shows one month (P1/P2) with month swiper; tables become cards |
