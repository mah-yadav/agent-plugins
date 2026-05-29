# Area: Build & Local Development (Node)

Reference for Phase 3.1. Produces a practical local development guide for a Node.js project.

**Goal**: From a fresh clone to a running application with passing tests, without asking anyone for help.

## Priority Tiers (within this area)

| Step | Priority | Rationale |
|---|---|---|
| 1 Install & Build Commands | **P0** | Nothing happens without these |
| 2 Package Manager & Lockfile Hygiene | **P0** | Wrong PM = broken install |
| 3 Node Version & Engines | **P0** | App won't start on wrong version |
| 4 Environment Configuration | **P0** | App won't start without config |
| 5 Local Services & Dependencies | **P0** | App needs its dependencies running |
| 6 Docker & Container Setup | **P0** | Most modern projects rely on it |
| 7 IDE & Developer Tooling | **P1** | Important but not day-1 blocker |
| 8 Scripts & Automation | **P1** | Helpful, not essential |

**Quick mode**: Steps 1–6, plus report generation. Skip 7, 8.

## Ledger Read-Before

Read the ledger first. If `package.json`, `tsconfig.json`, lockfile, or any env-template file is already in `Files Read`, use those findings. Pull `Workspace / Monorepo Layout`, `Path Aliases`, and `Generated-code & Build Output Paths` to scope every search.

## Step 1: Install & Build Commands — P0

The `package.json` `scripts` block is the **canonical interface** to the project. Read every entry.

Common scripts to look for:

| Script name | What it usually does |
|---|---|
| `install` (postinstall) | Build native modules, prepare hooks |
| `build` | Compile TS, bundle, or copy assets |
| `dev` / `start:dev` | Watch-mode development server |
| `start` / `start:prod` | Production startup (usually after `build`) |
| `test` | Run unit tests |
| `test:e2e` / `test:integration` | Separate test suites |
| `lint` | ESLint check |
| `format` | Prettier write |
| `typecheck` | `tsc --noEmit` |
| `db:migrate` / `prisma:migrate` | Database migrations |
| `db:seed` | Seed data |
| `prepare` | Husky hook setup |

**Build chain detection**:

| Indicator | Bundler / Compiler |
|---|---|
| `tsc` in scripts, `noEmit: false` in tsconfig | TypeScript compiler emits JS |
| `esbuild` in devDeps | esbuild (fast bundler) |
| `swc`, `@swc/core` | SWC (Rust-based, very fast) |
| `vite`, `vite.config.*` | Vite (also a bundler in some contexts) |
| `webpack`, `webpack.config.*` | Webpack (legacy backend, common frontend) |
| `rollup`, `rollup.config.*` | Rollup (libraries) |
| `tsup` | tsup (esbuild wrapper for libraries) |
| `ncc`, `@vercel/ncc` | ncc (single-file bundle for serverless) |
| `turbopack`, Next.js with `--turbo` | Turbopack (Next.js dev) |
| `bun build` | Bun bundler |
| `nest build` | NestJS CLI (wraps tsc by default, or webpack) |

**Extract**: Exact commands for install, build, dev, prod, test, single test. Note whether build runs typecheck or skips it (build-time speed vs. type safety).

## Step 2: Package Manager & Lockfile Hygiene — P0

| Lockfile | Package Manager | Install Command |
|---|---|---|
| `package-lock.json` | npm | `npm install` or `npm ci` |
| `yarn.lock` (no `.yarnrc.yml`) | Yarn Classic (v1) | `yarn install` |
| `yarn.lock` + `.yarnrc.yml` | Yarn Berry (v2+) | `yarn install` |
| `pnpm-lock.yaml` | pnpm | `pnpm install` |
| `bun.lockb` / `bun.lock` | Bun | `bun install` |

**Check `package.json` `packageManager` field** — if set (`"packageManager": "pnpm@9.0.0"`), Corepack enforces this. Document it.

**Anti-patterns to flag immediately**:
- **Multiple lockfiles present** — team disagreement on PM; CI may be using one and devs another. Ask which is canonical.
- **`.npmrc` with `legacy-peer-deps=true`** — papering over dependency conflicts. Note it.
- **`yarn.lock` + `package-lock.json`** in the same repo — install will differ between machines.
- **No lockfile** — builds are non-reproducible. Major risk.

