# Spring Boot Microservices Patterns

## Microservices Architecture

Spring Boot is widely used for microservices due to lightweight deployment, embedded servers, and strong Spring Cloud integration.

Common patterns:

* service discovery
* centralized configuration
* circuit breakers
* API gateway
* distributed tracing
* event-driven communication

## Service Discovery

### Client-side discovery

A service registers with a discovery server (Eureka, Consul, or Nacos) and discovers instances through `DiscoveryClient`.

### Server-side discovery

A gateway or load balancer resolves service instances and forwards requests.

Internally, `DiscoveryClient` implementations register instances and refresh service lists. `@EnableDiscoveryClient` and Boot auto-configuration simplify registration and health status propagation.

## Configuration Management

Spring Cloud Config Server centralizes config over HTTP. Since Boot 2.4, clients import it with `spring.config.import=configserver:` (the legacy bootstrap context now requires the optional `spring-cloud-starter-bootstrap`).

`ConfigServicePropertySourceLocator` fetches remote configuration and inserts it into the environment during startup, and `@RefreshScope` beans can be refreshed when it changes.

## Resilience and Fault Tolerance

### Circuit breaker

Use Resilience4j or Spring Cloud Circuit Breaker. The circuit breaker monitors failures, opens on threshold breaches, and short-circuits downstream calls.

```java
@CircuitBreaker(name = "inventory", fallbackMethod = "fallbackStock")
public StockLevel getStock(String sku) {
    return inventoryClient.getStock(sku);
}

private StockLevel fallbackStock(String sku, Throwable ex) {
    return StockLevel.unknown(sku); // degraded but available
}
```

### Retry

Use `@Retryable` for transient failures. Combine with `@Recover` for fallback logic.

### Bulkhead and rate limiting

Bulkheads isolate resource usage, while rate limiting throttles traffic at the gateway or service boundary.

## API Gateway

Spring Cloud Gateway routes requests and applies cross-cutting concerns like authentication, rate limiting, and path rewriting.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
```

### Internal flow

`GatewayFilterChain` processes requests through a reactive pipeline. Route definitions are loaded from configuration and evaluated by `RoutePredicateFactory`.

## Distributed Tracing

Use Micrometer Tracing (the successor to Spring Cloud Sleuth in Boot 3), typically with an OpenTelemetry or Brave bridge, to trace requests across services.

### Trace propagation

Trace and span IDs travel in headers (W3C `traceparent` or B3) so a request can be correlated end to end. See [Spring_Boot_Observability_and_Performance.md](Spring_Boot_Observability_and_Performance.md) for the full observability and context-propagation details.

## Messaging and Event-driven Design

Use Spring Cloud Stream, Spring AMQP, Kafka, or RabbitMQ for async communication.

### Patterns

* publish-subscribe
* event sourcing
* saga orchestration
* CQRS

## Observability and Health

Microservices require health checks, metrics, and centralized logging.

* Actuator exposes readiness and liveness endpoints.
* Micrometer exports metrics to Prometheus, Datadog, or New Relic.

## Deployment Patterns

* containerized microservices with Docker
* Kubernetes deployments with probes and autoscaling
* blue-green and canary releases
* sidecar patterns for logging, config, or service mesh

## Service Discovery and Client Load Balancing

Instances register with a registry (Eureka/Consul/Nacos) and are looked up through the `DiscoveryClient` abstraction. **Spring Cloud LoadBalancer** (which replaced Ribbon) does client-side balancing: a `@LoadBalanced` `RestClient`/`WebClient`/`RestTemplate` resolves `http://order-service` to a healthy instance (round-robin by default, health-aware).

## Centralized Config and Refresh

**Config Server** serves configuration from a Git/Vault backend; clients import it with `spring.config.import=configserver:`. `@RefreshScope` beans rebuild on `POST /actuator/refresh`, and **Spring Cloud Bus** (`/actuator/busrefresh`) fans a refresh out to every instance over a broker.

## Resilience4j Patterns

* **CircuitBreaker** — a sliding window (count- or time-based) trips on a failure-rate or slow-call threshold, stays OPEN for a wait duration, then allows limited HALF_OPEN trial calls.
* **Retry**, **RateLimiter**, **TimeLimiter**, and **Bulkhead** (semaphore or thread-pool isolation) round out the set.

Compose them deliberately (e.g. retry -> circuit breaker -> bulkhead) with a fallback, and export the built-in Micrometer metrics.

