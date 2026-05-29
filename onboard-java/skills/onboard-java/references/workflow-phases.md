# Onboarding Workflow Phases

The phase-by-phase walkthrough for the onboarding workflow. Loaded by `SKILL.md` after the orchestrator finishes its identification work.

The output is a practical onboarding report — not a code explanation, but a **developer operations guide**: how to build, test, deploy, and work with the project day-to-day.

## Core Principle

**Mine the codebase first, ask people later.** Extract the 60–70% of onboarding knowledge that lives in the code, config files, and project structure — so the developer arrives at their first team conversations with informed questions, not blank-slate ones.

## Tools

- **`Read`**, **`Bash`** (for `ls`, `find`, `grep`), **`Glob`**, **`Grep`**: For codebase exploration.
- **`AskUserQuestion`**: Ask the user about experience level and focus areas. (Constraints in `SKILL.md` § Interactive Questions.)
- **`TodoWrite`**: Track which phase you're in. Update status as you move through phases.
- **`Write`**: Produce the final onboarding report and the findings ledger using the [onboarding report template](../assets/onboarding-report-template.md).
- **`Agent`** (with `subagent_type: "Explore"`): Offload heavy read-only searches. See `SKILL.md` § Subagent Usage for prompt templates.
- **`Edit`**: Update the shared findings ledger on disk.
- **`Read`** of `references/area-*.md`: Load the detailed steps for each Phase 3 area.

Set up the todo list at the start with the five phases below and update as you progress.

---

## Phase 1: Identify & Orient (automated — 1 round)

Establish the project landscape quickly.

1. **Find the project root**: Look for `pom.xml`, `build.gradle`, or `build.gradle.kts` at the workspace root or one level down.

2. **Read the project manifest**:
   - For Maven: `pom.xml` — groupId, artifactId, version, parent POM, modules (multi-module?), Java version, Spring Boot version.
   - For Gradle: `build.gradle` / `build.gradle.kts` — plugins, dependencies, Java toolchain.

3. **Read the README** if present — stated purpose, setup instructions, architecture notes.

4. **Map top-level structure**: `Bash ls` or `Glob` on the project root. Note key directories.

5. **Identify the tech stack**: From dependencies, identify frameworks, libraries, and tools. Detect:
   - **Reactive stack**: `spring-boot-starter-webflux`, `reactor-core`, `quarkus-resteasy-reactive`, `mutiny`. If present, record `Reactive stack? = Yes` in the ledger — this changes how Phase 3.2 (explain-code), 3.3 (security), and 3.4 (observability) analyze code.
   - **Lombok**: `org.projectlombok:lombok` dependency. If present, record `Lombok present? = Yes` — Phase 3.2 will need the Lombok-aware reading rules in `java-analysis-guide.md`.

6. **Detect non-standard layout**: Check for deviations that affect how subsequent phases explore:

   | Deviation | Detection | Adaptation |
   |---|---|---|
   | Custom source directories | `<sourceDirectory>` in pom.xml, `sourceSets` in Gradle | Use detected dirs instead of `src/main/java` |
   | Kotlin mixed with Java | `*.kt` files, `kotlin-maven-plugin`, `kotlin("jvm")` plugin | Note Kotlin presence, analyze `.kt` alongside `.java` |
   | Generated code | `target/generated-sources/`, `build/generated/`, `openapi-generator-maven-plugin` | Identify generators, record paths in the ledger's `Generated-code Paths`, skip in architecture analysis |
   | Polyglot build | `package.json` alongside `pom.xml` | Note frontend module. Flag it but don't deep-dive (out of scope) |

   Record any deviations in the findings ledger so all area analyses adapt.

7. **Estimate project size**: Count source files (see `SKILL.md` § Counting Source Files), modules, dependencies to determine Small/Medium/Large.

