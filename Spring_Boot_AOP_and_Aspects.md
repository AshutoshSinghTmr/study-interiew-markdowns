# Spring Boot AOP and Aspects

## Why AOP

Aspect-Oriented Programming modularizes cross-cutting concerns — logging, metrics, security, caching, transactions, retries — that would otherwise be scattered across many methods. Spring implements AOP with runtime proxies, and most Boot features (`@Transactional`, `@Cacheable`, `@Async`, method security) are built on it.

## Core Concepts

* **Aspect**: a module bundling a cross-cutting concern (a `@Aspect` class).
* **Join point**: a point in execution where advice can apply — in Spring AOP, always a method execution.
* **Pointcut**: an expression selecting join points.
* **Advice**: the action taken at a join point (`@Before`, `@Around`, ...).
* **Target**: the advised bean.
* **Proxy**: the object Spring hands out in place of the target to apply advice.
* **Weaving**: linking aspects with target code — in Spring, at runtime via proxies.

## Proxy-based AOP in Spring

Spring AOP is proxy-based, so advice applies only to calls that go **through the proxy** (external calls into the bean). It advises Spring-managed beans and method executions only — not field access or constructors.

### JDK dynamic proxies vs CGLIB

* If the target implements at least one interface, Spring can create a **JDK dynamic proxy** of that interface.
* Otherwise (or when `proxyTargetClass = true`), Spring generates a **CGLIB** subclass proxy. Spring Boot defaults to CGLIB (`spring.aop.proxy-target-class=true`).

### Enabling aspects

`@EnableAspectJAutoProxy` activates `@AspectJ`-style aspects; Spring Boot auto-configures this when `spring-boot-starter-aop` (AspectJ + Spring AOP) is on the classpath. Note this is **Spring AOP using AspectJ annotations and pointcut syntax**, not full AspectJ weaving.

## Advice Types

* `@Before` — runs before the method.
* `@AfterReturning` — runs after a successful return (can read the returned value).
* `@AfterThrowing` — runs when the method throws (can read the exception).
* `@After` — runs regardless of outcome (finally-style).
* `@Around` — wraps the method; must call `ProceedingJoinPoint.proceed()` and can alter arguments, return value, or short-circuit the call. The most powerful — use it only when you need control over invocation.

## Pointcut Expressions

Common designators, combinable with `&&`, `||`, `!`:

* `execution(* com.acme.service..*(..))` — method execution matching a signature.
* `within(com.acme.service..*)` — any join point within given types.
* `@annotation(com.acme.Audited)` — methods carrying an annotation.
* `bean(*Service)` — methods on beans whose name matches.
* `args(..)` / `@args(..)` — match on argument types/annotations.

Reusable named pointcuts keep expressions DRY:

```java
@Pointcut("@annotation(com.acme.Audited)")
void audited() {}
```

## Writing an Aspect

```java
@Aspect
@Component
public class TimingAspect {

    private static final Logger log = LoggerFactory.getLogger(TimingAspect.class);

    @Around("execution(* com.acme..*Service.*(..))")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            log.info("{} took {} ms", pjp.getSignature().toShortString(), ms);
        }
    }
}
```

An annotation-driven aspect targeting only `@Audited` methods:

```java
@Aspect
@Component
public class AuditAspect {

    @AfterReturning(pointcut = "@annotation(audited)", returning = "result")
    public void record(JoinPoint jp, Audited audited, Object result) {
        // publish an audit event for jp.getSignature()
    }
}
```

## Ordering Aspects

When multiple aspects advise the same join point, control precedence with `@Order` or `Ordered`. Lower value = higher precedence = outermost wrapper. This matters when combining concerns: for example, put retry **outside** transactions so each retry gets a fresh transaction.

## Self-invocation Limitation

Because advice runs on the proxy, an internal call (`this.method()`) bypasses it, so `@Transactional`, `@Cacheable`, `@Async`, etc. on the self-invoked method do nothing. Fixes:

* move the advised method to another bean;
* inject the bean into itself and call through the injected reference;
* use `AopContext.currentProxy()` (requires `exposeProxy = true`);
* switch to AspectJ load-time or compile-time weaving.

## Spring AOP vs Full AspectJ

* **Spring AOP**: proxy-based, method-execution join points only, Spring beans only. Simple, no special build steps — covers most needs.
* **AspectJ weaving** (compile-time or load-time): weaves bytecode, supports field access, constructors, non-Spring objects, and `this`-calls. Needed for finer-grained or non-proxyable concerns, at the cost of build/agent setup.

## How Spring Creates the Proxy

`AnnotationAwareAspectJAutoProxyCreator` — a `BeanPostProcessor` — inspects every bean during initialization for matching **`Advisor`s** (an `Advisor` = `Pointcut` + `Advice`; each `@Aspect` advice method becomes an advisor). If at least one advisor matches, the bean is wrapped in a proxy in `postProcessAfterInitialization`. A JDK dynamic proxy is used when the target implements an interface; otherwise a CGLIB subclass is generated (or always, with `proxyTargetClass=true`). `final` classes/methods and `private` methods cannot be proxied.

## Pointcut Expression Language

AspectJ designators supported by Spring AOP:

