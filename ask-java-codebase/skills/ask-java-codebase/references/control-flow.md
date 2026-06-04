# Control Flow: Request Lifecycle, Exceptions, Async/Reactive

Load this when the question is about how a request travels through the app, how errors propagate, or how concurrency/async work is handled.

For reactive (WebFlux/Mutiny) detection and the MVC-vs-reactive differences table, see [foundation.md](./foundation.md) — a reactive app's lifecycle differs from the servlet flow below.

## Request Lifecycle (Spring MVC)

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

> Reactive (WebFlux): there is no `DispatcherServlet` or servlet filter chain. Requests flow through `WebFilter`s and a `HandlerMapping` on Netty event-loop threads; security uses `SecurityWebFilterChain`. See [foundation.md](./foundation.md).

## Exception Flow

- `@ExceptionHandler` on controller methods — controller-scoped.
- `@ControllerAdvice` / `@RestControllerAdvice` — global.
- Custom exception hierarchies mapping to HTTP status codes.
- `ResponseStatusException` for inline status mapping.

> Reactive: `@ControllerAdvice` works for annotated routes; for `RouterFunction` routes, error handling is via `ErrorWebExceptionHandler`.

## Async and Concurrency

| Pattern             | Usage                         | What to Look For                                     |
| ------------------- | ----------------------------- | ---------------------------------------------------- |
| `@Async`            | Offload to another thread     | `@EnableAsync`, `CompletableFuture<T>` return        |
| `CompletableFuture` | Compose async operations      | `thenApply`, `thenCompose`, `exceptionally`, `allOf` |
| `@Scheduled`        | Cron/periodic tasks           | `@EnableScheduling`, cron expressions                |
| `Mono` / `Flux`     | Reactive streams              | `map`, `flatMap`, `subscribe`                        |
| Virtual Threads     | Lightweight concurrency (21+) | `Thread.ofVirtual()`, structured concurrency         |
