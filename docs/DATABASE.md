# Database Design — Koinomy Data Dictionary

**Sync rule.** This document is the single source of truth for the Koinomy relational database design. Any PR that modifies the Drizzle schema, a migration, or any state this document describes MUST update this document in the same PR. v1 failed exactly at this sync rule (documentation describing tables that did not exist, and omitting tables that did); Koinomy does not inherit that failure.

## 1. Sync Rule and Hard Law

1. **No fields or tables outside this document may be invented.** If a change needs a column, table, index, or enum value not listed here, this document is updated first (same PR), then the schema follows.
2. **Schema changes follow AGENTS.md §5.8:** `drizzle-kit generate` → committed and reviewed SQL migrations → `drizzle-kit migrate` → `tsc --noEmit`. Migration SQL is code (§7).
3. **Re-port fidelity.** Koinomy is a clean-slate restart (ADR-0001), but the v1 model names and field semantics are preserved exactly so the v1 OpenSpec behavior contracts re-port without translation drift. Where v1's own `DATABASE.md` drifted from the v1 Prisma schema, the v1 Prisma schema was authoritative for this document and the drift is not propagated (§6, §7).
4. **Target engine.** PostgreSQL 16+ through Drizzle ORM (ADR-0003). This dictionary is written in relational notation (tables, columns, types, constraints) — deliberately no ORM snippets, so the design does not drift before code exists.

### 1.1 Naming convention

Logical identifiers in this document match the v1 names exactly: `PascalCase` table names, `camelCase` column names. The Drizzle schema maps them 1:1. Renaming a table or column is a design change and requires updating this document first (rule 1).

### 1.2 Type notation

| In this document | PostgreSQL type | Notes |
|------------------|-----------------|-------|
| `uuid` | `uuid` | Primary keys default to `gen_random_uuid()` unless stated otherwise |
| `text` | `text` | |
| `numeric(18,6)` | `numeric(18,6)` | All money amounts stored in plaintext (rates only — money itself is encrypted, §5) and all exchange rates |
| `boolean` | `boolean` | |
| `integer` | `integer` | |
| `timestamptz` | `timestamp with time zone` | All date/time columns; instants are stored and compared in UTC |
| Enum name (e.g. `Role`) | PostgreSQL enum type | Catalog in §4; values preserved exactly from v1 |

`updatedAt` columns carry no database default or trigger: they are set by the application on every update.

## 2. Ownership and Isolation Model

Binding decisions: ADR-0006 (isolation), ARCHITECTURE.md §6 (RLS mechanics). Every table belongs to exactly one ownership class:

| Class | Meaning | RLS treatment |
|-------|---------|---------------|
| User-domain | Tenant-owned data; carries `userId` | Direct RLS policy comparing `userId` to `current_setting('app.current_user_id')` |
| User-domain child | Tenant-owned data without a direct `userId` column | RLS policy via parent-existence check (ARCHITECTURE.md §6) |
| better-auth owned | Owned and written exclusively by the auth layer | RLS-exempt (ARCHITECTURE.md §6); application reads through the auth boundary (§6) |
| Shared global catalog | Reference data for all tenants; intentionally no `userId` | RLS-exempt; justified below |
| System log | Audit trail written by system jobs; user-scoped rows | Direct RLS policy (rows carry `userId`); writers are system jobs, see note below |

### 2.1 Classification of all 19 tables

| Model | Class | `userId` column | Notes |
|-------|-------|-----------------|-------|
| `Currency` | Shared global catalog | No | ISO currency reference data; shared by design — every tenant sees the same currencies, and per-tenant copies would make exchange-rate ingestion and cross-tenant consistency impossible |
| `ExchangeRate` | Shared global catalog | No | Daily rate history per catalog currency; written by the system ingestion cron (ARCHITECTURE.md §7), read by all tenants |
| `User` | better-auth owned | — | Identity root (§6) |
| `Session` | better-auth owned | Yes (FK) | DB-backed cookie sessions; owned by better-auth, not by application queries |
| `AuthAccount` | better-auth owned | Yes (FK) | Provider accounts (credential, OAuth) |
| `Verification` | better-auth owned | No | Keyed by `identifier`, not by user |
| `Category` | User-domain | Yes | |
| `EntityBank` | User-domain | Yes | |
| `Account` | User-domain | Yes | |
| `Transaction` | User-domain | Yes | |
| `RecurringPayment` | User-domain | Yes | The reminder cron scans it cross-tenant — the documented exception (AGENTS.md §5.6) |
| `Budget` | User-domain | Yes | |
| `Snapshot` | User-domain | Yes | |
| `SnapshotAccountBalance` | User-domain child | **No** | RLS via parent `Snapshot.userId` (parent-existence check) |
| `SavingsGoal` | User-domain | Yes | |
| `SavingsGoalContribution` | User-domain | Yes | |
| `DebtConsumption` | User-domain | Yes | |
| `Installment` | User-domain child | **No** | RLS via parent `DebtConsumption.userId` (parent-existence check) |
| `NotificationLog` | System log (user-scoped) | Yes | See §2.2 |

