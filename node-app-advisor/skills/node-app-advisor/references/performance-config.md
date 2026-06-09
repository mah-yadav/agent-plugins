# Performance & Configuration Analysis

Find performance bottlenecks, resource misconfiguration, missing optimizations, and configuration-management issues. Every finding needs evidence, severity, and a concrete fix.

**Core principle:** In Node.js, performance is mostly about the event loop. A single blocking operation, an unbounded query, or a leaked listener degrades *every* concurrent request — Node has one main thread, so one slow synchronous call stalls the whole process. Most production issues stem from event-loop blocking, data-access patterns, missing caching, and config that works in dev but fails in production.

## Contents

- Step 1 — Event loop & async correctness
- Step 2 — Data access performance (N+1, queries, connections)
- Step 3 — Caching
- Step 4 — Memory & resource management
- Step 5 — Configuration management
- Step 6 — Observability & logging
- Step 7 — Resilience & fault tolerance
- Step 8 — Compile findings

## Tools

- `Grep` — sync calls, query/cache/pool config, async patterns, listeners.
- `Glob` — config files (`**/*.config.{js,ts}`, `**/.env*`, `**/ormconfig.*`, `prisma/schema.prisma`).
- `Read` — data-access code and config files in full.
- `Bash` — read-only dependency/build analysis if needed.
- `Agent` (Explore) — fan-out: "find all DB queries inside loops", "find all `*Sync(` calls", "find all `.on(` event registrations". Re-`Read` leads before citing.
- **Usages:** `Grep` the function name to find call sites (no `listCodeUsages` tool).

## Step 1 — Event loop & async correctness

The Node-specific heart of performance. The main thread is shared by all requests — synchronous work and unhandled async bugs hurt everyone.

| Issue | Detection | Severity |
|-------|-----------|----------|
| Blocking sync I/O on hot path | `readFileSync`/`writeFileSync`/`existsSync`/`execSync` inside request handlers (fine at startup) | High |
| CPU-bound work on main thread | Heavy crypto, JSON of huge payloads, image/PDF processing, big loops in handlers — no worker thread | High |
| Synchronous crypto | `crypto.pbkdf2Sync`/`scryptSync`/`bcrypt.hashSync` in request path (use async variants) | High |
| Blocking JSON | `JSON.parse`/`stringify` on large untrusted bodies with no size cap | Medium |
| Floating promises | `async` calls not `await`ed or `.catch`ed — silent failures, lost errors | High |
| `await` in a loop (serial) | Sequential `await` where `Promise.all` would parallelize independent work | Medium |
| Unbounded `Promise.all` | `Promise.all` over a user-sized array — no concurrency cap (resource exhaustion) | Medium |
| Missing async error handling | Async Express handlers without try/catch or an async-error wrapper (unhandled rejection → crash) | High |
| Sync in streams/transforms | Blocking work inside stream `transform`/`data` handlers | Medium |

*Detect:* grep `Sync(` (`readFileSync`, `execSync`, `hashSync`, `pbkdf2Sync`); grep `for`/`while` blocks containing `await`; grep `async (req` handlers lacking try/catch; check for an async-handler wrapper (`express-async-errors`, NestJS handles this) ; look for `worker_threads` usage where CPU work exists.

## Step 2 — Data access performance

The #1 source of production latency after event-loop blocking.

### 2.1 N+1 queries

| Issue | Detection | Severity |
|-------|-----------|----------|
| Query inside a loop | `await repo.find/findById` / `model.findOne` inside `for`/`map`/`forEach` | Critical |
| Missing eager loading | Lazy relations accessed per-row instead of `include`/`relations`/`populate`/`with` | High |
| Awaiting in a `.map` without batching | `await Promise.all(items.map(i => db.query(...)))` firing N parallel queries instead of one `IN`/join | High |
| ORM lazy relations | TypeORM lazy relations / Sequelize separate lookups not eager-loaded | High |
| No dataloader for GraphQL | GraphQL resolvers issuing per-field queries with no batching/`DataLoader` | High |

*Detect:* Explore → "find DB calls inside loops/maps"; check Prisma `include`, TypeORM `relations`/`leftJoinAndSelect`, Sequelize `include`, Mongoose `populate`; for GraphQL grep `DataLoader`.

### 2.2 Query optimization

