# Node Design Patterns, Modern Features & Anti-Patterns

Design patterns to recognize, modern Node/TypeScript features and the versions that introduced them, and code-level anti-patterns to flag. Load this when documenting design idioms (Phase 3.2 § 2.5) or calling out tech debt.

These are *code-level* anti-patterns (performance, correctness, type discipline). Security-specific risks (injection, secrets, auth misconfig) live in `area-security.md`.

## Contents

- **Design patterns**
- **Modern Node / TypeScript features**
- **Anti-patterns**

---

## Design patterns

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

## Modern Node / TypeScript features

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

## Anti-patterns

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
