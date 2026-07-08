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

## Interview Notes

* Explain the proxy model and why self-invocation bypasses advice.
* Know the five advice types and when `@Around` is required.
* Be able to write an `execution(...)` or `@annotation(...)` pointcut and order two aspects.
