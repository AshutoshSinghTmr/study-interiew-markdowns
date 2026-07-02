# Spring Boot Overview

## Why Spring Boot

Spring Boot is an opinionated extension of Spring Framework designed to remove boilerplate and simplify production-ready application setup. It bundles starters, auto-configuration, runtime defaults, and embedded servers to help developers focus on business logic instead of infrastructure wiring.

## Core Concepts

### `@SpringBootApplication`

`@SpringBootApplication` is a meta-annotation combining:

* `@Configuration`
* `@EnableAutoConfiguration`
* `@ComponentScan`

Internally, `@EnableAutoConfiguration` imports `AutoConfigurationImportSelector`, which scans `spring.factories` entries to locate auto-configuration classes.

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

Spring Boot reads `META-INF/spring.factories` from each dependency. `AutoConfigurationImportSelector` aggregates these classes, then `AutoConfigurationMetadataLoader` uses metadata to avoid loading disabled or irrelevant configurations.

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

## Interview Notes

* Explain `@SpringBootApplication` and the startup lifecycle.
* Know how Boot assembles property sources and evaluates auto-configuration.
* Be prepared to discuss embedded server selection and overriding server defaults.
