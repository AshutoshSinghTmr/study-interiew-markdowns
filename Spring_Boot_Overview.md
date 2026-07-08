# Spring Boot Overview

## Why Spring Boot

Spring Boot is an opinionated extension of Spring Framework designed to remove boilerplate and simplify production-ready application setup. It bundles starters, auto-configuration, runtime defaults, and embedded servers to help developers focus on business logic instead of infrastructure wiring.

## Core Concepts

### `@SpringBootApplication`

`@SpringBootApplication` is a meta-annotation combining:

* `@Configuration`
* `@EnableAutoConfiguration`
* `@ComponentScan`

Internally, `@EnableAutoConfiguration` imports `AutoConfigurationImportSelector`, which reads candidate auto-configuration classes from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 2.7+/3.x; the legacy `spring.factories` `EnableAutoConfiguration` key is deprecated).

### Auto-configuration

Auto-configuration applies configuration beans conditionally based on classpath and environment state. Key annotations include:

* `@ConditionalOnClass`
* `@ConditionalOnMissingBean`
* `@ConditionalOnProperty`
* `@ConditionalOnResource`

At startup, Spring Boot builds a list of candidate auto-config classes, evaluates each condition with `ConditionEvaluator`, and registers the matching beans.

### Starters and Dependency Management

Starter POMs, like `spring-boot-starter-web`, bundle opinionated dependency sets. `spring-boot-dependencies` is a BOM that provides compatible versions for transitive libraries and prevents dependency conflicts.

### Application Context and Lifecycle

Spring Boot chooses a context type based on the application environment:

* `AnnotationConfigServletWebServerApplicationContext` for servlet-based web apps
* `ReactiveWebServerApplicationContext` for WebFlux apps
* `AnnotationConfigApplicationContext` for non-web apps

Startup flow:

1. `SpringApplication.run()` creates the `SpringApplication` object.
2. `SpringApplication` builds the `SpringApplicationRunListeners` and prepares the environment.
3. It selects the appropriate `ApplicationContext` type.
4. `ConfigFileApplicationListener` loads property sources.
5. The bean definition registry is populated.
6. `BeanFactoryPostProcessor`s and `BeanPostProcessor`s execute.
7. The context refresh is invoked.
8. `ApplicationRunner` and `CommandLineRunner` beans execute after refresh.

### Bootstrap Context

When using Spring Cloud Config, Boot creates a lightweight bootstrap context first. `ConfigServicePropertySourceLocator` loads remote properties into the environment before the main application context starts.

## Internal Mechanics

### How auto-configuration classes are discovered

Spring Boot reads the `AutoConfiguration.imports` files (one per dependency, under `META-INF/spring/`) — the modern replacement for the `spring.factories` `EnableAutoConfiguration` key. `AutoConfigurationImportSelector` aggregates these classes, then `AutoConfigurationMetadataLoader` uses metadata to avoid loading disabled or irrelevant configurations.

### Web application detection

Boot detects a web app by checking for servlet or reactive classes on the classpath and by inspecting `webApplicationType`. This determines whether to use embedded Tomcat/Jetty/Undertow or Netty.

### Application event model

Boot publishes lifecycle events like `ApplicationStartingEvent`, `ApplicationEnvironmentPreparedEvent`, `ApplicationPreparedEvent`, `ApplicationReadyEvent`, and `ApplicationFailedEvent`. Listeners can react to startup phases.

## Spring Boot Modules to Know

* Actuator: health, metrics, audit, and management endpoints
* DevTools: fast reload, live restart, and development utilities
* Data: JPA, MongoDB, Redis, Cassandra, JDBC
* Security: Spring Security integration for authentication and authorization
* Test: Boot-friendly testing utilities and slice annotations

## Common Patterns

### Minimal configuration

Use starter dependencies and `application.yml` to bootstrap applications quickly. Override defaults only where necessary.

### Layered JARs

Layered JARs split the executable JAR into separate layers for dependencies, resources, and application classes. This improves Docker caching and deployment speed.

### Profiles and environment-specific behavior

Spring Boot merges property sources by precedence and activates profiles using `spring.profiles.active` or `spring.profiles.include`. Profile-specific files like `application-dev.yml` are loaded automatically by `ConfigFileApplicationListener`.

