# Product Requirements Document — Koinomy (SGFP)

**Sync rule.** This document is the source of truth for Koinomy product scope. It tracks the functional contract being re-ported from v1's PRD and v1's 24 canonical OpenSpec specs under the SDD workflow (`AGENTS.md` §4). Any PR that adds, removes, or changes an agreed product requirement MUST update this document in the same PR. Silent drift is a defect.

## 1. Introduction and Overview

- **Product identity.** Koinomy is a Personal Financial Management System (SGFP): a **private, self-hosted, multi-tenant web application**. Access is **invite-only** — an administrator issues invitations; there is **no public registration**.
- **MVP objective.** Strict budget control, multi-currency transaction tracking, exchange operations, and debt and savings management.
- **Data entry model.** Manual entry by the user; automated posting assistance for recurring obligations and debt installments; exchange rates provided automatically by the backend (daily ingestion from a public API plus manual refresh).
- **Prioritization.** MoSCoW. All 11 modules carried over from v1 are **Must-have**; the prioritization discipline is preserved even where every current module is a Must.
- **Future vision.** Scalability to a mobile application (React Native / Flutter). The Koinomy architecture keeps a standalone REST API so this stays possible (ADR-0002).
- **Provenance.** Koinomy is a clean-slate restart (ADR-0001). The functional scope below is a faithful re-port of v1's PRD; the acceptance criteria are mined from v1's OpenSpec specs, which remain the detailed behavior contract. v1 code is reference material only — behavior is re-ported spec by spec, never copied wholesale.

### What changed from the v1 PRD

| Area | v1 claim | Koinomy truth |
|------|----------|----------|
| Module 11 (2FA, invitations, admin, HIBP, session timeout) | Listed as Must-have; never implemented; dead schema columns | MUST requirements with real acceptance criteria; columns ship only with their implementation (ADR-0004) |
| Code coverage | "100% mandatory" — enforced nothing | Enforceable thresholds in CI: 90% financial/security, 80% overall, ratchet up only (ADR-0007) |
| Savings goal currency | PRD text said goals carry their own `currencyCode` | Final v1 schema/spec dropped it: a goal's currency is always its host account's currency (REQ-SG-18) |
| Deployment targets | Vercel / Render / Neon named | Self-hosted; deployment target is an operations concern (`docs/OPERATIONS.md`) |

## 2. Architecture and Tech Stack

Decoupled client/server architecture, locked by ADR. Details in `docs/ARCHITECTURE.md`.

| Layer | Technology | ADR |
|-------|-----------|-----|
| Frontend | React 19 + Vite, TypeScript strict, Tailwind 4, shadcn/ui + Radix primitives, TanStack Query, react-hook-form + zod, react-router 7 | ADR-0002 |
| Backend | NestJS 11, TypeScript strict, zod as the single validation language | ADR-0002 |
| Data layer | Drizzle ORM over PostgreSQL 16+; SQL-file migrations committed to git | ADR-0003 |
| Authentication | better-auth with DB-backed cookie sessions; plugins for 2FA and admin | ADR-0004 |
| Field-level encryption | AES-256-GCM behind an `EncryptionService` adapter | ADR-0005 |
| Tenant isolation | `userId` filtering in every user-domain query + PostgreSQL Row-Level Security | ADR-0006 |
| Repository shape | npm workspaces monorepo: `api/` and `client/` | ADR-0002 |

## 3. Module Map

Koinomy keeps v1's module numbering. Module 3 covers both the daily journal/transfers and the recurring-payments engine (v1 grouped them together); each gets its own requirements and acceptance-criteria block below.

| Koinomy module | v1 PRD module | v1 OpenSpec specs (behavior contract) |
|-----------|---------------|----------------------------------------|
| 1. Accounts & Financial Entities | 1. Account and Financial Entity Management | `accounts`, `entity-banks`, `accounts-frontend` |
| 2. Categories & Subcategories | 2. Category and Subcategory Management | `categories` |
| 3. Transactions, Transfers & Recurring Payments | 3. Daily Journal, Recurring Obligations, and Exchange Operations | `transactions`, `transactions-frontend`, `recurring-payment`, `recurring-payments-frontend` |
| 4. Currencies & Exchange Rates | 4. Exchange Rate Historical Monitor | `currency-catalog`, `exchange-rates`, `exchange-rates-frontend` |
| 5. Debt & Installments (Debt Consumption) | 5. Credit Consumption and Debt Management | `debt-consumption`, `debt-consumption-frontend` |
| 6. Budgets & Planning | 6. Budget Planning (Cash Flow) | `budgets` |
| 7. Snapshots (Net Worth) | 7. Financial Snapshots and Closings | `snapshots` (backend + frontend requirements) |
| 8. Savings Goals | 8. Dynamic Savings Goals | `savings-goals`, `savings-goal-contributions`, `savings-goals-frontend` |
| 9. Operational Dashboard | 9. Operational Dashboard | `dashboard` |
| 10. Notifications & Alerts | 10. Notifications and Alerts | `notifications`, `notifications-frontend` |
| 11. Security & Administration | 11. Advanced Security and Privacy | **No v1 spec exists** — v1 never implemented Module 11. Koinomy acceptance criteria below are new, derived from ADR-0004 and the v1 PRD text |

