# Stage 1: Explain Usage

This is an **internal building block** of the `modernize` skill — not a user-invokable skill. The orchestrator reads this file when starting Stage 1.

**Goal:** Identify, analyze, and document how the old component/library/framework is used across the codebase. Produce a structured **Usage Report** with real examples and patterns — covering code usage plus build tooling, dynamic references, and framework-specific integrations. This report is the handoff to Stage 2.

This stage is **read-only**: explore and document, but do not modify code, install dependencies, or run builds. The only file you create is the Usage Report.

**Contents:** Tools to use · Step 1 Understand the component · Step 2 Scope the search · Step 3 Search and collect usage (9 dimensions) · Step 4 Analyze patterns · Step 5 Create the Usage Report · Conversational rules · On completion

## Tools to use

- **Explore subagent** (Agent tool, `subagent_type: "Explore"`): for broad discovery — fan out to find files and code related to the component by concept, and to trace references of a symbol across the codebase. Specify breadth `medium` for a focused sweep or `very thorough` for many locations/naming conventions. It offloads heavy read-only research and returns `file:line` **leads**, not ground truth — re-`Read` any spot before quoting it.
- **Grep**: the default for exact matches — imports, function/class references, configuration by name. Prefer it over Bash `grep` (faster, glob-scoped).
- **Glob**: find files by name or path pattern (config files, test files related to the library).
- **Read**: read actual file contents — never infer usage from file names alone. For a large source file, Grep to the relevant line then `Read` that region rather than the whole file.
- **Bash**: inspect dependency trees (`npm ls`, `pip show`, `mvn dependency:tree`), and run a raw recursive `grep -rn` (or `rg --hidden --no-ignore`) only for the string sweep over hidden dotfile configs (`.eslintrc`, `.babelrc`), hidden directories (`.github/`), and gitignored files — which the Grep tool skips by default.
- **WebFetch / WebSearch**: retrieve official docs or API references when needed.
- **TodoWrite**: track which step you're in; update status as you progress so the user has visibility.
- **Write**: create the final Usage Report.

## Steps to follow

Set up a todo list at the start with these steps and update them as you progress.

### Step 1: Understand the component or library

- Search the codebase for the component/library definition, README, or inline docs (Explore subagent for concept discovery, Grep for exact names).
- If it's a third-party library you don't confidently know, use WebFetch/WebSearch for official docs.
- If you can't find enough context, ask the user for a link, docs, or context.
- Summarize your understanding of what it is and its intended purpose. Ask the user to confirm or correct before moving on.

Rules:
- If the user provides documentation or context, acknowledge it and incorporate it.
- Restate your understanding back before moving on.
- Don't fetch docs for well-known libraries (React, Express, lodash, etc.) — you already know them. Only fetch for unfamiliar or internal libraries.

### Step 2: Scope the search

- Use Grep to count how many files import or reference the component/library.
- If usage spans **<= 20 files**: plan to analyze all of them.
- If usage spans **> 20 files**: tell the user the scale and ask which areas to prioritize, or propose a sampling strategy (one example per usage pattern, focus on core modules). Use AskUserQuestion when the choice is discrete.
- Confirm the scope with the user before proceeding.

Rules:
- Don't silently skip files. Be transparent about coverage.
- If the user has a specific area of interest ("how do we use it in the API layer?"), narrow the search to that area.

### Step 3: Search and collect usage

Search systematically across these dimensions using Grep, the Explore subagent, and Bash:

1. **Imports & initialization**: How is it imported? Is there a wrapper or re-export? Any custom setup/configuration?
2. **Core usage patterns**: The main ways it's called or rendered. What parameters/props are commonly passed?
3. **Configuration**: Config files, environment variables, or setup code related to it.
4. **Composition & wrapping**: Higher-order components, custom hooks, decorators, or utility functions built around it.
5. **Error handling**: How errors from this library/component are handled.
6. **Testing**: How it's mocked or tested; any test utilities built around it.
7. **Build tooling & config references**: References in build configuration — babel plugins, webpack/vite/rollup loaders or plugins, ESLint rules/plugins, CI/CD pipeline configs, Dockerfiles, build scripts. Use Glob for common config files (`*.config.{js,ts}`, `.babelrc`, `.eslintrc*`, `webpack.*`, `vite.*`, `rollup.*`, `Makefile`, `Dockerfile`, `Jenkinsfile`, `.github/workflows/*`) and Grep within them.
8. **Dynamic & string-based references**: Search for the library name as a plain string (not just import syntax) to catch dynamic property access (`lib[methodName]()`), plugin configs referencing it by name, DI bindings, reflection-based usage, and documentation/comments. Use the Grep tool first; then run Bash `grep -rn` (or `rg --hidden --no-ignore`) to also cover hidden dotfile configs and gitignored files the Grep tool skips by default.
9. **Framework-specific integrations**: Middleware registration, DI container bindings, lifecycle hooks, route/handler registrations, decorator usage, and framework patterns (Spring `@Bean`, Express `app.use()`, Angular module declarations, NestJS providers).

For each file you examine, note the file path and the specific pattern.

Rules:
- Read actual code — don't guess from file names.
- Treat Explore/search results as leads, not ground truth — re-`Read` the cited `file:line` before using a snippet as evidence in the report.
- Collect concrete code snippets as evidence; you'll need them for the document.
- If you find a wrapper or abstraction layer, trace through it to understand what it adds.
- Dimensions 7-9 are often where migrations break. Don't treat them as optional.

### Step 4: Analyze patterns

Synthesize what you found:

- **Common patterns**: The 2-5 most common ways it's used, ranked by frequency.
- **Variations**: Inconsistencies across the codebase — different teams/modules doing it differently.
- **Idioms**: Codebase-specific conventions built around it.
- **Anti-patterns**: Usage that looks problematic, deprecated, or inconsistent with the library's intended API.
- **Architectural role**: Where it fits — core dependency or peripheral.
- **Build/tooling footprint**: Build configs, plugins, or tooling that depends on it.
- **Dynamic/non-import references**: String-based, dynamic, or reflection-based references found.

Present findings and ask: "Does this match your understanding? Are there patterns I missed? Any areas to explore deeper?"

Rules:
- Be opinionated — call out anti-patterns or inconsistencies with reasoning.
- Don't just list files. Explain *what* you see and *why* it matters.

### Step 5: Create the Usage Report

- Write the document using the template at [../assets/usage-report-template.md](../assets/usage-report-template.md).
- Place it where the user specifies, or propose a sensible default (e.g. `docs/migrations/usage-report-<component>.md`).
- Include real code snippets from the codebase — not fabricated ones.
- Present a brief summary and ask if adjustments are needed.

Rules:
- Every example must come from actual code found in the codebase.
- If the analysis revealed anti-patterns or improvement opportunities, include a "Recommendations" section.
- The document must cover all 9 search dimensions from Step 3. If a dimension had no findings, state "None found" explicitly rather than omitting it.

## Conversational rules

- **Pace**: Don't dump all findings at once. Confirm understanding at Step 1, confirm scope at Step 2, share analysis at Step 4, then produce the document at Step 5.
- **Active listening**: Reflect back what you heard. "So if I understand correctly..." / "You mentioned X — let me focus on that."
- **Adapt to scale**: Small codebases — be thorough. Large codebases — be strategic: sample, summarize, offer to drill down.
- **Don't assume**: If a usage pattern is ambiguous, ask about intent rather than guessing.
- **Progress updates**: When Step 3 searches many files, give periodic updates so the user isn't waiting in silence.

## On completion

The Usage Report is the Stage 1 artifact. Return to the orchestrator's Stage 1 handoff instructions in `SKILL.md`.
