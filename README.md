# agent-plugins

A Claude Code plugin marketplace for **onboarding onto** and **asking questions about** Java and Node.js codebases.

Each plugin gives Claude framework-specific expertise so its answers and reports are grounded in how real Spring/Quarkus/Express/NestJS projects are actually structured — with `file:line` evidence rather than generic guesses.

## Plugins

| Plugin | Purpose |
| --- | --- |
| [`ask-java-codebase`](ask-java-codebase/) | Conversational, evidence-first Q&A about a Java codebase (Spring Boot, Quarkus, Micronaut, Jakarta EE, WebFlux, Lombok). Answers go in chat — no report files. |
| [`ask-node-codebase`](ask-node-codebase/) | Conversational, evidence-first Q&A about a Node.js codebase (Express, Fastify, NestJS, Hono, Next.js, tRPC, Prisma/Drizzle/TypeORM, monorepos, ESM/CJS). Answers go in chat — no report files. |
| [`onboard-java`](onboard-java/) | Onboard a developer onto a Java codebase via a phased workflow (orient → scope → mine build/architecture/security/observability/CI-CD/code-quality → questions → report). Produces a written onboarding report. |
| [`onboard-node`](onboard-node/) | Onboard a developer onto a Node.js codebase via the same phased workflow. Produces a written onboarding report. |

**Which do I want?** Use an `ask-*` plugin when you have a question and want the answer in chat. Use an `onboard-*` plugin when you want a written onboarding artifact to share or hand off.

## Installation

Add this marketplace, then install the plugins you want:

```
/plugin marketplace add mah-yadav/agent-plugins
/plugin install ask-java-codebase
/plugin install ask-node-codebase
/plugin install onboard-java
/plugin install onboard-node
```

You can also browse and install interactively:

```
/plugin
```

After installing, each plugin's skill becomes discoverable — just ask your question (for `ask-*`) or ask Claude to onboard you onto the codebase (for `onboard-*`).

## Repository layout

```
agent-plugins/
├── .claude-plugin/
│   └── marketplace.json     ← marketplace manifest (lists the four plugins)
├── ask-java-codebase/
├── ask-node-codebase/
├── onboard-java/
└── onboard-node/
```

Each plugin directory has its own `.claude-plugin/plugin.json`, a `skills/` folder, and a `README.md` with full details — see each plugin's `README.md` to learn more.

## Author

Mahesh Yadav
