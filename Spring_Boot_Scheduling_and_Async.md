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

## Scheduler Internals

`@EnableScheduling` adds a `ScheduledAnnotationBeanPostProcessor` that registers each `@Scheduled` method against a `TaskScheduler` (default `ThreadPoolTaskScheduler`). Timing is driven by a `Trigger`: `CronTrigger` parses Spring's **6-field** cron (`second minute hour day-of-month month day-of-week`, plus macros like `@daily`), and `PeriodicTrigger` handles fixed rate/delay.

## Distributed Scheduling

Every instance runs its own scheduler, so in a cluster a job fires once per instance. Guard it with a distributed lock — **ShedLock** (`@SchedulerLock` + a JDBC/Redis `LockProvider`) ensures a single execution — or use Quartz clustering / leader election for heavier needs.

## Async Internals

`@EnableAsync` adds an `AsyncAnnotationBeanPostProcessor`; its `AsyncExecutionInterceptor` submits the call to an `Executor` chosen by the `@Async` qualifier and adapts the return type (`void`, `Future`, `CompletableFuture`).

## Executor Tuning

`ThreadPoolTaskExecutor` has a subtle rule: the pool grows to `corePoolSize`, then **fills the queue** (`queueCapacity`), and only then grows to `maxPoolSize`. So an unbounded queue means `maxPoolSize` is never reached. Tune `keepAlive`, `allowCoreThreadTimeOut`, and a `RejectedExecutionHandler` (`AbortPolicy` vs `CallerRunsPolicy` for backpressure).

## CompletableFuture Composition

Compose async work without blocking: `thenApply`/`thenCompose` (chain), `thenCombine`/`allOf`/`anyOf` (fan-in), and `exceptionally`/`handle` (recovery). Always pass an explicit executor to `supplyAsync` rather than relying on the common `ForkJoinPool` for I/O.

## Context Propagation

Worker threads do not inherit `ThreadLocal` state, so security, MDC, and trace context are lost. Copy them with a `TaskDecorator` on the executor (or `DelegatingSecurityContextExecutor`), and use Micrometer context-propagation for tracing.

## Virtual Threads

On Java 21 with `spring.threads.virtual.enabled=true`, each task runs on a lightweight virtual thread, giving massive concurrency for blocking I/O without pool tuning. Watch for **pinning** (blocking inside `synchronized` or native calls), which ties a virtual thread to its carrier.

## Graceful Shutdown and Monitoring

Set `setWaitForTasksToCompleteOnShutdown(true)` + `awaitTerminationSeconds`, and `server.shutdown=graceful`, so in-flight work drains on shutdown. Export executor metrics (queue depth, active count, completed) via Micrometer, and ensure scheduled methods handle their own exceptions — an uncaught exception is logged and the next scheduled run still proceeds.

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

**Q: How do `queueCapacity` and `maxPoolSize` interact in `ThreadPoolTaskExecutor`?**
A: The pool grows to `corePoolSize`, then tasks queue until `queueCapacity` is full, and only then does it grow to `maxPoolSize`. An unbounded queue means `maxPoolSize` is never reached.

**Q: How do you propagate security or MDC/trace context to `@Async` threads?**
A: Worker threads do not inherit thread-locals; copy them with a `TaskDecorator` on the executor (or `DelegatingSecurityContextExecutor`) and Micrometer context-propagation for tracing.

**Q: What cron format does `@Scheduled` use?**
A: A six-field expression — `second minute hour day-of-month month day-of-week` — plus macros like `@daily`/`@hourly`; you can externalize it with `cron = "${job.cron}"`.

**Q: How do you gracefully drain in-flight tasks on shutdown?**
A: Set `setWaitForTasksToCompleteOnShutdown(true)` and `awaitTerminationSeconds` on the executor, plus `server.shutdown=graceful`, so running tasks finish before the app exits.

**Q: What is a `RejectedExecutionHandler`, and which policy gives backpressure?**
A: It decides what happens when the pool and queue are full. `CallerRunsPolicy` runs the task on the submitting thread (natural backpressure); `AbortPolicy` (default) throws `RejectedExecutionException`.

## Interview Notes

* Contrast `fixedRate`, `fixedDelay`, and `cron`, and know the default single-thread scheduler.
* Explain `@Async` return types and how exceptions are (or aren't) surfaced.
* Know the self-invocation caveat, cluster-safe scheduling, and virtual threads (Boot 3.2+).
