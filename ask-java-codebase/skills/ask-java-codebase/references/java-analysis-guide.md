# Java Code Analysis Guide

Detailed techniques for analyzing and explaining Java code bases. This guide is split into two sections:

- **Essential Analysis** (Sections 1–4): Always load. Covers project identification, framework detection, DI analysis, and control flow.
- **Extended Analysis** (Sections 5–10): Load on demand. Covers data layer depth, design patterns, modern Java features, testing depth, security considerations, and anti-patterns.

**Loading guidance for the ask-java-codebase skill**:
- Read lines 1 through the `=== END ESSENTIAL ANALYSIS ===` marker for Quick mode.
- Read the full file for Standard/Deep mode or when analyzing data layer, testing, or design patterns.

---

# =================================================================
# === ESSENTIAL ANALYSIS (Always Load) ============================
# =================================================================

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

# =================================================================
# === END ESSENTIAL ANALYSIS ======================================
# =================================================================

# =================================================================
# === EXTENDED ANALYSIS (Load on Demand) ==========================
# =================================================================

## 5. Data Layer Analysis

### JPA Entities

- Map entity relationships: `@OneToMany`, `@ManyToOne`, `@ManyToMany` with cascade types and fetch strategies.
- Identify value objects vs. entities: `@Embeddable` classes, records as DTOs.
- Check inheritance strategies: `@Inheritance` with `SINGLE_TABLE`, `JOINED`, `TABLE_PER_CLASS`.
- Look for `@Converter` / `AttributeConverter` for custom type mappings.
- Note `@EntityListeners` for audit fields (`@CreatedDate`, `@LastModifiedDate`).

### Repository Layer

- Derived query methods, `@Query` with JPQL/native SQL, Specifications.
- Custom repository implementations: `*RepositoryCustom` + `*RepositoryImpl`.
- Pagination: `Pageable`, `Page<T>`.
- Projections: interface-based, `@Value`, DTO projections.

### Transaction Management

- `@Transactional`: readOnly, propagation, isolation, rollback rules.
- `TransactionTemplate` for programmatic transactions.
- Transaction boundaries: service layer (correct) vs. repository/controller (often incorrect).

### Caching

**Detection**: `spring-boot-starter-cache`, `@EnableCaching`, `@Cacheable`.

| Annotation    | Purpose                              |
| ------------- | ------------------------------------ |
| `@Cacheable`  | Cache method results                 |
| `@CacheEvict` | Remove entries from cache            |
| `@CachePut`   | Update cache without skipping method |
| `@Caching`    | Combine multiple cache operations    |

- **Provider**: `caffeine` (in-process), `spring-data-redis` (distributed), `ehcache`, `hazelcast`.
- **Config**: `spring.cache.type`, `CacheManager` bean, TTL, max size, eviction policy.
- **Key strategy**: default vs. `@Cacheable(key = "#id")` SpEL vs. custom `KeyGenerator`.

### Connection Pooling

HikariCP (Spring Boot default): `spring.datasource.hikari.*`.
- Pool size: `maximum-pool-size`, `minimum-idle`.
- Timeouts: `connection-timeout`, `idle-timeout`, `max-lifetime`, `leak-detection-threshold`.

### Multi-Datasource

- Multiple `DataSource` `@Bean` definitions, `@Qualifier`, `@Primary`.
- `@EntityScan` + `@EnableJpaRepositories` with separate `basePackages` and `entityManagerFactoryRef`.
- `AbstractRoutingDataSource` for dynamic switching.

### NoSQL Data Stores

| Store         | Dependency                               | Annotations                  | Repository                |
| ------------- | ---------------------------------------- | ---------------------------- | ------------------------- |
| MongoDB       | `spring-boot-starter-data-mongodb`       | `@Document`, `@Field`, `@Id` | `MongoRepository`         |
| Elasticsearch | `spring-boot-starter-data-elasticsearch` | `@Document`, `@Field`        | `ElasticsearchRepository` |
| Redis (data)  | `spring-boot-starter-data-redis`         | `@RedisHash`, `@Indexed`     | `CrudRepository`          |
| Cassandra     | `spring-boot-starter-data-cassandra`     | `@Table`, `@PrimaryKey`      | `CassandraRepository`     |
| Neo4j         | `spring-boot-starter-data-neo4j`         | `@Node`, `@Relationship`     | `Neo4jRepository`         |

---

## 6. Design Patterns to Identify

| Pattern             | Java Indicators                                                                    |
| ------------------- | ---------------------------------------------------------------------------------- |
| **Builder**         | Lombok `@Builder`, manual builder classes, fluent APIs                             |
| **Factory**         | Static factory methods, `@Bean` methods, `FactoryBean<T>`                          |
| **Strategy**        | Interface with multiple implementations, selected by `@Qualifier` or runtime logic |
| **Template Method** | Abstract classes with `abstract` methods                                           |
| **Observer**        | `ApplicationEvent` + `@EventListener`, `ApplicationEventPublisher`                 |
| **Decorator**       | Wrapper classes implementing same interface, delegating to wrapped instance        |
| **Proxy**           | Spring AOP, `@Transactional`, `@Async`                                             |
| **Repository**      | Spring Data repositories, DAO pattern                                              |
| **Adapter**         | Classes converting between external/internal models, `@Converter`                  |
| **Specification**   | Spring Data `Specification<T>` for dynamic queries                                 |

---

## 7. Modern Java Features

