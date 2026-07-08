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

`SpringBootContextLoader` loads the application context and honors `@TestConfiguration`, active profiles, and properties. Use its `webEnvironment` attribute (`MOCK`, `RANDOM_PORT`, `DEFINED_PORT`, `NONE`) to control whether a real embedded server starts.

## Mocking and Slicing

Slice tests narrow the loaded context and improve speed. Replace collaborators outside the slice with a mock — `@MockitoBean` in Boot 3.4+ (the older `@MockBean` is deprecated).

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mvc;
    @MockitoBean OrderService orderService; // @MockBean before Boot 3.4

    @Test
    void returnsOrder() throws Exception {
        given(orderService.findById(1L)).willReturn(new OrderDto(1L, "NEW"));

        mvc.perform(get("/api/orders/1"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.status").value("NEW"));
    }
}
```

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

## Internal Test Mechanics

### Context caching

The Spring TestContext Framework caches application contexts keyed by configuration. Reusing contexts across tests reduces startup time.

### `@DirtiesContext`

`@DirtiesContext` invalidates cache entries when a test modifies shared state, but it slows down the suite.

## Quality Practices

* use static analysis tools like SpotBugs, Checkstyle, and SonarQube
* write clear assertions and avoid brittle test setup
* keep tests isolated and avoid shared mutable state
* prefer real scenarios over implementation-specific tests

## Interview Q&A

**Q: When should you use `@SpringBootTest` versus a slice test?**
A: Use a slice (`@WebMvcTest`, `@DataJpaTest`, `@WebFluxTest`) to test one layer with a minimal, fast context; use `@SpringBootTest` for full integration behavior across the whole context, accepting the slower startup.

**Q: What replaced `@MockBean` in recent Spring Boot?**
A: `@MockitoBean` (and `@MockitoSpyBean`) from Spring Framework 6.2 / Boot 3.4; `@MockBean`/`@SpyBean` are deprecated. Both add or replace a mock bean in the test context.

**Q: How does the TestContext Framework speed up test suites?**
A: It caches the loaded `ApplicationContext` keyed by configuration (classes, profiles, properties) and reuses it across tests. Anything that changes that key — or `@DirtiesContext` — forces a new context, which is expensive.

**Q: Why prefer Testcontainers over an in-memory database?**
A: Testcontainers runs the real engine (e.g. PostgreSQL) so tests catch dialect/behavior differences an in-memory DB (H2) would hide; wire container URLs in with `@DynamicPropertySource`.

**Q: How do you test a secured or JSON-serialization concern in isolation?**
A: Use focused slices — `@WebMvcTest` with `spring-security-test` for auth, `@JsonTest` for serialization — so only the relevant auto-configuration loads.

**Q: How do you test reactive code deterministically?**
A: Use `StepVerifier` to assert the sequence of emitted signals on a `Mono`/`Flux`, and `WebTestClient` for end-to-end WebFlux endpoints.

## Interview Notes

* Describe the difference between `@SpringBootTest` and slice tests.
* Explain context caching and the impact of `@DirtiesContext`.
* Know when to use Testcontainers versus embedded databases.
