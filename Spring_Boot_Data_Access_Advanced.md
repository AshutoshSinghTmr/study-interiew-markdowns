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

* `AUTO` (default): flushes before queries and commit
* `COMMIT`: flushes only at commit time
* `MANUAL`: flushes only on explicit `flush()` calls

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

Use `hibernate.statement_cache.size` to reduce statement preparation overhead.

### Fetch size

Set `spring.jpa.properties.hibernate.jdbc.fetch_size` for large result sets and streaming queries.

## Locking and Concurrency Control

### Optimistic locking

Use `@Version` for version-based concurrency control. Hibernate adds a version check to the update statement and increments the version upon success.

### Pessimistic locking

Use `@Lock(LockModeType.PESSIMISTIC_WRITE)` or `EntityManager.lock()`. The database enforces the lock semantics, and behavior depends on the JDBC driver.

## Projections and DTO Mapping

### Interface projections

Spring Data can create projections using interfaces or classes. Interface projections are mapped lazily and avoid entity hydration when supported.

### Class-based DTOs

Use constructor expressions in JPQL or map query results manually with `Tuple` or `Object[]`.

### Entity graphs

`@EntityGraph` defines fetch plans declaratively. The JPA provider uses the graph to optimize SQL and avoid unnecessary joins.

## Query tuning Internals

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

## Interview Notes

* Explain persistence context lifecycle and flush behavior.
* Describe first-level versus second-level caches.
* Know how optimistic locking works with `@Version`.