## Practical Notes for Experienced Developers

* Use `spring.main.lazy-initialization=true` carefully to improve startup time, but be aware of delayed failures.
* Override auto-configuration explicitly with `spring.autoconfigure.exclude` when needed.
* Use `--debug` to print auto-configuration decisions during startup.

## The Run Sequence in Depth

`SpringApplication.run()` drives a fixed sequence, and `SpringApplicationRunListeners` (default `EventPublishingRunListener`) broadcast each phase as an event. Knowing the order explains *where* to hook in:

1. `ApplicationStartingEvent` — before any environment or context work; the `BootstrapContext` exists.
2. `ApplicationEnvironmentPreparedEvent` — the `Environment` is built and property sources/profiles are resolved, but no beans exist. Ideal for an `EnvironmentPostProcessor`.
3. `ApplicationContextInitializedEvent` — the context is created and `ApplicationContextInitializer`s have run, before bean definitions load.
4. `ApplicationPreparedEvent` — bean definitions are loaded but the context is not yet refreshed.
5. `ContextRefreshedEvent` — the core Spring refresh completed (all singletons instantiated).
6. `ApplicationStartedEvent` — after refresh, before runners; liveness becomes `CORRECT`.
7. `ApplicationReadyEvent` — after `ApplicationRunner`/`CommandLineRunner` beans; readiness becomes `ACCEPTING_TRAFFIC`.
8. `ApplicationFailedEvent` — published if startup throws, so listeners can report before exit.

Because some events fire before the context can resolve beans, early listeners are registered through `META-INF/spring.factories` `ApplicationListener` entries or `SpringApplication.addListeners(...)`, not as `@Bean`s.

## Auto-configuration Ordering and Overriding

Auto-configuration classes are annotated with `@AutoConfiguration`, which combines `@Configuration(proxyBeanMethods = false)` with ordering metadata. Order matters because conditions like `@ConditionalOnMissingBean` are evaluated in sequence:

* `@AutoConfiguration(before = ..., after = ...)` and `@AutoConfigureOrder` position a class relative to others.
* User configuration is processed **before** auto-configuration, so a bean you define wins over a `@ConditionalOnMissingBean` default.
* Exclude with `@SpringBootApplication(exclude = X.class)`, `@EnableAutoConfiguration(excludeName = ...)`, or `spring.autoconfigure.exclude`.
* Inspect decisions via the `ConditionEvaluationReport` (`--debug`) or Actuator's `conditions` endpoint, which lists positive and negative matches with the reason.

This ordering plus `@ConditionalOnMissingBean` is the mechanism that makes Boot "opinionated but overridable": defaults apply only when you have not expressed an opinion.

## Customizing the Bootstrap

`SpringApplicationBuilder` gives fluent control for advanced startups (parent/child contexts, banners, sources):

```java
new SpringApplicationBuilder(App.class)
    .web(WebApplicationType.SERVLET)
    .bannerMode(Banner.Mode.OFF)
    .initializers(new CustomContextInitializer())
    .listeners(new CustomListener())
    .properties("spring.config.import=optional:configserver:")
    .run(args);
```

Key extension points: `ApplicationContextInitializer` (mutate the context before refresh), `ApplicationListener` (react to lifecycle events), `EnvironmentPostProcessor` (add property sources), and `BeanFactoryPostProcessor`/`BeanPostProcessor` (definition/instance customization). All are registered in `AutoConfiguration.imports` or `spring.factories` so they load without an active context.

## Failure Analysis

When startup fails, Boot routes the exception through registered `FailureAnalyzer`s, which turn low-level stack traces into actionable messages (for example, "port already in use" or "a required bean could not be found"). Implement `AbstractFailureAnalyzer<T>` and register it so your own libraries emit clear startup diagnostics instead of raw stack traces.

## Startup Performance and AOT

