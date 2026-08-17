# ADR-0007: Testing Strategy — TDD, Jest + Vitest, testcontainers, Enforced Coverage

TDD is mandatory for financial and security logic; DB-backed tests run on testcontainers; coverage thresholds are enforced in CI.

## Status

Accepted — amendable only via the ADR process (`docs/adr/README.md`).

## Date

2026-08-07

## Context

The PRD makes TDD a non-functional requirement and demands full coverage of financial logic: this system computes budgets, balances, amortizations, and exchange operations, where a silent error is a data-corruption bug. v1's testing setup had a structural flaw this ADR closes: tests were coupled to a live local database schema, so results depended on developer-machine state and CI had nothing reproducible to run (v1 had no CI at all — ADR-0001).

## Decision

- **TDD is mandatory** for all financial and security logic: tests are written first and must fail (Red) before implementation (Green), then Refactor (`AGENTS.md` §4).
- **Runners:** Jest for `api/`, Vitest for `client/` (matches each workspace's toolchain, ADR-0002).
- **Database-backed tests use testcontainers.** Integration and e2e tests get a real, disposable PostgreSQL instance. **No test may depend on developer-machine state** — no shared local schema, no manually seeded database.
- **Coverage thresholds are enforced in CI** (initial values, to be ratcheted up over time, never down):
  - **90% line coverage** for financial and security modules in `api/src` (transactions, budgets, exchange, encryption, tenant isolation, auth).
  - **80% line coverage** overall.
- **API e2e:** Supertest against the NestJS application with a testcontainers database.
- **Frontend e2e:** Playwright.
- **Mocking:** backend unit tests mock dependency injection — especially the `EncryptionService` (ADR-0005) — so encryption behavior is tested deliberately, not incidentally.

## Alternatives Considered

| Option | Reason rejected |
|--------|-----------------|
| Tests against a shared local/dev database (v1 approach) | Results depend on developer-machine state; flaky, unreproducible, and impossible to gate in CI |
| A single runner for both workspaces | Forces Jest onto the Vite/React 19 client or Vitest onto NestJS against each toolchain's grain; the mismatch cost outweighs the consistency benefit |
| Coverage as a dashboard metric only | v1 had zero thresholds and coverage silently rotted; enforcement in CI is the only rule that holds |

## Consequences

### Positive

- Reproducible, isolated test runs in CI and locally; every PR is gateable.
- Financial logic carries a quantified, enforced coverage floor.
- testcontainers make schema migrations testable end-to-end (pairs with ADR-0003's SQL migrations).

### Negative

- testcontainers requires a container runtime on every machine and in CI; setup is a one-time infrastructure cost.
- Two runners mean two configurations to maintain.
- Thresholds will occasionally block PRs — by design; the ratchet only moves up.

## Related Documents

- ADR-0002 (core stack), ADR-0003 (data layer), ADR-0005 (encryption mocking), ADR-0008 (CI gating)
- `AGENTS.md` §4 (SDD + TDD workflow)
- `docs/TESTS.md` (QA standards)