8. **Multi-module discovery recipe**: Use this concrete procedure before anything else — every Phase 3 area depends on knowing all module paths.

   **Maven**:
   - Read the root `pom.xml`. Extract every `<module>NAME</module>` from `<modules>`.
   - For each `<module>`, the path is `<root>/NAME/pom.xml`. If that child POM also has `<modules>`, recurse — Maven allows nested multi-module trees.
   - Run `find . -name pom.xml -not -path '*/target/*' -not -path '*/node_modules/*'` as a cross-check; any `pom.xml` found that is **not** referenced via `<modules>` from the root may be an orphan or an unrelated submodule — flag it.
   - For each module, the standard source path is `<module>/src/main/java` unless `<sourceDirectory>` overrides it.

   **Gradle**:
   - Read `settings.gradle` or `settings.gradle.kts`. Extract every `include 'PATH'` / `include("PATH")` call. Colons translate to slashes: `include ':services:user'` → `services/user`.
   - Check for **composite builds**: `includeBuild 'PATH'` — those are separate Gradle builds pulled in. List them but treat them as separate projects for analysis unless the user wants them merged.
   - Check for **convention plugins**: a `buildSrc/` directory or `build-logic/` directory. These contain shared build logic, not application code — skip them for architectural analysis but read them once to understand cross-module build rules.
   - Check for a **version catalog**: `gradle/libs.versions.toml`. If present, dependency versions live there, not in individual `build.gradle` files.
   - Cross-check with `find . -name 'build.gradle*' -not -path '*/build/*' -not -path '*/.gradle/*'`.

   **Record in the ledger** (under `Module Source Paths`):
   ```
   | Module | Path | Source path | Priority |
   |---|---|---|---|
   | app | services/app | services/app/src/main/java | 1 (Application) |
   | api | services/api | services/api/src/main/java | 2 (API) |
   | common | libraries/common | libraries/common/src/main/java | 4 (Shared) |
   ```

   Every subagent prompt and `grep` from now on uses this path list (or `**/src/main/java`) as scope.

9. **Multi-module prioritization** (if multi-module): Determine which modules to analyze and in what order.

   | Priority | Module Type | How to Identify |
   |---|---|---|
   | 1st | Application module | Contains `@SpringBootApplication` / `main()` / `@QuarkusMain` |
   | 2nd | API module | Contains `@RestController` / `@Path` / `@Controller` classes |
   | 3rd | Domain/core module | Contains entity/model classes, business logic services |
   | 4th | Shared/common library | Imported by multiple other modules, utility classes |
   | 5th | Infrastructure module | Database config, messaging config, external integrations |
   | Skip | Generated code modules | Code-generation plugins, proto-generated, openapi-generated |
   | Skip | Test-only modules | Modules containing only test utilities or fixtures |

   **Depth by mode**: Quick → deep-dive 1–2 modules, summarize rest. Standard → deep-dive 3–4, summarize rest. Deep → all modules.

**Choose the onboarding output directory** (ledger + reports live here):

1. Apply `SKILL.md` § Ledger Location → discovery recipe. If a ledger already exists, reuse its `Output dir`.
2. If none exists, ask the user via `AskUserQuestion`:
   - `docs/` (default if the project already has `docs/`)
   - `.claude/onboarding/` (default if no `docs/` exists)
   - Project root (only on explicit user request)

**Create the findings ledger**: Use `Write` to create `<output-dir>/.onboarding-findings-ledger.md` with the structure defined in `SKILL.md` § Findings Ledger. Set `Output dir:` to the chosen path. Populate Project Identity (including `Reactive stack?` and `Lombok present?`), Module Source Paths, Generated-code Paths, Files Read (build file + README), and Frameworks Detected sections.

**Create the report skeleton**: Use `Write` to create the onboarding report at `<output-dir>/onboarding-report.md` using the [onboarding report template](../assets/onboarding-report-template.md) with Section 1 (Project Identity) and Section 2 (Tech Stack) filled in. Use the placeholder `[TODO — pending Phase 2 mode selection]` for every other section. After Phase 2 runs, replace these with either content (P0/in-scope sections) or `[Not analyzed — out of scope for <mode>]` (for sections the chosen mode skips). This ensures a partial report exists even if context runs out mid-analysis.

**Resume-instructions block**: At the very top of the report skeleton (above Section 1), insert a `<!-- onboarding-meta -->` HTML comment block:

```markdown
<!-- onboarding-meta
Ledger:  <output-dir>/.onboarding-findings-ledger.md
Report:  <output-dir>/onboarding-report.md
Mode:    [Quick | Standard | Deep]
Started: <ISO-8601>

To resume in a new chat:
  /onboard-java continue
or paste:
  "Continue the onboarding. The partial report is at <output-dir>/onboarding-report.md."
-->
```

Update this block whenever the ledger is updated.

**Present findings to the user**: Summarize — name, purpose, tech stack, Java version, build tool, estimated size, and recommended execution mode.

---

## Phase 2: Scope & Prioritize (interactive — 1 round)

Use `AskUserQuestion` to tailor the exploration. Suggested questions:

