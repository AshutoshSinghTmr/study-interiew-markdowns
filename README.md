# Spring Boot Interview Preparation

A standalone, revision-ready knowledge base for Spring Boot interviews. Each file is a self-contained topic in a consistent format:

* concise, internals-aware prose with focused code examples;
* an **Interview Q&A** section of rapid question → answer flashcards for active recall;
* an **Interview Notes** section listing the key points to be able to discuss.

**Target version:** Spring Boot 3.x (Jakarta EE, Spring Security 6, Micrometer Tracing, `AutoConfiguration.imports`). Legacy equivalents are called out where they still come up in interviews.

## Table of Contents

- [How to Revise](#how-to-revise)
- [Topics](#topics)
  - [Core & Container](#core--container)
  - [Web & API](#web--api)
  - [Data & Persistence](#data--persistence)
  - [Service & Integration](#service--integration)
  - [Operations & Delivery](#operations--delivery)

## How to Revise

1. Read a topic top-to-bottom once for understanding.
2. On later passes, cover the answers and drill the **Interview Q&A** flashcards.
3. Use the **Interview Notes** as a final pre-interview checklist per topic.

## Topics

### Core & Container

| Topic | Focus |
| --- | --- |
| [Spring Boot Overview](Spring_Boot_Overview.md) | Auto-configuration, startup lifecycle, `@SpringBootApplication`, context selection |
| [Dependency Injection](Spring_Boot_Dependency_Injection.md) | Injection styles, bean scopes/lifecycle, ambiguity resolution, circular refs, proxies |
| [Configuration and Properties](Spring_Boot_Configuration_and_Properties.md) | Externalized config, precedence, `@ConfigurationProperties`, profiles, conditionals |
| [AOP and Aspects](Spring_Boot_AOP_and_Aspects.md) | Proxy model, advice types, pointcuts, ordering, self-invocation |

### Web & API

| Topic | Focus |
| --- | --- |
| [Controller and REST API](Spring_Boot_Controller_and_REST_API.md) | `DispatcherServlet` flow, binding, validation, `@ControllerAdvice`, `ProblemDetail` |
| [Reactive and WebFlux](Spring_Boot_Reactive_and_WebFlux.md) | Reactor `Mono`/`Flux`, event-loop model, backpressure, R2DBC, context propagation |
| [Security and Auth](Spring_Boot_Security_and_Auth.md) | Filter chain, `SecurityFilterChain`, method security, JWT/OAuth2, CSRF/CORS |

### Data & Persistence

| Topic | Focus |
| --- | --- |
| [Repository Variations](Spring_Boot_Repository_Variations.md) | Spring Data hierarchy, query strategies, proxy internals, choosing an abstraction |
| [Data Access Advanced](Spring_Boot_Data_Access_Advanced.md) | Persistence context, fetch/N+1, caching, locking, batching, exception translation |
| [Caching](Spring_Boot_Caching.md) | Cache abstraction, `@Cacheable`/`@CachePut`/`@CacheEvict`, providers, pitfalls |

### Service & Integration

| Topic | Focus |
| --- | --- |
| [Service Layer Patterns](Spring_Boot_Service_Layer_Patterns.md) | Transactions, propagation/isolation, design patterns, events, AOP concerns |
| [Scheduling and Async](Spring_Boot_Scheduling_and_Async.md) | `@Scheduled`, `@Async`, executors, exception handling, virtual threads |
| [Messaging and Kafka](Spring_Boot_Messaging_and_Kafka.md) | Kafka producers/consumers, offsets, DLT, idempotency, outbox, RabbitMQ |
| [Microservices Patterns](Spring_Boot_Microservices_Patterns.md) | Discovery, config, resilience, gateway, tracing, event-driven design |

### Operations & Delivery

| Topic | Focus |
| --- | --- |
| [Observability and Performance](Spring_Boot_Observability_and_Performance.md) | Actuator, Micrometer metrics/tracing, logging, JVM/DB tuning, diagnostics |
| [Deployment and Release](Spring_Boot_Deployment_and_Release.md) | Executable/layered JARs, Docker, cloud/K8s, CI/CD, native image |
| [Testing and Quality](Spring_Boot_Testing_and_Quality.md) | Slices vs `@SpringBootTest`, `@MockitoBean`, Testcontainers, context caching |
