# Node Foundation: Project Identification, Types, DI, Decorators, ESM/CJS

Orientation layer — load this first for almost any question. Covers what kind of
project this is (runtime, package manager, layout), how its type system and DI
work, and the two Node traps that silently break source-reading: decorator
metadata and ESM-vs-CJS.

**Contents**
- **1. Project Identification** — manifest, lockfiles, runtime target, TS presence, layout
- **2. Type System & DI** — tsconfig strictness, type patterns, DI approaches
- **3. Decorator-aware reading** — when source is incomplete
- **4. ESM vs CommonJS** — module-system signals and gotchas

---

## 1. Project Identification

Determine the project type, runtime, package manager, and structure before reading any source.

### Manifest & lockfile

| Indicator | What It Tells You |
|---|---|
| `package.json` at root | Node project. Read `name`, `version`, `engines`, `type` (module / commonjs), `scripts`, `dependencies`, `devDependencies` |
| `package.json` with `"workspaces"` | npm or Yarn workspace monorepo — read the array for package locations |
| `pnpm-workspace.yaml` | pnpm workspace monorepo — read for package locations and catalogs |
| `package-lock.json` | npm — pinned dependency tree |
| `yarn.lock` | Yarn (Classic v1 or Berry v2+) — check `.yarnrc.yml` to distinguish |
| `pnpm-lock.yaml` | pnpm — the most strict resolver; `node_modules` will be symlink-heavy |
| `bun.lockb` or `bun.lock` | Bun runtime / package manager |
| `.nvmrc` / `.node-version` / `engines.node` | Pinned Node version — install this before running anything |
| Multiple lockfiles present | **Red flag** — different team members likely using different package managers. Flag immediately. |

### Runtime target

| Indicator | Runtime / Target |
|---|---|
| `engines.node` in package.json | Standard Node.js — match version |
| `"type": "module"` in package.json | ESM-default project (otherwise CommonJS) |
| `.cjs` or `.mjs` files | Mixed CJS/ESM — call out the boundary |
| `next.config.{js,ts,mjs}`, `pages/` or `app/` directory | Next.js (full-stack React). API routes live in `app/api/` or `pages/api/` |
| `nuxt.config.{ts,js}` | Nuxt (Vue full-stack) |
| `astro.config.{js,ts,mjs}` | Astro (mostly static + islands) |
| `wrangler.toml` | Cloudflare Workers — edge runtime, **not** full Node API |
| `vercel.json` with `runtime: "edge"` | Vercel Edge — V8 isolate, **not** Node API |
| `deno.json` / `deno.jsonc` | Deno runtime (overlap with Node but different stdlib) |
| `bun.toml` / Bun-specific scripts | Bun runtime |

**Edge runtime alert**: If any of `wrangler.toml`, `@cloudflare/workers-types`, or `runtime: "edge"` appears, the code may NOT have access to Node-only APIs (`fs`, `child_process`, parts of `crypto`, native modules). Read the actual runtime before assuming Node compatibility.

### TypeScript presence

| Indicator | What It Means |
|---|---|
| `tsconfig.json` | TypeScript project — read for `compilerOptions`, especially `module`, `moduleResolution`, `strict`, `paths`, `target` |
| `tsconfig.*.json` (e.g., `tsconfig.build.json`) | Multiple TS configs — one likely extends another. Read all. |
| `.ts` / `.tsx` files alongside `.js` | Mixed TS/JS — common during migration. Note ratio. |
| `// @ts-check` at top of `.js` files | JavaScript with type-checking enabled via JSDoc |
| `jsconfig.json` | JS-only project but VS Code uses it for IntelliSense (no compile-time checks) |
| `noEmit: true` in tsconfig | TypeScript used only for type-checking; another tool (esbuild/swc/babel) does the actual transpilation |

**Path aliases**: `compilerOptions.paths` in tsconfig defines aliases like `"@/*": ["./src/*"]`. Imports using these aliases (`import x from '@/lib/y'`) won't resolve via standard file-relative reading. Always check `paths` before tracing imports.

### Source layout

```
src/                          # Most common application root
├── index.{ts,js}             # Entry point
├── server.{ts,js}            # Or here, for explicit server bootstrap
├── routes/                   # Express / Fastify / Hono route handlers
├── controllers/              # Some teams; usually paired with routes
├── services/                 # Business logic (in NestJS this is @Injectable)
├── repositories/             # Data access; if missing, repository pattern not used
├── models/ or schemas/       # Validation schemas (Zod, Yup) or ORM models
├── middleware/               # Auth, logging, rate-limit middleware
├── lib/ or utils/            # Helpers
├── config/                   # Configuration loading
└── types/                    # Shared TS types

dist/ or build/ or out/       # Compiled output — skip during analysis
node_modules/                 # Dependencies — skip during analysis
.next/ or .nuxt/              # Framework build caches — skip
```

