# Usage Report: [Component/Library Name]

**Date**: [Date]
**Codebase**: [Repository or project name]
**Scope**: [What was analyzed — full codebase, specific modules, etc.]

## 1. Overview

[2-3 sentence summary: what this component/library is, what it does, and its role in the codebase.]

## 2. Setup & Configuration

[How is the library initialized or configured in the project? Include relevant config files, environment variables, or bootstrap code.]

```
[Code snippet: initialization/setup]
```

## 3. Usage Patterns

### Pattern 1: [Name]

**Frequency**: [Most common / Common / Occasional / Rare]
**Description**: [What this pattern does and when it's used.]

```
[Code snippet from codebase]
```

**Files using this pattern**:
- [file path]
- [file path]

---

### Pattern 2: [Name]

**Frequency**: [Most common / Common / Occasional / Rare]
**Description**: [What this pattern does and when it's used.]

```
[Code snippet from codebase]
```

**Files using this pattern**:
- [file path]
- [file path]

---

[Repeat for additional patterns]

## 4. Wrappers & Abstractions

[Are there any custom wrappers, hooks, utilities, or abstraction layers built on top of this library/component? Describe what they add and why they exist.]

| Wrapper | Location | Purpose |
| ------- | -------- | ------- |
|         |          |         |

## 5. Error Handling

[How are errors from this library/component handled in the codebase?]

## 6. Testing

[How is this library/component tested? Any custom mocks, test utilities, or testing patterns?]

## 7. Build Tooling & Configuration References

[List all build configuration files, plugins, loaders, CI/CD pipeline configs, and scripts that reference the library. This includes babel plugins, webpack/vite/rollup loaders, ESLint rules, Dockerfiles, Makefiles, GitHub Actions workflows, etc.]

| Config File | Reference Type | Details |
| ----------- | -------------- | ------- |
|             |                |         |

[If none found, state "No build tooling or configuration references found."]

## 8. Dynamic & String-Based References

[List any non-import references: dynamic property access, plugin configs referencing the library by string name, dependency injection bindings, reflection-based usage, environment variables, and documentation/comments that reference the library.]

| Location | Reference Type | Code/Context |
| -------- | -------------- | ------------ |
|          |                |              |

[If none found, state "No dynamic or string-based references found."]

## 9. Framework-Specific Integrations

[List any framework-specific integration points: middleware registration, DI container bindings, lifecycle hooks, route/handler registrations, decorator usage, module declarations, provider registrations, etc.]

| Integration Point | Location | Details |
| ----------------- | -------- | ------- |
|                   |          |         |

[If none found, state "No framework-specific integrations found."]

## 10. Summary

| Aspect                    | Details                  |
| ------------------------- | ------------------------ |
| Total files using it      |                          |
| Primary usage pattern     |                          |
| Has custom wrappers       | Yes / No                 |
| Consistent usage          | Yes / Mostly / No        |
| Test coverage             | Good / Partial / Minimal |
| Build tooling references  | Yes / No                 |
| Dynamic/string references | Yes / No                 |
| Framework integrations    | Yes / No                 |

## 11. Recommendations

[Optional — include only if anti-patterns, inconsistencies, or improvement opportunities were found.]

- [Recommendation 1]
- [Recommendation 2]
