# Node Application Advisor — Findings Report

> **Application**: [Application Name]
> **Analyzed**: [Date]
> **Scope**: [Full analysis / Specific areas]
> **Plugin**: node-app-advisor v1.0

---

## Executive Summary

| Severity  | Count |
| --------- | ----- |
| Critical  | 0     |
| High      | 0     |
| Medium    | 0     |
| Low       | 0     |
| **Total** | **0** |

### Top 5 Critical Findings

| #   | Finding | Category | Location |
| --- | ------- | -------- | -------- |
| 1   |         |          |          |
| 2   |         |          |          |
| 3   |         |          |          |
| 4   |         |          |          |
| 5   |         |          |          |

### Application Profile

| Attribute         | Value                                                                       |
| ----------------- | --------------------------------------------------------------------------- |
| **Type**          | [Web API / SSR app / Worker / CLI]                                          |
| **Framework**     | [Express / Fastify / NestJS / Koa / Hono / Next.js / tRPC / Plain Node] X.Y |
| **Language**      | [TypeScript X.Y / JavaScript]                                               |
| **Module system** | [ESM / CommonJS]                                                            |
| **Node Version**  | [engines.node]                                                              |
| **Package Mgr**   | [npm / pnpm / yarn]                                                          |
| **Structure**     | [Single package / Monorepo (list packages)]                                 |
| **LOC (approx)**  | [X]                                                                         |

---

## 1. Architecture & Design Findings

### 1.1 Architecture Overview

- **Pattern**: [Layered / Hexagonal / Clean / Package-by-feature / NestJS modules / No discernible pattern]
- **Directory Structure**: [Brief description]
- **Domain Model**: [Rich / Anemic / Mixed]

### 1.2 Layering & Module-Boundary Violations

| ID     | Description | Severity | Evidence                   | Impact |
| ------ | ----------- | -------- | -------------------------- | ------ |
| AD-001 |             |          | `file:line` — code snippet |        |

### 1.3 SOLID Violations

| ID     | Principle | Description | Severity | Evidence                   | Impact |
| ------ | --------- | ----------- | -------- | -------------------------- | ------ |
| AD-010 |           |             |          | `file:line` — code snippet |        |

### 1.4 Coupling & Cohesion Issues

| ID     | Description | Severity | Evidence                   | Impact |
| ------ | ----------- | -------- | -------------------------- | ------ |
| AD-020 |             |          | `file:line` — code snippet |        |

### 1.5 Design Anti-Patterns

| ID     | Anti-Pattern | Description | Severity | Evidence                   | Impact |
| ------ | ------------ | ----------- | -------- | -------------------------- | ------ |
| AD-030 |              |             |          | `file:line` — code snippet |        |

---

## 2. Security & Dependencies Findings

### 2.1 Security Stack

| Component          | Implementation                             | Notes |
| ------------------ | ------------------------------------------ | ----- |
| **Authentication** | [OAuth2/OIDC / JWT / Session / API Key]    |       |
| **Authorization**  | [Route-level / Role-based / Guard / Both]  |       |
| **CORS**           | [Configured / Missing / Overly permissive] |       |
| **CSRF**           | [Enabled / Disabled — reason]              |       |
| **Headers**        | [helmet / equivalent / Missing]            |       |

### 2.2 OWASP Top 10 Findings

| ID      | OWASP Category             | Description | Severity | Evidence                   | Impact |
| ------- | -------------------------- | ----------- | -------- | -------------------------- | ------ |
| SEC-001 | A01: Broken Access Control |             |          | `file:line` — code snippet |        |
| SEC-002 | A03: Injection             |             |          | `file:line` — code snippet |        |

### 2.3 Hardcoded Secrets

| ID      | Type | Location    | Severity |
| ------- | ---- | ----------- | -------- |
| SEC-050 |      | `file:line` |          |

### 2.4 Dependency Health

| Dependency | Current Version | Latest Version | Risk                    | Severity |
| ---------- | --------------- | -------------- | ----------------------- | -------- |
|            |                 |                | [CVE / EOL / Abandoned] |          |

