# Performance & Configuration Analysis

Find performance bottlenecks, resource misconfiguration, missing optimizations, and configuration-management issues. Every finding needs evidence, severity, and a concrete fix.

**Core principle:** Performance problems are architecture problems discovered under load. Most stem from data-access patterns, resource-pool sizing, missing caching, and config that works in dev but fails in production.

## Contents

- Step 1 — Data access performance (N+1, queries, connections)
- Step 2 — Caching
- Step 3 — Thread pool & async
- Step 4 — Memory, resources & thread safety
- Step 5 — Configuration management
- Step 6 — Observability & logging
- Step 7 — Resilience & fault tolerance
- Step 8 — Compile findings

## Tools

- `Grep` — query/cache/pool/thread annotations and config keys.
- `Glob` — config files (`**/application*.yml`, `**/application*.properties`, `**/*.xml`).
- `Read` — data-access code and config files in full.
- `Bash` — read-only dependency/build analysis if needed.
- `Agent` (Explore) — fan-out: "find all `@Query` annotations", "find all `@Cacheable`". Re-`Read` leads before citing.
- **Usages:** `Grep` the method name to find call sites (no `listCodeUsages` tool).

## Framework equivalents

The detection tables below name Spring symbols because they're the most common, but the *concepts* are framework-neutral. On a Micronaut, Quarkus, or plain-Java app, translate before you grep — don't report "no caching" just because `@Cacheable` is absent.

| Concern | Spring Boot | Micronaut | Quarkus | Plain Java |
|---|---|---|---|---|
| Connection pool | `spring.datasource.hikari.*` | `datasources.*.maximum-pool-size` | `quarkus.datasource.jdbc.max-size` / `min-size` | Hikari/Tomcat/DBCP config directly |
| Caching | `@Cacheable`/`@CacheEvict`/`@EnableCaching` | `@Cacheable`/`@CachePut`/`@CacheInvalidate` (`io.micronaut.cache`) | `@CacheResult`/`@CacheInvalidate` (MicroProfile) | manual Caffeine/EhCache |
| Async / thread pool | `@Async` + `ThreadPoolTaskExecutor` | `@ExecuteOn(TaskExecutors.IO)`, `@Async` | `@Blocking`/`@NonBlocking`, `ManagedExecutor` | `ExecutorService`/`ThreadPoolExecutor` |
| Transactions | `@Transactional` (Spring) | `@Transactional` (`io.micronaut.tx`/jakarta) | `@Transactional` (JTA) | manual tx / `EntityTransaction` |
| HTTP server threads | `server.tomcat.threads.*` | `micronaut.server.netty.*` (event loop) | `quarkus.thread-pool.*`, Vert.x event loop | embedded-server config |
| Metrics | Micrometer + actuator | `micronaut-micrometer`, `/metrics` | MicroProfile Metrics / Micrometer, `/q/metrics` | Dropwizard / manual |
| Health | actuator `HealthIndicator` | `HealthIndicator`, `/health` | MicroProfile Health `@Liveness`/`@Readiness`, `/q/health` | custom |
| Config profiles | `application-{profile}.yml` | `application-{env}.yml`, `MICRONAUT_ENVIRONMENTS` | `%{profile}.`-prefixed props | env vars / props |

**Reactive runtimes** (Spring WebFlux, Micronaut/Netty, Quarkus reactive): blocking I/O on an event-loop thread is the analog of servlet thread starvation (Step 3) — flag blocking JDBC/`RestTemplate` calls inside reactive (`Mono`/`Flux`/`Uni`) chains.

## Step 1 — Data access performance

The #1 source of production performance problems in Java apps.

### 1.1 N+1 queries

| Issue | Detection | Severity |
|-------|-----------|----------|
| Eager collection fetching | `@OneToMany(fetch = EAGER)`, `@ManyToMany(fetch = EAGER)` | High |
| Lazy loading without fetch join | Iterating lazy collections in the service layer with no `JOIN FETCH` | High |
| Missing `@EntityGraph` | Repo methods whose associations are accessed later without `@EntityGraph` | Medium |
| Repository call in a loop | `repository.findById()` inside `for`/`forEach` | Critical |
| Missing batch fetching | No `@BatchSize` or `hibernate.default_batch_fetch_size` for lazy collections | Medium |

*Detect:* grep `@OneToMany`/`@ManyToMany`/`@ManyToOne` and check fetch type; find repo calls inside loops; check `application.yml` for `spring.jpa.properties.hibernate.default_batch_fetch_size`; look for `@EntityGraph`.

### 1.2 Query optimization

| Issue | Detection | Severity |
|-------|-----------|----------|
| SELECT * patterns | Fetching full entities when a few fields are needed | Medium |
| Missing pagination | `findAll()` without `Pageable` on large tables | High |
| Missing projections | No DTO projections for read-only queries | Medium |
| Cartesian product | `JOIN FETCH` on multiple collections at once | High |
| Missing indexes | Queries on frequently filtered columns without `@Index` | Medium |

