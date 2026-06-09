# Testing & API Design Analysis

Audit test-suite quality and REST API design. Identify missing coverage, test anti-patterns, flaky tests, API inconsistencies, and contract violations. Every finding needs evidence, severity, and a concrete fix.

**Core principle:** Tests tell you what the code is supposed to do; APIs tell you what the code promises to do. Bad tests give false confidence; bad APIs create integration nightmares.

## Contents

- Step 1 — Test infrastructure assessment
- Step 2 — Test coverage analysis
- Step 3 — Test quality analysis
- Step 4 — API design analysis
- Step 5 — Contract & integration testing
- Step 6 — Compile findings

## Tools

- `Grep` — test annotations, assertions, API mappings, response types.
- `Glob` — test files, test resources, API specs (`src/test/**`, `**/*Test.java`, `**/openapi*.yml`).
- `Read` — test classes, controllers, and API contracts in full.
- `Bash` — run test/coverage commands only if the user asks (otherwise read-only).
- `Agent` (Explore) — fan-out: "find all controller classes and their test counterparts", "find all `@*Mapping` annotations and their URLs". Re-`Read` leads before citing.
- **Coverage of a method:** `Grep` the method name across `src/test` to see if it's exercised (no `listCodeUsages` tool).

## Step 1 — Test infrastructure assessment

1. **Testing frameworks** (from the build file):

   | Dependency | Type |
   |---|---|
   | `junit-jupiter` / `junit-vintage` | Unit testing (JUnit 5 / 4) |
   | `mockito-core` / `mockito-junit-jupiter` | Mocking |
   | `spring-boot-starter-test` | Spring Boot test support (`@SpringBootTest`, `@WebMvcTest`) |
   | `micronaut-test-junit5` | Micronaut test support (`@MicronautTest`) |
   | `quarkus-junit5` / `rest-assured` | Quarkus test support (`@QuarkusTest`, `@QuarkusIntegrationTest`) |
   | `testcontainers` | Integration testing with containers |
   | `rest-assured` | API testing |
   | `wiremock` | HTTP mock server |
   | `archunit-junit5` | Architecture tests |
   | `cucumber-java` / `cucumber-spring` | BDD |
   | `assertj-core` | Fluent assertions |
   | `awaitility` | Async testing |
   | `jacoco-maven-plugin` | Code coverage |
   | `pitest-maven` | Mutation testing |
   | `spring-cloud-contract` / `pact-jvm` | Contract testing |

2. **Test directory structure** — `src/test/java/`, `src/test/resources/`, `src/integration-test/` or `src/it/`, BDD `features/`, test config (`application-test.yml`).
3. **Naming conventions** — `*Test` vs `*Tests` vs `*IT` vs `*Spec`; Surefire/Failsafe include patterns.

## Step 2 — Test coverage analysis

### 2.1 Test-to-source mapping

| Source | Expected test | If missing |
|--------|---------------|-----------|
| `controller/` | controller tests | API contract untested — High |
| `service/` | service tests | Business logic untested — Critical |
| `repository/` | repo or integration tests | Data access untested — High |
| `config/` | config tests | May fail at startup — Medium |
| `security/` | security tests | Security rules untested — Critical |
| `exception/` | handler tests | Error responses untested — Medium |

*Detect:* list packages under `src/main/java` and `src/test/java`; diff to find source packages with no test package; count test methods per test class.

### 2.2 Critical-path coverage

| Critical path | Verify | If missing |
|---------------|--------|-----------|
| Authentication flows | Tests for filter chain, login, token validation | Critical |
| Authorization rules | Tests for role-based access (admin can X, user cannot) | Critical |
| Data mutation endpoints | Tests for POST/PUT/DELETE | High |
| Error handling | Tests for exception handlers and error responses | High |
| Validation rules | Tests for `@Valid` constraints and custom validators | High |
| Business edge cases | Boundary conditions, null inputs, empty collections | High |
| Integration points | External service calls, database operations | High |

### 2.3 Coverage metrics (if available)

Read JaCoCo plugin config — thresholds? Enforced in CI (`<rule>` minimums)? If coverage reports exist, review line/branch coverage per package.

## Step 3 — Test quality analysis

| Anti-pattern | Detection | Severity |
|--------------|-----------|----------|
| No assertions | `@Test` methods with no `assert*()`/`verify()`/assertion call | Critical |
| Single assert per complex flow | Long tests with one assertion at the end | Medium |
| Testing implementation not behavior | Verifying internal calls rather than outcomes | Medium |
| Mocking everything | > 3 mocked dependencies, testing nothing real | Medium |
| Brittle tests | Asserting exact error messages, log output, or system time | Medium |
| Missing negative tests | Only happy-path, no error/edge cases | High |
| Test data coupling | Tests depending on shared mutable state from other tests | High |
| `@Disabled`/`@Ignore` abuse | Many disabled tests without explanation | High |
| Flaky indicators | `Thread.sleep()`, time-dependent assertions, order dependence | High |
| Missing cleanup | Creating external resources without `@AfterEach` cleanup | Medium |
| Test method too long | `@Test` methods > 50 lines | Low |

