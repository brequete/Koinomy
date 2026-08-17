# QA Standards — Koinomy

**Sync rule.** This document operationalizes ADR-0007 (testing strategy) and ADR-0008 (quality gates). Any PR that changes a test convention, threshold, runner, or gate documented here MUST update this document in the same PR. Thresholds ratchet up only — lowering a threshold requires a new ADR.

## 1. Scope

Koinomy computes budgets, balances, amortizations, and exchange operations: a silent error is a data-corruption bug, and a leak is the worst failure class of a multi-tenant financial app. Everything here serves two goals: financial correctness is proven **before** the code exists (TDD), and isolation/encryption are proven **on every merge** (enforced gates).

## 2. TDD Discipline

**Red → Green → Refactor is mandatory for all financial and security logic** (`AGENTS.md` §4 Phase 2):

1. **Red:** tests are written first from the approved SDD and MUST fail — the failure is evidenced in the SDD flow (recorded in the change's verify report).
2. **Green:** the minimum implementation to pass.
3. **Refactor:** debt cleanup and typing improvements under green tests.

Financial correctness is **never tested after the fact**: a module whose tests were written after its implementation fails the workflow, not just the gate. The same applies to security logic (isolation, encryption, auth, validation).

## 3. Stack and Layout

| Workspace | Runner | Layout |
|-----------|--------|--------|
| `api/` | Jest | Unit tests: `*.spec.ts` next to the code under test. Integration tests: `*.integration.spec.ts` next to the code. API e2e: `test/` directory via Supertest against the NestJS application (ADR-0007) |
| `client/` | Vitest + Testing Library | Hooks, components, and pages tested at `client/src` next to the code |
| `client/` e2e | Playwright | Installed and **gated in CI from the first frontend re-port**. v1 claimed Playwright and never installed it; Koinomy states only what exists — if no Playwright spec exists yet, the claim does not either |

## 4. Database-Backed Tests

- **testcontainers PostgreSQL 16+ exclusively.** Integration and e2e tests get a real, disposable PostgreSQL instance; the committed migrations are applied to the container, so the test schema is always the real schema (ADR-0007, pairs with ADR-0003 SQL migrations).
- **Never a developer's local database.** No shared schema, no manually seeded database, no `DATABASE_URL_TEST` pointing at a machine-local instance. v1 coupled its suite to a live local schema and results depended on developer-machine state; that failure mode is closed (v1's drift-guard snapshots of dev-DB row counts are obsolete — there is nothing left to drift from).
- **Deterministic test key:** tests use `TEST_ENCRYPTION_KEY` (`docs/OPERATIONS.md` §2.2) — never a real key.

## 5. Mocking Rules

| Dependency | Rule |
|------------|------|
| `EncryptionService` | Mocked at the DI boundary in **unit** tests, so encryption behavior is tested deliberately, not incidentally (ADR-0005, ADR-0007). **At least one integration test per module exercises the REAL encryption round-trip** against the real `LocalEncryptionService` |
| External HTTP (exchange-rate upstream) | Mocked with contract fixtures — recorded payload shapes for success, malformed, and failure cases (PRD FR-4.3 acceptance criteria) |
| SMTP | Mocked; sends are asserted via `NotificationLog` rows (SENT/FAILED), never via the transport directly (PRD FR-10.2) |
| Time | Frozen/injected clock for crons, due dates, month-end clamping, and the 15-minute session timeout |

## 6. Coverage

- **Enforced in CI, not dashboards:** thresholds are gate failures, with coverage reports published as CI artifacts.
- **Thresholds (line coverage):** **90%** for `api/` financial and security modules (transactions, budgets, exchange, encryption, tenant isolation, auth); **80%** overall.
- **Ratchet:** thresholds only go up. A PR that lowers a threshold requires a new ADR (ADR-0007, ADR-0008).
- **The v1 correction:** the v1 PRD claimed "100% mandatory coverage" and enforced nothing — no CI existed to enforce it with. Koinomy states only enforceable truth: real numbers, real gates.

## 7. Required Test Classes per Re-Ported Module

Every module re-port lands all applicable classes before its verify report can close:

1. **Happy path** — the module's acceptance criteria from `docs/PRD.md` (mined from the v1 OpenSpec specs).
2. **Validation failures** — zod boundary rejections **and** service-layer guard rejections (defense in depth, `AGENTS.md` §5.1), including direct service calls that bypass the controller.
3. **Tenant-isolation negative test** — user A cannot read/mutate user B's data, returning 404 indistinguishable from a missing resource. **Mandatory for EVERY module touching user-domain tables** (`docs/SECURITY.md` §3; THREAT-MODEL TM-01/TM-03).
4. **Encryption round-trip + tamper detection** — real-adapter round-trip (integration), and a tampered ciphertext (flipped auth-tag byte) fails decryption (THREAT-MODEL TM-09).
5. **Money precision** — Decimal(18,6) semantics: no float drift; installment remainder absorption sums exactly to the total; aggregations use exact decimal arithmetic (PRD Modules 3, 5 acceptance criteria).
6. **Concurrency/atomicity** — where the module multi-writes (`db.transaction`): both-or-neither commit assertions (e.g., transaction insert + balance update; snapshot row + N balance rows).
7. **RLS backstop** — at least one integration test per module proving the policy holds when the application filter is absent (THREAT-MODEL TM-02).

## 8. Frontend Test Requirements

Per re-ported frontend area:

1. **Form validation** — react-hook-form + zod resolver: invalid input (negative amounts, empty mandatory fields, malformed decimal strings) blocks submission with field-level messages.
2. **Error-envelope rendering** — API errors surface through the typed client wrapper as user-visible messages; no silent failures, no raw payloads (`docs/ARCHITECTURE.md` §9, §11).
3. **401 session-expired flow** — a 401 redirects to login with the return path preserved.
4. **Widget states** — loading (skeleton), empty, and error states for every widget/list; one failing widget never blocks the rest (PRD FR-9.4).
5. **Stored-input rendering** — hostile strings in rendered fields (category names, period names, notes) render inert (THREAT-MODEL TM-06).

## 9. CI Gate Order

Mirrors ADR-0008 — every merge passes, in order:

1. **Lint** — ESLint (flat config) + Prettier check.
2. **Typecheck** — `tsc --noEmit` on both workspaces.
3. **Unit + integration tests** — with coverage thresholds enforced (§6).
4. **e2e** — Supertest API e2e (testcontainers) and Playwright frontend e2e.
5. **Build** — both workspaces.

**CI red blocks merge. No manual overrides, no merging on red** (ADR-0008). Husky + lint-staged gate commits locally before CI ever sees them.
