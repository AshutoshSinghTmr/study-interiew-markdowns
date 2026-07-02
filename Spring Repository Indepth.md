# Spring Boot Repository Variations Guide

Spring Boot repository patterns are built on Spring Data. The right repository abstraction depends on API needs, complexity, and how much query control you need.

---

## 1. Built-in Spring Data Interfaces

Spring Data defines a repository interface hierarchy. Choose the right level of abstraction based on your operations and query patterns.

### `Repository`

A marker interface with no methods. Use it when you want a strict contract and only expose selected operations.

### `CrudRepository`

Provides baseline CRUD operations such as `save()`, `findById()`, `findAll()`, `count()`, and `deleteById()`.

### `ListCrudRepository`

Extends `CrudRepository` and returns `List<T>` instead of `Iterable<T>` for retrieval methods.

### `PagingAndSortingRepository`

Adds sorting and pagination capabilities with `findAll(Sort)` and `findAll(Pageable)`.

### `JpaRepository`

The most common interface for JPA apps. It adds JPA-specific methods like `flush()`, `saveAll()`, `deleteAllInBatch()`, and `getOne()`.

---

## 2. Query Strategy

### Derived query methods

Spring Data parses repository method names to build queries via `PartTree`. This is ideal for simple filters and paging.

Example:

```java
List<Product> findByCategoryAndPriceLessThan(String category, Double price);
```

Internally, Spring Data creates a `JpaQueryMethod` and a `PartTreeJpaQuery`.

### `@Query` with JPQL

Use `@Query` for joins, projections, and queries that cannot be expressed by method names.

```java
@Query("SELECT p FROM Product p WHERE p.status = :status")
List<Product> findByStatus(@Param("status") String status);
```

### Native SQL

Use `nativeQuery = true` when JPQL cannot express the needed SQL. Result mapping is handled by JPA metadata and must match entity mappings.

### Modifying queries

Annotate bulk updates or deletes with `@Modifying`. Use `clearAutomatically = true` to avoid stale persistence context state.

### Projections and DTOs

Use interface or class-based projections to reduce payload and avoid entity exposure. Spring Data can map query results into DTOs when supported by the provider.

---

## 3. Custom Implementations

### Fragment pattern

Define a custom fragment interface and provide an implementation class suffixed with `Impl`. Spring Data merges the fragment with the repository proxy.

Example:

```java
public interface ProductCustomRepository {
    List<Product> findByComplexCriteria(ProductFilter filter);
}

@Component
public class ProductCustomRepositoryImpl implements ProductCustomRepository {
    @PersistenceContext
    private EntityManager em;

    @Override
    public List<Product> findByComplexCriteria(ProductFilter filter) {
        // custom EntityManager logic
    }
}
```

### Custom repository bean

For full control, implement a plain `@Repository` bean using `JdbcTemplate`, `NamedParameterJdbcTemplate`, or `EntityManager`.

---

## 4. Dynamic Query Patterns

### Specifications

Use `JpaSpecificationExecutor` to build dynamic predicates with the Criteria API.

### Query by Example (QBE)

QBE creates queries from a probe object and matcher configuration, useful for simple dynamic search screens.

---

## 5. Choosing the Best Strategy

* `JpaRepository` for full JPA apps with CRUD, paging, and convenience methods
* `PagingAndSortingRepository` when sorting and pagination are required but you want a narrower API
* `CrudRepository` or `ListCrudRepository` for simple CRUD-focused services
* `Repository` for a minimal explicit contract
* Derived queries for straightforward lookups
* `@Query` for complex joins or projections
* Custom fragments for reusable custom behavior
* Pure repository beans when SQL control is essential

---

## Summary

| Variation | Best Use Case |
| :--- | :--- |
| `Repository` | Strict API contract, minimal exposure |
| `CrudRepository` | Basic CRUD without extra features |
| `ListCrudRepository` | CRUD with list semantics |
| `PagingAndSortingRepository` | Paging and sorting requirements |
| `JpaRepository` | Feature-rich JPA persistence |
| Custom fragment | Complex custom behavior with Spring Data wiring |
| Pure repository bean | Legacy SQL or non-standard persistence |
