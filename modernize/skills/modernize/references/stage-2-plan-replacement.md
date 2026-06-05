# Stage 2: Plan Replacement

This is an **internal building block** of the `modernize` skill — not a user-invokable skill. The orchestrator reads this file when starting Stage 2.

**Goal:** Using the **Usage Report** from Stage 1, analyze how the old component is used, understand the new replacement, run pre-flight compatibility checks, and produce a concrete, actionable **Replacement Plan** with an API mapping table (including a behavior-changes column), execution order, build/test commands, and a rollback strategy. This plan is the handoff to Stage 3.

This stage is **planning only**: you may run read-only inspections (dependency-tree and version checks), but do not modify code, install/remove dependencies, or run builds. The only file you create is the Replacement Plan. Actual changes happen in Stage 3.

**Contents:** Prerequisites · Tools to use · Step 1 Read the Usage Report · Step 2 Pre-flight compatibility check · Step 3 Scope the replacement · Step 4 Analyze the new component · Step 5 Create the replacement plan (API mapping table) · Step 6 Identify build/test commands · Step 7 Document the plan · Conversational rules · On completion

## Prerequisites

This stage requires a **Usage Report** for the old component (Stage 1 artifact). If none exists, run Stage 1 first. Do not proceed without one.

## Tools to use

- **Glob**: locate the Usage Report document in the workspace.
- **Grep**: exact matches — verify usage counts, find specific references when spot-checking.
- **Read**: read the Usage Report, dependency manifests, and any extra files needed for context.
- **Bash**: run dependency-tree checks, runtime/version compatibility checks, and package-manager commands for pre-flight validation (`npm ls`, `pip check`, `mvn dependency:tree`, `node -v`, etc.).
- **WebFetch / WebSearch**: retrieve official docs or API references for the new component if needed.
- **TodoWrite**: track which step you're in; update status as you progress.
- **Write**: create the final Replacement Plan document.

## Steps to follow

Set up a todo list at the start with these steps and update them as you progress.

### Step 1: Read and understand the Usage Report

- Locate the Usage Report (Glob/Grep for the component name with "Usage Report" in markdown files).
- If none exists, run Stage 1 (`stage-1-explain-usage.md`) first. Do not proceed without one.
- Read it thoroughly. Extract and summarize:
  - How the old component is imported and initialized
  - Core usage patterns and their frequency
  - Wrappers, abstractions, or custom utilities around it
  - Configuration and setup details
  - Error handling patterns
  - How it's tested and mocked
  - **Build tooling and config references** (Section 7)
  - **Dynamic and string-based references** (Section 8)
  - **Framework-specific integrations** (Section 9)
- Summarize your understanding back to the user and ask them to confirm or correct it.

Rules:
- If the report seems stale or the codebase has changed, verify key findings by spot-checking a few files with Grep/Read. If significantly outdated, regenerate it via Stage 1.
- If the user provides additional documentation or context, acknowledge and incorporate it.
- If the report is missing Sections 7-9 (build tooling, dynamic references, framework integrations), flag this — they are critical for complete planning. Regenerate the report or research those areas before proceeding.

### Step 2: Pre-flight compatibility check

Before planning the strategy, verify the new component is compatible with the project:

1. **Runtime version compatibility**: Compare the project's runtime version (Node.js, Python, Java, etc.) against the new library's requirements. Read the version config (`.nvmrc`, `runtime.txt`, `pom.xml`, `package.json` engines, etc.).
2. **Peer dependency conflicts**: Check whether the new library's peer deps conflict with existing dependencies. Inspect the dependency tree via Bash (`npm ls`, `pip check`, `mvn dependency:tree`).
3. **Known incompatibilities**: Check for documented conflicts with libraries already in use.
4. **Lock file implications**: Note whether the lock file needs regeneration and the associated risk.
5. **Bundle size / performance impact** (frontend libraries): Note the size difference. Use Bash (`npm pack --dry-run`) or known bundle-size data.

Present findings. If any blocker is found, stop and discuss before planning.

Rules:
- Don't skip this even for well-known libraries — version mismatches are the most common migration blocker.
- If you can't determine compatibility (e.g. an internal library with no published requirements), flag it as a risk and ask the user.

### Step 3: Scope the replacement

- Use the Usage Report's file count and summary table to gauge scale.
- **<= 20 files**: plan to replace all at once.
- **21-100 files**: batch by module/directory; confirm the order with the user.
- **> 100 files**: propose a sampling strategy — replace one module first as a proof of concept, then proceed after user review.
- **Include build/config files** in the scope count — often missed, but must be part of the replacement.
- Confirm the scope with the user before proceeding.

Rules:
- Don't silently skip files. Be transparent about coverage.
- If the user has a specific area of interest ("start with the API layer"), narrow the initial scope there.

### Step 4: Analyze the new component/library/framework

