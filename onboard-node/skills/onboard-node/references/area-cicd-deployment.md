# Area: CI/CD & Deployment Analysis (Node)

Reference for Phase 3.5. Produces a structured report covering pipeline stages, containerization, deployment targets, infrastructure as code, release process, environment promotion.

**Core principle**: Trace the path to production. Map the entire delivery pipeline so the developer knows what happens when they push code, what gates their PR, and how deployments work.

## Contents

- **Priority tiers** · **Ledger read-before**
- **Step 1** Identify the CI/CD & deployment stack · **Step 2** Analyze CI pipeline (structure, build, test, quality gates, secrets)
- **Step 3** Containerization · **Step 4** Deployment (k8s, Helm, serverless, edge, IaC) · **Step 5** Release process · **Step 6** Environment management
- **Step 7** Generate report · **Ledger update after** · **Exploration guidelines**

## Priority Tiers (within this area)

| Step | Priority | Rationale |
|---|---|---|
| Step 1: Identify CI/CD stack | **P0** | Must know where the pipeline is |
| Step 2.1: Pipeline structure | **P0** | Must understand the flow |
| Step 2.4: Quality gates | **P0** | Must know what blocks a PR |
| Step 3.1: Dockerfile analysis | **P1** | Container image details |
| Step 4: Deployment target | **P1** | Where the code runs |
| Step 2.2–2.3: Build & test stage detail | **P1** | Useful context |
| Step 5: Release process | **P1** | Versioning, changelogs |
| Step 6: Environment management | **P1** | Environment differences |
| Step 2.5: Secrets in pipeline | **P2** | DevOps detail |

**Quick mode**: Steps 1, 2.1, 2.4. Skip the rest.

## Ledger Read-Before

Check the ledger. If `area-build-local-dev` already analyzed `Dockerfile` and `docker-compose.yml`, skip re-reading — use those findings. Focus this analysis on CI pipeline files and deployment manifests.

Note `Edge runtime target` — Cloudflare/Vercel Edge deployments use different tooling (Wrangler, Vercel CLI) than container-based.

## Step 1: Identify the CI/CD & Deployment Stack — P0

1. **CI pipeline files**:

   | File / Pattern | CI/CD Platform |
   |---|---|
   | `.github/workflows/*.yml` | GitHub Actions |
   | `.gitlab-ci.yml` | GitLab CI |
   | `Jenkinsfile` | Jenkins |
   | `azure-pipelines.yml` | Azure DevOps |
   | `.circleci/config.yml` | CircleCI |
   | `bitbucket-pipelines.yml` | Bitbucket Pipelines |
   | `cloudbuild.yaml` | Google Cloud Build |
   | `buildspec.yml` | AWS CodeBuild |
   | `drone.yml` | Drone CI |
   | `.woodpecker.yml` | Woodpecker CI |

2. **Container files** (check ledger first):

   | File / Pattern | What |
   |---|---|
   | `Dockerfile` | Application container |
   | `Dockerfile.*` (e.g., `Dockerfile.prod`) | Profile-specific images |
   | `apps/<name>/Dockerfile` | Per-package images in monorepos |
   | `docker-bake.hcl` | Docker Buildx multi-image builds |

3. **Deployment manifests / configs**:

   | Directory / File | Target |
   |---|---|
   | `k8s/`, `kubernetes/`, `deploy/` | Raw Kubernetes YAML |
   | `charts/`, `helm/` | Helm charts |
   | `kustomize/`, `kustomization.yaml` | Kustomize overlays |
   | `terraform/`, `*.tf` | Terraform IaC |
   | `pulumi/`, `Pulumi.yaml` | Pulumi IaC |
   | `serverless.yml` | Serverless Framework |
   | `sst.config.{ts,js}` | SST (AWS serverless) |
   | `cdk.json`, `bin/` (with CDK code) | AWS CDK |
   | `vercel.json` / `.vercel/` | Vercel |
   | `netlify.toml` | Netlify |
   | `wrangler.toml` / `wrangler.jsonc` | Cloudflare Workers / Pages |
   | `fly.toml` | Fly.io |
   | `railway.toml` / `railway.json` | Railway |
   | `render.yaml` | Render |
   | `app.yaml` | Google App Engine |
   | `Procfile` | Heroku / Cloud Foundry |
   | `apprunner.yaml` | AWS App Runner |

**Present findings**: CI platform, deployment target(s), containerization approach.

## Step 2: Analyze CI Pipeline — P0/P1

### 2.1 Pipeline Structure — P0