### 2.2 NotificationLog classification

`NotificationLog` rows are written by system jobs (notification crons, ARCHITECTURE.md §7), never by user-facing endpoints — but each row belongs to a recipient user (`userId`, ON DELETE CASCADE). It is therefore classified as a **user-scoped system log**: it carries a direct RLS policy like any user-domain table, and the system jobs that write it operate under the documented cross-tenant exception pattern (AGENTS.md §5.6), establishing the RLS context per recipient when writing.

### 2.3 Child tables without a direct `userId` column

| Child table | Parent carrying `userId` | RLS strategy |
|-------------|--------------------------|--------------|
| `SnapshotAccountBalance` | `Snapshot` | Policy checks that the parent `Snapshot` row belongs to `current_setting('app.current_user_id')` |
| `Installment` | `DebtConsumption` | Policy checks that the parent `DebtConsumption` row belongs to `current_setting('app.current_user_id')` |

Do not add a denormalized `userId` to these tables to "simplify" the policy — the parent-existence check is the designed mechanism (ARCHITECTURE.md §6).

### 2.4 Checklist for every new tenant table (ADR-0006)

- [ ] `userId` column with FK to `User.id` (ON DELETE CASCADE) — or an explicit entry in §2.3 naming the parent that carries it.
- [ ] Application-level `userId` filtering in every repository method that touches it.
- [ ] RLS policy (direct or parent-existence) added in the same committed migration as the table.
- [ ] If it holds sensitive values: entry in the encrypted-field catalog (§5) in the same PR.
- [ ] This document updated in the same PR.

## 3. Model Catalog

Tables are listed in dependency order: shared catalogs → auth → user domain. Each entry states purpose, columns, constraints and indexes, and relations. FK actions are normative.

### 3.1 Currency

Shared ISO currency catalog (e.g. `USD`, `VES`). No `userId` — shared reference data (§2).

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `code` | text | NO | — | Primary key; natural key (ISO code) |
| `name` | text | NO | — | Display name |
| `symbol` | text | NO | — | Display symbol |
| `isActive` | boolean | NO | `true` | Soft deactivation; inactive currencies cannot back new user data (service-layer rule) |

**Constraints and indexes**

- PK: `code`.
- Index: `(isActive, code)`.

**Relations**

- Referenced by `ExchangeRate.currencyCode`, `Account.currencyCode`, `Transaction.currencyCode`, `SavingsGoalContribution.currencyCode` — all ON DELETE RESTRICT, ON UPDATE CASCADE.

### 3.2 ExchangeRate

Daily exchange-rate history per catalog currency (official and parallel rates). No `userId` — shared catalog written by the ingestion cron (ARCHITECTURE.md §7).

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `currencyCode` | text | NO | — | FK → `Currency.code` |
| `officialRate` | numeric(18,6) | NO | — | Official rate for the date |
| `parallelRate` | numeric(18,6) | NO | — | Parallel/market rate for the date |
| `date` | timestamptz | NO | — | Rate date (idempotent per UTC day) |
| `source` | text | NO | — | Ingestion source label (e.g. the rate API, `Manual`) |

**Constraints and indexes**

- UNIQUE: `(currencyCode, date)` — one row per currency per day.
- Index: `(currencyCode, date DESC)` — latest-rate lookups.
- FK: `currencyCode` → `Currency.code`, ON DELETE RESTRICT, ON UPDATE CASCADE.

### 3.3 User

