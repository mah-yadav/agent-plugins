# ask-node-codebase

**Answer focused questions about a Node.js codebase (JavaScript or TypeScript), with Node-specific expertise.**

A Claude Code plugin that turns Claude into a Node codebase consultant. Ask any question about the project — *"how does auth work here?"*, *"where are background jobs defined?"*, *"walk me through what happens when `POST /orders` arrives"* — and get an answer with concrete `file:line` evidence, not a structured document.

## When to use this plugin

Use `ask-node-codebase` when you have a **question** about a Node codebase:

- You're new to a team and want to understand one specific area before pair-programming time is available.
- You're debugging and want to find where a particular flow is implemented.
- You're reviewing a PR and want to know why a piece of code is structured a certain way.
- You're researching how the codebase handles a cross-cutting concern (auth, observability, caching, queues).

**Don't use this plugin** when you need a written onboarding artifact to share with a team or hand off — for that, install the sibling plugin [`onboard-node`](../onboard-node/). That plugin produces files; this one is conversational.

## What this plugin adds over plain Claude Code

Plain Claude can read code. What this plugin adds:

- **Node framework expertise** baked into the workflow: knows where Express/Fastify/NestJS/Hono/Next.js/tRPC routes live, how middleware composes, where decorators register metadata, how request lifecycles differ across frameworks.
- **Decorator-aware reading**: doesn't conclude "this NestJS controller has no routes" when `@Controller` + `@Get` register routes via reflection. Knows TypeORM `@Entity` defines DB schema. Knows class-validator decorators define runtime validation rules.
- **ESM vs CJS awareness**: knows when `__dirname` works and doesn't; knows top-level `await` is ESM-only; knows `require()` of an ESM-only package fails.
- **Monorepo scope safety**: searches use `**/src/**`, not `src/` — catches code in sibling packages that single-package searches miss. Reads `workspaces`, `pnpm-workspace.yaml`, `turbo.json`, `nx.json` to map the topology.
- **TypeScript path-alias awareness**: knows that `import x from '@/lib/y'` in source won't grep against `'./src/lib/y'` — searches for both forms.
- **Edge-runtime awareness**: doesn't recommend `fs.readFile` or `child_process` for code that targets Cloudflare Workers / Vercel Edge.
- **Generated-code excludes**: filters out `node_modules`, `dist`, `.next`, `.turbo`, Prisma generated client, OpenAPI codegen — so searches don't return false positives.
- **Evidence-first answer format**: every claim cited with `file:line`; if it can't be found in the code, it's said plainly — not fabricated.

## Installation

Standard Claude Code plugin install:

```
~/.claude/plugins/ask-node-codebase/
```

After installing, the `ask-node-codebase` skill becomes discoverable.

## Usage

Just ask. Examples:

```
How does authentication work in this codebase?
```

```
Where are background jobs defined? Show me one.
```

```
Walk me through what happens when a POST /orders arrives.
```

```
What does this @UseGuards(JwtAuthGuard) decorator actually do?
```

```
How is the Prisma client instantiated and shared?
```

```
Why is helmet configured with contentSecurityPolicy: false?
```

```
Where would I add a new tRPC procedure?
```

## What an answer looks like

- **Direct answer** in 1–3 sentences.
- **`file:line` citations** for every claim, clickable in IDEs and GitHub.
- **A real code snippet** (3–10 lines) pasted when it makes the answer concrete. Never invented.
- **A Mermaid sequence diagram** when the flow crosses multiple files; rendered in GitHub / VS Code / IntelliJ.
- **One concrete follow-up** offered, not a menu (*"Want me to check how this is tested?"*).
- **Uncertainty stated plainly** (*"I see the JWT middleware but I don't see where the secret is set — likely in `.env.production` which isn't checked in. Want me to confirm by checking the deployment config?"*).

## Follow-up rounds

Most real sessions have 2–6 follow-up questions building on the same investigation. The plugin caches what it's already learned about the project — you can pivot to a new area or drill deeper without restarting.

## What's inside

```
ask-node-codebase/
├── .claude-plugin/plugin.json
└── skills/ask-node-codebase/
    ├── SKILL.md                          ← Q&A workflow + Node traps + routing
    └── references/
        ├── foundation.md                 ← project ID, types, DI, decorators, ESM/CJS
        ├── web-frameworks.md             ← Express/Fastify/NestJS/Koa/Hono/tRPC/Next
        ├── integrations.md               ← GraphQL, gRPC, WebSockets, queues
        ├── control-flow.md               ← request lifecycles, async, error handling
        ├── data-layer.md                 ← ORMs, validation, caching, pooling
        ├── testing.md                    ← runners, coverage, mocking
        ├── design-and-modern.md          ← design patterns + modern Node/TS features
        └── security-and-antipatterns.md  ← security setup + smells
```

Each reference file is scoped to one concern. Based on the question shape, the skill reads only the matching file(s) — a single whole-file `Read` with no over-reading. It never pre-loads everything.

## Scope and limits

**Targets**: Node.js codebases — Express, Fastify, NestJS, Koa, Hono, Next.js (backend portion), tRPC. JavaScript or TypeScript. npm, pnpm, Yarn, or Bun.

**Does not**:
- Generate report files. Answers go in chat. (For artifacts, use `onboard-node`.)
- Modify or write code. This skill is read-only; if you want code changes, ask Claude directly.
- Audit security. It answers *"how is security set up?"*, not *"is the setup good?"*.
- Deep-analyze frontend code (React/Vue/Svelte components, Next.js client-side routing).
- Cover Deno or Bun runtime-specific features beyond basic detection.

**Honest trade-offs**:
- Without a persistent ledger, complex multi-session investigations lose state. If you need that, use `onboard-node` instead.
- Investigation depth scales with the question's specificity. Very broad questions (*"explain the whole codebase"*) will get a high-level answer and a follow-up offering to narrow.
- Node has more framework choices than Java; coverage prioritizes the most common (Express, Fastify, NestJS). For very-niche frameworks, expect more reliance on the codebase itself for context.

## Version

v1.1.0 — split the single `node-analysis-guide.md` into per-concern reference files (`foundation`, `web-frameworks`, `integrations`, `control-flow`, `data-layer`, `testing`, `design-and-modern`, `security-and-antipatterns`) so each question loads only what it needs; removed leftover Quick/Standard/Deep "mode" language and the section-offset loading marker. This plugin's analysis guide is now its own, intentionally diverged from `onboard-node` — keep changes scoped to this plugin.

## Author

Mahesh Yadav
