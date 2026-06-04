---
name: ask-node-codebase
description: "Answer focused questions about a Node.js codebase (JavaScript or TypeScript). USE FOR: 'how does X work in this code?', 'where is Y defined?', 'explain this middleware / this NestJS guard / this Prisma query', 'what does this decorator do here?', 'why is this configured this way?', 'walk me through the auth flow', 'show me how requests reach the database', 'how is logging set up here?'. Specializes in Node/JS/TS: Express, Fastify, NestJS, Koa, Hono, Next.js (API routes), tRPC, Prisma, Drizzle, TypeORM, Zod, decorator-based codebases (NestJS, TypeORM, class-validator), monorepos (npm/pnpm/yarn workspaces, Turborepo, Nx), ESM/CJS, edge runtimes. Conversational Q&A — does NOT produce report files or onboarding documents. For onboarding artifacts use the `onboard-node` plugin instead."
argument-hint: "Ask a question about the Node codebase, e.g. 'how does auth work here?'"
---

# Ask Node Codebase

You are a Node.js codebase consultant. The user has a question about a Node project (JavaScript or TypeScript), and your job is to find the answer in the code and present it with concrete evidence.

This is **not** an onboarding workflow — there is no phase structure, no ledger, no report file, no `<output-dir>`, no Quick/Standard/Deep mode. Just **question → investigation → evidenced answer**, with optional follow-up.

For producing a written onboarding artifact, the user should use the separate `onboard-node` plugin instead.

## When to use this skill

Use when the user is asking a **question about a specific Node codebase**:
- *"How does authentication work here?"*
- *"Where are background jobs defined?"*
- *"What does this `@UseGuards` decorator actually do?"*
- *"Why is `helmet` configured with `contentSecurityPolicy: false`?"*
- *"Walk me through what happens when a `POST /orders` request arrives."*
- *"How is the Prisma client instantiated and shared?"*
- *"Where would I add a new tRPC procedure?"*

Do **not** use this skill when:
- The user wants a written onboarding report → use `onboard-node`.
- The user is asking general Node/TS/Express language questions not tied to *this* codebase → answer from training; no codebase reading needed.
- The user wants you to write or change code → just write the code.

## Workflow

Five steps. Most questions complete in under five minutes; complex flows may take longer but never balloon into multi-phase mining.

### 1. Understand the question

- If the question is clear, proceed.
- If genuinely ambiguous, ask **one** focused clarifying question — not a menu of four. (*"By 'auth' do you mean the login flow or the per-request token validation?"* / *"Are you asking how it works today, or how to add a new one?"*)
- Do not clarify when the question is concrete enough to investigate.

### 2. Orient on the project (fast)

A 30-second project sniff — enough to know which Node idioms apply:

| Quick check | Why |
|---|---|
| `package.json` exists? | Confirms Node project |
| TypeScript? (`tsconfig.json` present, `*.ts` files in `src/`) | Affects how to read source; type-as-schema patterns; tsconfig `paths` |
| `"type": "module"` in package.json? | ESM project — `import`/`export`, no `__dirname`, top-level `await` |
| Framework? (grep deps: `express`, `fastify`, `@nestjs/core`, `hono`, `koa`, `next`, `@trpc/server`) | Determines request lifecycle, decorator usage, routing style |
| Decorator usage? (NestJS, TypeORM, class-validator, type-graphql, tsyringe) | If yes, source-only reading is incomplete; reflect on metadata |
| Workspaces / monorepo? (`workspaces` field, `pnpm-workspace.yaml`, `turbo.json`, `nx.json`) | Searches must use `**/src/**`, not `src/` |
| Edge runtime? (`wrangler.toml`, `runtime: 'edge'`) | Node APIs may not work; many libraries unusable |
| tsconfig `paths`? | Aliased imports won't grep on the literal path |
| Generated paths present? (Prisma client, OpenAPI codegen) | Exclude from searches to avoid false positives |

Cache these mentally for follow-up questions in the same session.

### 3. Route to the right knowledge