- **Execution mode**: Based on my analysis, I recommend [Quick/Standard/Deep] mode. Does that work? (Quick — single report, ~1 session / Standard — main + key reports, 1–2 sessions / Deep — all reports, 2–4 sessions)
- **Experience level**: How familiar are you with Java / Spring Boot / [detected framework]? (Beginner / Familiar / Experienced)
- **Priority areas**: Which area matters most for your first tasks? (Build & Run / Architecture & APIs / Security / CI/CD)
- **Role focus**: What will you primarily work on? (Backend features / Bug fixes / DevOps & deployment / Full stack)

**After receiving answers**:
1. Confirm the execution mode (Quick/Standard/Deep).
2. Map user priorities to area priority tiers — user's "most important" areas become P0, others stay at their default tier.
3. Update the findings ledger with the execution mode.
4. Replace the report's `[TODO — pending Phase 2 mode selection]` placeholders with either content scaffolding (in-scope sections) or `[Not analyzed — out of scope for <mode>]` (out-of-scope).

**Depth calibration**:

| User Level | P0 Area | P1 Area | P2 Area |
|---|---|---|---|
| **Beginner** | Full analysis + code examples + explanations | Key findings + brief examples | Summary table only |
| **Familiar** | Key findings + patterns + gotchas | Summary + notable patterns | One-liner per item |
| **Experienced** | Patterns, conventions, gotchas only | Summary table | Skip or mention |

---

## Phase 3: Systematic Codebase Mining (automated — multiple rounds)

Explore each area methodically. **Load each area's reference file via `Read`**, execute its steps, update the findings ledger, then move on.

**Area priority by execution mode** — this is the canonical source of truth for which areas run in which mode.

| Area | Quick (P0 only) | Standard (P0+P1) | Deep (all) |
|---|---|---|---|
| 3.1 Build & Local Dev | Full analysis | Full + standalone report | Full + standalone report |
| 3.2 Architecture/APIs/Data/Testing | Summary tables (P0 rows of `area-explain-code.md`) | Full + standalone report | Full + standalone report |
| 3.3 Security | Auth mechanism + local dev auth only | Full analysis + standalone report | Full + standalone report |
| 3.4 Observability | Log location + health endpoint only | Summary | Full + standalone report |
| 3.5 CI/CD & Deployment | Pipeline stages + "what fails my PR" only | Full analysis + standalone report | Full + standalone report |
| 3.6 Code Quality | Format command + PR fail checklist only | Full analysis + standalone report | Full + standalone report |

A row tagged "Summary only" in Quick mode produces a section whose content is exactly the per-area "Minimum viable output" or "Quick mode" snippet. Any other P1/P2 content for that section is replaced with `[Not analyzed — out of scope for Quick mode]`.

**Checkpoint cadence**: Share findings with the user after every 2 areas. Ask: "Does this match your understanding? Should I adjust depth on any area?"

---

### Area 3.1: Build & Local Development — P0

**Goal**: Can the developer build and run the project from scratch?

**Load**: `Read` [references/area-build-local-dev.md](./area-build-local-dev.md) and follow its steps.

**Minimum viable output (Quick mode)**: Build command, run command, test command, key environment variables, local services list.

Incorporate key findings into the onboarding report's "Build & Run Locally" section.

---

### Area 3.2: Architecture, APIs, Data & Testing — P0

**Goal**: How is the code structured, what APIs does it expose/consume, how does it manage data, and how does the team test?

**Load**: `Read` [references/area-explain-code.md](./area-explain-code.md) and follow its steps. It in turn refers to [references/java-analysis-guide.md](./java-analysis-guide.md) — read Essential Analysis always; read Extended Analysis only in Standard/Deep mode or when analyzing data layer, design patterns, testing, or security in depth.

**Boundary with other areas**: `area-explain-code` should identify but NOT deeply analyze Spring Security, quality tools, or logging/metrics — those belong to 3.3, 3.6, and 3.4 respectively.

**Minimum viable output (Quick mode)**: Architectural pattern, package-to-layer mapping table, top 5 entities, endpoint summary count, test command, test framework list.

---

### Area 3.3: Security — P1 (P0 in Quick mode: auth mechanism + local dev auth only)

**Goal**: How is the application secured?

**Load**: `Read` [references/area-security.md](./area-security.md) and follow its steps.

**Prior findings check**: If `area-explain-code` already identified the security framework and read the `SecurityFilterChain` (or `SecurityWebFilterChain`) class, pass those findings — start from the detailed authentication flow analysis, not re-discover the stack.

