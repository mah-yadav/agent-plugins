# Area: CI/CD & Deployment Analysis

Reference for Phase 3.5 of the onboarding workflow. Produces a structured report covering pipeline stages, containerization, deployment targets, infrastructure as code, release process, and environment promotion.

**Core principle**: Trace the path to production. Map the entire delivery pipeline so the developer knows what happens when they push code, what gates their PR, and how deployments work.

## Contents

- Step 1: Identify the CI/CD & Deployment Stack
- Step 2: Analyze CI Pipeline (2.1 structure, 2.2 build, 2.3 test, 2.4 quality gates, 2.5 secrets)
- Step 3: Analyze Containerization (3.1 Dockerfile, 3.2 alternatives, 3.3 registry)
- Step 4: Analyze Deployment (4.1 K8s, 4.2 Helm, 4.3 Kustomize, 4.4 cloud/VM, 4.5 IaC)
- Step 5: Analyze Release Process
- Step 6: Analyze Environment Management
- Step 7: Generate CI/CD Report
- Ledger update, exploration guidelines

## Priority Tiers (within this area)

| Step | Priority | Rationale |
|---|---|---|
| Step 1: Identify CI/CD stack | **P0** | Must know where the pipeline is |
| Step 2.1: Pipeline structure | **P0** | Must understand the flow |
| Step 2.4: Quality gates | **P0** | Must know what blocks a PR |
| Step 3.1: Dockerfile analysis | **P1** | Important for understanding the container |
| Step 4.1: K8s deployment | **P1** | Important for understanding where code runs |
| Step 2.2–2.3: Build & test stage detail | **P1** | Useful context |
| Step 4.2–4.5: Helm, Kustomize, cloud/VM, IaC | **P1** | Important for DevOps-focused developers |
| Step 5: Release process | **P1** | How versions are cut |
| Step 6: Environment management | **P1** | Environment differences |
| Step 2.5: Secrets in pipeline | **P2** | DevOps detail |
| Step 3.2–3.3: Alternative containerization, registries | **P2** | Detail |

**Quick mode**: Complete Step 1, Step 2.1 (pipeline structure), Step 2.4 (quality gates / what blocks a PR). Skip everything else.

## Ledger Read-Before

Check the ledger. If `build-local-dev` already analyzed Docker Compose or the Dockerfile, do NOT re-read those files — pull findings from the ledger. Focus this analysis on CI pipeline files (`.github/workflows/`, `Jenkinsfile`, etc.) and deployment manifests (`k8s/`, `charts/`, `terraform/`) which are unique to this area.

## Step 1: Identify the CI/CD & Deployment Stack — P0

1. **Find pipeline files**:

   | File / Pattern | CI/CD Platform |
   |---|---|
   | `.github/workflows/*.yml` | GitHub Actions |
   | `Jenkinsfile` | Jenkins |
   | `.gitlab-ci.yml` | GitLab CI |
   | `azure-pipelines.yml` | Azure DevOps |
   | `.circleci/config.yml` | CircleCI |
   | `bitbucket-pipelines.yml` | Bitbucket Pipelines |
   | `cloudbuild.yaml` | Google Cloud Build |
   | `buildspec.yml` | AWS CodeBuild |

2. **Find containerization files** (check ledger first):

   | File / Pattern | Indicates |
   |---|---|
   | `Dockerfile` | Application container |
   | `jib` plugin in build file | Google Jib |

3. **Find deployment manifests**:

   | Directory / File | Deployment Target |
   |---|---|
   | `k8s/`, `kubernetes/`, `deploy/` | Raw Kubernetes YAML |
   | `charts/`, `helm/` | Helm charts |
   | `kustomize/`, `kustomization.yaml` | Kustomize overlays |
   | `terraform/`, `*.tf` | Terraform IaC |
   | `serverless.yml` | Serverless Framework |
   | `Procfile` | Heroku / Cloud Foundry |

**Present findings**: CI platform, deployment target, containerization approach.