### 1.3 Database connection management

| Issue | Detection | Severity |
|-------|-----------|----------|
| Default pool size | No HikariCP/pool config (defaults to 10) | Medium |
| Pool too small | `maximumPoolSize` < expected concurrent requests | High |
| Missing connection timeout | No `connectionTimeout` | Medium |
| Missing validation query | No `connectionTestQuery` for health | Low |
| Long transactions | `@Transactional` on controllers or spanning external HTTP calls | High |
| Missing read-only transactions | Read operations not marked `@Transactional(readOnly = true)` | Medium |

*Detect:* read `spring.datasource.hikari.*`; grep `@Transactional` on controllers and on methods that make external HTTP calls.

## Step 2 — Caching

| Issue | Detection | Severity |
|-------|-----------|----------|
| No caching layer | No `@EnableCaching`, no cache deps (Caffeine/Redis/EhCache) | Medium |
| Missing cache on hot paths | Frequently called expensive queries uncached | High |
| Cache without TTL | `@Cacheable` with no expiration — grows unbounded | High |
| Cache without eviction | Data-modifying methods missing `@CacheEvict`/`@CachePut` | High |
| Caching mutable objects | Cached objects mutated after retrieval | High |
| Over-caching | Caching frequently changing or user-specific data | Medium |
| Missing cache metrics | No hit/miss monitoring | Low |
| String key collisions | `@Cacheable` simple string keys that could collide across methods | Medium |

*Detect:* grep `@EnableCaching`, `@Cacheable`/`@CacheEvict`/`@CachePut`; read cache config (Caffeine spec, Redis TTL, EhCache); find hot uncached DB-backed methods.

## Step 3 — Thread pool & async

| Issue | Detection | Severity |
|-------|-----------|----------|
| Default thread pool | `@Async` without a custom `TaskExecutor` (uses unbounded `SimpleAsyncTaskExecutor`) | High |
| Missing rejection policy | Pool without `RejectedExecutionHandler` | Medium |
| Unbounded queue | Pool with an unbounded `LinkedBlockingQueue` | High |
| Blocking in async | `@Async` methods doing blocking I/O without reactive streams | Medium |
| Missing async exception handling | No `AsyncUncaughtExceptionHandler` | Medium |
| Thread pool per method | Many small pools instead of properly sized shared ones | Low |
| Servlet thread starvation | Long operations on Tomcat/Jetty request threads without `@Async` offloading | High |

*Detect:* grep `@Async` (and `@EnableAsync`), `ThreadPoolTaskExecutor`/`TaskExecutor` beans; check `corePoolSize`/`maxPoolSize`/`queueCapacity` and `server.tomcat.threads.*`.

## Step 4 — Memory, resources & thread safety

| Issue | Detection | Severity |
|-------|-----------|----------|
| Unclosed resources | Streams/connections/readers without try-with-resources | High |
| String concatenation in loops | `String +=` in loops instead of `StringBuilder` | Medium |
| Large in-memory collections | Loading whole tables into lists (`findAll()` unbounded) | High |
| Static mutable collections | `static List/Map/Set` without size limits | High |
| Unclosed result streams | Java Streams from DB results not closed | High |
| Excessive object creation | `new SimpleDateFormat()` etc. per call in tight loops | Medium |
| Missing JVM tuning | No `-Xmx`/`-Xms`/`-XX:MaxMetaspaceSize` in startup scripts/Dockerfiles | Medium |

*Detect:* grep `new FileInputStream`/`new BufferedReader` (try-with-resources?), `static .*(List|Map|Set)`, `findAll()`; check Dockerfile/startup scripts for JVM flags.

### 4.1 Thread safety

Singleton-scoped beans (the default for `@Service`/`@Component`/`@Singleton`/`@ApplicationScoped`) are shared across all request threads — mutable instance state on them is a data race.

| Issue | Detection | Severity |
|-------|-----------|----------|
| Mutable state on a singleton bean | Non-`final` instance fields mutated by request methods on a `@Service`/`@Component`/`@Singleton` | High |
| Non-thread-safe field reused across threads | `SimpleDateFormat`, `Calendar`, Jackson `ObjectMapper` builders, `SimpleDateFormat`/`DecimalFormat` as an instance/static field of a shared bean | High |
| Unsynchronized lazy init | Double-checked locking without `volatile`; lazy singletons built without synchronization | Medium |
| Non-atomic compound actions | `check-then-act`/`put-if-absent` on a shared `HashMap` instead of `ConcurrentHashMap`/`computeIfAbsent` | High |
| Mutable static shared state | `static` non-`final` fields mutated at runtime (also a memory risk above) | High |

