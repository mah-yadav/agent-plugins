# Java Core Analysis (Always Load)

Core techniques for reading a Java codebase: project identification, framework detection (incl. reactive stacks), dependency injection / Lombok, and control flow. `area-explain-code` loads this file in **every** mode. For deeper topics (data layer, design patterns, modern Java, testing, security, anti-patterns), load the companion [java-deep-analysis.md](./java-deep-analysis.md) in Standard/Deep mode or when those topics need depth.

## Contents

1. Project Identification — build system, source layout
2. Framework Detection and Analysis — Spring MVC, WebFlux, Mutiny/Quarkus, JPA, API docs, messaging, HTTP clients, gRPC, GraphQL, Jakarta EE/Micronaut/Quarkus
3. Dependency Injection Analysis — injection styles, Lombok-aware reading
4. Control Flow Patterns — request lifecycle, exception flow, async/concurrency

---

## 1. Project Identification

Determine the project type, build system, and structure before diving into code.

| Indicator                                                            | What It Tells You                                                 |
| -------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `pom.xml`                                                            | Maven project — read for dependencies, plugins, modules, profiles |
| `build.gradle` / `build.gradle.kts`                                  | Gradle project — read for dependencies, tasks, plugins            |
| `src/main/java` + `src/test/java`                                    | Standard Maven/Gradle source layout                               |
| `application.properties` / `application.yml`                         | Spring Boot configuration                                         |
| `persistence.xml`                                                    | JPA configuration                                                 |
| `web.xml`                                                            | Traditional servlet-based deployment descriptor                   |
| `settings.gradle` / `settings.gradle.kts`                            | Multi-module Gradle project                                       |
| `openapi-generator-maven-plugin` / `openapi-generator-gradle-plugin` | Contract-first API: code generated from OpenAPI spec              |
| `*.yaml` / `*.json` in `src/main/resources/openapi/` or `api/`       | OpenAPI spec location — the source of truth for the API           |
| `target/generated-sources/openapi`                                   | Generated API stubs — do NOT modify these files                   |
| `quarkus.properties` / `application.properties` with `quarkus.*`     | Quarkus application                                               |
| `micronaut-cli.yml`                                                  | Micronaut application                                             |
| `mvnw` / `gradlew`                                                   | Wrapper scripts — project pins a specific build tool version      |

### Build System Analysis

**Maven** (`pom.xml`):
- Read `<dependencies>` to understand the tech stack.
- Check `<modules>` for multi-module structure.
- Look at `<profiles>` for environment-specific builds.
- Check `<plugins>` for code generation, static analysis, and packaging.
- Read `<parent>` for inherited configuration.

**Gradle** (`build.gradle` / `build.gradle.kts`):
- Read `dependencies {}` — note `implementation`, `api`, `compileOnly`, `runtimeOnly`, `testImplementation`.
- Check `plugins {}` for framework integration.
- Look at `subprojects {}` / `allprojects {}` for multi-module config.
- Check for custom tasks.

### Source Layout

Map the source tree to understand organizational conventions:

```
src/
├── main/
│   ├── java/com/example/project/
│   │   ├── config/          # Configuration classes
│   │   ├── controller/      # REST/MVC controllers
│   │   ├── service/         # Business logic
│   │   ├── repository/      # Data access
│   │   ├── model/           # Domain entities / DTOs
│   │   ├── dto/             # Data transfer objects (if separate)
│   │   ├── exception/       # Custom exceptions
│   │   ├── security/        # Security configuration
│   │   ├── util/            # Utility classes
│   │   └── Application.java # Entry point
│   └── resources/
│       ├── application.yml  # App configuration
│       ├── db/migration/    # Flyway/Liquibase migrations
│       └── templates/       # Thymeleaf/Freemarker templates
└── test/
    └── java/com/example/project/
```

---

## 2. Framework Detection and Analysis

### Spring Boot (most common)

**Detection**: `@SpringBootApplication` on main class, Spring Boot starter dependencies.

**Key annotations to trace**:

| Annotation                 | Layer     | Purpose                                                                 |
| -------------------------- | --------- | ----------------------------------------------------------------------- |
| `@SpringBootApplication`   | Bootstrap | Combines `@Configuration`, `@EnableAutoConfiguration`, `@ComponentScan` |
| `@RestController`          | Web       | REST API controller                                                     |
| `@Controller`              | Web       | MVC controller returning views                                          |
| `@Service`                 | Business  | Business logic layer bean                                               |
| `@Repository`              | Data      | Data access layer bean                                                  |
| `@Component`               | General   | Generic Spring-managed bean                                             |
| `@Configuration`           | Config    | Declares bean definitions                                               |
| `@ConfigurationProperties` | Config    | Type-safe configuration binding                                         |
| `@Profile`                 | Config    | Conditional bean activation by profile                                  |

