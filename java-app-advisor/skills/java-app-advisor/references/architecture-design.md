# Architecture & Design Analysis

Identify structural issues, layering violations, SOLID violations, coupling problems, missing abstractions, and design anti-patterns. Every finding needs a file path, code evidence, severity, and a concrete fix.

**Core principle:** Architecture is the skeleton of maintainability. Bad architecture doesn't crash the app today — it makes every future change harder, slower, and riskier. Find the structural rot before it becomes load-bearing.

## Contents

- Step 1 — Map the architecture
- Step 2 — Layering violations (incl. standalone/batch)
- Step 3 — SOLID principles
- Step 4 — Coupling & cohesion
- Step 5 — Design anti-patterns
- Step 6 — Compile findings

## Tools

- `Grep` — annotations, imports, class/package declarations, the patterns in the tables below.
- `Glob` — package/module structure (`src/main/java/**`, `**/pom.xml`, `**/settings.gradle*`).
- `Read` — read suspect classes in full (architectural issues span whole files); for very large files, grep the symbol then read that region.
- `Agent` (Explore, breadth `medium`/`very thorough`) — fan-out searches: "find all classes over 500 lines", "map all cross-package imports". Re-`Read` any lead before citing it.
- **Code usages / coupling:** `Grep` the class name across the tree to find who depends on it (no `listCodeUsages` tool exists).
- `Bash` — read-only `git log` for the change-frequency signals in Step 3 (e.g. `git log --since=12.months --name-only --pretty=format: -- '*.java' | sort | uniq -c | sort -rn | head`). Skip gracefully if the project isn't a git repo.
- `TodoWrite` — track the steps.

## Step 1 — Map the architecture

Understand the *intended* architecture before judging it.

1. **Architectural pattern** — Layered (Controller → Service → Repository)? Hexagonal/Ports & Adapters? Clean Architecture? Package-by-feature vs. package-by-layer? Event-driven/CQRS? No discernible pattern (a finding in itself)?
2. **Package structure** — list top-level packages under `src/main/java`; identify each one's role (controller, service, repository, model, config, util); flag packages that don't fit the pattern or have ambiguous names.
3. **Module boundaries** (multi-module) — read parent POM / `settings.gradle` for the module list; map the inter-module dependency graph.
4. **Domain model** — find entity/domain classes (`@Entity`, `@Document`, domain package); is the model anemic (just getters/setters) or rich (contains business logic)?
5. **Entry points by framework:**

   | Framework | Detection |
   |---|---|
   | Spring Boot | `@SpringBootApplication`, `@RestController`, `@Controller`, `@Service`, `@Repository` |
   | Micronaut | `Micronaut.run()`, `@Controller`, `@Singleton`, `@Prototype`, `@Infrastructure` |
   | Quarkus | `@QuarkusMain`, `@ApplicationScoped`, `@RequestScoped`, `@Path` (JAX-RS), `@Startup` |
   | Plain Java | `public static void main(String[])`, `Runnable`/`Callable`, scheduler entry points |
   | Standalone/Batch | `CommandLineRunner`, `ApplicationRunner`, `@Scheduled`, Quartz jobs, `@Observes StartupEvent` |

**Present** a brief description of the architecture — pattern, structure, modules, domain model.

## Step 2 — Layering violations

Find places where layers bypass each other or depend in the wrong direction.

| Violation | Detection | Severity |
|-----------|-----------|----------|
| Controller accessing Repository directly | Controller importing `*Repository` | High |
| Service handling HTTP concerns | Service importing `HttpServletRequest`, `@RequestParam`, `@PathVariable` | High |
| Repository containing business logic | Repository methods with complex logic that belongs in the service layer | Medium |
| Domain depending on infrastructure | Entity/domain classes importing Spring/JPA/framework annotations in a hexagonal/clean architecture | Medium |
| Circular package dependencies | Package A imports B and B imports A | High |
| Util/Helper doing business logic | `*Utils`/`*Helper` classes holding domain logic instead of utilities | Medium |
| Cross-feature direct access | Feature A's service calling Feature B's repository, bypassing B's service | High |

**Standalone/batch-specific:**

