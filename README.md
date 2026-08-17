# Koinomy

Personal Financial Management System (SGFP): a private, self-hosted, multi-tenant web application with invite-only access (no public registration). It provides strict budget control, multi-currency transaction tracking, exchange operations, and debt and savings management.

This repository is in its **documentation-first foundation phase**. Architecture decisions, agent rules, and process documentation are written first. There is **no application code in this repository yet**.

## Status

| Aspect | State |
|--------|-------|
| Application code | None — foundation phase |
| Foundation documents (`README.md`, `AGENTS.md`, `docs/adr/` ADR-0001 through ADR-0008) | Present |
| Product and technical docs (PRD, ARCHITECTURE, DATABASE, SECURITY, THREAT-MODEL, OPERATIONS, TESTS, DESIGN) | Present |
| Monorepo scaffolding (`api/`, `client/`) | Not started |

**Sync rule:** this README must reflect the actual state of the repository. Any PR that changes the state of the repository — adding code, scaffolding, or documents — must update this README in the same PR. Silent drift is a defect.

## Documentation map

| Document | Purpose | Status |
|----------|---------|--------|
| `README.md` | Repository state, product overview, documentation index | Present |
| `AGENTS.md` | Orchestrator and sub-agent rules: unbreakable rules, SDD+TDD workflow, acceptable patterns | Present |
| `docs/adr/README.md` | ADR process, template, and index | Present |
| `docs/adr/0001-…` to `0008-…` | Locked architecture decisions: clean slate, core stack, data layer, auth, encryption, tenant isolation, testing, quality gates | Present |
| `docs/PRD.md` | Functional scope: 11 modules, MoSCoW prioritization, acceptance criteria | Present |
| `docs/ARCHITECTURE.md` | Repository structure and design patterns (self-hosted) | Present |
| `docs/DATABASE.md` | Relational schema and encrypted-field catalog | Present |
| `docs/SECURITY.md` | Hardening baseline and security requirements | Present |
| `docs/THREAT-MODEL.md` | Threat model: assets, STRIDE threats, mitigations, verification | Present |
| `docs/DESIGN.md` | Design system: palette, typography, base components | Present |
| `docs/TESTS.md` | QA standards: TDD, coverage, e2e | Present |
| `docs/OPERATIONS.md` | Operational runbooks: env contract, key rotation, deployment, backups | Present |

## Architecture decisions

All stack, security, and process decisions are recorded as Architecture Decision Records. Start at the index: [`docs/adr/README.md`](docs/adr/README.md).

| Decision | ADR |
|----------|-----|
| Clean-slate restart, docs-first | [ADR-0001](docs/adr/0001-clean-slate-restart.md) |
| NestJS 11 + React 19 + TypeScript strict + PostgreSQL 16+ | [ADR-0002](docs/adr/0002-core-stack.md) |
| Drizzle ORM data layer | [ADR-0003](docs/adr/0003-drizzle-data-layer.md) |
| better-auth with DB-backed cookie sessions | [ADR-0004](docs/adr/0004-auth-better-auth.md) |
| AES-256-GCM field-level encryption (adapter pattern) | [ADR-0005](docs/adr/0005-field-level-encryption.md) |
| userId filtering + Row-Level Security isolation | [ADR-0006](docs/adr/0006-multi-tenant-isolation.md) |
| TDD with enforced coverage thresholds | [ADR-0007](docs/adr/0007-testing-strategy.md) |
| Renovate + CI quality gates | [ADR-0008](docs/adr/0008-dependency-hygiene-and-quality-gates.md) |

## Relationship to v1

v1 lives at `C:\Users\cabrerae\Desarrollo\ArcaSavings` and is **reference material only**. Its code is not migrated wholesale; behavior is re-ported module by module, guided by v1's 24 canonical OpenSpec specs. See [ADR-0001](docs/adr/0001-clean-slate-restart.md) for what carries over and what does not.