- Read any user-provided documentation or context and incorporate it into your understanding of the new component and how it differs from the old one.
- If you lack enough context, ask the user for a link, docs, or context.
- Use WebFetch/WebSearch for official docs or API references if needed. If a fetch fails or returns unusable content, explicitly ask the user to paste the relevant API reference.
- Summarize your understanding of the new component, its purpose, and how it differs from the old one. Ask the user to confirm or correct before moving on.

Rules:
- Restate your understanding before moving on.
- Don't fetch docs for well-known libraries (React, Express, lodash, etc.) — you already know them. Only fetch for unfamiliar or internal libraries.
- If a fetch fails, do not guess the API. Ask the user for documentation.

### Step 5: Create the replacement plan

- Based on the Usage Report (Step 1), pre-flight results (Step 2), and the new-component analysis (Step 4), plan how to replace usages. Consider code transformations, compatibility issues, and testing requirements.
- If the replacement is complex, propose a phased approach (module-by-module, adapters/shims, etc.). Consult [migration-patterns.md](./migration-patterns.md) for common strategies and how to choose between them.
- Produce a concrete **API mapping table** that maps each old API surface to its new equivalent. Cross-reference every usage pattern from the Usage Report so nothing is missed. Include a mandatory **Behavior Changes** column for semantic differences beyond syntax:

| Old API | New API | Behavior Changes | Notes |
|---------|---------|------------------|-------|
| `OldButton({ variant })` | `NewButton({ type })` | `variant="primary"` maps to `type="filled"` — visual weight differs slightly | Prop renamed |
| `import { Button } from 'old-lib'` | `import { Button } from 'new-lib'` | None | Direct swap |
| `useOldHook()` | `useNewHook({ strict: true })` | Returns `null` instead of `undefined` on empty state; throws on invalid input instead of returning an error object | Requires extra option; error handling must change |
| `oldLib.format(date, 'YYYY')` | `newLib.format(date, 'yyyy')` | Token syntax changed; `YYYY` is ISO week-year in new lib | Format string tokens differ |

- **Build tooling plan**: for every build config reference (Usage Report Section 7), specify what changes — plugin swaps, loader updates, ESLint rule changes, CI config updates.
- **Dynamic reference plan**: for every dynamic/string reference (Section 8), specify how to handle it — rename strings, update configs, or flag for manual intervention.
- **Framework integration plan**: for every framework integration (Section 9), specify the replacement approach — new middleware registration, updated DI bindings, etc.
- Identify potential risks, edge cases, and anything requiring manual intervention.
- **Create a rollback strategy**: document how to revert — git-based rollback, feature flags, adapter removal, dependency restoration. Be specific.
- Present the full plan to the user for approval.

Rules:
- Be transparent about complexity and risk.
- Factor in user constraints (e.g. "maintain backward compatibility for 3 months").
- Every usage pattern from the Usage Report must have a corresponding entry in the API mapping table. If a pattern has no clear mapping, flag it as needing user input.
- The "Behavior Changes" column is mandatory. If there are no behavioral differences, write "None" — never leave it blank.
- The rollback strategy is mandatory. Even "revert the commit" counts, but it must be stated.

### Step 6: Identify build and test commands

- Discover and document the project's build/test commands so Stage 3 doesn't rediscover them. Read `package.json` scripts, `Makefile`, `build.gradle`, `pom.xml`, `Cargo.toml`, `pyproject.toml`, or equivalent.
- Identify: build command, test command, lint command, type-check command, and any relevant dev scripts.
- Include these in the plan document.

### Step 7: Document the replacement plan

- Write the document using the template at [../assets/replacement-plan-template.md](../assets/replacement-plan-template.md).
- Place it where the user specifies, or propose a sensible default (e.g. `docs/migrations/replacement-plan-<old>-to-<new>.md`).
- Present a brief summary and ask if adjustments are needed.

Rules:
- The plan must be detailed enough that Stage 3 can execute it without re-analyzing the old or new component.
- Include the API mapping table (with behavior changes), execution order, scope/batching strategy, build/test commands, rollback strategy, and risk notes.

## Conversational rules

- **Pacing**: Confirm understanding at Step 1, present pre-flight results at Step 2, confirm scope at Step 3, confirm new-component understanding at Step 4, present and approve the full plan at Step 5, produce the document at Step 7.
- **Active listening**: Reflect back what you heard. "So if I understand correctly..." / "You mentioned X — let me focus on that."
- **Adapt to scale**: Small codebases — be thorough. Large codebases — be strategic: batch, summarize, offer to drill down.
- **Don't assume**: If a usage pattern is ambiguous or has no clear mapping, ask the user rather than guessing.

## On completion

The Replacement Plan is the Stage 2 artifact. Return to the orchestrator's Stage 2 handoff instructions in `SKILL.md`.
