# Spring Boot Testing and Quality

## Testing Strategy

A solid Spring Boot test strategy includes:

* unit tests for business logic
* integration tests for Spring context and persistence
* slice tests for web, JPA, or reactive layers
* end-to-end tests for full application behavior

## Spring Boot Test Support

`@SpringBootTest` loads the full application context and is useful for integration tests, but it is slower than slice tests.

### Common annotations

* `@SpringBootTest`
* `@WebMvcTest`
* `@DataJpaTest`
* `@WebFluxTest`
* `@AutoConfigureMockMvc`
* `@JsonTest`

### `@SpringBootTest` specifics

`SpringBootContextLoader` loads the application context and honors `@TestConfiguration`, active profiles, and properties. The Spring TestContext Framework caches contexts keyed by configuration.

## Mocking and slicing

Slice tests narrow the loaded context and improve speed. Use `@MockBean` to replace dependencies not needed for the slice.

### Slicing examples

* `@WebMvcTest` for MVC layer behavior
* `@DataJpaTest` for persistence and repository behavior
* `@WebFluxTest` for reactive web APIs

## Testcontainers

Testcontainers provides real containerized services for databases, Kafka, Redis, and other infrastructure.

Example:

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
```

Use `@DynamicPropertySource` to wire container URLs into the Spring environment.

## Contract and API Testing

Use Spring Cloud Contract or Pact for consumer-driven contract testing. Validate API contracts in CI and ensure compatibility across services.

## Reactive and Asynchronous Testing

Use `StepVerifier` for Reactor sequences, `WebTestClient` for WebFlux, and `async` support in `MockMvc` for deferred results.

## Internal test mechanics

### Context caching

The Spring TestContext Framework caches application contexts keyed by configuration. Reusing contexts across tests reduces startup time.

### `@DirtiesContext`

`@DirtiesContext` invalidates cache entries when a test modifies shared state, but it slows down the suite.

## Quality Practices

* use static analysis tools like SpotBugs, Checkstyle, and SonarQube
* write clear assertions and avoid brittle test setup
* keep tests isolated and avoid shared mutable state
* prefer real scenarios over implementation-specific tests

## Interview Notes

* Describe the difference between `@SpringBootTest` and slice tests.
* Explain context caching and the impact of `@DirtiesContext`.
* Know when to use Testcontainers versus embedded databases.
