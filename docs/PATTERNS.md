# Coding Patterns & Conventions

> Catalog of positive coding patterns enforced in Koinomy. Deviations from these patterns are documented as **Acceptable Patterns** in `AGENTS.md` §5, and the architectural decisions that lock them are the ADRs in `docs/adr/`.

## How to read this document

Each pattern follows four parts:

1. **Definition** — what the pattern says.
2. **When it applies in Koinomy** — the context that triggers it.
3. **Example ✅ / Anti-example ❌** — a short, concrete snippet from the financial domain.
4. **ADR reference** — when the pattern is locked by an ADR.

Patterns are opinionated and binding. Code review flags anything that does not match unless it is covered by an Acceptable Pattern (§5).

## 1. Principles

Short, load-bearing rules. Definitions are enough; the body of each pattern lives in the sections below.

| Principle | One-line rule |
|-----------|---------------|
| **SOLID** | Single responsibility, open/closed, Liskov substitution, interface segregation, dependency inversion. Apply at module, class, and function level. |
| **DRY** | Don't Repeat Yourself. Three or more identical occurrences → extract. Two occurrences → leave and document the intent. |
| **KISS** | Prefer the simpler solution. Complexity must justify itself in writing. |
| **YAGNI** | Don't build for hypothetical futures. Add when a concrete need appears. |
| **Tell-don't-Ask** | Objects expose behavior, not data. Don't query an object to decide what to do; tell it to do it. |

## 2. Architecture

### Hexagonal / Ports & Adapters

- **Definition:** Domain at the center; adapters (DB, HTTP, crypto) at the edge. Domain depends on nothing outside itself.
- **When:** Every backend module. The domain must compile and be unit-tested without NestJS, Drizzle, or any external service.
- ✅ A `TransactionService` accepts an `EncryptionService` port and a `TransactionRepository` port via constructor injection; nothing in `domain/` imports `@nestjs/*` or `drizzle-orm`.
- ❌ A service that imports `drizzle-orm` directly and writes SQL inside a domain method.

### Repository pattern

- **Definition:** Persistence sits behind an interface owned by the domain layer. The interface is the contract; the Drizzle implementation is an adapter.
- **When:** Every read or write of a domain entity. The repository interface lives next to the domain type, not next to the Drizzle schema.
- ✅ `interface TransactionRepository { findById(userId: UserId, id: TransactionId): Promise<Transaction | null> }` with a Drizzle implementation in `infrastructure/`.
- ❌ A service that calls `db.select().from(transactions)` directly. The Drizzle row type leaks past the boundary.

### Use Case / Service

- **Definition:** One use case per service method. Services orchestrate repositories and adapters; they do not own SQL or transport.
- **When:** Every controller action maps to one service method. The service method is the seam for unit tests.
- ✅ `async createTransaction(userId: UserId, input: CreateTransactionInput): Promise<TransactionId>` — one job, one return value, no HTTP awareness.
- ❌ A service method that validates input, hits the DB, sends an email, and writes an audit log all in one body with no extraction.

### Adapter

- **Definition:** External concerns (encryption, clock, HTTP clients, SMTP) sit behind a typed interface. The interface lives in the domain; the implementation lives in infrastructure.
- **When:** Any time the application crosses a boundary that does not belong to the domain. The `EncryptionService` is the canonical example.
- ✅ `EncryptionService` interface with `LocalEncryptionService` (AES-256-GCM) today and a vault adapter reserved for later. ADR-0005.
- ❌ A class that calls `crypto.randomBytes` inline inside a service method.

## 3. Backend (NestJS)

### Dependency Injection by constructor

- **Definition:** Collaborators are injected; never `new`-ed inline.
- **When:** Every service, controller, and adapter. Tests substitute test doubles by passing them through the constructor.
- ✅ `constructor(private readonly repo: TransactionRepository, private readonly crypto: EncryptionService) {}`.
- ❌ `const repo = new DrizzleTransactionRepository(db);` inside a service method.

### Modules per bounded context

- **Definition:** One NestJS module per domain module (e.g., `BudgetsModule`, `TransactionsModule`).
- **When:** Always. Module boundaries match domain boundaries.
- ✅ `BudgetsModule` exports `BudgetService`; `TransactionsModule` imports it via the budgets port.
- ❌ A `FinanceModule` that wires every entity into one god module.

### DTOs with zod schemas

- **Definition:** A single zod schema per request DTO; the TypeScript type is `z.infer<typeof X>`. Validation and typing share one source.
- **When:** Every controller boundary. Env config too. ADR-0002.
- ✅ `const CreateTransactionDto = z.object({ amount: decimalString, currency: iso4217, date: isoDate }); type CreateTransactionDto = z.infer<typeof CreateTransactionDto>;`.
- ❌ A TypeScript `interface CreateTransactionDto { amount: number }` — no runtime validation, drift between type and reality.

