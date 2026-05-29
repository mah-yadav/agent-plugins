# CI/CD & Deployment Report: [Project Name]

**Date**: [Date]

## 1. Delivery Stack

| Topic | Tool |
|---|---|
| CI platform | [GitHub Actions / GitLab CI / Jenkins / CircleCI / Azure DevOps] |
| Container | [Dockerfile / Jib / Buildpacks / none] |
| Registry | |
| Deployment target | [k8s / Vercel / Netlify / Cloudflare / Lambda / Fly / Render / ...] |
| IaC | [Terraform / Pulumi / CDK / SST / none] |
| Release tooling | [semantic-release / changesets / manual] |
| Feature flags | [LaunchDarkly / Unleash / GrowthBook / custom / none] |

## 2. CI Pipeline

### 2.1 Trigger events

- Push to `main`: [yes — what runs]
- PR: [yes — what runs]
- Tag: [yes — what runs]
- Manual / workflow_dispatch: [yes — purpose]

### 2.2 Stages

| Stage | Command | Required for PR |
|---|---|---|
| Install | | |
| Type check | | |
| Lint | | |
| Format check | | |
| Build | | |
| Unit tests | | |
| Integration tests | | |
| Coverage | | |
| Container build | | |
| Container scan | | |
| Dep vulnerability scan | | |
| Deploy preview | | |

### 2.3 Caching

- Package manager cache: [actions/setup-node `cache:` / explicit `actions/cache`]
- Turborepo / Nx remote cache: [yes — `TURBO_TOKEN` / no]
- Build output cache:

### 2.4 Quality Gates — What Blocks a PR

| Check | Tool | Fail behavior |
|---|---|---|
| | | |

### 2.5 Secrets in pipeline

- Repository / Organization / Environment secrets used: [names only]
- OIDC / Workload identity: [yes — provider / no]

## 3. Containerization

### 3.1 Dockerfile

- Base image: [node:20-alpine / node:20-slim / distroless]
- Multi-stage: [yes — describe / no]
- Native module rebuild step: [yes — for bcrypt/sharp/etc. / no]
- Runtime user: [`USER node` / root — flag]
- Entry point: [`CMD ["node", "dist/index.js"]` / `npm start` / custom]
- Signal handling: [tini / dumb-init / native]
- Health check directive: [yes / no — k8s probes instead]

### 3.2 Image registry

- Registry: [ghcr.io / Docker Hub / ECR / Artifact Registry / ...]
- Tag strategy: [semver / git SHA / latest / multi-arch]

## 4. Deployment

### 4.1 Target

[Describe the runtime — k8s cluster, Vercel project, Cloudflare account, etc.]

### 4.2 Manifests / Config

| Type | File / Location |
|---|---|
| | |

### 4.3 Resource sizing

| Resource | Value |
|---|---|
| CPU request / limit | |
| Memory request / limit | |
| Replicas / min-instances | |
| Autoscaling | |

### 4.4 Probes / Health

| Probe | Path | Port | Timing |
|---|---|---|---|
| Liveness | | | |
| Readiness | | | |

### 4.5 Rollback

- Strategy: [previous image tag / blue-green / canary / feature flags]
- Command / process:

## 5. Release Process

- Versioning: [semver via semantic-release / changesets / manual]
- Changelog: [generated from commits / hand-written]
- Approval gates: [required reviewers / manual deploy step]

## 6. Environment Management

| Environment | URL | Config Source | Promotion From |
|---|---|---|---|
| dev | | | n/a |
| staging | | | dev |
| prod | | | staging |

## 7. Notable Findings

-

## 8. Recommendations (Deep mode)

-
