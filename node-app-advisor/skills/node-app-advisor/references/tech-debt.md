# Technical Debt Analysis

Inventory accumulated debt — the gap between the current state of the code and where it should be. Identify dead code, deprecated usage, inconsistent patterns, duplication, and shortcuts. Every finding needs evidence, severity, and a concrete fix.

**Core principle:** Technical debt is the sum of all shortcuts taken yesterday that slow you down today. Unlike bugs, it doesn't break the app — it breaks the team's velocity. Inventory it so it can be prioritized and paid down.

## Contents

- Step 1 — Code marker inventory
- Step 2 — Dead code detection
- Step 3 — Deprecated & risky API usage
- Step 4 — Inconsistent patterns
- Step 5 — Code duplication
- Step 6 — Missing error handling
- Step 7 — Type-safety debt (TypeScript)
- Step 8 — Compile findings

## Tools

- `Grep` — TODO/FIXME/HACK markers, `@deprecated`, eslint-disable, dead-code indicators.
- `Glob` — orphaned files, old config, unused resources.
- `Read` — read suspect modules; context matters for judging intent.
- `Bash` — `npx depcheck` for unused declared dependencies (ask first — it runs a tool); `npx ts-prune`/`npx knip` for unused exports (ask first).
- `Agent` (Explore) — fan-out: "find all TODO comments", "find all `eslint-disable`", "find all `@ts-ignore`/`@ts-expect-error`". Re-`Read` leads before citing.
- **Dead code:** `Grep` the export/function name across the tree and count call sites; 0 references → candidate (no `listCodeUsages` tool). Mind tsconfig `paths` aliases and barrel re-exports.

## Step 1 — Code marker inventory

### 1.1 TODO/FIXME/HACK/XXX comments

| Marker | Meaning | Severity |
|--------|---------|----------|
| `TODO` | Acknowledged incomplete implementation | Medium |
| `FIXME` | Known bug or broken functionality | High |
| `HACK` / `WORKAROUND` | Intentional shortcut, likely fragile | High |
| `XXX` | Dangerous or questionable code | High |
| `TEMP` / `TEMPORARY` | Code intended to be removed | Medium |

*Detect:* grep each marker across `src/`; read the surrounding context (ticket reference? how old via `git blame`?); categorize as still-relevant, already-fixed-comment-remains, ticketed, or no-plan.

### 1.2 Suppressed lint / type warnings

| Pattern | Meaning | Severity |
|---------|---------|----------|
| `// eslint-disable-next-line` (specific rule) | Targeted suppression — usually OK, check rationale | Low |
| `/* eslint-disable */` (file-wide) | Whole file opts out of linting | Medium |
| `// @ts-ignore` | Suppresses the next line's type error silently (hides real errors) | Medium |
| `// @ts-expect-error` (no comment) | Better than ignore, but unexplained ones are debt | Low |
| `// @ts-nocheck` | Whole file opts out of type checking | High |
| `eslint-disable-next-line no-explicit-any` | Hiding `any` usage | Medium |

## Step 2 — Dead code detection

| Type | Detection | Severity |
|------|-----------|----------|
| Unused exports | Exported functions/consts/types with 0 importers | Medium |
| Unused local functions | Module-private functions with 0 references | Medium |
| Unreachable code | Code after `return`/`throw`, or inside always-false conditions | Medium |
| Unused imports | Imports for unreferenced symbols (lint usually catches — flag if not enforced) | Low |
| Unused dependencies | `package.json` deps never imported | Medium |
| Commented-out code | Large commented-out blocks (> 5 lines) | Medium |
| Dead feature flags | Flags permanently on/off with the other branch never taken | Medium |
| Orphaned files | Modules not reachable from any entry point | Low |
| Orphaned tests | Tests for source modules that no longer exist | Low |
| Unused env vars | `.env.example` keys never read via `process.env` | Low |

*Detect:* `npx depcheck` (unused deps, ask first); `npx ts-prune`/`npx knip` (unused exports, ask first); `Grep` suspected-unused export names for importers; grep large commented-out blocks. Note: dynamic `require`/`import()`, DI-registered providers, and string-keyed lookups can create false positives — verify before flagging.

## Step 3 — Deprecated & risky API usage

| Pattern | Detection | Severity |
|---------|-----------|----------|
| Deprecated Node core APIs | `new Buffer()` (use `Buffer.from`/`Buffer.alloc`), `url.parse` (use `URL`), `domain`, `crypto.createCipher` | High |
| Deprecated/unmaintained packages | `request`, `node-sass`, `moment` (for new code → `date-fns`/`dayjs`/`Temporal`), `node-fetch` v2 where native `fetch` exists | Medium |
| Callback-style core APIs for new code | `fs` callbacks instead of `fs/promises`; manual promisify where promises exist | Low |
| Project's own deprecated code | Internal `@deprecated` JSDoc symbols still used | Medium |
| Deprecated framework APIs | Framework methods marked deprecated in the installed major version | Medium |
| CommonJS in an ESM codebase (or vice versa) | `require()` in `"type": "module"`, or `__dirname` in ESM without shim | Medium |

