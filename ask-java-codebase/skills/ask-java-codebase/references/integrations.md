# Integrations: API Docs, Messaging, HTTP Clients, gRPC, GraphQL

Load this when the question is about how the service talks to the outside world — message brokers (Kafka/RabbitMQ/JMS), outbound HTTP calls, gRPC, GraphQL, or the API documentation/contract.

For the core web framework and DI, see [foundation.md](./foundation.md).

## API Documentation & Versioning

**Detection**: `springdoc-openapi-starter-webmvc-ui` or `springfox-boot-starter` dependencies.

- **Springdoc OpenAPI** (modern): `@Operation`, `@ApiResponse`, `@Tag`, `@Schema` annotations.
- **Springfox** (legacy): `@Api`, `@ApiOperation`, `@ApiModel` annotations.
- **Swagger UI**: `/swagger-ui.html` (Springfox) or `/swagger-ui/index.html` (Springdoc).
- **API versioning**: URL-based (`/api/v1/`), header-based, or media type.

**Contract-first note**: If `openapi-generator-*` is in the build file, the OpenAPI YAML/JSON spec is the source of truth — don't trace generated controllers in `target/generated-sources/`; trace the spec and the hand-written delegate/implementation classes.

## Messaging

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

## HTTP Clients

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

## gRPC

**Detection**: `grpc-spring-boot-starter` or `io.grpc` dependencies; `.proto` files.
- Proto definitions: `.proto` files define service contracts.
- Server: `*ImplBase` classes.
- Client: `@GrpcClient` or `ManagedChannel` usage.

## GraphQL

**Detection**: `spring-boot-starter-graphql`; `.graphqls` schema files.
- Schema: `*.graphqls` in `src/main/resources/graphql/`.
- Controllers: `@QueryMapping`, `@MutationMapping`, `@SchemaMapping`.
- N+1 handling: `@BatchMapping`.