| Feature          | Java Version | What to Note                                       |
| ---------------- | ------------ | -------------------------------------------------- |
| `var`            | 10+          | Local variable type inference                      |
| `Optional`       | 8+           | Return types, `map`/`flatMap`/`orElseThrow` chains |
| Streams API      | 8+           | `stream()`, `map`, `filter`, `collect`, `reduce`   |
| Records          | 14+          | Immutable data carriers for DTOs, value objects    |
| Sealed classes   | 17+          | Restricted type hierarchies                        |
| Pattern matching | 16+          | `instanceof` with binding, switch expressions      |
| Text blocks      | 15+          | Multi-line strings with `"""`                      |
| Virtual threads  | 21+          | `Thread.ofVirtual()`, structured concurrency       |

---

## 8. Testing Analysis

### Test Build Plugins

**Maven Surefire** (unit tests):
- Runs during `mvn test`. Includes: `**/Test*.java`, `**/*Test.java`, `**/*Tests.java`.
- Config: `<includes>`, `<excludes>`, `<argLine>`, `<systemPropertyVariables>`.

**Maven Failsafe** (integration tests):
- Runs during `mvn verify`. Includes: `**/IT*.java`, `**/*IT.java`, `**/*ITCase.java`.
- Must be explicitly configured.

**Test commands**:

| What              | Maven                             | Gradle                                     |
| ----------------- | --------------------------------- | ------------------------------------------ |
| Unit tests only   | `mvn test`                        | `./gradlew test`                           |
| Integration tests | `mvn verify`                      | `./gradlew integrationTest`                |
| Skip tests        | `mvn install -DskipTests`         | `./gradlew build -x test`                  |
| Single test class | `mvn test -Dtest=MyTest`          | `./gradlew test --tests MyTest`            |
| Single method     | `mvn test -Dtest=MyTest#myMethod` | `./gradlew test --tests "MyTest.myMethod"` |

### Coverage (JaCoCo)

- Agent: `prepare-agent` goal attaches via `argLine`.
- Report: `target/site/jacoco/index.html`.
- Thresholds: `check` goal with `<rules>` → `<limit>` for INSTRUCTION, BRANCH, LINE counters.
- Enforcement: `<haltOnFailure>true</haltOnFailure>`.
- Exclusions: generated code, config classes, DTOs.

### Unit Tests

- **JUnit 5**: `@Test`, `@BeforeEach`, `@Nested`, `@ParameterizedTest`, `@ExtendWith`.
- **Mockito**: `@Mock`, `@InjectMocks`, `when().thenReturn()`, `verify()`, `ArgumentCaptor`.
- **AssertJ**: `assertThat(result).isEqualTo(expected)`.

### Integration Tests

- **Spring Boot Test**: `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, `@MockBean`.
- **MockMvc**: `mockMvc.perform(get("/api/...")).andExpect(status().isOk())`.
- **Testcontainers**: `@Testcontainers`, `@Container` — real Docker services.
- **WireMock**: Mock external HTTP services.
- **`@Sql`**: Load SQL scripts before tests.

### Contract Tests

**Spring Cloud Contract**: Contract DSL/YAML in `src/test/resources/contracts/`, auto-generated tests, stub JAR.
**Pact**: `@ExtendWith(PactConsumerTestExt.class)`, `@Pact`, `@PactTestFor`.

### Architecture Tests (ArchUnit)

- `ArchRuleDefinition`, `layeredArchitecture()`, `onionArchitecture()`.
- Layer rules, package rules, naming conventions, cycle detection.
- `FreezingArchRule` for legacy codebases.

### BDD with Cucumber

**Detection**: `cucumber-java`, `cucumber-spring` dependencies; `.feature` files.

- Feature files in `src/test/resources/features/`.
- Step definitions: `@Given`, `@When`, `@Then` in Java classes.
- Spring integration: `@CucumberContextConfiguration` + `@SpringBootTest`.
- Config: `@Suite` + `@ConfigurationParameter` or `@CucumberOptions`.

**BDD checklist**:
1. Map feature files to business domains.
2. Trace Gherkin steps to step definitions.
3. Check for unimplemented steps.
4. Verify Spring context bootstrap.
5. Check reporting plugins.

### Test Organization

- Mirror `main` package structure in `test`.
- Test utilities: custom assertions, fixtures, `@TestConfiguration`.
- Test profiles: `application-test.yml`.

---

## 9. Security Considerations

Flag these when encountered:

- Missing `@Valid` / `@Validated` on controller inputs.
- String concatenation in `@Query(nativeQuery = true)` — SQL injection risk.
- Hardcoded credentials — passwords, API keys as string literals.
- Missing `@PreAuthorize` on service methods modifying data.
- Overly permissive CORS: `allowedOrigins("*")`.
- PII in log statements.

---

## 10. Common Anti-Patterns

| Anti-Pattern                 | Indicators                                                                       |
| ---------------------------- | -------------------------------------------------------------------------------- |
| Anemic domain model          | Entities with only getters/setters, all logic in services                        |
| God class                    | Services with too many responsibilities, hundreds of lines                       |
| N+1 queries                  | Lazy-loaded collections accessed in loops without `@EntityGraph` or `JOIN FETCH` |
| Field injection              | `@Autowired` on fields instead of constructor injection                          |
| Catching generic exceptions  | `catch (Exception e)` swallowing specific exceptions                             |
| Transactional on wrong layer | `@Transactional` on repositories or controllers                                  |

# =================================================================
# === END EXTENDED ANALYSIS =======================================
# =================================================================
