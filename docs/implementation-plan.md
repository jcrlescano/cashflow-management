# Cashflow — Phased Implementation Plan

| | |
|---|---|
| **Status** | Draft v1 — 2026-11-21 |
| **Implements** | [cashflow-app-plan.md](cashflow-app-plan.md) §14 roadmap, using [architecture](architecture.md) · [database](database-design.md) · [API](api-design.md) · [UI](ui-design.md) · [privacy](privacy-encryption.md) · [engineering standards](engineering-standards.md) |
| **Screens** | Canvas **Cashflow App Design** — https://claude.ai/artifact/D1BRXpqnxGpYXpLjMZJU8h (24 screens, light + dark) · tokens in **Cashflow Ledger** |
| **Scaffold today** | `Cashflow.slnx` with 6 empty `src` projects + 2 empty test projects, CPM, analyzers, `BannedSymbols.txt`, SDK 10.0.404. No commits yet. |

This is the master plan: phases, what each phase builds, which canvas screens it delivers, and how we know it is done. Each phase gets its own detailed task-by-task plan (TDD steps, exact files and signatures) in `docs/plans/<date>-phase-N-*.md`, written when that phase starts, so each one is based on the code that exists by then.

| Phase | Task-level plan |
|---|---|
| 1 — Foundations | [plans/2026-11-21-phase-1-foundations.md](plans/2026-11-21-phase-1-foundations.md) (20 tasks) — **code complete 2026-11-22** on `feat/phase-1-foundations` (603 tests + 24 E2E, coverage Domain 96.9 / Application 94.4 / Total 94.3). Remaining: the live steps — droplet runbook (Task 19 step 6), tag `v0.1.0`, live sign-in and operator-view check (Task 20 steps 2–3). Rulings and deferred minor findings: [plans/2026-11-21-phase-1-execution-ledger.md](plans/2026-11-21-phase-1-execution-ledger.md) |
| 2–5 | Written when the previous phase exits |

---

## 0. Decisions to close before Phase 1

Reading the docs side by side turned up these conflicts. **All recommendations were accepted on 2026-11-21** and the design docs were updated to match. D7 (open questions) takes the defaults in plan §15; D8 is handled per phase.

| # | Conflict | Where | Decision (accepted) |
|---|---|---|---|
| D1 | Encrypted storage shape: one generic `EncryptedRecord {Kind, Metadata, Payload}` table, or each table keeping plain metadata columns plus `payload/nonce/key_version` | architecture §4.7.1 vs database §10 | **Per-table** (database §10): keeps FKs, unique keys and the schedule exclusion constraint in PostgreSQL. One shared EF base type `EncryptedEntity` holds the three crypto columns. |
| D2 | Workbook import: ClosedXML on the server + `POST /imports/workbook` upload, or Open XML SDK in the browser | plan §9, API §3.7 vs privacy §4 | **In the browser** (Open XML SDK, WASM). Drop the upload endpoint. Staged import lives in browser memory; commit = encrypted record batch. ClosedXML is not used. |
| D3 | Server jobs for interest estimates and net-worth snapshots can't read amounts | architecture §4.8 vs privacy §4 | Server keeps only **date-based** jobs (payday/due reminders, invite cleanup). Interest and snapshots are computed **in the browser on the next visit after the 1st**. |
| D4 | Where client use cases live: new `Cashflow.Client.Application` project, or a folder in `Cashflow.Web` | architecture §4.7.1 | **New project `Cashflow.Client`** (class library, references Domain + Contracts; no Blazor). Lets use cases be unit-tested without bUnit. Server `Cashflow.Application` keeps only sync, membership, keys and invites. |
| D5 | Typed API client: Kiota or hand-written | API §5 | **Hand-written** thin `HttpClient` wrappers over Contracts — the sync API is small (records, keys, changes, invites). ADR 0013. |
| D6 | Requirement priorities `P2/P3/P4` don't match roadmap phase numbers (P2 items ship in Phase 3) | plan §6 vs §14 | This plan maps **P2 → Phase 3, P3 → Phase 4, P4 → Phase 5**. Fix the labels in plan §6. |
| D7 | Open questions #4–#9 and #12 still say "default assumed"; Phase 0 exit requires them answered | plan §15 | Confirm the defaults, especially **#12: lost recovery key = lost data**. |
| D8 | Screens in the design but not drawn on the canvas | ui-design §3, §7 | Draw before their phase starts (table in §6). |
| D9 | Canvas shows Nov P2 payday as **Mon 30 Nov 2026**, but that date is Bonifacio Day (a PH regular holiday), so "business day before, PH calendar" gives **Fri 27 Nov** | canvas *Dashboard*, *Payday*, *Settings* vs plan §15 #4 | **Open.** The engine follows the rule (tested). Either fix the canvas dates or default the holiday calendar to *None* if your employer pays on holidays. |

