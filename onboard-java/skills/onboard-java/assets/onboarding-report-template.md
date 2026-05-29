# Codebase Onboarding Guide: [Project Name]

**Date**: [Date]
**Project**: [groupId:artifactId or project name]
**Repository**: [repo URL if known]
**Developer**: [Name or role]
**Focus**: [Backend features / Bug fixes / API integrations / DevOps / Full stack]
**Mode**: [Quick / Standard / Deep]

---

## 1. Project Identity

| Attribute         | Value                                      |
| ----------------- | ------------------------------------------ |
| **Project name**  |                                            |
| **Purpose**       | [1-2 sentence description]                 |
| **Java version**  |                                            |
| **Framework**     | [Spring Boot X.Y / Micronaut / Quarkus]    |
| **Build tool**    | [Maven / Gradle]                           |
| **Architecture**  | [Layered / Hexagonal / DDD / Microservice] |
| **Multi-module?** | [Yes — list modules / No]                  |

## 2. Tech Stack at a Glance

| Layer      | Technology | Notes |
| ---------- | ---------- | ----- |
| Framework  |            |       |
| Build      |            |       |
| Database   |            |       |
| Migrations |            |       |
| Caching    |            |       |
| Messaging  |            |       |
| Security   |            |       |
| Testing    |            |       |
| API docs   |            |       |
| Logging    |            |       |
| Metrics    |            |       |
| CI/CD      |            |       |
| Deployment |            |       |

## 3. Build & Run Locally

> **Full guide**: [build-local-dev-guide.md](./build-local-dev-guide.md) *(Standard/Deep mode only)*

### Quick Start

```bash
# Prerequisites: [Java version], [Maven/Gradle version], [Docker if needed]

# Build
[mvn clean install / ./gradlew build]

# Run
[mvn spring-boot:run / ./gradlew bootRun / java -jar ...]

# Run tests
[mvn test / ./gradlew test]
```

### Key Environment Variables

| Variable | Purpose | Default |
| -------- | ------- | ------- |
|          |         |         |

### Local Services

| Service | Purpose | How to Start | Port |
| ------- | ------- | ------------ | ---- |
|         |         |              |      |

## 4. Architecture, APIs, Data & Testing

> **Full report**: [code-explanation-report.md](./code-explanation-report.md) *(Standard/Deep mode only)*

### Architecture

| Attribute        | Value                                      |
| ---------------- | ------------------------------------------ |
| Pattern          | [Layered / Hexagonal / DDD / Microservice] |
| Main entry point | [Application class + path]                 |
| Key packages     | [package-to-layer mapping summary]         |

### APIs & Communication

| Type             | Summary                                                       |
| ---------------- | ------------------------------------------------------------- |
| REST endpoints   | [count] endpoints — API docs at [URL if available]            |
| Messaging        | [Kafka / RabbitMQ / JMS — topics/queues summary, or None]     |
| External clients | [Feign / WebClient / RestTemplate — services called, or None] |
| gRPC / GraphQL   | [present / not present]                                       |

### Data Layer

| Attribute    | Value                                |
| ------------ | ------------------------------------ |
| Database     | [PostgreSQL / MySQL / MongoDB / ...] |
| ORM          | [JPA / MyBatis / jOOQ]               |
| Migrations   | [Flyway / Liquibase / None]          |
| Caching      | [Caffeine / Redis / EhCache / None]  |
| Key entities | [list top 3-5 domain entities]       |

### Testing

| What              | Command   | Notes                  |
| ----------------- | --------- | ---------------------- |
| Unit tests        | [command] | [naming: `*Test.java`] |
| Integration tests | [command] | [naming: `*IT.java`]   |
| All tests         | [command] |                        |

| Attribute          | Value                                           |
| ------------------ | ----------------------------------------------- |
| Coverage threshold | [e.g., 80% — enforced by JaCoCo / not enforced] |
| Test frameworks    | [JUnit 5, Mockito, Testcontainers, etc.]        |

## 5. Security

> **Full report**: [security-analysis-report.md](./security-analysis-report.md) *(Deep mode only)*

| Attribute      | Value                                      |
| -------------- | ------------------------------------------ |
| Authentication | [OAuth2 / JWT / SAML / Basic / API Key]    |
| Provider       | [Keycloak / Okta / Auth0 / Custom]         |
| Authorization  | [Role-based / Permission-based]            |
| CORS           | [configured / not configured — summary]    |
| Local dev auth | [How to authenticate when running locally] |

<!-- In Quick mode, only Authentication and Local dev auth rows are required -->

## 6. Observability

> **Full report**: [observability-analysis-report.md](./observability-analysis-report.md) *(Deep mode only)*