**Workspace install** (monorepo):
- npm: `npm install` at root installs everything; `npm install -w <pkg>` adds a dep to one package.
- pnpm: `pnpm install` at root; `pnpm add <dep> --filter <pkg>` adds to one.
- Yarn Berry: `yarn install` at root.
- Turborepo: `turbo install` is not a thing; use the package manager directly.

## Step 3: Node Version & Engines — P0

| Source | Where |
|---|---|
| `.nvmrc` | Single-line file with Node version (e.g., `20.11.0`) |
| `.node-version` | Same idea, fnm/asdf-compatible |
| `package.json` `engines.node` | `"node": ">=20.0.0 <21"` semver range |
| `volta` field in package.json | Volta-pinned Node + PM versions |
| `.tool-versions` (asdf) | Lists Node and other tools |

**Document the canonical version pin** and the recommended version manager (nvm, fnm, volta, asdf, mise, n).

If a project lists Node 16 but uses features requiring Node 18+ (e.g., `fetch` global, `node:test`), that's a real bug — flag it.

## Step 4: Environment Configuration — P0

| What to Find | Where to Look |
|---|---|
| `.env.example` / `.env.template` / `.env.sample` | Convention varies; one of these is the spec |
| `.env`, `.env.local`, `.env.development`, `.env.test`, `.env.production` | Profile-specific env files |
| `dotenv` / `dotenv-flow` / `@dotenvx/dotenvx` in deps | dotenv-style loader |
| `process.env.<VAR>` calls in source | The actual variables in use (cross-reference with `.env.example`) |
| `zod`, `envalid`, `znv`, `t3-env` for env-var validation | Runtime schema for env vars — read it; it IS the spec |
| `config/` directory or `node-config` library | Hierarchical config layers |
| Framework-specific config:<br>NestJS `ConfigModule.forRoot(...)`<br>Next.js `next.config.{js,ts}`<br>Fastify env plugin (`@fastify/env`) | Framework-managed env loading |

**Extract**:
- The complete list of env vars the app reads (`process.env.X` greps + zod/envalid schemas).
- Which are required vs. have defaults.
- Which is the "local dev" profile and how it's selected (NODE_ENV, .env loading order).
- Any secrets that need to be obtained from outside the repo.

**`NODE_ENV` semantics**:
- `development` enables verbose logging, dev-only middleware, hot reload.
- `production` strips debug, enables prod optimizations.
- `test` used by Jest/Vitest; some libraries (e.g., bcrypt) speed up in test mode.
- Setting `NODE_ENV=production` for *all* environments is a real anti-pattern — note if so.

## Step 5: Local Services & Dependencies — P0

| What to Find | Where to Look |
|---|---|
| Database connection | `DATABASE_URL` env var, Prisma datasource, TypeORM config, mongoose connect |
| Redis | `REDIS_URL`, `ioredis` / `redis` deps, BullMQ usage |
| Message broker | `kafkajs`, `amqplib` deps + connection URL in env |
| Cache services | In addition to Redis: Memcached (`memjs`), CDN |
| Mock external services | `msw`, WireMock in Docker Compose, recorded fixtures |
| Email / SMS sandbox | Mailpit / MailHog in Compose, Twilio test creds |
| S3 / object storage | LocalStack in Compose, `aws-sdk` env config |
| Search | Elasticsearch / OpenSearch / Meilisearch / Typesense |
| Seed scripts | `db:seed` script in package.json, `prisma/seed.ts`, `seeds/` directory |

**Extract**: Every required service, how to start it locally (Docker Compose, brew install, manual), connection details, seed data availability.

## Step 6: Docker & Container Setup — P0

| What to Find | Where to Look |
|---|---|
| Docker Compose | `docker-compose.yml`, `docker-compose.override.yml`, `compose.yml`, `compose.yaml` |
| Application Dockerfile | `Dockerfile`, `docker/Dockerfile`, package-local `Dockerfile`s in monorepo |
| Multi-stage builds | `FROM ... AS build` patterns — what's compiled vs. runtime image |
| Compose services | DBs, brokers, caches, mocks defined as services |
| Port mappings | Compose port configs |
| Volume mounts | Are source files mounted for hot-reload? |
| Health checks | `healthcheck:` in Compose, `HEALTHCHECK` in Dockerfile |
| Container entry | `CMD`, `ENTRYPOINT` — `node`, `npm start`, custom entrypoint script |

