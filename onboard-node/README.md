# onboard-node

**Produce a structured onboarding report for a Node.js codebase (JavaScript or TypeScript).**

A Claude Code plugin that systematically mines a Node project — Express, Fastify, NestJS, Koa, Hono, Next.js API routes, tRPC — and writes an onboarding artifact a new developer or a receiving team can use as their map of the project.

## When to use this plugin

Use `onboard-node` when you need a **written deliverable**:

- A new developer needs an onboarding document before pair-programming time is available.
- A team is offshoring or handing off a codebase and wants a single artifact to share.
- An auditor, consultant, or new tech lead needs a structured assessment.
- The repo will be revisited later and you want a reusable map of its build, architecture, security, observability, CI/CD, and code-quality stack.

**Don't use this plugin** when you just want to *ask a question* about a codebase (*"how does auth work here?"*) — for that, install the sibling plugin [`ask-node-codebase`](../ask-node-codebase/). That plugin is conversational; this one produces files.

## What you get

After running, the plugin saves to your chosen output directory (`docs/` or `.claude/onboarding/` by default):

| File | Mode | Contents |
|---|---|---|
| `onboarding-report.md` | All | Main report — project identity, tech stack, build & run, architecture, security, observability, CI/CD, code quality, "still need to ask" list |
| `.onboarding-findings-ledger.md` | All | Internal state. Tracks what's been analyzed. Resumable across sessions. |
| `build-local-dev-guide.md` | Standard / Deep | Full local-dev guide |
| `code-explanation-report.md` | Standard / Deep | Architecture, APIs, data, testing detail |
| `security-analysis-report.md` | Deep | Full security analysis |
| `observability-analysis-report.md` | Deep | Full observability analysis |
| `cicd-deployment-report.md` | Deep | Full CI/CD analysis |
| `code-quality-report.md` | Deep | Full code quality analysis |

## Modes

The plugin auto-detects project size and recommends a mode in Phase 2; you can override.

| Mode | Project size | Sessions | Output |
|---|---|---|---|
| Quick | < 100 source files, single package | 1 | P0-only report; out-of-scope sections explicitly marked `[Not analyzed]` |
| Standard | 100–500 source files, 2–8 packages | 1–2 | Main + build-local-dev + code-explanation reports |
| Deep | 500+ source files, 9+ packages | 2–4 | All 8 reports |

## Workflow at a glance

1. **Phase 1 — Orient** — find `package.json`, detect package manager (npm/pnpm/yarn/bun), Node version, framework, TypeScript usage, decorator usage (NestJS/TypeORM/class-validator), edge-runtime target (Cloudflare Workers, Vercel Edge), monorepo structure (workspaces, Turborepo, Nx); choose an output dir; create the ledger and report skeleton.
2. **Phase 2 — Scope** — confirm execution mode and priority areas with the user.
3. **Phase 3 — Mine** — analyze six areas (build/local-dev, architecture, security, observability, CI/CD, code quality), one at a time, with user checkpoints every two areas.
4. **Phase 4 — Questions** — compile the "still need to ask the team" list.
5. **Phase 5 — Report** — finalize the artifact(s).

## Installation

Standard Claude Code plugin install — drop the plugin directory under `~/.claude/plugins/` or wherever your Claude Code instance loads plugins from:

```
~/.claude/plugins/onboard-node/
```

After installing, the `onboard-node` skill becomes discoverable.

## Usage

Invoke from the Claude Code chat:

```
Onboard me to this Node codebase.
```

```
Create an onboarding guide for the Express app at apps/api.
```

```
I need an onboarding artifact for this NestJS project I can share with a new hire.
```

```
Generate an onboarding document for this TypeScript monorepo.
```

## Resuming a partial run

If a session ends mid-onboarding, the partial report and ledger remain on disk. In a new chat, say:

```
Continue the onboarding. The partial report is at <output-dir>/onboarding-report.md.
```

The plugin runs a staleness check — `git log` since the ledger's last-updated timestamp — and resumes from the first incomplete area, re-reading any files that changed in the meantime.

## What's inside

```
onboard-node/
├── .claude-plugin/plugin.json
└── skills/onboard-node/
    ├── SKILL.md                       ← orchestrator: tiers, ledger contract, subagent rules
    ├── references/
    │   ├── workflow-phases.md         ← Phase 1–5 walkthrough
    │   ├── area-build-local-dev.md    ← Phase 3.1 detail
    │   ├── area-explain-code.md       ← Phase 3.2 detail
    │   ├── area-security.md           ← Phase 3.3 detail
    │   ├── area-observability.md      ← Phase 3.4 detail
    │   ├── area-cicd-deployment.md    ← Phase 3.5 detail
    │   ├── area-code-quality.md       ← Phase 3.6 detail
    │   └── node-analysis-guide.md     ← Node framework + decorator + ESM/CJS guide
    └── assets/
        ├── onboarding-report-template.md
        └── (six standalone report templates)
```

The `references/` files are internal building blocks loaded on demand — they are not user-invokable as separate skills.

## Scope and limits

**Targets**: Node.js server codebases using npm, pnpm, Yarn, or Bun. JavaScript or TypeScript. Specialized for Express, Fastify, NestJS, Koa, Hono, Next.js (backend portion), tRPC. Handles decorators (NestJS, TypeORM, class-validator, type-graphql), monorepos (npm/pnpm/yarn workspaces, Turborepo, Nx), ESM/CJS, edge runtimes (Cloudflare Workers, Vercel Edge), and contract-first generators (OpenAPI codegen, Prisma).

**Does not**:
- Run the build by default. Verify is opt-in and per-command-confirmed; onboarding is a read-only mining exercise.
- Deep-analyze frontend code in monorepos — React/Vue/Svelte are flagged but not pursued.
- Cover Deno or Bun runtime-specific features beyond basic detection.
- Verify correctness of any documented command. Commands are extracted from code, not executed.

**Honest trade-offs**:
- The 5-phase workflow is heavyweight. For a single question about a codebase, the sibling `ask-node-codebase` plugin is the better tool.
- Quick mode produces a partial report with explicit `[Not analyzed]` markers for out-of-scope sections. By design, not a bug — but it means the artifact is shorter than it looks.
- Node has more framework choices than Java (no Spring-equivalent dominant choice). Coverage prioritizes the most common — Express, Fastify, NestJS — and may need user input for less-common frameworks.

## Version

v1.0.0 — initial release. Parallel to the `onboard-java` plugin.

## Author

Mahesh Yadav
