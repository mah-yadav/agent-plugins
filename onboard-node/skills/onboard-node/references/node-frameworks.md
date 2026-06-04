# Node Frameworks, Source Layout & Control Flow

Framework detection, the request lifecycle each framework imposes, the conventional source layout, async patterns, and error handling. Load this when reading a Node codebase's architecture and tracing how a request flows end-to-end (Phase 3.2, control-flow analysis).

Project identity (package manager, Node version, module system, TypeScript, edge runtime) is already detected in Phase 1 and recorded in the ledger — read it there, don't re-derive it here.

## Contents

- **Source layout**
- **Framework detection & analysis** — Express, Fastify, NestJS, Koa, Hapi, Hono, tRPC, Next.js, GraphQL, gRPC, WebSockets, message queues
- **Request lifecycles** — Express, Fastify, NestJS
- **Async patterns**
- **Error handling**

---

## Source layout

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

For monorepos, this layout repeats under `packages/*/src/`, `apps/*/src/`, or `services/*/src/`. Use the package-paths list captured in the ledger.

**Don't guess from file names** — always read content. `auth.ts` could be middleware, a service, a controller, or a config.

---

## Framework detection & analysis

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

**Important — decorator-generated behavior is invisible in source**: `@Controller`, `@Get`, `@Injectable`, and many others register metadata at module-init time. The visible source doesn't show *how* routes are wired — NestJS reflects on metadata. If you see decorators, the wiring is implicit. See `node-type-system.md` § Decorator-aware reading.

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

This plugin focuses on **backend Node**. Next.js frontend (React) is out of scope here — note its presence but recommend a separate analysis.

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

## Request lifecycles

### Express

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

### Fastify

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

### NestJS

1. Middleware (Express/Fastify middleware bound to NestJS modules).
2. Guards (`@UseGuards`) — auth checks; return `true` to proceed.
3. Interceptors (before) — `intercept(ctx, next)` runs code before `next.handle()`.
4. Pipes — input validation/transformation.
5. Controller method runs.
6. Interceptors (after) — code after `next.handle()`.
7. Exception filters — catch and format errors.

---

## Async patterns

| Pattern | Indicator | Notes |
|---|---|---|
| Promises | `.then().catch()` | Older style; chained |
| async/await | `async function`, `await` | Modern default |
| Callbacks | `(err, result) => ...` | Legacy; Node's stdlib still has these (`fs.readFile`) |
| Promise.all / allSettled | Parallel awaits | Check for failure handling: `all` rejects on first error; `allSettled` returns array |
| Async iterators | `for await (const x of asyncIterable)` | Streams, paginated APIs |
| Streams | `import { pipeline } from 'node:stream/promises'` | Node streams; backpressure-aware |
| EventEmitter | `extends EventEmitter` or `new EventEmitter()` | Node core pattern; flag if errors emitted but not handled |

---

## Error handling

- **Unhandled rejection**: A promise rejection with no `.catch` or `try/catch` → `process.on('unhandledRejection', ...)`. Some projects crash on this; some swallow it. Critical to know which.
- **Uncaught exception**: Synchronous throw with no `try/catch` → `process.on('uncaughtException', ...)`. Almost always should crash the process (state may be corrupted); restart via PM2/systemd/k8s.
- **Custom error classes**: `class HttpError extends Error { constructor(public status: number, message: string) {...} }` — search for `extends Error`.
