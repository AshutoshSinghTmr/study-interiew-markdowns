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

## Test Slice Catalog

Each slice auto-configures only its layer, so contexts stay small and fast:

| Slice | Loads |
| --- | --- |
| `@WebMvcTest` | MVC infrastructure + the controller under test (no services) |
| `@DataJpaTest` | JPA, repositories, an embedded DB, transactional rollback |
| `@WebFluxTest` | WebFlux handlers + `WebTestClient` |
| `@JsonTest` | Jackson/serialization |
| `@RestClientTest` | `RestClient`/`RestTemplate` + `MockRestServiceServer` |
| `@DataR2dbcTest` / `@JdbcTest` | reactive / plain JDBC data layers |

## Context Caching Deep Dive

The TestContext framework caches the `ApplicationContext` keyed by its full configuration (config classes, locations, properties, profiles, initializers, and mock-bean definitions). Any difference creates a **new** context — expensive. Keep test configurations uniform, avoid gratuitous `@TestPropertySource`/`@MockitoBean` variety, and use `@DirtiesContext` only when a test truly corrupts shared state.

## Mocking and Test Configuration

`@MockitoBean`/`@MockitoSpyBean` (Boot 3.4) add or replace beans with mocks/spies. `@TestConfiguration` supplies test-only beans that scanning ignores unless imported. `@TestBean` (Boot 3.4) overrides a bean via a static factory method without Mockito.

## Testcontainers Integration

`@Testcontainers` + `@Container` run real dependencies. **`@ServiceConnection`** (Boot 3.1) auto-wires the container's connection details into the environment — no manual `@DynamicPropertySource`. Enable container **reuse** for speed, and use `@ServiceConnection` with Compose or at dev time via `spring-boot-docker-compose`/Devtools.

```java
@Testcontainers
@SpringBootTest
class OrderIT {
    @Container @ServiceConnection
    static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16");
}
```

## Web Layer Testing

`MockMvc` exercises the MVC stack without a running server (fast); `@SpringBootTest(webEnvironment = RANDOM_PORT)` with `TestRestTemplate`/`WebTestClient` (or RestAssured) hits a real port for true end-to-end checks. Boot 3.4 adds the AssertJ-style `MockMvcTester`.

## Data and Transaction Testing

`@DataJpaTest` wraps each test in a transaction and **rolls back** by default, keeping tests isolated. Seed data with `@Sql` or `TestEntityManager`, use `@Commit` when you need persistence, and remember Hibernate may not flush until a query runs — call `flush()` to assert constraint violations.

## Contract and Reliability Testing

Consumer-driven **contract tests** (Spring Cloud Contract or Pact) verify API compatibility in CI without full integration. Keep tests deterministic: inject a fixed `Clock`, replace `Thread.sleep` with **Awaitility**, and consider **mutation testing** (PIT) to measure whether assertions actually catch regressions.

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

**Q: What is `@ServiceConnection` and what does it replace?**
A: A Boot 3.1 annotation on a Testcontainers container that auto-wires its connection details into the environment, replacing manual `@DynamicPropertySource` wiring.

**Q: What is the difference between `MockMvc` and a `RANDOM_PORT` test?**
A: `MockMvc` drives the MVC stack in-process without a real server (fast); `@SpringBootTest(webEnvironment = RANDOM_PORT)` starts an embedded server and uses `TestRestTemplate`/`WebTestClient` for true end-to-end HTTP.

**Q: How does `@DataJpaTest` keep tests isolated?**
A: It configures an embedded/Testcontainers DB and wraps each test in a transaction that **rolls back** afterward, so tests do not leak data into one another.

**Q: What is the difference between `@TestConfiguration` and `@Configuration`?**
A: `@TestConfiguration` is not picked up by component scanning; it applies only when explicitly imported (or as a static nested class of the test), so it customizes beans just for that test.

**Q: What is consumer-driven contract testing?**
A: The consumer defines the expected request/response contract; tools like Spring Cloud Contract or Pact generate provider tests and consumer stubs from it, catching incompatibilities in CI without full integration.

## Interview Notes

* Describe the difference between `@SpringBootTest` and slice tests.
* Explain context caching and the impact of `@DirtiesContext`.
* Know when to use Testcontainers versus embedded databases.
