# Threat Model — Koinomy (STRIDE-lite)

**Sync rule.** This document explains **why** each security requirement in `docs/SECURITY.md` exists: the assets, the threats against them, and the verification that proves each mitigation. Any PR that adds an asset, changes a mitigation cited here, or lands a security-touching ADR MUST update this document in the same PR.

## 1. Method and Scope

Practical STRIDE-lite: assets first, then one entry per threat with its STRIDE category, a concrete scenario, the mitigation (cited to its binding document), and the check that proves it. Every mitigation is checkable — a test, a review rule, or a runbook step. This is not an exhaustive academic model; it is the working list the review rules in `docs/SECURITY.md` §8 enforce.

## 2. Assets

| Asset | Value to an attacker | Where it lives |
|-------|----------------------|----------------|
| Tenant financial data | Primary target: balances, amounts, names, notes, debts, budgets — encrypted at rest (`docs/DATABASE.md` §5) | PostgreSQL user-domain tables |
| Encryption key (`ENCRYPTION_KEY`) | Decrypts every tenant's financial data at once | Environment only (`docs/OPERATIONS.md` §2) |
| Sessions | Account takeover | `Session` table + HttpOnly cookie (ADR-0004) |
| Uploaded files (entity logos) | Malware/XSS delivery vector, cross-tenant leak | Storage outside webroot, authenticated delivery (`docs/SECURITY.md` §4.7) |
| Shared catalogs (`Currency`, `ExchangeRate`) | Integrity: poisoned rates corrupt every tenant's conversions | PostgreSQL, written by the ingestion cron |
| SMTP credentials | Email-sending abuse, phishing from a trusted domain | Environment only |
| External rate API (upstream) | Availability and integrity of rate ingestion | `open.er-api.com`, fetched with timeout and validation (PRD FR-4.3) |

## 3. Threats

### TM-01 — Cross-tenant read via missing `userId` filter

- **Category:** Information Disclosure / Elevation of Privilege.
- **Scenario:** A repository method is written without the `userId` filter; authenticated user B requests or lists resources and receives user A's rows.
- **Mitigation:** Primary control — every user-facing query filters by the session-derived `userId`; repository signatures take `userId` so omission is a compile-time or review-time failure (ADR-0006 layer 1; `AGENTS.md` §5.7; ARCHITECTURE.md §6).
- **Verification:** Mandatory tenant-isolation negative test per module — user A cannot read/mutate user B's data (`docs/TESTS.md` §7); review rule flags any unfiltered user-facing query as CRITICAL (`docs/SECURITY.md` §8).

### TM-02 — Cross-tenant read via RLS misconfiguration (pooled connection identity leak)

- **Category:** Information Disclosure.
- **Scenario:** The request identity is set with a session-level `SET`, and a pooled connection returns to the pool still carrying the previous request's identity — the next request on that connection reads another tenant's rows through an otherwise-correct policy.
- **Mitigation:** Defense-in-depth layer — RLS policies evaluate `current_setting('app.current_user_id')`; the application sets `SET LOCAL app.current_user_id` **inside each transaction** (transaction-scoped, reverts on commit/rollback). Session-level `SET` is prohibited; RLS-protected work always runs inside an explicit transaction (ARCHITECTURE.md §6; ADR-0006).
- **Verification:** Integration test on a testcontainers database: two sequential requests as different users over the same pool never cross; a query with the application filter deliberately removed returns zero rows under RLS (`docs/TESTS.md` §4, §7).

### TM-03 — Tenant enumeration via error responses

- **Category:** Information Disclosure.
- **Scenario:** User B requests user A's account by id; a 403 or "belongs to another user" message confirms the resource exists, letting an attacker map tenant ids.
- **Mitigation:** Cross-tenant access returns **404 NOT_FOUND** with one indistinguishable message, identical to a genuinely missing resource; 403 is reserved for role violations (ARCHITECTURE.md §11; PRD Modules 1–8 acceptance criteria).
- **Verification:** e2e per module: cross-tenant GET/PATCH returns 404 with the same envelope as a nonexistent id (`docs/TESTS.md` §7).

### TM-04 — Session hijack / fixation

- **Category:** Spoofing.
- **Scenario:** An attacker steals or pre-plants a session cookie (XSS, network interception, fixation) and rides the victim's session.
- **Mitigation:** Cookie is HttpOnly (no script access), Secure (TLS only), SameSite=Strict; 15-minute idle timeout enforced server-side on DB sessions; sessions are revocable (deactivation, logout, password change); a new session is created at login, so pre-login tokens are not carried over (ADR-0004; `docs/SECURITY.md` §2; PRD FR-11.2).
- **Verification:** Integration tests: idle-expired session → 401; deactivated user's existing session → 401 on next request; cookie flags asserted on the login response (`docs/TESTS.md` §7).

### TM-05 — CSRF on cookie-authenticated mutations

