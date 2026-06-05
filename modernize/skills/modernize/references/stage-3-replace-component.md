# Stage 3: Replace Component

This is an **internal building block** of the `modernize` skill — not a user-invokable skill. The orchestrator reads this file when starting (or resuming) Stage 3.

**Goal:** Execute the replacement following the **Replacement Plan** from Stage 2 — apply the code changes in the prescribed order, manage dependencies, sweep for every leftover reference, run the build/tests, and produce a **Replacement Report**.

This is the only **side-effecting** stage — it edits code, installs/removes dependencies, and runs builds/tests, all on a feature branch after confirming with the user.

**Contents:** Prerequisites · Tools to use · Step 1 Read the Replacement Plan · Step 2 Prepare the workspace · Step 3 Execute the replacement · Step 4 Final reference sweep · Step 5 Verify the replacement · Step 6 Document the changes · Resuming a partial replacement · Conversational rules · On completion

## Prerequisites

This stage requires a **Replacement Plan** (Stage 2 artifact). If none exists, run Stage 2 first. Do not proceed without one.

## Tools to use

- **Glob**: locate the Replacement Plan document in the workspace.
- **Grep**: the default for exact matches — find imports, calls, and class/type references to replace, and for the import/type reference sweeps. Prefer it over Bash `grep`.
- **Read**: read file contents before editing.
- **Edit**: make code edits. Use `replace_all: true` for a precise repeated rename within a file. Batch independent edits across files in parallel tool calls.
- **Bash**: install/remove dependencies, run the build, lint, type-check and test commands, and run the comprehensive string sweep with `grep -rn` (or `rg --hidden --no-ignore`) to cover hidden dotfile configs and gitignored files the Grep tool skips by default.
- **TodoWrite**: track which step you're in; update status as you progress.
- **Write**: create the final Replacement Report document.

> Claude Code has no LSP-backed rename or "get errors" tool. For symbol renames, use Grep to find every occurrence and Edit (with `replace_all` where safe) to change them. To catch compile/lint errors, run the project's **build, type-check, and lint commands via Bash** (e.g. `tsc --noEmit`, the compiler, the linter) after each batch — that is your error check.

## Steps to follow

Set up a todo list at the start with these steps and update them as you progress.

### Step 1: Read and understand the Replacement Plan

- Locate the Replacement Plan (Glob/Grep for "Replacement Plan" or the component name in markdown files).
- If none exists, run Stage 2 (`stage-2-plan-replacement.md`) first. Do not proceed without one.
- Read it thoroughly. Extract:
  - The API mapping table (old API -> new API, including behavior changes)
  - The execution order and batching strategy
  - The scope (which files/modules to cover, including build/config files)
  - Build and test commands (Section 11)
  - The rollback strategy (Section 14)
  - Risk notes, edge cases, items flagged for manual intervention
  - Build tooling changes (Section 7), dynamic reference changes (Section 8), framework integration changes (Section 9)
- Summarize your understanding of the plan back to the user and confirm before proceeding.

Rules:
- If the plan seems stale or the codebase has changed, spot-check files. If significantly outdated, regenerate it via Stage 2.
- The plan is your contract — follow it. Don't introduce transformations that aren't in it.
- If the plan is missing build/test commands (Section 11), discover them now (read `package.json`, `Makefile`, `build.gradle`, etc.). You need them for verification.

### Step 2: Prepare the workspace

Before any code changes:

1. **Check for a clean working state**: run `git status` via Bash. If there are uncommitted changes, warn the user and ask whether to proceed or stash first.
2. **Create a feature branch** (if not already on one): suggest a branch like `migrate/old-to-new`. Ask the user before creating it.
3. **Install the new dependency**: run the install command from the plan (`npm install new-lib`, `pip install new-lib`, etc.). Verify the install succeeds before proceeding.
4. **Verify baseline**: run the build and test commands to establish a passing baseline. If the baseline is already failing, tell the user — existing failures must not be confused with migration-caused failures. Record baseline pass/fail counts for the report.

Rules:
- Do not skip dependency installation — code edits will cause import errors if the new library isn't installed.
- If the install fails (version/peer-dep conflicts), stop and consult the user — this may mean a pre-flight check was missed.

### Step 3: Execute the replacement

Systematically replace all usages following the plan's execution order. The typical order:

1. **Replace imports**: update all import/require statements to the new component/library.
2. **Transform API calls**: rewrite function calls, props, class instantiations, and method invocations per the API mapping table. Mind the **Behavior Changes** column — syntax replacement alone is insufficient when behavior differs.
3. **Update type definitions**: interfaces, type aliases, generics, and type-level references.
4. **Update build tooling & configs**: apply Section 7 changes — plugin swaps, loader updates, ESLint rules, CI configs, Dockerfile changes.
5. **Update dynamic & string references**: apply Section 8 changes — string references in plugin configs, DI bindings, etc.
6. **Update framework integrations**: apply Section 9 changes — middleware registration, DI bindings, lifecycle hooks.
7. **Update tests and mocks**: rewrite test files — imports, mocks, fixtures, assertions, test utilities. Update assertions that check old behavioral patterns (error types, return values) per the behavior-changes column.
8. **Remove old dependency**: remove the old library from manifests (`npm uninstall old-lib`, etc.). Delete now-unused wrapper/shim files.
9. **Run validation**: after each batch, run the build/type-check via Bash to catch compile and lint errors early.

