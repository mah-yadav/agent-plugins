# Area: Code Quality & Conventions Analysis (Node)

Reference for Phase 3.6. Produces a structured report covering linting, formatting, type checking, architecture rules, dependency governance, and git workflow conventions.

**Core principle**: Know what will reject your code before you write it. PR failures from unknown conventions are the #1 onboarding friction.

## Contents

- **Priority tiers** · **Ledger read-before** · **Symbol tracing**
- **Step 1** Identify the quality stack · **Step 2** Linting rules · **Step 3** Formatting · **Step 4** Type discipline
- **Step 5** Architecture tests · **Step 6** Dependency management · **Step 7** Git hooks & commit conventions · **Step 8** Generate report
- **Ledger update after** · **Exploration guidelines**

## Priority Tiers (within this area)

| Step | Priority | Rationale |
|---|---|---|
| Step 1: Identify quality stack | **P0** | Must know what's enforced |
| Step 3.1: Formatter + auto-format command | **P0** | Most common new-dev PR failure |
| Step 2 (summary): Which lint/typecheck tools fail the build | **P0** | Must know what blocks PRs |
| Step 5: Architecture rules (dependency-cruiser, ESLint boundaries) | **P1** | Layer/module rules |
| Step 2 (detail): ESLint rule details | **P1** | Understanding violations |
| Step 3.2–3.3: EditorConfig, import ordering | **P1** | First-day setup |
| Step 4: Type discipline (TS strict, etc.) | **P1** | TS-specific quality dimension |
| Step 6: Dependency management | **P1** | How to add deps correctly |
| Step 7: Git hooks & commit conventions | **P1** | Workflow conventions |
| Step 2.4: SonarQube / CodeQL detail | **P2** | Deep detail |
| Step 6.3: License compliance | **P2** | Rarely blocks dev |

**Quick mode**: Step 1, Step 2 (summary), Step 3.1 (format command). Skip the rest.

## Ledger Read-Before

Check the ledger. Do NOT re-read `package.json` or `tsconfig.json`. Pull `devDependencies` from prior analysis to identify quality tools. If `area-explain-code` noted ESLint/Prettier presence, start there. Only read quality-specific config files not yet analyzed.

## Symbol Tracing

Useful greps: `// eslint-disable`, `// @ts-ignore`, `// @ts-expect-error`, `as any`, `as unknown as`. High density of these is a quality signal.

## Step 1: Identify the Code Quality Stack — P0

### 1.1 Linters

| Tool | Detection |
|---|---|
| **ESLint** | `eslint` in devDeps; `.eslintrc.{js,cjs,json,yml}`, `eslint.config.{js,mjs,ts}` (flat config) |
| **Biome** | `@biomejs/biome` in devDeps; `biome.json` or `biome.jsonc` |
| **oxlint** | `oxlint` in devDeps; uses ESLint configs |
| **dprint** | `dprint` in devDeps; `.dprint.json` (rare in JS projects) |
| **TypeScript itself** | `tsc --noEmit` is effectively a static analyzer |

**ESLint config format matters**:
- **Legacy config**: `.eslintrc.*` files; `extends`/`plugins`/`rules` keys.
- **Flat config** (ESLint 9+, opt-in earlier): `eslint.config.js` exporting an array of objects.

If both are present, ESLint uses flat by default in v9+; flag the redundancy.

**Common ESLint shareable configs to look for**:
- `eslint-config-airbnb`, `eslint-config-airbnb-base`, `eslint-config-airbnb-typescript`
- `eslint-config-standard`, `eslint-config-standard-with-typescript`
- `eslint-config-prettier` (disables formatting rules; must be LAST in extends)
- `@typescript-eslint/recommended`, `@typescript-eslint/strict`
- `eslint-config-next` (Next.js)
- `@nestjs/eslint-config`
- `eslint-plugin-import` (import order, no-cycle)
- `eslint-plugin-react`, `eslint-plugin-react-hooks` (frontend, but common in Node-FE monorepos)
- `eslint-plugin-node` / `eslint-plugin-n` (Node-specific rules)
- `eslint-plugin-security`
- `eslint-plugin-promise`
- `eslint-plugin-unicorn`
- `eslint-plugin-jest` / `eslint-plugin-vitest` / `eslint-plugin-testing-library`

### 1.2 Formatters

| Tool | Detection |
|---|---|
| **Prettier** | `prettier` in devDeps; `.prettierrc{,.json,.yml,.js,.cjs,.toml}`, `prettier.config.*`, `"prettier"` key in package.json |
| **Biome** | `@biomejs/biome` — also formats |
| **dprint** | `.dprint.json` |
| **Built-in** (none) | Format-via-ESLint (deprecated; usually paired with `eslint-config-prettier`) |

### 1.3 Type Checking

