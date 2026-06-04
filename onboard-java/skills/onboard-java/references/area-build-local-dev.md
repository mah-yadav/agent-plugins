# Area: Build & Local Development

Reference for Phase 3.1 of the onboarding workflow. Loaded by the orchestrator when analyzing the build-and-local-dev area.

**Goal**: Produce a practical local development guide — everything a developer needs to clone, build, run, and develop against the project on their machine.

**Core principle**: From zero to running in one document. The output should let a developer go from a fresh clone to a running application with passing tests, without asking anyone for help.

## Contents

- Step 1: Identify the Build System
- Step 2: Mine Build Configuration (2.1 Build Commands, 2.2 Docker, 2.3 Environment Config, 2.4 Local Services, 2.5 IDE Tooling, 2.6 Scripts)
- Step 3: Verify (OFF by default)
- Step 4: Generate Local Dev Guide
- Ledger update, exploration guidelines

## Priority Tiers (within this area)

| Step | Priority | Rationale |
|---|---|---|
| 2.1 Build Commands & Plugins | **P0** | Can't do anything without building |
| 2.2 Docker & Container Setup | **P0** | Most projects need local services |
| 2.3 Environment Configuration | **P0** | App won't start without config |
| 2.4 Local Services & Dependencies | **P0** | App needs its dependencies running |
| 2.5 IDE & Developer Tooling | **P1** | Important but not day-1 blocker |
| 2.6 Scripts & Automation | **P1** | Helpful but not essential |

**Quick mode**: Complete Step 1, Steps 2.1–2.4, and Step 4 (generate report). Skip 2.5, 2.6, and Step 3 (verify).

## Ledger Read-Before

Read the ledger first. If the build file (`pom.xml` / `build.gradle`) is already in `Files Read`, use those findings — don't re-read. Pull the `Module Source Paths` and `Generated-code Paths` to scope every search correctly.

## Step 1: Identify the Build System — P0

If the orchestrator's Phase 1 already read the build file, skip to extracting build-specific details not yet captured.

1. **Find the project root**: `pom.xml`, `build.gradle`, or `build.gradle.kts`.
2. **Read the project manifest** (if not already in the ledger):
   - **Maven**: `pom.xml` — parent POM, modules, Java version, Spring Boot version, plugins.
   - **Gradle**: `build.gradle` / `build.gradle.kts` — plugins, dependencies, Java toolchain.
3. **Read the README** (if not already in the ledger) — look for setup instructions.
4. **Map top-level structure**: Note `scripts/`, `docker/`, `config/`, `.env*` files.

## Step 2: Mine Build Configuration — P0/P1

### 2.1 Build Commands & Plugins — P0

| What to Find | Where to Look |
|---|---|
| Build commands | `pom.xml` plugins, `build.gradle` tasks, `Makefile`, `scripts/`, README |
| Maven profiles | `<profiles>` in `pom.xml` |
| Gradle tasks | Custom tasks in `build.gradle` |
| Multi-module build | Root `pom.xml` `<modules>`, `settings.gradle` `include` |
| Wrapper scripts | `mvnw` / `gradlew` presence |
| Plugin configuration | Surefire, Failsafe, Spring Boot plugin, JaCoCo |
| Code generation | Lombok, MapStruct, annotation processors, protobuf |

**Extract**: Exact build commands (clean build, skip tests, specific module, specific profile), wrapper availability, build order for multi-module.

### 2.2 Docker & Container Setup — P0

| What to Find | Where to Look |
|---|---|
| Docker Compose | `docker-compose.yml`, `docker-compose.override.yml`, `compose.yml` |
| Application Dockerfile | `Dockerfile`, `docker/Dockerfile` |
| Container services | Services in compose — databases, brokers, caches, mocks |
| Port mappings | Compose port configurations |
| Health checks | Docker healthcheck configuration |

**Extract**: Start/stop commands, port mappings per service, data persistence behavior.

### 2.3 Environment Configuration — P0

| What to Find | Where to Look |
|---|---|
| Application config | `application.yml`, `application.properties` |
| Profile-specific config | `application-{profile}.yml` |
| Environment variable placeholders | `${VAR_NAME}` or `${VAR_NAME:default}` patterns |
| Environment templates | `.env.example`, `.env.template` |
| Feature flags | `@ConditionalOnProperty` |

**Extract**: Every environment variable with purpose and defaults, which profile for local dev, how to override config locally.

### 2.4 Local Services & Dependencies — P0

| What to Find | Where to Look |
|---|---|
| Database | JDBC URL in config, Docker Compose service |
| Message broker | Kafka/RabbitMQ in config and Compose |
| Cache | Redis/Memcached in config and Compose |
| Mock services | WireMock, stub services in compose |
| Seed data | Migration scripts, seed data scripts |

**Extract**: Every required service, how to start it, connection details, seed data availability.

### 2.5 IDE & Developer Tooling — P1

| What to Find | Where to Look |
|---|---|
| EditorConfig | `.editorconfig` |
| IDE settings | `.idea/`, `.vscode/` |
| Annotation processing | Lombok, MapStruct — IDE plugin requirements |
| Hot reload | Spring DevTools, JRebel |
| Debug configuration | Run/debug configs |

**Extract**: Required IDE plugins, format-on-save setup, hot reload availability, debug configs.

### 2.6 Scripts & Automation — P1

| What to Find | Where to Look |
|---|---|
| Setup scripts | `scripts/setup.sh`, `Makefile` targets |
| Database scripts | `scripts/db/`, seed/reset scripts |
| Git hooks | `.husky/`, `pre-commit-config.yaml` |
| Utility scripts | Helper scripts for common tasks |

**Extract**: Available scripts/targets, first-time setup script, git hooks.

## Step 3: Verify (OFF by default) — P2

**Default: SKIP.** Onboarding is a read-only mining exercise. Only run verify when the user has explicitly told the orchestrator to verify, and even then re-confirm each command before running it. Prior approval of "verify in general" is not approval of any specific command.

If you skip verify, write the documented commands into the guide with a note: *"Commands documented but not executed. Run them yourself to confirm."*

When verifying:
1. Run the build command and confirm success.
2. Check Docker Compose services start.
3. Verify the app starts and health endpoint responds.
4. Note issues and fixes.

**Stop conditions** (abort verify and document what happened):
- A command takes > 10 minutes — likely needs flags or a VPN you don't have.
- A command needs credentials/secrets you don't have access to.
- A command would push to a remote registry or shared resource.

## Step 4: Generate Local Dev Guide

Write the document using the [build & local dev template](../assets/build-local-dev-template.md).

**Quick mode**: Fill only Prerequisites, Quick Start, Environment Variables (required only), Local Services, Common Tasks (build/run/test), Troubleshooting. Mark other sections `[Not analyzed — out of scope for Quick mode]`.

**Standard/Deep mode**: Fill all sections.

Default location: `<output-dir>/build-local-dev-guide.md` (Standard/Deep only — Quick mode rolls into the main report).

## Ledger Update After

Mark area 3.1 complete. Add:
- Build commands discovered
- Profiles and their purposes
- Environment variables
- Local services and ports
- Docker Compose details
- Files read during this step

## Exploration Guidelines

- **Read config files in full** — they're small and information-dense.
- **Follow the evidence chain**: DevTools dependency → check hot-reload config. Docker Compose → check service ports.
- **Check for defaults**: `${VAR_NAME:defaultValue}` — record the default, developer may not need to set it.
- **Note the absence of expected things**: No `.env.example`? No README setup section? Flag these gaps.
