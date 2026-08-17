# Security Baseline — Koinomy

**Sync rule.** This document is the Koinomy security hardening contract. It defines **what must hold**; `docs/THREAT-MODEL.md` defines **why and against whom**; implementation evidence lives in each SDD change's verify report (`AGENTS.md` §4). Any PR that changes a security behavior documented here MUST update this document in the same PR.

## 1. Scope

| Concern | Owner document |
|---------|----------------|
| What must hold (requirements, baseline, checklist) | This document |
| Why it must hold (threats, attackers, scenarios) | `docs/THREAT-MODEL.md` |
| Evidence that it holds (test runs, review checks) | SDD verify reports per change |
| Locked decisions behind each requirement | ADR-0004 (auth), ADR-0005 (encryption), ADR-0006 (isolation), ADR-0008 (quality gates); red lines in `AGENTS.md` §2.1 |
| Functional security requirements | `docs/PRD.md` Module 11 (all MUST) |

Every requirement below carries an owner stage from the re-port plan:

- **scaffold** — installed with the monorepo skeleton, before any feature module.
- **auth foundation** — lands with the Module 11 core re-port (better-auth, sessions, invitation flow — re-port order 1 in `docs/PRD.md` §7).
- **per-module** — lands with the specific module re-port that owns the behavior.

## 2. Authentication Hardening

Binding decisions: ADR-0004, PRD Module 11 (FR-11.2…FR-11.6).

1. **Session model.** better-auth with **DB-backed cookie sessions** (`Session` table, `docs/DATABASE.md` §3.4). Sessions are revocable server-side; there are no stateless JWTs.
2. **Cookie flags.** HttpOnly, Secure, SameSite=Strict on the session cookie, in every environment.
3. **Idle timeout.** 15-minute idle session timeout enforced **server-side** against the DB session (PRD FR-11.2). Client-side timers are UX sugar, never the enforcement point.
4. **Rate limiting.** Auth endpoints (login, invitation registration, password recovery) are rate-limited per IP; limits in the strategy table (§6).
5. **Trusted origins.** better-auth `trustedOrigins` comes from the validated env config (`BETTER_AUTH_TRUSTED_ORIGINS`, `docs/OPERATIONS.md` §2) — never hardcoded (v1 hardcoded localhost).
6. **No impersonation bypass — red line.** No `X-User-Id` header, no query-parameter identity, no dev-only override, in **any environment including development**. Identity comes only from the authenticated session (`AGENTS.md` §2.1 rule 1). The v1 `auth-guard` spec is explicitly not ported (`docs/PRD.md` §3).
7. **Password policy.** Minimum 12 characters, and rejection of passwords found in Have-I-Been-Pwned breach data, at registration and at every password change (PRD FR-11.5).
8. **Two-factor authentication.** Mandatory 2FA for all users via the better-auth `twoFactor` plugin (PRD FR-11.3). Enrollment is mandatory before the account is usable; once enrolled, login requires the second factor. The `isTwoFactorEnabled` / `twoFactorSecret` columns ship **together with** the working implementation — never as dead columns (`docs/DATABASE.md` §6).
9. **Invitation-only registration.** There is no public registration path. Administrators issue invitations; an invitation is a **real artifact** — cryptographically random token, expiry, single-use — not the v1 phantom (v1 documented an `Invitation` table that never existed; `docs/DATABASE.md` §6). Registration requires a valid, unexpired, unused token; the token is marked used on completion. Any resulting table is added to `docs/DATABASE.md` first (§1 rule 1 of that document).

## 3. Authorization and Tenant Isolation

Binding decisions: ADR-0006, ARCHITECTURE.md §6, PRD FR-11.8.