Identity root. Owned by better-auth — see §6 before touching this table.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `email` | text | NO | — | UNIQUE |
| `name` | text | YES | — | Display name |
| `emailVerified` | boolean | NO | `false` | better-auth email verification flag |
| `image` | text | YES | — | Avatar URL |
| `isTwoFactorEnabled` | boolean | NO | `false` | Custom field — §6 warning applies |
| `twoFactorSecret` | text | YES | — | Custom field — §6 warning applies |
| `role` | `Role` | NO | `USER` | Custom field; ADMIN unlocks the backoffice |
| `isActive` | boolean | NO | `true` | Custom field; deactivation blocks access |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- UNIQUE: `email`.
- Index: `(email)`.

**Relations**

- Parent of `Session`, `AuthAccount`, and every user-domain table; all inbound FKs are ON DELETE CASCADE.

### 3.4 Session

DB-backed cookie session, owned by better-auth (ADR-0004).

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | text | NO | — | PK; better-auth generates its own ID format (not UUID) |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE |
| `token` | text | NO | — | UNIQUE; session cookie value |
| `expiresAt` | timestamptz | NO | — | Server-enforced expiry |
| `ipAddress` | text | YES | — | |
| `userAgent` | text | YES | — | |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- UNIQUE: `token`.
- Indexes: `(userId)`, `(token)`.
- FK: `userId` → `User.id`, ON DELETE CASCADE.

### 3.5 AuthAccount

Auth provider accounts (credential, OAuth), owned by better-auth.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | text | NO | — | PK; better-auth generates its own ID |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE |
| `accountId` | text | NO | — | Provider-scoped account identifier |
| `providerId` | text | NO | — | Provider identifier (e.g. `credential`) |
| `accessToken` | text | YES | — | OAuth access token |
| `refreshToken` | text | YES | — | OAuth refresh token |
| `accessTokenExpiresAt` | timestamptz | YES | — | |
| `refreshTokenExpiresAt` | timestamptz | YES | — | |
| `scope` | text | YES | — | Granted OAuth scopes |
| `idToken` | text | YES | — | OIDC id token |
| `password` | text | YES | — | Hashed password for credential accounts |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- UNIQUE: `(accountId, providerId)`.
- Index: `(userId)`.
- FK: `userId` → `User.id`, ON DELETE CASCADE.

### 3.6 Verification

One-time verification records (email verification, password reset), owned by better-auth.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | text | NO | — | PK; better-auth generates its own ID |
| `identifier` | text | NO | — | Lookup key (e.g. user email) |
| `value` | text | NO | — | Token value |
| `expiresAt` | timestamptz | NO | — | |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- Index: `(identifier)`.
- No `userId` column — keyed by `identifier`; better-auth owned and RLS-exempt (§2).

### 3.7 Category

Per-user transaction taxonomy with two levels: main categories (`parentId IS NULL`) and subcategories.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; tenant isolation |
| `name` | text | NO | — | Plaintext (not encrypted) |
| `type` | `CategoryType` | NO | `EXPENSE` | |
| `color` | text | YES | — | UI color |
| `parentId` | uuid | YES | — | FK → `Category.id`, ON DELETE RESTRICT; NULL = main category |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- Partial unique index: `UNIQUE (userId, name) WHERE parentId IS NULL` — main-category names are unique per user.
- Partial unique index: `UNIQUE (userId, parentId, name) WHERE parentId IS NOT NULL` — subcategory names are unique per parent per user.
- Indexes: `(userId)`, `(userId, parentId)`.
- FK: `userId` → `User.id`, ON DELETE CASCADE; `parentId` → `Category.id`, ON DELETE RESTRICT.
- The two-level depth cap (main → sub, no deeper) is enforced by a service-layer guard, not by the database.

### 3.8 EntityBank

Banks and credit entities owned by a user.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; tenant isolation |
| `name` | text | NO | — | Plaintext column; the value is encrypted at the application layer (§5) |
| `logoUrl` | text | YES | — | Public URL of the uploaded logo image |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- Index: `(userId)`.
- FK: `userId` → `User.id`, ON DELETE CASCADE.

**Relations**

- Referenced by `Account.entityId` (ON DELETE SET NULL).

### 3.9 Account