- **Category:** Tampering / Spoofing.
- **Scenario:** A malicious site open in the same browser induces the victim's browser to POST a transfer or account mutation with the session cookie attached.
- **Mitigation:** SameSite=Strict prevents the cookie from being sent on cross-site requests (primary control); the CORS allowlist blocks cross-origin credentialed XHR/fetch responses; better-auth validates origins against `trustedOrigins` from the validated env (ADR-0004; `docs/SECURITY.md` §2.5, §4.4).
- **Verification:** Integration test: a mutation request carrying a disallowed `Origin` header is rejected; e2e covers the credentialed same-origin happy path (`docs/TESTS.md` §7).

### TM-06 — XSS via stored user input

- **Category:** Tampering (client-side elevation).
- **Scenario:** A user stores `<script>` or event-handler markup in a rendered field — category names (plaintext by design), snapshot period names, decrypted notes — and it executes in another browser context.
- **Mitigation:** React escapes by default; `dangerouslySetInnerHTML` is prohibited; the CSP baseline restricts inline script (`docs/SECURITY.md` §4.3); length and content validation at the boundary (zod) plus service guards. Most financial fields are encrypted at rest, which limits bulk exposure but does not replace output escaping after decryption.
- **Verification:** Component tests rendering hostile strings in list/detail widgets; e2e asserting no script execution from stored input (`docs/TESTS.md` §8).

### TM-07 — SQL injection

- **Category:** Tampering / Information Disclosure.
- **Scenario:** Untrusted input reaches a SQL statement and alters its structure.
- **Mitigation:** Drizzle ORM parameterizes all queries; no string-concatenated SQL in application code. Hand-authored SQL exists only in committed, reviewed migrations (partial indexes, RLS policies — `docs/DATABASE.md` §7) and never interpolates request input. **Residual risk: approximately none** — the review rule is kept anyway.
- **Verification:** Review rule: any raw SQL outside migrations is flagged; e2e with hostile input in ids/filters returns the validation envelope, never a database error (`docs/SECURITY.md` §8; `docs/TESTS.md` §7).

### TM-08 — Encryption key compromise (env leak, logs)

- **Category:** Information Disclosure (total).
- **Scenario:** `ENCRYPTION_KEY` leaks through a committed `.env`, a log line, an error response, or a backup exported with the key beside the data — decrypting every tenant's financial data.
- **Mitigation:** The key exists only in the environment, validated at boot by the zod env schema; never logged, never in error envelopes, never in git; `.env` gitignored with placeholder-only `.env.example`; backups and keys are stored and transmitted separately (`docs/SECURITY.md` §5, §6.5; `docs/OPERATIONS.md` §2, §4).
- **Verification:** Review check: `ENCRYPTION_KEY` appears only in the zod env schema and the `.env.example` placeholder; boot test fails on missing/malformed key; test assertions never contain key material in logs (`docs/TESTS.md` §5).

### TM-09 — Ciphertext tampering

- **Category:** Tampering.
- **Scenario:** An attacker with database write access edits an `encryptedAmount` value (e.g., inflating a balance) hoping the application decrypts the forged ciphertext.
- **Mitigation:** AES-256-GCM authentication tag: any modification of IV, ciphertext, or tag makes decryption fail. Storage layout `base64(IV || ciphertext || authTag)` is self-describing (ADR-0005; ARCHITECTURE.md §5).
- **Verification:** Mandatory tamper test per module: flip a byte in the stored value, assert decrypt throws and the request maps to the generic 500 envelope without leaking internals (`docs/TESTS.md` §7).

### TM-10 — Tampered or replayed uploads served to other users

- **Category:** Tampering / Information Disclosure.
- **Scenario:** An attacker uploads a malicious file (scriptable SVG, oversized payload), guesses another tenant's logo path, or replays a stored file reference across tenants.
- **Mitigation:** Type allowlist PNG/JPEG/WebP with content validation, SVG rejected; 2 MB limit enforced before write; files stored outside the webroot under random names with tenant-prefixed paths; served **only** through an authenticated endpoint that checks ownership; cross-tenant paths resolve to 404 (`docs/SECURITY.md` §4.7; PRD FR-1.1). Logos are per-user (`EntityBank.userId`), so there is no shared-file surface.
- **Verification:** e2e: SVG upload → 415, oversized → 413 with nothing written; unauthenticated logo URL → 404; another tenant's logo path → 404 (`docs/TESTS.md` §7).

### TM-11 — Rate API poisoning / upstream failure

