# Spring Boot Dependency Injection

## Core DI Concepts

Spring Boot builds on Spring Framework dependency injection. The `ApplicationContext` is the IoC container that manages bean definitions, bean creation, dependency resolution, and lifecycle callbacks.

### Bean definition and registration

Spring registers beans through:

* classpath scanning of `@Component`, `@Service`, `@Repository`, and `@Controller`
* `@Configuration` classes and `@Bean` methods
* XML or Java config imports
* auto-configuration (candidate classes listed in `AutoConfiguration.imports`)

`@SpringBootApplication` includes `@ComponentScan`, and the scan base package is typically the package of the main application class. Bean definitions are represented as `BeanDefinition` objects in the `BeanDefinitionRegistry`.

### Bean metadata and wiring

Spring uses `DependencyDescriptor` to inspect injection points and determine required bean types. `AutowiredAnnotationBeanPostProcessor` resolves `@Autowired`, `@Value`, and `@Inject` dependencies before actual bean construction.

## Injection Styles

### Constructor injection

Constructor injection is recommended for mandatory dependencies and immutable beans. It also makes beans easier to test and supports `final` fields.

```java
@Service
public class OrderService {
    private final ProductRepository repository;

    public OrderService(ProductRepository repository) {
        this.repository = repository;
    }
}
```

This is the most reliable style. A pure constructor cycle, however, cannot be resolved by Spring's early-reference mechanism and needs `@Lazy` on one dependency (or a redesign) to break it.

### Setter injection

Use setter injection for optional or replaceable dependencies. Setters run after instantiation, allowing reconfiguration after construction. The trade-off is that fields cannot be `final` and the bean may be observed before it is fully wired.

```java
@Service
public class OrderService {
    private NotificationService notificationService;

    @Autowired
    public void setNotificationService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

### Field injection

Field injection uses reflection and is discouraged for production code because it hides dependencies, prevents `final` fields, and complicates testing (the bean cannot be created with a plain `new`). It is mainly useful for quick prototypes or legacy code.

```java
@Service
public class InventoryService {
    @Autowired
    private InventoryRepository inventoryRepository;
}
```

## Resolving Ambiguous Dependencies

When several beans implement the same type, Spring needs a rule to select one; otherwise it throws `NoUniqueBeanDefinitionException`.

### `@Primary`

Marks one candidate as the default when no qualifier is supplied.

```java
@Component
@Primary
public class CreditCardProcessor implements PaymentProcessor {}
```

### `@Qualifier`

Selects an exact bean by name at the injection point. More explicit and refactor-visible than relying on a primary.

```java
public BookingService(@Qualifier("payPalProcessor") PaymentProcessor processor) {
    this.processor = processor;
}
```

### `@Profile`

Registers a bean only when a given profile is active, so environment-specific implementations never collide.

```java
@Component
@Profile("prod")
public class SendGridEmailService implements EmailService {}
```

### Collection and map injection

Injecting `List<T>` or `Map<String, T>` gathers every bean of that type — a clean basis for strategy or plug-in designs. Ordering is undefined unless controlled with `@Order` or `Ordered`.

```java
@Component
public class TaxCalculationEngine {
    private final List<TaxRule> rules; // each rule ordered with @Order