Financial accounts: checking, savings, credit cards, loans, cash.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; tenant isolation |
| `entityId` | uuid | YES | — | FK → `EntityBank.id`, ON DELETE SET NULL |
| `currencyCode` | text | NO | — | FK → `Currency.code`, ON DELETE RESTRICT, ON UPDATE CASCADE |
| `name` | text | NO | — | Plaintext column; the value is encrypted at the application layer (§5) |
| `type` | `AccountType` | NO | — | No database default; required at creation |
| `encryptedBalance` | text | NO | — | Encrypted current balance (§5) |
| `encryptedCreditLimit` | text | YES | — | Encrypted credit/overdraft limit; CREDIT and LOAN accounts only (§5) |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- Indexes: `(userId)`, `(userId, entityId)`.
- FKs: `userId` → `User.id` CASCADE; `entityId` → `EntityBank.id` SET NULL; `currencyCode` → `Currency.code` RESTRICT / UPDATE CASCADE.

**Relations**

- Referenced by `Transaction.accountId`, `SnapshotAccountBalance.accountId`, `SavingsGoal.hostAccountId`, `DebtConsumption.accountId`.

### 3.10 RecurringPayment

Recurring rules (fixed expenses, debt payments) with due-date tracking and reminder state. Outgoing types only — INCOME and TRANSFER rules are rejected (PRD FR-3.8). Listed before `Transaction` because `Transaction.generatedFromRecurringPaymentId` references it.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; tenant isolation |
| `categoryId` | uuid | NO | — | FK → `Category.id`, ON DELETE RESTRICT |
| `type` | `TransactionType` | NO | — | Kind of movement the rule generates |
| `encryptedAmount` | text | NO | — | Encrypted amount (§5) |
| `encryptedNote` | text | YES | — | Encrypted note (§5) |
| `frequency` | `RecurringPaymentFrequency` | NO | — | |
| `dayOfMonth` | integer | YES | — | For MONTHLY frequency |
| `dayOfWeek` | integer | YES | — | For WEEKLY frequency |
| `intervalDays` | integer | YES | — | For EVERY_N_DAYS frequency |
| `nextDueDate` | timestamptz | NO | — | Next occurrence; advanced on Pay Now |
| `lastNotifiedAt` | timestamptz | YES | — | Reminder state; set by the reminder cron |
| `notificationCount` | integer | NO | `0` | Reminder state; incremented by the reminder cron |
| `isActive` | boolean | NO | — | No database default; required at creation |
| `cancelledAt` | timestamptz | YES | — | Soft-delete marker for cancelled rules |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- Indexes: `(userId)`, `(userId, categoryId)`, `(userId, nextDueDate)`, `(nextDueDate)`.
- The `(nextDueDate)` index without `userId` supports the cross-tenant reminder cron — the documented exception pattern (AGENTS.md §5.6), not an isolation oversight.
- FKs: `userId` → `User.id` CASCADE; `categoryId` → `Category.id` RESTRICT.

### 3.11 Transaction

General ledger of all money movements (income, expense, transfer, debt payment).

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; tenant isolation |
| `accountId` | uuid | NO | — | FK → `Account.id`, ON DELETE RESTRICT |
| `categoryId` | uuid | NO | — | FK → `Category.id`, ON DELETE RESTRICT |
| `linkedTransactionId` | uuid | YES | — | FK → `Transaction.id`, ON DELETE SET NULL; pairs the two legs of a transfer/currency exchange (double entry) |
| `generatedFromRecurringPaymentId` | uuid | YES | — | FK → `RecurringPayment.id`, ON DELETE SET NULL; set when the transaction was created via Pay Now |
| `type` | `TransactionType` | NO | — | |
| `encryptedAmount` | text | NO | — | Encrypted amount (§5) |
| `currencyCode` | text | NO | — | FK → `Currency.code`, ON DELETE RESTRICT, ON UPDATE CASCADE |
| `exchangeRateApplied` | numeric(18,6) | NO | — | Frozen rate at transaction time; `1.000000` for same-currency operations |
| `date` | timestamptz | NO | — | When the movement occurred |
| `encryptedNote` | text | YES | — | Encrypted note (§5) |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- Indexes: `(userId)`, `(userId, accountId)`, `(userId, categoryId)`, `(userId, date DESC)`, `(linkedTransactionId)`, `(generatedFromRecurringPaymentId)`.
- FKs: `userId` → `User.id` CASCADE; `accountId` → `Account.id` RESTRICT; `categoryId` → `Category.id` RESTRICT; `currencyCode` → `Currency.code` RESTRICT / UPDATE CASCADE; `linkedTransactionId` → `Transaction.id` SET NULL; `generatedFromRecurringPaymentId` → `RecurringPayment.id` SET NULL.

