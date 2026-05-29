# Node.js Code Analysis Guide

Detailed techniques for analyzing and explaining Node.js code bases (JavaScript + TypeScript). This guide is split into two sections:

- **Essential Analysis** (Sections 1–4): Always load. Covers project identification, framework detection, type system handling, and control flow.
- **Extended Analysis** (Sections 5–10): Load on demand. Covers data layer, design patterns, modern Node/TS features, testing, security, anti-patterns.

**Loading guidance**:
- Read lines 1 through the `=== END ESSENTIAL ANALYSIS ===` marker for Quick mode or for single-question Q&A.
- Read the full file for Standard/Deep mode or when analyzing data layer, testing, or design patterns in depth.

---

# =================================================================
# === ESSENTIAL ANALYSIS (Always Load) ============================
# =================================================================

## 1. Project Identification

Determine the project type, runtime, package manager, and structure before reading any source.

### Manifest & lockfile

| Indicator | What It Tells You |
|---|---|
| `package.json` at root | Node project. Read `name`, `version`, `engines`, `type` (module / commonjs), `scripts`, `dependencies`, `devDependencies` |
| `package.json` with `"workspaces"` | npm or Yarn workspace monorepo — read the array for package locations |
| `pnpm-workspace.yaml` | pnpm workspace monorepo — read for package locations and catalogs |
| `package-lock.json` | npm — pinned dependency tree |
| `yarn.lock` | Yarn (Classic v1 or Berry v2+) — check `.yarnrc.yml` to distinguish |
| `pnpm-lock.yaml` | pnpm — the most strict resolver; `node_modules` will be symlink-heavy |
| `bun.lockb` or `bun.lock` | Bun runtime / package manager |
| `.nvmrc` / `.node-version` / `engines.node` | Pinned Node version — install this before running anything |
| Multiple lockfiles present | **Red flag** — different team members likely using different package managers. Flag immediately. |

### Runtime target

| Indicator | Runtime / Target |
|---|---|
| `engines.node` in package.json | Standard Node.js — match version |
| `"type": "module"` in package.json | ESM-default project (otherwise CommonJS) |
| `.cjs` or `.mjs` files | Mixed CJS/ESM — call out the boundary |
| `next.config.{js,ts,mjs}`, `pages/` or `app/` directory | Next.js (full-stack React). API routes live in `app/api/` or `pages/api/` |
| `nuxt.config.{ts,js}` | Nuxt (Vue full-stack) |
| `astro.config.{js,ts,mjs}` | Astro (mostly static + islands) |
| `wrangler.toml` | Cloudflare Workers — edge runtime, **not** full Node API |
| `vercel.json` with `runtime: "edge"` | Vercel Edge — V8 isolate, **not** Node API |
| `deno.json` / `deno.jsonc` | Deno runtime (overlap with Node but different stdlib) |
| `bun.toml` / Bun-specific scripts | Bun runtime |

**Edge runtime alert**: If any of `wrangler.toml`, `@cloudflare/workers-types`, or `runtime: "edge"` appears, the code may NOT have access to Node-only APIs (`fs`, `child_process`, parts of `crypto`, native modules). Read the actual runtime before assuming Node compatibility.

### TypeScript presence

| Indicator | What It Means |
|---|---|
| `tsconfig.json` | TypeScript project — read for `compilerOptions`, especially `module`, `moduleResolution`, `strict`, `paths`, `target` |
| `tsconfig.*.json` (e.g., `tsconfig.build.json`) | Multiple TS configs — one likely extends another. Read all. |
| `.ts` / `.tsx` files alongside `.js` | Mixed TS/JS — common during migration. Note ratio. |
| `// @ts-check` at top of `.js` files | JavaScript with type-checking enabled via JSDoc |
| `jsconfig.json` | JS-only project but VS Code uses it for IntelliSense (no compile-time checks) |
| `noEmit: true` in tsconfig | TypeScript used only for type-checking; another tool (esbuild/swc/babel) does the actual transpilation |

**Path aliases**: `compilerOptions.paths` in tsconfig defines aliases like `"@/*": ["./src/*"]`. Imports using these aliases (`import x from '@/lib/y'`) won't resolve via standard file-relative reading. Always check `paths` before tracing imports.

