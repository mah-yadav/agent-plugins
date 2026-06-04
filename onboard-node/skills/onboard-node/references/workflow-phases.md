# Onboarding Workflow Phases (Node)

The phase-by-phase walkthrough for the Node onboarding workflow. Loaded by `SKILL.md` after the orchestrator finishes its identification work.

The output is a practical onboarding report — not a code explanation, but a **developer operations guide**: how to install, build, test, deploy, and work with the project day-to-day.

## Contents

- **Tools**
- **Phase 1: Identify & Orient**
- **Phase 2: Scope & Prioritize**
- **Phase 3: Systematic Codebase Mining** — the canonical area-priority-by-mode table lives here
- **Phase 4: Compile "Still Need to Ask" List**
- **Phase 5: Finalize Onboarding Report**
- **Exploration & conversational guidelines**

## Core Principle

**Mine the codebase first, ask people later.** Extract the 60–70% of onboarding knowledge that lives in the code, config files, and project structure.

## Tools

- **`Read`**, **`Bash`** (`ls`, `find`, `grep`), **`Glob`**, **`Grep`**: For codebase exploration.
- **`AskUserQuestion`**: Ask the user about experience level and focus areas. (Constraints in `SKILL.md` § Interactive Questions.)
- **`TodoWrite`**: Track which phase you're in.
- **`Write`**: Produce the onboarding report and ledger using the [onboarding report template](../assets/onboarding-report-template.md).
- **`Agent`** (with `subagent_type: "Explore"`): Offload heavy read-only searches. See `SKILL.md` § Subagent Usage.
- **`Edit`**: Update the shared findings ledger on disk.
- **`Read`** of `references/area-*.md`: Load the detailed steps for each Phase 3 area.

Set up the todo list at the start with the five phases below.

---

## Phase 1: Identify & Orient (automated — 1 round)

Establish the project landscape quickly.

1. **Find the project root**: Look for `package.json` at the workspace root or one level down.

2. **Read the manifest** (`package.json`):
   - `name`, `version`, `description`.
   - `engines.node` → required Node version.
   - `type`: `module` (ESM) or unset/`commonjs` (CJS).
   - `scripts` — these are the actual interface to the project; read all of them.
   - `dependencies` — runtime.
   - `devDependencies` — build/test/lint.
   - `workspaces` (npm/Yarn) or check for `pnpm-workspace.yaml` (pnpm) — monorepo signal.
   - `packageManager` field — pins the package manager.

3. **Detect the package manager**:

   | Lockfile present | Package manager |
   |---|---|
   | `package-lock.json` | npm |
   | `yarn.lock` + `.yarnrc.yml` | Yarn Berry (v2+) |
   | `yarn.lock` + no `.yarnrc.yml` | Yarn Classic (v1) |
   | `pnpm-lock.yaml` | pnpm |
   | `bun.lockb` or `bun.lock` | Bun |
   | Multiple lockfiles | **Flag** — team is split; ask the user which is canonical |

4. **Read the README** if present — stated purpose, setup instructions, architecture notes.

5. **Detect TypeScript**:
   - `tsconfig.json` at root or per-package.
   - If multiple `tsconfig.*.json`, read all and note which is for build vs. test vs. type-check.
   - Note `strict`, `noImplicitAny`, `strictNullChecks` — type discipline indicator.
   - Note `paths` for path aliases — these affect every subsequent import trace.
   - Note `module`, `moduleResolution`, `target` — affects ESM/CJS behavior.

6. **Detect framework(s)** from dependencies:

   | Dep | Framework |
   |---|---|
   | `express` | Express |
   | `fastify` | Fastify |
   | `@nestjs/core` | NestJS |
   | `koa` | Koa |
   | `@hapi/hapi` | Hapi |
   | `hono` | Hono |
   | `next` | Next.js (note: backend portion only — `pages/api/` or `app/api/`) |
   | `@trpc/server` | tRPC |
   | `@apollo/server`, `graphql-yoga`, `mercurius`, `@nestjs/graphql` | GraphQL server |

