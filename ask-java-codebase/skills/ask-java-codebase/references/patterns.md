# Design Patterns & Modern Java Features

Load this when the question is about design patterns/idioms in the code, or which modern Java language features are in use.

## Design Patterns to Identify

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

## Modern Java Features

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
