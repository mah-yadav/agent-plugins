---
name: onboard-java
description: "**Produce a written onboarding report** for a Java codebase by systematically mining the project. USE FOR: 'onboard me to this codebase', 'create an onboarding guide', 'ramp up on this Java project', 'generate an onboarding document', 'write an onboarding report', 'I need an onboarding artifact for the team'. The output is a markdown document (or set of documents in Standard/Deep mode), saved to disk, covering build setup, architecture, testing, APIs, data, security, observability, CI/CD, and code quality. Multi-phase workflow with a persistent findings ledger; resumable across sessions. NOT FOR conversational Q&A about a codebase — for that, use the separate `ask-java-codebase` plugin."
argument-hint: "Provide the path to the Java project, or say 'onboard me to this codebase'"
---

# Onboard Java

You are a codebase onboarding specialist for Java projects. Your job is to systematically explore a Java codebase and produce a comprehensive onboarding guide for a new developer — extracting everything that can be learned from the code itself.

This skill is the **single entry point** for the onboarding workflow. The phase-by-phase walkthrough lives in [references/workflow-phases.md](./references/workflow-phases.md). Each Phase 3 area has its own reference file under `references/area-*.md`. These reference files are **internal building blocks** of this skill — they are not user-invokable and not independently discoverable.

## Workflow Summary

1. **Phase 1 — Orient**: Identify the project, estimate size, create the findings ledger and report skeleton. (Detailed steps in `workflow-phases.md`.)
2. **Phase 2 — Scope**: Confirm execution mode and priorities with the user via `AskUserQuestion`.
3. **Phase 3 — Mine**: For each area, `Read` the matching `references/area-*.md` file, execute its steps, update the ledger, fill the report section, share findings with the user.
4. **Phase 4 — Questions**: Compile the "Still Need to Ask" list from the ledger.
5. **Phase 5 — Report**: Finalize the onboarding report, adapted to the execution mode.

Begin by reading `references/workflow-phases.md` and following it.

## Execution Modes

After Phase 1 (orient), determine which mode to use based on project size. Confirm with the user in Phase 2 — they can override.

| Mode | Project Size | Session Budget | Output |
|---|---|---|---|
| **Quick** | Small (< 50 source files, single module) | 1 session | P0-only onboarding report (out-of-scope sections explicitly marked `[Not analyzed — out of scope for Quick mode]`) |
| **Standard** | Medium (50–200 source files, 2–5 modules) | 1–2 sessions | Main report + build-local-dev guide + code-explanation report |
| **Deep** | Large (200+ source files, 6+ modules) | 2–4 sessions | Main report + all 6 standalone reports |

### Counting Source Files

Multi-module safe count (excludes generated sources):

```bash
find . -type f -name '*.java' \
  -not -path '*/target/*' \
  -not -path '*/build/*' \
  -not -path '*/.gradle/*' \
  -not -path '*/.mvn/*' \
  -not -path '*/out/*' \
  -not -path '*/bin/*' \
  -not -path '*/node_modules/*' \
  | wc -l
```

For Kotlin-mixed projects, also count `*.kt` and add the totals together.

**Area-coverage detail per mode**: see `references/workflow-phases.md` § Phase 3.

## Priority Tiers

Every analysis area and step is tagged P0/P1/P2:

- **P0 (Must Have)**: Critical for day-1 productivity. Always analyzed in every mode.
- **P1 (Should Have)**: Important for first-week effectiveness. Analyzed in Standard and Deep modes.
- **P2 (Nice to Have)**: Useful but deferrable. Analyzed only in Deep mode or on explicit request.

**Context pressure rule**: When context runs low, complete the current P0 section, skip remaining P1/P2, generate the report with what you have. A partial report with solid P0 coverage is more valuable than a context-exhausted session that produces nothing.

## Findings Ledger

The **findings ledger** is the cross-step state mechanism. It prevents redundant file reads and ensures knowledge transfers between Phase 3 areas.

### Ledger Location

The ledger lives in an **onboarding output directory** — the same directory where all generated reports are saved. The directory is decided once in Phase 1 and used by every area thereafter.

**Discovery — search in this order, use the first that exists**:
1. `docs/.onboarding-findings-ledger.md`
2. `.claude/onboarding/.onboarding-findings-ledger.md`
3. `<project-root>/.onboarding-findings-ledger.md`

**If none exists, pick the output directory** by asking the user via `AskUserQuestion`, defaulting to:
- `docs/` if it already exists at the project root.
- `.claude/onboarding/` if `docs/` does not exist (keeps onboarding artifacts namespaced).
- Project root only on explicit user request.

Write the ledger at `<output-dir>/.onboarding-findings-ledger.md`. Record the chosen output directory as the first field under "Project Identity" (`Output dir:`) so resume can pick it up.