**Node-specific Dockerfile tells**:
- `FROM node:20-alpine` — small image but missing some native build deps (flag if `prebuild` is needed).
- `FROM node:20-slim` — Debian-based, larger but compatible.
- `RUN npm ci --omit=dev` vs. `npm install` — production vs. dev image.
- Native module rebuild step (`node-gyp`, `prebuild-install`) — common for `bcrypt`, `sharp`, `better-sqlite3`, `node-canvas`.

**Extract**: Start/stop commands, port mappings per service, data persistence behavior, multi-stage shape, hot-reload availability inside Docker.

## Step 7: IDE & Developer Tooling — P1

| What to Find | Where to Look |
|---|---|
| EditorConfig | `.editorconfig` |
| VS Code settings | `.vscode/settings.json`, `.vscode/extensions.json` (recommended extensions) |
| IDE-specific debug configs | `.vscode/launch.json`, JetBrains `.idea/runConfigurations/` |
| TypeScript watch | `tsc --watch` in dev script, or framework dev server doing it |
| Hot reload | `nodemon`, `ts-node-dev`, `tsx --watch`, Node's native `--watch`, Vite HMR for Vite-based apps, NestJS `--watch` |
| Debugger config | `--inspect`, `--inspect-brk` flags in dev script |
| Type checking on save | Editor TS server, separate watcher |

**Extract**: Required IDE extensions, format-on-save setup, hot reload availability, debug attach instructions.

## Step 8: Scripts & Automation — P1

| What to Find | Where to Look |
|---|---|
| Setup scripts | `scripts/setup.sh`, `Makefile`, custom CLI in `bin/` |
| DB scripts | `scripts/db/`, `prisma/seed.ts`, migration commands |
| Git hooks | `.husky/`, `simple-git-hooks` config, `lefthook.yml` |
| Pre-commit | `lint-staged.config.*`, `lint-staged` field in package.json |
| Code generators | `plop` / `hygen` / `nx generate` / custom scaffolders |
| Repo tasks | Turborepo `turbo.json`, Nx `project.json` targets |

**Extract**: Available scripts / tasks, first-time setup script, git hooks behavior (and how to bypass when needed).

## Step 9: Verify (OFF by default) — P2

**Default: SKIP.** Onboarding is read-only. Only run verify when the user has explicitly told the orchestrator to verify, and re-confirm each command before running. Prior approval of "verify in general" is not approval of any specific command.

If you skip verify, write the documented commands into the guide with a note: *"Commands documented but not executed. Run them yourself to confirm."*

When verifying (in order):
1. Install: `npm ci` / `pnpm install --frozen-lockfile` / `yarn install --frozen-lockfile`.
2. Build: `npm run build` (or PM equivalent).
3. Type-check: `npm run typecheck` if defined.
4. Start Docker Compose if needed.
5. Run unit tests: `npm test`.
6. Start the app: `npm run dev`, verify a health endpoint or "listening on port X" log.

**Stop conditions** (abort and document what happened):
- Install takes > 5 minutes or requires credentials.
- A native module fails to compile.
- A command needs credentials/secrets you don't have.
- A command would push to a remote registry or shared resource.

## Step 10: Generate Local Dev Guide

Write the document using the [build & local dev template](../assets/build-local-dev-template.md).

**Quick mode**: Fill Prerequisites, Quick Start, Environment Variables (required only), Local Services, Common Tasks. Mark others `[Not analyzed — out of scope for Quick mode]`.

**Standard/Deep mode**: Fill all sections.

Default location: `<output-dir>/build-local-dev-guide.md` (Standard/Deep only).

## Ledger Update After

Mark area 3.1 complete. Add:
- Install / build / dev / test commands
- Package manager + lockfile + Node version
- All env vars discovered (required vs. defaulted)
- Local services and ports
- Docker Compose / Dockerfile findings
- Path aliases (if not already in the ledger)
- Files read during this step

## Exploration Guidelines

- **Read `package.json` and `tsconfig.json` in full** — small and dense.
- **Read all `.env*` files** — but never echo their contents to the user; reference variable *names* only.
- **Follow the evidence chain**: `prisma` in deps → check `schema.prisma`. `bullmq` → check Redis env. Native module → check Dockerfile build stage.
- **Note absences**: No `.env.example`? No README setup section? Flag these gaps prominently.
- **Distinguish "what's documented" from "what's actually used"**: the README may be stale. Cross-check against `package.json` scripts and source.
