# Local Development Guide: [Project Name]

**Date**: [Date]
**Project**: [groupId:artifactId or project name]
**Build tool**: [Maven / Gradle]
**Java version**: [e.g., 17]
**Framework**: [Spring Boot X.Y / Micronaut / Quarkus]

---

## 1. Prerequisites

| Requirement    | Version                    | Install                           | Verify                       |
| -------------- | -------------------------- | --------------------------------- | ---------------------------- |
| Java (JDK)     | [e.g., 17+]                | [sdkman, brew, apt]               | `java -version`              |
| Maven/Gradle   | [version or "use wrapper"] | [brew, sdkman, or N/A if wrapper] | `./mvnw -v` / `./gradlew -v` |
| Docker         | [e.g., 20+]                | [docker.com]                      | `docker --version`           |
| Docker Compose | [e.g., v2+]                | [bundled with Docker Desktop]     | `docker compose version`     |
| [IDE plugin]   | [e.g., Lombok plugin]      | [marketplace link]                |                              |

## 2. Quick Start

```bash
# 1. Clone the repository
git clone [repo-url]
cd [project-dir]

# 2. Start local services (databases, message brokers, etc.)
[docker compose up -d]

# 3. Build the project
[./mvnw clean install / ./gradlew build]

# 4. Run the application
[./mvnw spring-boot:run / ./gradlew bootRun / java -jar target/*.jar]

# 5. Verify it's running
[curl http://localhost:8080/actuator/health]
```

## 3. Environment Variables

### Required

| Variable | Purpose | Default | Where to Set |
| -------- | ------- | ------- | ------------ |
|          |         |         |              |

### Optional

| Variable | Purpose | Default | When to Set |
| -------- | ------- | ------- | ----------- |
|          |         |         |             |

### Setting Environment Variables

```bash
# Option 1: Export in terminal
export VAR_NAME=value

# Option 2: Create a local .env file (if supported)
cp .env.example .env

# Option 3: Pass via command line
[./mvnw spring-boot:run -Dspring-boot.run.arguments="--var.name=value"]
```

## 4. Build Profiles

| Profile      | Purpose                     | Command                            |
| ------------ | --------------------------- | ---------------------------------- |
| [default]    | [Standard build with tests] | `./mvnw clean install`             |
| [local]      | [Local development config]  | `./mvnw spring-boot:run -Plocal`   |
| [skip-tests] | [Fast build, no tests]      | `./mvnw clean install -DskipTests` |

## 5. Local Services

### Starting All Services

```bash
docker compose up -d
```

### Individual Services

| Service      | Purpose            | Port   | Credentials              | Health Check        |
| ------------ | ------------------ | ------ | ------------------------ | ------------------- |
| [PostgreSQL] | [Primary database] | [5432] | [user/pass from compose] | `docker compose ps` |
| [Redis]      | [Cache]            | [6379] | [none]                   |                     |
| [Kafka]      | [Messaging]        | [9092] | [none]                   |                     |

### Stopping Services

```bash
# Stop but keep data
docker compose stop

# Stop and remove data
docker compose down -v
```

### Seed Data

```bash
# [How to populate initial/test data, if applicable]
[command or script path]
```

## 6. Multi-Module Build (if applicable)

| Module | Purpose | Build Command |
| ------ | ------- | ------------- |
|        |         |               |

```bash
# Build a specific module only
[./mvnw -pl module-name clean install]

# Build a module and its dependencies
[./mvnw -pl module-name -am clean install]
```

## 7. Common Tasks

### Build

```bash
# Full build with tests
[./mvnw clean install]

# Fast build (skip tests)
[./mvnw clean install -DskipTests]

# Build a single module
[./mvnw -pl module-name clean install]
```

### Test

```bash
# Run all unit tests
[./mvnw test]

# Run integration tests
[./mvnw verify]

# Run a single test class
[./mvnw test -Dtest=MyTestClass]

# Run a single test method
[./mvnw test -Dtest=MyTestClass#myTestMethod]

# Generate coverage report
[./mvnw verify -Pjacoco]
```

### Run

```bash
# Run with default profile
[./mvnw spring-boot:run]

# Run with a specific profile
[./mvnw spring-boot:run -Dspring-boot.run.profiles=local]

# Run with debug port
[./mvnw spring-boot:run -Dspring-boot.run.jvmArguments="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"]
```

### Format & Lint

```bash
# Check code style
[./mvnw checkstyle:check]

# Auto-format
[./mvnw spotless:apply]
```

## 8. IDE Setup

### IntelliJ IDEA

1. Open the project: File → Open → select `[pom.xml / build.gradle]`
2. Import as [Maven / Gradle] project
3. Install required plugins:
   - [ ] [Lombok]
   - [ ] [MapStruct] (if used)
   - [ ] [EditorConfig]
4. Enable annotation processing: Settings → Build → Compiler → Annotation Processors → Enable
5. Set Java SDK: Settings → Project Structure → Project SDK → [Java version]

### VS Code

1. Install extensions:
   - [ ] Extension Pack for Java
   - [ ] Spring Boot Extension Pack (if Spring Boot)
   - [ ] Lombok Annotations Support
2. Open the project root folder
3. Java: Clean Java Language Server Workspace (if build issues)

### Hot Reload

[If Spring DevTools or JRebel is configured, explain how it works and any setup needed]

## 9. Scripts & Makefile

| Target / Script | What It Does | Command |
| --------------- | ------------ | ------- |
|                 |              |         |

## 10. Application Configuration Profiles

| Profile   | Config File             | Purpose                        | Activate With                    |
| --------- | ----------------------- | ------------------------------ | -------------------------------- |
| [default] | `application.yml`       | [Base config, shared settings] | [always active]                  |
| [local]   | `application-local.yml` | [Local development overrides]  | `--spring.profiles.active=local` |
| [test]    | `application-test.yml`  | [Test configuration]           | [auto-activated in tests]        |

## 11. Troubleshooting

| Problem                               | Cause                             | Fix                                                 |
| ------------------------------------- | --------------------------------- | --------------------------------------------------- |
| Build fails with "Java version" error | Wrong JDK version                 | Install JDK [version]: `sdk install java [version]` |
| "Connection refused" on startup       | Local services not running        | Run `docker compose up -d` first                    |
| Tests fail with "port already in use" | Previous test run didn't clean up | Kill process: `lsof -i :[port]` then `kill [pid]`   |
| Lombok not recognized in IDE          | Annotation processing disabled    | Enable in IDE settings (see IDE Setup)              |

## 12. Useful URLs (when running locally)

| URL                                       | What                       |
| ----------------------------------------- | -------------------------- |
| `http://localhost:[port]`                 | Application                |
| `http://localhost:[port]/actuator/health` | Health check               |
| `http://localhost:[port]/swagger-ui.html` | API docs (if available)    |
| `http://localhost:[port]/h2-console`      | H2 DB console (if H2 used) |

---

*Generated by the build-local-dev skill on [Date]*