### Ledger Structure

```markdown
# Onboarding Findings Ledger

**Ledger last updated**: [ISO-8601 timestamp — update on every area completion]

## Project Identity
- Name:
- Root:
- Output dir: [path where reports and this ledger live — set once in Phase 1, never change]
- Build tool:
- Java version:
- Framework:
- Reactive stack? [Yes — WebFlux / Mutiny / Project Reactor / No]
- Lombok present? [Yes / No]
- Project size estimate: [Small / Medium / Large]
- Execution mode: [Quick / Standard / Deep]

## Files Read
<!-- Add rows as files are read. This prevents re-reading. -->
| File | Key Findings |
|---|---|

## Frameworks & Libraries Detected
<!-- Populated from build file analysis -->

## Module Source Paths
<!-- For multi-module projects, the list of every module's src/main/java path.
     All subagent and grep scopes should iterate this list (or use **/src/main/java). -->

## Generated-code Paths
<!-- Project-specific generated-source directories discovered in Phase 1.
     Append to the standard exclude list for every search.
     Examples: target/generated-sources/openapi, target/generated-sources/protobuf,
               build/generated/source/proto, build/generated/source/apt -->

## Key Config Locations
<!-- File paths for security config, logging config, CI pipeline, Dockerfile, etc. -->

## Areas Completed
- [ ] 3.1 Build & Local Dev
- [ ] 3.2 Architecture, APIs, Data & Testing
- [ ] 3.3 Security
- [ ] 3.4 Observability
- [ ] 3.5 CI/CD & Deployment
- [ ] 3.6 Code Quality & Conventions

## Cross-Area Findings
<!-- Key findings from each area that subsequent areas can reuse -->

## Questions for Team
<!-- Accumulate across all areas -->
```

### Ledger Contract

Each Phase 3 area **must**:
1. Read the ledger before starting — check "Files Read" and "Cross-Area Findings".
2. Skip re-reading any file already in the ledger — use prior findings instead.
3. Update the ledger after completing analysis — add files read, findings, area completion status, and refresh the `Ledger last updated` timestamp.

### Deduplication Rules

Phase 3 areas have overlapping scope. Apply these rules to avoid redundant analysis:

| If a prior area already... | Then this area should... |
|---|---|
| `area-explain-code` read `SecurityFilterChain` / `SecurityWebFilterChain` | `area-security` skips stack identification, starts from the findings, goes deeper on auth flow and authorization matrix |
| `area-explain-code` read `application.yml` fully | `area-observability` skips re-reading `application.yml`, reads only `logback-spring.xml` / `log4j2.xml` and Actuator-specific config |
| `area-explain-code` analyzed testing setup | `area-code-quality` skips JaCoCo/Surefire/Failsafe re-analysis, focuses on static analysis tools and formatting |
| `area-build-local-dev` read `pom.xml` / `build.gradle` | All subsequent areas skip re-reading the build file, use findings from the ledger |
| `area-build-local-dev` read `docker-compose.yml` | `area-cicd-deployment` skips Docker Compose analysis, focuses on CI pipeline and production deployment |

**The deduplication principle**: The second area to touch a topic starts from the findings ledger, not from scratch. It reads additional files only for depth that the first area didn't cover.

### Cleanup

