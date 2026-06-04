# Node Integrations: GraphQL, gRPC, real-time, message queues

Non-REST transports and out-of-process integrations. Load this for "where are
the GraphQL resolvers", "how does gRPC work here", "where are the Kafka
consumers / job queues / websockets". For the HTTP request frameworks
(Express/Fastify/NestJS/etc.) see **web-frameworks.md**.

## GraphQL servers

**Detection**: One of:

| Library | Indicates |
|---|---|
| `apollo-server` / `@apollo/server` | Apollo Server (most common) |
| `graphql-yoga` | Yoga (lighter, edge-compatible) |
| `mercurius` | Mercurius (Fastify-native) |
| `@nestjs/graphql` | NestJS GraphQL module |
| `type-graphql` | Decorator-driven schema-first |
| `pothos` | Code-first, plugin-rich |

**Analysis**:
- Schema location: `.graphql` files, or inline as template literals.
- Resolvers: search for `resolvers = {...}` object or `@Resolver`/`@Query`/`@Mutation` decorators.
- DataLoader (`dataloader` package) — solves N+1; flag absence.
- Subscriptions: WebSocket transport (`graphql-ws`, legacy `subscriptions-transport-ws`).

## gRPC

**Detection**: `@grpc/grpc-js` (modern) or `grpc` (legacy, deprecated). `.proto` files.

- Server: `server.addService(definition, handlers)`.
- Client: generated stub from proto.
- NestJS has `@nestjs/microservices` for gRPC support.

## WebSockets / Real-time

| Library | Style |
|---|---|
| `ws` | Raw WebSocket library |
| `socket.io` | Event-based, with rooms, automatic reconnection |
| `@socket.io/admin-ui` | Admin dashboard for socket.io |
| `bullmq` / `bull` | Job queues (Redis-backed); not real-time but often in real-time apps |

## Message queues / brokers

| Library | What |
|---|---|
| `kafkajs` | Kafka client — search for `.consumer({...})`, `.producer()` |
| `amqplib` | RabbitMQ low-level client |
| `bullmq` | Redis-backed job queue (most common) |
| `agenda` | MongoDB-backed scheduler |
| `node-rdkafka` | Native Kafka binding (fewer projects) |
| `aws-sdk` SQS client | AWS SQS |
| `@google-cloud/pubsub` | GCP Pub/Sub |