- **Category:** Tampering (shared-catalog integrity) / Denial of Service.
- **Scenario:** The upstream returns malformed, non-numeric, or fabricated rates — or goes down — and poisoned rows propagate into every tenant's transfer conversions.
- **Mitigation:** Ingestion validates payload shape, persists only codes present in the ISO catalog, stores rates as `numeric(18,6)`, applies a 5-second timeout, and upserts idempotently per UTC day; unknown codes are skipped with a warning. Failures increment a counter; at the configured threshold one email alerts the first ADMIN; manual sync maps upstream failure to 503 (PRD FR-4.3…FR-4.6; ARCHITECTURE.md §7). Contingency: users may apply a custom rate per transfer, with the frozen rate on the row as the audit record (FR-4.7).
- **Verification:** Contract-fixture tests: malformed/poisoned payloads persist nothing; the failure threshold sends exactly one `exchange_rate_sync_failure` notification, asserted via `NotificationLog` (`docs/TESTS.md` §5, §7).

### TM-12 — Brute force on auth endpoints

- **Category:** Spoofing.
- **Scenario:** Automated credential stuffing or password guessing against login, invitation registration, or password recovery.
- **Mitigation:** Per-IP rate limiting on auth endpoints (initial: 10 requests / 15 min, `docs/SECURITY.md` §6.3); password policy (12+ chars, HIBP breach rejection) raises guess cost; invitation-only registration removes the public signup surface entirely (PRD FR-11.4…FR-11.6).
- **Verification:** Integration test: exceeding the limit returns 429 `RATE_LIMITED` for the rest of the window; HIBP-rejected passwords return 400 with a clear message (`docs/TESTS.md` §7).

### TM-13 — Invitation token guess or leak

- **Category:** Spoofing / Elevation of Privilege.
- **Scenario:** An attacker guesses an invitation token, or an invitation email is forwarded/intercepted and redeemed by someone other than the invitee.
- **Mitigation:** Tokens are cryptographically random (generated by better-auth's invitation capabilities — adopted, not hand-rolled, ADR-0004), single-use, and expiry-bearing; registration is rate-limited per IP; the token is marked used atomically on completion, and expired/used tokens are rejected (PRD FR-11.4 acceptance criteria; `docs/SECURITY.md` §2.9). Delivery rides SMTP over the provider's transport; SMTP credentials stay in the environment.
- **Verification:** e2e: registration without a token has no path; expired token rejected; double-redemption rejected (second attempt fails); rate limit bounds online guessing (`docs/TESTS.md` §7).

### TM-14 — Insider / admin abuse

- **Category:** Elevation of Privilege.
- **Scenario:** An ADMIN account (compromised or malicious) deactivates users, issues invitations to outsiders, or browses tenant data through backoffice endpoints.
- **Mitigation:** Backoffice endpoints are role-guarded (USER → 403, `docs/SECURITY.md` §3.3); admin mutations are auditable through the structured request log (method, path, status, `userId` — ARCHITECTURE.md §3 stage 11, §10); user-domain data access still requires the normal session-derived `userId` path — the backoffice manages users and invitations, it does not expose tenant financial data. A dedicated audit table is not in the current schema; adding one requires `docs/DATABASE.md` first (§1 rule 1 of that document).
- **Verification:** Integration test: USER-role session rejected on every backoffice endpoint; review check: every admin mutation route appears in the request-log assertions; log entries carry the acting `userId` (`docs/TESTS.md` §7).

### TM-15 — Dependency supply chain

- **Category:** Tampering (introduced upstream).
- **Scenario:** A compromised or vulnerable dependency (better-auth, NestJS, Drizzle, zod, any frontend package) ships malicious or flawed code into the app.
- **Mitigation:** Renovate keeps dependencies current — weekly grouped minors/patches, **security alerts handled immediately**; `npm audit` gates CI (high/critical blocks merge); lockfile committed; versions pinned at repo birth and recorded (ADR-0008). No dead dependencies are carried (v1's dead bcrypt).
- **Verification:** CI audit step red on high/critical findings; Renovate activity visible in PR history; lockfile integrity checked by CI install (`docs/SECURITY.md` §6.4).

### TM-16 — Backup exposure

- **Category:** Information Disclosure.
- **Scenario:** A database backup is stolen or misplaced; the attacker restores it and reads tenant data.
- **Mitigation:** Backups contain **ciphertext** for every protected field (`docs/DATABASE.md` §5) — useless without the key. The key is stored and transmitted separately from backups (key custody, `docs/OPERATIONS.md` §4, §8). Restore drills verify both halves independently.
- **Verification:** Restore drill per `docs/OPERATIONS.md` §4: restore succeeds only when backup and key are reunited deliberately; drill repeated on schedule.

## 4. Review Cadence

This threat model is revisited:

1. **At each new module re-port** — the module's SDD change checks which TM entries apply and confirms their verification landed (recorded in the change's verify report).
2. **On any ADR that touches security** (auth, encryption, isolation, quality gates, or a new one) — the affected entries are updated in the same PR, per the sync rule.
3. **After any security incident or near-miss** — a new entry is added if the existing list did not anticipate it.
4. **At least the checklist in `docs/SECURITY.md` §7 is walked** whenever the hardening baseline changes.

Threats not yet modeled are not assumed absent — they are added here when identified. The review rules in `docs/SECURITY.md` §8 apply regardless of whether a threat entry exists.
