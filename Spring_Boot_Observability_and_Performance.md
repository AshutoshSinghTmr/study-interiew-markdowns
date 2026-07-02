# Spring Boot Observability and Performance

## Actuator and Health Endpoints

Spring Boot Actuator exposes production-ready endpoints like `/actuator/health`, `/actuator/metrics`, `/actuator/info`, and `/actuator/env`.

### Health indicators

Actuator auto-configures indicators for disk space, database connectivity, and custom health checks. Implement `HealthIndicator` to expose application-specific readiness.

### Endpoint exposure

Control exposure with `management.endpoints.web.exposure.include` and `management.endpoints.web.exposure.exclude`. Use `management.endpoint.health.group.readiness.include` for readiness probe composition.

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

## Tracing and Distributed Context

Use OpenTelemetry or Spring Cloud Sleuth for distributed tracing.

### Trace propagation

Trace IDs are forwarded via headers such as `X-B3-TraceId` or W3C Trace Context. Instrumentation captures spans for HTTP requests, messaging, and async operations.

### Context propagation

Propagate tracing context through thread pools, Reactor context, and messaging channels.

## Logging

Spring Boot uses Logback by default and processes `logback-spring.xml` through `SpringBootLoggingInitializer`.

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

## Interview Notes

* Explain how Actuator endpoints are exposed and secured.
* Describe Micrometer meter types and how metrics are published.
* Know startup debugging tactics and runtime tuning patterns.
