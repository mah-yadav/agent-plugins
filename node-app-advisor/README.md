# node-app-advisor

**Critically analyze a Node.js application and produce a prioritized findings report plus a remediation backlog.**

A Claude Code plugin that studies a Node.js codebase — Express, Fastify, NestJS, Koa, Hono, Next.js (API routes), tRPC, or plain Node, in JavaScript or TypeScript (web API, SSR app, worker, CLI) — with the rigor of a principal engineer running a production-readiness review, and writes two deliverables backed by `file:line` evidence.

## When to use this plugin

Use `node-app-advisor` when you want a **written assessment with severities and fixes**:

- A production-readiness or pre-release review.
- A security audit, architecture review, performance review, or tech-debt assessment.
- A post-incident review that needs a structured catalog of risks.
- A consultant, auditor, or new tech lead who needs prioritized findings and a remediation plan.

**Don't use this plugin** when you just want to *ask a question* about a codebase (*"how does auth work here?"*) — use the sibling [`ask-node-codebase`](../ask-node-codebase/). And use [`onboard-node`](../onboard-node/) when you want a developer-onboarding map rather than an issue/remediation catalog.

## What you get

Two files written to your chosen output directory (default `docs/node-app-advisor/`; the plugin confirms the location before writing):

| File | Contents |
|---|---|
| `findings-report.md` | Inventory of issues across five areas, each with severity, `file:line` evidence, and impact; executive-summary dashboard with severity counts, top-5 critical findings, and an application profile. |
| `remediation-backlog.md` | Prioritized work items with exact before/after code, T-shirt effort sizing, dependencies, risk, and verification — grouped into four sprint batches. |

## The five analysis areas

| Area | Covers |
|---|---|
| 1. Architecture & Design | Layering violations, SOLID, coupling/cohesion, design anti-patterns, module-boundary issues |
| 2. Security & Dependencies | OWASP Top 10, auth/authz, injection, prototype pollution, hardcoded secrets, vulnerable/outdated dependencies (`npm audit`) |
| 3. Performance & Configuration | Event-loop blocking, N+1 queries, connection pools, caching, memory leaks, config externalization, observability, resilience (timeouts/retries/circuit breakers) |
| 4. Testing & API | Coverage gaps, test quality, REST/RPC/GraphQL API design, contracts, integration testing |
| 5. Technical Debt | Dead code, TODO/FIXME, deprecated APIs, inconsistency, duplication, error handling, type-safety debt |

## Severity classification

| Severity | Criteria | Timeline |
|---|---|---|
| Critical | Exploitable vulnerability, data-loss, crash potential | Immediately — blocks release |
| High | Significant design flaw, perf degradation under load, missing security control | Current sprint |
| Medium | Maintenance-cost code smell, minor hardening | Next 2–3 sprints |
| Low | Convention violation, cosmetic | Opportunistically |

## Workflow at a glance

1. **Phase 1 — Orient** — identify framework, language (JS/TS), module system (ESM/CJS), Node version, package manager, structure, and existing quality tooling.
2. **Phase 2 — Scope** — ask the user for primary concern, known issues, constraints, depth (all five areas or a subset), and cadence (checkpoint every 1–2 areas, or run straight through).
3. **Phase 3 — Analyze** — work the in-scope areas one at a time, loading each area's reference file on demand, sharing findings at the chosen cadence.
4. **Phase 4 — Findings Report** — compile everything into `findings-report.md`.
5. **Phase 5 — Remediation Backlog** — turn findings into a prioritized, batched `remediation-backlog.md`.

For large monorepos (> ~50K LOC), the workflow splits across sessions using the partial Findings Report as the handoff document; the plugin can resume from where it left off.

## Installation

Standard Claude Code plugin install — place the plugin directory where your Claude Code instance loads plugins from:

```
~/.claude/plugins/node-app-advisor/
```

After installing, the `node-app-advisor` skill becomes discoverable.

## Usage

Invoke from the Claude Code chat:

```
Analyze this application and tell me what's wrong with it.
```

```
Run a security audit on the API at services/payments.
```

```
Application health check — production readiness review.
```

```
Just a performance and configuration review of this codebase.
```

A targeted request (e.g. "security audit only") runs just that area and produces a standalone findings section.

## Resuming a partial run

If a session ends mid-analysis, the partial Findings Report remains on disk with completed sections filled in and the rest marked `[TODO — not yet analyzed]`. In a new chat, say:

```
Continue the analysis. The partial report is at docs/node-app-advisor/findings-report.md.
```

## What's inside

```
node-app-advisor/
├── .claude-plugin/plugin.json
├── README.md
└── skills/node-app-advisor/
    ├── SKILL.md                          ← orchestrator: phases, severity, context strategy, tool rules
    ├── references/
    │   ├── architecture-design.md        ← Area 1 detection playbook
    │   ├── security-dependencies.md      ← Area 2 detection playbook
    │   ├── performance-config.md         ← Area 3 detection playbook
    │   ├── testing-api.md                ← Area 4 detection playbook
    │   └── tech-debt.md                  ← Area 5 detection playbook
    └── assets/
        ├── findings-report-template.md
        └── remediation-backlog-template.md
```

The `references/` files are internal building blocks loaded on demand — they are not user-invokable as separate skills.

## Scope and limits

**Targets**: Node.js codebases (JavaScript or TypeScript) using npm, pnpm, or yarn. Specialized for Express, Fastify, NestJS, Koa, Hono, Next.js (API routes), tRPC, and plain Node (web API, SSR app, worker, CLI).

**Does not**:
- Modify source, run builds, or change config by default — it's a read-only analysis that writes only the two report files. Applying fixes is opt-in and only on explicit request.
- Run a dynamic penetration test — the security area is a white-box, code-level review, not runtime exploitation.
- Replace an SCA tool — dependency findings lean on `npm audit` (authoritative) plus EOL/abandonment heuristics; cross-check critical ones and run `npm audit` in CI.

**Honest trade-offs**:
- The 5-phase, evidence-first workflow is thorough but heavyweight. For a single question, the sibling `ask-node-codebase` plugin is the better tool.
- There is no semantic-search tool in Claude Code, so discovery leans on `Grep` regexes and the Explore subagent; very dynamically-wired code (DI containers, decorator metadata, string-keyed lookups, dynamic `import()`) may need a manual pointer, and tsconfig `paths` aliases / barrel re-exports can hide real usages.

## Version

v1.0.0 — single-skill architecture; five area reference files loaded on demand; Node/JS/TS analog of the sibling `java-app-advisor` plugin, with all detection re-grounded in Node.js idioms (event loop, ESM/CJS, npm ecosystem) and Claude Code's native tools.

## Author

Mahesh Yadav