**Spring Boot analysis checklist**:
1. Read `application.yml` / `application.properties` for all configuration.
2. Trace auto-configuration — which starters are included.
3. Map the bean dependency graph: `@Service` → `@Repository`, etc.
4. Check for custom `@Configuration` classes.
5. Look for `@Profile` annotations.
6. Check for `@Scheduled` tasks, `@EventListener`.
7. Trace `@EnableAsync`, `@EnableCaching`, `@EnableScheduling`.

**Spring Web / REST analysis**:
- Map all `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping` endpoints.
- Check `@Valid` / `@Validated` for request validation.
- Look for `@ControllerAdvice` / `@RestControllerAdvice` for centralized error handling.

### Spring WebFlux (Reactive)

**Detection**: `spring-boot-starter-webflux` dependency, `reactor-core`, `Mono`/`Flux` return types on controllers, `@EnableWebFlux`.

**Why it matters for analysis**: WebFlux is **not** a drop-in replacement for Spring MVC. The request lifecycle, security filter chain, error handling, and threading model all differ. Misreading a WebFlux app as MVC produces wrong sequence diagrams and wrong security analysis.

**Key differences from Spring MVC**:

| Topic | Spring MVC (servlet) | Spring WebFlux (reactive) |
|---|---|---|
| Server | Tomcat / Jetty (servlet) | Netty (default) / Undertow / servlet 3.1+ |
| Endpoint annotations | `@RestController` + `@GetMapping` | Same annotations, or `RouterFunction` + `HandlerFunction` (functional routing) |
| Return type | `T`, `ResponseEntity<T>` | `Mono<T>`, `Flux<T>`, `ResponseEntity<Mono<T>>` |
| Security filter chain bean | `SecurityFilterChain` | `SecurityWebFilterChain` |
| Security DSL | `HttpSecurity` | `ServerHttpSecurity` |
| User principal in controller | `Authentication` arg / `@AuthenticationPrincipal` | `@AuthenticationPrincipal` + `Mono<Principal>` or `ReactiveSecurityContextHolder.getContext()` |
| Database access | JPA / JDBC (blocking) | R2DBC, Spring Data Reactive (Mongo/Redis/Cassandra) |
| Transactions | `@Transactional` | `@Transactional` with `ReactiveTransactionManager` |
| Threading | Servlet thread per request | Event-loop threads — never block; use `publishOn(Schedulers.boundedElastic())` for blocking work |
| MDC / logging context | `MDC.put()` in filter | MDC does NOT propagate across operators — use `contextWrite` and Reactor `Context` |
| Error handling | `@ControllerAdvice` works | `@ControllerAdvice` works for annotated routes; for `RouterFunction`, use `ErrorWebExceptionHandler` |
| Testing | `MockMvc` | `WebTestClient` |

**Functional routing detection**: If `@RestController` is sparse but you see `@Bean RouterFunction<ServerResponse>` definitions returning `RouterFunctions.route(...)`, the API is defined functionally. The route definitions live in `@Configuration` classes, not controllers — search there for the endpoint map.

**Reactive data layer indicators**:

| Dependency | Indicates |
|---|---|
| `spring-boot-starter-data-r2dbc` | R2DBC reactive SQL |
| `spring-boot-starter-data-mongodb-reactive` | Reactive MongoDB |
| `spring-boot-starter-data-redis-reactive` | Reactive Redis |
| `spring-boot-starter-data-cassandra-reactive` | Reactive Cassandra |
| `r2dbc-postgresql` / `r2dbc-mysql` / `r2dbc-h2` | R2DBC driver — confirms reactive SQL |

**Reactive anti-patterns to flag**:
- Calling JPA / `RestTemplate` / `JdbcTemplate` (blocking) directly inside a `Mono`/`Flux` chain without `publishOn(Schedulers.boundedElastic())`.
- `.block()` on the event-loop thread.
- Subscribing manually (`.subscribe()`) and discarding the `Disposable` — leaks.
- Using `ThreadLocal` for request-scoped state.

### Mutiny / Quarkus Reactive

