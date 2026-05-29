# Area: Code Quality & Conventions Analysis

Reference for Phase 3.6 of the onboarding workflow. Produces a structured report covering static analysis, formatting, architecture tests, dependency governance, and git workflow conventions.

**Core principle**: Know what will reject your code before you write it. Nothing slows onboarding more than PR failures from unknown conventions.

## Priority Tiers (within this area)

| Step | Priority | Rationale |
|---|---|---|
| Step 1: Identify quality stack | **P0** | Must know what's enforced |
| Step 4.1: Formatting tool + auto-format command | **P0** | Most common PR failure for new devs |
| Step 3 (summary): Which static analysis tools fail the build | **P0** | Must know what blocks PRs |
| Step 5: Architecture tests (ArchUnit) | **P1** | Must know layer rules |
| Step 3 (detail): Checkstyle/PMD/SpotBugs rule details | **P1** | Important for understanding violations |
| Step 4.3–4.4: IDE setup, import ordering | **P1** | Important for first-day setup |
| Step 6: Dependency management | **P1** | How to add dependencies correctly |
| Step 7: Git hooks & commit conventions | **P1** | Workflow conventions |
| Step 3.4–3.5: Error Prone, SonarQube details | **P2** | Deep detail |
| Step 6.3: License compliance | **P2** | Rarely blocks development |

**Quick mode**: Complete Step 1, Step 3 (summary: which tools fail the build), Step 4.1 (format command). Skip everything else.

## Ledger Read-Before

Check the ledger. Do NOT re-read the build file to discover quality plugins — pull from the ledger. If `explain-code` noted JaCoCo/Surefire/Failsafe config, use those findings. If ArchUnit tests were identified, start from those findings. Only read quality-specific config files not yet analyzed: `checkstyle.xml`, `pmd-ruleset.xml`, `spotbugs-exclude.xml`, `.editorconfig`, ArchUnit test classes.

## Symbol Tracing

Use `Grep` for `@SuppressWarnings`, `@ArchTest`, etc.

## Step 1: Identify the Code Quality Stack — P0

1. **From the build file** (or ledger), identify quality-related plugins:

   **Maven**:

   | Plugin / Dependency | Indicates |
   |---|---|
   | `maven-checkstyle-plugin` | Checkstyle enforcement |
   | `maven-pmd-plugin` | PMD static analysis |
   | `spotbugs-maven-plugin` | SpotBugs bug detection |
   | `error_prone` | Compile-time checks |
   | `spotless-maven-plugin` | Formatting enforcement |
   | `fmt-maven-plugin` | google-java-format |
   | `maven-enforcer-plugin` | Dependency governance |
   | `sonar-maven-plugin` | SonarQube analysis |
   | `jacoco-maven-plugin` | Coverage thresholds |
   | `com.tngtech.archunit:archunit-junit5` | Architecture tests |
   | `dependency-check-maven` | OWASP vulnerability scan |

   **Gradle equivalents**:

   | Gradle Plugin ID | Equivalent Maven Plugin |
   |---|---|
   | `checkstyle` | `maven-checkstyle-plugin` |
   | `pmd` | `maven-pmd-plugin` |
   | `com.github.spotbugs` | `spotbugs-maven-plugin` |
   | `net.ltgt.errorprone` | `error_prone` compiler plugin |
   | `com.diffplug.spotless` | `spotless-maven-plugin` |
   | `jacoco` | `jacoco-maven-plugin` |
   | `org.sonarqube` | `sonar-maven-plugin` |
   | `org.owasp.dependencycheck` | `dependency-check-maven` |

2. **Find quality config files**:

   | File | Tool |
   |---|---|
   | `checkstyle.xml`, `config/checkstyle/` | Checkstyle rules |
   | `pmd-ruleset.xml`, `pmd.xml` | PMD rules |
   | `spotbugs-exclude.xml` | SpotBugs filters |
   | `.editorconfig` | Editor style settings |
   | `sonar-project.properties` | SonarQube config |
   | `.pre-commit-config.yaml` | Pre-commit hooks |
   | `.github/PULL_REQUEST_TEMPLATE.md` | PR template |
   | `CODEOWNERS` | Code ownership |

3. **Check which tools fail the build** vs. just warn.