Cross-cutting v1 specs with no module home: `auth-guard` (the v1 dev-mode `X-User-Id` guard — **explicitly NOT ported**; superseded by ADR-0004/0006), `encryption` (ported into NFR §5.1/§5.2 with the ADR-0005 fail-boot correction), `monorepo-infrastructure` (ported into `docs/ARCHITECTURE.md` and ADR-0008).

> **Acceptance-criteria convention.** The Given/When/Then criteria below summarize the contract at requirement level. The detailed scenario-level specs live in the v1 OpenSpec collection (`C:\Users\cabrerae\Desarrollo\ArcaSavings\openspec\specs\`) and are re-created as delta specs during each Koinomy SDD change (`AGENTS.md` §4). Where Koinomy intentionally deviates from a v1 scenario, the deviation is stated inline.

## 4. Functional Requirements (MoSCoW)

### Module 1: Accounts & Financial Entities

**Description.** Management of banks/financial entities (`EntityBank`) and user-owned accounts: Checking, Savings, Credit Card, Loan, and independent Cash accounts. Each account has a native currency from the ISO 4217 global catalog, an encrypted balance, and — for credit/loan accounts — an encrypted maximum approved limit used for overdraft and available-credit control.

**Priority:** Must-have.

**Functional requirements.**

- FR-1.1 Register, list, rename, and delete banks/financial entities; upload, replace, and remove an entity logo (PNG/JPEG/WebP, max 2 MB; SVG rejected).
- FR-1.2 Create accounts of type CHECKING, SAVINGS, CREDIT, LOAN, CASH with an opening balance and a native currency that must be an ACTIVE catalog currency.
- FR-1.3 `creditLimit` is mandatory for CREDIT and LOAN accounts and rejected for all other types; it drives overdraft alerts and available-credit computation.
- FR-1.4 List, fetch, and update own accounts; update covers name and balance. Currency, type, and credit limit are immutable post-create.
- FR-1.5 Account deletion is blocked while snapshots reference the account or an active DEDICATED savings goal is linked to it; VIRTUAL goals cascade with the account.
- FR-1.6 Available credit for CREDIT/LOAN accounts: `creditLimit − SUM(active debt consumptions)`.
- FR-1.7 Real available balance per account: `balance − SUM(VIRTUAL goal reservations)`.
- FR-1.8 Amounts and names are decimal strings (6 decimal places) encrypted at rest; responses return decrypted strings, never JavaScript numbers.

**Acceptance criteria** (mined from `accounts`, `entity-banks`).

- GIVEN an authenticated user WHEN they create an account with a valid payload THEN the account is stored with the caller's `userId`, `name` and `openingBalance` encrypted at rest, and the response returns decrypted decimal strings.
- GIVEN a CREDIT or LOAN create request without `creditLimit` WHEN processed THEN HTTP 400; GIVEN a CHECKING/SAVINGS/CASH request WITH `creditLimit` WHEN processed THEN HTTP 400 (also enforced at the service layer, defense in depth).
- GIVEN a `currencyCode` that is inactive or absent from the catalog WHEN an account is created THEN HTTP 400, including when the service is called directly bypassing the controller.
- GIVEN user B WHEN they fetch or update user A's account THEN HTTP 404 (never 403 — anti-enumeration).
- GIVEN an account referenced by snapshot rows or a DEDICATED goal WHEN the owner deletes it THEN HTTP 400 naming both reference counts; GIVEN only VIRTUAL goals WHEN deleted THEN deletion proceeds and VIRTUAL goals cascade.
- GIVEN a CREDIT account with limit "5000.00" and active consumptions totaling "1500.00" WHEN available credit is requested THEN `{ availableCredit: "3500.00", ... }`; inactive consumptions are excluded.
- GIVEN an account with balance "5000.00" and VIRTUAL reservations "500.00" + "300.00" WHEN available balance is requested THEN `realAvailableBalance: "4200.00"` with the reservation breakdown; DEDICATED goals do not reduce it.
- GIVEN a logo upload WHEN the file is SVG or larger than 2 MB THEN HTTP 415 / 413 and nothing is written; logo URLs are tenant-prefixed so cross-tenant paths resolve to 404.

### Module 2: Categories & Subcategories

**Description.** User-owned financial taxonomy: Main Categories and Subcategories with a hard depth cap of 2. Category names are plaintext by design (they render in dropdowns and carry no sensitive financial data).

**Priority:** Must-have.

**Functional requirements.**

- FR-2.1 Create, list, fetch, rename, and reparent categories; `userId` always comes from the authenticated session, never from the body.
- FR-2.2 Depth cap of 2: a subcategory cannot have children (HTTP 400), enforced at the service layer as defense in depth.
- FR-2.3 Name uniqueness: unique per `(userId, name)` for Mains and per `(userId, parentId, name)` for Subs (DB partial unique indexes); duplicates return HTTP 409, including under parallel race conditions.
- FR-2.4 Reparenting rules: a Main cannot gain a parent (400); a Sub may move between the user's own Mains; a Sub may be promoted to Main if no name collision exists.
- FR-2.5 DELETE is not part of the v1 contract (deferred follow-up); Koinomy re-ports the v1 surface unless an SDD change amends it.

**Acceptance criteria** (mined from `categories`).

- GIVEN a user WHEN they POST `{ name }` THEN a Main is created with `depth: 0`; empty/whitespace names and names over 100 characters return HTTP 400.
- GIVEN an existing Main "Food" WHEN the same user creates another Main "Food" THEN HTTP 409; GIVEN parallel duplicate creates THEN one wins with 201 and the other gets 409 (DB constraint mapped to 409).
- GIVEN a `parentId` that does not exist or belongs to another user WHEN creating a Sub THEN HTTP 404 (anti-enumeration); GIVEN a depth-1 parent WHEN creating a Sub THEN HTTP 400.
- GIVEN a Sub under Main A WHEN the owner reparents it to their own Main B THEN 200; GIVEN another user's Main THEN 404.
- GIVEN any cross-tenant GET/PATCH/list-filter on categories THEN HTTP 404 (never 403).

### Module 3: Transactions, Transfers & Recurring Payments

**Description.** The financial engine: an append-only double-entry daily journal for income, expenses, transfers, and debt payments (3.1), plus the recurring-obligation manager that reminds users of due dates and posts payments on demand via "Pay Now" (3.2). Multi-currency operations freeze the applied exchange rate on each row.

**Priority:** Must-have.

#### 3.1 Transactions & Transfers (Multi-Currency)

**Functional requirements.**

- FR-3.1 Record INCOME and EXPENSE transactions; creation atomically inserts the row and updates the account balance (decrypt → signed delta → re-encrypt) in one database transaction.
- FR-3.2 Transfers between own accounts create two linked TRANSFER rows (`linkedTransactionId`); cross-currency transfers freeze `exchangeRateApplied` per row; same-currency transfers use `1.000000`.
- FR-3.3 A transaction's `currencyCode` must equal its account's currency (service-layer defense in depth); amounts match `^\d+\.\d{1,6}$`; rates are `> 0`.
- FR-3.4 List with filters (`accountId`, `categoryId`, `type`, `from`, `to`) and pagination (default 20, max 100), ordered `date DESC`; fetch by id.
- FR-3.5 Limited mutability: only `note`, `categoryId`, and `date` are editable; amount, account, type, and currency are immutable. The journal is append-only — no DELETE.
- FR-3.6 Programmatic DEBT_PAYMENT creation participates in an outer transaction when one is supplied (used by Module 5 installment payment).
- FR-3.7 Aggregation support: sum transaction amounts by category and period with exact decimal arithmetic (feeds Module 6).

**Acceptance criteria** (mined from `transactions`).

- GIVEN an account with balance 100 WHEN the user posts an INCOME of 50 THEN the transaction insert and the balance update to 150 commit atomically (both or neither); EXPENSE decrements analogously.
- GIVEN a cross-currency transfer of 100 USD to a EUR account at rate 0.95 WHEN processed THEN the source row stores `1.000000`, the destination row stores `0.95`, both balances update atomically, and the rows are linked.
- GIVEN a direct DB read of `encryptedAmount`/`encryptedNote` WHEN inspected THEN only ciphertext is present (encryption-boundary assertion).
- GIVEN a PATCH with `amount`, `accountId`, `type`, or `currencyCode` WHEN processed THEN HTTP 400; GIVEN a date change WHEN processed THEN the account balance is unchanged.
- GIVEN user B WHEN they fetch user A's transaction THEN HTTP 404 with a single indistinguishable message.
- GIVEN any transaction response WHEN serialized THEN it never contains `userId` or `encrypted*` fields; monetary values are decimal strings.
- GIVEN a DELETE request on a transaction THEN HTTP 404 (no delete route — append-only journal).

#### 3.2 Recurring Payments

**Functional requirements.**

- FR-3.8 Manage rules (templates) for outgoing-only operations: `EXPENSE` or `DEBT_PAYMENT`; frequencies MONTHLY, WEEKLY, EVERY_N_DAYS, DAILY with discriminator-validated sub-fields (`dayOfMonth` 1–31, `dayOfWeek` 0–6, `intervalDays` ≥ 1).
- FR-3.9 Rules remind, they do not auto-bill: a daily cron at 09:00 UTC notifies rules due within 7 days (once per day per rule); transactions are created only by the manual Pay Now action.
- FR-3.10 Pay Now creates a real transaction carrying `generatedFromRecurringPaymentId` (FK with SET NULL), validates account ownership and currency match, rejects future dates, then advances `nextDueDate` and resets notification counters.
- FR-3.11 Update is future-only: past transactions stay immutable; schedule changes recompute `nextDueDate` and reset notification state. Cancel is a soft delete (`cancelledAt`) that excludes the rule from the cron while preserving history.
- FR-3.12 End-of-month rule: a MONTHLY `dayOfMonth` beyond the target month's length lands on the month's last day.
- FR-3.13 Aggregation support: sum active rule template amounts by category and period with period-boundary awareness (feeds Module 6 `fixedExpected`).

**Acceptance criteria** (mined from `recurring-payment`).

- GIVEN a create request with `type: INCOME` or `TRANSFER` WHEN processed THEN HTTP 400; EXPENSE and DEBT_PAYMENT are accepted with the amount stored as ciphertext.
- GIVEN MONTHLY without `dayOfMonth` (or WEEKLY without `dayOfWeek`, EVERY_N_DAYS without `intervalDays`) WHEN processed THEN HTTP 400; CRON and YEARLY frequencies are rejected.
- GIVEN a MONTHLY rule with `dayOfMonth: 31` WHEN the next due date lands in April THEN it is April 30; in a non-leap February, February 28.
- GIVEN 3 active rules due within 7 days WHEN the 09:00 UTC cron runs THEN each gets one reminder email, `lastNotifiedAt` is set, and `notificationCount` increments; cancelled, inactive, already-notified-today, and >7-day rules are skipped (this is the documented cross-tenant scan exception, `AGENTS.md` §5.6).
- GIVEN an active rule WHEN Pay Now is called with the owner's account, matching currency, valid amount, and non-future date THEN a transaction is created with the rule FK and `nextDueDate` advances; GIVEN a cancelled rule THEN HTTP 410; GIVEN another user's account id THEN HTTP 404.
- GIVEN a rule with template amount "100.00" WHEN Pay Now sends amount "120.00" THEN the transaction stores "120.00" (template is informational).

### Module 4: Currencies & Exchange Rates

**Description.** Global, shared catalogs (no `userId`): the ISO 4217 currency catalog and the historical exchange-rate series, anchored to USD as the fixed base currency (a rate expresses how much of the target currency equals 1 USD). Rates are ingested daily from `open.er-api.com`, refreshable manually, with failure alerting.

**Priority:** Must-have.

**Functional requirements.**

- FR-4.1 Read-only currency catalog: list active currencies (`code`, `name`, `symbol`); fetch one by ISO code; unknown codes return 404.
- FR-4.2 Latest rate per currency and historical series with optional date-range filter, ordered date descending; a currency with no history returns 404.
- FR-4.3 Live ingestion: fetch `https://open.er-api.com/v6/latest/USD` with a 5-second timeout; persist one row per upstream code that exists in the catalog with `officialRate = parallelRate = rate` (6 decimals), `source = "open.er-api"`, date floored to UTC midnight; unknown codes are skipped with a warning; same-day re-runs upsert in place (idempotent).
- FR-4.4 Daily cron at 08:00 UTC runs the ingestion (system-level, no `userId` — documented exception).
- FR-4.5 Manual refresh: any authenticated user can trigger a sync; upstream non-2xx/timeout/malformed JSON map to HTTP 503.
- FR-4.6 Failure alerting: an in-memory consecutive-failure counter logs each failure; at the configured threshold (default 3, env-overridable) one email goes to the first ADMIN user; the counter resets on success.
- FR-4.7 Contingency custom rates: users may apply a custom rate when recording a transfer (Module 3, rate selection "Official" / "Parallel" / "Custom"); the frozen rate on the transaction row is the audit record.
- FR-4.8 The API response DTO omits `source` (DB column preserved for audit).

**Acceptance criteria** (mined from `currency-catalog`, `exchange-rates`).

- GIVEN the catalog contains active and inactive currencies WHEN the list is requested THEN only active currencies are returned.
- GIVEN upstream returns `"EUR": 0.92` and EUR exists in the catalog WHEN ingestion runs THEN one row exists for EUR that UTC day with both rates `0.920000`; GIVEN a second run the same day with `0.93` THEN the row is updated in place, no duplicate.
- GIVEN upstream includes a code absent from the catalog WHEN ingestion runs THEN no row is created and a warning is logged.
- GIVEN the upstream is down or slow (>5 s) WHEN a manual sync is triggered THEN HTTP 503.
- GIVEN 2 consecutive failures and threshold 3 WHEN the cron fails again THEN one email is sent to the first ADMIN user; GIVEN no ADMIN exists THEN the email is skipped with a warning; GIVEN the next run succeeds THEN the counter resets to 0.

### Module 5: Debt & Installments (Debt Consumption)

**Description.** Granular liability tracking on CREDIT/LOAN accounts: each consumption (purchase) generates a prorated installment schedule; paying an installment creates a DEBT_PAYMENT transaction, reduces the liability, and is recorded as an expense. Consumptions are append-only.

**Priority:** Must-have.

**Functional requirements.**

- FR-5.1 Create a consumption on a CREDIT/LOAN account owned by the caller: total amount, 1–60 installments, mandatory description (1–100 chars, encrypted), optional category.
- FR-5.2 Installment schedule generation is atomic: N rows, `totalAmount / installments` each, last row absorbs the rounding remainder so the sum is exact; first due date is next month's same day (month-end clamped).
- FR-5.3 List consumptions (active + inactive, newest first); fetch one with its full installment list.
- FR-5.4 Pay an installment: creates the DEBT_PAYMENT transaction in the same database transaction, marks the installment PAID, decrements `remainingInstallments`, and deactivates the consumption when the last installment is paid. The category is read from the consumption, never from the pay request.
- FR-5.5 Double payment returns HTTP 409. Available credit (Module 1) reflects active consumptions.

**Acceptance criteria** (mined from `debt-consumption`).

- GIVEN totalAmount "1000.00" with 3 installments WHEN the consumption is created THEN installment amounts are [333.333333, 333.333333, 333.333334] summing exactly to the total, created atomically with the parent row.
- GIVEN a CHECKING or SAVINGS account WHEN a consumption is created against it THEN HTTP 400; GIVEN a description over 100 characters THEN HTTP 400.
- GIVEN a PENDING installment WHEN pay is called with a valid transaction account THEN the DEBT_PAYMENT transaction exists with the consumption's category, the installment is PAID with `paidTransactionId` set, and `remainingInstallments` decrements — all atomically.
- GIVEN `remainingInstallments = 1` WHEN the last installment is paid THEN `remainingInstallments = 0` and `isActive = false`.
- GIVEN an already-PAID installment WHEN pay is called again THEN HTTP 409.
- GIVEN user B WHEN they fetch user A's consumption THEN HTTP 404; every query filters by the direct `userId` column.

### Module 6: Budgets & Planning

**Description.** Strictly monthly budget cycles. One plan line per (category × month) with upsert semantics; actuals are computed on read by aggregating transactions, and expected fixed spending is aggregated from active recurring rules.

**Priority:** Must-have.

**Functional requirements.**

- FR-6.1 Upsert a planned amount per (category, year, month): amount matches `^\d+\.\d{1,6}$`, month 1–12, year 2000–2100, category owned by the caller; the composite unique constraint enforces one line per category per month.
- FR-6.2 List plan lines for a period (ordered by category name); delete a line (hard delete).
- FR-6.3 Period summary: for ALL of the user's categories — planned or not — return `planned` (null when un-planned), `actual` (decrypted transaction sum for the period, DEBT_PAYMENT included), `fixedExpected` (active recurring template sum with period-boundary awareness), plus totals and variance.
- FR-6.4 Automated feeding comes from the aggregation of recurring rules and transactions at read time; manual assignment covers variable expenses, savings, and investments.

**Acceptance criteria** (mined from `budgets`).

- GIVEN an existing line "200.00" for (category-A, 2026, 6) WHEN the user POSTs "250.00" for the same key THEN the line is replaced (upsert), not duplicated.
- GIVEN transactions totaling "150.00" in a budgeted category for the period WHEN the summary is requested THEN the line shows `planned: "200.00"`, `actual: "150.00"`; GIVEN transactions but no plan line THEN `planned: null` with the actual still shown.
- GIVEN EXPENSE "100.00" and DEBT_PAYMENT "75.00" in the same category/period WHEN the summary is computed THEN actual = "175.00".
- GIVEN another user's `categoryId` WHEN posted THEN HTTP 404; GIVEN cross-tenant access to any budget resource THEN HTTP 404.
- GIVEN any budget response WHEN serialized THEN no `userId` field and all monetary values are decimal strings.

### Module 7: Snapshots (Net Worth)

**Description.** User-initiated financial closings. A snapshot captures every account balance at a point in time, stores an encrypted net-worth aggregate, and supports comparative analysis against the previous closing.

**Priority:** Must-have.

**Functional requirements.**

- FR-7.1 Create a snapshot with a free-form period name (1–100 chars) and a non-future date: captures all account balances and the net-worth sum atomically (snapshot row + N balance rows in one transaction). A user with zero accounts gets HTTP 404.
- FR-7.2 List snapshots with date-range filters and pagination; items expose decrypted net worth without per-account breakdown.
- FR-7.3 Detail view includes the per-account captured balances and a comparative object versus the most recent earlier snapshot: signed delta and direction UP/DOWN/FLAT, or null for the first snapshot.
- FR-7.4 Defense in depth: the detail read re-sums the captured balances and, on mismatch with the stored aggregate, logs a warning and returns the re-computed value (no write-back).
- FR-7.5 A closing-report email is sent after successful creation; email failure never rolls back the snapshot.

**Acceptance criteria** (mined from `snapshots`).

- GIVEN accounts with balances "1000.00" and "500.00" WHEN a snapshot is created THEN one Snapshot plus 2 balance rows exist atomically, net worth "1500.00" encrypted at rest, and the response returns decrypted values.
- GIVEN a future date, an empty name, or a name over 100 chars WHEN creating THEN HTTP 400.
- GIVEN snapshots A (net worth "1000.00") then B ("1500.00") WHEN B is fetched THEN `comparative.delta = "500.00"`, `direction = "UP"`; GIVEN B at "800.00" THEN "DOWN"; GIVEN equal values THEN "FLAT"; GIVEN only one snapshot THEN `comparative: null`.
- GIVEN a corrupted stored aggregate WHEN the detail is read THEN the re-summed value is returned and a warning is logged.
- GIVEN snapshot creation succeeds WHEN the closing-report email fails THEN the client still receives HTTP 201 and the failure is logged.

### Module 8: Savings Goals

**Description.** Savings projections with two operating modes. **DEDICATED** links a goal 1:1 to a real account (progress = account balance / target). **VIRTUAL** tracks progress independently via manual deposit/withdrawal contributions without moving real money (progress = SUM(contributions) / target). A goal's currency is always its host account's currency. At most one DEDICATED goal per host account (partial unique index); VIRTUAL goals never block account deletion.

**Priority:** Must-have.

**Functional requirements.**

- FR-8.1 Create goals on CHECKING/SAVINGS/CASH accounts (CREDIT/LOAN rejected, service-layer defense in depth) with name, target amount, expected contribution per period, contribution frequency (WEEKLY/BIWEEKLY/MONTHLY), and a required `startDate` ≤ today.
- FR-8.2 DEDICATED uniqueness per host account returns HTTP 409 on duplicates; multiple VIRTUAL goals per account are allowed.
- FR-8.3 Live progress on read: `currentAmount`, `progressPercent` (clamped 0–100), `remainingAmount`, `isMet`, `isOnTrack`, period-aware `projectedFulfillmentDate`, and `nextContributionDate` computed from `startDate` + frequency (month-end aware).
- FR-8.4 VIRTUAL contributions: DEPOSIT without category, WITHDRAWAL with mandatory category; contributions inherit the host account's currency; deposit/withdrawal history is paginated; deleting a contribution changes the SUM-based progress.
- FR-8.5 Update covers name, target, frequency, expected contribution, startDate, isActive; `hostAccountId` is immutable. Delete is hard and cascades contributions.
- FR-8.6 Real available balance on accounts accounts for VIRTUAL reservations (Module 1, FR-1.7).

**Acceptance criteria** (mined from `savings-goals`, `savings-goal-contributions`).

- GIVEN a DEDICATED goal (target "1000.00", account balance "425.00") WHEN the list is read THEN `currentAmount: "425.000000"` and `progressPercent: 42.50`; GIVEN a VIRTUAL goal with deposits "600.00" and withdrawals "100.00" THEN `currentAmount: "500.000000"`.
- GIVEN a missing or future `startDate` WHEN creating THEN HTTP 400; GIVEN a CREDIT or LOAN host account THEN HTTP 400 including via direct service call.
- GIVEN an existing DEDICATED goal on an account WHEN a second DEDICATED goal is created THEN HTTP 409; GIVEN a VIRTUAL goal on the same account THEN HTTP 201.
- GIVEN target "1000.00", current "400.00", expected "50.00" WEEKLY WHEN the projection is computed THEN `periodsRemaining = 12` and the projected date anchors on `nextContributionDate + 77 days`; GIVEN expected contribution ≤ 0 or > 600 periods THEN null; GIVEN an already-met goal THEN today.
- GIVEN a WITHDRAWAL contribution without `categoryId` WHEN created THEN HTTP 400; GIVEN any contribution on a DEDICATED goal THEN HTTP 400.
- GIVEN a VIRTUAL goal with contributions WHEN the goal is deleted THEN all contribution rows cascade-delete; GIVEN a goal on an account WHEN the account is deleted THEN only DEDICATED goals block it.
- GIVEN any goal or contribution response WHEN serialized THEN no `userId`, no `encrypted*` fields, no goal-level `currencyCode` (currency appears on the expanded account object).

### Module 9: Operational Dashboard

**Description.** The authenticated landing page: a widget grid composing existing endpoints into a single operational overview. All data is tenant-isolated; widgets load in parallel.

**Priority:** Must-have.

**Functional requirements.**

- FR-9.1 Widgets: latest net worth with comparative delta and direction; income vs expenses with per-currency breakdown for the selected period; budget progress per category for the current month; last 10 movements; next 5 upcoming recurring payments by due date; overdraft alerts (negative balance or balance over credit limit).
- FR-9.2 Time filter bar: fixed periods (This Month, Last 30 Days, Last Quarter, This Year) plus a custom range; changing it refreshes the period-sensitive widgets.
- FR-9.3 The dashboard is composed client-side from existing module endpoints — no new backend aggregation endpoints.
- FR-9.4 Resilience: parallel loading with skeleton states; a failed widget shows an error state without blocking the others; every widget has an empty state.
- FR-9.5 The route is guarded: unauthenticated users are redirected to login; after login the user lands on the dashboard.

**Acceptance criteria** (mined from `dashboard`).

- GIVEN an authenticated user WHEN the dashboard loads THEN each widget fetches from its assigned existing endpoint in parallel, showing skeletons until resolved.
- GIVEN the transactions endpoint fails WHEN the dashboard loads THEN the other widgets render normally and only the failed widget shows an error state.
- GIVEN a user with no transactions WHEN the dashboard renders THEN the movements widget shows its empty-state message.
- GIVEN an unauthenticated visitor WHEN they navigate to the dashboard THEN they are redirected to login.
- GIVEN a CREDIT account whose balance exceeds its credit limit WHEN the dashboard renders THEN the overdraft widget lists it.

### Module 10: Notifications & Alerts

**Description.** Proactive communication via transactional email (SMTP), with every send audit-logged. Plain-text delivery; graceful dry-run when SMTP is unconfigured.

**Priority:** Must-have.

**Functional requirements.**

- FR-10.1 Email delivery through a configured SMTP transport; missing configuration puts the system in dry-run mode (log only) — it never crashes on missing SMTP config.
- FR-10.2 Every send creates a `NotificationLog` row; SMTP failure creates a FAILED row with the error and never throws to the caller; batch sends continue past individual failures.
- FR-10.3 Preventive reminders: upcoming recurring payments (Module 3), upcoming installments, deposit reminders. Delinquency emails for past-due unpaid installments. Closing report on snapshot creation (Module 7). Exchange-rate sync failure alert to ADMIN (Module 4).
- FR-10.4 Read-only notification history for the user: paginated list with type/status/date filters; recipient addresses are masked; body and error details are never exposed; any client-supplied `userId` parameter is ignored.

**Acceptance criteria** (mined from `notifications`).

- GIVEN full SMTP configuration WHEN a notification is sent THEN the email goes out and a SENT row is logged; GIVEN a transport error THEN a FAILED row with `errorMessage` is logged and no exception propagates.
- GIVEN missing SMTP configuration WHEN the module initializes THEN a warning is logged and sends are dry-run only.
- GIVEN 30 log entries WHEN the user lists notifications with defaults THEN `{ items: [20], page: 1, limit: 20, total: 30 }`; GIVEN combined type + status + date filters THEN only matching rows return.
- GIVEN recipient "user@example.com" WHEN listed THEN it renders masked as "u***r@example.com"; `body` and `errorMessage` are absent from every list item.
- GIVEN a request with `?userId=<other-tenant>` WHEN listing THEN the parameter is ignored and only the session user's rows return.

### Module 11: Security & Administration

**Description.** Integral protection and attack mitigation. In v1 this module was written into the PRD and never implemented — the schema carried dead columns for it (ADR-0001). In Koinomy these are MUST requirements with real acceptance criteria; the schema columns ship together with their working implementation, never before (ADR-0004). The hardening baseline (headers, CORS, rate limiting, env validation) is tracked in `docs/SECURITY.md` and `AGENTS.md` §2.1.

**Priority:** Must-have.

**Functional requirements.**

- FR-11.1 Application-level encryption (AES-256-GCM) of sensitive financial data via the `EncryptionService` adapter pattern; local provider now, KMS/Vault provider later; unimplemented adapters fail boot (ADR-0005; NFR §5.2).
- FR-11.2 Authentication with email/password through better-auth, DB-backed cookie sessions (HttpOnly, Secure, SameSite=Strict), and a 15-minute idle session timeout enforced server-side.
- FR-11.3 Mandatory two-factor authentication (2FA) for all users, via the better-auth two-factor plugin.
- FR-11.4 Invitation-only access: administrators issue invitations; registration requires a valid, unexpired, unused invitation token. There is no public registration path.
- FR-11.5 Password policy: minimum 12 characters and rejection of passwords found in Have-I-Been-Pwned breach data.
- FR-11.6 Rate limiting by IP on authentication endpoints (login, invitation registration, password recovery).
- FR-11.7 Administration backoffice (role-based, integrated in the same client app): user management, invitation issuance and revocation, deactivation.
- FR-11.8 Data isolation: strict `userId` filtering on every user-domain query, backed by PostgreSQL Row-Level Security (ADR-0006; NFR §5.1).

**Acceptance criteria** (new in Koinomy — no v1 spec exists for this module; derived from ADR-0004 and the v1 PRD text).

- GIVEN the registration endpoint WHEN a request arrives without a valid invitation token THEN there is no path to create an account (no public registration endpoint exists at all).
- GIVEN a valid invitation token WHEN the invitee completes registration THEN the account is created and the token is marked used; GIVEN an expired or already-used token THEN registration is rejected.
- GIVEN a registered user WHEN they sign in THEN 2FA enrollment is mandatory before the account is usable; once enrolled, login requires the second factor.
- GIVEN a password shorter than 12 characters or present in HIBP breach data WHEN used at registration or password change THEN it is rejected with a clear message.
- GIVEN an authenticated session idle for 15 minutes WHEN the next request arrives THEN the session is rejected and the user must re-authenticate.
- GIVEN any environment, including development WHEN any request arrives THEN identity comes only from the authenticated session; no header, query parameter, or dev-only code path can override it.
- GIVEN repeated failed login attempts from one IP WHEN the rate limit is exceeded THEN further attempts are rejected for the window.
- GIVEN a USER-role session WHEN an admin backoffice endpoint is called THEN the request is rejected; GIVEN an ADMIN session THEN user and invitation management operations succeed.

## 5. Non-Functional Requirements

### 5.1 Security

Binding decisions: ADR-0004 (authentication hardening), ADR-0005 (field-level encryption), ADR-0006 (tenant isolation), and the red lines in `AGENTS.md` §2.1. Hardening baseline (security headers via helmet, CORS allowlist from validated env, rate limiting, boot-time env validation): `docs/SECURITY.md`.

- No impersonation bypass in any environment. No wildcard CORS. Every environment variable validated at boot. Every user-facing query filtered by `userId`. No silent security stubs — unimplemented security components fail boot.
- Cross-tenant access attempts return 404 (anti-enumeration), never 403.

### 5.2 Privacy

- Sensitive financial values (amounts, balances, names, notes, descriptions) are encrypted at the application layer before persistence. The authoritative encrypted-field catalog — every protected field, what it protects, and its storage format — lives in `docs/DATABASE.md` §5.
- Adding a new sensitive field requires a catalog entry in the same PR (ADR-0005).
- API responses never expose `userId`, `encrypted*` columns, notification bodies, or unmasked recipient addresses.

### 5.3 Testing

Binding decision: ADR-0007. Details in `docs/TESTS.md`.

- TDD is mandatory for financial and security logic: Red → Green → Refactor (`AGENTS.md` §4).
- **Enforced coverage thresholds (CI-gated, ratchet up only, never down): 90% line coverage for financial and security modules; 80% line coverage overall.** v1 claimed "100% mandatory" and enforced nothing; Koinomy states only enforceable truth.
- Database-backed tests run on testcontainers — never a developer's local database. Backend unit tests mock dependency injection, especially the `EncryptionService`. API e2e via Supertest; frontend e2e via Playwright.

### 5.4 CI and Quality Gates

Binding decision: ADR-0008. Every merge passes, in order: lint (ESLint + Prettier), typecheck (`tsc --noEmit` both workspaces), unit + integration tests with coverage thresholds, e2e tests, build. CI red blocks merge with no manual overrides. Renovate keeps dependencies current (weekly grouped minors/patches; security alerts immediately). Husky + lint-staged gate commits.

### 5.5 Performance and UX Baselines

- Paginated list endpoints: default page size 20, maximum 100.
- Dashboard widgets load in parallel with skeleton states; one failing widget never blocks the rest (FR-9.4).
- Monetary values travel as decimal strings with 6 decimal places — never binary floats — across the API boundary.
- Encrypted fields cannot be filtered or aggregated in SQL; queries decrypt in the application layer (ADR-0005 consequence). Aggregations use exact decimal arithmetic.
- The client caches server state through TanStack Query; hand-rolled fetch-and-cache hooks are prohibited (v1 lesson, `docs/ARCHITECTURE.md` §9).

## 6. Non-Goals

Explicitly out of scope for Koinomy:

1. **Public registration.** Access is invitation-only, always (FR-11.4).
2. **Native mobile applications.** The mobile vision shapes the architecture (standalone API) but no native app is built in Koinomy.
3. **Automated bank syncing / Open Banking.** Data entry is manual plus the automated recurring/installment/rate flows described above.
4. **Crypto assets.** No cryptocurrency holdings, tokens, or on-chain integrations.
5. **Automatic materialization of recurring transactions.** Rules remind and Pay Now posts; no auto-billing (v1 pivot, REQ-RP-10/REQ-RP-11).
6. **Multi-currency conversion inside budget actuals and snapshot net-worth sums.** Carried over from v1 as documented limitations; revisiting either requires an SDD change and, for budgets, a base-currency decision.
7. **Push notifications, SMS, in-app notification center, per-user notification preferences.** Email only (v1 notifications scope).
8. **Attachments/receipts on transactions.** No storage column exists in the data model.

Module-level deferred follow-ups inherited from v1 (e.g., category deletion, PayNow server-side idempotency, budget rollover) are not permanent non-goals; each enters through its own SDD change when prioritized.

## 7. Re-Porting Note

Behavior is re-ported module by module from the v1 OpenSpec specs under the SDD workflow (`AGENTS.md` §4): each re-port creates its own Koinomy change with delta specs re-created from the v1 scenarios, a design, tasks, and a verify report. v1 code is never copied wholesale.

**Recommended module order** (foundations first, dependents later):

| Order | Module | Rationale |
|-------|--------|-----------|
| 1 | Authentication & security foundation (Module 11 core: better-auth, sessions, invitation flow) | Every other module requires an authenticated `userId`; isolation and hardening must exist before any tenant data does |
| 2 | Currencies & exchange rates (Module 4) | Shared global catalog; accounts and transactions FK into it; ingestion cron is self-contained |
| 3 | Categories (Module 2) and Accounts & entities (Module 1) | The taxonomy and the ledger's containers; both are prerequisites for any movement of money |
| 4 | Transactions & transfers (Module 3.1) | The journal itself; balances, budgets, and the dashboard all aggregate it |
| 5 | Budgets (Module 6) | Reads transaction and recurring aggregates; needs the journal in place |
| 6 | Recurring payments (Module 3.2) | Reminders feed budgets' `fixedExpected` and notifications; Pay Now reuses the transaction service |
| 7 | Savings goals (Module 8) | Builds on accounts (host account rules, real available balance) |
| 8 | Debt & installments (Module 5) | Reuses the transaction service's DEBT_PAYMENT path and accounts' available credit |
| 9 | Snapshots (Module 7) | Captures account balances; comparative logic needs history to accumulate |
| 10 | Notifications (Module 10) | Cross-cutting delivery used by recurring, debt, snapshots, and rate alerting; wired last so producers already exist |
| 11 | Administration backoffice & remaining Module 11 features (2FA enforcement, HIBP, timeout tuning) | Admin operations manage the tenant base; 2FA/admin plugins land on top of the working auth foundation |
