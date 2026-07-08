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

## Interview Notes

* Explain the difference between Spring MVC and WebFlux.
* Describe how Reactor handles backpressure.
* Know why blocking calls must be avoided in WebFlux and how to isolate them.