## API Gateway

Spring Cloud Gateway (reactive) defines routes as **predicates** (path, method, header) + **filters** (rewrite, retry, circuit breaker, **request rate limiter** backed by Redis). Global filters apply cross-cutting concerns like authentication at the edge, keeping downstream services simpler.

## Distributed Tracing Propagation

Micrometer Tracing (OpenTelemetry or Brave bridge) assigns a trace id per request and a span per hop, propagated in W3C `traceparent`/B3 headers across HTTP and messaging. **Baggage** carries extra correlated key-values, and **sampling** controls volume. The Observation API instruments the code once for both metrics and traces.

## The Saga Pattern

Cross-service business transactions avoid distributed 2PC by using a **saga**: a sequence of local transactions where each step publishes an event/command and **compensating actions** undo prior steps on failure. Orchestration uses a central coordinator/state machine; choreography relies on events. Pair it with the **outbox** pattern for reliable event emission.

## CQRS and Event Sourcing

**CQRS** separates the write model from read-optimized projections. **Event sourcing** stores state as an append-only log of events and rebuilds read models from it, giving a full audit trail and temporal queries — at the cost of eventual consistency and added complexity. Reach for them only when the domain truly benefits.

## Consistency, Idempotency, and Service Mesh

Distributed systems favor **eventual consistency**; make operations **idempotent** (idempotency keys, dedup tables) because retries and at-least-once delivery cause duplicates. A **service mesh** (Istio/Linkerd) can offload mTLS, retries, timeouts, and traffic shaping from the app. Verify integrations with **contract tests** (Spring Cloud Contract/Pact) and Testcontainers.

## Interview Q&A

**Q: What problem does service discovery solve, and how does client-side discovery work?**
A: It removes hard-coded host/port coupling. Instances register with a registry (Eureka/Consul/Nacos); a client queries the registry via `DiscoveryClient` and a client-side load balancer picks an instance.

**Q: What are the states of a circuit breaker?**
A: CLOSED (calls flow, failures counted), OPEN (calls short-circuit to the fallback after the failure threshold), and HALF_OPEN (a few trial calls decide whether to close again).

**Q: How do you keep one slow dependency from exhausting all threads?**
A: Apply a bulkhead (isolated thread pool or semaphore) plus timeouts, so failures in one dependency cannot consume the whole app's capacity.

**Q: What is the saga pattern and when do you use it?**
A: A way to keep data consistent across services without a distributed transaction: each step publishes an event/command and compensating actions undo prior steps on failure. Use it for long-running, cross-service business transactions.

**Q: How does an API gateway differ from service discovery?**
A: Discovery is the registry/lookup of where instances live; the gateway is a single edge entry point that routes to those instances and applies cross-cutting concerns (auth, rate limiting, rewriting).

**Q: What replaced Spring Cloud Sleuth in Spring Boot 3?**
A: Micrometer Tracing (with OpenTelemetry or Brave bridges) plus the Micrometer Observation API, which unifies metrics and tracing instrumentation.

**Q: What is `@RefreshScope` and how does config refresh work?**
A: `@RefreshScope` beans are recreated when configuration changes; `POST /actuator/refresh` rebinds properties, and Spring Cloud Bus (`/actuator/busrefresh`) fans the refresh out to all instances.

**Q: How does client-side load balancing work now that Ribbon is gone?**
A: Spring Cloud LoadBalancer replaces Ribbon; a `@LoadBalanced` `RestClient`/`WebClient`/`RestTemplate` resolves a logical service id to a healthy instance from the registry (round-robin by default).

**Q: Orchestration vs choreography sagas — what is the difference?**
A: Orchestration uses a central coordinator that drives each step and its compensation (explicit, easy to trace); choreography has services react to each other's events (decoupled, but harder to follow).

**Q: What does a service mesh offload from the application?**
A: Cross-cutting network concerns — mTLS, retries, timeouts, circuit breaking, traffic shifting, and telemetry — handled by sidecars (Istio/Linkerd) outside your code.

**Q: How is a trace propagated across services?**
A: Micrometer Tracing injects/extracts trace and span ids in W3C `traceparent` (or B3) headers on each hop, so the whole request correlates end to end; baggage carries extra context.

## Interview Notes

* Explain the difference between service discovery and API gateway.
* Describe how Spring Cloud Config loads remote properties at startup.
* Know circuit breaker states and fallback handling.
