# ADR-0008: Dependency Hygiene and Quality Gates

Renovate keeps dependencies current; ESLint, Prettier, Husky, and a staged GitHub Actions pipeline gate every merge. CI red blocks merge with no manual overrides.

## Status

Accepted — amendable only via the ADR process (`docs/adr/README.md`).

## Date

2026-08-07

## Context

v1 had zero process infrastructure: no CI, no linter, no formatter, no git hooks, no dependency-update mechanism, and no env validation (ADR-0001). The consequences were concrete — version rot across every major, gitignored migrations merged silently, and raw `process.env` reads crashing at request time. Koinomy installs the full gate system at repo birth so that no code ever enters the repository without passing it.

## Decision

- **Dependency updates — Renovate:**
  - Weekly schedule.
  - Minor and patch updates grouped into single PRs.
  - Security alerts handled immediately, outside the weekly cadence.
- **Static checks:** ESLint (flat config) + Prettier, run in CI and pre-commit.
- **Git hooks:** Husky + lint-staged — staged files are linted and formatted before commit.
- **Environment validation:** all environment variables are parsed at boot with a zod schema; invalid or missing config fails boot with a clear message (`AGENTS.md` §2.1).
- **CI pipeline (GitHub Actions), in order:**
  1. Lint (ESLint + Prettier check)
  2. Typecheck (`tsc --noEmit`, both workspaces)
  3. Unit + integration tests with coverage thresholds enforced (ADR-0007)
  4. e2e tests (Supertest API e2e, Playwright frontend e2e)
  5. Build
- **Merge rule:** CI red blocks merge. No manual overrides, no merging on red.

### Versions pinned at repo birth

| Component | Major line |
|-----------|-----------|
| NestJS | 11.x |
| React | 19.x |
| TypeScript | 5.x strict (the TS 7 native port is not adopted at birth; revisit via ADR) |
| PostgreSQL | 16+ |
| Vite | 8.x |
| Tailwind CSS | 4.x |
| Jest | 30.x |
| Vitest | 4.x |
| Drizzle ORM / drizzle-kit | current stable at scaffolding; exact version recorded in the lockfile |
| better-auth / zod | current stable at scaffolding; exact versions recorded in the lockfile |

Exact versions are recorded in `package.json` and the lockfile when the monorepo is scaffolded; Renovate keeps them moving from that baseline.

## Alternatives Considered

| Option | Reason rejected |
|--------|-----------------|
| Dependabot | Renovate's grouping and scheduling controls fit the weekly-grouped-minors policy better |
| Manual dependency reviews only | v1 proved the discipline fails under time pressure; version rot was the result |
| Quality gates as optional checks | v1 had no gates at all; advisory checks are ignored under deadline pressure |

## Consequences

### Positive

- Dependencies stay on supported majors automatically; security patches land immediately.
- Every merge passes lint, typecheck, tests with coverage, e2e, and build — the v1 gap categories are structurally closed.
- Boot-time env validation turns configuration errors into startup failures with clear messages instead of request-time crashes.

### Negative

- Renovate PR volume requires the grouped-minors/patches policy to stay manageable.
- The full pipeline adds CI minutes per PR; acceptable given the gating guarantee.
- Threshold and gate maintenance is ongoing work; changes to gates require an ADR.

## Related Documents

- ADR-0001 (process gaps that motivated this), ADR-0002 (pinned stack), ADR-0007 (coverage thresholds), ADR-0003 (migration files committed and gated)
- `AGENTS.md` §2.1 (red lines), §5.8 (schema-change done-criteria)
