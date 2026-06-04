# Node Control Flow: request lifecycles, async, error handling

Load this for "walk me through what happens when a request arrives", middleware
ordering, async patterns, and error propagation. For *detecting* the framework,
see **web-frameworks.md**.

## Request lifecycle (Express)

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

## Request lifecycle (Fastify)

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

## Request lifecycle (NestJS)

1. Middleware (Express/Fastify middleware bound to NestJS modules).
2. Guards (`@UseGuards`) — auth checks; return `true` to proceed.
3. Interceptors (before) — `intercept(ctx, next)` runs code before `next.handle()`.
4. Pipes — input validation/transformation.
5. Controller method runs.
6. Interceptors (after) — code after `next.handle()`.
7. Exception filters — catch and format errors.

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

## Error handling

- **Unhandled rejection**: A promise rejection with no `.catch` or `try/catch` → `process.on('unhandledRejection', ...)`. Some projects crash on this; some swallow it. Critical to know which.
- **Uncaught exception**: Synchronous throw with no `try/catch` → `process.on('uncaughtException', ...)`. Almost always should crash the process (state may be corrupted); restart via PM2/systemd/k8s.
- **Custom error classes**: `class HttpError extends Error { constructor(public status: number, message: string) {...} }` — search for `extends Error`.
