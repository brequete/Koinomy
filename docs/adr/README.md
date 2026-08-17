# Architecture Decision Records (ADR)

Every decision that shapes the system lives here. Each record captures one decision with its context, alternatives, and consequences, so future readers can verify intent without reconstructing the story.

## Quick path

1. Find the decision you care about in the index below.
2. Open the ADR file for full context and consequences.
3. To change an accepted decision, write a new ADR that supersedes it. Never edit history silently.

## What is an ADR

An ADR is a short document that records a decision that is expensive or dangerous to reverse: what was decided, why, what else was considered, and what we accept as a consequence. Accepted ADRs are binding for all code and documentation in this repository, and they are amendable only through this process.

## When an ADR is required

| Area | Examples |
|------|----------|
| Stack | Frameworks, languages, runtimes, major version jumps |
| Data layer | ORM, migration strategy, schema ownership |
| Authentication | Session model, identity library, 2FA approach |
| Security model | Encryption, tenant isolation, headers, rate limiting |
| Cross-cutting patterns | Validation language, error handling, testing strategy |
| Process and tooling | CI gates, dependency updates, hooks, coverage policy |

Small, local implementation choices do not need an ADR; record those in the change's design document (SDD flow, see `AGENTS.md` §4).

## Status lifecycle

```text
Proposed --> Accepted --> Superseded
                      \-> Deprecated
```

| Status | Meaning |
|--------|---------|
| Proposed | Under discussion; not binding yet |
| Accepted | Locked and binding; amendable only via a new ADR |
| Superseded | Replaced by a newer ADR (the replacement must be linked) |
| Deprecated | No longer applies but not replaced (the reason must be stated) |

## Numbering convention

- File name: `NNNN-kebab-case-title.md` (zero-padded, four-digit, sequential).
- Numbers are never reused and never reordered.
- The title states the decision, not the topic (good: `0003-drizzle-data-layer.md`; weak: `0003-database.md`).

## Template

Copy this block for every new ADR:

```markdown
# ADR-NNNN: <Decision title>

<One-line summary: the decision and its headline consequence.>

## Status

<Proposed | Accepted | Superseded by ADR-XXXX | Deprecated>

## Date

<YYYY-MM-DD>

## Context

<The problem, constraints, and evidence that force a decision.>

## Decision

<The decision, stated with active verbs. What we do, and what we deliberately do not do.>

## Alternatives Considered

| Option | Reason rejected |
|--------|-----------------|
| <option> | <reason> |

## Consequences

### Positive

- <what this makes easier or safer>

### Negative

- <what this makes harder or costlier>

## Related Documents

- <links to ADRs, docs, specs>
```

## Index

| ADR | Title | Status | Summary |
|-----|-------|--------|---------|
| [0001](0001-clean-slate-restart.md) | Clean-Slate Restart | Accepted | New docs-first repository; behavior re-ported module by module from v1's 24 canonical OpenSpec specs |
| [0002](0002-core-stack.md) | Core Stack | Accepted | NestJS 11 + React 19 + TypeScript strict + PostgreSQL 16+ in an npm workspaces monorepo |
| [0003](0003-drizzle-data-layer.md) | Drizzle Data Layer | Accepted | Drizzle ORM (SQL-first, SQL-file migrations) chosen over Prisma 7 and Kysely |
| [0004](0004-auth-better-auth.md) | better-auth Authentication | Accepted | better-auth with DB-backed cookie sessions; Module 11 features are real requirements |
| [0005](0005-field-level-encryption.md) | Field-Level Encryption | Accepted | AES-256-GCM adapter pattern with a fail-fast provider contract |
| [0006](0006-multi-tenant-isolation.md) | Multi-Tenant Isolation | Accepted | userId filter in every user-domain query, plus PostgreSQL Row-Level Security as defense-in-depth |
| [0007](0007-testing-strategy.md) | Testing Strategy | Accepted | TDD, Jest + Vitest, testcontainers, coverage thresholds enforced in CI |
| [0008](0008-dependency-hygiene-and-quality-gates.md) | Dependency Hygiene and Quality Gates | Accepted | Renovate, ESLint + Prettier, Husky, GitHub Actions CI gating merge |
