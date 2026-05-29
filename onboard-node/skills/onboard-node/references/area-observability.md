# Area: Observability Analysis (Node)

Reference for Phase 3.4. Produces a structured report covering logging, metrics, health checks, distributed tracing.

**Core principle**: If you can't observe it, you can't debug it. Node observability is library-driven (not framework-driven like Spring Actuator), so the libraries chosen and their configuration determine the entire surface.

## Priority Tiers (within this area)

| Step | Priority | Rationale |
|---|---|---|
| Step 2: Logging config + how to read logs | **P0** | Day-1 debugging essential |
| Step 4: Health checks | **P0** | Must verify app is running |
| Step 7: Debugging cheat sheet | **P0** | Practical commands |
| Step 3: Metrics | **P1** | Understanding system behavior |
| Step 5: Distributed tracing | **P1** | Multi-service debugging |
| Step 6: Error tracking / APM | **P1** | Production triage |
| Step 3.3: Custom metrics inventory | **P2** | Deep detail |
| Step 5.3: Custom spans inventory | **P2** | Deep detail |

**Quick mode**: Steps 1, 2.1 (logging config), 4.1 (health endpoint), and the debugging cheat sheet. Skip the rest.

## Ledger Read-Before

Check the ledger. Do NOT re-read `package.json`. Pull observability dependencies from the prior analysis. If `area-explain-code` noted the logging library, start from that.

Note `Framework`, `Edge runtime target`. Edge runtimes have limited observability options (no file appenders, no `process.cpuUsage()`, often no OpenTelemetry SDK support).

## Symbol Tracing

Useful greps: `logger.info`, `log.info`, `pino()`, `winston.createLogger`, `console.log` (note prevalence — heavy `console.log` use is a code smell), `metrics.`, `counter.inc`, `histogram.observe`.

## Step 1: Identify the Observability Stack — P0

### 1.1 Logging libraries

| Dependency | Library | Style |
|---|---|---|
| `pino` | Pino | Fast, JSON-by-default, child-logger pattern |
| `winston` | Winston | Configurable transports, more flexible, slower |
| `bunyan` | Bunyan | JSON, older, less common today |
| `consola` | Consola | Nuxt/Unjs ecosystem, dev-friendly |
| `loglevel` | loglevel | Simple, common in libraries |
| `debug` | debug | Namespace-based; enabled via DEBUG env var |
| None (only `console`) | Native | Common in small projects; flag if production-grade is expected |

**Configuration sources**:
- A `logger.ts` / `logging.ts` module exporting a configured instance.
- Framework-specific:
  - NestJS: `Logger` injectable, `WinstonModule.forRoot(...)` or pino integration.
  - Fastify: `fastify.log` (Pino built-in by default).
  - Next.js: typically a custom logger in `lib/logger.ts`.

### 1.2 Metrics libraries

| Dependency | What it does |
|---|---|
| `prom-client` | Prometheus client; expose `/metrics` endpoint |
| `@opentelemetry/sdk-metrics`, `@opentelemetry/exporter-prometheus` | OTel metrics |
| `datadog-metrics`, `hot-shots` (StatsD) | Datadog/StatsD |
| `@newrelic/agent` | New Relic APM (also metrics) |
| `appsignal` | AppSignal |
| `swagger-stats` | HTTP metrics dashboard (more for dev/staging) |
| `express-prom-bundle`, `@fastify/metrics` | Framework-level metric middleware |

### 1.3 Tracing libraries

| Dependency | What |
|---|---|
| `@opentelemetry/sdk-node`, `@opentelemetry/api`, exporters | OpenTelemetry (most common in 2024+) |
| `dd-trace` | Datadog APM tracer |
| `elastic-apm-node` | Elastic APM |
| `@sentry/node` (Sentry has performance/tracing as well) | Sentry tracing |
| `newrelic` | New Relic |
| `applicationinsights` | Azure App Insights |

### 1.4 Error tracking

