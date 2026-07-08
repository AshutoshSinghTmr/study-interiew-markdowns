# Spring Boot Reactive and WebFlux

## Reactive Programming Fundamentals

Spring WebFlux is built on Project Reactor. Core types are:

* `Mono<T>` for zero-or-one values
* `Flux<T>` for zero-or-many values

Reactive programming uses asynchronous, non-blocking flows and backpressure to process streams efficiently.

## WebFlux Architecture

WebFlux supports two runtimes:

* Netty via `ReactorNetty` (default)
* Servlet 3.1+ containers with `Tomcat`, `Jetty`, or `Undertow`

### Request processing

`DispatcherHandler` routes requests through a reactive pipeline. Controller methods or functional handlers return `Mono` or `Flux`. `HttpMessageReader` reads request bodies and `HttpMessageWriter` serializes responses.

### Runtime wiring

WebFlux configures `HandlerMapping`, `HandlerAdapter`, and `WebExceptionHandler` beans. `WebFluxConfigurationSupport` registers codecs, validation, and reactive adapters.

## Threading Model

WebFlux uses event-loop threads instead of one thread per request. Blocking operations must be isolated using `publishOn`, `subscribeOn`, or dedicated `Scheduler`s to avoid starving the event loop.

## Spring Boot Integration

### Auto-configuration

Spring Boot auto-configures WebFlux on the classpath and sets up codecs, validators, and the HTTP server.

### Functional endpoints

Routes can be defined using `RouterFunction` and `HandlerFunction` for concise, functional-style APIs.

```java
@RestController
@RequestMapping("/api/prices")
class PriceController {
    private final PriceService prices;

    PriceController(PriceService prices) {
        this.prices = prices;
    }

    @GetMapping("/{sku}")
    Mono<Price> bySku(@PathVariable String sku) {
        return prices.findBySku(sku);
    }

    @GetMapping
    Flux<Price> all() {
        return prices.findAll();
    }
}
```

The same endpoints expressed as a functional route:

```java
@Bean
RouterFunction<ServerResponse> priceRoutes(PriceHandler handler) {
    return route(GET("/api/prices/{sku}"), handler::bySku)
        .andRoute(GET("/api/prices"), handler::all);
}
```

## Reactive Data Access

Spring Data reactive repositories work with R2DBC and reactive MongoDB drivers. Repositories return `Mono` or `Flux`, and backpressure is propagated through the driver stack.

```java
public interface PriceRepository extends ReactiveCrudRepository<Price, Long> {
    Flux<Price> findByCurrency(String currency);
    Mono<Price> findBySku(String sku);
}
```

### Non-blocking persistence

Reactive database support requires a non-blocking driver. Blocking JDBC cannot be used directly without offloading to a separate thread pool.

## Internal Signal Flow

Reactor connects `Publisher` and `Subscriber` through subscriptions. The subscriber requests demand, which enables backpressure and controlled emission.

### Common operators

* `map`, `flatMap`
* `filter`
* `zip`, `merge`
* `publishOn`, `subscribeOn`
* `retry`, `retryWhen`

## Error Handling

Handle exceptions with `onErrorResume`, `onErrorMap`, `onErrorContinue`, and `doOnError`. Use global WebFlux error handlers to return consistent API responses.

## Context Propagation

Reactive context propagation is handled by Reactor `Context`. Use `Context.of(...)` or `contextWrite(...)` for request-scoped values such as security or tracing.

## Testing Reactive Applications

* `WebTestClient` for end-to-end testing
* `StepVerifier` for validating reactive sequences
* `@WebFluxTest` for slice tests

## Assembly Time vs Subscription Time

Building a `Mono`/`Flux` pipeline (**assembly**) does no work; execution starts only on **subscribe** — "nothing happens until you subscribe." Most sources are **cold** (each subscriber triggers the work afresh); a **hot** publisher (e.g. `Sinks.many().multicast()`) emits regardless of subscribers. `Mono.defer`/`Flux.defer` postpone source creation to subscription time so per-request state is captured correctly.

## Schedulers

Reactor never implicitly changes threads; you choose one:

* `Schedulers.parallel()` — fixed pool for CPU-bound work.
* `Schedulers.boundedElastic()` — for wrapping unavoidable blocking I/O.
* `Schedulers.single()` / `immediate()` — single-threaded / caller thread.

`subscribeOn` sets the thread for the **source** (affects the whole upstream chain); `publishOn` switches threads for everything **downstream** of it.

## Backpressure

Backpressure flows through the `Subscription`: a subscriber calls `request(n)` and the source emits at most `n`. When a producer outpaces a consumer, choose a strategy: `onBackpressureBuffer` (queue, optionally bounded), `onBackpressureDrop`, `onBackpressureLatest`, or `onBackpressureError`. `Flux.create(sink, overflowStrategy)` sets it at the source.

## Key Operators

