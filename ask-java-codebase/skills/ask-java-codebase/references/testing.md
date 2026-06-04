# Testing Analysis

Load this when the question is about the test setup — how tests are run, what frameworks are used, integration/contract/architecture tests, or coverage.

## Test Build Plugins

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

> These commands are reference for answering "how are tests run here". This skill does not run them itself unless the user explicitly asks.

## Coverage (JaCoCo)

- Agent: `prepare-agent` goal attaches via `argLine`.
- Report: `target/site/jacoco/index.html`.
- Thresholds: `check` goal with `<rules>` → `<limit>` for INSTRUCTION, BRANCH, LINE counters.
- Enforcement: `<haltOnFailure>true</haltOnFailure>`.
- Exclusions: generated code, config classes, DTOs.

## Unit Tests

- **JUnit 5**: `@Test`, `@BeforeEach`, `@Nested`, `@ParameterizedTest`, `@ExtendWith`.
- **Mockito**: `@Mock`, `@InjectMocks`, `when().thenReturn()`, `verify()`, `ArgumentCaptor`.
- **AssertJ**: `assertThat(result).isEqualTo(expected)`.

## Integration Tests

- **Spring Boot Test**: `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, `@MockBean`.
- **MockMvc**: `mockMvc.perform(get("/api/...")).andExpect(status().isOk())`.
- **Testcontainers**: `@Testcontainers`, `@Container` — real Docker services.
- **WireMock**: Mock external HTTP services.
- **`@Sql`**: Load SQL scripts before tests.

## Contract Tests

**Spring Cloud Contract**: Contract DSL/YAML in `src/test/resources/contracts/`, auto-generated tests, stub JAR.
**Pact**: `@ExtendWith(PactConsumerTestExt.class)`, `@Pact`, `@PactTestFor`.

## Architecture Tests (ArchUnit)

- `ArchRuleDefinition`, `layeredArchitecture()`, `onionArchitecture()`.
- Layer rules, package rules, naming conventions, cycle detection.
- `FreezingArchRule` for legacy codebases.

## BDD with Cucumber

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

## Test Organization

- Mirror `main` package structure in `test`.
- Test utilities: custom assertions, fixtures, `@TestConfiguration`.
- Test profiles: `application-test.yml`.