For TS projects, this is its own quality gate:
- `tsc --noEmit` as a CI step.
- `tsconfig.json` strictness flags (see Step 4).

### 1.4 Architecture rules

| Tool | What it does |
|---|---|
| `dependency-cruiser` | Disallow cycles, enforce layer boundaries, forbid imports across packages |
| `eslint-plugin-import` `no-cycle`, `no-restricted-imports` | Lighter version of the same |
| `eslint-plugin-boundaries` | Layer enforcement via ESLint |
| `eslint-plugin-import-zones` | Path-based import zones |
| `madge` | Cycle detection only |

### 1.5 Quality config files

| File | Tool |
|---|---|
| `.eslintrc.*` / `eslint.config.*` | ESLint |
| `.prettierrc*` | Prettier |
| `biome.json{,c}` | Biome |
| `.editorconfig` | Editor settings |
| `.eslintignore` / `.prettierignore` | Tool-specific ignores |
| `.dependency-cruiser.{cjs,js,json}` | dependency-cruiser |
| `sonar-project.properties` | SonarQube |
| `.github/CODEOWNERS` | Code ownership rules |
| `.husky/`, `.lefthook.yml`, `lint-staged.config.*` | Git hooks |

### 1.6 Which tools fail the build vs. warn

Check the CI pipeline (ledger from 3.5) and `package.json` scripts:
- Is `eslint` exit code checked in CI? Most teams yes; some are advisory-only.
- Is `prettier --check` part of CI?
- Is `tsc --noEmit` a required step?

**Present findings**: Quality stack — what tools exist, which fail the build, which auto-fix.

## Step 2: Analyze Linting Rules — P0 summary / P1 detail

### 2.1 ESLint configuration — P1

Read the active ESLint config (the one ESLint will actually use; flat config takes precedence over legacy in v9+).

| What to extract | Why |
|---|---|
| `extends` chain | Shows the base ruleset |
| Project-specific `rules` | The actual enforced behavior |
| Per-file overrides | Different rules for test files, configs, scripts |
| `ignorePatterns` / `.eslintignore` | What's excluded |
| Plugin list | Which extra rulesets are active |
| `parserOptions.project` | Type-aware linting? (slow but powerful) |

**Rules most likely to fail new code**:
- Import ordering (`import/order`)
- No console.log (`no-console`)
- Naming conventions (`@typescript-eslint/naming-convention`)
- Explicit return types (`@typescript-eslint/explicit-function-return-type`)
- No `any` (`@typescript-eslint/no-explicit-any`, `@typescript-eslint/no-unsafe-*`)
- No floating promises (`@typescript-eslint/no-floating-promises`)

### 2.2 Biome configuration — P1

(If used instead of ESLint+Prettier.) Read `biome.json{c}`:
- `linter.rules` — categories enabled/disabled.
- `formatter` — settings.
- `organizeImports` — auto-import sort.
- `files.include` / `files.ignore`.

### 2.3 Type-aware lint vs. surface lint — P1

Type-aware rules (the `*-type-checked` variants in `@typescript-eslint`) require `parserOptions.project`. These are slow but catch much more — flag whether they're enabled.

### 2.4 SonarQube / CodeQL — P2

- `sonar-project.properties` for SonarQube.
- GitHub Code Scanning / CodeQL workflows in `.github/workflows/codeql.yml`.

## Step 3: Analyze Code Formatting — P0/P1

### 3.1 Formatter & commands — P0

| Source | What |
|---|---|
| `npm run format` / `yarn format` / `pnpm format` | Convention; usually writes |
| `npm run format:check` / `format:write` | Explicit split |
| `prettier --write .` | Direct invocation |
| `biome format --write .` | Biome equivalent |
| `biome check --write .` | Biome combined lint+format |

**Extract**: The exact auto-format command and the check command. Whether CI enforces format check.

### 3.2 EditorConfig — P1

`.editorconfig` contents: indent style/size, line endings, trailing whitespace, charset.

### 3.3 IDE settings — P1

- `.vscode/settings.json` if committed: format-on-save, default formatter, ESLint auto-fix.
- `.vscode/extensions.json` recommended extensions.
- Format-on-save expectation — set or not?

### 3.4 Import ordering — P1

`eslint-plugin-import` `import/order` rule config, `simple-import-sort` plugin, or Biome's `organizeImports`. Standard groupings:
1. Node built-ins (`node:*`)
2. External packages
3. Internal packages (workspace deps)
4. Aliased imports (`@/...`)
5. Relative imports

## Step 4: Type Discipline (TS projects) — P1

Read `tsconfig.json` `compilerOptions`:

