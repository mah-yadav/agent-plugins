# Area: Architecture, APIs, Data & Testing (Node)

Reference for Phase 3.2. Produces a structured code explanation covering architecture, control flow, data flow, design patterns, and testing.

## Contents

- **Priority tiers** · **Boundaries** (identify-don't-deep-dive)
- **Node reference routing** — which `node-*.md` file to load per topic
- **Ledger read-before** · **Decorator-aware / ESM / edge / symbol-tracing notes**
- **Step 2: Deep analysis** — 2.1 Architecture · 2.2 Dependencies · 2.3 Data · 2.4 Control flow · 2.5 Patterns · 2.6 Testing
- **Step 3: Synthesize and document**

## Node reference routing

This area draws on the `node-*.md` reference files. Load **only** the file the current topic needs — each is a clean whole-file `Read`:

| When analyzing… | `Read` |
|---|---|
| Framework, source layout, request lifecycle, async, error handling | `node-frameworks.md` |
| TypeScript strictness/idioms, DI, decorators, ESM/CJS | `node-type-system.md` |
| Data models, ORMs, migrations, validation, caching, pooling | `node-data-layer.md` |
| Design patterns, modern features, code-level anti-patterns | `node-patterns.md` |
| Test runners, commands, coverage, mocking | `node-testing.md` |

**Always** load `node-frameworks.md` (architecture + control flow are P0). Load the others when their topic comes up, or eagerly in Standard/Deep mode.

## Priority Tiers (within this area)

| Section | Priority | Rationale |
|---|---|---|
| 2.1 Architecture and Structure | **P0** | Foundation for everything else |
| 2.2 Dependencies and Integrations | **P1** | Deep coverage belongs in Standard+ |
| 2.3 Data Flow (key entities/schemas) | **P0** | Top-5 model table is day-1 essential |
| 2.3 Data Flow (full schema, transactions, caching) | **P1** | Deeper detail |
| 2.4 Control Flow (one representative request) | **P0** | End-to-end trace is day-1 essential |
| 2.4 Control Flow (full route map, error handling, async) | **P1** | Deeper detail |
| 2.5 Design Patterns | **P1** | Important but not day-1 blocking |
| 2.6 Testing (commands + frameworks) | **P0** | Devs must run tests on day 1 |
| 2.6 Testing (strategy, mocking, coverage detail) | **P1** | Useful for writing tests |

**Quick mode**: 2.1 summary, 2.3 key models, 2.4 one representative flow, 2.6 commands+frameworks. Skip 2.2 and 2.5.

## Boundaries — Identify-Don't-Deep-Dive

| Topic | What to Do | Belongs to |
|---|---|---|
| Auth library / middleware | Note its presence and the auth mechanism. Do NOT trace the full middleware stack or enumerate routes by access level. | 3.3 Security |
| Quality tools (ESLint, Prettier, Biome) | Note presence in `devDependencies`. Do NOT read rule configs. | 3.6 Code Quality |
| Logging / metrics library | Note the library (pino/winston/etc.) and metrics backend. Do NOT analyze log format details. | 3.4 Observability |
| CI/CD pipeline files | Note their existence. Do NOT analyze pipeline stages. | 3.5 CI/CD |

## Ledger Read-Before

Read the ledger. Check whether `package.json`, `tsconfig.json`, the framework's main config (next.config, nest-cli.json, etc.), and any env-schema file are already in `Files Read`. Note `Decorator usage`, `Module system`, `Edge runtime target`, `Language` from Project Identity.

## Diagrams

Embed Mermaid blocks for sequence diagrams; viewers (GitHub, VS Code, IntelliJ) render them.

## Decorator-Aware Reading

If the ledger marks `Decorator usage = Yes`:
- Read [node-type-system.md](./node-type-system.md) § Decorator-aware reading before tracing any class.
- NestJS routes are not visible as `app.get(...)` calls — they're declared via `@Controller`/`@Get`/`@Post` and wired by reflection.
- TypeORM/MikroORM entity columns are declared via `@Column` — DB schema is implicit.
- class-validator rules are declared via `@IsX` decorators on DTO classes.

If decorator usage = No, the wiring is explicit in source; trace via imports.

## ESM vs CJS

Check the ledger's `Module system`. This affects:
- Whether `require()` or `import` is used (don't be surprised by either).
- Whether `__dirname` exists natively (CJS) or needs `import.meta.url` (ESM).
- Whether top-level `await` is possible.
- Cross-module compatibility (CJS importing ESM is async-only).

If `Module system = Mixed`, identify the boundary — usually the build output is one and the source is another, or sub-packages differ.

## Edge Runtime

If `Edge runtime target = Yes`, the runtime is V8-isolate, not Node:
- `fs`, `child_process`, native modules — NOT available.
- `crypto` is the Web Crypto API, not Node's.
- Request lifecycle uses Web Standard `Request`/`Response`, not Node `http.IncomingMessage`.
- Many npm packages won't run.

Recommendations and code examples in the report must respect this.

## Symbol-Usage Tracing

No native LSP-backed "find all references" — use `Grep` precisely, paired with file reading for context. **Path aliases** from tsconfig (`@/foo`, `~/bar`, `@app/x`) mean a search for the literal path won't find aliased imports. Always search for both the literal path and the aliased form.

## Step 1: Load the Node References

Load `node-frameworks.md` now (framework + source layout + control flow are needed for everything below). Load the other `node-*.md` files per the routing table above as each topic comes up — or all of them eagerly in Standard/Deep mode.

If the project includes substantial non-Node code (Python services, Go binaries called via child_process, Rust native modules), note their presence but don't deep-dive — recommend a separate analysis.

## Step 2: Deep Analysis

### 2.1 Architecture and Structure — P0

- **Project layout**: Source organization. Conventional patterns:
  - **Layered**: `routes/`, `controllers/`, `services/`, `repositories/`, `models/`
  - **Feature/domain-grouped**: `users/`, `orders/`, `payments/` each with their own routes/services/etc.
  - **NestJS modular**: each domain is an `@Module` with its own `@Controller`, `@Service`, `@Repository`
  - **Hexagonal / clean**: `domain/`, `application/`, `infrastructure/`, `interfaces/`
  - **Single-file**: small Express apps in one file
- **Entry point(s)**: `index.{ts,js}`, `server.{ts,js}`, `main.ts` (NestJS), `bin/start.js`. Check `package.json` `main`/`module`/`exports`/`bin`.
- **Bootstrap sequence**: What happens before the server listens? Order of:
  - Env loading
  - DB connection / migrations
  - DI container setup (NestJS / InversifyJS / awilix)
  - Middleware registration
  - Route mounting
  - Background job startup
  - Server listen

**Minimum viable output**: Layer/feature mapping table, architectural pattern name, main entry point file, bootstrap sequence summary.

### 2.2 Dependencies and Integrations — P1 (skip in Quick mode)

- **Runtime dependencies**: Group `package.json` `dependencies` by purpose (framework / data / auth / observability / utility).
- **External integrations**: Look for SDK clients — `@aws-sdk/*`, `@google-cloud/*`, `stripe`, `twilio`, `sendgrid`, `@octokit/*`. Note each: protocol, env vars used, where instantiated.
- **HTTP clients**: `axios`, `got`, `node-fetch`, `undici`, native `fetch` (Node 18+), `ky`. Where is the timeout? Where is the retry policy? Where is the error handling?
- **Internal dependencies** (monorepo): Which package imports which. Trace `workspace:*` deps in package.json files.

**Minimum viable output**: External integration list (service name, protocol, config key).

### 2.3 Data Flow — P0

- **Data models / schemas**: Trace via the ORM/validation library detected:
  - **Prisma**: `prisma/schema.prisma` is the source of truth for models AND DB schema.
  - **Drizzle**: `src/db/schema.ts` (or similar) — tables defined as code.
  - **TypeORM/MikroORM**: `@Entity()` classes.
  - **Mongoose**: `new Schema({...})` then `model('Name', schema)`.
  - **Zod**: schemas in `schemas/` or `validators/` directory — often the runtime contract.
- **Validation boundary**: Where does input get validated? Top of the request flow (middleware), per-route, or inside the service? Note libraries (Zod, Yup, Joi, class-validator, AJV).
- **DTOs vs. domain models**: Are they the same type, or transformed? Look for mapper functions.
- **Migration tooling**: Prisma Migrate, Drizzle Kit, TypeORM CLI, Knex Migrate, raw SQL files in `migrations/`.
- **Data store(s)**: Postgres, MySQL, MongoDB, SQLite, Redis (cache vs. primary), DynamoDB. Sometimes multiple.

**Minimum viable output**: Top 5 model/schema table (name, purpose, key fields), data store(s) + ORM identified.

### 2.4 Control Flow — P0

- **Request lifecycle**: Trace end-to-end. Use the framework-specific lifecycle from [node-frameworks.md](./node-frameworks.md) § Request lifecycles.
- **Routing**: How are routes registered?
  - Express/Fastify/Koa/Hono: explicit `app.METHOD(path, handler)` calls.
  - NestJS: `@Controller` + `@Get`/`@Post` decorators; routes implicit.
  - Next.js App Router: filesystem-based (`app/api/foo/route.ts` → `/api/foo`).
  - Next.js Pages Router: filesystem-based (`pages/api/foo.ts` → `/api/foo`).
  - tRPC: procedure registration in routers; no HTTP routes per procedure.
- **Middleware chain / hooks**: The actual lifecycle. Note ordering — middleware order is often a bug source.
- **Error handling**: Express 4 error middleware (4-arg), Express 5 auto-catch, Fastify `setErrorHandler`, NestJS exception filters, Hono `app.onError`. Centralized or scattered?
- **Async patterns**: async/await everywhere, or callback-based legacy code? Promise.all usage? `for await` loops?
- **Background work**: BullMQ workers, cron jobs (`node-cron`, `@nestjs/schedule`), one-off scripts.

**Minimum viable output**: One representative request traced end-to-end with a Mermaid sequence diagram showing middleware/hook → handler → service → repo → response.

### 2.5 Design Patterns and Idioms — P1

- Patterns from [node-patterns.md](./node-patterns.md) § Design patterns.
- Note language idioms:
  - JS/TS: discriminated unions, Result types, branded types, `satisfies`, `as const`.
  - Promise composition: `Promise.all` vs. `allSettled` vs. `any`.
  - Generator functions, async iterators.
  - Symbol-keyed properties.
- Call out anti-patterns (see [node-patterns.md](./node-patterns.md) § Anti-patterns).

**Minimum viable output**: Pattern table (pattern, where used, purpose).

### 2.6 Testing — P0

- Test framework: Jest / Vitest / Mocha / node:test / AVA — see [node-testing.md](./node-testing.md) for detection.
- Test commands: from `package.json` scripts.
- **Test types** present:
  - **Unit**: usually colocated (`foo.ts` + `foo.test.ts`) or in `__tests__/` or in `tests/`.
  - **Integration**: typically `test:integration`, `test:e2e`, sometimes their own directory; often use Testcontainers, MSW, or a test docker-compose.
  - **Contract**: Pact, Spring Cloud Contract.
  - **E2E** (browser): Playwright, Cypress — these test the frontend; note presence but boundary applies.
- Mocking: Jest/Vitest built-in, `sinon`, MSW, `nock`.
- Test fixtures / factories: `factory.ts` files, `@faker-js/faker`, `fishery`.
- Coverage tool: Istanbul (legacy), c8, Jest --coverage, Vitest --coverage. Threshold config in `jest.config.*` or `vitest.config.*`.

**Minimum viable output**: Test framework list, test commands table, naming convention.

## Step 3: Synthesize and Document

1. Review findings and organize into a narrative.
2. Use the [code explanation template](../assets/code-explanation-template.md).
3. Include real code snippets (3–10 lines) — never fabricate.
4. Include Mermaid diagrams for non-trivial flows.
5. Write to `<output-dir>/code-explanation-report.md` (Standard/Deep mode only).

**Quick mode document**: Fill the *Overview*, *Tech Stack*, *Project Structure*, *Architecture*, *Request Flow* (one representative), *Data Models* (key entities), and *Testing* (commands + frameworks) sections. Mark others `[Not analyzed]`.

## Ledger Update After

Mark area 3.2 complete. Add:
- Architectural pattern identified
- Bootstrap sequence summary
- Key models/schemas and relationships
- Route count, key route examples
- Test framework(s) and commands
- Boundary findings for downstream areas (auth library, logging library, quality tools)

## Conversational Rules

- **Depth over breadth**: Better to deeply explain fewer modules than superficially cover everything.
- **Adapt to audience**: Experienced devs want patterns and gotchas; newcomers want step-by-step.
- **Don't assume**: If purpose is ambiguous, say so and ask.
- **Progress updates**: Periodically during deep analysis.
- **Explain the why**: Connect implementation to architectural decisions.
- **Call out noteworthy patterns**: Anti-patterns, clever solutions, tech debt — be opinionated with reasoning.
