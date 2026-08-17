# Operations Contract — Koinomy

**Sync rule.** The environment contract in §2 MUST match the zod env schema 1:1 — they are updated together in the same PR (red line: every variable is validated at boot, `AGENTS.md` §2.1 rule 3). Any PR that adds, removes, or renames an environment variable, or changes a deployment or key-management procedure described here, MUST update this document in the same PR.

## 1. Scope

This is the operations contract for the self-hosted deployment: environment configuration, local development, database operations, encryption key rotation, deployment, monitoring, and disaster notes. Security rationale for each control lives in `docs/SECURITY.md` and `docs/THREAT-MODEL.md`; the binding decisions are ADR-0005 (encryption), ADR-0007 (testcontainers), ADR-0008 (quality gates), and `AGENTS.md` §5.8 (schema changes).

## 2. Environment Contract

Every variable below is parsed **once, at boot**, by the API's zod env schema into a typed config object (ARCHITECTURE.md §8). Missing or invalid configuration **fails boot with the variable named** — configuration errors never surface as request-time crashes. Application code reads only the typed config object; scattered `process.env` reads are prohibited.

### 2.1 API runtime variables

| Variable | Required | Example | Validated at boot by |
|----------|----------|---------|----------------------|
| `DATABASE_URL` | Yes | `postgresql://user:pass@localhost:5432/koinomy` | PostgreSQL connection URL (`postgresql://` or `postgres://`) |
| `NODE_ENV` | Yes | `development` | Enum: `development` \| `test` \| `production` |
| `PORT` | No — default `3000` | `3000` | Integer 1–65535 |
| `ENCRYPTION_KEY` | Yes | output of `openssl rand -base64 32` | Base64 string decoding to **exactly 32 bytes** (AES-256 key, ADR-0005) |
| `CORS_ORIGINS` | Yes | `https://app.example.com` | Comma-separated list; each entry a valid http(s) origin; wildcards rejected (`AGENTS.md` §2.1 rule 2) |
| `BETTER_AUTH_URL` | Yes | `https://api.example.com` | Valid absolute URL (public base URL of the API) |
| `BETTER_AUTH_SECRET` | Yes | (random string) | String, minimum 32 characters |
| `BETTER_AUTH_TRUSTED_ORIGINS` | Yes | `https://app.example.com` | Comma-separated list of valid origins; **no hardcoded fallback** — v1 hardcoded localhost (ADR-0004) |
| `MAIL_HOST` | No | `smtp.example.com` | String. Absent → notifications run in dry-run mode (log only, PRD FR-10.1) |
| `MAIL_PORT` | With `MAIL_HOST` | `587` | Integer 1–65535 |
| `MAIL_USER` | No | `apikey` | String (provider-dependent) |
| `MAIL_PASS` | No | (provider secret) | String (provider-dependent) |
| `MAIL_FROM` | With `MAIL_HOST` | `noreply@koinomy.app` | Valid email address |
| `EXCHANGE_RATE_FAILURE_EMAIL_THRESHOLD` | No — default `3` | `3` | Positive integer; consecutive ingestion failures before the ADMIN alert email (PRD FR-4.6) |

### 2.2 Test-environment variables

| Variable | Required | Example | Validated at boot by |
|----------|----------|---------|----------------------|
| `TEST_ENCRYPTION_KEY` | Yes (test runs only) | `AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=` (base64 of 32 zero bytes) | Base64 decoding to exactly 32 bytes. Deterministic fixture key — **never used outside tests** |

**Removed:** v1's `DATABASE_URL_TEST` pointed tests at a shared local schema (`public_test`) on the developer's machine. Koinomy tests get a disposable PostgreSQL 16 container from testcontainers (ADR-0007); the container DSN is injected at test-run time, so no static variable exists. Tests never touch a developer's databases (`docs/TESTS.md` §4).

Client build-time variables (e.g., the API base URL for the SPA) are documented here in the same PR that introduces them, per the sync rule.