7. **Detect runtime target**:

   | Indicator | Target |
   |---|---|
   | `wrangler.toml` | Cloudflare Workers (edge) |
   | `vercel.json` with `runtime: "edge"` | Vercel Edge |
   | `deno.json` | Deno |
   | None of the above | Standard Node.js |

   **Edge runtime alert**: If targeting edge, flag in the ledger. Node APIs (`fs`, `child_process`, parts of `crypto`) may not work — the report's recommendations must respect this.

8. **Detect decorator usage**:
   - Grep for `^@[A-Z]\w+` in source. If matches in NestJS / TypeORM / class-validator / type-graphql patterns, mark `Decorator usage = Yes` in the ledger.
   - If yes, all subsequent reading must use the decorator-aware rules in `node-type-system.md` § Decorator-aware reading.

9. **Detect path aliases** in `tsconfig.json` `compilerOptions.paths`. Record them in the ledger — they determine how to resolve imports later.

10. **Map top-level structure**: `Bash ls` on the project root. Note key directories (`src/`, `apps/`, `packages/`, `services/`, `libs/`).

11. **Detect non-standard layout / mixed languages**:

    | Deviation | Detection | Adaptation |
    |---|---|---|
    | Mixed JS + TS | Both `.js` and `.ts` files in source | Note ratio. Both are in scope. Migration in progress is common. |
    | Polyglot (Node + Python/Go/Rust) | Other language sources at root | Scope analysis to Node directories only. Note other languages. |
    | Frontend-only (no server entry) | No `server.{ts,js}`, no Express/Fastify/Nest, only React/Vue/Svelte | Confirm with user — this plugin targets server-side. |
    | Mixed CJS / ESM in same package | `.cjs` and `.mjs` files | Map the boundary; flag interop risks. |
    | Generated code | Prisma `node_modules/.prisma`, OpenAPI-generated, swagger output | Identify generators; record paths in ledger; exclude from architecture analysis. |

12. **Estimate project size**: Use the source-file count command from `SKILL.md`.

13. **Monorepo discovery recipe**: Use this concrete procedure before anything else.

    **npm workspaces**:
    - Read `package.json` `workspaces` field — array of glob patterns: `["packages/*", "apps/*"]`.
    - For each match, read its `package.json`.

    **pnpm workspaces**:
    - Read `pnpm-workspace.yaml` — `packages:` list with globs.
    - Check for `catalog:` blocks — shared dependency versions.

    **Yarn workspaces (Berry)**:
    - Read root `package.json` `workspaces`. Same as npm.
    - Read `.yarnrc.yml` for plugin config.

    **Turborepo** (`turbo.json` present):
    - Read `turbo.json` — `pipeline:` block defines task dependencies and inputs/outputs.
    - The workspace topology is implicit from package.json workspaces; turbo just orchestrates tasks.

    **Nx** (`nx.json` present):
    - Read `nx.json` — `tasksRunnerOptions`, `targetDefaults`.
    - Read `project.json` in each package — defines project type, build/test targets, tags.

    **Lerna** (`lerna.json` present, increasingly rare):
    - Read `lerna.json` — `packages:` list and version-management strategy.

    **Record in the ledger** (under `Workspace / Monorepo Layout`):
    ```
    | Package | Path | Source path | Role |
    |---|---|---|---|
    | api      | apps/api          | apps/api/src       | 1 (App/Server) |
    | web      | apps/web          | apps/web/src       | (Frontend — out of scope; note only) |
    | db       | packages/db       | packages/db/src    | 2 (Domain/data) |
    | ui       | packages/ui       | packages/ui/src    | (Frontend lib — out of scope) |
    | common   | packages/common   | packages/common/src| 4 (Shared utils) |
    ```

    Every subagent prompt and `grep` from now on uses this path list (or `**/src/**` for monorepos).

14. **Workspace prioritization** (if monorepo): Determine which packages to analyze and in what order.

    | Priority | Package Type | How to Identify |
    |---|---|---|
    | 1st | Application / Server | Has entry point (`index.ts`, `server.ts`, `main.ts`), depends on a framework (Express/Fastify/Nest) |
    | 2nd | API or Routes package | Exports route handlers, controllers |
    | 3rd | Domain / Service / Data | Business logic, ORM models, repositories |
    | 4th | Shared library | Utility packages depended on by multiple others |
    | 5th | Infrastructure | DB clients, queue adapters, external integrations |
    | Skip | Frontend / UI | Out of scope for this plugin (note only) |
    | Skip | Test-only / fixtures | Test utilities |
    | Skip | Generated | Anything in `dist/`, generated client output |

    **Depth by mode**: Quick → deep-dive 1–2 packages, summarize rest. Standard → deep-dive 3–5, summarize rest. Deep → all.

