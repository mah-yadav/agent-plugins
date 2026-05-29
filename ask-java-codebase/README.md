# ask-java-codebase

**Answer focused questions about a Java codebase, with Java-specific expertise.**

A Claude Code plugin that turns Claude into a Java codebase consultant. Ask any question about the project — *"how does auth work here?"*, *"where are background jobs defined?"*, *"walk me through what happens when `POST /orders` arrives"* — and get an answer with concrete `file:line` evidence, not a structured document.

## When to use this plugin

Use `ask-java-codebase` when you have a **question** about a Java codebase:

- You're new to a team and want to understand one specific area before pair-programming time is available.
- You're debugging and want to find where a particular flow is implemented.
- You're reviewing a PR and want to know why a piece of code is structured a certain way.
- You're researching how the codebase handles a cross-cutting concern (security, observability, caching, messaging).

**Don't use this plugin** when you need a written onboarding artifact to share with a team or hand off — for that, install the sibling plugin [`onboard-java`](../onboard-java/). That plugin produces files; this one is conversational.

## What this plugin adds over plain Claude Code

Plain Claude can read code. What this plugin adds:

- **Java framework expertise** baked into the workflow: knows where Spring / Quarkus / Micronaut configs live, what `@EnableWebSecurity` implies, how to trace `@KafkaListener`s, how to find `@ConfigurationProperties` classes.
- **Lombok-aware reading**: doesn't conclude "this class has no constructor" when Lombok is generating one. Knows `@Slf4j` provides an invisible `log` field; knows `@RequiredArgsConstructor` + `final` fields is the canonical constructor-injection pattern.
- **Reactive vs. servlet branch awareness**: greps the right Spring Security bean (`SecurityWebFilterChain` vs. `SecurityFilterChain`); knows MDC doesn't propagate across reactive operators without explicit context propagation.
- **Multi-module scope safety**: searches use `**/src/main/java`, not `src/main/java` — catches code in sibling modules that single-module searches miss.
- **Generated-code excludes**: filters out `target/generated-sources/`, `build/generated/`, MapStruct/QueryDSL/protobuf output dirs, so annotation searches don't return false positives from generated stubs.
- **Evidence-first answer format**: every claim cited with `file:line`; if it can't be found in the code, it's said plainly — not fabricated.

## Installation

Standard Claude Code plugin install:

```
~/.claude/plugins/ask-java-codebase/
```

After installing, the `ask-java-codebase` skill becomes discoverable.

## Usage

Just ask. Examples:

```
How does authentication work in this codebase?
```

```
Where are background jobs defined? Show me one.
```

```
Walk me through what happens when a POST /orders arrives.
```

```
Why is csrf().disable() set in SecurityConfig?
```

```
What does this @Aspect actually do?
```

```
How is the database connection pool configured?
```

```
Where would I add a new REST endpoint?
```

## What an answer looks like

- **Direct answer** in 1–3 sentences.
- **File:line citations** for every claim, clickable in IDEs and GitHub.
- **A real code snippet** (3–10 lines) pasted when it makes the answer concrete. Never invented.
- **A Mermaid sequence diagram** when the flow crosses multiple files; rendered in GitHub / IntelliJ / VS Code.
- **One concrete follow-up** offered, not a menu (*"Want me to check how this is tested?"*).
- **Uncertainty stated plainly** when something isn't in the code (*"I don't see where Y is wired up — likely either in a profile-specific config I haven't loaded, or convention-based. Want me to dig?"*).

## Follow-up rounds

Most real sessions have 2–6 follow-up questions building on the same investigation. The plugin caches what it's already learned about the project, so you can pivot to a new area or drill deeper without restarting.

## What's inside

```
ask-java-codebase/
├── .claude-plugin/plugin.json
└── skills/ask-java-codebase/
    ├── SKILL.md                       ← Q&A workflow + Java traps + routing
    └── references/
        └── java-analysis-guide.md     ← Java framework + Lombok + reactive guide
```

The plugin loads sections of `java-analysis-guide.md` on demand based on the question shape (framework detection / control flow / data layer / security / testing / design patterns / anti-patterns). It does not pre-load the whole guide.

## Scope and limits

**Targets**: Java / JVM codebases — Spring Boot, Spring WebFlux, Quarkus, Micronaut, Jakarta EE. Maven or Gradle.

**Does not**:
- Generate report files. Answers go in chat. (For artifacts, use `onboard-java`.)
- Modify or write code. This skill is read-only; if you want code changes, ask Claude directly.
- Audit security. It answers *"how is security set up?"*, not *"is the setup good?"*.
- Deep-analyze non-JVM modules in polyglot repos.

**Honest trade-offs**:
- Without a persistent ledger, complex multi-session investigations lose state. If you need that, use `onboard-java` instead.
- Investigation depth scales with the question's specificity. Very broad questions (*"explain the whole codebase"*) will get a high-level answer and a follow-up offering to narrow.

## Version

v1.0.0 — initial release. Shares `java-analysis-guide.md` (verbatim) with the sibling `onboard-java` plugin. Parallel to the `ask-node-codebase` plugin.

## Author

Mahesh Yadav