*Detect:* grep `@Test` (assertions present?), `@Disabled`/`@Ignore` (count + comments), `Thread.sleep` in tests, `@Mock`/`Mockito.mock` (count per class); check method names describe behavior vs. `test1()`.

## Step 4 — API design analysis

### 4.1 Endpoint consistency

| Issue | Detection | Severity |
|-------|-----------|----------|
| Inconsistent URL patterns | Mix of `/getUser`, `/users/{id}`, `/user/fetch` | High |
| Verb in URL | `/createOrder`, `/deleteItem` instead of resource URLs | Medium |
| Inconsistent pluralization | Mix of `/user/{id}` and `/orders/{id}` | Medium |
| Missing resource nesting | `/order-items?orderId=` vs `/orders/{id}/items` | Low |
| Inconsistent path-param naming | Mix of `{userId}`, `{user_id}`, `{id}` | Medium |

*Detect:* Explore subagent → find all endpoint mappings and their URL patterns, using the framework's annotations — **Spring MVC/WebFlux:** `@RequestMapping`/`@GetMapping`/`@PostMapping`/`@PutMapping`/`@DeleteMapping`/`@PatchMapping`; **Micronaut:** `@Controller` + `@Get`/`@Post`/`@Put`/`@Delete`/`@Patch`; **Quarkus / JAX-RS:** `@Path` + `@GET`/`@POST`/`@PUT`/`@DELETE`/`@PATCH`. Catalog method/URL/params/return type; check naming consistency. (The Spring-named rows in the HTTP-semantics and request/response tables below apply equally — translate the annotation, keep the rule.)

### 4.2 HTTP semantics

| Issue | Detection | Severity |
|-------|-----------|----------|
| POST for reads | `@PostMapping` used for retrieval | High |
| GET with body | `@GetMapping` with `@RequestBody` | High |
| Wrong status codes | 200 for creation (should be 201) / deletion (should be 204) | Medium |
| Missing status codes | No 404 for not-found, no 409 for conflict | Medium |
| Missing idempotency | Non-idempotent PUT/DELETE | High |
| Missing content negotiation | No `produces`/`consumes` | Low |

### 4.3 Request/response design

| Issue | Detection | Severity |
|-------|-----------|----------|
| Exposing entities directly | Controllers returning JPA `@Entity` objects | High |
| Missing pagination | List endpoints returning unbounded collections | High |
| Inconsistent error format | Different error structures across endpoints | High |
| Missing field validation | `@RequestBody` without `@Valid` | High |
| Leaking internal details | Stack traces, internal IDs, DB column names in responses | High |
| Missing HATEOAS | Claims REST Level 3 but has no hypermedia links | Low |

### 4.4 Versioning & documentation

| Issue | Detection | Severity |
|-------|-----------|----------|
| No versioning strategy | No URL prefix (`/v1/`), header, or media-type versioning | Medium |
| Missing API documentation | No OpenAPI/Swagger annotations or spec | Medium |
| Docs out of sync | API docs don't match actual endpoints (spot-check) | High |
| No deprecation strategy | Old endpoints with no `@Deprecated`/sunset headers | Low |

## Step 5 — Contract & integration testing

| Issue | Detection | Severity |
|-------|-----------|----------|
| No controller/integration tests | Controllers only unit-tested with mocked services; real HTTP untested | High |
| Missing framework-context REST tests | No `@WebMvcTest`/`@SpringBootTest` (Spring), `@MicronautTest` (Micronaut), or `@QuarkusTest` (Quarkus) exercising the real HTTP layer | High |
| No contract tests | No Spring Cloud Contract / Pact for inter-service comms | Medium |
| Missing external mocking | Integration tests hitting real external services | High |
| No test DB strategy | H2 in tests while prod uses PostgreSQL/MySQL — behavior drift | Medium |
| Missing Testcontainers | Integration tests without containerized dependencies | Medium |

## Step 6 — Compile findings

For each finding: **Category** (Coverage / Test Quality / API Design / API Contract / Integration) · **Severity** · **Description** · **Evidence** (`file:line`, test class names, endpoint details) · **Impact** (reliability, maintainability, consumer experience) · **Recommended fix** (test to add or API change, with examples) · **Effort** (XS/S/M/L/XL). Present the compiled findings.
