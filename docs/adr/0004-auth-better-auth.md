# ADR-0004: better-auth with DB-Backed Cookie Sessions

better-auth is the authentication layer, with DB-backed cookie sessions; Module 11 features are real requirements, not dead columns.

## Status

Accepted — amendable only via the ADR process (`docs/adr/README.md`).

## Date

2026-08-07

## Context

v1 used better-auth with DB-backed cookie sessions, and it worked well — it is one of the things Koinomy deliberately carries over (ADR-0001). What failed around it was the configuration and scope discipline:

- A development impersonation bypass via an `X-User-Id` header was active in every non-production environment.
- `trustedOrigins` was hardcoded to localhost.
- PRD Module 11 (2FA, invitation-only access, admin backoffice, Have-I-Been-Pwned password check, 15-minute session timeout) was entirely unimplemented while the schema carried dead columns for it.

The auth decision therefore covers both the library choice and the hard requirements that prevent a repeat.

## Decision

Use **better-auth** with **DB-backed cookie sessions**. Plugins are added as needed (2FA, admin) rather than hand-rolling those features.

Hard requirements encoded by this ADR:

1. **No impersonation bypass in any environment.** No header, query parameter, or dev-only code path may override the authenticated identity.
2. **`trustedOrigins` comes from validated env config** (zod env schema, ADR-0008 boot validation) — never hardcoded.
3. **Session and cookie hardening:** HttpOnly, Secure, SameSite=Strict, per PRD Module 11.
4. **Auth endpoints are rate-limited** (login, registration-by-invitation, password recovery).
5. **Module 11 features are REQUIREMENTS with real implementations and tests:** mandatory 2FA, invitation-only access (no public registration), admin backoffice, password policy (12+ characters, Have-I-Been-Pwned check), 15-minute session timeout. Schema columns for these features exist only together with their working implementation — never as dead columns.

## Alternatives Considered

| Option | Reason rejected |
|--------|-----------------|
| Hand-rolled JWT auth | v1 lesson: session revocation is hard with stateless tokens; hand-rolling also rebuilds the plugin ecosystem better-auth already provides for 2FA and admin |
| Framework-coupled auth (e.g., Next.js Auth) | Incompatible with the decoupled NestJS + React architecture (ADR-0002) |
| Session library + custom 2FA | More security-critical code to own and test; better-auth's DB sessions already worked well in v1 |

## Consequences

### Positive

- Proven foundation carried over from v1; the known failures are configuration and scope problems that this ADR closes explicitly.
- DB-backed sessions make revocation and the 15-minute timeout enforceable server-side.
- Plugin path covers 2FA and admin without custom security code.

### Negative

- Dependency on better-auth's release cadence; managed by Renovate with security alerts escalated immediately (ADR-0008).
- Cookie sessions require correct CORS/origin configuration (allowlist from env) and CSRF awareness on state-changing endpoints.

## Related Documents

- ADR-0001 (clean-slate restart), ADR-0006 (tenant isolation), ADR-0008 (quality gates — env validation, rate limiting)
- `docs/PRD.md` Module 11, `docs/SECURITY.md` §2 (authentication hardening)