| Dependency | Service |
|---|---|
| `@sentry/node`, `@sentry/nextjs`, `@sentry/nestjs` | Sentry |
| `rollbar` | Rollbar |
| `bugsnag` / `@bugsnag/node` | Bugsnag |
| `@honeybadger-io/js` | Honeybadger |
| Custom `process.on('uncaughtException', ...)` only | Manual logging only |

### 1.5 Health check libraries

| Dependency | What |
|---|---|
| `@godaddy/terminus` | Graceful shutdown + health checks for Express |
| `@nestjs/terminus` | Health module for NestJS |
| `fastify-healthcheck`, `@fastify/under-pressure` | Fastify health |
| Custom `GET /health` route | Most common |
| Kubernetes-style `/healthz`, `/livez`, `/readyz` paths | k8s convention |

**Present findings**: Logging lib, metrics backend, tracing library, error tracker, health endpoint.

## Step 2: Analyze Logging — P0

### 2.1 Logging configuration — P0

| What to Find | Where to Look |
|---|---|
| Logger setup | `src/logger.ts`, `lib/logger.ts`, framework-integrated logger |
| Log format | JSON vs. pretty-printed; pino prettier (`pino-pretty`) for dev |
| Log levels | `level: 'info'` (Pino), `levels: { ... }` (Winston) |
| Transports / destinations | stdout (default), file (`pino-roll`), HTTP (Logstash), external (Loki, Datadog) |
| Profile-specific | NODE_ENV-aware level/format switching |
| Redaction | `redact: ['password', 'authorization']` in Pino; sensitive-field filtering |

**Extract**: Log format, default level, per-module level overrides, where logs go (stdout? file? remote?), how to change level at runtime (env var? signal? endpoint?).

### 2.2 Correlation / Request ID — P1

Node has no MDC; correlation IDs are passed explicitly.

| Approach | Detection |
|---|---|
| Middleware that generates a request ID and stores it on `req` | `express-request-id`, `@fastify/request-context`, custom `req.id = uuid()` |
| Child logger per request | `logger.child({ requestId: req.id })` in middleware; passed to subsequent code |
| AsyncLocalStorage | Node `async_hooks`'s `AsyncLocalStorage` — request-scoped storage |
| OpenTelemetry context | trace ID auto-propagated by OTel SDK |

**Flag if absent**: Production-grade Node apps without correlation IDs make distributed debugging painful.

### 2.3 Sensitive data masking — P1

- Pino `redact` option: list of paths to redact.
- Winston format that filters fields.
- Custom serializers stripping PII.
- Flag if `req.body` or full requests are being logged without filtering.

## Step 3: Analyze Metrics — P1

### 3.1 Metrics framework — P1

- The library detected (Step 1.2), the registry, the export endpoint (`/metrics`), the format (Prometheus exposition / StatsD / OTLP).
- Push vs. pull model.

### 3.2 Auto-instrumented metrics — P1

| Source | Metrics |
|---|---|
| `prom-client` `collectDefaultMetrics` | Process metrics (RSS, heap, event loop lag, GC) |
| `express-prom-bundle` / `@fastify/metrics` | HTTP request duration, status code, route, error rate |
| Database client integrations | Connection pool, query duration (Prisma metrics, pg-pool stats) |
| BullMQ Prometheus exporter | Job throughput, queue depth, failure rate |
| OTel auto-instrumentation | HTTP, gRPC, fs, DNS, common DB drivers (with `@opentelemetry/auto-instrumentations-node`) |

### 3.3 Custom metrics — P2

Grep for `new Counter`, `new Gauge`, `new Histogram`, `new Summary` (`prom-client`), or `meter.createCounter`/`createHistogram` (OTel).

**Extract**: Table of custom metrics with name, type, what they measure.

## Step 4: Health Checks — P0

### 4.1 Health endpoints — P0

- The route(s) exposed: `/health`, `/healthz`, `/livez`, `/readyz`.
- What checks run inside: DB ping, Redis ping, downstream service check.
- Distinguished liveness vs. readiness?
- Kubernetes probe config (if k8s manifests exist): which path/port/initialDelay.

