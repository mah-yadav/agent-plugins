# Replacement Plan: [Old Component] → [New Component]

**Date**: [Date]
**Codebase**: [Repository or project name]
**Usage Report**: [Path to the usage report document used as input]
**Status**: Draft / Approved

## 1. Overview

[2-3 sentence summary: what is being replaced, why, and the high-level approach.]

## 2. Component/Library Summary

| Aspect          | Old | New |
| --------------- | --- | --- |
| Name            |     |     |
| Version         |     |     |
| Purpose         |     |     |
| Key differences |     |     |

## 3. Pre-Flight Compatibility

| Check                         | Result     | Details                                                   |
| ----------------------------- | ---------- | --------------------------------------------------------- |
| Runtime version compatible    | ✅ / ❌ / ⚠️  | [e.g., "Requires Node ≥ 18, project uses 20.x — OK"]      |
| Peer dependency conflicts     | ✅ / ❌ / ⚠️  | [e.g., "No conflicts found" or "Conflicts with react@17"] |
| Known incompatibilities       | ✅ / ❌ / ⚠️  | [e.g., "None documented"]                                 |
| Lock file regeneration needed | Yes / No   | [e.g., "Lock file will be regenerated after install"]     |
| Bundle size impact            | +/- [size] | [e.g., "+12KB gzipped" or "N/A for backend library"]      |

[If any blocker was found, note the resolution agreed with the user.]

## 4. Scope

**Total files affected**: [count — including build/config files]
**Code files**: [count]
**Build/config files**: [count]
**Test files**: [count]
**Batching strategy**: [All at once / Module-by-module / Sampling first]

| Batch | Module/Directory | File count | Order |
| ----- | ---------------- | ---------- | ----- |
| 1     |                  |            |       |
| 2     |                  |            |       |

## 5. Migration Pattern

**Chosen pattern**: [Direct Swap / Adapter / Module-by-Module / Strangler Fig / Big Bang]
**Rationale**: [Why this pattern was chosen based on the codebase characteristics.]

## 6. API Mapping Table

| Old API | New API | Behavior Changes | Notes |
| ------- | ------- | ---------------- | ----- |
|         |         |                  |       |

**Behavior changes legend**:
- **None**: Drop-in replacement, no semantic difference.
- **Minor**: Different defaults, renamed options, or cosmetic differences.
- **Breaking**: Different return types, error handling, async behavior, or side effects that require code changes beyond the API call itself.

## 7. Build Tooling & Config Changes

| Config File | Current Reference | Required Change |
| ----------- | ----------------- | --------------- |
|             |                   |                 |

[If no build tooling changes needed, state "No build tooling changes required."]

## 8. Dynamic & String-Based Reference Changes

| Location | Current Reference | Required Change |
| -------- | ----------------- | --------------- |
|          |                   |                 |

[If no dynamic references need changing, state "No dynamic reference changes required."]

## 9. Framework Integration Changes

| Integration Point | Current Implementation | New Implementation |
| ----------------- | ---------------------- | ------------------ |
|                   |                        |                    |

[If no framework integration changes needed, state "No framework integration changes required."]

## 10. Execution Order

1. **Preparation**: [Create feature branch, ensure clean working state]
2. **Update dependency files**: [What to add/remove in dependency manifests]
3. **Install dependencies**: [Exact install command to run, e.g., `npm install new-lib && npm uninstall old-lib`]
4. **Create adapters/shims**: [Describe adapter if needed, or "N/A"]
5. **Replace imports**: [Old import → New import patterns]
6. **Transform API calls**: [Key transformations to apply, referencing API mapping table]
7. **Update type definitions**: [Type-level changes needed]
8. **Update build tooling & configs**: [Changes from Section 7]
9. **Update dynamic/string references**: [Changes from Section 8]
10. **Update framework integrations**: [Changes from Section 9]
11. **Update tests and mocks**: [Test changes needed]
12. **Remove old dependency**: [What to clean up after full replacement]
13. **Final reference sweep**: [Grep for ALL remaining references to old library — imports, strings, comments]
14. **Verification**: [Run build, tests, lint using commands from Section 11]

## 11. Build & Test Commands

| Command    | Purpose | Invocation                 |
| ---------- | ------- | -------------------------- |
| Build      |         | [e.g., `npm run build`]    |
| Test       |         | [e.g., `npm test`]         |
| Lint       |         | [e.g., `npm run lint`]     |
| Type check |         | [e.g., `npx tsc --noEmit`] |

## 12. Risks and Edge Cases

| Risk | Likelihood   | Impact       | Mitigation |
| ---- | ------------ | ------------ | ---------- |
|      | High/Med/Low | High/Med/Low |            |

## 13. Items Requiring Manual Intervention

- [Any usage patterns that can't be mechanically transformed]
- [Edge cases flagged during planning]
- [Dynamic references that need runtime verification]

## 14. Rollback Strategy

**Approach**: [e.g., "Git revert — all changes are in a single feature branch" / "Remove adapter, restore old imports" / "Feature flag toggle"]

**Steps to rollback**:
1. [Step 1]
2. [Step 2]
3. [Step 3]

**Rollback constraints**: [e.g., "Must rollback before data migration runs" / "No constraints — pure code change"]

## 15. Constraints

- [User-specified constraints, e.g., backward compatibility requirements, timeline, etc.]

## 16. Next Steps

Run the `replace-component` skill to execute this plan.
