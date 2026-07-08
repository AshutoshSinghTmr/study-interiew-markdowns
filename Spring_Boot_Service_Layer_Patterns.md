# Spring Boot Service Layer Patterns

## Purpose of the Service Layer

The service layer separates business logic from controllers and repositories. It defines transactional boundaries, enforces domain rules, and orchestrates integration with external systems.

## Transaction Management

Spring manages transactions with `@Transactional`, implemented using AOP proxies and `TransactionInterceptor`.

```java
@Service
public class TransferService {
    private final AccountRepository accounts;

    public TransferService(AccountRepository accounts) {
        this.accounts = accounts;
    }

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accounts.findById(fromId).orElseThrow();
        Account to = accounts.findById(toId).orElseThrow();
        from.debit(amount);
        to.credit(amount);
        // managed entities are flushed automatically at commit
    }
}
```

### Propagation behaviors

* `REQUIRED` (default)
* `REQUIRES_NEW`
* `NESTED`
* `SUPPORTS`
* `NOT_SUPPORTED`
* `MANDATORY`
* `NEVER`

### Isolation levels

* `READ_COMMITTED`
* `REPEATABLE_READ`
* `SERIALIZABLE`
* `READ_UNCOMMITTED`

### Rollback rules

Spring rolls back on unchecked exceptions by default. Use `rollbackFor` or `noRollbackFor` to customize rollback behavior for checked exceptions.

### Internal transaction flow

1. `@EnableTransactionManagement` registers a `TransactionInterceptor` and `TransactionAttributeSourceAdvisor`.
2. The proxy intercepts service calls and delegates to `TransactionAspectSupport`.
3. The `PlatformTransactionManager` begins, commits, or rolls back the transaction.
4. Spring synchronizes the transaction with the persistence context and triggers `beforeCommit` callbacks.

### Self-invocation pitfall

A method call within the same bean bypasses the proxy, so `@Transactional` on self-invoked methods is not applied unless using AspectJ weaving.

## Service Design Patterns

### Layered architecture

* Controller → Service → Repository
* Domain services hold business rules.
* Application services coordinate use cases.

### Transaction Script vs Domain Model

* Transaction Script: service methods orchestrate operations in a procedural fashion.
* Domain Model: entities encapsulate behavior and invariants.

### Anti-corruption layer

Use an anti-corruption layer to translate between external models and your internal domain, preventing leakage of external implementation details.

### Service orchestration

Centralize cross-cutting concerns in the service layer such as caching, retries, metrics, and authorization checks.

## Events and Messaging

Spring supports synchronous domain events via `ApplicationEventPublisher` and asynchronous events using `@Async` or messaging.

### Internal event handling

`ApplicationEventMulticaster` dispatches events to listeners. Boot uses `SimpleApplicationEventMulticaster` by default, which can be backed by an `Executor` for async handling.

### Best practices

* Use events to decouple side effects from transactional work.
* Avoid events for simple direct control flows.
* Prefer domain events for significant state transitions.

## Exception Handling and Validation

The service layer validates business rules and translates lower-level exceptions.

* Use `@Validated` on service beans for method-level validation.
* Catch provider-specific exceptions and throw domain-specific exceptions.
* Keep exception translation close to the boundary between persistence and domain logic.

## Advanced Topic: AOP and Cross-cutting Concerns

The service layer is a natural place for cross-cutting concerns like security, caching, retries, and metrics. See [Spring_Boot_AOP_and_Aspects.md](Spring_Boot_AOP_and_Aspects.md) for the proxy internals in depth.

### How AOP works

Spring uses dynamic proxies or CGLIB-generated subclasses to intercept method calls. Advisors such as `TransactionAttributeSourceAdvisor` wrap matching methods with advice.

### Common advices

* `@Transactional`
* `@Cacheable`, `@CacheEvict`
* `@Retryable`
* security method interceptors

### Order of advices

Advice order is important. For example, retry should often wrap transaction advice to avoid retrying partially committed operations.

## Integration with resilience libraries

Spring Boot integrates with Resilience4j and Spring Retry to add circuit breakers, bulkheads, and retry logic at the service boundary.

## Testing Service Layers

* unit test services with mocked repositories and collaborators
* use `@SpringBootTest` or sliced contexts for integration tests
* verify transactional behavior with real persistence contexts
* use `@DataJpaTest` for repository-focused persistence tests

## Interview Q&A

**Q: How does `@Transactional` actually work under the hood?**
A: Spring wraps the bean in an AOP proxy; `TransactionInterceptor` starts a transaction via a `PlatformTransactionManager` before the method and commits or rolls back afterward, based on the outcome and rollback rules.

**Q: Why doesn't `@Transactional` work when a method calls another method in the same class?**
A: Self-invocation goes through `this`, not the proxy, so the interceptor never runs. Fix it by injecting the bean into itself, moving the method to another bean, or using AspectJ load-time weaving.

**Q: What is the default rollback behavior and how do you change it?**
A: Spring rolls back on `RuntimeException` and `Error` but *not* on checked exceptions. Override with `@Transactional(rollbackFor = ...)` or `noRollbackFor`.

**Q: When would you use `REQUIRES_NEW` vs `REQUIRED`?**
A: `REQUIRED` (default) joins an existing transaction. `REQUIRES_NEW` suspends the current one and runs independently — useful for actions like audit logging that must commit even if the outer transaction rolls back.

**Q: What is the difference between Transaction Script and Domain Model?**
A: Transaction Script puts procedural logic in service methods (simple, but can grow anemic); Domain Model pushes behavior and invariants into rich entities (better for complex domains). The service layer still owns transactions and orchestration in both.

**Q: Why does advice ordering matter for `@Retryable` and `@Transactional`?**
A: If retry sits inside the transaction, a retry reuses a transaction that may already be rollback-only. Put retry *outside* (higher precedence) so each attempt gets a fresh transaction.

## Interview Notes

* Explain how `@Transactional` proxies work and why self-invocation bypasses them.
* Describe the difference between application services and domain services.
* Know common pitfalls such as lazy loading outside a transaction and misordered AOP advice.
