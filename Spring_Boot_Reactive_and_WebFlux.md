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

## Reactive Data Access

Spring Data reactive repositories work with R2DBC and reactive MongoDB drivers. Repositories return `Mono` or `Flux`, and backpressure is propagated through the driver stack.

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

## Context propagation

Reactive context propagation is handled by Reactor `Context`. Use `Context.of(...)` or `contextWrite(...)` for request-scoped values such as security or tracing.

## Testing Reactive Applications

* `WebTestClient` for end-to-end testing
* `StepVerifier` for validating reactive sequences
* `@WebFluxTest` for slice tests

## Interview Notes

* Explain the difference between Spring MVC and WebFlux.
* Describe how Reactor handles backpressure.
* Know why blocking calls must be avoided in WebFlux and how to isolate them.