| Issue | Detection | Severity |
|-------|-----------|----------|
| Over-fetching | `SELECT *` / fetching full documents when a few fields are needed (no `select`/projection) | Medium |
| Missing pagination | `findMany()`/`find()`/`findAll()` with no `take`/`limit` on large tables | High |
| Missing indexes | Queries filtering/sorting on un-indexed columns (check schema) | Medium |
| Counting via fetch-all | `results.length` after loading all rows instead of a `count` query | Medium |
| Cartesian explosion | Joining/`include` of multiple one-to-many relations in one query | High |

### 2.3 Database connection management

| Issue | Detection | Severity |
|-------|-----------|----------|
| New client per request | `new PrismaClient()`/`new Pool()`/`createConnection()` inside a handler instead of a shared singleton | Critical |
| No connection pooling | Raw driver without a pool; serverless without a pooler (PgBouncer/Prisma Data Proxy) | High |
| Pool too small / default | `pg` Pool `max` at default (10) under high concurrency; no tuning | Medium |
| Missing query/connection timeout | No `statement_timeout`/`connectionTimeoutMillis`/query timeout | Medium |
| Leaked connections | `pool.connect()` without releasing in a `finally` | High |
| Long transactions | Transaction spanning external HTTP/await-heavy work, holding a connection | High |

*Detect:* grep `new PrismaClient`/`new Pool`/`createConnection`/`mongoose.connect` and confirm it's module-level (singleton), not per-request; read pool config; grep `.connect()` paired with `.release()`/`finally`.

## Step 3 — Caching

| Issue | Detection | Severity |
|-------|-----------|----------|
| No caching layer | No Redis/`lru-cache`/`node-cache` where expensive repeated reads exist | Medium |
| Missing cache on hot paths | Frequently called expensive queries/computations uncached | High |
| Cache without TTL | In-memory cache (`Map`/object) that only grows — unbounded memory | High |
| Stale-on-write | Data-modifying paths not invalidating/updating the cache | High |
| In-memory cache across instances | Process-local cache assumed shared across replicas (incoherent in horizontal scale) | Medium |
| Caching per-user data globally | Caching user-specific data under a shared key | High |
| Missing cache metrics | No hit/miss visibility | Low |
| Unbounded `Map` as cache | `const cache = new Map()` at module scope with no eviction (also a leak — Step 4) | High |

*Detect:* grep `lru-cache`/`node-cache`/`ioredis`/`redis`; grep module-level `new Map()`/`{}` used as a cache; check TTL/eviction config; find hot uncached DB-backed functions.

## Step 4 — Memory & resource management

| Issue | Detection | Severity |
|-------|-----------|----------|
| Event-listener leaks | `.on(...)` / `addEventListener` / `emitter.on` added per request without removal; `MaxListenersExceeded` | High |
| Unbounded module-level collections | `const x = []`/`new Map()` at module scope that grows without bound | High |
| Loading whole tables into memory | `findMany()`/`find().toArray()` of large tables into an array | High |
| Unclosed streams/handles | File/network streams/DB cursors not closed or piped to completion | High |
| Buffering large responses | Building a huge string/Buffer instead of streaming the response | Medium |
| Closures capturing big objects | Long-lived callbacks/timers retaining large scopes | Medium |
| `setInterval` never cleared | Timers created and never `clearInterval`'d (leak + duplicate work) | Medium |
| Missing backpressure | Piping fast source to slow sink without honoring `drain`/stream backpressure | Medium |
| String concat in hot loops | Building big strings with `+=` instead of array `join`/streams | Low |

*Detect:* grep `.on(`/`addListener` in request scope; grep module-level mutable `Map`/array/object; grep `createReadStream`/`createWriteStream` without `.close()`/pipe completion; grep `setInterval` without `clearInterval`; check for `--max-old-space-size` / container memory limits.

## Step 5 — Configuration management

