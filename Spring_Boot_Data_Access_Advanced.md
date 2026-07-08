# Spring Boot Data Access Advanced

## JPA and Hibernate Internal Architecture

Spring Data JPA builds on `EntityManager` and typically uses Hibernate as the provider. Key concepts include:

* persistence context
* entity state transitions (transient, managed, detached, removed)
* first-level cache
* dirty checking

### Persistence context

The persistence context is associated with a transaction. It caches managed entities and tracks changes. During `flush()`, Hibernate computes SQL for insert/update/delete operations and synchronizes the database state.

### Flush modes

* `AUTO` (default): flushes before matching queries and at commit
* `COMMIT`: flushes only at commit time

The JPA standard `FlushModeType` defines only `AUTO` and `COMMIT`. Hibernate's native `FlushMode` adds `MANUAL` (flush only on an explicit `flush()`) and `ALWAYS`.

### Dirty checking

Hibernate detects changed entity state by comparing snapshots captured at load or flush time, producing update statements only when necessary.

## Fetch Strategies and Lazy Loading

### Eager vs Lazy

* `FetchType.EAGER` loads associations immediately.
* `FetchType.LAZY` uses proxies or bytecode enhancement.

Lazy loading outside a transaction typically throws `LazyInitializationException`. Use DTO queries, `JOIN FETCH`, or Open Session in View carefully when necessary.

### N+1 problem

Occurs when lazy associations fire separate SQL queries. Avoid it using:

* `JOIN FETCH`
* `@EntityGraph`
* batch fetching settings
* query projections

```java
@Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.status = :status")
List<Order> findWithLines(@Param("status") OrderStatus status);
```

## Caching

### First-level cache

Automatic per persistence context. Repeated retrieval of the same entity returns the same managed instance.

### Second-level cache

Hibernate supports a shared cache across sessions. Providers include Ehcache, Hazelcast, Infinispan, and Redis.

### Query cache

Caches query result IDs, but execution still needs to fetch entities from the second-level cache or database.

## Batching and Performance

### JDBC batching

Enable with `spring.jpa.properties.hibernate.jdbc.batch_size`. Boot configures JDBC batching for inserts and updates through `PreparedStatement` batching.

### Statement caching

`PreparedStatement` caching lives at the JDBC driver / connection-pool level, not in Hibernate. With HikariCP and a driver such as PostgreSQL or MySQL, enable it through data-source properties like `spring.datasource.hikari.data-source-properties.cachePrepStmts=true`, `prepStmtCacheSize`, and `prepStmtCacheSqlLimit`.

### Fetch size

Set `spring.jpa.properties.hibernate.jdbc.fetch_size` for large result sets and streaming queries.

## Locking and Concurrency Control

### Optimistic locking

Use `@Version` for version-based concurrency control. Hibernate adds a version check to the update statement and increments the version upon success; a stale update throws `OptimisticLockException`.

```java
@Entity
public class Account {
    @Id
    private Long id;

    @Version
    private long version;

    private BigDecimal balance;
}
```

### Pessimistic locking

Use `@Lock(LockModeType.PESSIMISTIC_WRITE)` or `EntityManager.lock()`. The database enforces the lock semantics, and behavior depends on the JDBC driver.

## Projections and DTO Mapping

### Interface projections

Spring Data can create projections using interfaces or classes. Interface projections are mapped lazily and avoid entity hydration when supported.

### Class-based DTOs

Use constructor expressions in JPQL or map query results manually with `Tuple` or `Object[]`.

### Entity graphs

`@EntityGraph` defines fetch plans declaratively. The JPA provider uses the graph to eagerly load the named associations in a single query and avoid the N+1 problem.

```java
@EntityGraph(attributePaths = {"lines", "customer"})
@Query("SELECT o FROM Order o WHERE o.id = :id")
Optional<Order> findDetailById(@Param("id") Long id);
```

## Query Tuning Internals