### 4.2 Graceful shutdown — P1

- `process.on('SIGTERM', ...)` handler that closes the server, drains connections, exits cleanly?
- `@godaddy/terminus` / `@nestjs/terminus` / `http-graceful-shutdown` integration?
- Worker / consumer shutdown sequence (BullMQ, Kafka consumers).

### 4.3 How to check locally — P0

```bash
curl http://localhost:<port>/health
```

## Step 5: Analyze Distributed Tracing — P1

### 5.1 Tracing framework — P1

- The library (OTel / Datadog / Elastic / New Relic / Sentry).
- Propagation format (W3C Trace Context default for OTel; B3 for Zipkin; Datadog has its own).
- Exporter / endpoint (OTLP HTTP/gRPC, Datadog agent, etc.).
- Sampling: head-based (% of requests sampled at the start) vs. tail-based (decide after request completes).

### 5.2 Auto-instrumentation — P1

What's automatically traced:
- HTTP client and server.
- DB clients (mysql2, pg, mongodb, ioredis).
- Some message brokers (kafkajs, amqplib).
- gRPC.
- fs (sometimes).

### 5.3 Custom spans — P2

Grep for `tracer.startSpan`, `tracer.startActiveSpan`, `@WithSpan` (rare in JS), or vendor-specific equivalents.

## Step 6: Error Tracking & APM — P1

- Sentry / Rollbar / Bugsnag init: typically in the bootstrap file. Check DSN env var.
- Global error handlers: `process.on('uncaughtException', ...)`, `process.on('unhandledRejection', ...)`. Are they sending to the error tracker?
- Framework integrations: `Sentry.Handlers.errorHandler()` for Express, `setupNestErrorHandler` for NestJS.
- Source maps uploaded for production? — relevant for Sentry to deminify stacks. Check CI for `sentry-cli sourcemaps upload`.

## Step 7: Debugging Cheat Sheet — P0

A practical table for the report:

| Scenario | What to do |
|---|---|
| API returns 500 | Check logs; if Sentry configured, find the issue there; correlate by request ID |
| Slow endpoint | If APM/OTel: check the trace. If `prom-client` metrics: check `http_request_duration_seconds` |
| DB issues | Check `/health/db` if exposed; query slow log; connection pool metrics |
| Memory leak | `node --inspect` + Chrome DevTools heap snapshot; Clinic.js Doctor; `process.memoryUsage()` |
| Event loop lag | Pino reports `eventLoopUtilization`; `prom-client` default metric; Clinic.js Doctor |
| Hanging requests | `node --inspect` + async hooks debugger; check unresolved promises |
| Unhandled rejection | `process.on('unhandledRejection', ...)` log; Sentry catches by default |
| Crashing process | Stack trace in logs; check `process.on('uncaughtException')` handler |

## Step 8: Generate Observability Report

Write using the [observability template](../assets/observability-analysis-template.md).

**Quick mode**: Fill sections 1 (Stack), 2.1 (Logging config), 4.1 (Health endpoint), 7 (Debugging cheat sheet). Mark others `[Not analyzed]`.

Default location: `<output-dir>/observability-analysis-report.md` (Standard/Deep only).

## Ledger Update After

Mark area 3.4 complete. Add:
- Logging library and format
- Health check URL(s)
- Metrics backend (or "none")
- Tracing setup (or "none")
- Error tracker (or "none")
- Key debugging commands

## Exploration Guidelines

- **Read logger config files in full** — they define what's captured.
- **Check ALL profiles**: dev console logs vs. prod JSON-to-stdout vs. test silent.
- **Follow the evidence chain**: `prom-client` registry → `/metrics` endpoint. `@sentry/node` → DSN env var. Tracing exporter → collector URL.
- **Note absences**: No structured logging? No metrics? No tracing? No correlation IDs? Each is a finding.
- **Don't expose secrets**: Note config keys (e.g., `SENTRY_DSN`), never values.
