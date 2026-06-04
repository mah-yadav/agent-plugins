# Data Layer: JPA, Repositories, Transactions, Caching, Pooling, NoSQL

Load this when the question is about persistence — entities, repositories, queries, transaction boundaries, caching, the connection pool, multiple datasources, or NoSQL stores.

For reactive data access (R2DBC / reactive Mongo/Redis/Cassandra) indicators, see [foundation.md](./foundation.md).

## Spring Data JPA (detection)

**Detection**: `spring-boot-starter-data-jpa` dependency, `JpaRepository` / `CrudRepository` interfaces.

**Analysis points**:
- Entity classes: `@Entity`, `@Table`, `@Column`, `@Id`, `@GeneratedValue`.
- Relationships: `@OneToMany`, `@ManyToOne`, `@ManyToMany` — note fetch types.
- Repository methods: derived queries vs. `@Query` (JPQL/native SQL).
- `@Transactional` boundaries.
- Flyway (`db/migration/V*.sql`) or Liquibase (`changelog*.xml/yaml`).

## JPA Entities

- Map entity relationships: `@OneToMany`, `@ManyToOne`, `@ManyToMany` with cascade types and fetch strategies.
- Identify value objects vs. entities: `@Embeddable` classes, records as DTOs.
- Check inheritance strategies: `@Inheritance` with `SINGLE_TABLE`, `JOINED`, `TABLE_PER_CLASS`.
- Look for `@Converter` / `AttributeConverter` for custom type mappings.
- Note `@EntityListeners` for audit fields (`@CreatedDate`, `@LastModifiedDate`).

## Repository Layer

- Derived query methods, `@Query` with JPQL/native SQL, Specifications.
- Custom repository implementations: `*RepositoryCustom` + `*RepositoryImpl`.
- Pagination: `Pageable`, `Page<T>`.
- Projections: interface-based, `@Value`, DTO projections.

## Transaction Management

- `@Transactional`: readOnly, propagation, isolation, rollback rules.
- `TransactionTemplate` for programmatic transactions.
- Transaction boundaries: service layer (correct) vs. repository/controller (often incorrect).

## Caching

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

## Connection Pooling

HikariCP (Spring Boot default): `spring.datasource.hikari.*`.
- Pool size: `maximum-pool-size`, `minimum-idle`.
- Timeouts: `connection-timeout`, `idle-timeout`, `max-lifetime`, `leak-detection-threshold`.

## Multi-Datasource

- Multiple `DataSource` `@Bean` definitions, `@Qualifier`, `@Primary`.
- `@EntityScan` + `@EnableJpaRepositories` with separate `basePackages` and `entityManagerFactoryRef`.
- `AbstractRoutingDataSource` for dynamic switching.

## NoSQL Data Stores

| Store         | Dependency                               | Annotations                  | Repository                |
| ------------- | ---------------------------------------- | ---------------------------- | ------------------------- |
| MongoDB       | `spring-boot-starter-data-mongodb`       | `@Document`, `@Field`, `@Id` | `MongoRepository`         |
| Elasticsearch | `spring-boot-starter-data-elasticsearch` | `@Document`, `@Field`        | `ElasticsearchRepository` |
| Redis (data)  | `spring-boot-starter-data-redis`         | `@RedisHash`, `@Indexed`     | `CrudRepository`          |
| Cassandra     | `spring-boot-starter-data-cassandra`     | `@Table`, `@PrimaryKey`      | `CassandraRepository`     |
| Neo4j         | `spring-boot-starter-data-neo4j`         | `@Node`, `@Relationship`     | `Neo4jRepository`         |
