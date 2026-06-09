# Technical Debt Analysis

Inventory accumulated debt — the gap between the current state of the code and where it should be. Identify dead code, deprecated usage, inconsistent patterns, duplication, and shortcuts. Every finding needs evidence, severity, and a concrete fix.

**Core principle:** Technical debt is the sum of all shortcuts taken yesterday that slow you down today. Unlike bugs, it doesn't break the app — it breaks the team's velocity. Inventory it so it can be prioritized and paid down.

## Contents

- Step 1 — Code marker inventory
- Step 2 — Dead code detection
- Step 3 — Deprecated API usage
- Step 4 — Inconsistent patterns
- Step 5 — Code duplication
- Step 6 — Missing error handling
- Step 7 — Compile findings

## Tools

- `Grep` — TODO/FIXME/HACK markers, `@Deprecated`, `@SuppressWarnings`, dead-code indicators.
- `Glob` — orphaned files, old config, unused resources.
- `Read` — read suspect classes; context matters for judging intent.
- `Bash` — `mvn dependency:analyze` for unused declared dependencies.
- `Agent` (Explore) — fan-out: "find all TODO comments", "find all `@Deprecated`", "find all `@SuppressWarnings`". Re-`Read` leads before citing.
- **Dead code:** `Grep` the method/class name across the tree and count call sites; 0 references → candidate (no `listCodeUsages` tool).

## Step 1 — Code marker inventory

### 1.1 TODO/FIXME/HACK/XXX comments

| Marker | Meaning | Severity |
|--------|---------|----------|
| `TODO` | Acknowledged incomplete implementation | Medium |
| `FIXME` | Known bug or broken functionality | High |
| `HACK` / `WORKAROUND` | Intentional shortcut, likely fragile | High |
| `XXX` | Dangerous or questionable code | High |
| `TEMP` / `TEMPORARY` | Code intended to be removed | Medium |
| `NOSONAR` | SonarQube suppression — bypassed quality gate | Medium |

*Detect:* grep each marker across `src/main/java/`; read the surrounding context (ticket reference? how old?); categorize as still-relevant, already-fixed-comment-remains, ticketed, or no-plan.

### 1.2 Suppressed warnings

| Pattern | Meaning | Severity |
|---------|---------|----------|
| `@SuppressWarnings("unchecked")` | Type-safety bypass — may hide `ClassCastException` | Medium |
| `@SuppressWarnings("deprecation")` | Using deprecated API without a migration plan | Medium |
| `@SuppressWarnings("rawtypes")` | Raw types — generics ignored | Low |
| `@SuppressWarnings("all")` | Blanket suppression — hides everything | High |
| `//noinspection` | IntelliJ inspection suppression | Low |
| `@SuppressFBWarnings` | SpotBugs finding suppressed | Medium |

## Step 2 — Dead code detection

| Type | Detection | Severity |
|------|-----------|----------|
| Unused private methods | Private methods with 0 references | Medium |
| Unused public methods | Public methods with 0 call sites | Medium |
| Unreachable code | Code after `return`/`throw`, or inside always-false conditions | Medium |
| Unused imports | Imports for unreferenced classes | Low |
| Unused dependencies | Build-file deps not imported anywhere | Medium |
| Commented-out code | Large commented-out blocks (> 5 lines) | Medium |
| Empty classes/interfaces | No methods or only inherited methods | Low |
| Orphaned test classes | Tests for source classes that no longer exist | Low |
| Unused config properties | `application.yml` keys not referenced via `@Value`/`@ConfigurationProperties` | Low |

*Detect:* `mvn dependency:analyze` (Maven) for unused declared deps; `Grep` suspected-unused method names for call sites; grep large commented-out blocks; diff source vs. test class names for orphans.

## Step 3 — Deprecated API usage

