# Code Quality & Conventions Report: [Project Name]

**Date**: [Date]

## 1. Quality Stack

| Tool | Present? | Fails build? |
|---|---|---|
| ESLint | | |
| Prettier | | |
| Biome | | |
| oxlint | | |
| TypeScript (`tsc --noEmit`) | | |
| dependency-cruiser | | |
| SonarQube / CodeQL | | |
| Husky / lefthook / simple-git-hooks | | |
| lint-staged | | |
| commitlint | | |
| Renovate / Dependabot | | |

## 2. What Will Fail My PR?

| Check | Tool | Fix Command |
|---|---|---|
| | | |

## 3. Code Formatting

### 3.1 Formatter

- Tool: [Prettier / Biome / other]
- Config file: [path]
- **Auto-format command**:
  ```bash
  [npm run format / pnpm format / biome format --write . / ...]
  ```
- Check-only command:
  ```bash
  [...]
  ```
- CI enforced: [yes / no]

### 3.2 EditorConfig

[Contents of `.editorconfig` if present]

### 3.3 IDE settings

- VS Code default formatter: [yes — set to / no]
- Format on save: [yes / no]
- Auto-fix ESLint on save: [yes / no]

### 3.4 Import ordering

- Tool: [`eslint-plugin-import` `import/order` / `simple-import-sort` / Biome `organizeImports`]
- Convention: [groups, order]

## 4. Linting (ESLint or Biome)

### 4.1 Config style

- ESLint: [legacy `.eslintrc.*` / flat `eslint.config.*`]
- Biome: [`biome.json{c}`]

### 4.2 Bases / extends

[List of `extends`, e.g., airbnb-typescript + prettier + @typescript-eslint/recommended]

### 4.3 Plugins active

| Plugin | Purpose |
|---|---|
| | |

### 4.4 Notable rules

| Rule | Severity | Why it matters |
|---|---|---|
| | | |

### 4.5 Type-aware lint?

- `parserOptions.project` set: [yes — to / no — surface lint only]

## 5. Type Discipline (TypeScript)

- `strict`: [on / off]
- Other notable flags: [`noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, ...]
- Escape hatches (counts):
  - `// @ts-ignore`: [N]
  - `// @ts-expect-error`: [N]
  - `as any`: [N]
  - `as unknown as`: [N]

## 6. Architecture Rules

| Rule | Tool | Where defined |
|---|---|---|
| | | |

How to run:
```bash
[depcruise --validate / npm run arch-test / ...]
```

## 7. Dependency Management

- BOM / catalog: [pnpm `catalog:` / none]
- Renovate / Dependabot config: [yes — `renovate.json` / yes — `dependabot.yml` / no]
- Vulnerability scan: [npm audit / Snyk / Socket — and policy]
- Banned deps / overrides: [yes — `overrides` / `resolutions` / none]
- Engine strict: [yes — `.npmrc engine-strict=true` / no]

## 8. Git Hooks & Commit Conventions

### 8.1 Hooks

| Hook | Tool | What it runs |
|---|---|---|
| pre-commit | | |
| commit-msg | | |
| pre-push | | |

### 8.2 Commit messages

- Convention: [Conventional Commits / custom / none]
- Enforced by: [commitlint / not enforced]

### 8.3 Branch naming

[Convention if any]

### 8.4 PR requirements

- Template: [yes — file / no]
- CODEOWNERS: [yes — coverage / no]
- Required reviews: [count]
- Required CI checks: [list]

## 9. Notable Findings

-

## 10. Recommendations (Deep mode)

-
