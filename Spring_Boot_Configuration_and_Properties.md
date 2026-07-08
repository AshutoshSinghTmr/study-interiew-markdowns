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

## Config Data API Internals

Since Boot 2.4, external configuration is loaded by the **Config Data API**, replacing the old `bootstrap`/`ConfigFileApplicationListener` mechanism. Two SPIs drive it:

* `ConfigDataLocationResolver` — turns a location string (`optional:file:./config/`, `configserver:`, `vault:`) into concrete resources.
* `ConfigDataLoader` — loads each resolved resource into `ConfigData` (property sources).

`spring.config.import` composes locations declaratively, and imports are resolved in document order with later imports overriding earlier ones. Prefix a location with `optional:` so a missing source does not fail startup. This model makes Kubernetes ConfigMaps, Vault, and Config Server first-class without a separate bootstrap context.

## Property Source Ordering — the Complete Picture

Each source is a `PropertySource` inside the ordered `MutablePropertySources`; the first source that contains a key wins. Every resolved value also carries an **`Origin`**, so Actuator's `env` endpoint and error messages can point at the exact file and line a value came from. A subtle gotcha: `@PropertySource` does **not** load YAML (it is properties-only) — use `spring.config.import` or a custom `PropertySourceFactory` for YAML.

## Relaxed Binding and the Binder API

Relaxed binding is implemented by the programmatic `Binder`, which normalizes names to a canonical, lower-kebab form before matching. This is why environment variables work: `SPRING_DATASOURCE_URL` maps to `spring.datasource.url`. Binding also supports structured types:

```yaml
app:
  servers:
    - host: a.example.com   # list -> List<Server>
      port: 8080
  limits:
    orders: 100             # map -> Map<String,Integer>
```

For environment variables, list indices use `APP_SERVERS_0_HOST` and map keys use `APP_LIMITS_ORDERS`. You can drive binding manually with `Binder.get(environment).bind("app", Bindable.of(AppProps.class))` and intercept it with a `BindHandler` (for defaulting, validation, or auditing).

## Validation and Metadata

Annotate a `@ConfigurationProperties` type with `@Validated` to enforce JSR-380 constraints (`@NotNull`, `@Min`, nested `@Valid`) at startup, so misconfiguration fails fast rather than at first use. Add the `spring-boot-configuration-processor` dependency to generate `META-INF/spring-configuration-metadata.json`, which powers IDE auto-completion and documentation for your keys; supplement it with `additional-spring-configuration-metadata.json` for hints and deprecations.

## Profiles Deep Dive

* **Activation**: `spring.profiles.active`, `SPRING_PROFILES_ACTIVE`, `--spring.profiles.active`, or `@ActiveProfiles` in tests. `spring.profiles.default` (default `default`) applies when none are active.
* **Groups**: `spring.profiles.group.prod=metrics,db-prod` activates several profiles from one name.
* **Expressions**: `@Profile("prod & !legacy")` supports `&`, `|`, `!` for conditional beans.
* **Config activation**: within a file, `spring.config.activate.on-profile` gates a document, replacing the older `spring.profiles` document key.

Profile-specific files (`application-prod.yml`) always override the base `application.yml` when active, regardless of import order.

## Secrets Management

Keep secrets out of images and source control. Practical patterns: mount Kubernetes Secrets as env vars or files and import with `spring.config.import=optional:file:/etc/secrets/`; use `spring.config.import=vault://` with Spring Cloud Vault; or use Config Server encrypted `{cipher}` values. Mask them in Actuator with `management.endpoint.env.keys-to-sanitize` and never log the `Environment`.

## Runtime Refresh

With Spring Cloud, beans annotated `@RefreshScope` are re-created on a `POST /actuator/refresh` (or a bus-wide `/actuator/busrefresh`). `ContextRefresher` rebuilds the `Environment`, publishes `EnvironmentChangeEvent`, and re-binds affected `@ConfigurationProperties`, so a config change can take effect without a redeploy — useful for feature flags, log levels, and connection tuning.

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

**Q: What is the precedence among command-line args, environment variables, and `application.yml`?**
A: Higher wins: command-line args > `SPRING_APPLICATION_JSON` > OS environment variables / system properties > profile-specific files > `application.yml` > defaults.

**Q: How does `@ConfigurationProperties` validation work?**
A: Add `@Validated` with JSR-380 constraints (`@NotNull`, `@Min`, nested `@Valid`); binding runs at startup and a violation throws, so bad config fails fast rather than at first use.

**Q: How do you bind a list or map from environment variables?**
A: Use indexed/keyed names: `APP_SERVERS_0_HOST` for `app.servers[0].host` and `APP_LIMITS_ORDERS` for `app.limits.orders`. Relaxed binding maps the upper-snake form to the canonical property.

**Q: What is the difference between `spring.config.import` and `@PropertySource`?**
A: `spring.config.import` is part of the Config Data API — it supports YAML, profiles, and `optional:`/`configserver:`/`vault:` locations; `@PropertySource` is an older annotation that only loads `.properties` (not YAML) without those features.

**Q: How do profile groups help?**
A: `spring.profiles.group.prod=db-prod,metrics` activates several profiles from one name, so setting `prod` pulls in the whole set — cleaner than repeating them everywhere.

## Interview Notes

* Explain property source ordering and relaxed binding.
* Know when to choose `@Value` vs `@ConfigurationProperties`.
* Describe how Spring Boot evaluates conditionals during auto-configuration.