1. **Session-derived identity only.** `userId` is resolved from the authenticated session by the session guard (request lifecycle stage 6, ARCHITECTURE.md §3) and flows down as an explicit argument. Client-supplied identity parameters are ignored, never parsed (PRD Module 10 acceptance criteria demonstrate the pattern).
2. **Role model.** `Role` enum: `USER`, `ADMIN` (`docs/DATABASE.md` §4). ADMIN unlocks the administration backoffice; `isActive = false` blocks access.
3. **Admin backoffice guard.** Backoffice endpoints (user management, invitation issuance/revocation, deactivation — PRD FR-11.7) reject non-ADMIN sessions. Role violations return **403 FORBIDDEN** — the one legitimate use of 403 (ARCHITECTURE.md §11).
4. **Anti-enumeration.** Cross-tenant access to any user-domain resource returns **404 NOT_FOUND** with a single indistinguishable message — never 403, never a response that reveals the resource exists (ARCHITECTURE.md §11, PRD Module 1/2 acceptance criteria).
5. **Primary control — application filtering.** Every user-facing query filters by `userId`. Repository method signatures take `userId` so omitting it is a compile-time or review-time failure. Any user-facing query without the filter is a **CRITICAL** defect (`AGENTS.md` §5.7).
6. **Defense-in-depth — Row-Level Security.** RLS policies on all tenant tables evaluate `current_setting('app.current_user_id')`; the application sets `SET LOCAL app.current_user_id` as the first statement of **every transaction** (transaction-scoped, pool-safe; session-level `SET` is prohibited). Child tables without `userId` (`SnapshotAccountBalance`, `Installment`) are covered via parent-existence policies. better-auth tables and shared catalogs are the only exemptions (ARCHITECTURE.md §6, `docs/DATABASE.md` §2).

## 4. Input and Output Hardening

1. **Boundary validation.** Every route validates body/params/query with a zod schema at the controller boundary (request lifecycle stage 5). Invalid input never reaches a service.
2. **Service-layer guards.** Services re-validate security-relevant inputs (defense in depth against direct calls from crons, jobs, and tests). This is an accepted, documented pattern — reviewers must not flag it (`AGENTS.md` §5.1).
3. **Security headers.** helmet on the API with a CSP baseline: `default-src 'self'; object-src 'none'; frame-ancestors 'none'`, plus the standard helmet set (HSTS, X-Content-Type-Options, etc.). The static client hosting applies an equivalent or stricter CSP at the web server (`docs/OPERATIONS.md` §6). v1 shipped no headers at all.
4. **CORS.** Allowlist from the validated env (`CORS_ORIGINS`); never wildcard, never origin reflection with credentials (`AGENTS.md` §2.1 rule 2).
5. **Body size limits.** JSON bodies limited at the parser (initial: 100 KB); file uploads limited before any write (2 MB per logo, PRD FR-1.1) — oversized payloads return 413, unsupported types 415, with nothing written.
6. **Uniform error envelope.** Every non-2xx response uses the envelope of ARCHITECTURE.md §11, produced by the centralized exception filter. No stack traces, no SQL, no tenant-existence leakage, ever. Unknown exceptions log full detail server-side and return the generic 500 envelope.
7. **Upload security** (closes the v1 unauthenticated `/static` logo leak):
   - Type allowlist: PNG, JPEG, WebP only; **SVG rejected** (scriptable format). Validate declared type and content, not filename alone.
   - Size limit: 2 MB (PRD FR-1.1).
   - Storage: outside the webroot, random storage names, tenant-prefixed paths.
   - Delivery: **only through an authenticated endpoint** that checks ownership. There is no unauthenticated static mount. Cross-tenant logo paths resolve to 404 (anti-enumeration).

## 5. Cryptography

Binding decision: ADR-0005.

1. **Algorithm and adapter.** AES-256-GCM behind the `EncryptionService` adapter; `LocalEncryptionService` is implemented from day one. Storage layout `base64(IV || ciphertext || authTag)`, fresh random 12-byte IV per operation (ARCHITECTURE.md §5).
2. **Key provenance.** One 32-byte key, base64-encoded, loaded from `ENCRYPTION_KEY` in the validated env (`docs/OPERATIONS.md` §2). Missing or malformed key fails boot.
3. **No silent stubs — red line.** An adapter that is selected but not implemented MUST fail boot (`AGENTS.md` §2.1 rule 5). The v1 `VaultEncryptionService` — a throwing stub while docs claimed it worked — is the failure mode this permanently closes.
4. **Encrypted-field catalog.** Authoritative list of every protected field: `docs/DATABASE.md` §5 (17 `encrypted*` columns + 2 application-encrypted plain `name` columns). Adding a new sensitive field requires a catalog entry in the same PR.
5. **Key rotation.** Operational runbook in `docs/OPERATIONS.md` §5, exercised at least once before production.
6. **No home-grown crypto.** Nothing beyond the adapter. Password hashing is owned by better-auth (bcrypt/argon2 per its configuration) — never hand-rolled, never a dead dependency carried in the lockfile (v1 shipped a dead bcrypt).