| Option | Discipline Level |
|---|---|
| `strict: true` | Enables all strict checks — well-disciplined |
| `noImplicitAny: true` | No silent `any` |
| `strictNullChecks: true` | Null/undefined explicit |
| `noUncheckedIndexedAccess: true` | Array access returns `T | undefined` |
| `noImplicitReturns: true` | All code paths return |
| `noFallthroughCasesInSwitch: true` | Switch cases must break/return |
| `exactOptionalPropertyTypes: true` | `?:` doesn't include `undefined` in the value |
| `noUnusedLocals` / `noUnusedParameters` | Unused = error |

**Count escape hatches**: grep for `// @ts-ignore`, `// @ts-expect-error`, `as any`, `as unknown as`. Density tells you whether the type system is respected or just present.

**Mixed JS/TS projects**: Note the ratio. If JS files lack `// @ts-check`, they're effectively type-untracked.

## Step 5: Analyze Architecture Tests — P1

| Tool | Config / Test |
|---|---|
| `dependency-cruiser` | `.dependency-cruiser.cjs`; runs via `depcruise` command, often in CI |
| `eslint-plugin-import` `no-cycle` | ESLint rule |
| `eslint-plugin-boundaries` | ESLint config for layer rules |
| `madge` | Cycle detection command |
| Custom tests | Sometimes a test file using `ts-morph` or `typescript` compiler API to enforce arbitrary rules |

**Extract**: Table of architecture rules, layer dependency diagram, how to run the check.

If a monorepo and there's no architectural enforcement, **flag it** — large workspaces drift quickly.

## Step 6: Analyze Dependency Management — P1

### 6.1 Version Management — P1

- Workspace catalogs (pnpm `catalog:`) for shared version pinning.
- Renovate / Dependabot config: `renovate.json` or `.github/dependabot.yml`. Group rules, schedule.
- `syncpack` for keeping monorepo deps aligned.
- Resolutions / overrides: `package.json` `resolutions` (Yarn) or `overrides` (npm/pnpm).
- Peer dependencies: `peerDependencies` and `peerDependenciesMeta` (esp. in libraries).

### 6.2 Dependency Enforcement — P1

- `engines.node` enforcement: `engine-strict=true` in `.npmrc` blocks install on wrong Node version.
- `npm audit` / `pnpm audit` policy: failing on high/critical?
- Snyk / Socket integration.
- Banned dependencies: custom check, npm-audit-resolver allow-list.

**Extract**: How to add a new dependency correctly, banned libraries, vulnerability scanning tool.

### 6.3 License Compliance — P2

`license-checker`, `@nodesecure/licenser`, manual `LICENSE_REPORT.md`. Allowed license list.

## Step 7: Analyze Git Hooks & Commit Conventions — P1

### 7.1 Git Hooks — P1

| Tool | Config |
|---|---|
| **Husky** | `.husky/` directory with hook scripts; `prepare` script in package.json |
| **lefthook** | `lefthook.yml` |
| **simple-git-hooks** | `simple-git-hooks` key in package.json |
| **`.git/hooks/*`** (manual) | Rare; non-portable |

What hooks run:
- `pre-commit`: usually `lint-staged` for staged-file linting/formatting.
- `commit-msg`: commit message validation (commitlint).
- `pre-push`: tests, type check.

**lint-staged config** (`lint-staged.config.*` or `lint-staged` in package.json) — what commands run on which staged files.

### 7.2 Commit Message Conventions — P1

- `commitlint` config: `commitlint.config.*` or `.commitlintrc.*`. Often extends `@commitlint/config-conventional` (Conventional Commits).
- Ticket reference requirements (custom rules).
- Changelog generation from commits (`@changesets/cli`, `conventional-changelog`, `semantic-release`).

### 7.3 Branch Naming & PR Standards — P1

- Branch naming: convention by team; sometimes enforced via GH branch protection regex.
- PR template: `.github/pull_request_template.md`.
- CODEOWNERS: `.github/CODEOWNERS`.
- Required reviews count.
- Auto-merge configuration.

## Step 8: Generate Report

Write using the [code quality template](../assets/code-quality-template.md).

**Quick mode**: Fill the *Quality Stack*, *Code Formatting → Formatter*, and *What Will Fail My PR?* sections. Mark others `[Not analyzed]`.

Default location: `<output-dir>/code-quality-report.md` (Standard/Deep only).

## Ledger Update After

Mark area 3.6 complete. Add:
- What will fail a PR (summary table)
- Auto-format command
- IDE setup instructions
- Git conventions

## Exploration Guidelines

- **Read ESLint and Prettier configs in full** — small and dense; rule sets define what passes.
- **Distinguish enforcement from advice**: `"error"` blocks the build; `"warn"` doesn't.
- **Check CI for additional quality gates**: a tool may run only in CI even if not in `package.json` scripts.
- **Note what's missing**: No formatter? No type check? No architecture enforcement? No `lint-staged`? Each is a finding.
- **Prominently feature the auto-format command** — most useful single command for a new dev.
- **Count escape hatches**: high density of `// @ts-ignore` / `as any` is a maintenance signal.
