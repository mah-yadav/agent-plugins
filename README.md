# agent-plugins

A Claude Code plugin marketplace for **onboarding onto**, **asking questions about**, **analyzing**, and **modernizing** codebases.

The `ask-*`, `onboard-*`, and `*-app-advisor` plugins give Claude framework-specific expertise so its answers, reports, and findings are grounded in how real Spring/Quarkus/Express/NestJS projects are actually structured — with `file:line` evidence rather than generic guesses. The `modernize` plugin is language-agnostic: it drives a safe, staged replacement of a library, framework, or component across any codebase.

## Plugins

| Plugin | Purpose |
| --- | --- |
| [`ask-java-codebase`](ask-java-codebase/) | Conversational, evidence-first Q&A about a Java codebase (Spring Boot, Quarkus, Micronaut, Jakarta EE, WebFlux, Lombok). Answers go in chat — no report files. |
| [`ask-node-codebase`](ask-node-codebase/) | Conversational, evidence-first Q&A about a Node.js codebase (Express, Fastify, NestJS, Hono, Next.js, tRPC, Prisma/Drizzle/TypeORM, monorepos, ESM/CJS). Answers go in chat — no report files. |
| [`onboard-java`](onboard-java/) | Onboard a developer onto a Java codebase via a phased workflow (orient → scope → mine build/architecture/security/observability/CI-CD/code-quality → questions → report). Produces a written onboarding report. |
| [`onboard-node`](onboard-node/) | Onboard a developer onto a Node.js codebase via the same phased workflow. Produces a written onboarding report. |
| [`java-app-advisor`](java-app-advisor/) | Critically analyze a Java application (Spring Boot, Micronaut, Quarkus, plain Java) for architectural, design, security, performance, and technical-debt issues via a phased, evidence-first workflow. Produces a prioritized Findings Report and a Remediation Backlog with exact before/after code changes. |
| [`node-app-advisor`](node-app-advisor/) | Critically analyze a Node.js application (Express, Fastify, NestJS, Koa, Hono, Next.js, tRPC, plain Node; JS or TS) for the same five issue areas via the same phased workflow. Produces a prioritized Findings Report and a Remediation Backlog with exact before/after code changes. |
| [`modernize`](modernize/) | Replace a library, framework, or component with a new one across a codebase via a 3-stage, document-driven pipeline (analyze usage → plan the replacement → execute & verify). Language-agnostic. Produces a Usage Report, Replacement Plan, and Replacement Report. |

**Which do I want?** Use an `ask-*` plugin when you have a question and want the answer in chat. Use an `onboard-*` plugin when you want a written onboarding artifact to share or hand off. Use a `*-app-advisor` plugin when you want a critical analysis of an app's architecture, security, performance, and technical debt with a prioritized remediation backlog. Use `modernize` when you want to swap one library, framework, or component for another across the codebase.

## Installation

Add this marketplace, then install the plugins you want:

```
/plugin marketplace add mah-yadav/agent-plugins
/plugin install ask-java-codebase
/plugin install ask-node-codebase
/plugin install onboard-java
/plugin install onboard-node
/plugin install java-app-advisor
/plugin install node-app-advisor
/plugin install modernize
```

You can also browse and install interactively:

```
/plugin
```

After installing, each plugin's skill becomes discoverable — just ask your question (for `ask-*`), ask Claude to onboard you onto the codebase (for `onboard-*`), ask Claude to analyze or review the application (for `*-app-advisor`), or ask Claude to replace one library/framework/component with another (for `modernize`).

## Repository layout

```
agent-plugins/
├── .claude-plugin/
│   └── marketplace.json     ← marketplace manifest (lists the seven plugins)
├── ask-java-codebase/
├── ask-node-codebase/
├── java-app-advisor/
├── modernize/
├── node-app-advisor/
├── onboard-java/
└── onboard-node/
```

Each plugin directory has its own `.claude-plugin/plugin.json`, a `skills/` folder, and a `README.md` with full details — see each plugin's `README.md` to learn more.

## Author

Mahesh Yadav
