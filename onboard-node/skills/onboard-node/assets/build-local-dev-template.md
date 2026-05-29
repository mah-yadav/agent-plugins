# Build & Local Development Guide: [Project Name]

**Date**: [Date]

## 1. Prerequisites

- Node.js: [version]
- Package manager: [npm/pnpm/yarn/bun version]
- Docker: [yes/no — and version if yes]
- [Other tools: e.g., direnv, mkcert, AWS CLI]

## 2. Quick Start

```bash
# Clone (if not already)
git clone <repo>
cd <repo>

# Install
[install command]

# Copy env template
cp .env.example .env

# Start local services
docker compose up -d   # if applicable

# Run migrations / seed
[db:migrate command]
[db:seed command]

# Start the app
[npm run dev]
```

The app should be available at: `http://localhost:[port]`
Health check: `curl http://localhost:[port]/health`

## 3. Package Manager & Lockfile

- **Active package manager**: [npm / pnpm / yarn / bun]
- **Lockfile**: [package-lock.json / pnpm-lock.yaml / yarn.lock / bun.lock]
- **`packageManager` pin**: [version or "not pinned"]
- **Install command (CI-clean)**: `[npm ci / pnpm install --frozen-lockfile / yarn install --frozen-lockfile]`
- **Adding a dep**: `[npm install <pkg> / pnpm add <pkg> / yarn add <pkg>]`
- **Workspace dep** (monorepo): `[npm install -w <workspace> <pkg> / pnpm add <pkg> --filter <pkg> / yarn workspace <name> add <pkg>]`

## 4. Node Version Management

- **Required**: [version]
- **Recommended manager**: [nvm / fnm / volta / asdf / mise]
- **Switch**: `[nvm use / fnm use / etc.]`

## 5. Environment Variables

### Required

| Variable | Purpose | Example |
|---|---|---|
| | | |

### Optional (have defaults)

| Variable | Purpose | Default |
|---|---|---|
| | | |

### Where they're loaded

- [.env (development), .env.test (testing), etc. — and loader: dotenv / dotenv-flow / framework-native]

## 6. Local Services

| Service | Purpose | How to Start | Connection |
|---|---|---|---|
| | | | |

### Seed Data

- Command: `[db:seed]`
- What it creates: [test users, fixtures, ...]

## 7. Common Tasks

| Task | Command |
|---|---|
| Build | |
| Type check | |
| Lint | |
| Format | |
| Test (unit) | |
| Test (integration / e2e) | |
| Test (single file) | |
| Test (single name) | |
| Coverage | |
| Migration: create | |
| Migration: apply | |
| Reset DB | |
| Start (dev) | |
| Start (prod-like) | |

## 8. IDE Setup

### VS Code

- Recommended extensions: [from .vscode/extensions.json]
- Format on save: [yes/no — and setup]
- ESLint auto-fix on save: [yes/no]

### JetBrains (WebStorm / IntelliJ)

- [Required settings]

## 9. Hot Reload / Watch

- Dev server: [`npm run dev` uses `tsx watch` / `nodemon` / `node --watch` / `nest start --watch` / etc.]
- Re-attach on save?: [yes/no]

## 10. Debug

- Node inspector: `node --inspect dist/index.js` then attach via chrome://inspect or IDE.
- Launch config (VS Code): [if `.vscode/launch.json` exists, summarize]

## 11. Scripts in `package.json`

| Script | What it does |
|---|---|
| | |

## 12. Troubleshooting

| Symptom | Likely cause / Fix |
|---|---|
| | |