| What to Find | Where to Look |
|---|---|
| Trigger events | `on:` (GitHub Actions), `triggers` (Jenkins), `workflow:rules` (GitLab) |
| Pipeline jobs/stages | Job definitions and ordering |
| Job dependencies | `needs:` (GHA), `stage:` (GitLab), `depends_on` |
| Conditional execution | `if:`, branch/path filters, matrix strategies |
| Matrix builds | Node version × OS combinations |
| Reusable workflows / composite actions | `uses: ./.github/actions/*`, `workflow_call` |

**Extract**: Pipeline flow (stages, ordering), triggers, required checks for PR.

### 2.2 Build Stage — P1

| What to Find | Where to Look |
|---|---|
| Node version setup | `actions/setup-node` `node-version` input; matches `.nvmrc`? |
| Package manager setup | `actions/setup-node` `cache: 'npm'` / `'pnpm'`, or explicit `pnpm/action-setup` |
| Install command | `npm ci`, `pnpm install --frozen-lockfile`, `yarn install --frozen-lockfile` |
| Build command | `npm run build` (or PM equiv) |
| Type check | Separate step? `tsc --noEmit` or `npm run typecheck` |
| Artifacts | What's uploaded (build output, coverage, SBOMs)? Where? |
| Build cache | `actions/cache` keys for node_modules, build output, Turborepo/Nx remote cache |

Look for **Turborepo remote cache** (`turbo` env vars `TURBO_TOKEN`, `TURBO_TEAM`) or **Nx Cloud** — major CI speedup if used; flag if missing in large monorepos.

### 2.3 Test Stage — P1

- Unit test command and runner detected.
- Integration/E2E test command — often a separate job with services started via Compose or service containers.
- Test reporting: JUnit XML upload (`mocha-junit-reporter`, Jest `jest-junit`), GitHub annotations.
- Coverage upload: Codecov / Coveralls / SonarQube.

### 2.4 Quality Gates — P0

| What to Find | Where to Look |
|---|---|
| Type check | `tsc --noEmit` step |
| Lint | `eslint` / `oxlint` / `biome lint` step |
| Format check | `prettier --check` / `biome format --check` |
| Tests passing | Required test step |
| Coverage threshold | Jest/Vitest `coverageThreshold`, or CI threshold gate |
| Dependency vulnerability scan | `npm audit` / `pnpm audit`, Snyk, Socket, Dependabot alerts |
| Container scan | Trivy, Grype, Snyk container |
| Bundle size | `size-limit`, `bundlesize`, `bundlewatch` — common for libraries |
| License check | `license-checker`, custom |
| Branch protection clues | Required status checks (visible in `.github/branch-protection.yml` rarely, or via API) |

**Extract**: Table of checks that gate a PR merge.

### 2.5 Secrets & Credentials in Pipeline — P2

- Repository secrets, organization secrets, environment-scoped secrets.
- OIDC / workload identity (modern; avoids long-lived creds).
- Service accounts.
- Where secrets are injected: env vars in jobs, runtime config.

## Step 3: Analyze Containerization — P1

If `area-build-local-dev` already analyzed the Dockerfile, use those findings; only re-read for CI/CD-relevant aspects.

### 3.1 Dockerfile Analysis — P1

| What to look for | Why |
|---|---|
| Base image | `node:20-alpine` (small but missing native deps), `node:20-slim` (Debian, larger compat), `distroless/nodejs20` (minimal attack surface), `gcr.io/distroless/static` for binaries |
| Multi-stage build | `FROM ... AS build` then `FROM ... AS runtime` copying `dist/` and prod deps only |
| Native module rebuild | `node-gyp` deps; `python3`, `make`, `g++` install in build stage |
| Production install | `npm ci --omit=dev` / `pnpm install --prod --frozen-lockfile` |
| Runtime user | `USER node` (uid 1000) — not running as root |
| Port exposure | `EXPOSE 3000` — informational only, not enforcement |
| Entry point | `CMD ["node", "dist/index.js"]` vs. `npm start` (extra process layer; signal handling subtler) |
| Health check | `HEALTHCHECK` instruction (rare; usually k8s probes instead) |
| Signal handling | `tini` / `dumb-init` for proper PID-1 signal forwarding (or use `node` directly which handles SIGTERM in recent versions) |
| Layer optimization | Are deps installed before source copy? (dep layer cache hit on source-only change) |
| Monorepo concerns | Build artifact extraction from a workspace; `pnpm deploy` or `turbo prune` |

### 3.2 Alternative containerization — P2

- **Buildpacks** (Heroku-style): `project.toml` or no Dockerfile at all.
- **`pkg`** / **`nexe`** / **Bun bundles**: single-binary distribution (rare).
- **Vercel / Netlify**: serverless functions, no container.

### 3.3 Image Registry — P2

- Registry: Docker Hub, GitHub Container Registry (ghcr.io), AWS ECR, GCP Artifact Registry, Azure ACR, private registry.
- Tag strategy: semver, git SHA, `latest`, multi-arch.
- Push step in CI: how authenticated.

