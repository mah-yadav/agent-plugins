# Java Deep Analysis (Load on Demand)

Deeper Java analysis techniques. Load this file in Standard/Deep mode, or whenever `area-explain-code` analyzes the data layer, design patterns, testing, or security in depth. For the always-load basics (project ID, framework detection, DI/Lombok, control flow), see [java-core-analysis.md](./java-core-analysis.md).

## Contents

5. Data Layer Analysis — JPA, repositories, transactions, caching, pooling, multi-datasource, NoSQL
6. Design Patterns to Identify
7. Modern Java Features
8. Testing Analysis — build plugins, coverage, unit/integration/contract/architecture/BDD tests
9. Security Considerations
10. Common Anti-Patterns

---

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
