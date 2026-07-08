# Spring Boot Observability and Performance

## Actuator and Health Endpoints

Spring Boot Actuator exposes production-ready endpoints like `/actuator/health`, `/actuator/metrics`, `/actuator/info`, and `/actuator/env`.

### Health indicators

Actuator auto-configures indicators for disk space, database connectivity, and custom health checks. Implement `HealthIndicator` to expose application-specific readiness.

```java
@Component
public class QueueHealthIndicator implements HealthIndicator {
    private final QueueClient queue;

    public QueueHealthIndicator(QueueClient queue) {
        this.queue = queue;
    }

    @Override
    public Health health() {
        int depth = queue.depth();
        return depth < 1000
            ? Health.up().withDetail("depth", depth).build()
            : Health.down().withDetail("depth", depth).build();
    }
}
```

### Endpoint exposure

Control exposure with `management.endpoints.web.exposure.include` and `management.endpoints.web.exposure.exclude`. Use `management.endpoint.health.group.readiness.include` for readiness probe composition.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true
```

## Metrics and Monitoring

Spring Boot integrates with Micrometer to export metrics to Prometheus, Datadog, New Relic, and others.

### Meter types

* `Counter`
* `Gauge`
* `Timer`
* `DistributionSummary`

### Automatic metrics

Boot auto-registers metrics for HTTP requests, JVM memory, CPU, garbage collection, and datasource pools.

### Meter registry

Micrometer uses a `MeterRegistry` implementation to publish metrics. Add custom meters with `MeterRegistryCustomizer` or `@Bean` methods.

```java
@Service
public class CheckoutService {
    private final Counter checkouts;

    public CheckoutService(MeterRegistry registry) {
        this.checkouts = registry.counter("checkout.total");
    }

    @Timed(value = "checkout.duration", percentiles = {0.95, 0.99})
    public void checkout(Cart cart) {
        // ... business logic ...
        checkouts.increment();
    }
}
```

## Tracing and Distributed Context

Use Micrometer Tracing (the Boot 3 successor to Spring Cloud Sleuth) with an OpenTelemetry or Brave bridge for distributed tracing, driven by the Micrometer `Observation` API.

### Trace propagation

Trace and span IDs are forwarded via headers such as W3C `traceparent` (or legacy B3 `X-B3-TraceId`). Instrumentation opens a span per operation for HTTP requests, messaging, and async execution, and injects/extracts the context at each hop so logs and spans correlate across services.

### Context propagation

Propagate tracing context through thread pools, Reactor context, and messaging channels.

## Logging

Spring Boot uses Logback by default. A `LoggingApplicationListener` initializes the logging system early in startup, and Boot's `SpringBootJoranConfigurator` processes `logback-spring.xml` (enabling the `<springProfile>` and `<springProperty>` tags).

### Custom logging

Customize levels with `logging.level.*` and configure appenders in `logback-spring.xml`. Use profile-specific logging config and `spring.profiles.active`.

## Performance Tuning

### JVM tuning

Tune heap sizes and GC options. Use `-XX:+UseG1GC` or ZGC for low-pause requirements. Monitor GC pauses and memory usage.

### Database tuning

Tune datasource connection pool size, max lifetime, and idle timeout. HikariCP defaults are CPU-aware, but production loads may require explicit configuration.

### Cache strategies

Use `@Cacheable`, `@CachePut`, and `@CacheEvict` with cache providers such as Caffeine, Redis, or Ehcache.

## Resource Management

### Thread pools

Configure executor beans for async tasks and request handling. In reactive apps, avoid blocking on event-loop threads and offload blocking work to dedicated schedulers.

### Connection pools

Monitor connection pool usage with Actuator metrics and tune `maximumPoolSize`, `minimumIdle`, and `maxLifetime`.

## Profiling and Diagnostics

### Startup analysis

Use `--debug` to print auto-configuration decisions. `spring.main.lazy-initialization=true` can reduce startup cost but delays failures.

### Dump analysis

Use heap dumps and thread dumps to diagnose leaks and deadlocks. Tools like VisualVM, Java Flight Recorder, and `jcmd` are valuable.

## Interview Q&A

**Q: How do you expose and secure Actuator endpoints in production?**
A: Only `health` (and often `info`) are web-exposed by default; opt others in with `management.endpoints.web.exposure.include`. Secure them via Spring Security (a `SecurityFilterChain` on `EndpointRequest.toAnyEndpoint()`) and optionally a separate `management.server.port`.

**Q: What is the difference between liveness and readiness probes?**
A: Liveness signals the app is running (restart if it fails); readiness signals it can serve traffic (remove from load balancing if it fails). Boot exposes them as health groups under `/actuator/health/liveness` and `/readiness`.

**Q: What are the main Micrometer meter types?**
A: `Counter` (monotonic count), `Gauge` (current value), `Timer` (count + latency distribution), and `DistributionSummary` (distribution of non-time values). Micrometer is a vendor-neutral facade over registries like Prometheus.

**Q: How does Boot 3 do tracing now that Sleuth is gone?**
A: Via Micrometer Tracing plus the `Observation` API, bridged to OpenTelemetry or Brave; a single `Observation` can emit both metrics and a trace span.

**Q: How would you diagnose slow startup or a memory leak?**
A: For startup, use `--debug` / `BufferingApplicationStartup` and consider lazy init; for leaks, capture a heap dump and analyze with VisualVM or Eclipse MAT, and use thread dumps (`jcmd`) or Java Flight Recorder for CPU/lock issues.

**Q: Why can `spring.main.lazy-initialization=true` be risky?**
A: It speeds startup by deferring bean creation, but wiring/configuration errors then surface on first use at runtime rather than at boot.

## Interview Notes

* Explain how Actuator endpoints are exposed and secured.
* Describe Micrometer meter types and how metrics are published.
* Know startup debugging tactics and runtime tuning patterns.