* `execution(...)` — the main one: method execution matching a signature.
* `within(type)` / `@within(annotation)` — join points inside a type / a type bearing an annotation.
* `@annotation(A)` — methods annotated with `A`.
* `target(T)` / `this(T)` — the target/proxy is an instance of `T`.
* `args(...)` / `@args(...)` — argument types / argument annotations.
* `bean(name)` — Spring-specific, by bean name.

An `execution` pattern reads `modifiers? ret-type declaring-type?.method(params) throws?`, e.g. `execution(public * com.acme..*Service.*(..))`.

## The Invocation Chain

At call time Spring builds a `ReflectiveMethodInvocation` — a chain of `MethodInterceptor`s (each advice adapted to an interceptor). Calling `proceed()` advances the chain until the target runs, then unwinds. `@Around` maps directly to a `MethodInterceptor`; the annotation advices are adapted onto the same chain.

## Advice Precedence

For a single join point advised by multiple aspects, precedence follows `@Order`/`Ordered` (lower = outer). Within one aspect the order is deterministic: the `@Around`/`@Before` of a higher-precedence aspect wrap those of a lower one, and on the way out its `@AfterReturning`/`@After` run last. This is why retry (outer) should wrap a transaction (inner).

## Introductions

`@DeclareParents` (introductions) add a new interface and a default implementation to targets — a mixin — letting you attach state or behavior (e.g. an `Auditable` capability) without editing the target classes.

## Weaving Modes

| Mode | When woven | Can advise |
| --- | --- | --- |
| Spring AOP | Runtime (proxy) | `public` method executions on Spring beans |
| AspectJ CTW | Compile time (`ajc`) | Fields, constructors, any object |
| AspectJ LTW | Class load (`-javaagent`) | Same as CTW, no build change |

Enable load-time weaving with `@EnableLoadTimeWeaving` or a Java agent. LTW/CTW also fix **self-invocation**, because advice is woven into the bytecode instead of applied by a proxy.

## Performance and Pitfalls

Per-call proxy overhead is small, but deep advisor chains on hot paths add up — avoid advising tight inner-loop methods. Recap of proxy limits: self-invocation bypasses advice (use `AopContext.currentProxy()` with `exposeProxy=true`, self-injection, or LTW); only `public` method executions are advised; `final`/`private`/static methods are not. When combining `@Transactional`, `@Cacheable`, and `@Retryable`, set explicit `@Order` so the semantics (retry -> transaction -> cache) are correct.

## Interview Q&A

**Q: Is Spring AOP the same as AspectJ?**
A: No. Spring AOP is a proxy-based subset that reuses AspectJ's annotations and pointcut language but only advises Spring-bean method executions at runtime. Full AspectJ weaves bytecode and can advise field access, constructors, and non-Spring objects.

**Q: When does Spring use a JDK proxy versus a CGLIB proxy?**
A: A JDK dynamic proxy when the target implements an interface; a CGLIB subclass proxy otherwise or when `proxyTargetClass=true` (Spring Boot's default).

**Q: Why does `@Transactional`/`@Cacheable` sometimes "not work"?**
A: Usually self-invocation — the method is called from within the same class, bypassing the proxy — or the method isn't `public`/isn't a Spring bean.

**Q: What is the difference between `@Around` and the other advice types?**
A: `@Around` wraps the invocation and must call `proceed()`; it can change arguments, alter or replace the return value, handle exceptions, or skip the call entirely. The others are notification-only around a fixed point.

**Q: How do you control which aspect runs first?**
A: With `@Order`/`Ordered` on the aspects — lower values wrap the outer layers. It's essential for correct composition (e.g. retry outside transaction, security before business logic).

**Q: What kinds of join points can Spring AOP advise?**
A: Only method executions on Spring-managed beans. Constructors, static methods, final methods, and field access are not advisable with proxy-based AOP.

**Q: What is an `Advisor` and how does it relate to pointcut and advice?**
A: An `Advisor` pairs a `Pointcut` (where) with an `Advice` (what). The auto-proxy creator collects matching advisors for each bean and wraps it in a proxy.

**Q: How is the advice chain executed at call time?**
A: Spring builds a `ReflectiveMethodInvocation` — a chain of `MethodInterceptor`s. Each `proceed()` advances to the next interceptor until the target runs, then the stack unwinds.

**Q: What is an introduction?**
A: `@DeclareParents` adds a new interface and a default implementation (a mixin) to targets, attaching behavior or state without modifying the classes.

**Q: When do you need AspectJ weaving instead of Spring AOP?**
A: When you must advise fields, constructors, `final`/`private` methods, non-Spring objects, or self-invoked calls — use compile-time or load-time weaving (`@EnableLoadTimeWeaving`).

**Q: How do you invoke an advised method on the same bean and still get the advice?**
A: Call through the proxy: enable `exposeProxy=true` and use `((MyType) AopContext.currentProxy()).method()`, self-inject the bean, or switch to AspectJ weaving.

## Interview Notes

* Explain the proxy model and why self-invocation bypasses advice.
* Know the five advice types and when `@Around` is required.
* Be able to write an `execution(...)` or `@annotation(...)` pointcut and order two aspects.