| Pattern | Detection | Severity |
|---------|-----------|----------|
| Deprecated Spring APIs | `WebSecurityConfigurerAdapter`, `@EnableGlobalMethodSecurity`, legacy `RestTemplate` for new code | High |
| Deprecated Java APIs | `java.util.Date`, `Calendar`, `SimpleDateFormat` (vs `java.time.*`) | Medium |
| Deprecated JPA/Hibernate | `GenerationType.AUTO` in some contexts, deprecated Hibernate calls | Medium |
| Project's own deprecated code | Internal `@Deprecated` classes/methods still used | Medium |
| Deprecated third-party APIs | Deprecated library methods (compiler/Javadoc warnings) | Medium |
| Jakarta namespace migration | Still using `javax.*` on Jakarta EE 9+ | High |

*Detect:* grep `@Deprecated` then find usages of those symbols; grep `java.util.Date`/`SimpleDateFormat` imports; grep `WebSecurityConfigurerAdapter` (deprecated since Spring Security 5.7); grep `javax.` imports when the project targets Jakarta EE 9+.

## Step 4 — Inconsistent patterns

| Inconsistency | Detection | Severity |
|----------------|-----------|----------|
| Mixed naming conventions | `*Service` vs `*Manager` vs `*Handler` for the same role | Medium |
| Inconsistent error handling | Some throw custom exceptions, some return `null`, some `Optional` | High |
| Mixed injection styles | Field `@Autowired` vs constructor injection | Medium |
| Inconsistent DTO patterns | DTOs vs raw entities vs `Map<String,Object>` across endpoints | High |
| Mixed logging frameworks | SLF4J mixed with `java.util.logging`, or mixed log styles | Low |
| Inconsistent transaction boundaries | Class-level vs method-level vs no `@Transactional` | Medium |
| Mixed API response wrappers | `ResponseEntity<>` vs raw `@ResponseBody` objects | Medium |
| Inconsistent validation | `@Valid` vs manual service-layer validation | Medium |
| Date handling | Mix of `java.util.Date`, `java.time.*`, `java.sql.Timestamp` | Medium |

*Detect:* grep class-name suffixes within a layer; grep `@Autowired` vs constructor injection; grep `throw new`/`return null`/`Optional` to compare error styles; compare controller return types.

## Step 5 — Code duplication

| Type | Detection | Severity |
|------|-----------|----------|
| Duplicated business logic | Same validation/calculation in multiple service methods | High |
| Duplicated query patterns | Near-identical JPQL/SQL across repositories | Medium |
| Duplicated exception handling | The same try/catch repeated across services | Medium |
| Copy-paste configuration | Near-identical `@Bean` definitions across config classes | Low |
| Duplicated mapping code | Manual entity↔DTO mapping repeated (should use MapStruct) | Medium |
| Duplicated test setup | Same `@BeforeEach` across test classes | Low |

*Detect:* look for similar method names across classes; grep similar JPQL/SQL strings; grep repeated try/catch patterns; compare `@Bean` methods across `@Configuration` classes.

## Step 6 — Missing error handling

| Issue | Detection | Severity |
|-------|-----------|----------|
| Empty catch blocks | `catch (Exception e) { }` with no logging or rethrow | Critical |
| Catch-and-log-only | `catch (...) { log.error(...) }` without recovery or rethrow | High |
| Pokémon exception handling | `catch (Exception e)` catching everything instead of specific types | Medium |
| Missing null checks at boundaries | Service methods not validating input params for null | Medium |
| Swallowed checked exceptions | `throw new RuntimeException(e)` without context | Medium |
| Missing global handler | No `@ControllerAdvice`/`@ExceptionHandler` for consistent error responses | High |
| Missing future error handling | `CompletableFuture` chains without `exceptionally()`/`handle()` | High |
| Missing rollback rules | `@Transactional` without `rollbackFor` for checked exceptions | Medium |

*Detect:* grep empty catch blocks; grep `catch (Exception `; grep `throw new RuntimeException(`; check for `@ControllerAdvice` + `@ExceptionHandler`.

## Step 7 — Compile findings

For each finding: **Category** (Code Markers / Dead Code / Deprecated API / Inconsistency / Duplication / Error Handling) · **Severity** · **Description** · **Evidence** (`file:line` + snippet) · **Impact** (maintenance cost, bug risk, onboarding friction) · **Recommended fix** (before/after) · **Effort** (XS/S/M/L/XL). Present the compiled findings.