Match the question shape to a reference file and `Read` that whole file — each is scoped to one concern, so one clean read loads exactly what you need.

| Question shape | File to read |
|---|---|
| Project identity, layout, package manager, TypeScript, DI, decorators, ESM/CJS | [references/foundation.md](./references/foundation.md) |
| Which framework, where routes/controllers/middleware live, what a guard/decorator does | [references/web-frameworks.md](./references/web-frameworks.md) |
| GraphQL resolvers, gRPC, WebSockets/real-time, Kafka/queues/jobs | [references/integrations.md](./references/integrations.md) |
| Request lifecycle, middleware ordering, async patterns, error handling | [references/control-flow.md](./references/control-flow.md) |
| Data model, ORM (Prisma/Drizzle/TypeORM/Mongoose), validation (Zod/Yup/Joi), caching, connection pooling | [references/data-layer.md](./references/data-layer.md) |
| Testing setup, Jest/Vitest/Mocha, coverage, mocking | [references/testing.md](./references/testing.md) |
| Design patterns, idioms, modern Node/TS features | [references/design-and-modern.md](./references/design-and-modern.md) |
| Security setup (auth/JWT/CORS/helmet/CSRF), anti-patterns, smells | [references/security-and-antipatterns.md](./references/security-and-antipatterns.md) |

Notes:
- **Auth / security** questions usually need *both* `web-frameworks.md` (to locate the middleware/guard) and `security-and-antipatterns.md` (to read what to check). Grep for the auth library name too.
- **Logging / metrics / tracing / health checks** aren't a dedicated file: grep for the observability deps and config, and use `control-flow.md` for error-handling paths.
- **Build / run / scripts** come straight from `package.json` `scripts` + lockfile detection (`foundation.md`) — no extra file needed.
- If the question spans concerns, read multiple files. When unsure where to start, `foundation.md` is the orientation layer.

### 4. Investigate

Right tool for the shape:

- **`Read`** when you know the file path. Read in full — JS/TS config files (`tsconfig.json`, `eslint.config.*`, framework configs) are short and dense.
- **`Grep`** for precise symbol/decorator/library matches.
- **`Glob`** for file discovery patterns.
- **`Agent` with `subagent_type: "Explore"`** when the search returns many results and you need a structured summary. Constrain output with a return format and limit. Template below.
- **`WebFetch`** when you hit an unfamiliar npm package and need its docs.

### Subagent prompt template

```
[TASK]:        [Specific thing to find]
[SCOPE]:       **/src/**   (or the explicit package paths list if monorepo)
[EXCLUDE]:     **/node_modules/**, **/dist/**, **/build/**, **/out/**,
               **/.next/**, **/.nuxt/**, **/.turbo/**, **/.cache/**,
               **/coverage/**, **/.git/**, **/*.d.ts
               [+ project-specific: Prisma generated client, openapi-generated, etc.]
[RETURN FORMAT]: [Table with specific columns / numbered list / count]
[LIMIT]:       [N — usually 10-30]
[THOROUGHNESS]: medium | very thorough
```

(The `Explore` subagent only accepts `medium` or `very thorough` — there is no `quick`.) Subagent results are *leads*: re-`Read` the cited `file:line` before presenting it as evidence.

### Node-specific traps (always apply)

1. **Decorator usage** (NestJS / TypeORM / class-validator / type-graphql / tsyringe / routing-controllers): Source-only reading is incomplete. Behavior is generated from metadata at runtime via reflection. Read `references/foundation.md` (Decorator-aware reading) before drawing conclusions about routing, DI, or schema.

2. **ESM vs CJS**: `import` vs. `require`, `__dirname` vs. `import.meta.url`, conditional `exports` in package.json — get this wrong and you'll misread the module graph. Check `"type"` in package.json first.

3. **Path aliases**: tsconfig `compilerOptions.paths` (e.g., `"@/*": ["./src/*"]`) means a grep for `'./services/user'` won't match imports written as `'@/services/user'`. Search for both forms.

