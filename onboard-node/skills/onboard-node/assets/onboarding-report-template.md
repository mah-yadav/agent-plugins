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

# Onboarding Report: [Project Name]

**Date**: [Date]
**Project**: [name from package.json]
**Repository**: [repo URL if known]
**Developer**: [Name or role]
**Focus**: [Backend features / Bug fixes / API integrations / DevOps / Full stack]
**Mode**: [Quick / Standard / Deep]

---

## 1. Project Identity

| Attribute | Value |
|---|---|
| **Project name** | |
| **Purpose** | [1–2 sentence description] |
| **Node version** | [from .nvmrc / engines] |
| **Package manager** | [npm / pnpm / yarn / bun] |
| **Module system** | [ESM / CJS / Mixed] |
| **Language** | [TypeScript / JavaScript / Mixed (X% TS)] |
| **Framework** | [Express / Fastify / NestJS / Next.js / Hono / other] |
| **Decorator usage** | [Yes (NestJS / TypeORM / ...) / No] |
| **Edge runtime target** | [None — Node only / Cloudflare Workers / Vercel Edge / Deno] |
| **Architecture** | [Layered / Feature-based / Hexagonal / Modular] |
| **Monorepo?** | [Yes — npm / pnpm / Yarn / Turborepo / Nx — list packages / No] |

## 2. Tech Stack at a Glance

| Layer | Technology | Notes |
|---|---|---|
| Framework | | |
| Build tool | | |
| Type system | | |
| Database | | |
| ORM / Query builder | | |
| Migrations | | |
| Validation | | |
| Caching | | |
| Messaging / Queue | | |
| Auth | | |
| Testing | | |
| API docs | | |
| Logging | | |
| Metrics | | |
| Tracing | | |
| Error tracking | | |
| CI/CD | | |
| Deployment | | |

## 3. Build & Run Locally

> **Full guide**: [build-local-dev-guide.md](./build-local-dev-guide.md) *(Standard/Deep mode only)*

### Quick Start

```bash
# Prerequisites: Node [version], [npm/pnpm/yarn], [Docker if needed]

# Install
[npm ci / pnpm install --frozen-lockfile / yarn install --frozen-lockfile]

# Build
[npm run build]

# Dev (watch mode)
[npm run dev]

# Test
[npm test]
```

### Key Environment Variables

| Variable | Purpose | Default | Required |
|---|---|---|---|
| | | | |

### Local Services

| Service | Purpose | How to Start | Port |
|---|---|---|---|
| | | | |

## 4. Architecture, APIs, Data & Testing

> **Full report**: [code-explanation-report.md](./code-explanation-report.md) *(Standard/Deep mode only)*

### Architecture

| Attribute | Value |
|---|---|
| Architectural pattern | |
| Entry point(s) | |
| Bootstrap sequence | |

### Package / Layer Map

| Package or Path | Role |
|---|---|
| | |

### APIs

- **Total routes / endpoints**:
- **Sample endpoints**:
  | Method | Path | Auth | Purpose |
  |---|---|---|---|
  | | | | |

### Data Models

| Name | Purpose | Key Fields |
|---|---|---|
| | | |

### Request Lifecycle

(Mermaid sequence diagram embedded here for one representative request)

### Testing

- **Frameworks**:
- **Test commands**:
  | What | Command |
  |---|---|
  | Unit | |
  | Integration | |
  | Single test | |
  | Coverage | |

## 5. Security

> **Full report**: [security-analysis-report.md](./security-analysis-report.md) *(Deep mode only)*

| Topic | Finding |
|---|---|
| Authentication mechanism | |
| Authorization model | |
| Secrets management | |
| Local dev auth steps | |
| Notable anti-patterns | |

## 6. Observability

> **Full report**: [observability-analysis-report.md](./observability-analysis-report.md) *(Deep mode only)*

| Topic | Finding |
|---|---|
| Logging library | |
| Log format | |
| Log destination(s) | |
| Health endpoint(s) | |
| Metrics backend | |
| Tracing setup | |
| Error tracking | |

### Debugging Cheat Sheet

| Scenario | What to do |
|---|---|
| | |

## 7. CI/CD & Deployment

> **Full report**: [cicd-deployment-report.md](./cicd-deployment-report.md) *(Standard/Deep mode only)*

| Topic | Finding |
|---|---|
| CI platform | |
| Pipeline stages | |
| Quality gates / what blocks PR | |
| Deployment target | |
| Environments | |
| Release process | |

## 8. Code Quality & Conventions

> **Full report**: [code-quality-report.md](./code-quality-report.md) *(Standard/Deep mode only)*

### What Will Fail My PR?

| Check | Tool | Fix Command |
|---|---|---|
| | | |

### Auto-format

```bash
[npm run format / pnpm format / etc.]
```

### Notable Conventions

-

## 9. First Tasks: Suggested Path

1. Install dependencies and run the project locally.
2. Run the test suite.
3. Walk one request end-to-end (refer to § 4 sequence diagram).
4. Try a small change in a non-critical area; run lint/format/typecheck.
5. Open a draft PR; observe the pipeline.

## 10. Still Need to Ask the Team

| Topic | Why |
|---|---|
| | |

## 11. Coverage Notes

[Sections analyzed vs. skipped due to mode/context]