### `@QueryHints`

Pass vendor-specific hints to the persistence provider to influence SQL generation, caching, or fetch strategy.

### Criteria API vs JPQL

The Criteria API builds type-safe queries programmatically. At runtime it produces JPQL and SQL equivalent to string-based queries.

## Native SQL and Stored Procedures

Use `@Query(nativeQuery = true)` for vendor-specific SQL. Native queries bypass Hibernate’s JPQL parser and require explicit result mappings. Use `@Procedure` for stored procedure calls.

## Transaction Context and Exception Translation

Spring translates persistence exceptions with `PersistenceExceptionTranslationPostProcessor`, converting provider-specific exceptions into Spring’s `DataAccessException` hierarchy.

## Advanced Patterns

### Read-write split

Use multiple `DataSource` beans and `AbstractRoutingDataSource` to route reads to replicas and writes to primary databases.

### Soft deletes

Implement soft deletes with a boolean flag and Hibernate annotations like `@Where` and `@SQLDelete`.

### JPA event listeners

Use JPA lifecycle callbacks (`@PrePersist`, `@PostLoad`) or Hibernate event listeners for audit logging and entity lifecycle behavior.

## Transaction Synchronization Internals

A `PlatformTransactionManager` (`JpaTransactionManager`) starts a transaction and binds the `EntityManager`/`Connection` to the thread through `TransactionSynchronizationManager`. Code (and Spring itself) can register `TransactionSynchronization` callbacks — `beforeCommit`, `afterCommit`, `afterCompletion` — which is exactly how `@TransactionalEventListener` and cache/messaging integrations hook the commit. `NESTED` propagation uses JDBC **savepoints**.

## Isolation Levels and Anomalies

| Isolation | Dirty read | Non-repeatable read | Phantom |
| --- | --- | --- | --- |
| READ_UNCOMMITTED | possible | possible | possible |
| READ_COMMITTED | prevented | possible | possible |
| REPEATABLE_READ | prevented | prevented | possible* |
| SERIALIZABLE | prevented | prevented | prevented |

Defaults differ by database (PostgreSQL `READ_COMMITTED`, MySQL/InnoDB `REPEATABLE_READ`); *InnoDB's next-key locks block many phantoms under RR.

## Connection Pool (HikariCP) Tuning

Key settings: `maximumPoolSize` (throughput is bounded by the DB, so a **small** pool often beats a large one), `minimumIdle`, `connectionTimeout`, `maxLifetime` (keep below the DB/infra idle cutoff), and `leakDetectionThreshold`. Oversized pools cause context-switching and DB contention — size to the database's real concurrency, not the app's request rate.

## Batch Inserts and Identity

Enable batching with `hibernate.jdbc.batch_size`, `order_inserts`, and `order_updates`. Critically, `GenerationType.IDENTITY` **disables** insert batching (Hibernate needs the generated key per row); use `SEQUENCE` with a pooled optimizer instead. For PostgreSQL, `reWriteBatchedInserts=true` collapses batches into multi-row inserts.

## Second-Level Cache Strategies

The L2 cache is shared across sessions: set `shared-cache-mode`, annotate cacheable entities, and pick a provider (JCache/Ehcache/Infinispan) with a strategy — `READ_ONLY` (immutable reference data), `NONSTRICT_READ_WRITE`, `READ_WRITE` (soft locks), or `TRANSACTIONAL`. Use it for rarely-changing, frequently-read data; avoid it for hot, write-heavy entities, and be wary of the query cache.

## Mapping Strategies

Model value objects with `@Embeddable`/`@Embedded`, convert custom types with `AttributeConverter`, and choose an inheritance strategy deliberately: `SINGLE_TABLE` (fast, nullable columns), `JOINED` (normalized, extra joins), or `TABLE_PER_CLASS` (rarely ideal). Composite keys use `@EmbeddedId`/`@IdClass`; share a primary key with `@MapsId`.