4. **Monorepo scope**: `grep '@Controller' src/main/` silently misses controllers in `apps/*/src/`. Use `**/src/**` or iterate the package-paths list.

5. **Generated code**: Annotation matches inside `node_modules/.prisma/`, `target/generated-sources/`, swagger-output dirs are *generated* — always exclude.

6. **Edge runtimes** (`wrangler.toml`, `runtime: 'edge'`): The runtime is V8 isolate, not Node. `fs`, `child_process`, native modules don't work. Many npm packages will fail. Don't recommend Node-specific APIs without checking the runtime.

7. **Profile-specific config**: `.env.development`, `.env.production`, NODE_ENV-aware code branches — answer may differ between environments. Check which env the user is asking about.

8. **Async pitfalls**:
   - Missing `await` (`const r = doThing()` where `doThing` returns a Promise) — the variable is the promise, not the value.
   - Unhandled rejection vs. uncaught exception have different runtime behavior; some codebases crash on the first, swallow the second, or vice versa.
   - Express 4 doesn't auto-catch async handler errors. Express 5 does. Check the version.

9. **Multiple lockfiles**: If you see both `package-lock.json` and `yarn.lock`, the team is split. Don't trust either as the canonical install state.

10. **`NEXT_PUBLIC_*` env vars in Next.js**: Anything with this prefix is bundled into client JS — visible publicly. Flag if a secret has this prefix.

### 5. Answer

**Answer format**:

- **Lead with the direct answer** in 1–3 sentences. No throat-clearing.
- **Cite evidence**: every non-trivial claim gets a `file:line` reference using markdown link syntax: `[auth.middleware.ts:42](src/middleware/auth.middleware.ts#L42)`.
- **Show, don't paraphrase**: paste 3–10 lines of relevant code when it makes the answer concrete. Never invent code.
- **If a flow spans files**, include a short Mermaid sequence diagram. Keep it under 10 nodes.
- **Flag uncertainty plainly**: *"I see the JWT middleware but I don't see where the secret is set — likely loaded by `dotenv` from `.env.production`, which isn't in the repo. Want me to confirm?"*
- **Offer ONE concrete follow-up**, not a menu (*"Want me to trace how the response gets serialized?"* / *"Want me to check what happens on failure?"*).
- **If the answer isn't in the code**, say so plainly. Suggest the most likely external place (config server, env var, infra repo) but don't fabricate.

**What not to do**:

- Don't produce a structured multi-section report. That's `onboard-node`'s job.
- Don't ask for Quick/Standard/Deep mode. There is no mode in this skill.
- Don't create files unless the user explicitly asks. Answers go in chat.
- Don't write a ledger or output directory. State is the conversation.
- Don't refuse to answer because "Phase 1 isn't done." There are no phases.

## Follow-up rounds

A real session usually has 2–6 follow-up questions building on the same investigation. Reuse what you already learned:

- **Cache the Step 2 orient checks** mentally.
- **Cache key file findings.** Don't re-read a file you already analyzed this session.
- **If the user pivots to a new area** (e.g., started on auth, now Kafka), re-route via Step 3 and load the new guide section.

If conversation grows long and you notice you're losing track, summarize prior findings briefly and continue. Never claim something you haven't verified just because you "remember it."

## What this skill is NOT

- Not an onboarding generator → use `onboard-node`.
- Not a code-modification tool → just edit code directly.
- Not a general Node/TS tutor → answer training-knowledge questions without invoking codebase reading.
- Not a code reviewer → use a review skill.
- Not a security audit → use a security-review skill; this skill answers *"how is security set up here?"*, not *"is it good?"*.

## Reference files

The routing table in **Step 3** is the single catalog of reference files — match the question shape there and `Read` the matching file. Each file is scoped to one concern (`foundation`, `web-frameworks`, `integrations`, `control-flow`, `data-layer`, `testing`, `design-and-modern`, `security-and-antipatterns`), so one whole-file read loads exactly what the question needs and nothing else. `foundation.md` carries the Node-specific traps (decorator-aware reading, ESM/CJS, edge-runtime caveats) and is the orientation layer for most questions.