*Detect:* for each singleton-scoped bean, grep its instance fields — flag non-`final` mutable ones written by request methods; grep `private (static )?SimpleDateFormat`/`DecimalFormat`/`Calendar` as fields; grep `synchronized`/`volatile`/`Atomic*`/`ConcurrentHashMap` to see whether shared state is guarded.

## Step 5 — Configuration management

| Issue | Detection | Severity |
|-------|-----------|----------|
| Hardcoded values | URLs/ports/hosts/timeouts in Java source instead of externalized config | High |
| Missing environment profiles | No `application-{profile}.yml` | Medium |
| Secrets in config files | Passwords/keys in `application.yml` not using `${ENV_VAR}`/vault | Critical |
| Missing config validation | `@ConfigurationProperties` without `@Validated`/`@NotNull` | Medium |
| Inconsistent config across profiles | Properties in dev but missing from prod | High |
| Production-unsafe defaults | e.g. `spring.jpa.hibernate.ddl-auto=update` in prod | Critical |
| Missing health checks | No custom `HealthIndicator` for critical dependencies | Medium |
| Missing graceful shutdown | No `server.shutdown=graceful` | Medium |
| Missing timeouts | HTTP client calls without connect/read timeouts | High |

*Detect:* read all `application*.yml`/`*.properties`; diff across profiles; grep hardcoded URLs/hosts/ports in source; check `@ConfigurationProperties` validation; check `RestTemplate`/`WebClient` timeouts.

## Step 6 — Observability & logging

| Issue | Detection | Severity |
|-------|-----------|----------|
| No structured logging | String-concatenated log messages instead of structured key/value | Medium |
| Missing correlation IDs | No MDC usage for request tracing | High |
| Sensitive data in logs | Passwords/tokens/PII/card numbers logged | Critical |
| Missing request/response logging | No HTTP logging filter for debugging | Medium |
| Inconsistent log levels | `info` for debug detail, `error` for non-errors | Medium |
| Missing metrics | No Micrometer/Prometheus/Dropwizard metrics for business ops | Medium |
| Missing distributed tracing | No OpenTelemetry/Zipkin/Jaeger for multi-service tracing | Medium |
| Missing custom health checks | No `HealthIndicator`/`HealthCheck` for DB/cache/external APIs | High |
| No alerting hooks | No metric thresholds or SLO definitions | Medium |
| Missing audit logging | No who-did-what logging (auth events, data mutations, admin actions) | High |
| Log output not externalized | Logging to local files instead of stdout/stderr in containers | Medium |

*Detect:* grep logging deps (`logback`/`log4j2`/`slf4j`/`micrometer`/`opentelemetry`); read `logback*.xml`/`log4j2.xml` (JSON vs plaintext); grep `MDC.put`/`MDC.setContextMap`; check actuator + Micrometer; grep `@Timed`/`@Counted`/`MeterRegistry`; check for tracing deps.

## Step 7 — Resilience & fault tolerance

Most relevant to production-readiness reviews and any app that calls external services (HTTP, DB, broker). A slow or failing dependency should degrade one call, not exhaust the thread pool and cascade into an outage.

| Issue | Detection | Severity |
|-------|-----------|----------|
| Missing timeouts on outbound calls | `RestTemplate`/`WebClient`/`HttpClient`/Feign without connect+read timeouts (also Step 5) | High |
| No retry on transient failures | Idempotent external calls with no retry (Spring Retry, Resilience4j, `@Retryable`) | Medium |
| Retry without backoff/jitter | Retries with no delay or fixed delay — thundering-herd / retry storms | Medium |
| Retrying non-idempotent calls | Retry wrapping a POST/charge/send with no idempotency key | High |
| No circuit breaker | Repeated calls to a flaky dependency with no breaker (Resilience4j `@CircuitBreaker`, Hystrix legacy) | Medium |
| No bulkhead / call isolation | All external calls share one pool — one slow dependency starves the rest | Medium |
| Missing fallback / graceful degradation | No fallback path when a non-critical dependency is down | Medium |
| Unbounded outbound concurrency | No rate limiter / semaphore on calls to a capacity-limited dependency | Medium |

*Detect:* check whether `resilience4j-*`, `spring-retry`, or `spring-cloud-starter-circuitbreaker` are on the classpath; grep `@Retryable`/`@CircuitBreaker`/`@Bulkhead`/`@RateLimiter`/`@TimeLimiter`; for each outbound client (`RestTemplate`/`WebClient`/`HttpClient`/Feign/broker producer), confirm timeouts and at least one resilience primitive. Absence on a service-to-service hot path is itself the finding.

## Step 8 — Compile findings

For each finding: **Category** (Data Access / Caching / Thread Pool / Memory / Thread Safety / Configuration / Observability / Resilience) · **Severity** · **Description** · **Evidence** (`file:line` + snippet) · **Impact** (what happens under load / in production) · **Recommended fix** (before/after) · **Effort** (XS/S/M/L/XL). Present the compiled findings.