## 6. Runtime and Infrastructure

1. **Fail-fast boot.** The bootstrap catches startup failures, logs them, and exits non-zero (ARCHITECTURE.md §10). A broken app never serves traffic (v1's `void bootstrap()` swallowed fatal errors).
2. **Structured logging without sensitive data.** JSON structured logs with context (module, request id, `userId`). **Never logged:** plaintext financial fields, decrypted names/notes, encryption keys, session tokens, passwords, full email bodies. Request logs carry method, path, status, duration, `userId` — not payloads (ARCHITECTURE.md §3 stage 11, §10).
3. **Rate limiting strategy table** (initial baseline — tuning changes update this document under the sync rule; state is in-memory, appropriate to the single-instance self-hosted deployment):

   | Endpoint class | Limit | Window | Keyed by | On exceed |
   |----------------|-------|--------|----------|-----------|
   | Login, invitation registration, password recovery | 10 requests | 15 min | IP | 429 `RATE_LIMITED` |
   | Session check, logout, 2FA challenge | 60 requests | 15 min | IP | 429 `RATE_LIMITED` |
   | Manual exchange-rate sync (FR-4.5) | 6 requests | 1 hour | user | 429 `RATE_LIMITED` |
   | File uploads (logos) | 30 requests | 1 hour | user | 429 `RATE_LIMITED` |
   | Authenticated mutations (CRUD writes) | 300 requests | 1 min | user | 429 `RATE_LIMITED` |
   | Authenticated reads | 600 requests | 1 min | user | 429 `RATE_LIMITED` |

4. **Dependency security.** Renovate: weekly grouped minors/patches, **security alerts handled immediately** outside the weekly cadence (ADR-0008). `npm audit` runs as a CI gate; high/critical findings block merge.
5. **Secrets handling.** Secrets live only in the environment, injected by the deployment target. Never in git, never in logs, never in error responses. `.env` files are gitignored; `.env.example` carries placeholders only (`docs/OPERATIONS.md` §2).

## 7. Hardening Checklist — v1 Gap Closure

Traceable closure record: every v1 audit finding, the Koinomy requirement that closes it, the stage that owns it, and the verification that proves it. Boxes are checked in the verify report of the change that lands the requirement.

| # | Requirement | Closes v1 gap | Owner stage | Verified by |
|---|-------------|---------------|-------------|-------------|
| 1 | helmet security headers + CSP baseline on API responses (§4.3) | No helmet / security headers | scaffold | e2e asserting header presence on responses |
| 2 | Rate limiting per strategy table (§6.3) | No rate limiting; auth and CRUD unthrottled | scaffold (middleware) + auth foundation (auth endpoints) | Integration test asserting 429 envelope after limit |
| 3 | CORS allowlist from validated env; no reflection with credentials (§4.4) | CORS reflected ANY origin with credentials | scaffold | Integration test: disallowed origin rejected |
| 4 | zod env schema validated at boot; fail-fast naming the variable (§6.1, OPERATIONS.md §2) | No env validation; raw `process.env`, request-time crashes | scaffold | Boot tests: missing/invalid variable fails boot |
| 5 | Identity from authenticated session only; no `X-User-Id` or equivalent in any env (§2.6) | Dev impersonation bypass in every non-production env | auth foundation | Negative e2e (header ignored) + review rule §8 |
| 6 | Unimplemented encryption adapter fails boot; `LocalEncryptionService` real (§5.3) | `VaultEncryptionService` throwing stub behind honest docs | scaffold (rule) + auth foundation (adapter) | Boot test + unit round-trip + tamper test |
| 7 | Uploads stored outside webroot, served only through an authenticated endpoint (§4.7) | Unauthenticated static serving of tenant logos at `/static` | per-module (Module 1) | e2e: unauthenticated logo URL → 404; cross-tenant → 404 |
| 8 | `trustedOrigins` from validated env (§2.5) | better-auth trustedOrigins hardcoded to localhost | auth foundation | Config test: boot fails without the variable |
| 9 | Centralized exception filter; uniform envelope; no stack traces (§4.6) | No exception filter; stack-trace leak risk | scaffold | e2e asserting 500 envelope shape, no internals |
| 10 | Bootstrap logs fatal errors and exits non-zero (§6.1) | `void bootstrap()` swallowed fatal startup errors | scaffold | Boot-failure test with non-zero exit |
| 11 | 15-minute idle session timeout, server-side (§2.3) | Module 11 unimplemented | auth foundation | Integration test: idle session → 401 |
| 12 | Mandatory 2FA via better-auth twoFactor plugin (§2.8) | Module 11 unimplemented; dead schema columns | per-module (Module 11 remainder) | e2e: enrollment mandatory; login requires second factor |
| 13 | Invitation-only registration; tokens with expiry and single use (§2.9) | Module 11 unimplemented; phantom invitations | auth foundation | e2e: no-token path absent; expired/used token rejected |
| 14 | Admin backoffice endpoints role-guarded (§3.3) | Module 11 unimplemented | per-module (Module 11 remainder) | Integration test: USER → 403, ADMIN → success |
| 15 | Password policy: 12+ chars + HIBP breach rejection (§2.7) | Module 11 unimplemented | per-module (Module 11 remainder) | Unit/integration tests on registration and password change |
| 16 | CI pipeline gates merges: lint → typecheck → tests → e2e → build | Zero CI | scaffold | CI configuration; red blocks merge |
| 17 | ESLint + Prettier enforced in CI and pre-commit | Zero lint / format | scaffold | CI lint step |
| 18 | Coverage thresholds enforced: 90% financial/security, 80% overall, ratchet up | Zero coverage thresholds | scaffold (config) + per-module (compliance) | CI coverage step (`docs/TESTS.md` §6) |
| 19 | Playwright installed and CI-gated from the first frontend re-port | Playwright claimed, never installed | scaffold (install) + per-module (specs) | CI e2e step executing real specs |
| 20 | react-hook-form + zod installed and used by all forms | RHF + Zod claimed, never installed | scaffold (install) + per-module (use) | Frontend form-validation tests (`docs/TESTS.md` §8) |
| 21 | DB-backed tests on testcontainers exclusively | Test infra coupled to a live developer database | scaffold | CI runs green with no external database |
| 22 | Migration SQL committed and reviewed; never gitignored (`AGENTS.md` §5.8) | Migrations directory gitignored | scaffold | Repository inspection; done-criteria enforcement |
| 23 | Container runtime for tests; deployment requirements documented | No Docker / infra-as-code | scaffold (test runtime) + first deployment | testcontainers green in CI; `docs/OPERATIONS.md` §6 |
| 24 | Renovate (security alerts immediate) + npm audit CI gate | No Renovate / Dependabot | scaffold | Renovate active; CI audit step |
| 25 | Husky + lint-staged pre-commit hooks | No git hooks | scaffold | Commit blocked on lint failure |
| 26 | No dead dependencies; password hashing owned by better-auth (§5.6) | Dead bcrypt dependency | scaffold | Dependency review at scaffold; `npm audit` |

## 8. Security Review Rules for Agents

Any reviewer (human or agent) MUST flag the following as **CRITICAL**, blocking merge:

1. **Any user-facing query without a `userId` filter** — tenant-isolation violation (`AGENTS.md` §5.7). The only exceptions are the shared global catalogs and the documented cross-tenant cron pattern (`AGENTS.md` §5.6).
2. **Any wildcard CORS configuration or origin reflection with credentials, in any environment** (`AGENTS.md` §2.1 rule 2).
3. **Any impersonation mechanism** — header, query parameter, or code path that lets a caller choose an identity — **in any environment** (`AGENTS.md` §2.1 rule 1).
4. **Any silent security stub** — a security component that is unimplemented but does not fail boot (`AGENTS.md` §2.1 rule 5).
5. **Any raw `process.env` read outside the zod env schema**, and any secret value in logs, error responses, or git (`AGENTS.md` §2.1 rule 3).

Supporting checks (major, not critical): missing service-layer guard on a security-relevant input; response serialization exposing `userId`, `encrypted*` columns, notification bodies, or unmasked recipients; upload handling outside §4.7.

When a flag conflicts with a documented acceptable pattern (`AGENTS.md` §5) or a locked ADR decision, the reviewer is wrong — follow the override process in `AGENTS.md` §6 and record the disagreement in the change's verify report.
