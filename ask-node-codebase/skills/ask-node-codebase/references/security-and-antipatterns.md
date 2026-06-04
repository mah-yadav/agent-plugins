# Node Security & Anti-Patterns

Load this for "how is security set up here" and "what smells / anti-patterns are
in this code". This skill answers *how security is configured*, not *whether it
is good* — it is not a security audit.

## Common Node-specific security risks to flag

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

## Common anti-patterns

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
