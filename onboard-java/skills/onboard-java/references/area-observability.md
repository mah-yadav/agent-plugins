# Area: Observability Analysis

Reference for Phase 3.4 of the onboarding workflow. Produces a structured report covering logging, metrics, health checks, distributed tracing, and Actuator endpoints.

**Core principle**: If you can't observe it, you can't debug it. Map the entire observability surface so the developer knows what's logged, what's measured, how requests are traced, and where to look when something goes wrong.

## Contents

- Step 1: Identify the Observability Stack
- Step 2: Analyze Logging (2.1 config, 2.2 structured/MDC, 2.3 patterns)
- Step 3: Analyze Metrics (3.1–3.4)
- Step 4: Analyze Health Checks (4.1 config, 4.2 indicators, 4.3 local check)
- Step 5: Analyze Distributed Tracing (5.1–5.3)
- Step 6: Analyze Actuator Endpoints
- Step 7: Error Tracking & Debugging Cheat Sheet (7.1 cheat sheet, 7.2 error tracking)
- Step 8: Generate Observability Report
- Ledger update, exploration guidelines

## Priority Tiers (within this area)

| Step | Priority | Rationale |
|---|---|---|
| Step 2: Logging config + how to read logs | **P0** | Day-1 debugging essential |
| Step 4: Health checks | **P0** | Must verify app is running |
| Step 7.1: Debugging cheat sheet | **P0** | Practical debugging commands |
| Step 3: Metrics | **P1** | Important for understanding system behavior |
| Step 5: Distributed tracing | **P1** | Important for multi-service debugging |
| Step 6: Actuator endpoints | **P1** | Useful operational knowledge |
| Step 3.3: Custom metrics inventory | **P2** | Detailed, not day-1 essential |
| Step 5.3: Custom spans inventory | **P2** | Detailed, not day-1 essential |
| Step 7.2: Error tracking deep dive | **P2** | Only if on-call |

**Quick mode**: Complete Step 1, Step 2.1 (logging config), Step 4.1 (health endpoint), and Step 7.1 (minimal debugging cheat sheet). Skip all other steps.

## Ledger Read-Before

Check the ledger. Do NOT re-read `application.yml` or the build file if they're already in `Files Read`. Only read observability-specific files not yet analyzed: `logback-spring.xml`, `log4j2.xml`, custom health indicators, custom metrics classes. If `explain-code` noted the logging framework, start from that finding.

Note `Reactive stack?` — MDC behaves differently in reactive code (see Step 3.2).

## Symbol Tracing

Use `Grep` for `MeterRegistry`, `Logger`, `HealthIndicator`, etc. The `Explore` subagent is useful for finding every `Counter.builder()` or `Timer.builder()` site.

## Step 1: Identify the Observability Stack — P0

If dependencies were already read, skip to finding observability-specific config files.

1. **From the build file** (or ledger), identify observability dependencies:

   **Spring Boot**:

   | Dependency | Indicates |
   |---|---|
   | `spring-boot-starter-actuator` | Actuator endpoints |
   | `micrometer-registry-prometheus` | Prometheus metrics |
   | `micrometer-registry-datadog` | Datadog metrics |
   | `micrometer-tracing-bridge-otel` | OpenTelemetry tracing |
   | `micrometer-tracing-bridge-brave` | Zipkin/Brave tracing |
   | `logstash-logback-encoder` | Structured JSON logging |
   | `sentry-spring-boot-starter` | Sentry error tracking |

   **Quarkus**:

   | Dependency | Indicates | Default Endpoint |
   |---|---|---|
   | `quarkus-smallrye-health` | Health checks | `/q/health` |
   | `quarkus-micrometer` / `quarkus-micrometer-registry-prometheus` | Metrics | `/q/metrics` |
   | `quarkus-opentelemetry` | Distributed tracing | — |
   | `quarkus-logging-json` | Structured JSON logging | — |

   **Micronaut**:

   | Dependency | Indicates | Default Endpoint |
   |---|---|---|
   | `micronaut-management` | Health & info endpoints | `/health` |
   | `micronaut-micrometer` | Metrics | `/metrics` |
   | `micronaut-tracing-opentelemetry` | Distributed tracing | — |

   **Jakarta EE / MicroProfile**:

   | Spec | Indicates | Default Endpoint |
   |---|---|---|
   | MicroProfile Health | Health checks | `/health` |
   | MicroProfile Metrics | Metrics (vendor, base, application) | `/metrics` |
   | MicroProfile OpenTracing / Telemetry | Distributed tracing | — |

2. **Find logging config**: `logback-spring.xml`, `logback.xml`, `log4j2-spring.xml`, `log4j2.xml`.
3. **Find health/management config**: `management.*` (Spring Boot), `quarkus.smallrye-health.*` (Quarkus), `endpoints.*` (Micronaut).

## Step 2: Analyze Logging — P0

### 2.1 Logging Framework & Configuration — P0

| What to Find | Where to Look |
|---|---|
| Framework | Dependencies — Logback (default), Log4j2 |
| Config file | `logback-spring.xml`, `log4j2-spring.xml` in `src/main/resources/` |
| Log format | Pattern layout — plain text vs. JSON |
| Log levels | Root level, package-specific levels |
| Output destinations | Console, file, Logstash TCP, Fluentd |
| Profile-specific logging | `<springProfile>` blocks |