---

## 1. Phase overview

```
Phase 1  Foundations        → deployed, sign-in, unlock, engine passes the golden values
Phase 2  MVP                → a full month of paydays run in the app instead of Excel
Phase 3  Income, goals, debts
Phase 4  Accounts, interest, net worth
Phase 5  Sharing, rollover, polish
```

Each phase ends with a tagged release deployed to the droplet. Within a phase, work is split into **slices** that each land as one PR (< 400 lines where possible), with CI green.

---

## 2. Phase 1 — Foundations (size M)

**Exit:** the engine reproduces every value in plan Appendix C; the app is live on your domain over HTTPS; you can sign in with Google, create a passkey or passphrase, save a recovery key, unlock, and round-trip an encrypted record.

| Slice | Builds | Docs | Tests that prove it |
|---|---|---|---|
| 1.1 Repo & CI | First commit; ADRs 0001–0013 in `docs/adr/`; GitHub Actions: restore, build, `dotnet format --verify-no-changes`, tests, coverage gate; Dependabot | eng-std §1, §4.4; arch §8 | CI green on an empty build |
| 1.2 Architecture guard | NetArchTest rules: Domain references nothing; Web → Contracts/Domain/Client only; `IgnoreQueryFilters` banned | arch §4.2; eng-std §2.4 | `Cashflow.Architecture.Tests` |
| 1.3 Domain value objects | `Money` (decimal, `Money.Round` half-away-from-zero, optional banker's), `Percentage`, `PeriodRef`, `DateRange`, `Result<T>`/`Error` | plan §7; arch §4.4; DB §1 | Rounding at .005 both modes; banned `Math.Round` is not used |
| 1.4 Engine — contributions | `IContributionMethod`: percentage with floor/ceiling, bracket table, fixed; overrides; placement P1/P2/split | plan §7.1, App. A.2 | SSS ₱1,750 · PhilHealth ₱2,000 · Pag-IBIG ₱200 at gross ₱80,000; floors, caps |
| 1.5 Engine — tax | `ITaxCalculator`: bracket (annual/monthly/period basis), flat, none, manual; `ITaxSplitMethod`: workbook, even, P1, P2, manual; explanation trace | plan §7.2, App. A.1 | Every TRAIN boundary at, ±0.01; ₱10,887.50 / ₱11,875 monthly; P1 ₱4,950, P2 ₱5,937.50 |
| 1.6 Engine — payslip & net pay | `PayrollEngine.Compute` → P1/P2/month payslips with `Explain()` | plan §7.3; arch §4.5 | Net P1 ₱41,100.00, P2 ₱42,062.50, month ₱83,162.50 |
| 1.7 Engine — schedules, allocation, transfers | Pay-period generation (semi-monthly, weekend rule, holidays); schedule resolution (even/P1/P2/custom/spread/once, active months, effective ranges); totals, unallocated, targets/surplus; nested transfer tree; unrouted flag | plan §7.5–7.7 | Nov P2 sample golden: Needs ₱20,759.50, Savings ₱4,500, Wants ₱3,154.55, Unallocated ₱13,648.45; transfers Spending ₱15,764.55 / Bills wallet ₱2,899.50 / Car loan account ₱9,500 / Sinking fund ₱2,460 / Travel fund ₱1,000; CsCheck: Σ buckets + unallocated = income, transfers = Σ routed |
| 1.8 Engine — fund phases | Targets as N × monthly net pay | plan §7.9 | Sinking/Emergency/Medical/Retirement values from App. C |
| 1.9 Local stack & DB | `compose.dev.yml` (Postgres 17); `AppDbContext` with snake_case, `identity` schema (`app_user`, `workspace`, `workspace_member`, `invite`, `workspace_settings`, `user_key`, `workspace_key`, `record_change`, `data_protection_key`), RLS + `SET LOCAL app.workspace_id` interceptor, `cashflow_app` role without BYPASSRLS | DB §1, §2, §4.1, §10 | Testcontainers: RLS blocks a query with no `app.workspace_id`; cross-workspace read returns nothing |
| 1.10 Google sign-in + BFF | `/bff/login`, `/signin-google`, `/bff/logout`, `/bff/user`, `/bff/workspace`; allow-list `Auth__AllowedEmails`; `email_verified` required; keyed by `sub`; cookie `HttpOnly/Secure/SameSite=Strict`, antiforgery, rate limit 10/min on login | API §1–2; arch §7 | Fake auth handler: unknown account → no access; allowed email → workspace created; antiforgery missing → 400 |
| 1.11 Crypto core (browser) | WebCrypto via `[JSImport]`: X25519 key pair, WDK AES-256-GCM, AAD = `workspace_id|table|row id|key version`, HKDF; passkey PRF unlock; passphrase via Argon2id (WASM lib); 24-word recovery key; "keep unlocked 30 days" device key in IndexedDB | privacy §3 | Known-answer vectors; AAD mismatch fails; swapped/truncated ciphertext rejected; unlock with passkey, passphrase, recovery key |
| 1.12 Key & sync API | `GET/PUT /keys/me`, `GET /keys/workspace`, encrypted record CRUD with ETag/If-Match, `GET /sync/changes?since=` | API header note; DB §10 | 412 on stale If-Match; payload size limits; viewer can't write |
| 1.13 Web shell & Ledger UI kit | Tailwind v4 standalone CLI via MSBuild; tokens from ui-design §2.1 in `@theme` (light/dark, `data-theme`); Inter + `tnum`; component library (Button, Input, MoneyInput, Select, Checkbox, Switch, Dialog, Drawer, Menu, Tabs, Toast, Table, Badge, Card, Tooltip, Popover) + `MoneyText`, `StatusBadge`, `EmptyState`, `ExplainPopover`, `EditedChip`; app shell: nav rail (canvas *Navigation component*), top bar with plan-year switcher, mobile bottom tabs | UI §2, §5; canvas *Foundations*, *Nav* | bUnit: focus trap, Esc closes, aria labels; money formats `₱41,100.00` and `−₱1,500.25` |
| 1.14 Sign-in & unlock screens | Canvas **Sign in with Google** (sign in, invite, no access), **Unlock & protect your data** (create passkey / passphrase, save recovery key with confirm, daily unlock, what the server can't see) | canvas; privacy §3 | bUnit + Playwright smoke: sign in → set up keys → unlock |
| 1.15 Seed: Philippine base | Seed migration: TRAIN table (2023+, 6 brackets, ₱90k exemption), SSS/PhilHealth/Pag-IBIG, pay component types, 50/30/20 buckets, starter categories, semi-monthly schedule, PH holidays — plain rows with null owners; `GET /base/*` | DB §6; API §3.2 | Seeded TRAIN table through the engine gives the 1.5 golden values |
| 1.16 Droplet & deploy | **Reworked 2026-11-23 (ADR 0014):** the shared droplet (nginx, host PostgreSQL 18, host .NET runtime), no Docker. `bootstrap-droplet.sh` (idempotent, `--dry-run`): users, sudoers, DB roles/database, systemd unit `cashflow-api`, nginx site + certbot; app-set CSP/nosniff/Referrer-Policy (HSTS in nginx); tag → publish → idempotent migration script → SSH deploy script (verified `pg_dump`, one-transaction migration, `current` symlink swap, restart, `/health/ready`, automatic rollback) → public smoke test | arch §5; DB §8; deploy/README.md | Deploy a tag; restore drill into a scratch container |

---

## 3. Phase 2 — MVP: replace the workbook (size L)

**Exit:** you run a full month of paydays in the app instead of Excel; the 2026 workbook import matches every period except the documented §3 fixes; a payday takes under 2 minutes.

All data is stored as encrypted records (D1); every calculation runs in `Cashflow.Client` in the browser.

| Slice | Requirements | Canvas screens (light + dark) |
|---|---|---|
| 2.1 Encrypted config records | Workspace copies of tax tables, contribution schemes, component types, buckets/allocation rules, pay schedule; copy-on-write from base with `source_template_id/version` | — |
| 2.2 Onboarding | SET-7; currency, pay schedule, tax choice (TRAIN / own / manual), contributions, budget split, import or start fresh | **Onboarding** |
| 2.3 Workspace settings | SET-1, SET-2; name, currency, locale, time zone, rounding, theme; pay schedule with next paydays preview; plan year create; members section read-only (invites in Phase 5) | **Settings — workspace, schedule & members** |
| 2.4 Tax tables | SET-6, PAY-3; template vs copy vs custom; versions by date; bracket editor with continuity check; "Try it" preview | **Settings — tax tables** |
| 2.5 Contributions | SET-6, PAY-2; schemes list, method editor (percentage / bracket table / fixed), versions, preview, Edited chip + Reset to template | **Settings — contributions** |
| 2.6 Earnings, buckets, categories | SET-3, SET-5; component types with flags; allocation rule (Σ = 100); categories drag-reorder between buckets | **Settings — earnings, deductions & buckets** |
| 2.7 Accounts & routes | SET-4; accounts with type, funding flag, parent route (cycle check); funding-routes view | Funding routes part of **Accounts & interest** (register/interest come in Phase 4) |
| 2.8 Salary profiles & calculator | PAY-1, PAY-2, PAY-4, PAY-5, PAY-6; profile history; components; tax mode and split; contribution placement and overrides; live payslip with explanation; bucket amounts; what-if without saving | **Salary profile & calculator** |
| 2.9 Planner grid | PLN-1, PLN-2, PLN-3, PLN-4, PLN-6, PLN-8, PLN-11; virtualized `PlannerGrid`, frozen column, periods/months/quarters, "this period only / from here onward", override dot, schedule-change mark, Ctrl+R fill-right, paste, unallocated row red when < 0, warnings | **Planner** |
| 2.10 Item editor drawer | PLN-2, PLN-5, PLN-7 (accounts), PLN-9; schedule rules timeline, split modes, active months, overrides table with reset, funding account, notes, archive | **Planner — item editor** |
| 2.11 Payday flow | DAY-1…DAY-6; 4 steps: confirm pay, adjust (this period only), transfer tree with copy + tick, tick items, mark all paid, close/reopen; side summary | **Payday**, **Mobile — payday** |
| 2.12 Allocation view & basic dashboard | ALC-1…ALC-4, RPT-1 basic; next payday card, 50/30/20 meters, income vs outflow chart (SVG, with data-table alternative), needs-attention list; goals/debts/net-worth tiles show empty states until Phases 3–4 | **Dashboard**, **Mobile — dashboard** |
| 2.13 Workbook import (browser) | IMP-1, D2; read cell values per Appendix B; schedule inference §11.3; report with per-period comparison, warnings, commit/discard | **Import workbook** |
| 2.14 Export & privacy page | IMP-2 (workspace export: encrypted JSON + Excel decrypted in the browser); privacy page from privacy §2 | — |
| 2.15 Operator-view test | Full payday flow, then dump every tenant table: no amount, name or note in plaintext | privacy §8 |

---

## 4. Phase 3 — Income accuracy, goals, debts (size L)

**Exit:** a year-end tax estimate is shown; goal and debt balances update from paid items.

| Slice | Requirements | Canvas |
|---|---|---|
| 3.1 Actual payslips | PAY-7; log actual, expected vs actual, actual replaces expected for the period | **Payslips & tax summary** |
| 3.2 Extra income & annualization | PAY-8, PAY-9; 13th month/bonus with ₱90k exemption; YTD tax, projected annual tax, December adjustment | **Payslips & tax summary** |
| 3.3 Goals & sinking items | GOL-1…GOL-5; target methods, phases, progress, ETA, sinking items with due months | **Goals & funds** |
| 3.4 Debts | DBT-1…DBT-3; balances, linked payment item lowers balance, payoff projection (§7.10), CC total | **Debts** |
| 3.5 Planner links | PLN-7 (goals/debts) | Item editor fields |
| 3.6 Carry-over & history | DAY-7, PLN-10 (undo, change history, names only in audit) | Payday, item editor |
| 3.7 Reports & Excel export | RPT-2, RPT-3, RPT-4, IMP-3 | **Reports** |

## 5. Phase 4 — Accounts, interest, net worth (size M)

**Exit:** interest estimate within ₱1 of bank-credited interest.

| Slice | Requirements | Canvas |
|---|---|---|
| 4.1 Transactions & register | ACC-1, ACC-2 (CSV import parsed in the browser), auto-transactions from transfers and paid items | **Accounts & interest** |
| 4.2 Reconcile | ACC-3 | **Accounts & interest** |
| 4.3 ADB interest | ACC-4, ACC-5; rate history, mid-month rate change, 360/365 basis, 20% final tax, monthly goal (computed in the browser, D3) | **Accounts & interest** |
| 4.4 Net worth | NW-1…NW-3; assets, liabilities from debts, monthly snapshot made by the browser on next visit, trend chart | **Net worth** |
| 4.5 Full dashboard | RPT-1 full | **Dashboard** (goals, debts, net worth tiles live) |

## 6. Phase 5 — Sharing, rollover, polish (size M)

**Exit:** the 2027 plan is created in minutes; a second user runs their own workspace.

| Slice | Requirements | Canvas |
|---|---|---|
| 5.1 Invites & roles | Invite links (7 days, single use, hash stored), Owner/Editor/Viewer, WDK wrapped for invitee, key rotation on removal | **Settings** members, **Sign in** invite card |
| 5.2 Recovery ladder | Trusted contact, shared-workspace re-wrap, restore from export, start over (30-day locked archive), 90-day reminder | **Lost keys — recovery & start over** |
| 5.3 Year rollover | IMP-4 | **Settings** plan years |
| 5.4 Reminders | DAY-8; in-app inbox + Web Push from plain dates only | *not drawn* |
| 5.5 Templates & base updates | Personal template library; "new PH base rates available" review → new effective-dated version | **My templates** |
| 5.6 PWA offline read, signed builds | Service worker read-only cache; cosign + SBOM; public deploy log | — |

### Screens the canvas doesn't have yet (D8)

Draw each one before its phase starts: **Log actual payslip** dialog (3.1) · **Add/edit account** (2.7) · **Goal editor** and **Debt editor** (3.3, 3.4) · **Base update review** diff (5.5) · **Notifications inbox** (5.4) · **Workspace switcher** (5.1) · **Mobile planner** month swiper and **More** menu (2.9, 1.13) · **Tablet** collapsed-rail layout (1.13) · empty, loading (skeleton) and error states for each screen.

---

## 7. Working rules for every phase

- Every screen is built from the canvas artboard and checked against its dark twin; no raw hex in components, only Ledger tokens.
- Every slice: failing test first, then code; analyzers clean; one migration per PR with its SQL reviewed.
- Golden tests (Appendix C) run on every build, and from Phase 2 the import fixture comparison does too. Use an anonymized fixture workbook, never the real one.
- No amounts, names, notes or emails in logs.
- Before starting a phase: write `docs/plans/phase-N-*.md` (task-level TDD plan), check the phase's canvas screens exist, and close any open decision it depends on.
