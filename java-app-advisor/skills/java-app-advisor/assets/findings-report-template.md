# Java Application Advisor — Findings Report

> **Application**: [Application Name]
> **Analyzed**: [Date]
> **Scope**: [Full analysis / Specific areas]
> **Plugin**: java-app-advisor v1.0

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

| Attribute        | Value                                                        |
| ---------------- | ------------------------------------------------------------ |
| **Type**         | [Web / Standalone / Batch / CLI]                             |
| **Framework**    | [Spring Boot X.Y / Micronaut X.Y / Quarkus X.Y / Plain Java] |
| **Java Version** | [X]                                                          |
| **Build Tool**   | [Maven / Gradle]                                             |
| **Modules**      | [Single / Multi (list)]                                      |
| **LOC (approx)** | [X]                                                          |

---

## 1. Architecture & Design Findings

### 1.1 Architecture Overview

- **Pattern**: [Layered / Hexagonal / Clean / Package-by-feature / No discernible pattern]
- **Package Structure**: [Brief description]
- **Domain Model**: [Rich / Anemic / Mixed]

### 1.2 Layering Violations

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
| **Authentication** | [OAuth2 / JWT / Basic / Form / API Key]    |       |
| **Authorization**  | [URL-based / Method-level / Both]          |       |
| **CORS**           | [Configured / Missing / Overly permissive] |       |
| **CSRF**           | [Enabled / Disabled — reason]              |       |

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

### 3.1 Data Access Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-001 |             |          | `file:line` — code snippet |        |

### 3.2 Caching Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-010 |             |          | `file:line` — code snippet |        |

### 3.3 Thread Pool & Async Issues

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

### 3.7 Thread-Safety Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-060 |             |          | `file:line` — code snippet |        |

### 3.8 Resilience & Fault-Tolerance Issues

| ID       | Description | Severity | Evidence                   | Impact |
| -------- | ----------- | -------- | -------------------------- | ------ |
| PERF-070 |             |          | `file:line` — code snippet |        |

---

## 4. Testing & API Design Findings

### 4.1 Test Coverage Summary

| Layer      | Source Classes | Test Classes | Coverage | Gaps |
| ---------- | -------------- | ------------ | -------- | ---- |
| Controller |                |              |          |      |
| Service    |                |              |          |      |
| Repository |                |              |          |      |
| Security   |                |              |          |      |
| Other      |                |              |          |      |

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

| Marker            | Count | Examples |
| ----------------- | ----- | -------- |
| TODO              | 0     |          |
| FIXME             | 0     |          |
| HACK              | 0     |          |
| @SuppressWarnings | 0     |          |

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

---

## Appendix: Analysis Methodology

- **Architecture & Design**: Package-structure analysis, import-graph analysis, class size/complexity metrics, annotation scanning.
- **Security**: OWASP Top 10 pattern matching, security-configuration review, secret scanning, dependency CVE check.
- **Performance**: JPA annotation analysis, query-pattern analysis, connection/thread-pool configuration review, resource-management audit, observability review.
- **Testing**: Test-to-source mapping, assertion-quality analysis, API endpoint-to-test coverage mapping.
- **Tech Debt**: Code-marker scanning, dead-code detection via usage analysis, deprecated-API identification, pattern-consistency analysis.
