# Area: Architecture, APIs, Data & Testing (Explain Code)

Reference for Phase 3.2 of the onboarding workflow. Produces a structured code explanation covering architecture, control flow, data flow, design patterns, and testing.

Specializes in **Java** (Spring Boot, Quarkus, Micronaut, Jakarta EE). Always pair this reference with [java-analysis-guide.md](./java-analysis-guide.md).

## Priority Tiers (within this area)

| Section | Priority | Rationale |
|---|---|---|
| 3.1 Architecture and Structure | **P0** | Foundation for everything else |
| 3.2 Dependencies and Integrations | **P1** | Deep coverage belongs in Standard+ |
| 3.3 Data Flow (key entities only) | **P0** | Top-5 entity table is day-1 essential |
| 3.3 Data Flow (full schema, transactions, caching) | **P1** | Deeper detail |
| 3.4 Control Flow (one representative request) | **P0** | One end-to-end trace is day-1 essential |
| 3.4 Control Flow (full endpoint map, error handling, async) | **P1** | Deeper detail |
| 3.5 Design Patterns | **P1** | Important but not day-1 blocking |
| 3.6 Testing (commands + frameworks) | **P0** | Devs must be able to run tests on day 1 |
| 3.6 Testing (strategy, mocking, coverage detail) | **P1** | Useful for writing tests |

**Quick mode**: 3.1 summary, 3.3 key entities, 3.4 one representative flow, 3.6 commands+frameworks. Skip 3.2 and 3.5.

## Boundaries — Identify-Don't-Deep-Dive

This area should **identify but NOT deeply analyze** these topics — record boundary findings in the ledger so the downstream areas can start from them:

| Topic | What to Do | Belongs to Area |
|---|---|---|
| Spring Security config | Note the authentication mechanism (e.g., "OAuth2 Resource Server with JWT"). Do NOT trace the full filter chain or map authorization rules. | 3.3 Security |
| Quality tools (Checkstyle, PMD, etc.) | Note which tools are present in the build file. Do NOT read their config files. | 3.6 Code Quality |
| Logging/metrics config | Note the logging framework and metrics backend. Do NOT read `logback-spring.xml` or analyze custom metrics. | 3.4 Observability |
| CI/CD pipeline files | Note their existence. Do NOT analyze pipeline stages. | 3.5 CI/CD |

## Ledger Read-Before

Read the ledger. Check whether the build file, README, and `application.yml` are already in `Files Read`. Use those findings. Note `Reactive stack?` and `Lombok present?` from Project Identity — they materially change how to read the code.

## Diagrams

Claude Code doesn't render diagrams in chat. Embed Mermaid blocks directly in the output document — viewers that support Mermaid (GitHub, VS Code, IntelliJ, mkdocs) will render them.

## Lombok-Aware Reading

