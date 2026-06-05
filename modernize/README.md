# modernize

A Claude Code plugin for **replacing a library, framework, or component with a new one** across a codebase — safely and systematically.

Migrations fail in predictable ways: a missed plugin in the build config, a string reference in a DI binding, a behavioral difference that survives a syntactically-correct swap, a leftover import in a file nobody indexed. `modernize` is built around those failure modes. It runs a **3-stage, document-driven pipeline** where each stage produces a durable artifact that hands off to the next.

## The pipeline

| Stage | Produces | What it does |
|---|---|---|
| 1 — Analyze usage | **Usage Report** | Maps how the old component is used across 9 dimensions — code, config, build tooling, dynamic/string references, framework integrations. |
| 2 — Plan replacement | **Replacement Plan** | Runs pre-flight compatibility checks, then writes an API mapping table with a mandatory *Behavior Changes* column, execution order, build/test commands, and a rollback strategy. |
| 3 — Execute replacement | **Replacement Report** | Applies the edits in order, manages dependencies, sweeps for every leftover reference, runs the build/tests, and documents the result. |

A single `modernize` skill orchestrates all three stages. It figures out which stage to start at based on which artifacts already exist, so you can run the whole pipeline at once or resume midway. The per-stage workflows live in internal reference files — there are no separate sub-skills to invoke.

## Usage

```
/modernize Replace moment.js with dayjs
```

Or just describe the task in natural language and the skill will trigger and pick the right starting stage:

```
How is lodash used in this codebase?          # runs Stage 1 (analysis) and stops
Plan replacing Enzyme with React Testing Library
Migrate us from axios to fetch
Continue the replacement of Express with Fastify; the report is at docs/migrations/...
```

## Why a document chain

Each artifact captures everything the next stage needs, so:

- **Work is resumable** — including across separate sessions. For a large codebase, run one stage per session and feed the saved document into the next.
- **The plan is a contract** — Stage 3 follows the agreed API mapping table and doesn't improvise transformations.
- **Large migrations batch cleanly** — Stage 3 records progress in the report's *Batch Progress* section and resumes from the first incomplete batch.

## Design principles

- **Real code only.** Every example in every document comes from the actual codebase — nothing fabricated.
- **Behavior over syntax.** The API mapping table forces an explicit *Behavior Changes* entry for every mapping; callers depending on old behavior get updated, not just the call sites.
- **Mandatory final sweep.** A migration isn't done until there are zero unintentional references to the old library — checked with both structured search and `grep -rn`.
- **Always a rollback.** Every plan states how to revert.
- **Interactive checkpoints.** The skills confirm understanding, scope, and plan approval with you before acting — they don't run away with assumptions.

## Layout

```
modernize/
  .claude-plugin/plugin.json
  skills/
    modernize/
      SKILL.md                                  # orchestrator / entry point
      references/
        stage-1-explain-usage.md                # Stage 1 workflow
        stage-2-plan-replacement.md             # Stage 2 workflow
        stage-3-replace-component.md            # Stage 3 workflow
        migration-patterns.md                   # strategy catalog (used in Stage 2)
      assets/
        usage-report-template.md
        replacement-plan-template.md
        replacement-report-template.md
```

A single discoverable skill (`modernize`); the stage workflows are internal reference files, consistent with the `onboard-*` plugins in this marketplace.