| Violation | Detection | Severity |
|-----------|-----------|----------|
| God main class | `main()` / `CommandLineRunner.run()` holding all business logic instead of delegating | High |
| No separation in batch | Reader/Processor/Writer logic mixed in one class | High |
| Scheduler doing business logic | `@Scheduled` methods with complex logic instead of delegating | Medium |
| Missing job orchestration | Independent jobs with no coordination strategy (ordering, dependency, failure handling) | Medium |
| IO mixed with business logic | File parsing, network calls, and business rules in the same method | High |

**How to detect:** use the Explore subagent to scan imports across controllers/services/repositories; for each controller check for `*Repository`/`*Dao` imports; for each service check for `javax.servlet.*`/`jakarta.servlet.*`/`org.springframework.web.*`; map cross-package imports for cycles.

## Step 3 — SOLID principles

### Single Responsibility (S)

| Smell | Detection | Severity |
|-------|-----------|----------|
| God class | > 500 lines, > 15 methods, or > 10 injected dependencies | High |
| Service doing unrelated things | Methods spanning different business domains | Medium |
| Controller with business logic | Controller methods with > 20 lines of logic (not just delegation) | Medium |
| Config doing init + business logic | `@Configuration` with `@Bean` methods plus utility methods | Low |

*Detect God classes:* Explore subagent → "find all Java classes over 500 lines"; for each, count injected deps (`@Autowired` / constructor params), public methods, and whether methods span concerns.

### Open/Closed (O)

| Smell | Detection | Severity |
|-------|-----------|----------|
| Long if/else or switch chains | Methods with > 5 branches dispatching on type | Medium |
| Repeated edits to the same files | Frequent changes to the same files for different features (git history) | Medium |
| Missing strategy/factory | Repeated conditional logic that could be polymorphic | Medium |

### Liskov Substitution (L)

| Smell | Detection | Severity |
|-------|-----------|----------|
| Override throws `UnsupportedOperationException` | Subclass refusing the parent contract | High |
| Type-checking in polymorphic code | `instanceof` after retrieving from a base-type collection | Medium |

### Interface Segregation (I)

| Smell | Detection | Severity |
|-------|-----------|----------|
| Fat interfaces | Interfaces with > 10 methods | Medium |
| Empty/stub implementations | Classes implementing an interface with no-op methods | Medium |
| Marker interface with methods | Intended-as-marker interface accumulating methods | Low |

### Dependency Inversion (D)

| Smell | Detection | Severity |
|-------|-----------|----------|
| Concrete-class injection | Injecting concrete classes instead of interfaces | Medium |
| Missing abstractions | `new`-ing dependencies instead of injecting | High |
| Framework coupling in domain | Domain/business classes depending on framework-specific types | Medium |

## Step 4 — Coupling & cohesion

| Issue | Detection | Severity |
|-------|-----------|----------|
| Tight module coupling | Module A imports B's internal packages (not its API/interface) | High |
| Feature envy | Methods using more data from another class than their own | Medium |
| Data clumps | The same group of parameters passed together to many methods | Low |
| Inappropriate intimacy | Classes reaching into each other's internals via reflection/package-private | High |
| Shotgun surgery | One business change touches > 5 files across packages | Medium |
| Missing bounded contexts | All entities in one package with no domain boundaries | Medium |

## Step 5 — Design anti-patterns

| Anti-pattern | Detection | Severity |
|--------------|-----------|----------|
| Anemic domain model | Entities with only getters/setters, all logic in services | Medium |
| Service Locator | `ApplicationContext.getBean()` directly instead of DI | High |
| Magic strings/numbers | Hardcoded config keys, role names, status values | Medium |
| Exception swallowing | Empty catch, or catch-log-and-continue | High |
| God Controller | REST controller with > 10 endpoints across business domains | Medium |
| DTO explosion | A separate DTO for every minor variation, excessive mapping | Low |
| Utility-class abuse | Static utility classes holding business logic | Medium |
| Premature abstraction | Single-implementation interface with no extension point | Low |
| Circular dependency | Two+ beans depending on each other | High |
| Overuse of inheritance | Inheritance hierarchies > 3 deep instead of composition | Medium |
| Missing aggregate roots | JPA entities with bidirectional relationships but no clear ownership | Medium |

## Step 6 — Compile findings

For each finding record: **Category** (Layering / SOLID / Coupling / Anti-Pattern) · **Severity** · **Description** · **Evidence** (`file:line` + snippet) · **Impact** (maintainability, testability, scalability, correctness) · **Recommended fix** (specific refactor, before/after) · **Effort** (XS/S/M/L/XL). Present the compiled findings.
