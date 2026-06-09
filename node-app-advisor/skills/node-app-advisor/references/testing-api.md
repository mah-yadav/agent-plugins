# Testing & API Design Analysis

Audit test-suite quality and API design (REST, RPC, GraphQL). Identify missing coverage, test anti-patterns, flaky tests, API inconsistencies, and contract violations. Every finding needs evidence, severity, and a concrete fix.

**Core principle:** Tests tell you what the code is supposed to do; APIs tell you what the code promises to do. Bad tests give false confidence; bad APIs create integration nightmares.

## Contents

- Step 1 — Test infrastructure assessment
- Step 2 — Test coverage analysis
- Step 3 — Test quality analysis
- Step 4 — API design analysis
- Step 5 — Contract & integration testing
- Step 6 — Compile findings

## Tools

- `Grep` — test functions/assertions, route/procedure definitions, response shapes.
- `Glob` — test files, fixtures, API specs (`**/*.{test,spec}.{js,ts}`, `**/__tests__/**`, `**/openapi*.{yml,json}`).
- `Read` — test files, route handlers/controllers, and API contracts in full.
- `Bash` — run test/coverage commands only if the user asks (otherwise read-only).
- `Agent` (Explore) — fan-out: "find all route/controller handlers and their test counterparts", "find all route registrations and their paths". Re-`Read` leads before citing.
- **Coverage of a function:** `Grep` the export name across test files to see if it's exercised (no `listCodeUsages` tool).

## Step 1 — Test infrastructure assessment

1. **Testing frameworks** (from `package.json` devDependencies):

   | Dependency | Type |
   |---|---|
   | `jest` / `ts-jest` | Test runner (JS/TS) |
   | `vitest` | Test runner (Vite-native, ESM-first) |
   | `mocha` + `chai` | Test runner + assertions |
   | `node:test` / `tap` / `ava` | Built-in / alternative runners |
   | `supertest` | HTTP integration testing for Express/Koa/Fastify |
   | `@nestjs/testing` | NestJS test module / DI in tests |
   | `light-my-request` | Fastify in-process HTTP testing |
   | `testcontainers` | Real DB/broker in containers for integration tests |
   | `msw` / `nock` | HTTP request mocking |
   | `@faker-js/faker` | Test data generation |
   | `sinon` / `jest.mock` / `vi.mock` | Mocking/stubbing/spies |
   | `playwright` / `cypress` | E2E |
   | `@pact-foundation/pact` | Consumer-driven contract testing |
   | `c8` / `nyc` / built-in coverage | Code coverage |
   | `@stryker-mutator/core` | Mutation testing |

2. **Test directory structure** — `__tests__/`, co-located `*.test.ts`/`*.spec.ts`, `test/`, separate `*.e2e-spec.ts` (NestJS), fixtures, test config (`jest.config`, `vitest.config`, test `.env`).
3. **Naming & scripts** — `test`, `test:unit`, `test:e2e`, `test:cov` scripts; whether tests run in CI; coverage thresholds in config.

## Step 2 — Test coverage analysis

### 2.1 Test-to-source mapping

| Source | Expected test | If missing |
|--------|---------------|-----------|
| routes / controllers | route/controller tests | API contract untested — High |
| services / use-cases | service tests | Business logic untested — Critical |
| repositories / data access | repo or integration tests | Data access untested — High |
| middleware / guards / interceptors | unit tests | Cross-cutting logic untested — High |
| validators / schemas | schema tests | Validation untested — High |
| config / bootstrap | smoke test | May fail at startup — Medium |
| auth | security tests | Auth rules untested — Critical |
| error handlers | handler tests | Error responses untested — Medium |

*Detect:* list source files under `src/` and test files; diff to find source modules with no test; count test cases per file.

### 2.2 Critical-path coverage

| Critical path | Verify | If missing |
|---------------|--------|-----------|
| Authentication flows | Tests for login, token issue/verify, middleware/guard | Critical |
| Authorization rules | Tests for role/ownership checks (admin can X, user cannot) | Critical |
| Data mutation endpoints | Tests for POST/PUT/PATCH/DELETE | High |
| Error handling | Tests for the global error handler and error responses | High |
| Validation rules | Tests for schema validation (valid + invalid input) | High |
| Business edge cases | Boundary conditions, null/undefined, empty arrays | High |
| Integration points | External service calls, DB operations, queue jobs | High |
| Async/concurrent paths | Race conditions, retries, idempotency | High |

### 2.3 Coverage metrics (if available)

Read coverage config (`jest`/`vitest` `coverageThreshold`, `c8`/`nyc`). Thresholds set? Enforced in CI? If a coverage report exists, review line/branch coverage per directory. Beware coverage that excludes the hard parts (`collectCoverageFrom` omitting services).

## Step 3 — Test quality analysis

| Anti-pattern | Detection | Severity |
|--------------|-----------|----------|
| No assertions | `test`/`it` blocks with no `expect`/`assert`/`verify` | Critical |
| Missing `await` on async assert | `expect(promise).resolves`/`.rejects` not returned/awaited; un-awaited async in test — false pass | High |
| Testing implementation not behavior | Asserting internal calls/mock invocations rather than outcomes | Medium |
| Over-mocking | Mocking everything (incl. the unit under test) — testing nothing real | Medium |
| Brittle tests | Asserting exact error strings, timestamps, log output, snapshot of huge objects | Medium |
| Missing negative tests | Only happy path; no error/edge/invalid-input cases | High |
| Shared mutable state between tests | Tests depending on order or a shared DB/global not reset | High |
| `.skip`/`.only`/`xit` abuse | Many skipped tests, or a committed `.only` silently disabling the suite | High |
| Flaky indicators | `setTimeout`/`sleep` for timing, real network calls, time/date dependence without fake timers | High |
| Missing cleanup | Creating DB rows/files/servers without `afterEach`/`afterAll` teardown | Medium |
| Real external calls in unit tests | Hitting real HTTP/DB instead of mocks/containers | High |
| Test too long / unfocused | Test cases > 50 lines doing many things | Low |

