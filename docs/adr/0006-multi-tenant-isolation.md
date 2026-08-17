# ADR-0006: Multi-Tenant Isolation — userId Filtering + Row-Level Security

Every user-domain query is filtered by the requester's userId as the primary control, with PostgreSQL Row-Level Security as defense-in-depth.

## Status

Accepted — amendable only via the ADR process (`docs/adr/README.md`).

## Date

2026-08-07

## Context

Koinomy is multi-tenant (invite-only, no public registration). A data leak between tenants is the single worst failure class for this product. v1 enforced isolation through discipline — every user-domain query filtered by the requester's `userId` — and that discipline is carried over. What v1 lacked was a second layer: if a query slipped through without the filter, nothing in the database stopped it.

## Decision

Two layers, both mandatory:

1. **Primary control — application-level filtering.** Every backend query that reads or writes tenant-owned/user-domain data is filtered by the requester's `userId`, taken from the authenticated session (never from client input). This is enforced by service- and repository-layer discipline and by code review: **any user-facing query without a `userId` filter is a critical defect.**
2. **Defense-in-depth — PostgreSQL Row-Level Security (new in Koinomy).** RLS policies on tenant tables compare the row's `userId` against the requester identity. The application sets the request-scoped identity (e.g., `SET LOCAL app.current_user_id = ...` within the request's transaction) and policies evaluate `current_setting('app.current_user_id')`. A query that forgets the application filter still cannot cross tenants.

**Documented exception pattern.** The only queries that may legitimately span tenants are **system-level scheduled tasks** (e.g., a daily cron finding due rules for notification). Such a repository method must carry an inline comment explaining the cross-tenant exception and the design decision, and the spec must explicitly carve out the exception (`AGENTS.md` §5.6).

**Shared catalogs.** Global system catalogs (e.g., `Currency`, `ExchangeRate`) are shared reference data and intentionally have no `userId`.

## Alternatives Considered

| Option | Reason rejected |
|--------|-----------------|
| Application filtering only (v1 approach) | A single forgotten filter leaks data; no second layer exists to catch it |
| RLS only | Couples security entirely to database configuration; application code becomes unauditable for isolation, and connection-pooling/session-state pitfalls go unnoticed |
| Separate schema or database per tenant | Operational complexity far beyond the product's self-hosted scale; migration and backup cost multiply |

## Consequences

### Positive

- A forgotten `userId` filter is caught by RLS instead of becoming a leak.
- Isolation is auditable in two places: code review (queries) and schema review (policies).
- The exception pattern keeps cross-tenant crons possible without weakening the default rule.

### Negative

- RLS policies and the per-request identity setting add schema and middleware surface that must be tested (ADR-0007) and documented.
- Connection pooling requires care: the request-scoped setting must be reset between requests (handled via transaction-scoped `SET LOCAL`).
- Every new tenant table needs both the application discipline and an RLS policy; a checklist belongs in `docs/DATABASE.md`.

## Related Documents

- ADR-0003 (data layer — explicit SQL makes isolation auditable), ADR-0004 (session identity), ADR-0007 (testing)
- `AGENTS.md` §2 (zero tolerance), §5.6 (cross-tenant cron exception), §5.7 (summary)
- `docs/DATABASE.md` §2 (ownership & isolation model)
