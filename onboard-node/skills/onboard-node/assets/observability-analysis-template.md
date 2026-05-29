# Observability Analysis Report: [Project Name]

**Date**: [Date]

## 1. Observability Stack

| Topic | Tool / Library |
|---|---|
| Logging library | |
| Log format | [JSON / pretty] |
| Log destinations | |
| Metrics library | |
| Metrics backend | [Prometheus / Datadog / StatsD / OTel] |
| Tracing library | |
| Tracing exporter | |
| Error tracking | |
| Health checks | |

## 2. Logging

### 2.1 Configuration

- File: [logger.ts / lib/logger.ts]
- Default level: [info / warn / debug]
- Format: [JSON via Pino / pretty via pino-pretty in dev]
- Destinations: [stdout / file / remote]
- Per-module overrides: [yes / no — list]
- Profile-specific: [dev / test / prod differences]

### 2.2 Correlation IDs / Request IDs

- Approach: [middleware-assigned + child logger / AsyncLocalStorage / OTel context / none — flag]
- Field name(s): [requestId, traceId, ...]

### 2.3 Redaction

- Library config: [pino redact paths, winston format, custom]
- Fields masked:

### 2.4 How to change log level at runtime

[env var / signal / config endpoint / restart]

## 3. Metrics

### 3.1 Endpoint

- URL: `[/metrics]`
- Format: [Prometheus exposition / OTLP / StatsD push]
- Auth: [public / internal-only / authenticated]

### 3.2 Auto-instrumented

- HTTP request duration: [yes — middleware/library]
- DB query timing: [yes — Prisma metrics / pg-pool / OTel auto-instrumentation]
- Event loop lag: [yes — prom-client default / no]
- GC / memory: [yes / no]

### 3.3 Custom metrics

| Name | Type | What it measures |
|---|---|---|
| | | |

## 4. Health Checks

### 4.1 Endpoints

| Path | What it checks | Auth |
|---|---|---|
| `/health` | | |
| `/healthz` (liveness) | | |
| `/readyz` (readiness) | | |

### 4.2 Graceful shutdown

- SIGTERM handler: [yes — file:line / no]
- Connection drain: [yes / no]
- Worker shutdown sequence: [BullMQ workers, Kafka consumers]

### 4.3 How to check locally

```bash
curl http://localhost:[port]/health
```

## 5. Distributed Tracing

- Library: [OTel SDK / Datadog / Elastic / Sentry / New Relic]
- Propagation: [W3C TraceContext / B3 / Datadog]
- Exporter: [endpoint, OTLP HTTP/gRPC, ...]
- Sampling: [percentage, rule]
- Auto-instrumented: [HTTP, DB, queue, ...]

## 6. Error Tracking / APM

- Service: [Sentry / Rollbar / Bugsnag / Honeybadger / New Relic / Datadog]
- Init: [file:line]
- DSN/secret env var:
- Global handlers wired: [uncaughtException, unhandledRejection]
- Source maps uploaded in CI: [yes / no]

## 7. Debugging Cheat Sheet

| Scenario | What to do |
|---|---|
| API returns 500 | |
| Slow endpoint | |
| DB issue | |
| Memory leak | |
| Event loop lag | |
| Hanging request | |
| Unhandled rejection | |

## 8. Notable Findings

-

## 9. Recommendations (Deep mode)

-