### Mappers Drizzle → domain

- **Definition:** Each service keeps a private `toDomain(row)` (and `fromDomain(entity)` where needed) over Drizzle inferred row types.
- **When:** Every service that reads from a repository. Duplication across services is accepted during module evolution (AGENTS.md §5.2); extract only when a third or fourth copy appears.
- ✅ `private toDomain(row: InferSelectModel<typeof transactions>): Transaction { return { id: row.id as TransactionId, ... }; }`.
- ❌ Passing the raw Drizzle row up to the controller, leaking `Date`, `string`, and `null` types into the HTTP layer.

### `Result<T, E>` for recoverable errors

- **Definition:** A `Result` (or `Outcome`) type wraps expected business failures (validation, not-found, conflict). `throw` is reserved for unexpected/programmer errors.
- **When:** Service methods whose failures are part of the contract (e.g., "insufficient funds", "duplicate payment").
- ✅ `type Result<T, E = DomainError> = { ok: true; value: T } | { ok: false; error: E }; return { ok: false, error: new InsufficientFundsError() };`.
- ❌ `throw new InsufficientFundsException()` and then catching it three layers up — exceptions are for unexpected failures, not control flow.

### `db.transaction(...)` direct

- **Definition:** Drizzle's transaction primitive is the idiomatic wrapper for atomic multi-writes. No custom wrappers.
- **When:** Any multi-write operation (insert + update, snapshot + balances). ADR-0003. AGENTS.md §5.3.
- ✅ `await this.db.transaction(async (tx) => { await tx.insert(transactions).values(...); await tx.update(accounts).set(...); });`.
- ❌ Two independent `await` calls without a transaction — partial failure leaves the system inconsistent.

## 4. Frontend (React 19)

### Container / Presentational

- **Definition:** A container fetches data and holds state; a presentational component receives props only and renders.
- **When:** Any non-trivial page or section. Containers call TanStack Query; presentational components consume the result.
- ✅ `<BudgetsList>` (presentational) receives `{ budgets: Budget[]; isLoading: boolean }`; `<BudgetsListContainer>` calls `useBudgets()` and passes the result.
- ❌ A component that calls `fetch('/api/budgets')` inside `useEffect` and renders the data — duplicating what TanStack Query already does.

### Custom Hooks

- **Definition:** Stateful logic that is reused across components lives in a hook named `useXxx`.
- **When:** Any logic that combines state, effects, and callbacks in a way that more than one component needs.
- ✅ `function useBudgetProgress(budgetId: BudgetId) { return useQuery({ queryKey: ['budget', budgetId], queryFn: ... }); }`.
- ❌ Copy-pasted `useEffect` + `useState` blocks in three components doing the same fetch.

### Composition over inheritance

- **Definition:** Combine small components and hooks; do not extend React components via class hierarchies.
- **When:** Always. There is no use case for a React class in Koinomy.
- ✅ `<Card>` composed with `<CardHeader>` + `<CardBody>` + `<CardFooter>`; behavior added via hooks.
- ❌ `class BudgetCard extends BaseCard { ... }` — React does not need inheritance, and it obscures composition.

### Component layering (tokens → primitives → compositions → pages)

- **Definition:** UI is built in four layers, matching the vocabulary of `docs/DESIGN.md`: **tokens** (color, radius, spacing, typography — the frontmatter tokens of `DESIGN.md` mapped 1:1 into the Tailwind 4 theme), **primitives** (shadcn/ui over Radix primitives — the smallest interactive units), **compositions** (multi-primitive blocks such as `Card`, `Form`, `Dialog` per `DESIGN.md` §Components), **pages** (route-level assemblies). Implementation substrate is shadcn/ui + Radix primitives over Tailwind 4 (ADR-0002); v1 hand-rolled components are not carried over.
- **When:** Every UI element. Tokens are referenced via `{token.refs}`; primitives are the only building blocks; compositions are designed once and reused.
- ✅ `colors.primary` token → `<Button>` primitive → `<CurrencyAmountInput>` composition → `<NewTransactionPage>` page.
- ❌ A `Button` redefined per page with hard-coded colors — token and primitive discipline broken.

### Strict TypeScript everywhere

- **Definition:** No `any`. `unknown` + narrowing. Hook return types are explicit.
- **When:** All client code, no exceptions. ADR-0002.
- ✅ `function useFormSubmit<T>(schema: ZodSchema<T>): { submit: (data: unknown) => Result<T, ValidationError> }`.
- ❌ `const [data, setData] = useState<any>(null);` — `any` defeats type checking.

