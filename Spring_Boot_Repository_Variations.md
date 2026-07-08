# Spring Boot Repository Variations

## Repository Interface Hierarchy

Spring Data provides a structured interface hierarchy for repository abstractions, and it is important to pick the right level of abstraction based on API requirements.

### `Repository`

A marker interface with no methods. Use it for an explicit contract when you want fine-grained control over exposed operations.

### `CrudRepository`

Base interface for common CRUD operations. It returns generic types such as `Iterable<T>`, `Optional<T>`, and primitives.

### `ListCrudRepository`

Extends `CrudRepository` and returns `List<T>` for retrieval methods instead of `Iterable<T>`.

### `PagingAndSortingRepository`

Adds sorting and pagination support:

* `findAll(Sort sort)`
* `findAll(Pageable pageable)`

### `JpaRepository`

Extends `PagingAndSortingRepository` and adds JPA-specific operations like `flush()`, `saveAll()`, and `deleteAllInBatch()`.

## Query Strategy

### Derived query methods

Spring Data parses method names and builds queries using `PartTree`. Supported keywords include `And`, `Or`, `Between`, `Like`, `IsNull`, `OrderBy`, and more.

Example:

```java
List<Product> findByCategoryAndPriceLessThan(String category, Double price);
```

Internally, `RepositoryFactorySupport` creates a `JpaQueryMethod` and then a `PartTreeJpaQuery` for derived queries.

### `@Query` with JPQL/HQL

Use `@Query` for expressive JPQL or HQL. Spring Data converts the query into a `JpaQuery` and binds method parameters via `ParamNameDiscoverer`.

```java
@Query("SELECT p FROM Product p WHERE p.status = :status")
List<Product> findByStatus(@Param("status") String status);
```

### Native SQL

Use `nativeQuery = true` for direct SQL. The mapping is performed by JPA provider metadata, and native queries can be returned as entity mappings or scalar projections.

```java
@Query(value = "SELECT * FROM products WHERE inventory_count < 10", nativeQuery = true)
List<Product> findLowStock();
```

### Modifying queries

`@Modifying` marks update/delete queries. Spring executes them within a transactional context, and `clearAutomatically = true` flushes and clears the persistence context to avoid stale entities.

### Projections and DTOs

Spring Data supports interface-based and class-based projections. Interface projections use getter methods and can be translated into SQL projections when supported by the JPA provider.

## Custom Implementations

### Fragment pattern

Use repository fragments to add custom behavior without losing Spring Data repository wiring.

Example:

```java
public interface ProductCustomRepository {
    List<Product> findByComplexCriteria(ProductFilter filter);
}
```

Spring locates the implementation class by suffix convention (e.g. `ProductCustomRepositoryImpl`) and merges it with the repository proxy.

### Custom `Repository` bean

For complex logic or legacy SQL, implement a regular `@Repository` bean using `JdbcTemplate`, `NamedParameterJdbcTemplate`, or `EntityManager` directly.

## Internals of Spring Data Repositories

### Proxy creation

Spring Data creates a proxy using `ProxyFactory`. Repository interfaces are backed by `RepositoryFactorySupport`, which resolves implementations and query handlers.

### Query lookup

`QueryLookupStrategy.Key` chooses the strategy:

* `CREATE_IF_NOT_FOUND` — derived query first, fallback to declared query
* `USE_DECLARED_QUERY` — always use `@Query`
* `CREATE` — always derive the query

### Entity metadata

`JpaEntityInformation` extracts ID and entity metadata and supplies it to `SimpleJpaRepository`, which executes repository operations.

## Advanced Variations

### `JpaSpecificationExecutor`

Supports dynamic queries using `Specification<T>`. Specifications are composed into a `CriteriaQuery` and translated by the JPA provider.

### Query by Example (QBE)

Creates queries from a probe object. `ExampleMatcher` defines matching rules, and null values are ignored by default.

### Projection execution

Spring Data resolves projections using `ProjectionFactory` and may create proxy DTOs or map values into concrete classes.

### Custom repository base classes

Use `repositoryBaseClass` in `@EnableJpaRepositories` to supply a custom base implementation for all repositories.