For each file you modify, note the file path and the specific changes made.

Rules:
- Follow the API mapping table. Don't introduce transformations that weren't agreed upon.
- Use Edit with `replace_all` for safe repeated renames; batch independent edits across files in parallel tool calls.
- If the plan specifies batching (e.g. module-by-module), follow the order and pause between batches for user review.
- **Behavioral-change handling**: when the API mapping table shows breaking behavioral changes, also update calling code that depends on the old behavior. For example, if the old library returns `undefined` and the new returns `null`, update callers that check `=== undefined`.
- **Rollback guidance**: if errors accumulate during replacement, stop and consult the user rather than pushing forward. Note which files were already modified so changes can be reverted. Don't continue replacing if the previous batch introduced unresolved errors. Reference the plan's rollback strategy.

### Step 4: Final reference sweep

After all planned replacements are complete, confirm no references to the old library remain:

1. **Import sweep**: use the Grep tool for old import patterns (`from 'old-lib'`, `require('old-lib')`, `import old_lib`).
2. **String reference sweep**: run Bash `grep -rni "old-lib-name"` (or `rg --hidden --no-ignore`) across the whole codebase to catch string references, comments, docs, and config values — including the hidden dotfile configs and gitignored files the Grep tool skips by default.
3. **Type reference sweep**: use the Grep tool for type names, interfaces, or generics from the old library.
4. **Dependency manifest check**: verify the old library is removed from all manifests (`package.json`, `requirements.txt`, `pom.xml`, etc.) and lock files.

For each remaining reference:
- Missed code reference -> fix it per the API mapping table.
- Comment or documentation -> update it to reference the new library.
- Lock file -> regenerate the lock file via Bash.
- Intentional (migration notes, changelog) -> leave it and note it in the report.

Rules:
- This step is mandatory. Do not skip it.
- The goal is **zero unintentional references** to the old library remaining.
- Use the Grep tool for normal source files; use Bash `grep -rn` (or `rg --hidden --no-ignore`) for the comprehensive string sweep so it includes hidden dotfile configs, hidden directories like `.github/`, and gitignored files the Grep tool skips by default.

### Step 5: Verify the replacement

After all replacements and the sweep:

1. Run the **build command** from the plan via Bash. Check for failures.
2. Run the **type-check command** (e.g. `tsc --noEmit`) if applicable.
3. Run the **lint command** if available. Check for failures.
4. Run the **test suite** from the plan via Bash. Report results.
5. Compare test results against the Step 2 baseline. Any new failures are migration-caused.
6. Present a verification summary: files modified; remaining references (should be zero unintentional); compile/lint errors found and fixed; build result; test results vs. baseline.
7. If errors remain, diagnose and fix them before documenting.

Rules:
- Don't skip verification. Every replacement session must include a validation pass.
- If errors are found, fix and re-verify before moving on.
- Be transparent about failing tests.
- Distinguish pre-existing failures (baseline) from migration-caused failures.

### Step 6: Document the changes

- Write a structured Replacement Report using the template at [../assets/replacement-report-template.md](../assets/replacement-report-template.md).
- Place it where the user specifies, or propose a sensible default (e.g. `docs/migrations/replacement-report-<old>-to-<new>.md`).
- Include real before/after code snippets from the codebase — not fabricated ones.
- Present a brief summary and ask if adjustments are needed.

Rules:
- Every example must come from actual code in the codebase.
- Be thorough in documenting changes and patterns observed.
- Make the documentation clear for future developers who need the rationale and details.
- If the replacement revealed anti-patterns or improvement opportunities, include them in the Recommendations section.

### Resuming a partial replacement (large codebases)

If a previous session completed some batches, read the Replacement Report's **Batch Progress** section and resume from the first incomplete batch. When context runs low, update Batch Progress (completed vs. remaining) before pausing so the next session can pick up cleanly.

## Conversational rules

- **Pacing**: Confirm the plan at Step 1, prepare the workspace at Step 2, show progress during Step 3, report sweep results at Step 4, share verification results at Step 5, produce the report at Step 6.
- **Batch confirmation**: for large replacements (> 20 files), pause between batches and ask the user to review before continuing.
- **Progress updates**: when Step 3 modifies many files, give periodic updates ("Replaced 12/47 files, moving to the test layer").
- **Active listening**: reflect back what you heard before proceeding.
- **Adapt to scale**: small codebases — be thorough. Large codebases — be strategic: batch, summarize, offer to drill down.
- **Don't assume**: if a usage pattern is ambiguous, ask about intent rather than guessing.

## On completion

The Replacement Report is the Stage 3 artifact. Return to the orchestrator's Stage 3 completion instructions in `SKILL.md`.
