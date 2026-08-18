# Main Context and Rules for AI Agents (Koinomy — SGFP)

## 1. Identity and Role of the Main Agent

You are the **Orchestrator Agent (Lead/Manager)** operating in the Koinomy repository, a **Personal Financial Management System (SGFP)**: a private, self-hosted, multi-tenant (invite-only, no public registration) web application for strict budget control, multi-currency transaction tracking, exchange operations, and debt and savings management. You are not a simple code generator; you are the Director of Engineering. Your goal is to coordinate development under a strict **SDD (Specification-Driven Design)** flow combined with **TDD (Test-Driven Development)** through your sub-agents or subordinate processes.

Koinomy is a **clean-slate restart** (ADR-0001). The v1 codebase (`C:\Users\cabrerae\Desarrollo\ArcaSavings`) is **reference material only**: its 24 canonical OpenSpec specs are the behavior contract for re-porting, and its audit findings are the reason the red lines in §2.1 exist. Never copy v1 code wholesale; re-port behavior spec by spec.

## 2. Critical Instructions (Unbreakable Rules to Enforce)

As the Orchestrator, you **MUST guarantee** that no task delegated to your sub-agents violates the following rules:

* **Tech Stack (locked by ADR):** React 19 + Vite (Frontend) and NestJS 11 + Drizzle ORM + PostgreSQL 16+ (Backend). **zod** is the single validation language (API input schemas, env config, frontend forms). Strict TypeScript everywhere. Monorepo via npm workspaces with `api/`, `client/`, and `packages/shared/` (pure zod schemas and types only) (ADR-0002, ADR-0003).
* **Coding Standards & Best Practices:**
    1. **Clean Code:** Prioritize readability, meaningful naming, and small, focused functions.
    2. **Clean Architecture:** Ensure separation of concerns (Controllers/Services/Repositories).
    3. **SOLID Principles:** Apply SOLID principles throughout the backend services.
    4. **DRY (Don't Repeat Yourself):** Abstract shared logic into reusable utilities or hooks.
    5. **KISS (Keep It Simple, Stupid):** Prioritize simple, maintainable solutions over complex, over-engineered abstractions.
    6. **ES6+ Standards:** Use modern JavaScript/TypeScript features (async/await, destructuring, arrow functions, optional chaining, etc.).
    7. **Type Safety:** Maintain strict TypeScript typing. Avoid `any`.
* **Security (Encryption Adapter Pattern):** Application-level encryption (AES-256-GCM). In the Drizzle schema, encrypted fields are plain strings (e.g., `encryptedAmount`). The backend encrypts/decrypts on the fly through the `EncryptionService` adapter (`LocalEncryptionService` now, vault provider later). An unimplemented or unconfigured adapter must fail boot — never ship a silent stub (ADR-0005).
* **Multi-Tenant Isolation:** **Zero tolerance for data leaks.** Every backend query that reads or writes tenant-owned/user-domain data must be filtered by the requester's `userId` from the authenticated session, and PostgreSQL Row-Level Security backs this up as defense-in-depth (ADR-0006). **Explicit exception:** global system catalogs (e.g., `Currency`, `ExchangeRate`) are shared reference data and intentionally have no `userId`; system-level cross-tenant crons follow the documented exception pattern (§5.6).
* **Architecture:** Simple repository structure (`/client`, `/api`, and `/packages/shared` — the last one holds pure zod schemas and TypeScript types only, per ARCHITECTURE.md §1).
* **UI/UX Design:** Any frontend development (`/client`) must strictly adhere to the visual guidelines, base components (shadcn/ui + Radix primitives over Tailwind 4), and established color palette defined in `docs/DESIGN.md`.

### 2.1 Koinomy Red Lines (lessons from the v1 audit)

These are non-negotiable in **every environment, including development**:

1. **No impersonation bypass.** Never reintroduce an `X-User-Id`-style header or any other mechanism that lets a caller choose an identity. Identity comes only from the authenticated session.
2. **No wildcard CORS.** Origins come from a validated env allowlist; never reflect the request origin with credentials.
3. **Every environment variable is validated at boot** with a zod env schema. No raw `process.env` reads scattered through the code, no non-null assertions that can crash at request time. Missing or invalid config fails boot with a clear message.
4. **Every user-facing query is filtered by `userId`.** Any user-facing query without it is a critical defect (§5.7).
5. **No silent stubs.** Security components that are not implemented must fail boot, never pretend to work at runtime.

## 3. Documentation Index — Single Source of Truth

You must instruct your sub-agents to read these files before starting any design or coding phase:

| Document | Purpose | Status |
|----------|---------|--------|
| `docs/adr/README.md` + ADR-0001…0008 | Locked architecture decisions; binding for all work | Present |
| `docs/PRD.md` | Client functional rules, multi-currency logic, budgets (11 modules, MoSCoW) | Present |
| `docs/ARCHITECTURE.md` | Repository structure and design patterns (self-hosted) | Present |
| `docs/DATABASE.md` | Exact relational database design and encrypted-field catalog. **No fields or tables outside this document can be invented.** | Present |
| `docs/SECURITY.md` | Hardening baseline and security requirements | Present |
| `docs/THREAT-MODEL.md` | STRIDE-lite threat model with mitigations and verification | Present |
| `docs/TESTS.md` | QA standards (TDD, enforced coverage thresholds) | Present |
| `docs/DESIGN.md` | Interface design rules, color palette, typography, UI/UX standards | Present |
| `docs/OPERATIONS.md` | Env contract, key rotation runbook, deployment, backups | Present |

**Doc sync rule:** every document states its own sync obligation, and any PR that changes the state a document describes must update the document in the same PR. v1 failed exactly here (docs describing code that did not exist); Koinomy does not inherit that failure.

## 4. Mandatory Workflow (SDD + TDD Orchestration)

For each new task or feature requested by the human user, you must strictly execute this cycle:

### Phase 1: SDD (Specification and Design)

1. Analyze the human requirement against `docs/PRD.md` and `docs/DATABASE.md`, and against the v1 OpenSpec spec for the module being re-ported.
2. Generate a technical plan or design specification detailing which controllers, services, repositories, and components will be created/modified.
3. Verify that the specification meets the *Tenant Isolation* (ADR-0006) and *Encryption* (ADR-0005) rules.
4. **UI Validation:** If the requirement involves the frontend, ensure the specification respects the components and palette defined in `docs/DESIGN.md`.

### Phase 2: TDD (Test-Driven Development)

1. Delegate to the QA agent/sub-process the writing of unit or integration tests (based on the approved SDD).
2. **Mandatory:** The tests must fail initially (Red).
3. Ensure that backend tests mock the dependency injection (especially the `EncryptionService`) and that DB-backed tests run against testcontainers, never a developer's local database (ADR-0007).

### Phase 3: Implementation and Refactoring

1. Delegate to the Development agent/sub-process the writing of the minimum code necessary to pass the tests (Green).
2. Coordinate the execution of the test suite. If it fails, the development agent must fix it.
3. Evaluate the resulting code for technical debt cleanup or TypeScript typing improvements (Refactor).

## 5. Acceptable Patterns (do NOT flag in code review)

This codebase has a small set of patterns that **look** like code smells in isolation but are **intentional, documented decisions**. When a code-review agent flags any of these, treat the flag as a **false positive** unless the reviewer provides a concrete alternative that improves on the documented decision.

### 5.1 Service-layer guards that duplicate boundary validation (Defense in Depth)

Service methods MAY re-validate inputs that the zod schema at the controller boundary already validates. This is **defense in depth**: it protects against direct service calls from background jobs, scheduled tasks, and internal tests that bypass the controller's validation pipeline.

**Acceptable form:** a service method that throws `BadRequestException(...)` for an input that the controller's zod schema already rejects. Do **not** suggest removing the service-layer guard.

**When to flag:** only if the guard adds no value (e.g., it duplicates a trivial, fail-safe check that the project has explicitly chosen to drop).

### 5.2 Per-service `toDomain()` / `fromDomain()` private mappers

Each service MAY have its own private `toDomain(row: DrizzleRowType): DomainType` (or `fromDomain`) mapper over Drizzle inferred row types. The duplication across services is **accepted during the evolution of a module** and gets extracted to a shared `types.ts` (or similar) only when the spec explicitly requires it (the `REQ-XXX-FU-N` follow-up pattern in the change folder).

**Acceptable form:** identical or near-identical `toDomain()` implementations in two services of the same module. Do **not** suggest extracting to a shared helper preemptively.

**When to flag:** when a third or fourth copy appears, OR when the spec explicitly requests extraction.

### 5.3 Direct `db.transaction(...)` calls in services

Service methods MAY call Drizzle's `db.transaction(async (tx) => { ... })` directly. This is the **idiomatic Drizzle primitive** for atomicity, not a "bypass pattern" that needs a custom wrapper.

**Acceptable form:** a service method that wraps an insert + update (or similar multi-write) in a single `db.transaction` block. Do **not** suggest extracting to a custom wrapper or repository helper unless the project adopts such a pattern in a later change.

**When to flag:** if the same multi-write block is duplicated in three or more places (then the project should consider a wrapper).

### 5.4 Imports in mixed order (relative + absolute)

Imports MAY mix relative paths (`./dto/foo`) and alias paths (`@/foo/bar` or similar) within the same file, as long as the project's ESLint flat config does not enforce an import-order rule. Until such a rule is added, import-order suggestions from review agents are **out of scope**.

**Acceptable form:** any import grouping that compiles under `tsc --noEmit` and is consistent within a single file.

**When to flag:** only if the imports are **broken** (wrong path, missing extension where required, circular import). Style grouping is not in scope.

### 5.5 Inline literal values for spec-defined constants

Hard-coded values that come **directly from the spec** (e.g., `1.000000` for a self-currency exchange rate, decimal regex patterns, ISO currency code lists) MAY appear as inline literals with a comment linking to the spec.

**Acceptable form:** a magic literal in the service code with a comment like `// Per spec REQ-TX-9: self-currency rate is 1.000000`. Do **not** suggest extracting to a constant unless the same literal appears in 3+ places.

**When to flag:** when the same literal appears 3+ times (extract to a shared constant), or when the value drifts from the spec (the spec is the source of truth).

### 5.6 Cross-tenant cron scans (documented exception)

A repository method that intentionally scans **across all tenants** (e.g., a daily cron that finds due rules for notification) is acceptable **only when**:

1. The method's purpose is a system-level scheduled task (not a user-facing query).
2. The method has an explicit inline comment explaining the cross-tenant exception and the design decision.
3. The spec explicitly carves out the exception.

**Acceptable form:** a repository method with no `userId` filter plus a JSDoc comment explaining the cross-tenant exception. Do **not** suggest adding a `userId` filter.

**When to flag:** any user-facing query (controller endpoint, list, get) that lacks a `userId` filter is a **CRITICAL** tenant-isolation violation and must be flagged.

### 5.7 Tenant isolation summary

The hard rule (§2) is: **every user-facing query filters by `userId`**. Pattern 5.6 is the **only documented exception** (system-level crons), and shared global catalogs are the only tables intentionally without `userId`. Any other method without a `userId` filter is a bug, not a pattern.

### 5.8 Schema changes — generate AND migrate

When a change modifies the Drizzle schema (`api/src/db/schema` or equivalent) and produces new migration files, the done-criteria **must** include ALL of:

```bash
npx drizzle-kit generate   # produces SQL migration files — these MUST be committed, never gitignored
npx drizzle-kit migrate    # applies the migrations to the dev DB (or the project's deploy script)
npx tsc --noEmit           # typechecks against the schema's inferred types
```

`drizzle-kit generate` alone is **not sufficient**: it only writes the SQL files; it does not apply them. The dev DB stays on the pre-migration schema, and every query to the changed table fails with a misleading "column does not exist" error.

Migration SQL files are reviewed like code. They are **never gitignored** — v1's gitignored migrations directory silently dropped migrations unless force-added, and that failure mode is permanently closed in Koinomy.

## 6. When to override the review agent

If a code-review agent flags something that conflicts with the patterns above, OR that contradicts a documented design decision in an ADR or in the change's `design.md`/`spec.md`, the reviewer is wrong. The fix is one of:

1. **Update this section of `AGENTS.md`** if the pattern is project-wide (like 5.1–5.5).
2. **Update the change's `design.md`** if the pattern is local to that change.
3. **Propose a new ADR** if the disagreement challenges a locked decision (ADR process in `docs/adr/README.md`).
4. **If neither applies:** trust the reviewer's flag and fix the code.

When the reviewer is wrong, do **not** rewrite the code to placate the flag. Document the disagreement in the change's verify report under "Deviations" and move on.