## Best Practices

* Prefer derived queries for simple lookups.
* Use `@Query` for joins, DTO projections, and queries that cannot be expressed by method names.
* Reserve native queries for vendor-specific SQL or performance-sensitive statements.
* Keep repository interfaces focused and delegate complex business logic to service or custom repository classes.

## Choosing the Right Abstraction

* `JpaRepository` for full JPA apps needing CRUD, paging, batching, and convenience methods.
* `PagingAndSortingRepository` when sorting and pagination are required but you want a narrower API.
* `CrudRepository` or `ListCrudRepository` for simple CRUD-focused services (`ListCrudRepository` when you prefer `List` over `Iterable`).
* `Repository` for a minimal, explicit contract that exposes only the operations you declare.
* Derived queries for straightforward lookups, `@Query` for complex joins or projections, native SQL for vendor-specific statements.
* Repository fragments for reusable custom behavior; a plain `@Repository` bean when full SQL control is essential.

| Variation | Best use case |
| :--- | :--- |
| `Repository` | Strict API contract, minimal exposure |
| `CrudRepository` | Basic CRUD without extra features |
| `ListCrudRepository` | CRUD with `List` return semantics |
| `PagingAndSortingRepository` | Paging and sorting requirements |
| `JpaRepository` | Feature-rich JPA persistence |
| Custom fragment | Complex custom behavior with Spring Data wiring |
| Pure repository bean | Legacy SQL or non-standard persistence |

## The Repository Proxy Pipeline

Boot auto-enables `@EnableJpaRepositories`, which registers a `JpaRepositoryFactoryBean` for each interface. At creation, `RepositoryFactorySupport` builds a **JDK dynamic proxy** whose `QueryExecutorMethodInterceptor` routes each call to a **fragment** implementation, the declared `@Query`, a **derived** query, or the shared base (`SimpleJpaRepository`). `RepositoryComposition` combines fragments and base class. Queries are resolved and **validated at bootstrap** (fail-fast), so a bad `@Query` fails startup rather than at first call.

## Derived Query Grammar

A method name parses into a *subject* (`find`/`read`/`get`/`query`/`count`/`exists`/`delete`) + `By` + a predicate built from keywords: `And`, `Or`, `Between`, `LessThan`, `Like`, `StartingWith`, `In`, `IgnoreCase`, `OrderBy...Desc`, `True`. Limit results with `findFirst10By...`/`findTop...`, dedupe with `Distinct`, and traverse nested properties (`findByCustomerAddressCity`), disambiguating with an underscore (`findByCustomer_Id`) when needed.

## Projections and Dynamic Results

* **Closed interface projection** — getters only; the provider can push it into the SQL `SELECT` (no full entity load).
* **Open projection** — `@Value("#{...}")` SpEL; forces loading the whole entity.
* **Class/DTO projection** — constructor binding into a record or POJO.
* **Dynamic projection** — `<T> List<T> findByStatus(Status s, Class<T> type)` chooses the shape at call time.

## Specifications and Criteria

`JpaSpecificationExecutor` composes type-safe `Specification<T>` predicates, ideal for optional filters:

```java
Specification<Order> spec = Specification
    .where(hasStatus(status))
    .and(placedAfter(date));
Page<Order> page = repo.findAll(spec, pageable);
```

Use the JPA metamodel (`Order_.status`) for compile-time safety.

## Auditing

`@EnableJpaAuditing` plus `@CreatedDate`/`@LastModifiedDate`/`@CreatedBy`/`@LastModifiedBy` (with an `AuditorAware<T>` bean) populates audit columns automatically on persist and update.

## Pagination and Streaming

* **`Page`** issues a second `count` query; **`Slice`** only knows `hasNext` (cheaper); a plain `List` returns everything.
* For deep pages, prefer **keyset/seek** pagination (`WHERE id > :last ORDER BY id`) over large offsets.
* Stream large result sets with `Stream<T>` + `@QueryHints`, or use `ScrollPosition`/`Window` (Spring Data 3.1) for keyset scrolling.

