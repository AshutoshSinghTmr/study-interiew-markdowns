# Spring Boot Dependency Injection

## Core DI Concepts

Spring Boot builds on Spring Framework dependency injection. The `ApplicationContext` is the IoC container that manages bean definitions, bean creation, dependency resolution, and lifecycle callbacks.

### Bean definition and registration

Spring registers beans through:

* classpath scanning of `@Component`, `@Service`, `@Repository`, and `@Controller`
* `@Configuration` classes and `@Bean` methods
* XML or Java config imports
* auto-configuration via `spring.factories`

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

This is the most reliable style and is fully compatible with Spring’s circular reference detection when combined with `@Lazy`.

### Setter injection

Use setter injection for optional or replaceable dependencies. Setters are called after bean instantiation, allowing property configuration after construction.

### Field injection

Field injection uses reflection and is discouraged for production code because it hides dependencies and complicates testing. It is mainly useful for quick prototypes or legacy code.

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

### `@ConfigurationProperties` binding

A dedicated binder binds external properties to POJOs. `@ConstructorBinding` creates immutable configuration objects that are validated at startup.

### `@Import`, `ImportSelector`, and `DeferredImportSelector`

`@Import` registers additional configuration classes. `ImportSelector` chooses imports based on classpath or environment. `DeferredImportSelector` delays imports until all regular configuration classes are processed.

### Auto-configuration conditions

Spring Boot auto-configuration uses `@ConditionalOnBean` and `@ConditionalOnMissingBean` to let user-defined beans override defaults.

## Interview Notes

* Explain how `ApplicationContext` loads beans and the difference between `BeanFactoryPostProcessor` and `BeanPostProcessor`.
* Describe constructor vs setter vs field injection and why constructor injection is preferred.
* Know how Spring resolves circular references and how proxies affect scoped beans.
