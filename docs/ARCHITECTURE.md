# Architecture — Koinomy (Target)

**Sync rule.** This document describes the Koinomy **target architecture — the architecture to be built; nothing is implemented yet**. It tracks the structural decisions locked by the ADRs. Any PR that introduces or changes architectural structure (layers, middleware order, adapters, isolation mechanics, error contract) MUST update this document in the same PR.

## 1. System Overview and Repository Layout

Koinomy is a self-hosted, decoupled client/server system: a REST API (NestJS 11) serving a single-page client (React 19), backed by PostgreSQL 16+ through Drizzle ORM (ADR-0002, ADR-0003).

```text
Koinomy/
├── api/        # NestJS 11 workspace — REST API, auth, encryption, crons
├── client/     # React 19 + Vite workspace — SPA, admin backoffice hidden by role
└── docs/       # Source-of-truth documentation (this folder)
```

- **Monorepo:** npm workspaces with two members, `api` and `client` (ADR-0002).
- **Sharing policy:** nothing is shared between workspaces yet — types and zod schemas are duplicated by convention ("types-by-convention") until a third use case justifies extracting a shared package. Premature sharing is a v1 lesson: it couples release cadence before the contract is stable.
- **Validation language:** zod everywhere — API input schemas, environment configuration, and client forms (AGENTS.md §2).

## 2. Backend Layering

Strict Controller → Service → Repository layering over Drizzle (Clean Architecture, AGENTS.md §2):

| Layer | Responsibility | Receives |
|-------|----------------|----------|
| Controller | HTTP boundary: routes, zod-validated input, response mapping. No business logic, no SQL | Validated DTOs; `userId` from the session guard |
| Service | Business rules, encryption/decryption, transaction boundaries (`db.transaction`), defense-in-depth guards | Domain inputs + `userId` as an explicit argument |
| Repository | Drizzle queries only. Every user-domain method takes `userId` in its type signature — omitting it is a compile error or a review-blocker (ADR-0006) | `userId` + query parameters |