## Modifying, Locking, and Batch

`@Modifying(clearAutomatically = true, flushAutomatically = true)` on bulk update/delete keeps the persistence context consistent. `@Lock(PESSIMISTIC_WRITE)` and `@QueryHints` control locking and fetch behavior. Note that `saveAll` is not automatically batched — see the batching notes in the Data Access guide.

## Spring Data 3.x Interface Graph

Spring Data Commons 3.0 **decoupled** `PagingAndSortingRepository` from `CrudRepository` and added `ListCrudRepository`/`ListPagingAndSortingRepository` (List-returning). In current versions `JpaRepository` extends the `List*` variants plus `QueryByExampleExecutor`, so you get CRUD, paging, and QBE together.

## Interview Q&A

**Q: How does Spring Data turn a repository interface into a working bean?**
A: `RepositoryFactorySupport` creates a JDK dynamic proxy for the interface. Calls are routed by an interceptor to derived-query implementations (`PartTreeJpaQuery`), declared `@Query` methods, fragment implementations, or the shared `SimpleJpaRepository` base for CRUD.

**Q: What does `QueryLookupStrategy` control?**
A: How a query is resolved for a method — `CREATE` (derive from the method name), `USE_DECLARED_QUERY` (require `@Query`), or `CREATE_IF_NOT_FOUND` (the default: try declared, then derive).

**Q: When would you use a repository fragment versus a full custom repository bean?**
A: Use a fragment (`XxxImpl`) when you want custom behavior *and* still keep Spring Data's derived/declared queries on the same interface. Use a standalone `@Repository` with `JdbcTemplate`/`EntityManager` when the logic is entirely custom or legacy SQL and you do not need the Spring Data proxy.

**Q: What is the difference between `JpaRepository` and `CrudRepository`?**
A: `JpaRepository` extends `PagingAndSortingRepository` (and `CrudRepository`) and adds JPA-specific operations such as `flush()`, `saveAllAndFlush()`, and `deleteAllInBatch()`, plus `List` return types. `CrudRepository` is the minimal persistence-agnostic CRUD contract.

**Q: How do you build a dynamic query whose predicates are known only at runtime?**
A: Extend `JpaSpecificationExecutor` and compose `Specification<T>` objects (Criteria API), or use Query by Example for simple probes. Both avoid string concatenation and stay type-safe.

**Q: Why prefer projections/DTOs over returning entities?**
A: They fetch only the needed columns, reduce payload, and avoid lazy-loading and entity-exposure pitfalls; interface projections can even be pushed into the SQL select list by the provider.

**Q: What is the cost difference between `Page` and `Slice`?**
A: `Page` runs an extra `count` query to compute total pages; `Slice` only fetches `pageSize + 1` rows to know whether a next page exists. Use `Slice` (or a `List`) when you do not need the total.

**Q: How do you avoid slow deep-offset pagination?**
A: Use keyset/seek pagination (`WHERE id > :lastSeen ORDER BY id LIMIT n`) instead of a large `OFFSET`, or `ScrollPosition`/`Window` (Spring Data 3.1) which does keyset scrolling.

**Q: What does `@Modifying` do and what are its caveats?**
A: It marks an `@Query` as an update/delete so Spring runs it as bulk DML. It bypasses the persistence context, so set `clearAutomatically`/`flushAutomatically` to avoid stale managed entities.

**Q: When do you choose an interface projection versus a class (DTO) projection?**
A: Closed interface projections let the provider select only the needed columns (no entity load); class/record DTOs use a constructor and are handy for computed values or passing across layers.

**Q: How does JPA auditing populate created/modified fields?**
A: `@EnableJpaAuditing` plus `@CreatedDate`/`@LastModifiedDate`/`@CreatedBy`/`@LastModifiedBy` (backed by an `AuditorAware` bean) fills them automatically on persist/update via an entity listener.

## Interview Notes

* Describe how Spring Data turns repository interfaces into proxy-backed beans.
* Explain `QueryLookupStrategy`, `PartTree`, and `RepositoryFactorySupport`.
* Know when to choose repository fragments versus custom repository beans.