**Present findings**: Quality stack — what tools exist, which fail the build.

## Step 2: Analyze Static Analysis — P0 summary / P1 detail

### 2.1 Checkstyle — P1

Read Checkstyle config file. Identify: ruleset base (Google/Sun/custom), key rules (naming, Javadoc, imports), suppressions, severity (error vs. warning).

**Extract**: Key rules likely to fail new code, how to suppress, IDE plugin.

### 2.2 PMD — P1

Read PMD ruleset. Identify: active categories, custom rules, failure threshold.

### 2.3 SpotBugs — P1

Effort level, confidence threshold, exclude filter, extra plugins (findsecbugs).

### 2.4 Error Prone — P2

Compiler flags, severity overrides, disabled checks.

### 2.5 SonarQube / SonarCloud — P2

Project key, server URL, quality gate rules (coverage threshold, duplication, issue counts), PR decoration.

## Step 3: Analyze Code Formatting — P0/P1

### 3.1 Formatting Tool — P0

| What to Find | Where to Look |
|---|---|
| Spotless | `spotless-maven-plugin` or Gradle plugin |
| google-java-format | `fmt-maven-plugin` |
| Formatter type | google-java-format, Palantir, Eclipse |
| Check vs. apply | `check` goal (fails build) vs. `apply` goal (fixes) |

**Extract**: Which formatter, auto-format command (`mvn spotless:apply`), check command, whether CI enforces it.

### 3.2 EditorConfig — P1

`.editorconfig` contents: indent style, indent size, line endings, trailing whitespace.

### 3.3 IDE-Specific Settings — P1

IntelliJ/Eclipse/VS Code settings committed to repo, shared run configs.

**Extract**: IDE setup instructions — import formatter, enable format-on-save.

### 3.4 Import Ordering — P1

Import group ordering (java → javax → org → com → project), wildcard import threshold.

## Step 4: Analyze Architecture Tests — P1

| What to Find | Where to Look |
|---|---|
| ArchUnit dependency | `archunit-junit5` in build file |
| Test classes | `@ArchTest`, `@AnalyzeClasses` in test sources |
| Layer rules | `layeredArchitecture()` |
| Naming conventions | `classes().that().implement()...should().haveSimpleNameEndingWith()` |
| Cycle detection | `slices().should().beFreeOfCycles()` |
| Frozen rules | `FreezingArchRule` |

**Extract**: Table of all architecture rules, layer dependency diagram, how to run architecture tests.

## Step 5: Analyze Dependency Management — P1

### 5.1 BOM & Version Management — P1

BOM imports, version properties, version catalogs, parent POM.

### 5.2 Dependency Enforcement — P1

Enforcer plugin rules, banned dependencies, required Java/Maven versions, vulnerability scanning.

**Extract**: How to add a new dependency correctly, banned libraries, vulnerability scanning tool.

### 5.3 License Compliance — P2

License plugin, required headers, allowed licenses.

## Step 6: Analyze Git Hooks & Commit Conventions — P1

### 6.1 Git Hooks — P1

Pre-commit framework, what hooks check, how to install, how to bypass.

### 6.2 Commit Message Conventions — P1

Conventional Commits, ticket reference requirements, changelog generation.

### 6.3 Branch Naming & PR Standards — P1

Branch naming convention, PR template, CODEOWNERS, required checks.

## Step 7: Generate Report

Write the report using the [code quality template](../assets/code-quality-template.md).

**Quick mode**: Fill sections 1 (Quality Stack), 3.1 (Formatter with commands), and the "What Will Fail My PR?" table. Mark other sections `[Not analyzed — out of scope for Quick mode]`.

Default location: `<output-dir>/code-quality-report.md` (Standard/Deep only).

## Ledger Update After

Mark area 3.6 complete. Add:
- What will fail a PR (summary table)
- Auto-format command
- IDE setup instructions
- Git conventions

## Exploration Guidelines

- **Read config files in full** — Checkstyle, PMD, Spotless configs define what passes.
- **Distinguish enforcement from advice**: `error` severity fails the build, `warning` doesn't.
- **Check CI for quality gates**: Tools might run in CI even if not in the build file.
- **Note what's missing**: No formatter? No architecture tests? Valuable findings.
- **Prominently feature the auto-format command** — it's the most useful command for a new dev.