* `map` (sync 1:1) vs `flatMap` (async, interleaved, with a concurrency argument) vs `concatMap` (async, ordered) vs `flatMapSequential` (concurrent, ordered output).
* `switchIfEmpty`/`defaultIfEmpty` for empty sources; `zip`/`merge`/`combineLatest` to combine streams.
* `window`/`buffer` for batching; `timeout` and `retryWhen(Retry.backoff(...))` for resilience.

## Error Handling

Use `onErrorResume` (fallback publisher), `onErrorReturn` (fallback value), `onErrorMap` (translate), and `doOnError` (side effect). Avoid `onErrorContinue` unless you understand its operator-fusion caveats. Retries belong in `retryWhen` with exponential backoff and jitter.

## Context Propagation

Thread-locals do not survive operator hops, so request-scoped values (security, trace, tenant) live in the immutable Reactor `Context`: write with `contextWrite(...)` (propagates **bottom-up**), read with `Mono.deferContextual(...)`. Boot 3's Micrometer **context-propagation** library bridges thread-local libraries into the Reactor context automatically.

## WebClient Configuration

Tune the underlying Reactor Netty client: connection pool size, `responseTimeout`, connect/read/write timeouts, `exchangeStrategies` (codec max in-memory buffer), retry filters, and `ExchangeFilterFunction`s for auth and logging. Reuse a single configured `WebClient` per downstream service.

## R2DBC and Reactive Transactions

Reactive persistence uses non-blocking drivers via `ReactiveCrudRepository` or `R2dbcEntityTemplate`. Transactions use a `ReactiveTransactionManager` with `@Transactional` or a `TransactionalOperator` so the transaction context flows through the reactive chain.

## Testing and Blocking Detection

`StepVerifier` asserts the exact sequence of signals (`expectNext`, `expectError`, `verifyComplete`); `StepVerifier.withVirtualTime(...)` fast-forwards virtual clocks for delay/interval operators. Add **BlockHound** in tests to detect accidental blocking calls on non-blocking threads.

## Interview Q&A

**Q: When should you choose WebFlux over Spring MVC?**
A: When you need high concurrency with limited threads and the stack is end-to-end non-blocking (reactive clients, R2DBC). For blocking JDBC/JPA workloads or simpler apps, MVC is usually the better, simpler choice.

**Q: What is the difference between `Mono` and `Flux`?**
A: `Mono<T>` emits zero or one item; `Flux<T>` emits zero to many. Both are lazy `Publisher`s that do nothing until subscribed.

**Q: Why is blocking dangerous in WebFlux, and how do you handle a required blocking call?**
A: WebFlux serves many requests on a small event-loop pool; a blocking call stalls that thread and kills throughput. Isolate unavoidable blocking work on a dedicated scheduler with `subscribeOn(Schedulers.boundedElastic())`.

**Q: What is backpressure and how does Reactor implement it?**
A: Backpressure lets a slow subscriber limit how much a fast producer emits. Reactor implements it through the `Subscription`: the subscriber signals demand via `request(n)`, and operators propagate that demand upstream.

**Q: What is the difference between `map` and `flatMap`?**
A: `map` transforms each element synchronously (1:1); `flatMap` maps each element to a `Publisher` and merges the results asynchronously — use it for nested reactive calls such as another WebClient request.

**Q: How do you propagate security or trace context reactively?**
A: Thread-locals don't survive operator hops, so values travel in the Reactor `Context` via `contextWrite(...)` and are read with `Mono.deferContextual(...)`; Micrometer context-propagation bridges thread-local libraries.

**Q: What does "nothing happens until you subscribe" mean?**
A: Assembling a `Mono`/`Flux` only builds a plan; no work runs until a subscriber connects. Returning a publisher from a controller lets the framework subscribe for you.

**Q: What is the difference between `subscribeOn` and `publishOn`?**
A: `subscribeOn` sets the thread the source runs on (affecting the whole upstream chain); `publishOn` switches threads for operators downstream of it.

**Q: Which `Scheduler` do you use for blocking I/O versus CPU work?**
A: `Schedulers.boundedElastic()` to isolate unavoidable blocking I/O; `Schedulers.parallel()` for CPU-bound work. Never block on the event-loop threads.

**Q: How do `flatMap`, `concatMap`, and `flatMapSequential` differ?**
A: All map to inner publishers asynchronously; `flatMap` interleaves results (unordered), `concatMap` runs them one at a time in order, and `flatMapSequential` runs concurrently but emits in source order.

**Q: How do you retry a reactive call with backoff?**
A: `retryWhen(Retry.backoff(maxAttempts, minBackoff))` with jitter, optionally filtered to transient errors; avoid a naive `retry()` that hammers a failing dependency.

## Interview Notes

* Explain the difference between Spring MVC and WebFlux.
* Describe how Reactor handles backpressure.
* Know why blocking calls must be avoided in WebFlux and how to isolate them.