## 5. Testing

### Test pyramid

- **Definition:** Many unit tests, fewer integration tests, few E2E tests. The unit layer proves business rules; the E2E layer proves the wiring.
- **When:** Every module. Tests are organized by layer, not by file.
- ✅ A budget module: ~30 unit tests for the service, ~5 integration tests against testcontainers, 1 E2E for the create-budget flow.
- ❌ A module covered only by E2E tests — slow feedback, hard to localize failures.

### AAA (Arrange-Act-Assert)

- **Definition:** Every test has three blocks: arrange the world, act, assert the outcome.
- **When:** Every test, regardless of framework.
- ✅

  ```ts
  it('rejects an installment when balance is insufficient', async () => {
    const account = aBudget().withBalance('0.00').build();
    const service = new BudgetService(repo, clock);
    const result = await service.payInstallment(account.id, installmentId);
    expect(result.ok).toBe(false);
  });
  ```

- ❌ A test that arranges, acts, asserts, then arranges again, then acts again — multiple cases in one body.

### Builders & fixtures

- **Definition:** Reusable factories expose the entity's parts as builder methods. No copy-pasted object literals across tests.
- **When:** Whenever the same entity appears in more than two tests.
- ✅ `aBudget().withCurrency('USD').withPeriod(monthOf(2026, 8)).withLimit('1500.000000').build()`.
- ❌ `{ id: 'b1', userId: 'u1', currency: 'USD', periodStart: new Date(...), limit: '1500.000000' }` pasted in eight tests.

### testcontainers for DB-backed tests

- **Definition:** Integration and E2E tests get a real, disposable PostgreSQL instance. Never a developer's local DB.
- **When:** Every test that reads or writes the database. ADR-0007. `docs/TESTS.md` §4.
- ✅ A test that boots `new PostgreSqlContainer('postgres:16')` in a global setup and runs migrations on it.
- ❌ A test that connects to `process.env.DATABASE_URL_TEST` pointing at the developer's local Postgres — results depend on machine state.

### TDD red-green-refactor

- **Definition:** Tests first, then implementation, then refactor. The Red step is mandatory for financial and security logic.
- **When:** Every change to financial or security logic. AGENTS.md §4 Phase 2. `docs/TESTS.md` §2.
- ✅ Spec written → failing test committed → implementation → green → refactor under green. The verify report cites the failing commit.
- ❌ Implementation first, tests second. The financial correctness gate does not pass.

## 6. TypeScript

### `strict: true` + `noUncheckedIndexedAccess: true`

- **Definition:** Both flags on, no opt-outs. Index access returns `T | undefined`; the compiler forces the missing case.
- **When:** Always. ADR-0002.
- ✅ `const first = arr[0]; if (first === undefined) throw new Error('empty');` — the compiler enforces the guard.
- ❌ `const first = arr[0]; first.toString();` — silent crash when `arr` is empty.

### zod at every boundary

- **Definition:** Untyped input (HTTP, env, files, third-party payloads) is parsed through a zod schema before it crosses into typed code.
- **When:** Every boundary between untyped and typed code. ADR-0002.
- ✅ `const env = EnvSchema.parse(process.env);` at boot, then `env` is typed for the rest of the process.
- ❌ `process.env.DATABASE_URL as string` — the missing value crashes at request time, not boot.

### Branded types for IDs

- **Definition:** `type UserId = string & { readonly __brand: 'UserId' }` prevents passing the wrong ID to a function that expects a specific ID type.
- **When:** Every ID that flows through the system: `UserId`, `TransactionId`, `AccountId`, `BudgetId`, `InstallmentId`, `CurrencyCode`.
- ✅ `function loadBudget(userId: UserId, budgetId: BudgetId): Promise<Budget>` — `loadBudget(budgetId, userId)` is a compile error.
- ❌ `function loadBudget(userId: string, budgetId: string)` — easy to swap the order, no help from the compiler.

### No `any`

- **Definition:** `any` is forbidden. Use `unknown` and narrow. `@typescript-eslint/no-explicit-any` is enforced.
- **When:** Always. There is no scenario where `any` is the right answer.
- ✅ `function parseAmount(input: unknown): Result<Decimal, ValidationError> { const parsed = DecimalString.safeParse(input); ... }`.
- ❌ `function parseAmount(input: any): any` — the compiler has nothing to say and the bug ships.

### No `as` casts to silence the compiler

