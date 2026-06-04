# Node Design Patterns & Modern Node/TS Features

Load this for "what patterns/idioms does this code use" and "is this using modern
Node/TS features". For anti-patterns and smells, see
**security-and-antipatterns.md**.

## Design patterns to identify

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