**Detection**: `io.smallrye.reactive:mutiny` or Quarkus reactive extensions; `Uni<T>` / `Multi<T>` return types.

- `Uni<T>` ≈ `Mono<T>` (0..1 emission). `Multi<T>` ≈ `Flux<T>` (0..N).
- Reactive routes: `quarkus-resteasy-reactive`, `quarkus-vertx-web`.
- Hibernate Reactive (`quarkus-hibernate-reactive-panache`) instead of classic Panache for reactive data access.

### Spring Data JPA

**Detection**: `spring-boot-starter-data-jpa` dependency, `JpaRepository` / `CrudRepository` interfaces.

**Analysis points**:
- Entity classes: `@Entity`, `@Table`, `@Column`, `@Id`, `@GeneratedValue`.
- Relationships: `@OneToMany`, `@ManyToOne`, `@ManyToMany` — note fetch types.
- Repository methods: derived queries vs. `@Query` (JPQL/native SQL).
- `@Transactional` boundaries.
- Flyway (`db/migration/V*.sql`) or Liquibase (`changelog*.xml/yaml`).

### API Documentation & Versioning

**Detection**: `springdoc-openapi-starter-webmvc-ui` or `springfox-boot-starter` dependencies.

- **Springdoc OpenAPI** (modern): `@Operation`, `@ApiResponse`, `@Tag`, `@Schema` annotations.
- **Springfox** (legacy): `@Api`, `@ApiOperation`, `@ApiModel` annotations.
- **Swagger UI**: `/swagger-ui.html` (Springfox) or `/swagger-ui/index.html` (Springdoc).
- **API versioning**: URL-based (`/api/v1/`), header-based, or media type.

### Messaging

**Detection**: `spring-kafka`, `spring-boot-starter-amqp`, `spring-jms` dependencies.

**Kafka analysis**:
- Consumers: `@KafkaListener` — topic names, group IDs, error handler.
- Producers: `KafkaTemplate` — topics produced to, message types.
- Config: `spring.kafka.*` properties.
- Error handling: `@RetryableTopic`, `DeadLetterPublishingRecoverer`.

**RabbitMQ analysis**:
- Consumers: `@RabbitListener` — queue names, acknowledgment modes.
- Producers: `RabbitTemplate` — exchange, routing key.
- Config: `spring.rabbitmq.*` properties, `Queue`/`Exchange`/`Binding` beans.

**Messaging checklist**:
1. List all topics/queues consumed and produced.
2. Identify serialization format (JSON, Avro, Protobuf).
3. Check error handling: retries, dead letter topics/queues.
4. Check for idempotency handling.
5. Check for transactional messaging.

### HTTP Clients

| Client         | Style                     | What to Look For                                                   |
| -------------- | ------------------------- | ------------------------------------------------------------------ |
| `RestTemplate` | Synchronous               | Bean config, interceptors, error handlers                          |
| `WebClient`    | Reactive                  | Filters, `retrieve()` vs `exchange()`, `onStatus()` error handling |
| `@FeignClient` | Declarative               | `@EnableFeignClients`, fallbacks, interceptors                     |
| `RestClient`   | Modern synchronous (6.1+) | Builder config, similar to WebClient API                           |

**HTTP client checklist**:
1. List all external services called — service name, base URL, operations.
2. Check retry logic: `@Retryable`, Resilience4j `@Retry`/`@CircuitBreaker`.
3. Check timeouts: connection, read timeout config.
4. Check authentication: token propagation, API keys.

### gRPC

**Detection**: `grpc-spring-boot-starter` or `io.grpc` dependencies; `.proto` files.
- Proto definitions: `.proto` files define service contracts.
- Server: `*ImplBase` classes.
- Client: `@GrpcClient` or `ManagedChannel` usage.

### GraphQL

**Detection**: `spring-boot-starter-graphql`; `.graphqls` schema files.
- Schema: `*.graphqls` in `src/main/resources/graphql/`.
- Controllers: `@QueryMapping`, `@MutationMapping`, `@SchemaMapping`.
- N+1 handling: `@BatchMapping`.

### Jakarta EE, Micronaut, Quarkus

**Jakarta EE**: `jakarta.*` / `javax.*` imports, `@Stateless`, `@Path`.
**Micronaut**: `@MicronautApplication`, compile-time DI, `@Client`.
**Quarkus**: `@QuarkusApplication`, `quarkus.properties`, Panache for data access.

---

## 3. Dependency Injection Analysis