## Multiple DataSources and Read Replicas

Define separate `DataSource`/`EntityManagerFactory`/`TransactionManager` beans (one `@Primary`). For read/write splitting, an `AbstractRoutingDataSource` selects a replica based on a `ThreadLocal` set from `@Transactional(readOnly = true)`, and `LazyConnectionDataSourceProxy` defers acquiring the connection until the first statement so the routing decision is correct.

## Schema Migrations

Manage schema with **Flyway** (`V1__init.sql`, repeatable `R__view.sql`) or **Liquibase** changelogs, applied automatically on startup. In production use `spring.jpa.hibernate.ddl-auto=validate` (never `update`) and let the migration tool own DDL.

## Interview Q&A

**Q: What is the persistence context and how does it relate to the first-level cache?**
A: It is the `EntityManager`'s set of managed entities within a transaction — and it *is* the first-level cache: repeated loads of the same id return the same instance, and it tracks changes for dirty checking. It is mandatory and cannot be disabled.

**Q: What causes `LazyInitializationException` and how do you fix it properly?**
A: Accessing a `LAZY` association after the persistence context/transaction has closed. Fix it by fetching what you need inside the transaction — `JOIN FETCH`, `@EntityGraph`, or a DTO projection — rather than relying on Open Session in View.

**Q: What is the N+1 problem and how do you solve it?**
A: One query loads N parents, then N extra queries load each parent's association. Detect it via SQL logging or Hibernate statistics; solve it with `JOIN FETCH`, `@EntityGraph`, or `@BatchSize`.

**Q: How does optimistic locking differ from pessimistic locking?**
A: Optimistic (`@Version`) assumes low contention and only checks a version column at update time, failing with `OptimisticLockException` on conflict. Pessimistic (`PESSIMISTIC_WRITE`) takes a database lock up front, blocking others — safer under high contention but with lower concurrency.

**Q: When does Hibernate actually run SQL for a change?**
A: Not immediately — it queues changes and flushes at the flush point (before matching queries and at commit under `AUTO`). Dirty checking then generates the minimal insert/update/delete statements.

**Q: Why does Spring translate persistence exceptions, and how?**
A: To decouple callers from provider-specific exceptions. `PersistenceExceptionTranslationPostProcessor` (active on `@Repository` beans) converts them into Spring's consistent, unchecked `DataAccessException` hierarchy.

**Q: How do you enable JDBC batch inserts and why do they help?**
A: Set `spring.jpa.properties.hibernate.jdbc.batch_size` (plus `order_inserts`/`order_updates`); Hibernate then sends inserts/updates as batched `PreparedStatement`s, cutting round-trips dramatically.

**Q: Why does `GenerationType.IDENTITY` hurt batching?**
A: With IDENTITY, Hibernate needs the database-generated key immediately after each insert, so it cannot batch them. Use a `SEQUENCE` with a pooled optimizer to keep batching.

**Q: When should you use the second-level cache?**
A: For small, read-mostly, rarely-changing reference data with an appropriate strategy (`READ_ONLY`/`READ_WRITE`). Avoid it for hot, write-heavy entities, and be cautious with the query cache.

**Q: How do you route reads to a replica?**
A: Use an `AbstractRoutingDataSource` keyed off a `ThreadLocal` set from `@Transactional(readOnly = true)`, with `LazyConnectionDataSourceProxy` so the connection (and routing) is chosen at the first statement.

**Q: What are the trade-offs of `SINGLE_TABLE` vs `JOINED` inheritance?**
A: `SINGLE_TABLE` is fast (no joins) but needs nullable columns and loses some constraints; `JOINED` is normalized and constraint-friendly but adds a join per query. `TABLE_PER_CLASS` is rarely ideal.

## Interview Notes

* Explain persistence context lifecycle and flush behavior.
* Describe first-level versus second-level caches.
* Know how optimistic locking works with `@Version`.