### Source layout

```
src/                          # Most common application root
├── index.{ts,js}             # Entry point
├── server.{ts,js}            # Or here, for explicit server bootstrap
├── routes/                   # Express / Fastify / Hono route handlers
├── controllers/              # Some teams; usually paired with routes
├── services/                 # Business logic (in NestJS this is @Injectable)
├── repositories/             # Data access; if missing, repository pattern not used
├── models/ or schemas/       # Validation schemas (Zod, Yup) or ORM models
├── middleware/               # Auth, logging, rate-limit middleware
├── lib/ or utils/            # Helpers
├── config/                   # Configuration loading
└── types/                    # Shared TS types

dist/ or build/ or out/       # Compiled output — skip during analysis
node_modules/                 # Dependencies — skip during analysis
.next/ or .nuxt/              # Framework build caches — skip
```

For monorepos, this layout repeats under `packages/*/src/` or `apps/*/src/` or `services/*/src/`.

---

## 2. Framework Detection and Analysis

### Express (most common server framework)

**Detection**: `express` in dependencies. Look for `app = express()` or `const router = express.Router()`.

**Key patterns**:
- **Routes**: `app.get('/path', handler)`, `app.post(...)`, or `router.use('/api/v1', subRouter)`.
- **Middleware order matters**: Express runs middleware in registration order. `app.use(cors())` before `app.use(bodyParser.json())` etc.
- **Error handling**: Express recognizes middleware with 4 args `(err, req, res, next)` as an error handler. Must be registered LAST.
- **Async errors**: Express 4 does NOT auto-catch errors from async handlers. Look for `express-async-errors`, `asyncHandler`, or manual `try/catch + next(err)`. Express 5 auto-catches.
- **Common middleware**: `helmet`, `cors`, `morgan`, `compression`, `express-rate-limit`, `cookie-parser`, `csurf`, `express-session`, `passport`, `multer` (file uploads).

**Analysis checklist**:
1. Map all routes — search for `\.(get|post|put|patch|delete|all|use)\(` patterns.
2. Identify the middleware stack — order is the request lifecycle.
3. Find the error handler — verify it has 4 args.
4. Check for async-handler wrapping pattern.
5. Look at how `req.user` (or similar) gets populated — that's the auth boundary.

### Fastify

**Detection**: `fastify` in dependencies. `const fastify = Fastify({...})` at bootstrap.

**Key differences from Express**:
- Schema-based: Routes can declare request/response JSON Schema for validation + serialization.
- Plugins: Fastify uses an encapsulation model (`fastify.register(plugin, opts)`). Plugin scope matters.
- Hooks: `onRequest`, `preHandler`, `onResponse`, `onError`, `preValidation`, `preSerialization` — these are the lifecycle.
- Better async handling natively.

**Analysis checklist**:
- Routes: `fastify.get('/path', { schema: {...} }, handler)`. Search for `.route(`, `.get(`, `.post(`.
- Plugins: `fastify-plugin`, `@fastify/cors`, `@fastify/jwt`, `@fastify/swagger`, `@fastify/multipart`.
- Schemas: type-providers (`@fastify/type-provider-typebox`, `@fastify/type-provider-json-schema-to-ts`).

### NestJS (Angular-like, decorator-heavy)

**Detection**: `@nestjs/core`, `@nestjs/common` in dependencies. `@Module`, `@Controller`, `@Injectable` decorators. `main.ts` calls `NestFactory.create(AppModule)`.

**This is the framework most similar to Spring Boot in spirit**: heavy DI, decorator-driven, opinionated module structure.

**Key annotations**:

| Decorator | Layer | Purpose |
|---|---|---|
| `@Module` | Bootstrap | Declares a module's controllers, providers, imports, exports |
| `@Controller('path')` | Web | Group of routes under a path |
| `@Get('/sub')`, `@Post`, `@Put`, `@Delete`, `@Patch` | Web | HTTP route on a controller |
| `@Injectable()` | Service / Repo | DI-managed class |
| `@Inject(TOKEN)` | DI | Inject by token (vs. by type) |
| `@Body()`, `@Param()`, `@Query()`, `@Headers()` | Web | Argument decorators |
| `@UseGuards(JwtAuthGuard)` | Web | Auth check for route/controller |
| `@UseInterceptors()` | Web | Transform request/response |
| `@UsePipes(ValidationPipe)` | Web | Validate input |
| `@CacheKey`, `@CacheTTL` | Cache | From `@nestjs/cache-manager` |

