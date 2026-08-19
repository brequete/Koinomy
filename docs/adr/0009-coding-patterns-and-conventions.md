# ADR-0009: Coding Patterns & Conventions

A positive catalog of patterns in `docs/PATTERNS.md` replaces the AGENTS.md §2 flat checklist; every PR is reviewed against the catalog and intentional deviations live in §5.

## Status

Accepted — amendable only via the ADR process (`docs/adr/README.md`).

## Date

2026-08-19

## Context

`AGENTS.md` §2 currently lists seven items (Clean Code, Clean Architecture, SOLID, DRY, KISS, ES6+ Standards, Type Safety) as a flat checklist with no definitions, no anti-examples, and no cross-references to the ADRs that lock the architectural decisions behind them. §5 documents only acceptable deviations, not the positive patterns they deviate from. The team needs a discoverable, opinionated catalog so PR reviews converge on the same notion of "good code" without re-litigating fundamentals on every change.

This is a documentation gap, not a process gap: the rules are enforced informally in review, but a new contributor cannot find them in one place, and an auditor cannot verify they exist.

## Decision

- Adopt `docs/PATTERNS.md` as the canonical catalog of coding patterns and conventions for Koinomy.
- `AGENTS.md` §2 is reduced to a single reference to this ADR and the catalog file. The seven-item checklist is removed.
- Every PR is reviewed against `docs/PATTERNS.md`.
- New patterns are added via a new ADR or by amending this one. Changes to existing patterns require an ADR.
- `AGENTS.md` §5 (Acceptable Patterns) remains intact and authoritative; it is the only place for intentional deviations from `docs/PATTERNS.md`.
- Patterns are grouped by area (Principles, Architecture, Backend, Frontend, Testing, TypeScript, Anti-patterns) and cross-reference the ADRs that bind them.
- Each pattern follows the four-part structure: Definition → When it applies → Example ✅ / Anti-example ❌ → ADR reference when locked. The Principles section uses a one-line table; the rest use the full structure.

## Alternatives Considered

| Option | Reason rejected |
|--------|-----------------|
| Keep the flat list in `AGENTS.md` §2 and add prose around it | No definitions, no anti-examples, no ADR cross-references; reviewers re-derive intent every time and the checklist rots silently |
| Inline patterns inside each ADR | Fragments the catalog across eight (soon more) documents; defeats the purpose of a single discoverable reference and makes cross-cutting patterns (e.g., TypeScript, Testing) impossible to find |
| Merge §5 (Acceptable Patterns) into `docs/PATTERNS.md` | The two documents serve distinct purposes — `PATTERNS.md` is the positive catalog, §5 is the registry of intentional deviations. Merging them blurs the boundary between "what good code looks like" and "where we deliberately break the rules" |

## Consequences

### Positive

- Explicit, discoverable conventions reduce ambiguity in PRs and shorten review cycles.
- Onboarding accelerates: a new contributor finds one document that defines "good code" for this codebase.
- Cross-references lock the relationship between patterns and the ADRs that bind them (e.g., Repository pattern → ADR-0006; Adapter pattern → ADR-0005).
- §5 stays the single source of truth for intentional deviations — no duplication, no contradiction.

### Negative

- Documentation maintenance burden: every pattern change now requires an ADR.
- Two artifacts (`AGENTS.md` §5 and `docs/PATTERNS.md`) must stay consistent — a pattern added to `PATTERNS.md` that contradicts §5 is a defect.
- A new ADR cycle is required to add or modify a pattern, which is heavier than a docs PR. This is intentional: patterns are binding, not advisory.
- `AGENTS.md` §5 (Acceptable Patterns) is not merged into `docs/PATTERNS.md`; the two documents serve distinct purposes (positive catalog vs. documented exceptions), and reviewers must consult both.

## Related Documents

- ADR-0002 (core stack — the patterns in §3, §4, §6 lock to this), ADR-0003 (data layer — repository pattern, §2), ADR-0005 (adapter pattern — §2), ADR-0006 (tenant isolation — §3, §7), ADR-0007 (testing strategy — §5), ADR-0008 (quality gates — gates that enforce the catalog)
- `AGENTS.md` §2 (this ADR replaces the §2 checklist), §5 (Acceptable Patterns — the registry of intentional deviations)
- `docs/ARCHITECTURE.md` (layering and request lifecycle referenced by §2, §3)
- `docs/TESTS.md` (testing standards referenced by §5)
- `docs/PATTERNS.md` (the catalog this ADR adopts)