| Attribute       | Value                                         |
| --------------- | --------------------------------------------- |
| Logging         | [Logback / Log4j2] — [Plain / JSON]           |
| Metrics backend | [Prometheus / Datadog / CloudWatch / None]    |
| Health endpoint | [`/actuator/health` — exposed / not exposed]  |
| Tracing         | [Sleuth / Micrometer Tracing / OpenTelemetry] |
| Error tracking  | [Sentry / Elastic APM / None]                 |

### Debugging Cheat Sheet

| Task                        | Command / URL                          |
| --------------------------- | -------------------------------------- |
| View logs                   | [how to access logs locally]           |
| Change log level at runtime | [Actuator loggers endpoint or restart] |
| Check health                | [health endpoint URL]                  |
| View metrics                | [metrics endpoint URL]                 |

<!-- In Quick mode, only "View logs" and "Check health" rows are required -->

## 7. CI/CD & Deployment

> **Full report**: [cicd-deployment-report.md](./cicd-deployment-report.md) *(Deep mode only)*

| Attribute         | Value                                                 |
| ----------------- | ----------------------------------------------------- |
| CI platform       | [GitHub Actions / Jenkins / GitLab CI / Azure DevOps] |
| Deployment target | [Kubernetes / ECS / VMs / Serverless]                 |
| Environments      | [dev / staging / prod]                                |
| Release process   | [tags / SemVer / manual]                              |
| Deploy trigger    | [auto on merge / manual approval / tag push]          |

### Pipeline Stages

| Stage | What It Does | Gate? |
| ----- | ------------ | ----- |
|       |              |       |

<!-- In Quick mode, Pipeline Stages table is the only required sub-section -->

## 8. Code Quality & Conventions

> **Full report**: [code-quality-report.md](./code-quality-report.md) *(Deep mode only)*

### What Will Fail My PR?

| Check | Tool | How to Fix Locally |
| ----- | ---- | ------------------ |
|       |      |                    |

### Essential Commands

```bash
# Auto-format code
[mvn spotless:apply / ./gradlew spotlessApply]

# Run all quality checks locally
[mvn verify / ./gradlew check]
```

### Git Conventions

| Aspect          | Convention                                         |
| --------------- | -------------------------------------------------- |
| Branch naming   | [e.g., `feature/JIRA-123-description`]             |
| Commit messages | [Conventional Commits / free-form / ticket prefix] |

<!-- "What Will Fail My PR?" table and "Essential Commands" are the only P0 items in Quick mode -->

## 9. Things to Ask the Team

These are important onboarding topics that **cannot** be answered from the codebase alone. You now have enough context to ask informed questions.

### Team Process
- [ ] What's the standup cadence and sprint ritual schedule?
- [ ] What does the PR review process look like? How many approvals?
- [ ] Is there an on-call rotation? How does escalation work?

### Monitoring & Access
- [ ] How do I access production/staging logs?
- [ ] Where are the dashboards?
- [ ] How do I get VPN / cluster / environment access?

### Domain Knowledge
- [ ] [Questions about business logic discovered in the code]
- [ ] [Questions about architectural decisions that aren't self-evident]

### Gaps Found in the Codebase
- [ ] [List each gap discovered — missing CI, no migration tool, no API docs, etc.]

## 10. First-Week Action Plan

<!-- Generate this plan based on findings. Adapt each day to what was actually discovered. Use the defaults below as a starting point, but replace actions that don't apply. -->

| Day | Goal                   | Action                                                                                                                               |
| --- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Build & run locally    | Clone, build, run tests, hit the health endpoint                                                                                     |
| 2   | Understand the domain  | Read key entities, trace one API request end-to-end                                                                                  |
| 3   | Make your first change | Pick a small bug or task, submit a PR                                                                                                |
| 4   | Understand testing     | [If integration tests exist: "Write a unit test and an integration test" / If not: "Write a unit test for a service method"]         |
| 5   | Shadow operations      | [If CI pipeline exists: "Watch a deployment, review the CI pipeline" / If not: "Ask the team about deployment process and CI plans"] |

<!-- Additional adaptations:
- If no Docker setup exists, Day 1 should note "install dependencies manually — see Local Services table"
- If the project uses contract-first API design, Day 2 should include "review the OpenAPI spec"
- If there's a complex security setup, add "Day 2: Get auth tokens working locally" before domain exploration
-->

## 11. Coverage Notes

<!-- Fill this section to indicate what was and wasn't analyzed -->

**Execution mode**: [Quick / Standard / Deep]

**Areas fully analyzed**: [list]

**Areas summarized only**: [list]

**Areas not analyzed**: [list — with reason: "Run Deep mode for full coverage" or "Not applicable to this project"]

---

*Generated by the codebase-onboarding skill on [Date]*
