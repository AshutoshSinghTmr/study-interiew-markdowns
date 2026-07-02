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

## Interview Notes

* Describe how Spring Data turns repository interfaces into proxy-backed beans.
* Explain `QueryLookupStrategy`, `PartTree`, and `RepositoryFactorySupport`.
* Know when to choose repository fragments versus custom repository beans.