---

## 3. Performance & Configuration Findings

### 3.1 Event Loop & Async Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-001 |             |          | `file:line` — code snippet |        |

### 3.2 Data Access Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-010 |             |          | `file:line` — code snippet |        |

### 3.3 Caching Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-020 |             |          | `file:line` — code snippet |        |

### 3.4 Memory & Resource Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-030 |             |          | `file:line` — code snippet |        |

### 3.5 Configuration Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-040 |             |          | `file:line` — code snippet |        |

### 3.6 Observability Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-050 |             |          | `file:line` — code snippet |        |

### 3.7 Resilience & Fault-Tolerance Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-060 |             |          | `file:line` — code snippet |        |

---

## 4. Testing & API Design Findings

### 4.1 Test Coverage Summary

| Layer            | Source Modules | Test Files | Coverage | Gaps |
| ---------------- | -------------- | ---------- | -------- | ---- |
| Routes/Ctrl      |                |            |          |      |
| Services         |                |            |          |      |
| Repositories     |                |            |          |      |
| Middleware/Guard |                |            |          |      |
| Auth             |                |            |          |      |
| Other            |                |            |          |      |

### 4.2 Test Quality Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| TEST-001 |             |          | `file:line` — code snippet |        |

### 4.3 API Design Issues

| ID      | Description | Severity | Evidence           | Impact |
| ------- | ----------- | -------- | ------------------ | ------ |
| API-001 |             |          | `endpoint details` |        |

### 4.4 Contract & Integration Issues

| ID      | Description | Severity | Evidence                   | Impact |
| ------- | ----------- | -------- | -------------------------- | ------ |
| API-010 |             |          | `file:line` — code snippet |        |

---

## 5. Technical Debt Findings

### 5.1 Code Marker Summary

| Marker                  | Count | Examples |
| ----------------------- | ----- | -------- |
| TODO                    | 0     |          |
| FIXME                   | 0     |          |
| HACK                    | 0     |          |
| @ts-ignore / @ts-nocheck| 0     |          |
| eslint-disable          | 0     |          |

### 5.2 Dead Code

| ID     | Description | Severity | Evidence                   | Impact |
| ------ | ----------- | -------- | -------------------------- | ------ |
| TD-001 |             |          | `file:line` — code snippet |        |

### 5.3 Deprecated API Usage

| ID     | Deprecated API | Replacement | Severity | Evidence    |
| ------ | -------------- | ----------- | -------- | ----------- |
| TD-010 |                |             |          | `file:line` |

### 5.4 Inconsistencies

| ID     | Description | Severity | Evidence                   | Impact |
| ------ | ----------- | -------- | -------------------------- | ------ |
| TD-020 |             |          | `file:line` — code snippet |        |

### 5.5 Code Duplication

| ID     | Description | Severity | Evidence                   | Impact |
| ------ | ----------- | -------- | -------------------------- | ------ |
| TD-030 |             |          | `file:line` — code snippet |        |

### 5.6 Error Handling Issues

| ID     | Description | Severity | Evidence                   | Impact |
| ------ | ----------- | -------- | -------------------------- | ------ |
| TD-040 |             |          | `file:line` — code snippet |        |

### 5.7 Type-Safety Debt (TypeScript)

| ID     | Description | Severity | Evidence                   | Impact |
| ------ | ----------- | -------- | -------------------------- | ------ |
| TD-050 |             |          | `file:line` — code snippet |        |

---

## Appendix: Analysis Methodology

- **Architecture & Design**: Directory-structure analysis, import-graph analysis, module size/complexity metrics, decorator/route scanning.
- **Security**: OWASP Top 10 pattern matching, auth-middleware/guard review, secret scanning, `npm audit` dependency review.
- **Performance**: Event-loop blocking analysis, query-pattern analysis, connection/pool configuration review, memory/resource audit, observability review.
- **Testing**: Test-to-source mapping, assertion-quality analysis, API endpoint-to-test coverage mapping.
- **Tech Debt**: Code-marker scanning, dead-code detection via usage analysis, deprecated-API identification, pattern-consistency and type-safety analysis.
