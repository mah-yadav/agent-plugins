# onboard-java

**Produce a structured onboarding report for a Java codebase.**

A Claude Code plugin that systematically mines a Java project — Spring Boot, Spring WebFlux, Quarkus, Micronaut, or Jakarta EE — and writes an onboarding artifact a new developer or a receiving team can use as their map of the project.

## When to use this plugin

Use `onboard-java` when you need a **written deliverable**:

- A new developer needs an onboarding document before pair-programming time is available.
- A team is offshoring or handing off a codebase and wants a single artifact to share.
- An auditor, consultant, or new tech lead needs a structured assessment.
- The repo will be revisited later and you want a reusable map of its build, architecture, security, observability, CI/CD, and code-quality stack.

**Don't use this plugin** when you just want to *ask a question* about a codebase (*"how does auth work here?"*) — for that, install the sibling plugin [`ask-java-codebase`](../ask-java-codebase/). That plugin is conversational; this one produces files.

## What you get

After running, the plugin saves to your chosen output directory (`docs/` or `.claude/onboarding/` by default):

| File | Mode | Contents |
|---|---|---|
| `onboarding-report.md` | All | Main report — project identity, tech stack, build & run, architecture, security, observability, CI/CD, code quality, "still need to ask" list |
| `.onboarding-findings-ledger.md` | All | Internal state. Tracks what's been analyzed. Resumable across sessions. |
| `build-local-dev-guide.md` | Standard / Deep | Full local-dev guide |
| `code-explanation-report.md` | Standard / Deep | Architecture, APIs, data, testing detail |
| `security-analysis-report.md` | Deep | Full security analysis |
| `observability-analysis-report.md` | Deep | Full observability analysis |
| `cicd-deployment-report.md` | Deep | Full CI/CD analysis |
| `code-quality-report.md` | Deep | Full code quality analysis |

## Modes

The plugin auto-detects project size and recommends a mode in Phase 2; you can override.

| Mode | Project size | Sessions | Output |
|---|---|---|---|
| Quick | < 50 source files, single module | 1 | P0-only report; out-of-scope sections explicitly marked `[Not analyzed]` |
| Standard | 50–200 source files, 2–5 modules | 1–2 | Main + build-local-dev + code-explanation reports |
| Deep | 200+ source files, 6+ modules | 2–4 | All 8 reports |

## Workflow at a glance

1. **Phase 1 — Orient** — find the project, read the build file, detect Lombok / reactive stack / multi-module structure, choose an output dir, create the ledger and report skeleton.
2. **Phase 2 — Scope** — confirm execution mode and priority areas with the user.
3. **Phase 3 — Mine** — analyze six areas (build/local-dev, architecture, security, observability, CI/CD, code quality), one at a time, with user checkpoints every two areas.
4. **Phase 4 — Questions** — compile the "still need to ask the team" list.
5. **Phase 5 — Report** — finalize the artifact(s).

## Installation

Standard Claude Code plugin install — drop the plugin directory under `~/.claude/plugins/` or wherever your Claude Code instance loads plugins from:

```
~/.claude/plugins/onboard-java/
```

After installing, the `onboard-java` skill becomes discoverable.

## Usage

Invoke from the Claude Code chat:

```
Onboard me to this Java codebase.
```

```
Create an onboarding guide for the project at services/payments.
```

```
I need an onboarding artifact I can share with a new hire.
```

```
Generate an onboarding document for this repo.
```

## Resuming a partial run

If a session ends mid-onboarding, the partial report and ledger remain on disk. In a new chat, say:

```
Continue the onboarding. The partial report is at <output-dir>/onboarding-report.md.
```

The plugin runs a staleness check — `git log` since the ledger's last-updated timestamp — and resumes from the first incomplete area, re-reading any files that changed in the meantime.

## What's inside

```
onboard-java/
├── .claude-plugin/plugin.json
└── skills/onboard-java/
    ├── SKILL.md                       ← orchestrator: tiers, ledger contract, subagent rules
    ├── references/
    │   ├── workflow-phases.md         ← Phase 1–5 walkthrough
    │   ├── area-build-local-dev.md    ← Phase 3.1 detail
    │   ├── area-explain-code.md       ← Phase 3.2 detail
    │   ├── area-security.md           ← Phase 3.3 detail
    │   ├── area-observability.md      ← Phase 3.4 detail
    │   ├── area-cicd-deployment.md    ← Phase 3.5 detail
    │   ├── area-code-quality.md       ← Phase 3.6 detail
    │   ├── java-core-analysis.md      ← Java essentials: frameworks, DI/Lombok, control flow (always loaded)
    │   └── java-deep-analysis.md      ← Java depth: data, patterns, testing, security (on demand)
    └── assets/
        ├── onboarding-report-template.md
        └── (six standalone report templates)
```

The `references/` files are internal building blocks loaded on demand — they are not user-invokable as separate skills.

## Scope and limits

**Targets**: Java/JVM server codebases using Maven or Gradle. Specialized for Spring Boot, Spring WebFlux, Quarkus, Micronaut, Jakarta EE. Handles Lombok, multi-module monorepos, and contract-first OpenAPI generators.

**Does not**:
- Run the build by default. Verify is opt-in and per-command-confirmed; onboarding is a read-only mining exercise.
- Deep-analyze non-JVM modules in polyglot repos — those are flagged but not pursued.
- Cover Bazel, SBT, or other less-common JVM build tools.
- Verify correctness of any documented command. Commands are extracted from code, not executed.

**Honest trade-offs**:
- The 5-phase workflow is heavyweight. For a single question about a codebase, the sibling `ask-java-codebase` plugin is the better tool.
- Quick mode produces a partial report with explicit `[Not analyzed]` markers for out-of-scope sections. By design, not a bug — but it means the artifact is shorter than it looks.

## Version

v1.0.0 — single-skill architecture; internal references loaded on demand; reactive-stack / Lombok / multi-module / generated-code handling included.

## Author

Mahesh Yadav