### 3.12 Budget

Monthly planned amount per category.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; tenant isolation |
| `categoryId` | uuid | NO | — | FK → `Category.id`, ON DELETE RESTRICT |
| `periodMonth` | integer | NO | — | 1–12 |
| `periodYear` | integer | NO | — | 4-digit year |
| `encryptedPlannedAmount` | text | NO | — | Encrypted planned amount (§5) |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- UNIQUE: `(userId, categoryId, periodYear, periodMonth)` — one budget per category per month.
- Index: `(userId, periodYear, periodMonth)`.
- FKs: `userId` → `User.id` CASCADE; `categoryId` → `Category.id` RESTRICT.

### 3.13 Snapshot

Period closing: frozen net worth plus per-account balances (via `SnapshotAccountBalance`).

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; tenant isolation |
| `periodName` | text | NO | — | Display label (e.g. "May 2026 Closing") |
| `date` | timestamptz | NO | — | Closing date |
| `encryptedNetWorth` | text | NO | — | Encrypted net worth at closing (§5) |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- Index: `(userId, date DESC)`.
- FK: `userId` → `User.id`, ON DELETE CASCADE.

### 3.14 SnapshotAccountBalance

Per-account balance frozen inside a snapshot. **Child table without `userId`** — RLS via parent-existence check on `Snapshot` (§2.3).

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `snapshotId` | uuid | NO | — | FK → `Snapshot.id`, ON DELETE CASCADE |
| `accountId` | uuid | NO | — | FK → `Account.id`, ON DELETE RESTRICT |
| `encryptedBalanceCaptured` | text | NO | — | Encrypted balance frozen at closing (§5) |

**Constraints and indexes**

- Index: `(snapshotId)`.
- FKs: `snapshotId` → `Snapshot.id` CASCADE; `accountId` → `Account.id` RESTRICT.
- No timestamp columns — rows are immutable once captured (preserved from v1).

### 3.15 SavingsGoal

Savings goals. `DEDICATED` mode: progress equals the host account balance (1:1 with the account). `VIRTUAL` mode: progress equals the sum of contributions (N:1 with the account).

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; tenant isolation |
| `hostAccountId` | uuid | NO | — | FK → `Account.id`, ON DELETE CASCADE |
| `mode` | `SavingsGoalMode` | NO | `DEDICATED` | |
| `contributionFrequency` | `ContributionFrequency` | NO | `MONTHLY` | Period for projected fulfillment (7/14/30 days) |
| `startDate` | timestamptz | YES | — | Anchor date for the recurrence pattern; nullable for legacy goals, required by a service-layer guard for new goals |
| `encryptedName` | text | NO | — | Encrypted goal name (§5) |
| `encryptedTargetAmount` | text | NO | — | Encrypted target amount (§5) |
| `encryptedExpectedContributionAmount` | text | NO | — | Encrypted expected contribution per period (§5) |
| `targetDate` | timestamptz | YES | — | Projected fulfillment date; nullable (stored for legacy goals, derived for new goals) |
| `isActive` | boolean | NO | `true` | |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- Partial unique index: `UNIQUE (hostAccountId) WHERE mode = 'DEDICATED'` — at most one DEDICATED-mode goal per host account; VIRTUAL-mode goals are unconstrained.
- Indexes: `(userId, targetDate)`, `(userId, startDate)`, `(userId, mode)`, `(hostAccountId)`.
- FKs: `userId` → `User.id` CASCADE; `hostAccountId` → `Account.id` CASCADE.
- The goal carries **no currency column** — the currency is always sourced from the host account (v1 final state after dropping `SavingsGoal.currencyCode`).

### 3.16 SavingsGoalContribution