1. **Injection style**: Constructor (preferred), field (`@Autowired`), or setter.
2. **Bean scoping**: `@Singleton` (default), `@Prototype`, `@RequestScope`.
3. **Bean lifecycle**: `@PostConstruct`, `@PreDestroy`.
4. **Qualifiers**: `@Qualifier`, `@Primary`.
5. **Conditional beans**: `@ConditionalOnProperty`, `@ConditionalOnMissingBean`, `@Profile`.
6. **Manual registration**: `@Bean` methods in `@Configuration` classes.

### Lombok-Aware Reading

**Detection**: `org.projectlombok:lombok` dependency, `lombok.config` file, `@Slf4j`/`@Data`/`@Builder`/`@RequiredArgsConstructor` imports.

When Lombok is present, the code you `Read` is incomplete — methods exist that are not visible. Infer them from annotations:

| Annotation | What's generated (invisible in source) | DI/analysis impact |
|---|---|---|
| `@RequiredArgsConstructor` | Constructor taking all `final` fields | This **is** the constructor injection — bean's dependencies are the `final` fields, in declaration order |
| `@AllArgsConstructor` | Constructor taking all fields | Same as above but all fields, not just `final` |
| `@NoArgsConstructor` | Default constructor | Often required by JPA `@Entity` |
| `@Data` | Getters + setters + `equals`/`hashCode`/`toString` + `@RequiredArgsConstructor` | Treat as mutable POJO; setters exist even if not in source |
| `@Value` | Final fields + getters + `@AllArgsConstructor` + `equals`/`hashCode`/`toString` | Immutable value object — no setters |
| `@Getter` / `@Setter` | Accessor methods | Called from other classes as `getFoo()`/`setFoo()` even though absent in source |
| `@Builder` | Static `builder()` method + inner `Builder` class | Search for `ClassName.builder()` usages |
| `@SuperBuilder` | Builder that handles inheritance | Inherited fields included in the builder |
| `@Slf4j` / `@Log4j2` / `@CommonsLog` | `private static final Logger log` field | `log.info(...)` calls reference this generated field |
| `@With` (records / classes) | `withFoo(value)` copy methods | Functional update pattern |
| `@SneakyThrows` | Wraps checked exceptions | Method may throw checked exceptions not in `throws` clause |
| `@Delegate` | Delegating method bodies | Class effectively implements delegated-type's methods |

**Practical checks when Lombok is in use**:
1. To find a bean's dependencies, look at `final` fields when `@RequiredArgsConstructor` is present — that's the constructor signature.
2. Do not flag "missing getters" or "missing constructor" as smells if a Lombok annotation generates them.
3. `lombok.config` at the project root can disable specific features (e.g., `lombok.equalsAndHashCode.callSuper=CALL`) — read it once before drawing conclusions.
4. Delombok'd output (`target/delombok/` or `build/delombok/`) shows the generated source — useful when you must see the final shape.

---

## 4. Control Flow Patterns

### Request Lifecycle (Spring MVC)

1. `DispatcherServlet` receives the request.
2. Filters execute (Security, CORS, etc.).
3. Handler mapping finds the `@Controller` method.
4. `HandlerInterceptor.preHandle()` runs.
5. Argument resolvers bind `@PathVariable`, `@RequestBody`, etc.
6. Validation runs if `@Valid` is present.
7. Controller → Service → Repository.
8. `HandlerInterceptor.postHandle()` runs.
9. Response serialization.
10. `HandlerInterceptor.afterCompletion()` runs.

### Exception Flow

- `@ExceptionHandler` on controller methods — controller-scoped.
- `@ControllerAdvice` / `@RestControllerAdvice` — global.
- Custom exception hierarchies mapping to HTTP status codes.
- `ResponseStatusException` for inline status mapping.

### Async and Concurrency

| Pattern             | Usage                         | What to Look For                                     |
| ------------------- | ----------------------------- | ---------------------------------------------------- |
| `@Async`            | Offload to another thread     | `@EnableAsync`, `CompletableFuture<T>` return        |
| `CompletableFuture` | Compose async operations      | `thenApply`, `thenCompose`, `exceptionally`, `allOf` |
| `@Scheduled`        | Cron/periodic tasks           | `@EnableScheduling`, cron expressions                |
| `Mono` / `Flux`     | Reactive streams              | `map`, `flatMap`, `subscribe`                        |
| Virtual Threads     | Lightweight concurrency (21+) | `Thread.ofVirtual()`, structured concurrency         |