    public TaxCalculationEngine(List<TaxRule> rules) {
        this.rules = rules;
    }
}
```

## Bean Scopes

Default scope is `singleton`. Common scopes include:

* `prototype`
* `request`
* `session`
* `application`
* `websocket`

Prototype beans are created on each injection request. Spring does not call `@PreDestroy` for prototype beans, so cleanup must be handled manually.

Scoped beans such as `request` and `session` require proxies when injected into singleton beans.

## Bean Lifecycle and Callbacks

Spring manages lifecycle callbacks using `BeanPostProcessor`s and lifecycle interfaces.

Common hooks:

* `@PostConstruct`
* `@PreDestroy`
* `InitializingBean`
* `DisposableBean`
* `SmartInitializingSingleton`

### Bean initialization flow

1. Bean definitions are loaded into the `BeanFactory`.
2. `BeanFactoryPostProcessor`s modify bean metadata before instantiation.
3. Dependencies are resolved and injected.
4. `postProcessBeforeInitialization()` is invoked.
5. Custom init methods and `@PostConstruct` execute.
6. `postProcessAfterInitialization()` runs.
7. `SmartInitializingSingleton.afterSingletonsInstantiated()` runs after all singletons are created.

Spring Boot registers internal processors like `AutowiredAnnotationBeanPostProcessor`, `ConfigurationPropertiesBindingPostProcessor`, and `CommonAnnotationBeanPostProcessor`.

## Conditional Beans and Auto-configuration

Spring Boot conditionals control bean registration during auto-configuration. Internally, `ConditionEvaluator` evaluates each `@Conditional` and caches results in `ConditionEvaluationReport`.

Common conditionals:

* `@ConditionalOnClass`
* `@ConditionalOnMissingBean`
* `@ConditionalOnProperty`
* `@ConditionalOnBean`
* `@ConditionalOnWebApplication`

`@Primary` and `@Qualifier` resolve ambiguous dependencies. When multiple beans match and no primary/qualifier exists, Spring throws `NoUniqueBeanDefinitionException`.

## Circular Dependency Resolution

Spring can resolve circular dependencies for singleton beans by creating a partially initialized bean and storing it in the singleton factory. The default strategy supports setter/field injection and constructor injection when one side is `@Lazy`.

Strategies to avoid cycles:

* use constructor injection with `@Lazy`
* refactor to remove the cycle
* extract shared behavior into a dedicated bean

### Lazy initialization

`@Lazy` creates a proxy that delays actual bean creation until first use. This can break cycles and reduce startup cost.

## Proxying and Scoped Beans

Spring uses proxies for aspects, transactions, and scoped beans.

* `proxyMode = ScopedProxyMode.TARGET_CLASS` uses CGLIB proxies.
* `proxyMode = ScopedProxyMode.INTERFACES` uses JDK dynamic proxies.

Proxies allow request-scoped beans to be injected into singleton beans and enable interceptors like `@Transactional` to wrap method calls.

## Advanced DI Features

### `@Bean` factory methods

Declare beans explicitly inside a `@Configuration` class when the type cannot be annotated (third-party classes) or needs custom construction. `@Configuration` classes are CGLIB-enhanced, so repeated calls to a `@Bean` method return the same singleton.

```java
@Configuration
public class AppConfig {
    @Bean
    public RestClient restClient() {
        return RestClient.create();
    }
}
```

### `@ConfigurationProperties` binding

A dedicated `Binder` binds external properties to POJOs. In Spring Boot 3 a single-constructor `@ConfigurationProperties` type is bound via its constructor automatically, so an explicit `@ConstructorBinding` is no longer required (it is only needed to pick one of several constructors). Immutable, validated configuration objects are created at startup.

### `@Import`, `ImportSelector`, and `DeferredImportSelector`

`@Import` registers additional configuration classes. `ImportSelector` chooses imports based on classpath or environment. `DeferredImportSelector` delays imports until all regular configuration classes are processed.

### Auto-configuration conditions

Spring Boot auto-configuration uses `@ConditionalOnBean` and `@ConditionalOnMissingBean` to let user-defined beans override defaults.

## Choosing an Injection Style

| Style | Immutable (`final`) | Testable with plain `new` | Best for | Main caveat |
|---|---|---|---|---|
| Constructor | Yes | Yes | Mandatory dependencies (default) | Long constructors hint at a class doing too much |
| Setter | No | Yes | Optional / reconfigurable dependencies | Bean can be used before it is fully wired |
| Field | No | No | Prototypes, legacy code | Hides dependencies, hard to test, couples code to Spring |

Rules of thumb:

* Prefer **constructor injection** for almost everything; it guarantees fully initialized, immutable beans.
* Use **setter injection** only for genuinely optional dependencies.
* Avoid **field injection** in production code.
* Reach for `@Primary` or `@Qualifier` only when multiple implementations of a type exist.
* Use `@Profile` for environment-specific beans and `@Bean` methods for third-party types.
* Use `@Lazy` for rarely used beans or to break a constructor cycle.

## The Container: `BeanFactory` vs `ApplicationContext`

`BeanFactory` is the base IoC container — a lazy bean registry. `ApplicationContext` extends it with event publication, message/i18n resolution, resource loading, and **eager pre-instantiation of singletons** at startup. The concrete workhorse is `DefaultListableBeanFactory`, which stores every `BeanDefinition` and resolves dependencies.

`AbstractApplicationContext.refresh()` is the canonical startup template:

1. `obtainFreshBeanFactory()` — load bean definitions.
2. `prepareBeanFactory()` — register environment and `*Aware` processors.
3. `invokeBeanFactoryPostProcessors()` — run `BeanFactoryPostProcessor`s (including `ConfigurationClassPostProcessor`, which parses `@Configuration`/`@Bean`).
4. `registerBeanPostProcessors()` — register `BeanPostProcessor`s (e.g. `AutowiredAnnotationBeanPostProcessor`).
5. `finishBeanFactoryInitialization()` — pre-instantiate non-lazy singletons.
6. `finishRefresh()` — publish `ContextRefreshedEvent`, start `Lifecycle` beans.

## Bean Creation Phases

`doCreateBean` runs three phases per bean:

1. **Instantiate** — pick a constructor (or factory method) and create the raw instance.
2. **Populate** — inject dependencies; `resolveDependency` matches by type, then narrows by `@Qualifier`/name/`@Primary`.
3. **Initialize** — `*Aware` callbacks -> `postProcessBeforeInitialization` -> `@PostConstruct`/`InitializingBean`/`init-method` -> `postProcessAfterInitialization` (where AOP proxies are usually created).

## Circular Dependencies — the Three-Level Cache

Spring resolves singleton cycles (setter/field only) with three maps:

* `singletonObjects` — fully initialized beans.
* `earlySingletonObjects` — early references exposed mid-creation.
* `singletonFactories` — `ObjectFactory`s that produce an early (possibly proxied) reference via `getEarlyBeanReference`.

When bean A needs B and B needs A, A's early factory is exposed before A is fully initialized, so B wires the early A and finishes, letting A complete. A **constructor** cycle cannot use this (the instance does not exist yet) and fails. Since Boot 2.6, circular references are **disallowed by default** (`spring.main.allow-circular-references=false`) — treat a cycle as a design smell.

## `@Configuration`: Full vs Lite Mode

* **Full** (`proxyBeanMethods = true`, default) — the class is CGLIB-enhanced, so calling one `@Bean` method from another returns the shared singleton.
* **Lite** (`proxyBeanMethods = false`) — no proxy: faster startup and native-friendly, but an inter-bean method call creates a *new* instance. Inject the dependency as a method parameter instead of calling the method.

## Advanced Injection Points

```java
// Optional / plural / lazy without null checks
private final ObjectProvider<AuditSink> sinks;

