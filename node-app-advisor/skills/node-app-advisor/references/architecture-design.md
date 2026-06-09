# Architecture & Design Analysis

Identify structural issues, layering violations, SOLID violations, coupling problems, missing abstractions, and design anti-patterns. Every finding needs a file path, code evidence, severity, and a concrete fix.

**Core principle:** Architecture is the skeleton of maintainability. Bad architecture doesn't crash the app today — it makes every future change harder, slower, and riskier. Find the structural rot before it becomes load-bearing.

## Contents

- Step 1 — Map the architecture
- Step 2 — Layering & module-boundary violations (incl. worker/CLI)
- Step 3 — SOLID principles
- Step 4 — Coupling & cohesion
- Step 5 — Design anti-patterns
- Step 6 — Compile findings

## Tools

- `Grep` — imports/`require`, exports, decorators, route registrations, class/function declarations, the patterns in the tables below.
- `Glob` — directory/package structure (`src/**`, `**/package.json`, `pnpm-workspace.yaml`, `turbo.json`, `nx.json`).
- `Read` — read suspect modules in full (architectural issues span whole files); for very large files, grep the symbol then read that region.
- `Agent` (Explore, breadth `medium`/`very thorough`) — fan-out searches: "find all modules over 500 lines", "map all cross-package imports". Re-`Read` any lead before citing it.
- **Code usages / coupling:** `Grep` the export/function name across the tree to find who depends on it (no `listCodeUsages` tool exists). Mind tsconfig `paths` aliases — an aliased import (`@/services/...`) won't match a relative-path grep.
- `Bash` — read-only `git log` for the change-frequency signals in Step 3 (e.g. `git log --since=12.months --name-only --pretty=format: -- '*.ts' '*.js' | sort | uniq -c | sort -rn | head`). Skip gracefully if the project isn't a git repo.
- `TodoWrite` — track the steps.

## Step 1 — Map the architecture

Understand the *intended* architecture before judging it.

1. **Architectural pattern** — Layered (Controller/Route → Service → Repository)? Hexagonal/Ports & Adapters? Clean Architecture? Package-by-feature vs. package-by-type (`controllers/`, `services/`, `models/`)? Event-driven/CQRS? NestJS module graph? No discernible pattern (a finding in itself)?
2. **Directory structure** — list top-level directories under `src`; identify each one's role (routes/controllers, services, repositories/data, models/entities, middleware, config, util); flag directories that don't fit the pattern or have ambiguous names.
3. **Package boundaries** (monorepo) — read the `workspaces` field / `pnpm-workspace.yaml` / `nx.json` for the package list; map the inter-package dependency graph; check whether packages import each other's internals vs. their published `exports`/`index` entry point.
4. **Domain model** — find entity/domain/model definitions (Prisma schema, TypeORM `@Entity`, Mongoose schemas, Zod schemas, plain types); is the model anemic (just data shapes) or rich (contains business logic)? Are domain types leaking framework/ORM types into the core?
5. **Entry points by framework:**

   | Framework | Detection |
   |---|---|
   | Express | `express()`, `app.use(...)`, `Router()`, `app.listen(...)` |
   | Fastify | `Fastify()`, `fastify.register(...)`, `fastify.route(...)`, `.listen(...)` |
   | NestJS | `@Module`, `@Controller`, `@Injectable`, `NestFactory.create(...)`, `bootstrap()` |
   | Koa | `new Koa()`, `app.use(...)`, `@koa/router` |
   | Hono | `new Hono()`, `app.get/post(...)`, `export default app` |
   | Next.js | `app/**/route.ts` (App Router), `pages/api/**` (Pages Router), `middleware.ts` |
   | tRPC | `initTRPC`, `router({...})`, `publicProcedure`/`protectedProcedure` |
   | Plain Node / CLI | `src/index.ts`/`main.ts`, `#!/usr/bin/env node` + `bin` in package.json, `process.argv` parsing |
   | Worker / scheduler | BullMQ `Worker`/`Queue`, `node-cron`, `setInterval` loops, message-broker consumers |

**Present** a brief description of the architecture — pattern, structure, packages, domain model.

## Step 2 — Layering & module-boundary violations

Find places where layers bypass each other or depend in the wrong direction.

| Violation | Detection | Severity |
|-----------|-----------|----------|
| Route/controller hitting the DB directly | Route handler importing the ORM client / repository and running queries inline | High |
| Service handling HTTP concerns | Service importing `req`/`res`/`Request`/`Response` types or reading headers/query params | High |
| Repository containing business logic | Data-access modules with complex domain logic that belongs in the service layer | Medium |
| Domain depending on infrastructure | Domain/core modules importing the ORM, the web framework, or `process.env` in a hexagonal/clean architecture | Medium |
| Circular dependencies | Module A imports B and B imports A (also breaks ESM/CJS init order) | High |
| Util/Helper doing business logic | `*.utils.ts`/`helpers/` modules holding domain logic instead of utilities | Medium |
| Cross-feature direct access | Feature A's code importing Feature B's repository/internal module, bypassing B's public API | High |
| Importing across package internals | In a monorepo, importing `other-pkg/src/...` instead of its package entry point | High |

**Worker / CLI-specific:**

