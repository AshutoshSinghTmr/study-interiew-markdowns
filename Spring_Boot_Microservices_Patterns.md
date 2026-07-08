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

## Interview Notes

* Explain the difference between service discovery and API gateway.
* Describe how Spring Cloud Config loads remote properties at startup.
* Know circuit breaker states and fallback handling.
