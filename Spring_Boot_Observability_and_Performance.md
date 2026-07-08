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

## Actuator Internals and Custom Endpoints

Actuator endpoints are beans annotated `@Endpoint` with `@ReadOperation`/`@WriteOperation`/`@DeleteOperation` methods, exposed over web (`@WebEndpoint`) and/or JMX. Health aggregates `HealthContributor`s from a registry; `info` aggregates `InfoContributor`s. Secure them with a dedicated `SecurityFilterChain` over `EndpointRequest.toAnyEndpoint()` and optionally isolate them on `management.server.port`.

```java
@Component
@Endpoint(id = "features")
public class FeatureEndpoint {
    @ReadOperation
    public Map<String, Boolean> features() { return flags.snapshot(); }
}
```

## Micrometer Architecture

Micrometer is a facade: your code records to a `MeterRegistry`, and a backend implementation (Prometheus, OTLP, Datadog) publishes. Meters are named in dotted form and carry **tags/dimensions** — keep tag cardinality low (never a user id or unbounded value) to avoid metric explosions. Distinguish `Timer` (short calls) from `LongTaskTimer` (in-flight long tasks), and decide between client-side percentiles and server-side histograms (`histogram_quantile` in Prometheus).

## The Observation API

An `Observation` models one instrumented operation and emits metrics **and** a trace span from a single instrumentation point. Register `ObservationHandler`s, annotate with `@Observed`, or wrap code with `Observation.createNotStarted(...)`. This unifies what previously required separate `@Timed` and tracing code.

## Distributed Tracing Concepts

A **span** has a start/stop, tags, and events; spans form a **trace** via parent/child links. **Baggage** propagates correlated values across services; **sampling** (head-based probabilistic or tail-based) bounds overhead. Spans export to Zipkin or an OTLP collector.

## Metrics Methodologies

* **RED** (Rate, Errors, Duration) — for request-driven services.
* **USE** (Utilization, Saturation, Errors) — for resources (CPU, pools, queues).
* **SLI/SLO/error budgets** — quantify reliability targets and guide alerting on symptoms, not causes.

## Logging and Correlation

Use structured JSON logging (Logstash encoder, or Boot 3.4's built-in structured logging) and put `traceId`/`spanId` in the MDC so logs correlate with traces. The Actuator `loggers` endpoint changes log levels at runtime without a redeploy.

## JVM and GC Tuning

Size the heap for the container (`-XX:MaxRAMPercentage`), keep container awareness on, and pick a collector: **G1** (balanced default), **ZGC** (low pause, large heaps), or Parallel (throughput). Enable GC logging and watch `jvm.gc.*`, `jvm.memory.*`, and pool metrics.

## Profiling and Diagnostics

**Java Flight Recorder** gives continuous low-overhead profiling; **async-profiler** produces accurate flamegraphs without safepoint bias. Capture heap dumps for leaks (analyze in Eclipse MAT) and thread dumps for deadlocks (`jcmd`, Actuator `threaddump`/`heapdump`).

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

**Q: Why must metric tag cardinality stay low?**
A: Each unique tag combination is a separate time series; high-cardinality tags (user id, request id) explode memory and storage and can overwhelm the metrics backend.

**Q: What is the Observation API and how does it relate to metrics and tracing?**
A: A single `Observation` around an operation emits both a metric (timer/counter) and a trace span from one instrumentation point, replacing separate `@Timed` and tracing code.

**Q: What are the RED and USE methods?**
A: RED (Rate, Errors, Duration) describes request-driven services; USE (Utilization, Saturation, Errors) describes resources like CPU, pools, and queues. Both guide dashboards and alerts.

**Q: How do you correlate application logs with traces?**
A: Put `traceId`/`spanId` in the MDC (Micrometer Tracing does this) and log structured JSON, so a log line links directly to its trace in the backend.

**Q: How do you change log levels without a redeploy?**
A: `POST` to the Actuator `loggers` endpoint (e.g. `/actuator/loggers/com.acme`) to set a level at runtime.

## Interview Notes

* Explain how Actuator endpoints are exposed and secured.
* Describe Micrometer meter types and how metrics are published.
* Know startup debugging tactics and runtime tuning patterns.
