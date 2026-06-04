# Security Setup & Common Anti-Patterns

Load this when the question is about how security is configured/observed in the code, or to recognize common Java/Spring code smells.

This describes **how security is set up**, not whether it is good — this skill does not perform a security audit. For framework-level security wiring (`SecurityFilterChain` vs `SecurityWebFilterChain`, the reactive vs MVC DSL), see [foundation.md](./foundation.md).

## Security Considerations

Flag these when encountered:

- Missing `@Valid` / `@Validated` on controller inputs.
- String concatenation in `@Query(nativeQuery = true)` — SQL injection risk.
- Hardcoded credentials — passwords, API keys as string literals.
- Missing `@PreAuthorize` on service methods modifying data.
- Overly permissive CORS: `allowedOrigins("*")`.
- PII in log statements.

## Common Anti-Patterns

| Anti-Pattern                 | Indicators                                                                       |
| ---------------------------- | -------------------------------------------------------------------------------- |
| Anemic domain model          | Entities with only getters/setters, all logic in services                        |
| God class                    | Services with too many responsibilities, hundreds of lines                       |
| N+1 queries                  | Lazy-loaded collections accessed in loops without `@EntityGraph` or `JOIN FETCH` |
| Field injection              | `@Autowired` on fields instead of constructor injection                          |
| Catching generic exceptions  | `catch (Exception e)` swallowing specific exceptions                             |
| Transactional on wrong layer | `@Transactional` on repositories or controllers                                  |
