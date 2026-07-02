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

Spring Cloud Config Server centralizes config over HTTP. Boot creates a bootstrap context to load remote property sources before the main context starts.

`ConfigServicePropertySourceLocator` fetches remote configuration and inserts it into the environment during bootstrap.

## Resilience and Fault Tolerance

### Circuit breaker

Use Resilience4j or Spring Cloud Circuit Breaker. The circuit breaker monitors failures, opens on threshold breaches, and short-circuits downstream calls.

### Retry

Use `@Retryable` for transient failures. Combine with `@Recover` for fallback logic.

### Bulkhead and rate limiting

Bulkheads isolate resource usage, while rate limiting throttles traffic at the gateway or service boundary.

## API Gateway

Spring Cloud Gateway routes requests and applies cross-cutting concerns like authentication, rate limiting, and path rewriting.

### Internal flow

`GatewayFilterChain` processes requests through a reactive pipeline. Route definitions are loaded from configuration and evaluated by `RoutePredicateFactory`.

## Distributed Tracing

Use OpenTelemetry or Sleuth to trace requests across services.

### Trace propagation

Trace IDs are forwarded via headers such as `X-B3-TraceId` or W3C Trace Context. Instrumentation captures spans for HTTP, messaging, and async execution.

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

## Interview Notes

* Explain the difference between service discovery and API gateway.
* Describe how Spring Cloud Config loads properties during bootstrap.
* Know circuit breaker states and fallback handling.
