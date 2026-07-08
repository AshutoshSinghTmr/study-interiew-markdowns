# Spring Boot Scheduling and Async

## Overview

Spring Boot offers two complementary building blocks for background work:

* **Scheduling** (`@Scheduled`) runs tasks on a timer or cron.
* **Asynchronous execution** (`@Async`) offloads a method call to another thread so the caller does not block.

Both are proxy-based, so the AOP **self-invocation** rule applies: calling an annotated method from within the same bean bypasses the behavior.

## Scheduling

Enable it with `@EnableScheduling`, then annotate `void`, no-argument methods with `@Scheduled`.

```java
@Component
public class ReportJobs {

    @Scheduled(fixedDelay = 60_000, initialDelay = 10_000)
    public void pollQueue() { /* runs 60s after the previous run finishes */ }

    @Scheduled(fixedRate = 30_000)
    public void heartbeat() { /* starts every 30s regardless of duration */ }

    @Scheduled(cron = "0 0 2 * * *", zone = "UTC")
    public void nightlyRollup() { /* 02:00 UTC daily */ }
}
```

* `fixedDelay` — gap measured from the **end** of the previous execution.
* `fixedRate` — gap measured from the **start**; runs can overlap if the task is slow.
* `cron` — full cron expression with optional `zone`.
* Values can come from properties: `fixedDelayString = "${job.delay}"`.

### The scheduler thread pool

By default the scheduler uses a **single thread**, so a long task delays every other scheduled task. Configure a pool for concurrency:

```yaml
spring:
  task:
    scheduling:
      pool:
        size: 5
```

## Asynchronous Execution

Enable with `@EnableAsync`, then annotate methods with `@Async`. Supported return types: `void` (fire-and-forget), `Future<T>`, and `CompletableFuture<T>` (compose results, propagate exceptions).

```java
@Service
public class EmailService {

    @Async
    public CompletableFuture<Boolean> send(EmailRequest request) {
        boolean ok = gateway.deliver(request);
        return CompletableFuture.completedFuture(ok);
    }
}
```

### The async executor

Without configuration Boot 3 provides a `ThreadPoolTaskExecutor` (`spring.task.execution.*`). Define a named executor for isolation and pick it per method with `@Async("reportsExecutor")`:

```java
@Bean
ThreadPoolTaskExecutor reportsExecutor() {
    ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(4);
    ex.setMaxPoolSize(8);
    ex.setQueueCapacity(100);
    ex.setThreadNamePrefix("reports-");
    ex.initialize();
    return ex;
}
```

## Exception Handling

* **`CompletableFuture`/`Future`** — exceptions surface when the caller resolves the future.
* **`void` `@Async`** — exceptions cannot reach the caller; register an `AsyncUncaughtExceptionHandler` via `AsyncConfigurer` to log or react to them.

## Virtual Threads (Boot 3.2+)

On Java 21, set `spring.threads.virtual.enabled=true` so blocking web request handling, `@Async`, and scheduling run on **virtual threads**. This gives high concurrency for blocking I/O without large native thread pools, and is often simpler than reactive for I/O-bound blocking workloads.

## Common Pitfalls

* **Single scheduler thread** — the default pool size of 1 serializes all `@Scheduled` tasks.
* **`fixedRate` overlap** — slow tasks pile up; use `fixedDelay` or a scheduler lock.
* **Self-invocation** — an internal call to an `@Async`/`@Scheduled` method runs synchronously/never.
* **Swallowed exceptions** — `void @Async` errors vanish without an exception handler.
* **Multi-instance scheduling** — every instance fires the same job; use a distributed lock (e.g. ShedLock) to run once.
* **Context loss** — security/trace context does not automatically cross threads; propagate it (task decorators, Micrometer context propagation).

## Interview Q&A

**Q: What is the difference between `fixedRate` and `fixedDelay`?**
A: `fixedRate` schedules the next run relative to the previous run's start time (runs can overlap if slow); `fixedDelay` waits a fixed gap after the previous run finishes (no overlap).

**Q: Why does my `@Scheduled` task block other scheduled tasks?**
A: The default scheduler has one thread. Increase `spring.task.scheduling.pool.size` or use separate schedulers so long tasks don't starve others.

**Q: How do exceptions behave with `@Async`?**
A: With `Future`/`CompletableFuture` they propagate when the caller resolves the future. With a `void` return they're lost unless you register an `AsyncUncaughtExceptionHandler`.

**Q: Why did `@Async` run on the caller's thread?**
A: Self-invocation (called from the same class) or a missing `@EnableAsync` — the proxy that dispatches to the executor was bypassed.

**Q: How do you stop a scheduled job from running on every instance in a cluster?**
A: Use a shared lock so only one node executes each fire — commonly ShedLock backed by a database or Redis.

**Q: How do virtual threads change scheduling/async in Boot 3.2+?**
A: With `spring.threads.virtual.enabled=true` on Java 21, tasks run on lightweight virtual threads, giving massive concurrency for blocking I/O without tuning large thread pools.

## Interview Notes

* Contrast `fixedRate`, `fixedDelay`, and `cron`, and know the default single-thread scheduler.
* Explain `@Async` return types and how exceptions are (or aren't) surfaced.
* Know the self-invocation caveat, cluster-safe scheduling, and virtual threads (Boot 3.2+).