**Choose the onboarding output directory** — see `SKILL.md` § Ledger Location.

**Create the findings ledger**: Use `Write` to create `<output-dir>/.onboarding-findings-ledger.md` with the structure defined in `SKILL.md`. Populate Project Identity (all fields including `Decorator usage`, `Edge runtime target`, `Module system`, `Language`, `TS strictness`), Workspace/Monorepo Layout, Generated-code & Build Output Paths, Path Aliases, Files Read, and Frameworks Detected.

**Create the report skeleton**: Use the [onboarding report template](../assets/onboarding-report-template.md). Fill the *Project Identity* and *Tech Stack* sections. Use the placeholder `[TODO — pending Phase 2 mode selection]` for every other section.

**Resume-instructions block**: At the very top of the report skeleton, insert:

```markdown
<!-- onboarding-meta
Ledger:  <output-dir>/.onboarding-findings-ledger.md
Report:  <output-dir>/onboarding-report.md
Mode:    [Quick | Standard | Deep]
Started: <ISO-8601>

To resume in a new chat:
  /onboard-node continue
or paste:
  "Continue the onboarding. The partial report is at <output-dir>/onboarding-report.md."
-->
```

**Present findings to the user**: Summarize — name, purpose, tech stack, Node version, package manager, framework, monorepo (yes/no), estimated size, and recommended execution mode.

---

## Phase 2: Scope & Prioritize (interactive — 1 round)

Use `AskUserQuestion`. Suggested questions:

- **Execution mode**: Based on my analysis, I recommend [Quick/Standard/Deep] mode. Does that work? (Quick — single report, ~1 session / Standard — main + key reports, 1–2 sessions / Deep — all reports, 2–4 sessions)
- **Experience level**: How familiar are you with Node.js / TypeScript / [detected framework]? (Beginner / Familiar / Experienced)
- **Priority areas**: Which area matters most for your first tasks? (Build & Run / Architecture & APIs / Security / CI/CD)
- **Role focus**: What will you primarily work on? (Backend features / Bug fixes / DevOps & deployment / Full stack)

**After receiving answers**:
1. Confirm the execution mode.
2. Map user priorities to area priority tiers.
3. Update the ledger with the execution mode.
4. Replace the report's `[TODO — pending Phase 2 mode selection]` placeholders with either content scaffolding (in-scope) or `[Not analyzed — out of scope for <mode>]`.

**Depth calibration**:

| User Level | P0 Area | P1 Area | P2 Area |
|---|---|---|---|
| **Beginner** | Full analysis + code examples + explanations | Key findings + brief examples | Summary table only |
| **Familiar** | Key findings + patterns + gotchas | Summary + notable patterns | One-liner per item |
| **Experienced** | Patterns, conventions, gotchas only | Summary table | Skip or mention |

---

## Phase 3: Systematic Codebase Mining (automated — multiple rounds)

Load each area's reference file via `Read`, execute its steps, update the ledger, then move on.

**Area priority by execution mode** — this is the canonical source of truth.

| Area | Quick (P0 only) | Standard (P0+P1) | Deep (all) |
|---|---|---|---|
| 3.1 Build & Local Dev | Full analysis | Full + standalone report | Full + standalone report |
| 3.2 Architecture/APIs/Data/Testing | Summary tables (P0 rows of `area-explain-code.md`) | Full + standalone report | Full + standalone report |
| 3.3 Security | Auth mechanism + local dev auth only | Full + standalone report | Full + standalone report |
| 3.4 Observability | Log location + health endpoint only | Summary | Full + standalone report |
| 3.5 CI/CD & Deployment | Pipeline stages + "what fails my PR" only | Full + standalone report | Full + standalone report |
| 3.6 Code Quality | Format command + PR fail checklist only | Full + standalone report | Full + standalone report |