- **DTOs are zod schemas** at the controller boundary; invalid input never reaches a service (global validation pipe, §3).
- **Domain types live per module** (e.g., `TransactionDomain` defined once in the module's `types.ts`); private `toDomain()`/`fromDomain()` mappers per service are an accepted pattern (AGENTS.md §5.2).
- **Encryption happens in the service layer**, never in controllers or repositories: services call the `EncryptionService` adapter before writes and after reads (§5).
- **Isolation happens in the service/repository layers:** `userId` comes from the authenticated session — never from client input — and flows down as an explicit argument (§6).
- **Atomicity:** multi-write operations use `db.transaction(...)` directly in services (idiomatic Drizzle primitive, AGENTS.md §5.3).

## 3. Request Lifecycle

Ordered pipeline for every incoming request:

| # | Stage | Behavior |
|---|-------|----------|
| 1 | Security headers (helmet) | Baseline hardening headers on every response |
| 2 | CORS allowlist | Origins validated against the env-configured allowlist; never wildcard, never origin reflection with credentials (red line, AGENTS.md §2.1) |
| 3 | Body parsing | JSON body parsing with size limits |
| 4 | better-auth handler | Requests under `/api/auth/*` are handled by better-auth and return before application guards |
| 5 | Global zod validation pipe | Body/params/query validated against the route's zod schema; failures become the uniform validation error envelope (§11) |
| 6 | Session guard | Resolves the DB-backed cookie session to a `userId`; unauthenticated requests get 401. This is the ONLY identity source (no impersonation bypass in any environment) |
| 7 | Controller | Route handler receives validated input + `userId` |
| 8 | Service | Business rules, encryption, transaction boundaries |
| 9 | Repository | Drizzle queries with `userId` filtering (+ RLS context, §6) |
| 10 | Global exception filter | Any thrown error is mapped to the uniform error envelope (§11); nothing leaks stack traces to clients |
| 11 | Request logging | Structured access log entry (method, path, status, duration, userId) written when the response finishes; failures are logged, never swallowed |

## 4. Authentication Architecture

Binding decision: ADR-0004.

- **Library:** better-auth with its **Drizzle adapter** (the Prisma-adapter-equivalent for the Drizzle data layer), **DB-backed cookie sessions** (`Session` table, §`docs/DATABASE.md`).
- **Session guard:** protected routes resolve `userId` from the session cookie via the better-auth session lookup; the guard populates the request context consumed by controllers.
- **Plugin roadmap (PRD Module 11):** `twoFactor` plugin for mandatory 2FA; admin/invitation capabilities for the invite-only flow and backoffice. Plugins are adopted, not hand-rolled.
- **Schema discipline:** the `isTwoFactorEnabled` / `twoFactorSecret` columns ship **together with** the working 2FA implementation — never as dead columns (the v1 failure mode, ADR-0001/0004).

**Hardening requirements (all environments, including development):**

1. No impersonation bypass: no `X-User-Id` header, no query-parameter identity, no dev-only override. Identity comes only from the authenticated session.
2. `trustedOrigins` comes from the validated env config — never hardcoded (v1 hardcoded localhost).
3. Cookies: HttpOnly, Secure, SameSite=Strict; 15-minute idle session timeout enforced server-side via DB sessions.
4. Auth endpoints (login, invitation registration, password recovery) are rate-limited.

## 5. Encryption Architecture

Binding decision: ADR-0005.

- **Adapter pattern:** an `EncryptionService` interface with `encrypt(plaintext): string` and `decrypt(ciphertext): string`.
  - `LocalEncryptionService` — implemented from day one: AES-256-GCM with a fresh random 12-byte IV per operation.
  - Vault/KMS adapter — interface reserved. **An adapter that is selected but not implemented MUST fail boot.** No stub may ever masquerade as a working provider (the v1 `VaultEncryptionService` failure mode, permanently closed).
- **Storage layout:** `base64(IV || ciphertext || authTag)` stored in plain string columns (e.g., `encryptedAmount`). The layout is self-describing; GCM authentication detects tampering.
- **Key management:** a 32-byte key, base64-encoded, loaded from an environment variable validated at boot by the zod env schema. Missing or malformed key → boot failure with a clear message.
- **Where it runs:** services encrypt before every write and decrypt after every read (§2). Encrypted fields cannot be filtered, sorted, or aggregated in SQL; computation decrypts in memory with exact decimal arithmetic.
- **Catalog and operations:** the authoritative encrypted-field catalog lives in `docs/DATABASE.md` §5; the key-rotation runbook lives in `docs/OPERATIONS.md`.

## 6. Multi-Tenant Isolation Architecture

Binding decision: ADR-0006. Two mandatory layers:

**Layer 1 — primary control: application-level `userId` filtering.** Every backend query that reads or writes tenant-owned data filters by the requester's `userId`, taken from the authenticated session (never from client input). Repository method signatures take `userId` so that omitting it is a compile-time or review-time failure. Any user-facing query without the filter is a **critical defect** (AGENTS.md §5.7).

**Layer 2 — defense-in-depth: PostgreSQL Row-Level Security.** RLS policies on tenant tables compare the row's `userId` against a request-scoped setting:

- The application sets `SET LOCAL app.current_user_id = <userId>` **inside the request's transaction**; policies evaluate `current_setting('app.current_user_id')`.
- **Connection-pool caveat:** `SET LOCAL` is transaction-scoped — it reverts automatically at commit/rollback, which is exactly why it is safe with pooled connections. The consequence: **RLS-protected work must run inside an explicit transaction**, and the application MUST set the variable per transaction (a transaction wrapper sets it as the first statement). A session-level `SET` would leak identity across pooled connections and is prohibited.
- Tables with a direct `userId` column get direct policies; child tables without one (`SnapshotAccountBalance`, `Installment`) get policies through a parent-existence check. better-auth-owned tables and shared catalogs are exempt (`docs/DATABASE.md` §2).

**Exceptions (the only ones):**

- **Shared global catalogs** — `Currency`, `ExchangeRate` — are reference data and intentionally have no `userId`.
- **Cross-tenant cron scans** — system-level scheduled tasks only (e.g., finding due rules for notification). Such repository methods carry an inline comment explaining the exception, and the spec carves it out explicitly (AGENTS.md §5.6).

## 7. Background Jobs

`@nestjs/schedule` crons; `ScheduleModule.forRoot()` is imported once at the root module.

| Job | Schedule | Behavior |
|-----|----------|----------|
| Exchange-rate ingestion | Daily 08:00 UTC | Fetches `open.er-api.com/v6/latest/USD`, upserts rates for catalog currencies (idempotent per UTC day). On failure: logs with a consecutive-failure counter; at the configured threshold (default 3, env-overridable) sends one email to the first ADMIN user; counter resets on success. Cross-tenant by design (shared catalog) |
| Recurring-payment reminders | Daily 09:00 UTC | Finds active, non-cancelled rules due within 7 days not yet notified today; sends `recurring_payment_reminder` emails; marks `lastNotifiedAt` and increments `notificationCount`. Cross-tenant scan — the documented exception pattern (AGENTS.md §5.6) |
| Installment / delinquency reminders | Daily | Upcoming installment reminders and past-due delinquency emails (PRD Module 10) |
| Savings deposit reminders | Daily | Deposit reminders for savings goals (PRD Module 10) |

Rules: cron failures never crash the process; notification sends log failures into `NotificationLog` (FAILED rows) and continue; cross-tenant scans exist only under the documented exception.

## 8. Configuration

- **One zod env schema, validated at boot.** The API parses its entire environment once, at startup, into a typed config object (red line, AGENTS.md §2.1).
- **Fail-fast with clear messages:** missing or invalid configuration fails boot naming the offending variable — configuration errors never surface as request-time crashes.
- **No scattered `process.env` reads.** Application code depends only on the typed config object injected through DI.
- **Env contract:** the full list of variables (database, auth origins, encryption key, SMTP, thresholds) is documented in `docs/OPERATIONS.md` and kept in sync with the zod schema.

## 9. Frontend Architecture

`client/` — React 19 + Vite, TypeScript strict (ADR-0002):

- **Routing:** react-router 7 with **lazy-loaded routes** (code-split per area; the admin backoffice is integrated in the same app and hidden by role-based access control).
- **Server state:** TanStack Query exclusively. **No hand-rolled fetch-and-cache hooks** — v1's ad-hoc hooks duplicated caching, loading, and retry logic inconsistently; that failure mode is closed by policy.
- **Forms:** react-hook-form + zod resolvers; the same validation language as the API boundary.
- **UI:** shadcn/ui + Radix primitives over Tailwind 4, implementing `docs/DESIGN.md`. v1 hand-rolled components are not carried over (ADR-0001).
- **Typed API client wrapper:** a single thin client used by all queries/mutations:
  - `credentials: 'include'` for cookie sessions;
  - uniform handling of the error envelope (§11) — errors surface as typed failures, never silent;
  - **401 → session-expired flow:** redirect to login with return path.
- **Dashboard composition:** widgets call existing module endpoints through TanStack Query in parallel; skeletons, error states, and empty states per widget (PRD Module 9).

## 10. Observability Baseline

- **Structured logging:** NestJS Logger with structured (JSON) output in production; context (module, userId where available, request id) on every line.
- **Request logging middleware:** method, path, status, duration, userId — stage 11 of the request lifecycle (§3).
- **Health endpoint:** liveness/readiness probe for self-hosted deployments (DB connectivity on readiness).
- **Cron failure alerting:** exchange-rate ingestion failure counter with ADMIN email threshold (§7); notification sends audit-logged in `NotificationLog` with SENT/FAILED status.
- **Bootstrap errors are fatal:** the bootstrap function catches startup failures, logs them, and exits non-zero. v1's `void bootstrap()` swallowed fatal errors and kept serving a broken app — that failure mode is closed (ADR-0001).

## 11. Error Handling Contract

**Uniform API error envelope** — every non-2xx response has this shape:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable summary safe to display",
    "details": [{ "field": "amount", "message": "must match ^\\d+\\.\\d{1,6}$" }]
  }
}
```

| Field | Rule |
|-------|------|
| `error.code` | Machine-readable, from the fixed set below |
| `error.message` | Human-readable; never exposes internals (no stack traces, no SQL, no tenant existence) |
| `error.details` | Optional; field-level entries for `VALIDATION_ERROR` only |

**Error code set and HTTP mapping:**

| HTTP | code | Typical source |
|------|------|----------------|
| 400 | `VALIDATION_ERROR` | zod pipe rejection; service-layer `BadRequestException` guards |
| 401 | `UNAUTHORIZED` | Missing/invalid/expired session |
| 403 | `FORBIDDEN` | Role violations (admin endpoints) — never used for cross-tenant access |
| 404 | `NOT_FOUND` | Missing resource **and** cross-tenant access (anti-enumeration) |
| 409 | `CONFLICT` | Uniqueness violations, double payment |
| 410 | `GONE` | Pay Now on a cancelled rule |
| 413 / 415 | `PAYLOAD_TOO_LARGE` / `UNSUPPORTED_MEDIA_TYPE` | Upload violations |
| 429 | `RATE_LIMITED` | Auth endpoint rate limiting |
| 503 | `SERVICE_UNAVAILABLE` | Upstream ingestion failure |
| 500 | `INTERNAL` | Anything unexpected |

**Centralized exception filter:** one global filter maps every thrown exception to the envelope; controllers and services throw typed exceptions, they never build error bodies by hand. Unknown exception types log full detail server-side and return the generic 500 envelope.