- **Definition:** `as` is a signal that the type system is telling you something. Refactor instead of silencing.
- **When:** When you reach for `as`, stop. Either narrow with a type guard or change the upstream type.
- ✅ `if (typeof raw === 'string') return raw as IsoDate;` — explicit guard.
- ❌ `return raw as IsoDate;` — you have moved a runtime bug into production because the compiler stopped arguing.

## 7. Anti-patterns (rejected in Koinomy)

### God objects / god services

- **Definition:** A class that knows everything and does everything. Split by responsibility.
- **When:** Flag whenever a service exceeds one screen or imports from more than two domain modules.
- ✅ `TransactionService`, `BudgetService`, `NotificationService` — each owns one slice.
- ❌ A `FinanceService` that creates transactions, manages budgets, runs cron jobs, sends emails, and writes audit logs.

### Anemic domain model

- **Definition:** Entities that are pure data and push all behavior into services. Move behavior to the entity.
- **When:** Flag whenever an entity has zero methods and a service has methods named `entity.doX()` that read the entity's fields.
- ✅ `class Budget { applyPayment(payment: Payment): Result<Budget, DomainError> { ... } }` — behavior lives with the data.
- ❌ `class Budget { id; limit; spent; }` + `BudgetService.applyPayment(budget, payment)` mutating fields from outside.

### Leaky abstractions

- **Definition:** A layer's types appearing in the layer above. Drizzle rows, HTTP request objects, ORM entities — none of these belong in a controller's response.
- **When:** Flag whenever a service or controller exposes a row type, a `Date`, a raw `string`, or a transport object to the layer above.
- ✅ The repository returns a `Transaction` domain object; the controller maps it to a response DTO.
- ❌ A controller that returns the Drizzle row directly in the JSON body — `Date` becomes a string, `null` becomes `null`, but `bigint` breaks JSON.

### Premature abstraction

- **Definition:** Generic base classes, plugin systems, factory-of-factory patterns, DI containers configured for things that do not exist yet. KISS and YAGNI.
- **When:** Flag whenever a class has zero concrete users but is "ready for future extension".
- ✅ A concrete `TransactionRepository` interface written for the current need; abstraction emerges when a second implementation arrives.
- ❌ `abstract class AbstractRepository<T, ID> { abstract findById(id: ID): Promise<T | null>; abstract save(entity: T): Promise<void>; }` with one implementation.

### Feature envy

- **Definition:** A method that uses more features of another class than its own. Move it.
- **When:** Flag whenever a service method reads three fields of an entity and zero of its own dependencies.
- ✅ `budget.progress(now: Date): Progress` — the behavior belongs to the budget.
- ❌ `BudgetService.computeProgress(budget)` reads `budget.spent`, `budget.limit`, `budget.period` — nothing else.

### Cross-tenant queries without a documented exception

- **Definition:** Every user-facing query filters by `userId`. The only exception is system-level crons, and only when the spec explicitly carves it out. ADR-0006. AGENTS.md §5.6, §5.7.
- **When:** Flag any user-facing query (controller endpoint, list, get) that lacks a `userId` filter — this is a critical defect.
- ✅ `db.select().from(transactions).where(and(eq(transactions.userId, userId), eq(transactions.id, id)))`.
- ❌ `db.select().from(transactions).where(eq(transactions.id, id))` in a list endpoint.

### Silent stubs

- **Definition:** An unimplemented security adapter that "works" at runtime instead of failing boot. Boot must fail loudly. ADR-0005. AGENTS.md §2.1 (red line 5).
- **When:** Flag whenever an adapter implementation logs "not implemented" and returns a fake value.
- ✅ The provider module refuses to boot if the configured adapter is `VaultEncryptionService` and the implementation is absent.
- ❌ `class VaultEncryptionService { encrypt(x: string) { console.warn('not implemented'); return x; } }` — encryption is now a no-op and the system is unsafe.

## Cross-references

- `AGENTS.md` §5 — Acceptable Patterns (intentional deviations from generic best practice).
- ADR-0002 — Core stack (NestJS, React, zod, strict TypeScript).
- ADR-0003 — Drizzle data layer (SQL migrations, repository pattern).
- ADR-0005 — Field-level encryption (adapter pattern, fail-fast provider).
- ADR-0006 — Multi-tenant isolation (repository pattern + cross-tenant cron exception).
- ADR-0007 — Testing strategy (TDD, testcontainers, coverage thresholds).
- ADR-0008 — Dependency hygiene & quality gates.
- `docs/ARCHITECTURE.md` — Layering, request lifecycle, error contract.
- `docs/TESTS.md` — Testing standards (TDD, mocking rules, required test classes).
- `docs/DESIGN.md` — Frontend conventions (atomic design, palette, tokens).