## 3. Local Development Setup

**Prerequisites:** Node.js LTS, PostgreSQL 16+ (local instance for the dev database), a container runtime (Docker or equivalent — required by testcontainers for the test suite), and `openssl` (key generation).

1. Clone the repository.
2. Install dependencies at the repository root (npm workspaces installs `api/` and `client/`):

   ```bash
   npm install
   ```

3. Create the API environment file and fill it from the contract (§2):

   ```bash
   cp api/.env.example api/.env
   openssl rand -base64 32   # paste the result as ENCRYPTION_KEY
   ```

4. Create the development database:

   ```bash
   createdb koinomy   # or: psql -c "CREATE DATABASE koinomy"
   ```

5. Apply migrations (first-time schema creation — §4.2):

   ```bash
   npx drizzle-kit migrate   # from api/
   ```

6. Seed the shared catalogs: ISO 4217 `Currency` rows (seed script provided with the Module 4 re-port).
7. Run both dev servers from the root:

   ```bash
   npm run dev               # api + client concurrently
   ```

**Test database strategy:** integration and e2e tests spin up PostgreSQL 16 via testcontainers and apply the committed migrations to the container (ADR-0007). **Tests never touch the developer's databases.** This closes the v1 failure mode where the test suite was coupled to a live local schema and results depended on developer-machine state.

## 4. Database Operations

### 4.1 Migration workflow

Binding rule: `AGENTS.md` §5.8. Every schema change runs, in order:

```bash
npx drizzle-kit generate   # produces SQL migration files — committed and reviewed, never gitignored
npx drizzle-kit migrate    # applies them to the target database
npx tsc --noEmit           # typechecks against the schema's inferred types
```

`generate` alone is not sufficient — it writes SQL files without applying them. Migration SQL files are code: reviewed, committed, and applied only through the migration tooling (`docs/DATABASE.md` §7).

### 4.2 First-time schema creation

A fresh database is created by running the full committed migration history with `drizzle-kit migrate` (step 5 of §3), followed by the catalog seed. Hand-authored SQL (partial unique indexes, RLS policies) lives inside committed migrations and is never applied manually to a database (`docs/DATABASE.md` §7 rule 4).

### 4.3 Backup and restore

- **Cadence:** `pg_dump` at least daily for any environment holding real tenant data. Recommended: custom format for flexible restore:

  ```bash
  pg_dump -Fc koinomy > koinomy_$(date +%F).dump
  ```

- **Restore:**

  ```bash
  pg_restore --clean --if-exists -d koinomy koinomy_YYYY-MM-DD.dump
  ```

- **Restore drill obligation:** a backup that has never been restored is not a backup. A restore drill is performed on a scheduled basis (recommendation: at least quarterly, and always after a schema change that touches encrypted fields), against a scratch database, including the key-reunion step below.
- **Key separation:** backups contain ciphertext for every protected field (`docs/DATABASE.md` §5). The encryption key is stored and transmitted **separately** from backups — a backup restore without deliberate key custody yields unreadable financial data by design (§8).

## 5. Encryption Key Rotation Runbook

Binding decision: ADR-0005. The catalog of fields this procedure covers is `docs/DATABASE.md` §5 (all 17 `encrypted*` columns plus the 2 application-encrypted plain `name` columns).

**When:**

- Suspected or confirmed key compromise (immediately).
- Periodic policy (recommendation: at least annually).
- Personnel change with prior key access.

**Procedure:**

1. **Generate** the new key: `openssl rand -base64 32`. Store it in the secret store; do not log it or commit it.
2. **Dual-key window:** configure the API to decrypt with the old key and encrypt with the new key. Run the re-encryption job across **every field in the catalog** (`docs/DATABASE.md` §5) — decrypt with old, re-encrypt with new, row by row.
3. **Verify counts:** assert the number of re-encrypted values per table equals the expected row counts; any mismatch aborts the rotation before cutover.
4. **Cut over:** deploy with `ENCRYPTION_KEY` set to the new key only; restart.
5. **Revoke the old key:** remove it from all environments and secret stores after a successful verification window.