*Detect:* grep `it(`/`test(` for blocks lacking `expect`; grep `.only`/`.skip`/`xit`/`xdescribe`; grep `setTimeout`/`sleep` in tests; grep `jest.mock`/`vi.mock` count per file; check `resolves`/`rejects` are returned/awaited.

## Step 4 — API design analysis

> Applies to REST (Express/Fastify/Koa/Hono/NestJS/Next route handlers). For **tRPC/GraphQL**, translate: procedure/resolver naming consistency, query-vs-mutation correctness (reads must not mutate), input validation per procedure, and error-shape consistency.

### 4.1 Endpoint consistency

| Issue | Detection | Severity |
|-------|-----------|----------|
| Inconsistent URL patterns | Mix of `/getUser`, `/users/:id`, `/user/fetch` | High |
| Verb in URL | `/createOrder`, `/deleteItem` instead of resource URLs + HTTP verbs | Medium |
| Inconsistent pluralization | Mix of `/user/:id` and `/orders/:id` | Medium |
| Missing resource nesting | `/order-items?orderId=` vs `/orders/:id/items` | Low |
| Inconsistent path-param naming | Mix of `:userId`, `:user_id`, `:id` | Medium |

*Detect:* Explore → find all route registrations and their paths/methods, using the framework's API — **Express/Koa:** `app/router.get/post/put/patch/delete(...)`; **Fastify:** `fastify.route({ method, url })` / `.get/.post(...)`; **NestJS:** `@Get`/`@Post`/`@Put`/`@Patch`/`@Delete` + `@Controller('...')`; **Hono:** `app.get/post(...)`; **Next.js:** exported `GET`/`POST`/... in `route.ts` + the folder path; **tRPC:** `router({ ... })` procedure names. Catalog method/path/input/output; check naming consistency.

### 4.2 HTTP semantics

| Issue | Detection | Severity |
|-------|-----------|----------|
| POST for reads | `POST` used for pure retrieval | High |
| GET with body / side effects | `GET` mutating state or requiring a body | High |
| Wrong status codes | 200 for creation (should be 201) / deletion (204); 200 wrapping an error | Medium |
| Missing status codes | No 404 for not-found, 409 for conflict, 401 vs 403 confusion | Medium |
| Non-idempotent PUT/DELETE | PUT/DELETE that aren't idempotent | High |
| Missing content negotiation / wrong content-type | Returning JSON without `application/json`, or no `consumes` enforcement | Low |

### 4.3 Request/response design

| Issue | Detection | Severity |
|-------|-----------|----------|
| Exposing ORM entities directly | Returning Prisma/TypeORM/Mongoose objects raw (leaks fields like `passwordHash`) | High |
| Over-fetching in responses | Returning whole records incl. sensitive/internal fields (no DTO/serializer/`select`) | High |
| Missing pagination | List endpoints returning unbounded arrays | High |
| Inconsistent error format | Different error JSON shapes across endpoints | High |
| Missing input validation | Body/params/query consumed without a schema (Zod/Joi/class-validator) | High |
| Leaking internal details | Stack traces, internal IDs, DB errors in responses | High |
| No response typing/serialization | Fastify without a response schema; ad-hoc `res.json(anything)` | Low |

### 4.4 Versioning & documentation

| Issue | Detection | Severity |
|-------|-----------|----------|
| No versioning strategy | No URL prefix (`/v1/`), header, or media-type versioning | Medium |
| Missing API documentation | No OpenAPI/Swagger spec (`@nestjs/swagger`, `fastify-swagger`, `tsoa`), no tRPC types exported | Medium |
| Docs out of sync | Spec doesn't match actual routes (spot-check) | High |
| No deprecation strategy | Old endpoints with no deprecation/sunset signal | Low |

## Step 5 — Contract & integration testing

| Issue | Detection | Severity |
|-------|-----------|----------|
| No HTTP-layer integration tests | Handlers only unit-tested with mocked services; real HTTP path (routing, middleware, validation) untested | High |
| Missing framework integration tests | No `supertest`/`light-my-request`/`@nestjs/testing` e2e exercising the real app | High |
| No contract tests | No Pact/consumer-driven contracts for inter-service comms | Medium |
| External services not mocked | Integration tests hitting real third-party APIs (`msw`/`nock` absent) | High |
| No real-DB strategy | SQLite/in-memory mock in tests while prod uses Postgres/Mongo — behavior drift; or no Testcontainers | Medium |
| Tests share prod-like resources | Tests pointing at shared/staging DBs | High |

## Step 6 — Compile findings

For each finding: **Category** (Coverage / Test Quality / API Design / API Contract / Integration) · **Severity** · **Description** · **Evidence** (`file:line`, test names, endpoint details) · **Impact** (reliability, maintainability, consumer experience) · **Recommended fix** (test to add or API change, with examples) · **Effort** (XS/S/M/L/XL). Present the compiled findings.