For monorepos, this layout repeats under `packages/*/src/` or `apps/*/src/` or `services/*/src/`.

---

## 2. Type System & DI

### TypeScript-specific patterns

**Strictness**: Read `tsconfig.json` `compilerOptions`:

| Option | What It Means |
|---|---|
| `strict: true` | Enables all strict checks — the project is type-disciplined |
| `strict: false` (default) or unset | TS is informational only; runtime behavior may diverge from types |
| `noImplicitAny` | Implicit `any` is an error |
| `strictNullChecks` | `null`/`undefined` are explicit, not assumed |
| `noUncheckedIndexedAccess` | Array/object indexing returns `T | undefined` |
| `verbatimModuleSyntax` | TS keeps import/export syntax verbatim — affects CJS/ESM interop |
| `moduleResolution: "node16"` / `"nodenext"` / `"bundler"` | How TS resolves imports — `bundler` is the modern choice |
| `paths` | Path aliases — see § 1 |

**Common type patterns**:

| Pattern | What to Look For |
|---|---|
| Branded types | `type UserId = string & { readonly __brand: 'UserId' }` — nominal typing simulation |
| Discriminated unions | `type Result = { ok: true, value: T } | { ok: false, error: E }` |
| `as const` assertions | Compile-time literal types from runtime objects |
| `satisfies` (TS 4.9+) | Type-safe object literals without widening |
| Conditional types | `T extends U ? X : Y` in generics — usually in library code |
| Mapped types | `{ [K in keyof T]: ... }` — type transformations |
| Template literal types | `` `${'get' | 'set'}${Capitalize<K>}` `` — for method-name derivation |

### Dependency injection (where applicable)

Most Node frameworks don't have DI. NestJS is the exception. Other patterns:

| Approach | Detection | Notes |
|---|---|---|
| **NestJS** | `@nestjs/common` | Class-based, decorator-driven DI |
| **InversifyJS** | `inversify` | Decorator-driven DI without NestJS |
| **tsyringe** | `tsyringe` (Microsoft) | Lightweight DI for TS |
| **awilix** | `awilix` | Function-based DI, popular outside NestJS |
| **Manual** | None of the above | Most Node apps just import and instantiate — no DI container |

For non-DI codebases, **trace dependencies via imports**: who imports whom defines the graph. There's no central registry.

---

## 3. Decorator-aware reading

(Node parallel to Lombok-aware reading in Java.)

If `experimentalDecorators: true` in `tsconfig.json` and the codebase uses decorators (NestJS, TypeORM, class-validator, type-graphql, tsyringe, routing-controllers), **the visible source is incomplete** — much behavior is generated at runtime via reflection.

| Decorator pattern | Generates / Implies |
|---|---|
| `@Controller('/api')` + `@Get('/x')` (NestJS) | Route registration; no explicit `app.get(...)` call exists |
| `@Injectable()` + constructor params (NestJS) | Constructor-injected DI; the constructor signature IS the dependency list |
| `@Entity()` + `@Column()` (TypeORM, MikroORM) | DB table mapping; column definitions, types, defaults |
| `@IsEmail()`, `@IsString()` (class-validator) | Validation rules attached to a class; applied by `ValidationPipe` |
| `@ObjectType()` + `@Field()` (type-graphql) | GraphQL schema definition |
| `@injectable()` + `@inject()` (InversifyJS, tsyringe) | DI binding |

**Reading rule**: When you see decorators, the file is not the full story. Look at the framework's bootstrap code (`main.ts`, `app.module.ts`) to understand how decorator metadata is consumed.

---

## 4. ESM vs CommonJS

Critical to know which the project uses — they behave differently at runtime.

| Signal | Interpretation |
|---|---|
| `"type": "module"` in package.json | All `.js` files are ESM |
| `"type": "commonjs"` or unset | All `.js` files are CJS |
| `.mjs` extension | Always ESM regardless of `type` |
| `.cjs` extension | Always CJS regardless of `type` |
| `import` statements with extensions in paths (`./foo.js`) | ESM (Node ESM requires explicit extensions) |
| `require()` calls | CJS |
| `__dirname`, `__filename` used directly | CJS (these don't exist in ESM by default) |
| `import.meta.url` used | ESM |
| `top-level await` | ESM only |

**ESM gotchas to flag**:
- ESM can't synchronously `require` from CJS in some bundler configs.
- `__dirname` replacement in ESM: `import { fileURLToPath } from 'node:url'; const __dirname = path.dirname(fileURLToPath(import.meta.url));`
- Conditional exports in `package.json` (`"exports": { "import": "...", "require": "..." }`) determine what gets loaded.
