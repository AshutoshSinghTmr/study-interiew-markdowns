# Spring Boot Configuration and Properties

## Externalized Configuration

Spring Boot externalizes configuration through `ConfigurableEnvironment`, which aggregates ordered property sources inside `MutablePropertySources`.

Common property sources:

* `application.properties` / `application.yml`
* command line arguments
* OS environment variables
* JVM system properties
* `@PropertySource`-annotated resources
* `ConfigData` imported sources via `spring.config.import`
* custom property sources from `EnvironmentPostProcessor`

### Loading flow

1. `SpringApplication.run()` starts and calls `prepareEnvironment()`.
2. `ConfigFileApplicationListener` loads config data from `application.*` files.
3. `ConfigDataEnvironmentPostProcessor` resolves imported config locations.
4. `ConfigurableEnvironment` merges sources in precedence order.
5. `Binder` resolves properties against beans and `@Value` placeholders.

## Property Resolution

Spring uses `Environment` for property access and relaxed binding to match names like `spring.datasource.url`, `SPRING_DATASOURCE_URL`, and `spring.datasource.URL`.

### Property source precedence

Spring Boot evaluates sources in predictable order. Later sources override earlier ones.

1. `DevTools` properties
2. `@TestPropertySource` locations
3. command line arguments
4. JNDI attributes
5. Java system properties
6. OS environment variables
7. `application.properties` / `application.yml`
8. `@PropertySource`

### Profiles

Profiles are activated via `spring.profiles.active`, `spring.profiles.include`, or `@ActiveProfiles` in tests. Profile-specific files such as `application-dev.yml` and `application-prod.properties` are loaded automatically by the config system.

## Type-safe Configuration Binding

### `@ConfigurationProperties`

`@ConfigurationProperties` binds a set of properties to a POJO. Boot uses `Binder` and either `@EnableConfigurationProperties` or component scanning.

```java
@ConfigurationProperties(prefix = "app")
@ConstructorBinding
public class AppProperties {
    private final String name;
    private final Security security;

    public AppProperties(String name, Security security) {
        this.name = name;
        this.security = security;
    }

    public static class Security {
        private final boolean enabled;

        public Security(boolean enabled) {
            this.enabled = enabled;
        }
    }
}
```

### `@Value`

`@Value` resolves a single expression from the environment, including SpEL. It is less maintainable than `@ConfigurationProperties` for many related properties.

## Conditional Configuration Internals

Spring Boot evaluates conditionals during auto-configuration using `ConditionEvaluator`. Results are cached in `ConditionEvaluationReport` so unavailable conditions are not re-evaluated repeatedly.

* `@ConditionalOnClass` checks for classpath presence.
* `@ConditionalOnMissingBean` checks the current bean registry.
* `@ConditionalOnProperty` checks property values.
* `@ConditionalOnBean` checks for specific bean types.
* `@ConditionalOnWebApplication` detects web environments.

## Bean Registration and Lifecycle

`@SpringBootApplication` includes `@ComponentScan`, so Boot auto-discovers annotated beans. Explicit registration through `@Import` or `@ImportResource` is still supported.

### Bean lifecycle order

1. Bean definitions load into `BeanDefinitionRegistry`.
2. `BeanFactoryPostProcessor`s modify definitions before beans instantiate.
3. `BeanPostProcessor`s intercept creation to handle injection and proxying.
4. `SmartInitializingSingleton` and `ApplicationRunner` beans run after initialization.

## Secrets and Sensitive Properties

Use `spring.config.import` for remote configuration sources, Vault, or encrypted values. Avoid committing secrets to source control.

Best practices:

* keep secrets in environment variables, Vault, or secret managers
* use `spring.config.activate.on-profile` to limit exposure
* avoid sensitive values in logs or `info` endpoints

## Advanced Configuration Patterns

### `spring.profiles.group`

Defines grouped profiles that activate multiple profiles together, useful for large environments.

### Custom property binding

Implement `Converter` or `PropertyEditorRegistrar` for custom types, and use `@ConstructorBinding` for immutable configuration beans.

### `EnvironmentPostProcessor`

Create an `EnvironmentPostProcessor` to register custom property sources before the application context refreshes.

## Interview Notes

* Explain property source ordering and relaxed binding.
* Know when to choose `@Value` vs `@ConfigurationProperties`.
* Describe how Spring Boot evaluates conditionals during auto-configuration.