*Detect:* grep `new Buffer(`, `url.parse(`, `createCipher(`; grep imports of known-deprecated packages; grep `@deprecated` then find usages; check `"type"` field vs. `require`/`import` usage.

## Step 4 — Inconsistent patterns

| Inconsistency | Detection | Severity |
|----------------|-----------|----------|
| Mixed naming conventions | `*Service` vs `*Manager` vs `*Handler` for the same role; camelCase vs snake_case files | Medium |
| Inconsistent error handling | Some `throw`, some return `null`/`undefined`, some `{ error }`, some `Result`-style | High |
| Mixed async styles | Callbacks vs `.then()` chains vs `async/await` across the codebase | Medium |
| Mixed module systems | `require`/`module.exports` mixed with `import`/`export` in one package | Medium |
| Inconsistent DTO/response patterns | Typed DTOs vs raw entities vs `any` across endpoints | High |
| Mixed validation | Zod here, Joi there, manual checks elsewhere | Medium |
| Inconsistent config access | Central config object vs scattered `process.env` reads | Medium |
| Date/time handling | Mix of `Date`, `moment`, `date-fns`, `dayjs`, raw timestamps | Medium |
| Mixed HTTP clients | `axios`, `got`, `node-fetch`, native `fetch` all in use | Low |
| Inconsistent logging | `console.log` mixed with a structured logger | Low |

*Detect:* grep export-name suffixes within a layer; grep `throw `/`return null`/`return {` to compare error styles; grep `require(` vs `import `; compare handler return shapes; grep the various HTTP-client/date imports.

## Step 5 — Code duplication

| Type | Detection | Severity |
|------|-----------|----------|
| Duplicated business logic | Same validation/calculation across multiple service functions | High |
| Duplicated query patterns | Near-identical queries across repositories | Medium |
| Duplicated error/try-catch wrapping | The same try/catch boilerplate repeated everywhere | Medium |
| Copy-paste route/middleware setup | Near-identical handler/middleware blocks | Low |
| Duplicated mapping code | Manual entity↔DTO mapping repeated | Medium |
| Duplicated test setup | Same `beforeEach`/fixtures copied across test files | Low |
| Duplicated config/constants | Same magic values redefined in multiple files | Low |

*Detect:* look for similar function names across modules; grep similar query strings; grep repeated try/catch patterns; compare setup blocks across files. (`npx jscpd src` reports copy-paste if the user approves running it.)

## Step 6 — Missing error handling

| Issue | Detection | Severity |
|-------|-----------|----------|
| Empty catch blocks | `catch (e) {}` with no logging or rethrow | Critical |
| Catch-and-log-only | `catch (e) { logger.error(e) }` without recovery/rethrow where the caller needs to know | High |
| Swallowed promise rejections | Floating promises / missing `.catch` / no `await` (also Performance) | High |
| Over-broad catch | `catch (e)` treating all errors the same when they need different handling | Medium |
| Missing async-handler wrapping (Express) | `async` route handlers without try/catch or an async-error wrapper → unhandled rejection | High |
| No global error handler | No central error middleware/filter for consistent responses (Express error mw / NestJS exception filter) | High |
| Re-throwing without context | `throw new Error(String(e))` losing the cause/stack (use `{ cause }`) | Medium |
| No `unhandledRejection`/`uncaughtException` handler | Process has no last-resort handler | High |
| Swallowing in event emitters | `emitter.on('error')` missing — uncaught `error` event crashes the process | High |

*Detect:* grep `catch ([\w]*) {}` / empty catches; grep `catch` blocks that only `console.log`/`logger`; grep async route handlers; check for global error middleware / NestJS exception filters; grep `process.on('unhandledRejection'`; check `.on('error'` coverage on streams/emitters.

## Step 7 — Type-safety debt (TypeScript)

Skip this step for a plain-JavaScript project (note the absence of types as its own Medium finding if the codebase is large).

| Issue | Detection | Severity |
|-------|-----------|----------|
| Loose tsconfig | `strict: false`, or `noImplicitAny`/`strictNullChecks` off | High |
| Pervasive `any` | Frequent `: any`/`as any`, especially at boundaries | Medium |
| Unsafe casts | `as unknown as T`, casting to bypass type errors | Medium |
| `@ts-ignore` over real errors | Suppressions hiding genuine type mismatches | Medium |
| Untyped external data | `req.body`/API responses/`JSON.parse` used without runtime validation + typing | High |
| `// @ts-nocheck` files | Whole files excluded from checking | High |
| Missing return types on public API | Exported functions relying on inference at module boundaries | Low |

*Detect:* read `tsconfig.json` `compilerOptions` (`strict`, `noImplicitAny`, `strictNullChecks`); grep `: any`/`as any`/`as unknown as`/`@ts-ignore`/`@ts-nocheck`; check that external input is validated before being typed.

## Step 8 — Compile findings

For each finding: **Category** (Code Markers / Dead Code / Deprecated API / Inconsistency / Duplication / Error Handling / Type Safety) · **Severity** · **Description** · **Evidence** (`file:line` + snippet) · **Impact** (maintenance cost, bug risk, onboarding friction) · **Recommended fix** (before/after) · **Effort** (XS/S/M/L/XL). Present the compiled findings.