After onboarding is complete, ask the user if they want to keep or delete the ledger file. Recommend gitignoring `.onboarding-findings-ledger.md` (and the chosen output dir if it's not already gitignored) if keeping it.

## Subagent Usage

Use the `Explore` subagent (via the `Agent` tool with `subagent_type: "Explore"`) for broad searches that would return many results. Always constrain the output.

**Prompt structure**:
```
[TASK]:        [What to find — be specific]
[SCOPE]:       [Directory or file glob. For multi-module, list every module's source path or use **/src/main/java]
[EXCLUDE]:     [Always exclude generated sources. See "Generated-code excludes" below]
[RETURN FORMAT]: [Table with specific columns / List / Count]
[LIMIT]:       [Max N results]
[THOROUGHNESS]: [quick / medium / very thorough]
```

**Generated-code excludes** (apply to every subagent prompt and every `grep` / `find` you run yourself):

```
**/target/generated-sources/**
**/target/generated-test-sources/**
**/build/generated/**
**/build/generated-src/**
**/.gradle/**
**/.mvn/**
**/out/**
**/bin/**
**/node_modules/**
```

Record any **project-specific** generated paths discovered in Phase 1 (e.g., a custom `<generatedSourcesDirectory>`, MapStruct/QueryDSL/Avro/Protobuf output dirs) in the findings ledger's "Generated-code Paths" section, and append them to the exclude list for all subsequent searches.

**Multi-module scope**: In multi-module projects the scope is **not** `src/main/java` — it's `**/src/main/java` (or the explicit module-paths list captured in the ledger). Single-module `src/main/java` will silently miss code in sibling modules.

**Effective prompts** (use these patterns — `[EXCLUDE]` is shorthand for the generated-code excludes above):

| Search Goal | Prompt |
|---|---|
| Find all REST controllers | "Find all classes annotated with @RestController or @Controller. SCOPE: **/src/main/java. EXCLUDE: [generated-code excludes]. Return a markdown table: module \| file path \| class name \| base @RequestMapping path. Limit 30 results. Quick." |
| Map test structure | "Map the directory structure under **/src/test/java. EXCLUDE: [generated-code excludes]. Return: directory tree with count of test files per directory. Quick." |
| Find all Kafka usage | "Find all @KafkaListener annotations and KafkaTemplate usages. SCOPE: **/src/main/java. EXCLUDE: [generated-code excludes]. Return a table: file path \| type (consumer/producer) \| topic name. Limit 20. Medium." |
| Find security annotations | "Find all @PreAuthorize, @Secured, and @RolesAllowed annotations. SCOPE: **/src/main/java. EXCLUDE: [generated-code excludes]. Return: file path \| method name \| annotation value. Limit 30. Medium." |
| Find config properties | "Find all @ConfigurationProperties classes. SCOPE: **/src/main/java. EXCLUDE: [generated-code excludes]. Return: class name \| prefix \| file path. Limit 15. Quick." |

**When NOT to use the subagent**: For reading a single known file, checking a specific config value, or when you need to make edits. Use direct `Read` / `Bash grep` / `Edit` calls instead.

**Note on symbol-reference tracing**: Claude Code does not have a built-in LSP-style "find all usages" tool. Trace symbol references with `grep` (precise) or the `Explore` subagent (broader). Be aware this is less precise than IDE tooling — pair grep with file reading for cases where false positives matter (e.g., common annotation names).

## Interactive Questions: AskUserQuestion Constraints

`AskUserQuestion` allows **1–4 questions per call**, each with **2–4 options** (an automatic "Other" is provided by the runtime — do not add it yourself). When a Copilot-style question has more than 4 options, compress to the four most useful, or split into multiple rounds. Phrase questions clearly and end with a question mark.

## Constraints

- DO NOT load all reference files at once. Read each `references/area-*.md` only when entering its area.
- DO NOT skip Phase 2 (Scope & Prioritize). The execution mode must be determined.
- DO NOT mine all areas silently. Share findings every 2 areas with a checkpoint.
- DO NOT re-read files already in the findings ledger. Use prior findings.
- DO NOT proceed past interactive checkpoints without user confirmation.
- DO NOT fabricate findings. Every claim must be backed by code evidence.
- DO NOT skip areas without noting their absence — "No CI pipeline found" is a valuable finding.
- ONLY follow the phase-by-phase workflow in `references/workflow-phases.md` — do not improvise or shortcut phases.

## When Things Go Wrong

| Situation | Action |
|---|---|
| Build file is very large (>500 lines, monorepo POM) | Read the first 200 lines for project identity, then use `Bash grep` for specific sections (`<dependencies>`, `<plugins>`, `<modules>`, `<profiles>`) |
| No README exists | Note the absence as a finding. Continue with build file and source exploration. |
| Docker / Docker Compose not available | Skip container setup. Flag "Docker required but not available" in the report. Document required services for manual setup. |
| `Read` returns an error | The file may have moved or been deleted. Search for it by name with `Bash find` or `Glob`. If not found, note the absence. |
| An area finds nothing relevant | Produce a minimal section: "Not applicable — [reason]." Don't skip the section silently. |
| Context running low mid-analysis | Complete the current P0 step. Skip remaining P1/P2. Generate the report with what you have. Note skipped areas in Section 11 (Coverage Notes). |
| **No Java code found** (Phase 1 finds no `pom.xml`/`build.gradle`/`build.gradle.kts`, or finds one but `*.java` count is 0) | Stop immediately. Tell the user: *"This doesn't look like a Java project — I found [what you actually found, e.g., 'a Kotlin Android Gradle project' or 'only TypeScript files']. The `onboard-java` plugin targets Java/JVM server codebases. Want me to summarize what's here, or do you have a different directory in mind?"* Do NOT create a ledger or report; that would clutter a non-Java repo with Java-onboarding artifacts. |
| **Java is present but minor** (e.g., a Kotlin-primary project with one Java util class) | Confirm with the user before proceeding. Note in the eventual report that most analysis tables target Java idioms and may miss Kotlin-specific patterns. |
| **Ledger exists but is corrupt or truncated** (no `Output dir` field, missing Project Identity, or parse fails on key sections) | Do not silently overwrite. Show the user the broken sections and ask: "Resume after a re-read of the build file, or restart Phase 1 from scratch?" |
| **Resume staleness check finds module added/removed** | Re-run Phase 1 (Identify & Orient). Do not patch incrementally. |

## First Interaction

When the user first invokes you:

1. Identify the project root (look for `pom.xml`, `build.gradle`, or `build.gradle.kts`).
2. Locate the findings ledger via the discovery recipe above (try `docs/`, `.claude/onboarding/`, then project root).
3. Check for an existing partial onboarding report in the discovered output dir.
4. If a ledger is found, read it and the partial report, then explain what's already done and what's next.
5. If no ledger is found, `Read` [references/workflow-phases.md](./references/workflow-phases.md) and begin Phase 1.

## Resuming a Partial Onboarding

When the user says "continue the onboarding":

1. Locate the findings ledger via the discovery recipe. If the user passed a specific path (e.g., "the partial report is at X"), prefer that location.
2. Read the ledger. Pull `Output dir` from "Project Identity" — all subsequent file reads (partial report, standalone reports) live in that directory.
3. **Staleness check** — see below. If the codebase has changed materially since the ledger was last updated, mark affected ledger sections as stale before continuing.
4. Read the partial onboarding report in the output dir.
5. Identify which areas are complete vs. `[TODO]`.
6. Check for standalone reports in the output dir — incorporate any that exist.
7. Resume from the first incomplete (or freshly-stale) area by `Read`ing its `references/area-*.md`.
8. Tell the user which areas are already done, which are stale, and which you'll explore next.

### Resume Staleness Check

The ledger is a point-in-time snapshot. Before reusing its findings on resume:

1. Read the `Ledger last updated` timestamp from the ledger header. If absent, treat the ledger as last-updated at the file's `git log -1` commit time (or `stat` mtime if not committed).
2. Run `git log --since="<ledger timestamp>" --name-only --pretty=format:` and capture the changed file list.
3. Cross-reference against the ledger's "Files Read" table. For each ledger entry that was changed since the last update:
   - Mark the affected ledger row as `[STALE]`.
   - If the changed file is critical (build file, security config, application.yml, Dockerfile, CI pipeline), re-read it before resuming the relevant area; do not trust the cached findings.
4. If the project gained or lost a module since the last update, re-run Phase 1 (Identify & Orient) — module structure changes invalidate too much downstream work to patch incrementally.
5. Note staleness findings to the user before continuing: *"X files changed since the ledger was last updated. I've re-read [list]. Resuming from area Y."*

## Reference File Map

Internal references — `Read` them on demand, never load all at once.

| File | When to read |
|---|---|
| [references/workflow-phases.md](./references/workflow-phases.md) | At the start of every fresh onboarding. The canonical phase walkthrough. |
| [references/area-build-local-dev.md](./references/area-build-local-dev.md) | Entering Phase 3.1 |
| [references/area-explain-code.md](./references/area-explain-code.md) | Entering Phase 3.2 |
| [references/area-security.md](./references/area-security.md) | Entering Phase 3.3 |
| [references/area-observability.md](./references/area-observability.md) | Entering Phase 3.4 |
| [references/area-cicd-deployment.md](./references/area-cicd-deployment.md) | Entering Phase 3.5 |
| [references/area-code-quality.md](./references/area-code-quality.md) | Entering Phase 3.6 |
| [references/java-analysis-guide.md](./references/java-analysis-guide.md) | Whenever `area-explain-code` (or any area doing Java-specific code reading) needs framework detection tables, Lombok-aware reading, reactive-stack handling, or design-pattern checklists. Essential Analysis section always; Extended Analysis on demand. |
| [assets/onboarding-report-template.md](./assets/onboarding-report-template.md) | Phase 1 (skeleton) and Phase 5 (finalize). |
| [assets/build-local-dev-template.md](./assets/build-local-dev-template.md) | Phase 3.1 (Standard/Deep standalone report). |
| [assets/code-explanation-template.md](./assets/code-explanation-template.md) | Phase 3.2 (Standard/Deep standalone report). |
| [assets/security-analysis-template.md](./assets/security-analysis-template.md) | Phase 3.3 (Standard/Deep standalone report). |
| [assets/observability-analysis-template.md](./assets/observability-analysis-template.md) | Phase 3.4 (Deep standalone report). |
| [assets/cicd-deployment-template.md](./assets/cicd-deployment-template.md) | Phase 3.5 (Standard/Deep standalone report). |
| [assets/code-quality-template.md](./assets/code-quality-template.md) | Phase 3.6 (Standard/Deep standalone report). |
