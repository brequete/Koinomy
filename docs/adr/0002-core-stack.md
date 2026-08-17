# ADR-0002: Core Stack — NestJS 11, React 19, TypeScript strict, PostgreSQL 16+, npm workspaces

Keep v1's proven decoupled client/server shape and move every layer to a current major, in an npm workspaces monorepo.

## Status

Accepted — amendable only via the ADR process (`docs/adr/README.md`).

## Date

2026-08-07

## Context

v1 validated the overall shape of the system: a decoupled client/server architecture, NestJS guards and DI for the API, and an SDD+TDD workflow layered on top. But its stack was frozen at older majors while current ones were available (NestJS 10 vs 11, React 18 vs 19, Vite 5 vs 8, Tailwind 3 vs 4, TypeScript 5.6 vs 7 — see ADR-0001, "Version rot"). The clean-slate restart is the moment to align every layer on current majors without paying migration cost inside a live codebase.

The product context is unchanged: a self-hosted, decoupled web application whose PRD includes a future mobile vision (React Native / Flutter), which requires a standalone API rather than a framework-coupled fullstack runtime.

## Decision

- **Backend:** NestJS 11 with TypeScript strict mode.
- **Frontend:** React 19 with TypeScript strict mode.
- **Database:** PostgreSQL 16+.
- **Repository shape:** npm workspaces monorepo with `api/` and `client/`.
- **Frontend patterns:** react-router 7 for routing, TanStack Query for server state, react-hook-form + zod for forms, shadcn/ui + Radix primitives over Tailwind 4. v1 hand-rolled components are NOT carried over.

## Alternatives Considered

| Option | Reason rejected |
|--------|-----------------|
| Next.js fullstack | Couples client and server runtime and deployment; conflicts with self-hosted decoupled operations and with the PRD's mobile vision, which needs a standalone API |
| Express or Fastify standalone | Loses the NestJS module/DI structure that the guard-based architecture and the test-mocking strategy rely on; more assembly code for the same outcome |
| Backend in another language (Go, .NET, Java) | Gives up end-to-end TypeScript: shared strict types and shared zod schemas between `api/` and `client/`, and proven team velocity on this stack |

## Consequences

### Positive

- The SDD flow and guard-based architecture (controller boundary validation → services with defense-in-depth guards → repositories) remain valid; v1 workflow lessons transfer directly.
- Current majors from day one, kept current by Renovate (ADR-0008).
- The monorepo shares zod schemas and TypeScript types between workspaces.

### Negative

- NestJS 11 + React 19 + Tailwind 4 require care with community libraries that still peer-depend on older majors; every dependency is verified at adoption.
- Two test runners (Jest for `api/`, Vitest for `client/`) instead of one (ADR-0007).

## Related Documents

- ADR-0001 (clean-slate restart), ADR-0003 (data layer), ADR-0007 (testing), ADR-0008 (quality gates)
- `docs/DESIGN.md` (design system carried over from v1)