* **`ApplicationStartup`** — set a `BufferingApplicationStartup` to record startup steps and export a flamegraph-friendly timeline of where the context spent time.
* **Lazy initialization** (`spring.main.lazy-initialization=true`) — defers bean creation to first use, cutting startup time but shifting wiring failures to runtime.
* **AOT processing** (Boot 3) — generates bean-registration code, proxy classes, and reflection hints ahead of time; it powers **GraalVM native images** (instant startup, low memory) and also speeds JVM startup.
* **CDS / checkpoint-restore** — Class Data Sharing and Project CRaC (Boot 3.2+) further reduce cold-start latency for serverless and autoscaled workloads.

## Spring Boot 3.x Specifics

Boot 3 raised the baseline to **Java 17**, migrated `javax.*` to the **Jakarta EE 9+** namespace, and adopted the **Micrometer Observation** API for unified metrics and tracing. It added first-class GraalVM native support, `AutoConfiguration.imports` for config discovery, and (3.2+) **virtual threads** via `spring.threads.virtual.enabled=true` on Java 21. Upgrading from Boot 2 chiefly means the Jakarta package move, Security 6 config changes, and replacing Sleuth with Micrometer Tracing.

## Interview Q&A

**Q: What does `@SpringBootApplication` combine, and what does each part do?**
A: `@Configuration` (bean definitions), `@ComponentScan` (discovers components from the main class's package downward), and `@EnableAutoConfiguration` (imports conditional auto-configuration via `AutoConfigurationImportSelector`).

**Q: How does auto-configuration know what to configure?**
A: Candidate classes listed in `AutoConfiguration.imports` are filtered by `@Conditional` checks (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, ...). Only conditions that pass register beans, so your own beans and the classpath drive the result.

**Q: How do you override or disable a specific auto-configuration?**
A: Define your own bean (most auto-config is `@ConditionalOnMissingBean`), set the relevant property, or exclude it with `@SpringBootApplication(exclude = ...)` / `spring.autoconfigure.exclude`.

**Q: How does Boot decide between a servlet, reactive, or non-web application?**
A: It inspects the classpath (`webApplicationType`): servlet classes → `AnnotationConfigServletWebServerApplicationContext` with embedded Tomcat/Jetty/Undertow; only reactive classes → `ReactiveWebServerApplicationContext` with Netty; neither → a plain `AnnotationConfigApplicationContext`.

**Q: What is the difference between `CommandLineRunner` and `ApplicationRunner`?**
A: Both run once after the context is ready; `CommandLineRunner` receives the raw `String[]` args, while `ApplicationRunner` receives a parsed `ApplicationArguments`.

**Q: How can you diagnose why a bean was or wasn't auto-configured?**
A: Run with `--debug` (or `debug=true`) to print the `ConditionEvaluationReport` of positive/negative matches, and inspect Actuator's `conditions` endpoint.

**Q: How does Boot choose the embedded server, and how do you switch to Jetty or Undertow?**
A: With `spring-boot-starter-web`, Tomcat is the default. Exclude it and add `spring-boot-starter-jetty` or `-undertow` to switch; the server is auto-configured from whichever is on the classpath.

**Q: What is a "starter" and what does the parent/BOM provide?**
A: A starter is a curated dependency aggregator (e.g. `spring-boot-starter-data-jpa`). `spring-boot-dependencies` (imported by `spring-boot-starter-parent`) is a BOM that pins compatible versions, so you omit versions and avoid conflicts.

**Q: How do `@ConditionalOnClass` and `@ConditionalOnMissingBean` enable graceful defaults?**
A: `@ConditionalOnClass` applies config only when a library is present; `@ConditionalOnMissingBean` backs off when you define your own bean — together they give sensible defaults that you can always override.

**Q: What runs code at startup, and how do you avoid blocking readiness?**
A: `CommandLineRunner`/`ApplicationRunner` run after the context is ready. For slow work, offload to `@Async` or an `ApplicationReadyEvent` listener so the readiness probe is not delayed.

**Q: How do you package the same app as a WAR for an external container?**
A: Extend `SpringBootServletInitializer`, override `configure(...)`, set packaging to `war`, and mark the embedded server dependency `provided`. It still runs as an executable JAR too.

## Interview Notes

* Explain `@SpringBootApplication` and the startup lifecycle.
* Know how Boot assembles property sources and evaluates auto-configuration.
* Be prepared to discuss embedded server selection and overriding server defaults.