| Violation | Detection | Severity |
|-----------|-----------|----------|
| God entry file | `index.ts`/`main.ts` / a single job handler holding all business logic instead of delegating | High |
| No separation in jobs | Fetch / transform / persist logic mixed in one handler | High |
| Scheduler doing business logic | `node-cron`/`setInterval` callbacks with complex inline logic instead of delegating | Medium |
| Missing job orchestration | Independent jobs with no coordination strategy (ordering, dependency, failure handling, retries) | Medium |
| I/O mixed with business logic | File parsing, network calls, and business rules in the same function | High |

**How to detect:** use the Explore subagent to scan imports across routes/services/repositories; for each route handler check for direct ORM/repository imports; for each service check for web-framework request/response imports; map cross-module imports for cycles (or run `npx madge --circular src` if the user approves running a tool).

## Step 3 — SOLID principles

### Single Responsibility (S)

| Smell | Detection | Severity |
|-------|-----------|----------|
| God module/class | > 500 lines, > 15 exported functions/methods, or > 10 injected/imported dependencies | High |
| Service doing unrelated things | Functions spanning different business domains in one module | Medium |
| Controller with business logic | Route/controller handlers with > 20 lines of logic (not just delegation + validation) | Medium |
| Barrel file doing init + logic | `index.ts` re-exporting *and* running side-effecting setup | Low |

*Detect God modules:* Explore subagent → "find all source files over 500 lines"; for each, count imported deps, exported functions, and whether they span concerns. For NestJS, count constructor-injected providers.

### Open/Closed (O)

| Smell | Detection | Severity |
|-------|-----------|----------|
| Long if/else or switch chains | Functions with > 5 branches dispatching on a type/string discriminant | Medium |
| Repeated edits to the same files | Frequent changes to the same files for different features (git history) | Medium |
| Missing strategy/registry | Repeated conditional logic that could be a lookup map / polymorphic handler | Medium |

### Liskov Substitution (L)

| Smell | Detection | Severity |
|-------|-----------|----------|
| Override throws "not implemented" | Subclass/implementer refusing the parent/interface contract | High |
| Type-narrowing in polymorphic code | `instanceof` / `typeof` / `in` discriminant checks after retrieving from a base-type collection | Medium |

### Interface Segregation (I)

| Smell | Detection | Severity |
|-------|-----------|----------|
| Fat interfaces/types | Interfaces with > 10 members forcing partial implementations | Medium |
| Empty/stub implementations | Classes implementing an interface with no-op methods | Medium |
| Over-broad public surface | A module exporting far more than its consumers use | Low |

### Dependency Inversion (D)

| Smell | Detection | Severity |
|-------|-----------|----------|
| Concrete-module coupling | Importing a concrete implementation where an interface/abstraction would decouple | Medium |
| Missing abstractions | `new`-ing dependencies (DB clients, HTTP clients) inline instead of injecting/passing them | High |
| Framework coupling in domain | Domain/business modules importing the ORM, web framework, or env directly | Medium |

## Step 4 — Coupling & cohesion

| Issue | Detection | Severity |
|-------|-----------|----------|
| Tight package coupling | Package A imports B's internal files (not its `exports`/index) | High |
| Feature envy | Functions using more data from another module than their own | Medium |
| Data clumps | The same group of parameters passed together to many functions | Low |
| Global singleton sprawl | Shared mutable module-level state imported everywhere (DB client, config object mutated at runtime) | Medium |
| Shotgun surgery | One business change touches > 5 files across directories | Medium |
| Missing bounded contexts | All models in one directory with no domain boundaries | Medium |
| `any`-typed boundaries (TS) | `any`/`as any` at module boundaries defeating the type contract | Medium |

## Step 5 — Design anti-patterns

| Anti-pattern | Detection | Severity |
|--------------|-----------|----------|
| Anemic domain model | Models/types with only data, all logic in services | Medium |
| Service Locator / global container | Reaching into a global DI container or `globalThis` instead of explicit deps | High |
| Magic strings/numbers | Hardcoded config keys, role names, status values, event names | Medium |
| Exception swallowing | Empty `catch`, or catch-log-and-continue; unhandled promise rejections | High |
| God controller/router | One router file with > 10 endpoints across business domains | Medium |
| Callback hell / nested `.then()` | Deeply nested callbacks/promise chains instead of `async/await` | Medium |
| Floating promises | `async` calls not `await`ed or `.catch()`ed (fire-and-forget by accident) | High |
| DTO/type explosion | A separate type for every minor variation, excessive mapping | Low |
| Utility-module abuse | Catch-all `utils.ts` holding business logic | Medium |
| Premature abstraction | Single-implementation interface/factory with no extension point | Low |
| Circular dependency | Two+ modules depending on each other | High |
| Overuse of inheritance | Class hierarchies > 3 deep instead of composition | Medium |
| Mixed module systems | `require()` and `import` mixed, or `.cjs`/`.mjs`/`.ts` interop hazards in one package | Medium |

## Step 6 — Compile findings

For each finding record: **Category** (Layering / Module Boundary / SOLID / Coupling / Anti-Pattern) · **Severity** · **Description** · **Evidence** (`file:line` + snippet) · **Impact** (maintainability, testability, scalability, correctness) · **Recommended fix** (specific refactor, before/after) · **Effort** (XS/S/M/L/XL). Present the compiled findings.