Deposit/withdrawal log for VIRTUAL-mode goals.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; tenant isolation |
| `savingsGoalId` | uuid | NO | — | FK → `SavingsGoal.id`, ON DELETE CASCADE (contributions die with the goal) |
| `type` | `ContributionType` | NO | — | |
| `encryptedAmount` | text | NO | — | Encrypted amount (§5) |
| `currencyCode` | text | NO | — | FK → `Currency.code`, ON DELETE RESTRICT, ON UPDATE CASCADE; copied from the host account's currency at creation time (see note below) |
| `date` | timestamptz | NO | — | When the deposit/withdrawal occurred |
| `encryptedNote` | text | YES | — | Encrypted memo (§5) |
| `categoryId` | uuid | YES | — | FK → `Category.id`, ON DELETE RESTRICT; required for WITHDRAWAL, null for DEPOSIT (service-layer rule) |
| `createdAt` | timestamptz | NO | `now()` | |

**Constraints and indexes**

- Indexes: `(savingsGoalId, date DESC)`, `(userId)`.
- FKs: `userId` → `User.id` CASCADE; `savingsGoalId` → `SavingsGoal.id` CASCADE; `categoryId` → `Category.id` RESTRICT; `currencyCode` → `Currency.code` RESTRICT / UPDATE CASCADE.
- No `updatedAt` — rows are not modified after creation (preserved from v1).
- Note: `currencyCode` had **no FK** to `Currency.code` in the v1 schema (drift). Koinomy adds the FK at design time — decided in the pre-implementation documentation audit — because no schema exists yet and referential integrity is free.

### 3.17 DebtConsumption

Installment-based consumption (extra-financing): a purchase split into installments on a CREDIT or LOAN account.

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; tenant isolation |
| `accountId` | uuid | NO | — | FK → `Account.id`, ON DELETE RESTRICT; account must be CREDIT or LOAN (service-layer rule) |
| `categoryId` | uuid | YES | — | FK → `Category.id`, ON DELETE SET NULL |
| `encryptedTotalAmount` | text | NO | — | Encrypted total financed amount (§5) |
| `installments` | integer | NO | — | Total number of installments |
| `remainingInstallments` | integer | NO | — | Installments not yet paid |
| `encryptedDescription` | text | NO | — | Encrypted description (§5) |
| `isActive` | boolean | NO | `true` | |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- Indexes: `(userId)`, `(accountId)`.
- FKs: `userId` → `User.id` CASCADE; `accountId` → `Account.id` RESTRICT; `categoryId` → `Category.id` SET NULL.

### 3.18 Installment

Individual installment of a `DebtConsumption`. **Child table without `userId`** — RLS via parent-existence check on `DebtConsumption` (§2.3).

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `debtConsumptionId` | uuid | NO | — | FK → `DebtConsumption.id`, ON DELETE CASCADE |
| `dueDate` | timestamptz | NO | — | Installment due date |
| `encryptedAmount` | text | NO | — | Encrypted installment amount (§5) |
| `status` | `InstallmentStatus` | NO | `PENDING` | |
| `paidAt` | timestamptz | YES | — | When it was paid |
| `paidTransactionId` | uuid | YES | — | UNIQUE; FK → `Transaction.id`, ON DELETE RESTRICT; identifies the paying transaction (see note below) |
| `createdAt` | timestamptz | NO | `now()` | |
| `updatedAt` | timestamptz | NO | — | Set by the application on update |

**Constraints and indexes**

- UNIQUE: `paidTransactionId` (at most one installment paid by a given transaction).
- Index: `(debtConsumptionId)`.
- FKs: `debtConsumptionId` → `DebtConsumption.id`, ON DELETE CASCADE; `paidTransactionId` → `Transaction.id`, ON DELETE RESTRICT.
- Note: `paidTransactionId` had **no FK** to `Transaction.id` in the v1 schema (unique column only — drift). Koinomy adds the FK at design time — decided in the pre-implementation documentation audit. ON DELETE RESTRICT is safe because transactions are append-only (no DELETE route, PRD FR-3.5).

### 3.19 NotificationLog

Audit log of outbound email notifications: one row per send attempt, SENT or FAILED (ARCHITECTURE.md §7, §10). User-scoped system log (§2.2).

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `userId` | uuid | NO | — | FK → `User.id`, ON DELETE CASCADE; recipient user |
| `type` | `NotificationType` | NO | — | |
| `recipient` | text | NO | — | Destination email address |
| `subject` | text | NO | — | Email subject |
| `body` | text | YES | — | Email body |
| `referenceId` | text | YES | — | Free-form reference to the related entity (no FK) |
| `status` | `NotificationStatus` | NO | — | Send outcome |
| `errorMessage` | text | YES | — | Failure detail for FAILED rows |
| `sentAt` | timestamptz | NO | `now()` | Attempt timestamp |

