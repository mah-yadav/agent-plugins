# Replacement Report: [Old Component] → [New Component]

**Date**: [Date]
**Codebase**: [Repository or project name]
**Scope**: [What was replaced — full codebase, specific modules, etc.]
**Replacement Plan**: [Path to the replacement plan document]

## 1. Overview

[2-3 sentence summary: what was replaced, why, and the overall outcome.]

## 2. Component/Library Summary

| Aspect          | Old | New |
| --------------- | --- | --- |
| Name            |     |     |
| Version         |     |     |
| Purpose         |     |     |
| Key differences |     |     |

## 3. Batch Progress

[Track batch completion for multi-session replacements. Update this section as batches complete.]

| Batch | Module/Directory | File Count | Status                                     | Session              |
| ----- | ---------------- | ---------- | ------------------------------------------ | -------------------- |
| 1     |                  |            | ✅ Complete / 🔄 In Progress / ⬜ Not Started | [Session date or ID] |
| 2     |                  |            | ✅ Complete / 🔄 In Progress / ⬜ Not Started |                      |

**Overall progress**: [X of Y batches complete]
**Next batch to execute**: [Batch N — description]

## 4. API Mapping (As Executed)

| Old API | New API | Behavior Changes | Notes |
| ------- | ------- | ---------------- | ----- |
|         |         |                  |       |

## 5. Changes Made

### 5.1 Dependency Changes

**Added**:
- [New dependency with version]

**Removed**:
- [Old dependency]

**Install command used**: `[e.g., npm install new-lib && npm uninstall old-lib]`

### 5.2 Import Changes

**Before**:
```
[Old import pattern]
```

**After**:
```
[New import pattern]
```

### 5.3 Usage Transformations

#### Pattern 1: [Name]

**Description**: [What this transformation does.]

**Before**:
```
[Code snippet from codebase — old usage]
```

**After**:
```
[Code snippet from codebase — new usage]
```

**Behavior change handled**: [Describe any behavioral difference addressed, or "None"]

**Files affected**:
- [file path]
- [file path]

---

#### Pattern 2: [Name]

**Description**: [What this transformation does.]

**Before**:
```
[Code snippet from codebase — old usage]
```

**After**:
```
[Code snippet from codebase — new usage]
```

**Behavior change handled**: [Describe any behavioral difference addressed, or "None"]

**Files affected**:
- [file path]
- [file path]

---

[Repeat for additional patterns]

### 5.4 Build Tooling & Config Changes

| Config File | Change Made |
| ----------- | ----------- |
|             |             |

[If no changes, state "No build tooling changes were needed."]

### 5.5 Dynamic & String Reference Changes

| Location | Change Made |
| -------- | ----------- |
|          |             |

[If no changes, state "No dynamic reference changes were needed."]

### 5.6 Framework Integration Changes

| Integration Point | Change Made |
| ----------------- | ----------- |
|                   |             |

[If no changes, state "No framework integration changes were needed."]

### 5.7 Adapters/Shims Created

[If any adapter or compatibility layer was created, describe it here. Include file path and purpose. Write "None" if not applicable.]

### 5.8 Test Changes

[What test files were modified? Were mocks, fixtures, or test utilities updated? Were assertions updated for behavioral differences?]

## 6. Files Modified

| File | Changes |
| ---- | ------- |
|      |         |

**Total files modified**: [count]

## 7. Final Reference Sweep

| Search                    | Query               | Results                           |
| ------------------------- | ------------------- | --------------------------------- |
| Import sweep              | `[search pattern]`  | [0 remaining / N found and fixed] |
| String reference sweep    | `[search pattern]`  | [0 remaining / N found and fixed] |
| Type reference sweep      | `[search pattern]`  | [0 remaining / N found and fixed] |
| Dependency manifest check | [manifests checked] | [Old dep removed: Yes/No]         |

**Intentional remaining references** (if any):
- [e.g., "Migration notes in CHANGELOG.md — intentionally kept"]

## 8. Verification Results

| Check                  | Result                            | Details                     |
| ---------------------- | --------------------------------- | --------------------------- |
| Compile/lint errors    | Pass / [N errors found and fixed] |                             |
| Build                  | Pass / Fail / Not run             | [Build command used]        |
| Lint                   | Pass / Fail / Not run             | [Lint command used]         |
| Tests (baseline)       | [X passed, Y failed]              | [Before migration]          |
| Tests (post-migration) | [X passed, Y failed, Z skipped]   | [After migration]           |
| New test failures      | [0 / N — list details]            | [Migration-caused failures] |

## 9. Known Issues & Manual Follow-ups

- [Any issues that could not be resolved automatically]
- [Areas that need manual review or testing]
- [Edge cases that may need attention]
- [Items flagged for manual intervention in the replacement plan]

## 10. Rollback Information

**Rollback strategy**: [From the plan — how to revert if needed]
**Rollback still viable**: Yes / No / Partial
**Notes**: [Any constraints on rollback, e.g., "Lock file was regenerated — reverting requires re-running npm install"]

## 11. Recommendations

[Optional — include only if anti-patterns, inconsistencies, or improvement opportunities were found during the replacement.]

- [Recommendation 1]
- [Recommendation 2]