**Important — decorator-generated behavior is invisible in source**: `@Controller`, `@Get`, `@Injectable`, and many others register metadata at module-init time. The visible source doesn't show *how* routes are wired — NestJS reflects on metadata. If you see decorators, the wiring is implicit.

**NestJS modules**: An import graph between `@Module`-decorated classes. Read `AppModule` first; trace its `imports:` array.

**HTTP under the hood**: NestJS sits on Express *or* Fastify (chosen at `NestFactory.create()`). For analysis, the framework-specific middleware applies underneath.

### Koa

**Detection**: `koa` package. `const app = new Koa()`.

- Middleware is a chain of async functions: `async (ctx, next) => { ...; await next(); ... }`.
- The `ctx` object holds request, response, state.
- Used by Strapi (CMS) and Egg.js.

### Hapi

**Detection**: `@hapi/hapi`. `const server = Hapi.server({...})`.

- Schema-first via `joi` (Hapi's sibling library).
- Route options object includes auth strategies, validation, response schemas.
- Less common today but still in production.

### Hono

**Detection**: `hono` in dependencies.

- Edge-runtime-first (Cloudflare Workers, Vercel Edge, Deno, Node).
- Express-like API but **request handlers return `Response` objects** (Web standard).
- Often paired with `@hono/zod-validator` for input validation.
- If you see Hono + `wrangler.toml`, treat as Workers app — Node-specific APIs may not work.

### tRPC

**Detection**: `@trpc/server` in dependencies.

- Procedure-based, not REST-based. Functions defined in routers, called over HTTP under the hood.
- Schema-first via Zod (input + output validation).
- Type-safe client-server contract — frontend imports the *types* (not runtime code) of the backend.
- Routers compose: `router({ user: userRouter, post: postRouter })`.
- Middleware via `t.middleware(...)` chained with `.use()`.

### Next.js (API routes only — backend portion)

**Detection**: `next` in dependencies, `pages/api/` or `app/api/` directories.

- **Pages Router** (`pages/api/foo.ts`): default export is the handler `(req, res) => ...`.
- **App Router** (`app/api/foo/route.ts`): named exports `GET`, `POST`, etc., returning `Response`.
- Edge runtime: `export const runtime = 'edge'` at the top of an API file — restricted runtime.
- Middleware: `middleware.ts` at project root or in `app/` directory.
- `next.config.{js,ts}`: rewrites, headers, image domains, runtime config.

This guide focuses on **backend Node**. Next.js frontend (React) is out of scope here — note its presence but recommend a separate analysis.

### GraphQL servers

**Detection**: One of:

| Library | Indicates |
|---|---|
| `apollo-server` / `@apollo/server` | Apollo Server (most common) |
| `graphql-yoga` | Yoga (lighter, edge-compatible) |
| `mercurius` | Mercurius (Fastify-native) |
| `@nestjs/graphql` | NestJS GraphQL module |
| `type-graphql` | Decorator-driven schema-first |
| `pothos` | Code-first, plugin-rich |

**Analysis**:
- Schema location: `.graphql` files, or inline as template literals.
- Resolvers: search for `resolvers = {...}` object or `@Resolver`/`@Query`/`@Mutation` decorators.
- DataLoader (`dataloader` package) — solves N+1; flag absence.
- Subscriptions: WebSocket transport (`graphql-ws`, legacy `subscriptions-transport-ws`).

### gRPC

**Detection**: `@grpc/grpc-js` (modern) or `grpc` (legacy, deprecated). `.proto` files.

- Server: `server.addService(definition, handlers)`.
- Client: generated stub from proto.
- NestJS has `@nestjs/microservices` for gRPC support.

### WebSockets / Real-time

| Library | Style |
|---|---|
| `ws` | Raw WebSocket library |
| `socket.io` | Event-based, with rooms, automatic reconnection |
| `@socket.io/admin-ui` | Admin dashboard for socket.io |
| `bullmq` / `bull` | Job queues (Redis-backed); not real-time but often in real-time apps |

### Message queues / brokers

| Library | What |
|---|---|
| `kafkajs` | Kafka client — search for `.consumer({...})`, `.producer()` |
| `amqplib` | RabbitMQ low-level client |
| `bullmq` | Redis-backed job queue (most common) |
| `agenda` | MongoDB-backed scheduler |
| `node-rdkafka` | Native Kafka binding (fewer projects) |
| `aws-sdk` SQS client | AWS SQS |
| `@google-cloud/pubsub` | GCP Pub/Sub |

---

## 3. Type System & DI Analysis

### TypeScript-specific patterns

**Strictness**: Read `tsconfig.json` `compilerOptions`:

| Option | What It Means |
|---|---|
| `strict: true` | Enables all strict checks — the project is type-disciplined |
| `strict: false` (default) or unset | TS is informational only; runtime behavior may diverge from types |
| `noImplicitAny` | Implicit `any` is an error |
| `strictNullChecks` | `null`/`undefined` are explicit, not assumed |
| `noUncheckedIndexedAccess` | Array/object indexing returns `T | undefined` |
| `verbatimModuleSyntax` | TS keeps import/export syntax verbatim — affects CJS/ESM interop |
| `moduleResolution: "node16"` / `"nodenext"` / `"bundler"` | How TS resolves imports — `bundler` is the modern choice |
| `paths` | Path aliases — see § 1 |

**Common type patterns**:

| Pattern | What to Look For |
|---|---|
| Branded types | `type UserId = string & { readonly __brand: 'UserId' }` — nominal typing simulation |
| Discriminated unions | `type Result = { ok: true, value: T } | { ok: false, error: E }` |
| `as const` assertions | Compile-time literal types from runtime objects |
| `satisfies` (TS 4.9+) | Type-safe object literals without widening |
| Conditional types | `T extends U ? X : Y` in generics — usually in library code |
| Mapped types | `{ [K in keyof T]: ... }` — type transformations |
| Template literal types | `\`${'get' | 'set'}${Capitalize<K>}\`` — for method-name derivation |

### Dependency injection (where applicable)

Most Node frameworks don't have DI. NestJS is the exception. Other patterns:

| Approach | Detection | Notes |
|---|---|---|
| **NestJS** | `@nestjs/common` | Class-based, decorator-driven DI |
| **InversifyJS** | `inversify` | Decorator-driven DI without NestJS |
| **tsyringe** | `tsyringe` (Microsoft) | Lightweight DI for TS |
| **awilix** | `awilix` | Function-based DI, popular outside NestJS |
| **Manual** | None of the above | Most Node apps just import and instantiate — no DI container |

For non-DI codebases, **trace dependencies via imports**: who imports whom defines the graph. There's no central registry.

### Decorator-aware reading (Node parallel to Lombok-aware reading in Java)

If `experimentalDecorators: true` in `tsconfig.json` and the codebase uses decorators (NestJS, TypeORM, class-validator, type-graphql, tsyringe, routing-controllers), **the visible source is incomplete** — much behavior is generated at runtime via reflection.

| Decorator pattern | Generates / Implies |
|---|---|
| `@Controller('/api')` + `@Get('/x')` (NestJS) | Route registration; no explicit `app.get(...)` call exists |
| `@Injectable()` + constructor params (NestJS) | Constructor-injected DI; the constructor signature IS the dependency list |
| `@Entity()` + `@Column()` (TypeORM, MikroORM) | DB table mapping; column definitions, types, defaults |
| `@IsEmail()`, `@IsString()` (class-validator) | Validation rules attached to a class; applied by `ValidationPipe` |
| `@ObjectType()` + `@Field()` (type-graphql) | GraphQL schema definition |
| `@injectable()` + `@inject()` (InversifyJS, tsyringe) | DI binding |

**Reading rule**: When you see decorators, the file is not the full story. Look at the framework's bootstrap code (`main.ts`, `app.module.ts`) to understand how decorator metadata is consumed.

### ESM vs CommonJS

Critical to know which the project uses — they behave differently at runtime.

| Signal | Interpretation |
|---|---|
| `"type": "module"` in package.json | All `.js` files are ESM |
| `"type": "commonjs"` or unset | All `.js` files are CJS |
| `.mjs` extension | Always ESM regardless of `type` |
| `.cjs` extension | Always CJS regardless of `type` |
| `import` statements with extensions in paths (`./foo.js`) | ESM (Node ESM requires explicit extensions) |
| `require()` calls | CJS |
| `__dirname`, `__filename` used directly | CJS (these don't exist in ESM by default) |
| `import.meta.url` used | ESM |
| `top-level await` | ESM only |

**ESM gotchas to flag**:
- ESM can't synchronously `require` from CJS in some bundler configs.
- `__dirname` replacement in ESM: `import { fileURLToPath } from 'node:url'; const __dirname = path.dirname(fileURLToPath(import.meta.url));`
- Conditional exports in `package.json` (`"exports": { "import": "...", "require": "..." }`) determine what gets loaded.

---

## 4. Control Flow Patterns

### Request lifecycle (Express)

1. Incoming request hits the server (HTTP module).
2. Global middleware runs in registration order: `app.use(...)`.
3. Route matching finds the handler.
4. Per-route middleware runs.
5. Handler runs.
6. If `next(err)` is called or handler throws (async — Express 5+, or with wrapper in 4), error-handling middleware runs.
7. Response sent via `res.send`, `res.json`, `res.status().end()`, or `res.sendStatus()`.

**Common ordering pitfalls to flag**:
- Auth middleware after a route handler → unprotected.
- `bodyParser` after a POST route → `req.body` is `undefined`.
- Error handler before routes → never called.

### Request lifecycle (Fastify)

Lifecycle hooks fire in order:
1. `onRequest`
2. `preParsing`
3. `preValidation`
4. `preHandler`
5. (handler runs)
6. `preSerialization`
7. `onSend`
8. `onResponse`
9. `onError` (if any step throws)

### Request lifecycle (NestJS)

1. Middleware (Express/Fastify middleware bound to NestJS modules).
2. Guards (`@UseGuards`) — auth checks; return `true` to proceed.
3. Interceptors (before) — `intercept(ctx, next)` runs code before `next.handle()`.
4. Pipes — input validation/transformation.
5. Controller method runs.
6. Interceptors (after) — code after `next.handle()`.
7. Exception filters — catch and format errors.

### Async patterns

| Pattern | Indicator | Notes |
|---|---|---|
| Promises | `.then().catch()` | Older style; chained |
| async/await | `async function`, `await` | Modern default |
| Callbacks | `(err, result) => ...` | Legacy; Node's stdlib still has these (`fs.readFile`) |
| Promise.all / allSettled | Parallel awaits | Check for failure handling: `all` rejects on first error; `allSettled` returns array |
| Async iterators | `for await (const x of asyncIterable)` | Streams, paginated APIs |
| Streams | `import { pipeline } from 'node:stream/promises'` | Node streams; backpressure-aware |
| EventEmitter | `extends EventEmitter` or `new EventEmitter()` | Node core pattern; flag if errors emitted but not handled |

### Error handling

- **Unhandled rejection**: A promise rejection with no `.catch` or `try/catch` → `process.on('unhandledRejection', ...)`. Some projects crash on this; some swallow it. Critical to know which.
- **Uncaught exception**: Synchronous throw with no `try/catch` → `process.on('uncaughtException', ...)`. Almost always should crash the process (state may be corrupted); restart via PM2/systemd/k8s.
- **Custom error classes**: `class HttpError extends Error { constructor(public status: number, message: string) {...} }` — search for `extends Error`.

# =================================================================
# === END ESSENTIAL ANALYSIS ======================================
# =================================================================

# =================================================================
# === EXTENDED ANALYSIS (Load on Demand) ==========================
# =================================================================

## 5. Data Layer Analysis

### ORM / Query Builder Detection

| Library | Style | Detection |
|---|---|---|
| **Prisma** | Schema-first ORM, type-safe client | `prisma/schema.prisma`, `@prisma/client` |
| **Drizzle** | TS-first query builder, schema as code | `drizzle-orm`, `drizzle.config.{ts,js}` |
| **TypeORM** | Decorator-based ORM (Java-like) | `typeorm`, `@Entity()` decorators |
| **MikroORM** | Decorator-based, Unit-of-Work pattern | `@mikro-orm/core` |
| **Sequelize** | Class-based, mature | `sequelize`, `Model.init({...})` |
| **Knex** | Query builder only (no ORM) | `knex`, raw query builder |
| **Mongoose** | MongoDB ODM | `mongoose`, `Schema({...})`, `model('Name', schema)` |
| **Kysely** | TS-first query builder, no ORM | `kysely`, type-safe SQL |
| **node-postgres** (`pg`) | Raw driver, parameterized queries | `pg` |

### Prisma deep dive

- **Schema**: `prisma/schema.prisma` defines models, relations, datasource, generator.
- **Migrations**: `prisma/migrations/<timestamp>_name/migration.sql`.
- **Client**: Generated to `node_modules/.prisma/client`. Imported as `import { PrismaClient } from '@prisma/client'`.
- **Common anti-patterns**: Creating a new `PrismaClient` per request (should be singleton); N+1 from forgetting `include` or `select`.

### Drizzle deep dive

- **Schema files**: `src/db/schema.ts` (convention) — tables defined with `pgTable`, `mysqlTable`, `sqliteTable`.
- **Migrations**: `drizzle-kit` generates SQL migrations.
- **Queries**: Two styles — query builder (`db.select().from(users).where(...)`) and relational queries (`db.query.users.findMany({...})`).

### TypeORM deep dive

- **Entities**: `@Entity()` classes with `@PrimaryGeneratedColumn`, `@Column`, `@OneToMany`, `@ManyToOne`, `@JoinColumn`.
- **Repository pattern** or **Active Record**: Check which is in use.
- **Migrations**: CLI-generated; `migrations/` directory.
- **DataSource**: `new DataSource({...})` configures the connection — find this to know DB type.

### Migration tooling

| Tool | Indicator |
|---|---|
| Prisma Migrate | `prisma/migrations/` |
| Drizzle Kit | `drizzle/` folder + `drizzle.config` |
| TypeORM CLI | `migrations/` + `data-source.ts` |
| Knex Migrate | `migrations/` with `up`/`down` exports |
| Umzug | `umzug` package |
| Custom SQL files | `migrations/*.sql` with no tooling — manual application |

### Validation libraries (often serve as runtime schema)

| Library | Style |
|---|---|
| **Zod** | TS-first, infer types from schema (`z.infer<typeof schema>`) |
| **Yup** | Older, also TS support |
| **Joi** | Hapi-affiliated, mature |
| **AJV** | JSON Schema validator (Fastify default) |
| **class-validator** | Decorator-driven (NestJS default) |
| **valibot** | Tree-shakeable Zod alternative |
| **arktype** | TS syntax as schema |

**Important**: In TypeScript codebases that use Zod, the Zod schema is often the **source of truth for both types and runtime validation**. The TS types are derived from the schema (`z.infer<...>`), not the other way around.

### Caching

| Pattern | Indicator |
|---|---|
| In-memory LRU | `lru-cache`, `node-lru-cache` |
| Redis client | `ioredis` (most common, supports cluster), `redis` (official), `node-redis` |
| Memcached | `memjs`, `memcached` |
| NestJS cache | `@nestjs/cache-manager` + `cache-manager` |
| Edge caching | Cloudflare KV (`@cloudflare/workers-types`), Vercel KV |

### Connection pooling

- **`pg` Pool**: `new Pool({...})` — set `max`, `idleTimeoutMillis`.
- **Prisma**: Manages pool internally; configure via `DATABASE_URL` (`?connection_limit=N`).
- **Drizzle / Kysely**: Built on top of `pg`/`mysql2` — pool is configured at the driver level.
- **Serverless gotcha**: In Lambda/Vercel/Workers, connection pools are *per-instance* and can exhaust the DB. Look for connection-pooling proxies (PgBouncer, Prisma Accelerate, Neon's serverless driver).

---

## 6. Design Patterns to Identify

| Pattern | Node Indicators |
|---|---|
| **Module pattern** | CommonJS / ESM modules with private internals exposed via exports |
| **Singleton** | Module-level instance (`export const db = new DB()`); cached across imports |
| **Factory** | Function returning a configured instance: `createServer(opts)` |
| **Middleware chain** | Express/Koa/Fastify middleware — each fn calls `next` |
| **Strategy** | Object/Map keyed by strategy name, dispatched at runtime |
| **Repository** | `XRepository` classes wrapping DB access; common with TypeORM/Prisma |
| **Observer** | `EventEmitter` subclass + `.on('event', listener)` |
| **Decorator** | Function wrapping another function/class; literal `@decorator` syntax (NestJS) |
| **Pub/Sub** | Redis pub/sub, EventEmitter, or in-process bus library (`mitt`, `nanoevents`) |
| **Result type** | `Result<T, E>` discriminated union; libraries `neverthrow`, `oxide.ts` |

---

## 7. Modern Node / TypeScript Features

| Feature | Version | What to Note |
|---|---|---|
| `node:` import prefix | Node 16+ | `import fs from 'node:fs'` — disambiguates from npm packages |
| Built-in test runner | Node 18+ (stable 20+) | `node:test` module — competes with Jest/Vitest |
| `--watch` mode | Node 18+ | `node --watch index.js` — built-in nodemon |
| `--env-file` | Node 20+ | `node --env-file=.env index.js` — built-in dotenv |
| Top-level await | Node 14+ ESM | `await` outside `async function` in ESM modules |
| `Promise.withResolvers()` | Node 22+ | New deferred-pattern primitive |
| `fetch` global | Node 18+ | No need for `node-fetch` |
| WebStreams | Node 18+ | `ReadableStream`, `WritableStream` — interop with Web platform |
| `import.meta.dirname` | Node 20.11+ | Cleaner ESM `__dirname` replacement |
| TypeScript `satisfies` | TS 4.9+ | Type-safe literals |
| TypeScript `using` / explicit resource management | TS 5.2+ | `using db = new Database();` — auto-cleanup |
| `const` type parameters | TS 5.0+ | `function f<const T>(x: T)` — preserves literal types |

---

## 8. Testing Analysis

### Test runner detection

| Runner | Indicator |
|---|---|
| **Jest** | `jest` in devDeps, `jest.config.{js,ts}`, `__tests__/` folders or `*.test.*` files |
| **Vitest** | `vitest`, `vitest.config.{js,ts}` — Vite-powered, ESM-native |
| **Mocha** | `mocha`, `.mocharc.{js,json,yml}` |
| **Node test runner** | `node --test` in scripts, `import { test } from 'node:test'` |
| **AVA** | `ava` in devDeps, `ava` key in package.json |
| **Tap / node-tap** | `tap` |
| **Playwright** | `@playwright/test` — E2E browser testing |
| **Cypress** | `cypress` — E2E browser testing |
| **Supertest** | `supertest` — HTTP integration tests (paired with Jest/Vitest) |

### Test commands

| What | Common commands |
|---|---|
| Unit tests | `npm test` / `pnpm test` / `yarn test` — find in `package.json` `scripts.test` |
| Watch mode | `jest --watch`, `vitest`, `node --test --watch` |
| Coverage | `jest --coverage`, `vitest --coverage`, `c8 node --test` |
| Single test file | `jest path/to/test`, `vitest run path/to/test` |
| Single test by name | `jest -t "test name"`, `vitest -t "test name"` |
| E2E | `npx playwright test`, `npx cypress run`, custom scripts |

### Coverage

- **Istanbul / nyc**: legacy coverage tooling.
- **c8**: native Node V8 coverage, no instrumentation.
- **Jest --coverage** / **Vitest --coverage**: built-in via c8.
- Thresholds: `coverageThreshold` in Jest config, `coverage.thresholds` in Vitest config.

### Mocking

| Approach | How |
|---|---|
| Jest mocks | `jest.mock('path')`, `jest.fn()`, `jest.spyOn(...)` |
| Vitest mocks | `vi.mock('path')`, `vi.fn()`, `vi.spyOn(...)` |
| `sinon` | Cross-runner stubs, spies, fakes |
| MSW (Mock Service Worker) | HTTP-level mocking via fetch interception |
| `nock` | HTTP mocking via `http.request` interception |
| `testcontainers` | Real services in Docker for integration tests |

### Common test types

- **Unit**: pure-function tests, single-module tests with mocks.
- **Integration**: tests touching DB/HTTP/queue — usually with Testcontainers, MSW, or test-specific docker-compose.
- **Contract**: Pact (`@pact-foundation/pact`) for consumer-driven contracts.
- **Snapshot**: `expect(x).toMatchSnapshot()` — verify shape stability.

---

## 9. Security Considerations

### Common Node-specific risks to flag

- **Hardcoded secrets**: API keys, tokens, passwords in source. Grep `password|secret|api[_-]?key|token` against string literals.
- **`eval()` / `new Function(...)`** with user input → RCE.
- **`child_process.exec(...)`** with concatenated user input → shell injection. Prefer `execFile`/`spawn` with arg arrays.
- **`require(userInput)`** or dynamic `import()` with user input → arbitrary module loading.
- **SQL injection via string concatenation**: Look for `query(\`SELECT ... ${userInput}\`)` instead of parameterized `query('SELECT ... $1', [userInput])`.
- **Path traversal**: `fs.readFile(path.join(dir, userInput))` without validation — user can pass `../../etc/passwd`.
- **Prototype pollution**: Deep-merge libraries (`lodash.merge` pre-fix, `defaults-deep` unfixed forks) — search for `__proto__` / `constructor.prototype` writes.
- **Unsafe deserialization**: `JSON.parse(userInput)` is safe; `eval(userInput)`, `node-serialize`'s `unserialize` are not.
- **Missing input validation**: Routes accepting `req.body` without Zod/Yup/Joi/class-validator validation.
- **CORS misconfiguration**: `cors({ origin: '*', credentials: true })` is a contradiction the browser will reject — but `origin: true` (reflect) + `credentials: true` is dangerous.
- **JWT pitfalls**: `none` algorithm allowed, secret stored as a string literal, no audience/issuer validation, infinite expiration.
- **CSRF**: Cookie-based sessions without CSRF token (look for `csurf`, `@fastify/csrf-protection`).
- **Rate limiting**: Absent on login, password reset, expensive endpoints (look for `express-rate-limit`, `@fastify/rate-limit`).
- **Helmet absence**: `helmet` middleware sets defensive headers; flag if missing in Express apps.

---

## 10. Common Anti-Patterns

| Anti-Pattern | Indicators |
|---|---|
| **Synchronous file I/O on request path** | `fs.readFileSync`, `fs.writeFileSync` inside a handler — blocks the event loop |
| **`await` in a loop instead of `Promise.all`** | `for (const x of items) await doThing(x)` when parallelism is safe |
| **Unhandled async errors in Express 4** | Async handlers without `try/catch` or `express-async-errors` wrapper |
| **Forgotten `await`** | `const r = doThing()` where `doThing` returns a promise but result used as plain value |
| **N+1 queries** | `for` loop calling `prisma.user.findUnique` instead of `findMany({ where: { id: { in: ids } } })` |
| **Module-level side effects** | `import './migrations'` that runs migrations on import — surprising at test time |
| **Mixing CJS and ESM without compatibility shims** | `require()` of an ESM-only package — fails at runtime |
| **Process state in module scope** | `let requestCount = 0` at module top — leaks across requests, breaks in serverless |
| **Catching errors only to swallow** | `.catch(() => {})` or `try {...} catch(e) {}` with no logging |
| **`any` escape hatches** | `as any`, `// @ts-ignore`, `// @ts-expect-error` — count them; high density signals weak type discipline |
| **Implicit any from missing TS strict** | `tsconfig` with `strict: false` — types become decorative |
| **Unbounded promise queues** | `Promise.all(thousands of items)` without `p-limit` — exhausts file descriptors / connections |
| **Memory leaks via listener accumulation** | `emitter.on('event', ...)` without `off`/`removeListener` — common in long-running processes |
| **God controllers** | Single file with 30+ routes |
| **Logic in routes** | Express handlers with 100+ lines of business logic instead of delegating to services |

# =================================================================
# === END EXTENDED ANALYSIS =======================================
# =================================================================