If the build file lists `org.projectlombok:lombok` (check the ledger's `Lombok present?` field), the code you `Read` is incomplete — constructors, getters, setters, `equals`/`hashCode`/`toString`, builders, and loggers are generated at compile time and invisible in source. Before drawing conclusions about a class's dependencies, mutability, or available methods, consult the **Lombok-Aware Reading** subsection of [java-analysis-guide.md](./java-analysis-guide.md) and apply the annotation→generated-code mapping. In particular, `@RequiredArgsConstructor` + `final` fields is the canonical constructor-injection pattern for many Spring codebases — do not flag the absence of an explicit constructor.

## Reactive Stack Check

If the ledger marks `Reactive stack?` as WebFlux/Mutiny/Reactor, **read the Reactive sections of [java-analysis-guide.md](./java-analysis-guide.md) before tracing control flow**. The request lifecycle, security filter chain, error handling, and threading model differ fundamentally from Spring MVC. Misreading a WebFlux app as MVC produces wrong sequence diagrams and wrong analysis.

## Symbol-Usage Tracing

No LSP-backed "find all references" tool exists. Use `Grep` for precise symbol matches, optionally combined with the `Explore` subagent for broader sweeps. Read surrounding context to filter false positives.

## Step 1: Load the Java Analysis Guide

Read [java-analysis-guide.md](./java-analysis-guide.md) — the **Essential Analysis** section always, the **Extended Analysis** section only in Standard/Deep mode or when analyzing data layer, testing, or design patterns in depth.

The guide is split with a `=== END ESSENTIAL ANALYSIS ===` marker. Load by passing `offset` and `limit` to `Read`, or read the whole file when needed.

If the project includes JavaScript or Node.js modules (e.g., a frontend in a monorepo), note their presence in the ledger but do not deep-dive — recommend a separate analysis for non-Java modules.

## Step 2: Deep Analysis

Apply the framework detection tables and checklists from `java-analysis-guide.md` systematically. Present findings incrementally — after each major module or section, share findings and ask: "Does this match your understanding?"

### 2.1 Architecture and Structure — P0

- **Project layout**: Packages/modules/directories and their responsibilities.
- **Architectural pattern**: MVC, layered, hexagonal, microservices, event-driven?
- **Entry points**: Where does execution begin? How is the app bootstrapped?
- **Module relationships**: How do modules depend on each other?

**Minimum viable output**: Package-to-layer mapping table, architectural pattern name, main entry point file.

### 2.2 Dependencies and Integrations — P1 (skip in Quick mode)

- **External dependencies**: Third-party libraries/frameworks and why.
- **Internal dependencies**: Module/class dependency direction.
- **External integrations**: Databases, APIs, message queues, file systems.
- **Configuration**: How is the application configured?

**Minimum viable output**: External integration list (service name, protocol, config key).

### 2.3 Data Flow — P0

- **Data models**: Core entities, what they represent.
- **Data transformations**: How data flows through layers.
- **Persistence**: How data is stored and retrieved.
- **Validation**: Where and how input is validated.

**Minimum viable output**: Top 5 entity table (name, purpose, key fields), database + ORM identified.

### 2.4 Control Flow — P0

- **Request lifecycle**: Trace how a request flows entry-to-response.
- **Routing**: How requests are routed to handlers.
- **Middleware/interceptors**: Processing before/after handlers.
- **Error handling**: How errors propagate, centralized handling.
- **Async patterns**: How async code is managed.
- **Contract-first API detection**: If `openapi-generator-maven-plugin` or similar is present, the OpenAPI spec is the API source of truth — read the spec, not the generated controllers. Note the spec location, generation config, and which parts are generated vs. hand-written (typically the delegate/implementation classes).

**Minimum viable output**: One representative request traced end-to-end with a sequence diagram (Mermaid).

### 2.5 Design Patterns and Idioms — P1

- Identify design patterns in use.
- Note language-specific idioms.
- Call out anti-patterns with reasoning.

**Minimum viable output**: Pattern table (pattern, where used, purpose).

### 2.6 Testing — P0

- Testing strategy: unit, integration, e2e.
- Frameworks: JUnit, Mockito, Testcontainers, AssertJ, etc.
- Mocking strategies.
- **Test commands**: How to run unit tests, integration tests, single test.

**Minimum viable output**: Test framework list, test commands table, naming convention.

## Step 3: Synthesize and Document

1. Review all findings and organize into a coherent narrative.
2. Create the explanation document using the [code explanation template](../assets/code-explanation-template.md).
3. Include real code snippets — never fabricate.
4. Include Mermaid diagrams where they add clarity.
5. Write to `<output-dir>/code-explanation-report.md` (Standard/Deep mode only — Quick mode rolls into the main report).

**Quick mode document**: Fill sections 1–4 (Overview, Tech Stack, Project Structure, Architecture), section 7 (Request Flow — one flow), section 8 (Data Models — key entities), section 13 (Testing — commands and frameworks). Mark other sections `[Not analyzed — out of scope for Quick mode]`.

**Standard/Deep mode**: Fill all sections. In Deep mode, include Observations and Recommendations.

## Ledger Update After

Mark area 3.2 complete. Add:
- Architectural pattern identified
- Key entities and their relationships
- Endpoint count and key endpoints
- Test frameworks and commands
- Boundary findings for downstream areas (security mechanism, quality tools, logging framework)

## Conversational Rules

- **Depth over breadth**: Better to deeply explain fewer modules than superficially cover everything.
- **Adapt to audience**: Experienced developers want patterns and gotchas; newcomers want step-by-step.
- **Don't assume**: If purpose is ambiguous, say so and ask.
- **Progress updates**: Give periodic updates during deep analysis.
- **Explain the why**: Connect implementation details to architectural decisions.
- **Call out noteworthy patterns**: Anti-patterns, clever solutions, tech debt — be opinionated with reasoning.
