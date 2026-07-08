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

Spring Boot evaluates sources in a predictable order; sources higher in the list win over lower ones. The real list is longer (it also includes servlet init params, `SPRING_APPLICATION_JSON`, `RandomValuePropertySource`, and profile-specific files) — the common high-to-low ordering is:

1. `DevTools` properties (when active)
2. `@TestPropertySource` / `@SpringBootTest` properties
3. command line arguments
4. `SPRING_APPLICATION_JSON` properties
5. Java system properties
6. OS environment variables
7. profile-specific files (`application-{profile}.yml`)
8. `application.properties` / `application.yml`
9. `@PropertySource`
10. default properties

### Profiles

Profiles are activated via `spring.profiles.active`, `spring.profiles.include`, or `@ActiveProfiles` in tests. Profile-specific files such as `application-dev.yml` and `application-prod.properties` are loaded automatically by the config system.

## Type-safe Configuration Binding

### `@ConfigurationProperties`

`@ConfigurationProperties` binds a set of properties to a POJO. Boot uses `Binder` and either `@EnableConfigurationProperties` or component scanning.

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private final String name;
    private final Security security;

    // Spring Boot 3 binds via the single constructor automatically;
    // @ConstructorBinding is only needed to choose among multiple constructors.
    public AppProperties(String name, Security security) {
        this.name = name;
        this.security = security;
    }

    public String getName() { return name; }
    public Security getSecurity() { return security; }

    public static class Security {
        private final boolean enabled;

        public Security(boolean enabled) { this.enabled = enabled; }

        public boolean isEnabled() { return enabled; }
    }
}
```

Register it with `@EnableConfigurationProperties(AppProperties.class)` or `@ConfigurationPropertiesScan`.

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

Implement `Converter` or `PropertyEditorRegistrar` for custom types. Immutable configuration beans are bound through their constructor (in Boot 3, `@ConstructorBinding` is only required — at the constructor level — to disambiguate multiple constructors).

### `EnvironmentPostProcessor`

Create an `EnvironmentPostProcessor` to register custom property sources before the application context refreshes.

## Interview Q&A

**Q: When should you use `@ConfigurationProperties` instead of `@Value`?**
A: Use `@ConfigurationProperties` for a group of related, hierarchical properties — it gives type safety, relaxed binding, validation (`@Validated`), and IDE metadata. Use `@Value` for a single one-off value or a SpEL expression.

**Q: What is relaxed binding?**
A: Spring maps one target property to many source formats, so `spring.datasource.url`, `SPRING_DATASOURCE_URL`, `spring.datasource-url`, and `spring.datasourceUrl` all bind to the same field — which is what lets environment variables configure any property.

**Q: How do profile-specific properties work and which wins?**
A: `application-{profile}.yml` is loaded when that profile is active and overrides the plain `application.yml`. Activate profiles with `spring.profiles.active`; `spring.profiles.group` can pull in several profiles at once.

**Q: What replaced `bootstrap.yml` / the bootstrap context?**
A: In Boot 2.4+ the `ConfigData` API and `spring.config.import` (e.g. `spring.config.import=configserver:` or `vault:`) replaced the legacy bootstrap context for importing external/remote configuration.

**Q: How do you inject configuration before the context is created?**
A: Register an `EnvironmentPostProcessor` (declared in `AutoConfiguration.imports`/`spring.factories`) to add or mutate property sources on the `Environment` during startup.

**Q: How do you keep secrets out of the codebase?**
A: Source them from environment variables, a secret manager, or Vault through `spring.config.import`; never commit them, and keep them out of logs and the Actuator `env`/`info` endpoints (mask with `management.endpoint.env.keys-to-sanitize`).

## Interview Notes

* Explain property source ordering and relaxed binding.
* Know when to choose `@Value` vs `@ConfigurationProperties`.
* Describe how Spring Boot evaluates conditionals during auto-configuration.