## Step 2: Analyze CI Pipeline — P0/P1

### 2.1 Pipeline Structure — P0

| What to Find | Where to Look |
|---|---|
| Trigger events | `on:` (GitHub Actions), `triggers` (Jenkins) |
| Pipeline stages/jobs | Stage definitions and ordering |
| Job dependencies | `needs:`, stage ordering |
| Conditional execution | `if:` conditions, branch/path filters |

**Extract**: Pipeline flow (stages, ordering), triggers, required checks for PR.

### 2.2 Build Stage — P1

Build command, Java version, build cache, artifact publishing.

### 2.3 Test Stage — P1

Test command, test reporting, integration test infrastructure, coverage upload.

### 2.4 Quality Gates — P0

| What to Find | Where to Look |
|---|---|
| Static analysis | Checkstyle, PMD, SpotBugs steps |
| SonarQube scan | `sonar:sonar` goal |
| Dependency scanning | OWASP, Snyk, Dependabot |
| Container scanning | Trivy, Grype |
| Required checks | Branch protection clues |

**Extract**: Table of checks that gate a PR merge.

### 2.5 Secrets & Credentials — P2

Secret references, service accounts, OIDC/workload identity.

## Step 3: Analyze Containerization — P1

If `build-local-dev` already analyzed the Dockerfile, use those findings. Only read the Dockerfile fresh if not in the ledger.

### 3.1 Dockerfile Analysis — P1

Base image, multi-stage build, runtime user, port exposure, entry point, health check, layer optimization.

### 3.2 Alternative Containerization — P2

Jib, Buildpacks, `spring-boot:build-image`.

### 3.3 Image Registry — P2

Registry, image naming, tag strategy, push step.

## Step 4: Analyze Deployment — P1

### 4.1 Kubernetes Deployment (if applicable) — P1

Deployment/StatefulSet manifests, Service/Ingress, ConfigMap/Secret references, resource limits, replicas/HPA, health probes, namespace.

### 4.2 Helm Charts (if applicable) — P1

Chart structure, default values, environment overrides, templates, dependencies.

### 4.3 Kustomize (if applicable) — P1

Base resources, overlays, patches, generators.

### 4.4 Cloud/VM Deployment (if not K8s) — P1

ECS, Lambda, App Engine, VM-based (Ansible/Chef), Serverless Framework.

### 4.5 Infrastructure as Code (if applicable) — P1

Provider, resources managed, state management, modules, variables, environments.

## Step 5: Analyze Release Process — P1

Versioning scheme, version bumping, git tagging, changelog, release branches, approval gates, rollback strategy, feature flags.

## Step 6: Analyze Environment Management — P1

Environment list, config per environment, promotion flow, environment URLs, access controls.

## Step 7: Generate CI/CD Report

Write the report using the [CI/CD template](../assets/cicd-deployment-template.md).

**Quick mode**: Fill template sections 1 (Delivery Stack) and 2 (Pipeline Overview — flow, triggers, stages, and the quality gates that block a PR). Mark other sections `[Not analyzed — out of scope for Quick mode]`.

Default location: `<output-dir>/cicd-deployment-report.md` (Deep mode only — in Standard mode, CI/CD findings roll into the main report).

## Ledger Update After

Mark area 3.5 complete. Add:
- CI platform and pipeline stages
- Quality gates / required checks for PR
- Deployment target and environment list
- Release process summary

## Exploration Guidelines

- **Read pipeline files in full** — they define the entire delivery process.
- **Read Dockerfiles in full** (if not already in ledger) — every instruction matters.
- **Follow the evidence chain**: Pipeline pushes image → find Dockerfile. Helm chart → find values files.
- **Check all environments**: Compare dev/staging/prod configs.
- **Note absences**: No CI? No Dockerfile? No health probes? No rollback strategy? Critical findings.
- **Don't expose secrets**: Note variable names, never values.
- **Flag risks**: Missing quality gates, no container scanning, no rollback strategy.