void record(Event e) {
    sinks.ifAvailable(sink -> sink.write(e));
    sinks.orderedStream().forEach(s -> s.write(e));
}
```

* `ObjectProvider<T>` — optional (`getIfAvailable`), plural (`orderedStream`), lazy resolution.
* `@Lookup` — method injection to obtain a fresh prototype from a singleton on each call.
* `FactoryBean<T>` — encapsulates complex construction; inject `&name` to get the factory itself.
* Generics are respected: `Converter<Order, Dto>` injects the correctly typed bean, and annotations can act as qualifiers.

## Ordering and Priority

`@Order`/`Ordered` sets order for collection injection and post-processor chains; `@Priority` additionally influences which single candidate wins autowiring when several match.

## AOT and Native Constraints

Boot 3 AOT pre-computes bean registrations, proxies, and reflection hints at build time. In native images you cannot register beans dynamically at runtime as freely as on the JVM; supply reflection/resource/proxy hints with a `RuntimeHintsRegistrar` (`@ImportRuntimeHints`) so the required metadata is available.

## Interview Q&A

**Q: Why is constructor injection preferred over field injection?**
A: It enforces mandatory dependencies at construction, allows `final` (immutable) fields, keeps dependencies explicit, and lets you unit-test the class with plain `new` — no reflection or running container required.

**Q: How does Spring resolve a circular dependency between two singletons?**
A: For setter/field injection it exposes an early reference through the three-level singleton cache, so the second bean receives a partially initialized reference. A pure constructor cycle cannot be resolved this way and fails unless one side is `@Lazy` or the design is refactored.

**Q: What is the difference between `BeanFactoryPostProcessor` and `BeanPostProcessor`?**
A: `BeanFactoryPostProcessor` modifies bean *definitions/metadata* before any bean is instantiated; `BeanPostProcessor` intercepts each bean *instance* around initialization.

**Q: What happens when two beans match an injection point and neither is primary?**
A: Spring throws `NoUniqueBeanDefinitionException`. Resolve it with `@Primary`, a `@Qualifier`, or by injecting a collection of all candidates.

**Q: What does `@Lazy` actually do?**
A: It injects a proxy and defers real bean creation until first use, lowering startup cost and optionally breaking a dependency cycle — at the risk of surfacing initialization errors later at runtime.

**Q: Why must prototype-scoped beans be used carefully?**
A: Spring does not manage their full lifecycle — `@PreDestroy` is never called — and injecting a prototype into a singleton yields a single instance unless you use a scoped proxy or `ObjectProvider`.

**Q: What is the difference between `BeanFactory` and `ApplicationContext`?**
A: `BeanFactory` is the base lazy container; `ApplicationContext` extends it with eager singleton instantiation, event publishing, i18n, and resource loading. Applications use `ApplicationContext`.

**Q: What does `proxyBeanMethods = true` do on `@Configuration`?**
A: It CGLIB-enhances the class so calling one `@Bean` method from another returns the shared singleton. `proxyBeanMethods = false` (lite) skips the proxy for faster, native-friendly startup, but inter-bean method calls create new instances — inject the dependency instead.

**Q: How do you inject all implementations of an interface and control their order?**
A: Inject `List<T>` (or `Map<String, T>` keyed by bean name) and order with `@Order`/`Ordered`. It is the idiomatic basis for strategy/plugin designs.

**Q: When would you use `ObjectProvider` or `@Lookup`?**
A: `ObjectProvider<T>` for optional (`getIfAvailable`), plural (`orderedStream`), or lazy resolution without null checks; `@Lookup` to obtain a fresh prototype from a singleton on each call.

**Q: What is a `FactoryBean`?**
A: A bean that produces another object via `getObject()`, used for complex construction. Injecting `name` gives the produced object; injecting `&name` gives the factory itself.

## Interview Notes

* Explain how `ApplicationContext` loads beans and the difference between `BeanFactoryPostProcessor` and `BeanPostProcessor`.
* Describe constructor vs setter vs field injection and why constructor injection is preferred.
* Know how Spring resolves circular references and how proxies affect scoped beans.