**Constraints and indexes**

- Indexes: `(userId, sentAt DESC)`, `(type)`, `(status)`, `(type, referenceId, sentAt)` — the last one supports the reminder-cron dedup lookups (ARCHITECTURE.md §7).
- FK: `userId` → `User.id`, ON DELETE CASCADE.
- No `updatedAt` — rows are immutable once written (preserved from v1).

## 4. Enum Catalog

All 11 enums. Values are preserved exactly from v1, including the lowercase snake_case values of `NotificationType`.

| Enum | Values | Semantics |
|------|--------|-----------|
| `Role` | `ADMIN`, `USER` | Authorization role; ADMIN unlocks the admin backoffice |
| `CategoryType` | `INCOME`, `EXPENSE`, `TRANSFER` | Kind of movement a category classifies |
| `AccountType` | `CHECKING`, `SAVINGS`, `CREDIT`, `LOAN`, `CASH` | Nature of a financial account |
| `TransactionType` | `INCOME`, `EXPENSE`, `TRANSFER`, `DEBT_PAYMENT` | Kind of ledger movement (transactions and recurring rules) |
| `RecurringPaymentFrequency` | `MONTHLY`, `WEEKLY`, `EVERY_N_DAYS`, `DAILY` | Cadence of a recurring rule |
| `SavingsGoalMode` | `DEDICATED`, `VIRTUAL` | Progress source: host account balance vs. sum of contributions |
| `ContributionFrequency` | `WEEKLY`, `BIWEEKLY`, `MONTHLY` | Expected contribution cadence (7/14/30-day projection) |
| `ContributionType` | `DEPOSIT`, `WITHDRAWAL` | Direction of a savings contribution |
| `InstallmentStatus` | `PENDING`, `PAID`, `OVERDUE` | Installment lifecycle state |
| `NotificationType` | `recurring_payment_reminder`, `installment_reminder`, `delinquency`, `deposit_reminder`, `closing_report`, `exchange_rate_sync_failure` | Kind of outbound notification |
| `NotificationStatus` | `SENT`, `FAILED` | Delivery outcome of a notification attempt |

## 5. Encrypted-Field Catalog

Binding decision: ADR-0005. This catalog is authoritative — ARCHITECTURE.md §5 and ADR-0005 point here.

**Storage format.** Every encrypted value is AES-256-GCM ciphertext stored as `base64(IV || ciphertext || authTag)` in a plain `text` column, with a fresh random 12-byte IV per encryption operation. The layout is self-describing; GCM authentication detects tampering. The master key is a 32-byte base64-encoded value loaded from an environment variable validated at boot (ADR-0005, ADR-0008); a missing or malformed key fails boot.

### 5.1 Encrypted columns (17)

| Model | Field | Protects | Notes |
|-------|-------|----------|-------|
| `Account` | `encryptedBalance` | Current account balance | Money amount — decrypted for all arithmetic |
| `Account` | `encryptedCreditLimit` | Credit/overdraft limit | Nullable; CREDIT and LOAN accounts |
| `Transaction` | `encryptedAmount` | Transaction amount | Money amount |
| `Transaction` | `encryptedNote` | Transaction note | Nullable |
| `RecurringPayment` | `encryptedAmount` | Rule amount | Money amount |
| `RecurringPayment` | `encryptedNote` | Rule note | Nullable |
| `Budget` | `encryptedPlannedAmount` | Monthly planned amount | Money amount |
| `Snapshot` | `encryptedNetWorth` | Net worth at closing | Money amount |
| `SnapshotAccountBalance` | `encryptedBalanceCaptured` | Account balance frozen at closing | Money amount |
| `SavingsGoal` | `encryptedName` | Goal name | Identity-sensitive |
| `SavingsGoal` | `encryptedTargetAmount` | Target amount | Money amount |
| `SavingsGoal` | `encryptedExpectedContributionAmount` | Expected contribution per period | Money amount |
| `SavingsGoalContribution` | `encryptedAmount` | Contribution amount | Money amount |
| `SavingsGoalContribution` | `encryptedNote` | Contribution memo | Nullable |
| `DebtConsumption` | `encryptedTotalAmount` | Total financed amount | Money amount |
| `DebtConsumption` | `encryptedDescription` | Purchase description | Required |
| `Installment` | `encryptedAmount` | Installment amount | Money amount |