**Rollback:** if verification fails during the dual-key window, stop the re-encryption job and keep the old key active — rows not yet re-encrypted still decrypt with it. After cutover (step 4), rollback requires re-running the procedure in reverse with the old key, which is why step 3 is mandatory before step 4.

**Drill rule:** this runbook is exercised end-to-end **at least once before production** (against a scratch database with realistic data). An untested runbook is aspirational — that is the v1 lesson encoded here.

## 6. Deployment

v1 docs named specific vendors; Koinomy keeps the target open and states **requirements** instead. Any deployment target satisfies all of:

| Requirement | Detail |
|-------------|--------|
| Node runtime | Node.js LTS running the built API workspace |
| PostgreSQL 16+ | Managed or self-run; reachable from the API; backups per §4.3 |
| Env injection | All §2 variables injected from a secret store — never baked into images or committed files |
| HTTPS termination | TLS terminated before the API (platform proxy or self-managed); Secure cookies require it |
| Health probes | Liveness/readiness endpoints per ARCHITECTURE.md §10; readiness includes DB connectivity |
| Static client headers | The SPA hosting applies security headers equivalent to the API's helmet baseline, including CSP (`docs/SECURITY.md` §4.3) |

**Deploy pipeline order:**

1. Build (CI-gated: lint, typecheck, tests with coverage, e2e have already passed — ADR-0008).
2. Run migrations: `npx drizzle-kit migrate` against the target database.
3. Rolling restart of the API with the new build.

**Bootstrap contract:** the bootstrap logs fatal startup failures and exits non-zero (ARCHITECTURE.md §10). A failed migration or invalid config therefore fails the deploy — the platform never serves a broken instance.

## 7. Monitoring and Alerting Baseline

| Signal | Mechanism | Alert path |
|--------|-----------|------------|
| API liveness/readiness | Health endpoints; readiness checks DB connectivity (ARCHITECTURE.md §10) | Platform health-probe failure → restart/alert per deployment target |
| Exchange-rate ingestion failures | Consecutive-failure counter; threshold from `EXCHANGE_RATE_FAILURE_EMAIL_THRESHOLD` (default 3) | One email to the first ADMIN user at threshold; counter resets on success (PRD FR-4.6) |
| Notification delivery | Every send attempt writes a `NotificationLog` row, SENT or FAILED (`docs/DATABASE.md` §3.19) | FAILED rows are the delivery audit; sustained FAILED volume is investigated |
| Cron health | Cron failures are logged and never crash the process (ARCHITECTURE.md §7) | Log-based; silent crons (no run entries) are treated as failures |
| Application logs | Structured JSON logs with request id, module, `userId`; sensitive data never logged (`docs/SECURITY.md` §6.2) | Recommendation: aggregate into any self-hosted log aggregator with retention; at minimum, persist logs beyond container lifetime |

## 8. Disaster Notes

- **`ENCRYPTION_KEY` lost:** tenant financial data is **unrecoverable by design** — amounts, balances, names, and notes exist only as AES-256-GCM ciphertext, and there is no escrow or backdoor. Key custody is the owner's responsibility: keep the key in a secret store with its own backup, stored separately from database backups (§4.3). Losing both a backup and the key is total data loss; losing only the key is equally total for encrypted fields.
- **SMTP failure / misconfiguration:** notifications degrade gracefully. Missing SMTP config → dry-run mode (log only); transport errors → FAILED `NotificationLog` rows, no exception propagates, and the application continues (PRD FR-10.1, FR-10.2). Email-dependent alerts (rate-ingestion failures, reminders) stop reaching users while the app itself keeps serving — monitor FAILED rows to detect it.
- **Database loss without backup:** unrecoverable. The §4.3 cadence and restore-drill obligation exist precisely for this case.