## Step 4: Analyze Deployment — P1

### 4.1 Kubernetes (if applicable)

- Deployment/StatefulSet manifests, Service/Ingress, ConfigMap/Secret refs.
- Resource limits/requests.
- Replicas / HPA.
- Liveness/readiness probes (path, port, timing).
- Namespace conventions.

### 4.2 Helm (if applicable)

- Chart structure: `Chart.yaml`, `values.yaml`, `templates/`, `charts/` (subcharts).
- Default values vs. environment overrides (`values-prod.yaml`).

### 4.3 Serverless (Vercel / Netlify / Cloudflare / AWS Lambda)

| Platform | Config |
|---|---|
| **Vercel** | `vercel.json` (rewrites, headers, runtimes), env vars in Vercel UI, `.vercel/` |
| **Netlify** | `netlify.toml`, `functions/` dir or `netlify/functions/`, `_redirects`, `_headers` |
| **Cloudflare Workers** | `wrangler.toml` (or `wrangler.jsonc`); routes, kv namespaces, durable objects, secrets via `wrangler secret put` |
| **AWS Lambda** | Serverless Framework (`serverless.yml`), SAM (`template.yaml`), CDK, SST. Cold starts. Layers. |
| **AWS App Runner** | `apprunner.yaml`; container-based but managed |
| **Fly.io** | `fly.toml`; closer to containers + machines |
| **Railway** | `railway.toml` / `nixpacks.toml` for build |
| **Render** | `render.yaml`; static + service spec |

For serverless Node, **note cold-start mitigations**: warming pings, provisioned concurrency, smaller bundles (`@vercel/ncc`).

### 4.4 Edge runtimes — special handling

If `Edge runtime target = Yes` in the ledger:
- Deployment is via `wrangler deploy` (Cloudflare), `vercel deploy` with edge config, or `deno deploy`.
- No Node-specific deployment artifacts (no Dockerfile, no k8s).
- Build target must produce edge-compatible JS (no Node-specific APIs in bundle).

### 4.5 Infrastructure as Code — P1

- Terraform: `*.tf` files, state backend (`s3`, `gcs`, `cloud` block), modules.
- Pulumi: `Pulumi.yaml`, language SDK choice (TS/Python/Go).
- AWS CDK: `cdk.json` + TS/Python entrypoint.
- SST: `sst.config.ts` (newer AWS-focused TS framework).

## Step 5: Analyze Release Process — P1

- Versioning: semantic-release, changesets (`.changeset/` directory + `@changesets/cli`), manual.
- Changelog: `CHANGELOG.md` generation tool — `conventional-changelog`, semantic-release, changesets.
- Release branches: `main`-only, `develop`+`main`, release branches, or trunk-based.
- Approval gates: required reviewers, manual approval steps in pipeline.
- Rollback strategy: previous image tag, blue/green, canary, feature flags.
- Feature flags: `unleash-client`, `launchdarkly-node-server-sdk`, `@growthbook/growthbook`, custom.

## Step 6: Analyze Environment Management — P1

- Environment list: dev, staging, prod (often more — preview, canary, qa).
- Config per environment: env vars defined in CI/CD config, in Vercel/Netlify dashboards, in k8s configmaps.
- Promotion flow: how a build promotes from staging → prod.
- Preview environments: Vercel/Netlify branch previews, Render preview environments, Kubernetes preview namespaces.

## Step 7: Generate CI/CD Report

Write using the [CI/CD template](../assets/cicd-deployment-template.md).

**Quick mode**: Fill the *Delivery Stack*, *CI Pipeline* (Trigger events, Stages, Caching), and *Quality Gates — What Blocks a PR* sections. Mark others `[Not analyzed]`.

Default location: `<output-dir>/cicd-deployment-report.md` (Standard/Deep only).

## Ledger Update After

Mark area 3.5 complete. Add:
- CI platform and pipeline stages
- Quality gates / required checks for PR
- Deployment target(s) and environment list
- Release process summary

## Exploration Guidelines

- **Read pipeline files in full** — the YAML defines the entire delivery process; skimming misses gates.
- **Read Dockerfiles in full** — every instruction matters.
- **Follow the evidence chain**: Pipeline pushes image → find Dockerfile. Helm chart → find values files. Wrangler config → find Workers bindings.
- **Check ALL environments**: Compare dev/staging/prod configs.
- **Note absences**: No CI? No type check in pipeline? No tests gated? No container scan? No rollback strategy? Critical findings.
- **Don't expose secrets**: Note variable names, never values.
- **Flag risks**: Missing quality gates, no container scanning, no rollback strategy, deployment to prod from any branch.
