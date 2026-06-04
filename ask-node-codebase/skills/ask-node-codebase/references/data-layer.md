# Node Data Layer: ORMs, validation, caching, pooling

Load this for "how does it talk to the database", ORM/query-builder questions,
schema/validation, caching, and connection pooling.

**Contents** — ORM/query-builder detection · Prisma/Drizzle/TypeORM deep dives · migration tooling · validation libraries · caching · connection pooling

---

## ORM / Query Builder Detection

| Library | Style | Detection |
|---|---|---|
| **Prisma** | Schema-first ORM, type-safe client | `prisma/schema.prisma`, `@prisma/client` |
| **Drizzle** | TS-first query builder, schema as code | `drizzle-orm`, `drizzle.config.{ts,js}` |
| **TypeORM** | Decorator-based ORM (Java-like) | `typeorm`, `@Entity()` decorators |
| **MikroORM** | Decorator-based, Unit-of-Work pattern | `@mikro-orm/core` |
| **Sequelize** | Class-based, mature | `sequelize`, `Model.init({...})` |
| **Knex** | Query builder only (no ORM) | `knex`, raw query builder |
| **Mongoose** | MongoDB ODM | `mongoose`, `Schema({...})`, `model('Name', schema)` |
| **Kysely** | TS-first query builder, no ORM | `kysely`, type-safe SQL |
| **node-postgres** (`pg`) | Raw driver, parameterized queries | `pg` |

## Prisma deep dive

- **Schema**: `prisma/schema.prisma` defines models, relations, datasource, generator.
- **Migrations**: `prisma/migrations/<timestamp>_name/migration.sql`.
- **Client**: Generated to `node_modules/.prisma/client`. Imported as `import { PrismaClient } from '@prisma/client'`.
- **Common anti-patterns**: Creating a new `PrismaClient` per request (should be singleton); N+1 from forgetting `include` or `select`.

## Drizzle deep dive

- **Schema files**: `src/db/schema.ts` (convention) — tables defined with `pgTable`, `mysqlTable`, `sqliteTable`.
- **Migrations**: `drizzle-kit` generates SQL migrations.
- **Queries**: Two styles — query builder (`db.select().from(users).where(...)`) and relational queries (`db.query.users.findMany({...})`).

## TypeORM deep dive

- **Entities**: `@Entity()` classes with `@PrimaryGeneratedColumn`, `@Column`, `@OneToMany`, `@ManyToOne`, `@JoinColumn`.
- **Repository pattern** or **Active Record**: Check which is in use.
- **Migrations**: CLI-generated; `migrations/` directory.
- **DataSource**: `new DataSource({...})` configures the connection — find this to know DB type.

## Migration tooling

| Tool | Indicator |
|---|---|
| Prisma Migrate | `prisma/migrations/` |
| Drizzle Kit | `drizzle/` folder + `drizzle.config` |
| TypeORM CLI | `migrations/` + `data-source.ts` |
| Knex Migrate | `migrations/` with `up`/`down` exports |
| Umzug | `umzug` package |
| Custom SQL files | `migrations/*.sql` with no tooling — manual application |

## Validation libraries (often serve as runtime schema)

| Library | Style |
|---|---|
| **Zod** | TS-first, infer types from schema (`z.infer<typeof schema>`) |
| **Yup** | Older, also TS support |
| **Joi** | Hapi-affiliated, mature |
| **AJV** | JSON Schema validator (Fastify default) |
| **class-validator** | Decorator-driven (NestJS default) |
| **valibot** | Tree-shakeable Zod alternative |
| **arktype** | TS syntax as schema |

**Important**: In TypeScript codebases that use Zod, the Zod schema is often the **source of truth for both types and runtime validation**. The TS types are derived from the schema (`z.infer<...>`), not the other way around.

## Caching

| Pattern | Indicator |
|---|---|
| In-memory LRU | `lru-cache`, `node-lru-cache` |
| Redis client | `ioredis` (most common, supports cluster), `redis` (official), `node-redis` |
| Memcached | `memjs`, `memcached` |
| NestJS cache | `@nestjs/cache-manager` + `cache-manager` |
| Edge caching | Cloudflare KV (`@cloudflare/workers-types`), Vercel KV |

## Connection pooling

- **`pg` Pool**: `new Pool({...})` — set `max`, `idleTimeoutMillis`.
- **Prisma**: Manages pool internally; configure via `DATABASE_URL` (`?connection_limit=N`).
- **Drizzle / Kysely**: Built on top of `pg`/`mysql2` — pool is configured at the driver level.
- **Serverless gotcha**: In Lambda/Vercel/Workers, connection pools are *per-instance* and can exhaust the DB. Look for connection-pooling proxies (PgBouncer, Prisma Accelerate, Neon's serverless driver).
