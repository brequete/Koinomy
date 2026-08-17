# ADR-0003: Drizzle ORM as the Data Layer

Drizzle ORM (SQL-first, TypeScript schema, SQL-file migrations) is adopted over Prisma 7 and Kysely.

## Status

Accepted — amendable only via the ADR process (`docs/adr/README.md`).

## Date

2026-08-07

## Context

v1 ran Prisma 5. Two experiences define this decision:

1. **The migrations incident.** The Prisma migrations directory was gitignored, so new migration files were silently dropped unless force-added. A migration tooling failure mode that severe — invisible schema drift between environments — is a trust problem, not a patch problem.
2. **Magic engine, heavy client.** Prisma's generated client and query engine hide the SQL being executed. In a multi-tenant system where every query must be auditable for tenant isolation (ADR-0006), hidden SQL is a liability.

Prisma 7 was evaluated and still centers on a magic engine and a heavy generated client. Kysely was evaluated and offers excellent typed SQL, but no schema DSL and no migration generator — the schema and migration discipline would be entirely manual.

## Decision

Adopt **Drizzle ORM** for the data layer:

- TypeScript schema definitions are the source of truth for tables and columns.
- Migrations are **SQL files** produced by `drizzle-kit generate`, committed to git, reviewed like code, and applied with `drizzle-kit migrate` (or the project's deploy script). They are never gitignored.
- Queries are explicit SQL-first Drizzle queries; nothing is hidden between the service layer and PostgreSQL.
- Done-criteria for schema changes: generate + commit migrations, apply them, then typecheck (`AGENTS.md` §5.8).

## Alternatives Considered

| Option | Reason rejected |
|--------|-----------------|
| Prisma 7 | Magic engine and heavy generated client; owner distrust after the v1 gitignored-migrations incident; SQL not directly auditable |
| Kysely | Strong typed query builder but no schema DSL and no migration tooling; requires more manual discipline than the team wants to spend |
| Raw `pg` driver | Maximum control but zero schema typing and zero migration tooling; reinvents what Drizzle already provides |

## Consequences

### Positive

- Explicit SQL everywhere: tenant-isolation queries (ADR-0006) are fully auditable in code review.
- Migrations are plain SQL files in git history — reviewable, diffable, impossible to lose silently.
- No generated-client build step coupling CI to codegen; inferred types come from the schema itself.

### Negative

- More explicit SQL to write by hand (joins, aggregates) compared to Prisma's generated DX.
- Fewer batteries included (no equivalent of Prisma Studio); tooling gaps are filled deliberately as they appear.
- Type/schema consistency is enforced by the typecheck gate, so `tsc --noEmit` must stay mandatory in CI (ADR-0008).

## Related Documents

- ADR-0001 (clean-slate restart), ADR-0002 (core stack), ADR-0006 (tenant isolation)
- `AGENTS.md` §5.8 (schema-change done-criteria)
- `docs/DATABASE.md` (schema source of truth)