**Minimum viable output (Quick mode)**: Authentication mechanism (one line), how to authenticate locally (step-by-step commands), where secrets are configured.

---

### Area 3.4: Observability — P2 (P0 in Quick mode: log location + health endpoint only)

**Goal**: How is the application monitored and debugged?

**Load**: `Read` [references/area-observability.md](./area-observability.md) and follow its steps.

**Prior findings check**: If `area-explain-code` or `area-build-local-dev` already read `application.yml`, use those findings. Only read `logback-spring.xml` / `log4j2.xml` and custom observability components fresh.

**Minimum viable output (Quick mode)**: Where logs appear locally (console/file), health check URL, how to change log level.

---

### Area 3.5: CI/CD & Deployment — P1 (P0 in Quick mode: pipeline stages + PR gates only)

**Goal**: How does code get from a PR to production?

**Load**: `Read` [references/area-cicd-deployment.md](./area-cicd-deployment.md) and follow its steps.

**Prior findings check**: If `area-build-local-dev` already read `Dockerfile` and `docker-compose.yml`, pass those findings. Focus on CI pipeline files and deployment manifests.

**Minimum viable output (Quick mode)**: CI platform name, pipeline stages list, what gates/checks must pass for a PR merge.

---

### Area 3.6: Code Quality & Conventions — P1 (P0 in Quick mode: PR fail checklist + format command only)

**Goal**: What standards does the team enforce?

**Load**: `Read` [references/area-code-quality.md](./area-code-quality.md) and follow its steps.

**Prior findings check**: If `area-explain-code` already identified quality plugins in the build file (Checkstyle, PMD, JaCoCo), pass those findings. Focus on reading the actual rule configs and ArchUnit tests.

**Minimum viable output (Quick mode)**: What checks will fail a PR (table: check | tool | fix command), auto-format command.

---

## Phase 4: Compile "Still Need to Ask" List

After mining the codebase, compile a list of important onboarding topics that **cannot** be answered from the code alone. Pull accumulated questions from the findings ledger.

Categories:
- **Team process**: Standup cadence, sprint rituals, PR review norms, on-call rotation.
- **Monitoring access**: How to access dashboards, log aggregators, alerting tools.
- **Domain context**: Business rules, why certain architectural decisions were made.
- **Environment access**: VPN, cluster access, secrets, credentials for local dev.
- **Tribal knowledge**: What burned the team before, things that aren't documented.

---

## Phase 5: Finalize Onboarding Report (1 round)

Finalize the onboarding report created in Phase 1. Fill any remaining sections from Phase 3–4 findings. Review the document for completeness and consistency.

**Output adapted to execution mode**:

| Mode | Main Report | Standalone Reports |
|---|---|---|
| Quick | Full (sections filled from P0 analysis, unfilled sections marked `[Not analyzed — out of scope for Quick mode]`) | None |
| Standard | Full | `build-local-dev-guide.md` + `code-explanation-report.md` |
| Deep | Full | All 6: `build-local-dev-guide.md`, `code-explanation-report.md`, `security-analysis-report.md`, `observability-analysis-report.md`, `cicd-deployment-report.md`, `code-quality-report.md` |

**Cross-referencing standalone reports**: The main onboarding report should NOT duplicate full standalone report content. Include key highlights and a "Full report" link.

All files go in `<output-dir>` chosen in Phase 1.

After creating documents, ask the user to review and provide feedback.

---

## Exploration Guidelines

- **Use the Explore subagent** for areas with broad searches. Use the prompt templates in `SKILL.md` § Subagent Usage.
- **Read config files in full** — they're small and dense. Don't skim.
- **Follow the evidence chain**: Flyway dependency → find migration directory. `@KafkaListener` → find topic configuration.
- **Don't guess from file names** — always read actual content.
- **Skip gracefully**: Note absences — "No CI pipeline found" is a valuable finding.
- **Check the findings ledger before every file read** — if it's already been read, use the prior findings.

## Conversational Guidelines

- **Incremental sharing**: Share findings every 2 areas. Don't mine silently.
- **Checkpoint format**: "Here's what I found about [areas]. Does this match? Anything surprising?"
- **Flag surprises**: Unusual patterns, deprecated libraries, missing expected configs.
- **Note absences**: Missing CI, no migration tool, no API docs — these are findings worth reporting.
- **Stay practical**: Everything should answer "what do I need to do?" not just "what does this project have?"