| Issue | Detection | Severity |
|-------|-----------|----------|
| Hardcoded values | URLs/ports/hosts/timeouts in source instead of env/config | High |
| Unvalidated env | `process.env.X` read ad-hoc with no schema validation (Zod/envalid) — missing var fails at runtime | High |
| Secrets in committed config | Credentials in `*.config.ts`/committed `.env` instead of injected env/secret manager | Critical |
| No environment separation | Same config for dev/prod; no `NODE_ENV` branching | Medium |
| `NODE_ENV` not `production` | Prod running without `NODE_ENV=production` (disables framework optimizations) | High |
| Production-unsafe defaults | ORM `synchronize: true` (TypeORM) / auto-migrate in prod; CORS `*`; debug logging on | Critical |
| Missing timeouts on outbound calls | `fetch`/`axios`/`got` without a timeout (also Resilience) | High |
| No graceful shutdown | No `SIGTERM` handler closing the server, DB pool, and in-flight requests | High |
| Missing health/readiness endpoint | No `/health` / `/ready` for the orchestrator | Medium |

*Detect:* read all config and `.env*`; grep `process.env.` scattered through source vs. a central validated config; grep `synchronize: true`; grep `process.on('SIGTERM'` / `SIGINT`; check outbound-client timeout options.

## Step 6 — Observability & logging

| Issue | Detection | Severity |
|-------|-----------|----------|
| `console.log` as logging | `console.log`/`console.error` instead of a real logger (pino/winston) — no levels, no structure | Medium |
| No structured logging | Plain-string logs instead of JSON key/value (hard to query in aggregation) | Medium |
| Missing correlation/request IDs | No request-ID propagation (`AsyncLocalStorage`, `cls-hooked`, `pino-http` `reqId`) | High |
| Sensitive data in logs | Passwords/tokens/PII/full `req.body`/`authorization` logged | Critical |
| Synchronous logging on hot path | `console.log` to stdout sync, or a transport that blocks the event loop | Medium |
| Missing metrics | No `prom-client`/OpenTelemetry metrics for request rate/latency/errors | Medium |
| Missing distributed tracing | No OpenTelemetry/`@opentelemetry/*` spans across services | Medium |
| No health checks for deps | `/health` not actually checking DB/cache/broker reachability | High |
| Logs not to stdout/stderr | Logging to local files instead of stdout in containers | Medium |
| Missing audit logging | No who-did-what logging (auth events, data mutations, admin actions) | High |
| Unhandled-rejection / uncaught handlers missing | No `process.on('unhandledRejection'/'uncaughtException')` (silent crashes / zombie state) | High |

*Detect:* grep `console.log`/`console.error` count; grep logger deps (`pino`/`winston`/`bunyan`); grep `AsyncLocalStorage`/`reqId`; grep `prom-client`/`@opentelemetry`; grep `process.on('unhandledRejection'`.

## Step 7 — Resilience & fault tolerance

Most relevant to production-readiness reviews and any app that calls external services (HTTP, DB, broker, third-party APIs). A slow or failing dependency should degrade one call, not stall the event loop and cascade into an outage.

| Issue | Detection | Severity |
|-------|-----------|----------|
| Missing timeouts on outbound calls | `fetch`/`axios`/`got`/`undici` with no timeout (default fetch never times out) | High |
| No retry on transient failures | Idempotent external calls with no retry (`p-retry`, axios-retry, `cockatiel`) | Medium |
| Retry without backoff/jitter | Fixed/zero-delay retries — thundering herd / retry storms | Medium |
| Retrying non-idempotent calls | Retry wrapping a POST/charge/send with no idempotency key | High |
| No circuit breaker | Repeated calls to a flaky dependency with no breaker (`opossum`, `cockatiel`) | Medium |
| Unbounded outbound concurrency | No `p-limit`/semaphore on calls to a capacity-limited dependency | Medium |
| Missing fallback / graceful degradation | No fallback path when a non-critical dependency is down | Medium |
| Job processing without retries/DLQ | BullMQ/queue consumers with no retry policy or dead-letter handling | Medium |
| No backpressure on queues | Producers outpacing consumers with no rate control | Medium |

*Detect:* check whether `p-retry`/`axios-retry`/`opossum`/`cockatiel`/`p-limit` are declared in `package.json`; for each outbound client confirm a timeout and at least one resilience primitive; for queue workers check retry/attempts/backoff config. Absence on a service-to-service hot path is itself the finding.

## Step 8 — Compile findings

For each finding: **Category** (Event Loop / Data Access / Caching / Memory / Configuration / Observability / Resilience) · **Severity** · **Description** · **Evidence** (`file:line` + snippet) · **Impact** (what happens under load / in production) · **Recommended fix** (before/after) · **Effort** (XS/S/M/L/XL). Present the compiled findings.