### 5.2 Application-encrypted plain columns (2)

These two columns carry no `encrypted*` prefix — the column is a plain `name` — but the application layer encrypts the value into it using the same adapter and storage format:

| Model | Field | Protects | Notes |
|-------|-------|----------|-------|
| `Account` | `name` | Account name | Encrypted at the application layer into a plain `name` column |
| `EntityBank` | `name` | Bank/entity name | Encrypted at the application layer into a plain `name` column |

### 5.3 Rules

1. **No SQL-side computation.** Encrypted fields cannot be filtered, sorted, or aggregated in SQL. Any computation (totals, comparisons, projections) decrypts in the application layer with exact decimal arithmetic (ARCHITECTURE.md §5).
2. **Catalog-first for new sensitive fields.** Any new sensitive field MUST be added to this catalog in the same PR that adds the field to the schema (ADR-0005 consequence).
3. **Encryption happens in the service layer only** — never in controllers or repositories (ARCHITECTURE.md §2).
4. Key rotation is an operational procedure documented in `docs/OPERATIONS.md`.

## 6. better-auth Tables Note

Binding decision: ADR-0004.

- `User`, `Session`, `AuthAccount`, and `Verification` are **owned by the auth layer**. Application code reads them through the better-auth boundary and never writes them directly — writes happen only through better-auth APIs. They are RLS-exempt (ARCHITECTURE.md §6).
- `Session.id`, `AuthAccount.id`, and `Verification.id` are plain `text`: better-auth generates its own ID format (not UUID). Do not convert them.
- **Custom `User` fields** (beyond the better-auth baseline): `role`, `isActive`, `isTwoFactorEnabled`, `twoFactorSecret`. Credential password hashes live exclusively in `AuthAccount.password`, owned by better-auth — v1's duplicated `User.passwordHash` is NOT carried over (no v1 spec backs it; two copies of the same hash are risk without benefit).
- **Dead-column warning.** `isTwoFactorEnabled` and `twoFactorSecret` MUST NOT exist as dead columns: they ship together with the working 2FA implementation (PRD Module 11, better-auth `twoFactor` plugin). v1 shipped them dead — that failure mode is a documented reason for the Koinomy restart (ADR-0001/0004). If 2FA work has not landed, these two columns are defined here as the contract it ships against, not as existing schema.
- **No `Invitation` table.** The v1 `DATABASE.md` documented an `Invitation` table that never existed in the v1 schema (known drift — not propagated). The invite-only flow will be modeled through better-auth's admin/invitation capabilities when PRD Module 11 is ported; any resulting table is added to this document first (§1 rule 1).

## 7. Migration Discipline

Binding rule: AGENTS.md §5.8.

1. **Generate and commit.** `drizzle-kit generate` produces SQL migration files. They are committed and reviewed like code — they are the auditable history of this design.
2. **Never gitignored.** v1's migrations directory was gitignored, so migrations were silently dropped unless force-added (the v1 "Issue #18 workaround"). Concrete evidence survives in the v1 repo: `DebtConsumption`, `Installment`, and `NotificationLog` have no committed CREATE TABLE migration at all — only later ALTERs — because their creation migrations never made it into version control. That failure mode is permanently closed in Koinomy.
3. **Generate is not enough.** `drizzle-kit generate` only writes the SQL files; `drizzle-kit migrate` (or the project's deploy script) applies them. Done-criteria for every schema change (AGENTS.md §5.8):

   ```bash
   npx drizzle-kit generate   # SQL migration files — committed, never gitignored
   npx drizzle-kit migrate    # applies them to the dev DB
   npx tsc --noEmit           # typechecks against the schema's inferred types
   ```

4. **Hand-tuned SQL lives in committed migrations.** Partial unique indexes (§3.7 Category, §3.15 SavingsGoal), RLS policies (§2), and any other hand-authored SQL are migration files like any other — reviewed, committed, and applied by the migration tooling. They are never applied manually to a database.