**Extract**: Log format, default level, package overrides, where logs go, how to change levels at runtime.

### 2.2 Structured Logging & MDC — P1

- JSON logging: `LogstashEncoder`, `JsonTemplateLayout`.
- MDC usage: `MDC.put()`, `MDC.get()` — what context is added (`correlationId`, `requestId`, `userId`).
- MDC filters: Servlet filters populating MDC per request.
- Sensitive data masking.

**Reactive caveat**: If the project is on a reactive stack (WebFlux / Mutiny), classic `MDC.put()` in a servlet filter will **not** propagate across operator boundaries because each `Mono`/`Flux` step may run on a different thread. Look instead for:

- `Hooks.enableAutomaticContextPropagation()` (Reactor 3.5+ with Micrometer context-propagation).
- `contextWrite(...)` calls populating Reactor `Context`.
- Custom `Schedulers.onScheduleHook` MDC lifters.
- Micrometer Observation API with `ObservationThreadLocalAccessor`.

If none of the above is present in a reactive app, MDC entries will appear sporadically (only for log lines emitted on the original thread) — flag this as a finding because it changes how developers must add correlation IDs.

### 2.3 Logging Patterns in Code — P1

- Logger declaration style: `LoggerFactory.getLogger()`, `@Slf4j` (Lombok).
- Parameterized logging vs. string concatenation.
- Exception logging patterns.
- Request/response logging filters.

## Step 3: Analyze Metrics — P1

### 3.1 Metrics Framework & Export — P1

Metrics facade (Micrometer), registry type, endpoint (`/actuator/prometheus`, `/actuator/metrics`), common tags, distribution config.

### 3.2 Auto-Configured Metrics — P1

| Category | Metrics | Detection |
|---|---|---|
| JVM | Memory, GC, threads | Always with Actuator |
| HTTP server | Request count, duration | Spring MVC/WebFlux |
| Database | Connection pool, query timing | HikariCP |
| Cache | Hits, misses | Caffeine/Redis |
| Kafka | Consumer lag | Spring Kafka |

### 3.3 Custom Metrics — P2

Search for `@Timed`, `@Counted`, `MeterRegistry` injection, `Counter.builder()`, `Timer.builder()`.

**Extract**: Table of custom metrics with name, type, what they measure.

### 3.4 Micrometer Observation API (Boot 3+) — P2

`ObservationRegistry`, `@Observed`, `ObservationHandler`.

## Step 4: Analyze Health Checks — P0

### 4.1 Health Endpoint Configuration — P0

- Endpoint exposure: `management.endpoints.web.exposure.include`.
- Detail visibility: `management.endpoint.health.show-details`.
- Kubernetes probes: liveness/readiness paths.

### 4.2 Health Indicators — P1

Auto-configured (db, diskSpace, redis, kafka) + custom `HealthIndicator` implementations.

### 4.3 How to Check Health Locally — P0

```bash
curl http://localhost:[port]/actuator/health
```

## Step 5: Analyze Distributed Tracing — P1

### 5.1 Tracing Framework — P1

Library, propagation format (W3C/B3/X-Ray), export destination, sampling rate.

### 5.2 Auto-Instrumentation — P1

What's auto-traced: HTTP server/client, database, messaging.

### 5.3 Custom Spans — P2

`@NewSpan`, `@Observed`, programmatic span creation.

## Step 6: Analyze Actuator Endpoints — P1

Exposed endpoints, custom base path, custom management port, custom endpoints.

## Step 7: Error Tracking & Debugging Cheat Sheet — P0/P2

### 7.1 Debugging Cheat Sheet — P0

| Scenario | What to Do |
|---|---|
| API returns 500 | Check logs for stack trace, search by traceId |
| Slow endpoint | Check `http.server.requests` timer, trace spans |
| Database issues | Check `/actuator/health/db`, HikariCP metrics |
| Memory issues | Check `jvm.memory.used`, heap dump via `/actuator/heapdump` |

### 7.2 Error Tracking Services — P2

Sentry, Elastic APM, dead letter queues, alert annotations.

## Step 8: Generate Observability Report

Write the report using the [observability template](../assets/observability-analysis-template.md).

**Quick mode**: Fill template sections 1 (Stack), 2 (Logging config), 4 (Health Checks), 7 (Debugging Cheat Sheet). Mark other sections `[Not analyzed — out of scope for Quick mode]`.

Default location: `<output-dir>/observability-analysis-report.md` (Deep mode only — in Standard mode, observability findings roll into the main report).

## Ledger Update After

Mark area 3.4 complete. Add:
- Logging framework and format
- Health check URL
- Metrics backend
- Tracing setup
- Key debugging commands

## Exploration Guidelines

- **Read logging config files in full** — they define entire logging behavior.
- **Check all profiles**: Logging varies by profile — dev console plain text, prod JSON to file.
- **Follow the evidence chain**: Prometheus registry → find metrics endpoint. Tracing bridge → find export destination.
- **Note absences**: No structured logging? No custom metrics? No tracing? Report them.
- **Don't expose secrets**: Note config keys, not values.
