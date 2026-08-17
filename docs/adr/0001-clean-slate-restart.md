# ADR-0001: Clean-Slate Restart (ArcaSavings v1 → Koinomy)

Koinomy starts as a new, docs-first repository; behavior is re-ported module by module from v1's canonical specs instead of migrating v1 code.

## Status

Accepted — amendable only via the ADR process (`docs/adr/README.md`).

## Date

2026-08-07

## Context

An audit of v1 (`C:\Users\cabrerae\Desarrollo\ArcaSavings`) found problems in four categories:

**Hardening gaps.** CORS reflecting ANY origin with credentials; no security headers (helmet); no rate limiting; no environment validation (raw `process.env` reads with non-null assertions that crash at request time); a development impersonation bypass via an `X-User-Id` header active in every non-production environment; a `VaultEncryptionService` that was a throwing stub while documentation claimed otherwise; unauthenticated static serving of tenant-uploaded logos; better-auth `trustedOrigins` hardcoded to localhost; no centralized exception filter; `void bootstrap()` swallowing fatal startup errors; and PRD Module 11 (2FA, invitations, admin backoffice, Have-I-Been-Pwned password check, 15-minute session timeout) entirely unimplemented while the schema carried dead columns for it.

**Process gaps.** Zero CI (no `.github/`), zero ESLint/Prettier, zero coverage thresholds, no Docker, no Renovate/Dependabot, no git hooks — and the Prisma migrations directory was gitignored, so new migrations were silently dropped unless force-added.

**Version rot.** At decision time the stack straddled majors: NestJS 10 vs 11, Prisma 5 vs 7, React 18 vs 19, Vite 5 vs 8, Tailwind 3 vs 4, Jest 29 vs 30, Vitest 2 vs 4, TypeScript 5.6 vs 7.

**Doc drift.** The v1 architecture document (misspelled filename) described guard/state libraries never installed; `DATABASE.md` missed four models and documented a nonexistent one; the README described a shell repository; `TESTS.md` claimed Playwright/React Hook Form/Zod that were never installed.

## Decision

Restart with a clean slate:

1. Koinomy is a new repository. v1 code is **not** migrated wholesale.
2. Documentation first: ADRs, agent rules, and product/technical docs are written before any application code.
3. Behavior is re-ported module by module, guided by v1's **24 canonical OpenSpec specs** as the behavior contract.
4. v1 remains on disk as read-only reference material.

### What carries over from v1

- The 24 canonical OpenSpec specs (behavior contract for re-porting).
- PRD functional scope (11 modules).
- `DESIGN.md` design system (Binance-inspired, faithfully implemented in v1).
- The field-level encryption approach (adapter pattern).
- Tenant-isolation discipline (userId filtering on every user-domain query).
- The SDD + TDD workflow.
- better-auth as the authentication library (it worked well in v1).

### What does NOT carry over

- Drifted documentation claims (docs describing code that did not exist).
- The dead bcrypt dependency.
- The vault encryption stub.
- The development impersonation bypass.
- Wildcard CORS.
- Gitignored migrations.
- Hand-rolled UI components (replaced by shadcn/ui + Radix primitives).

## Alternatives Considered

| Option | Reason rejected |
|--------|-----------------|
| Harden v1 incrementally | The gaps are cross-cutting (boot, middleware, auth, encryption, docs); fixing them in place touches nearly every layer while carrying dead schema columns and drifted docs |
| Fork v1 and delete the worst parts | Deleting inherited defects is slower than re-porting from specs that already describe the intended behavior, and keeps the distrusted data layer alive |

## Consequences

### Positive

- No inherited foot-guns: every v1 gap must be deliberately reintroduced to exist.
- Every dependency enters at a current major, pinned and Renovate-managed from day one (ADR-0008).
- Docs are written before code, so the source-of-truth sync rule can be enforced from the first PR.

### Negative

- Short-term re-port cost: behavior that worked in v1 must be re-implemented and re-tested module by module.
- Two repositories exist in parallel; v1 must stay clearly marked as reference-only.
- The OpenSpec specs must be kept reachable as the re-port contract.

## Related Documents

- `docs/adr/README.md` — ADR process and index
- ADR-0002 (core stack), ADR-0003 (data layer), ADR-0008 (quality gates)
- v1 reference: `C:\Users\cabrerae\Desarrollo\ArcaSavings` (OpenSpec specs, PRD, DESIGN)
