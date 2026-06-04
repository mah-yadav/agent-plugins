# Node Type System, DI & Module System

TypeScript strictness and idioms, dependency-injection approaches, decorator-aware reading, and ESM/CommonJS interop. Load this when reading a TypeScript codebase, anything using decorators (NestJS, TypeORM, class-validator, type-graphql, tsyringe), or when the module system affects how code is structured.

The ledger already records `Language`, `TS strictness`, `Module system`, and `Decorator usage` from Phase 1 — read those there; this file is for *interpreting* what they mean while reading code.

## Contents

- **TypeScript-specific patterns**
- **Dependency injection**
- **Decorator-aware reading**
- **ESM vs CommonJS**

---

## TypeScript-specific patterns

**Strictness**: Read `tsconfig.json` `compilerOptions`:

| Option | What It Means |
|---|---|
| `strict: true` | Enables all strict checks — the project is type-disciplined |
| `strict: false` (default) or unset | TS is informational only; runtime behavior may diverge from types |
| `noImplicitAny` | Implicit `any` is an error |
| `strictNullChecks` | `null`/`undefined` are explicit, not assumed |
| `noUncheckedIndexedAccess` | Array/object indexing returns `T \| undefined` |
| `verbatimModuleSyntax` | TS keeps import/export syntax verbatim — affects CJS/ESM interop |
| `moduleResolution: "node16"` / `"nodenext"` / `"bundler"` | How TS resolves imports — `bundler` is the modern choice |
| `paths` | Path aliases like `"@/*": ["./src/*"]`. Imports using these (`import x from '@/lib/y'`) won't resolve via file-relative reading — check `paths` before tracing imports |

**Common type patterns**:

| Pattern | What to Look For |
|---|---|
| Branded types | `type UserId = string & { readonly __brand: 'UserId' }` — nominal typing simulation |
| Discriminated unions | `type Result = { ok: true, value: T } \| { ok: false, error: E }` |
| `as const` assertions | Compile-time literal types from runtime objects |
| `satisfies` (TS 4.9+) | Type-safe object literals without widening |
| Conditional types | `T extends U ? X : Y` in generics — usually in library code |
| Mapped types | `{ [K in keyof T]: ... }` — type transformations |
| Template literal types | `` `${'get' \| 'set'}${Capitalize<K>}` `` — for method-name derivation |

---

## Dependency injection

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

## Decorator-aware reading

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

## ESM vs CommonJS

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
| top-level await | ESM only |

**ESM gotchas to flag**:
- ESM can't synchronously `require` from CJS in some bundler configs.
- `__dirname` replacement in ESM: `import { fileURLToPath } from 'node:url'; const __dirname = path.dirname(fileURLToPath(import.meta.url));`
- Conditional exports in `package.json` (`"exports": { "import": "...", "require": "..." }`) determine what gets loaded.