A "Summary only" Quick row produces a section that's exactly the per-area "Minimum viable output" snippet. Any other P1/P2 content for that section is replaced with `[Not analyzed — out of scope for Quick mode]`.

**Checkpoint cadence**: Share findings with the user after every 2 areas.

---

### Area 3.1: Build & Local Development — P0

**Goal**: Can the developer install, build, and run the project from scratch?

**Load**: `Read` [references/area-build-local-dev.md](./area-build-local-dev.md). Quick-mode output scope is defined in that file.

---

### Area 3.2: Architecture, APIs, Data & Testing — P0

**Goal**: How is the code structured, what APIs does it expose/consume, how does it manage data, and how does the team test?

**Load**: `Read` [references/area-explain-code.md](./area-explain-code.md). It routes to the `node-*.md` reference files (`node-frameworks` always; `node-type-system` / `node-data-layer` / `node-testing` / `node-patterns` per topic — eagerly in Standard/Deep).

**Boundary with other areas**: `area-explain-code` identifies but does NOT deeply analyze auth, observability, CI/CD, or quality tools — those belong to 3.3, 3.4, 3.5, 3.6.

---

### Area 3.3: Security — P1 (P0 in Quick: auth mechanism + local dev auth only)

**Goal**: How is the application secured?

**Load**: `Read` [references/area-security.md](./area-security.md).

**Prior findings check**: If `area-explain-code` already identified the auth middleware/library, start from that finding.

---

### Area 3.4: Observability — P2 (P0 in Quick: log location + health endpoint only)

**Goal**: How is the application monitored and debugged?

**Load**: `Read` [references/area-observability.md](./area-observability.md). Quick-mode output scope is defined in that file.

---

### Area 3.5: CI/CD & Deployment — P1 (P0 in Quick: pipeline stages + PR gates only)

**Goal**: How does code get from a PR to production?

**Load**: `Read` [references/area-cicd-deployment.md](./area-cicd-deployment.md). Quick-mode output scope is defined in that file.

---

### Area 3.6: Code Quality & Conventions — P1 (P0 in Quick: PR fail checklist + format command only)

**Goal**: What standards does the team enforce?

**Load**: `Read` [references/area-code-quality.md](./area-code-quality.md). Quick-mode output scope is defined in that file.

---

## Phase 4: Compile "Still Need to Ask" List

Compile a list of important topics that **cannot** be answered from the code alone:

- **Team process**: Standup cadence, sprint rituals, PR review norms, on-call rotation.
- **Monitoring access**: Dashboards, log aggregators, alerting.
- **Domain context**: Business rules, architectural decisions.
- **Environment access**: VPN, cluster access, secrets, credentials.
- **Tribal knowledge**: What burned the team before, what isn't documented.

---

## Phase 5: Finalize Onboarding Report

Finalize the report. Fill any remaining sections. Review for completeness.

**Output adapted to mode**:

| Mode | Main Report | Standalone Reports |
|---|---|---|
| Quick | Full (P0 filled, others marked) | None |
| Standard | Full | `build-local-dev-guide.md` + `code-explanation-report.md` |
| Deep | Full | All 6 standalone reports |

**Cross-referencing**: Main report should NOT duplicate full standalone content. Include key highlights and a link.

All files go in `<output-dir>` chosen in Phase 1.

After creating documents, ask the user to review and provide feedback.

---

## Exploration Guidelines

- **Use the Explore subagent** for broad searches.
- **Read config files in full** — `package.json`, `tsconfig.json`, `eslint.config.*`, framework configs are dense.
- **Follow the evidence chain**: Prisma in deps → find `schema.prisma`. `@KafkaListener` analog → find consumer registration. tsconfig `paths` → resolve aliased imports.
- **Don't guess from file names** — always read content. `auth.ts` could be middleware, a service, a controller, or a config.
- **Skip gracefully**: Note absences — "No CI pipeline found" is a valuable finding.
- **Check the ledger before every file read.**

## Conversational Guidelines

- **Incremental sharing**: Every 2 areas.
- **Flag surprises**: Unusual patterns, deprecated libraries, security-sensitive defaults.
- **Note absences**: Missing CI, no migration tool, no API validation library — findings worth reporting.
- **Stay practical**: Everything should answer "what do I need to do?" not just "what does this project have?"
