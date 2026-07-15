# Java Annotations and Reflection

## What Annotations Are

Annotations are metadata attached to code elements (types, methods, fields, parameters, packages, modules). They do not change program semantics directly; tools, compilers, and frameworks read them and act. Spring, JUnit, JPA, and Jackson are all annotation-driven.

## Built-in and Meta-Annotations

Common built-ins: `@Override`, `@Deprecated`, `@SuppressWarnings`, `@FunctionalInterface`, `@SafeVarargs`.

Meta-annotations describe other annotations:

* `@Retention` — how long the annotation is kept: `SOURCE` (compiler only, e.g. `@Override`), `CLASS` (in bytecode, not at runtime — the default), `RUNTIME` (readable via reflection).
* `@Target` — which elements it may annotate (`TYPE`, `METHOD`, `FIELD`, `PARAMETER`, ...).
* `@Documented` — include in Javadoc.
* `@Inherited` — a class-level annotation is inherited by subclasses.
* `@Repeatable` — allow multiple instances on one element.

## Defining a Custom Annotation

```java
@Retention(RetentionPolicy.RUNTIME)   // must be RUNTIME to read via reflection
@Target(ElementType.METHOD)
public @interface Audited {
    String value() default "";        // elements look like methods
    boolean sensitive() default false;
}

@Audited(value = "transfer", sensitive = true)
public void transfer() { /* ... */ }
```

Elements may be primitives, `String`, `Class`, enums, other annotations, or arrays of these. `value` is special: `@Audited("x")` is shorthand when only `value` is set.

## Reading Annotations via Reflection

Only `RUNTIME`-retained annotations are visible at runtime.

```java
Method m = Service.class.getMethod("transfer");
if (m.isAnnotationPresent(Audited.class)) {
    Audited a = m.getAnnotation(Audited.class);
    System.out.println(a.value() + " sensitive=" + a.sensitive());
}
```

This is exactly how frameworks discover mapped endpoints, injection points, and test methods.

## The Reflection API

Reflection inspects and manipulates classes at runtime. Entry point is `Class<?>`, obtained via `obj.getClass()`, `Type.class`, or `Class.forName("com.x.Y")`.

```java
Class<?> c = Class.forName("com.example.User");
Object user = c.getDeclaredConstructor().newInstance();   // reflective creation
Field f = c.getDeclaredField("name");
f.setAccessible(true);                                     // bypass private (if allowed)
f.set(user, "Ada");
Method greet = c.getMethod("greet", String.class);
Object result = greet.invoke(user, "hi");
```

Key types: `Class`, `Field`, `Method`, `Constructor`, `Parameter`, `Modifier`. `getDeclaredX` returns all members declared on the class (including private); `getX` returns public members including inherited ones.

## `setAccessible` and Strong Encapsulation

`setAccessible(true)` disables Java access checks so reflection can touch private members. Since JMPS (Java 9+), this is restricted: a module must `open` a package for deep reflection, otherwise `setAccessible` throws `InaccessibleObjectException`. This is why frameworks ask you to `opens` packages in `module-info.java`. Reflection also bypasses the compiler, so it is unsafe (no compile-time checking) and a security-sensitive operation.

## Dynamic Proxies

`java.lang.reflect.Proxy` creates, at runtime, a class implementing given interfaces and routes every call through an `InvocationHandler`. This underpins AOP, mocking libraries, and Spring's JDK-proxy-based beans.

```java
interface Repo { String find(int id); }

Repo proxy = (Repo) Proxy.newProxyInstance(
    Repo.class.getClassLoader(),
    new Class<?>[]{ Repo.class },
    (p, method, args) -> {                 // InvocationHandler
        System.out.println("calling " + method.getName());
        return "row-" + args[0];
    });
proxy.find(7);   // prints "calling find", returns "row-7"
```

JDK dynamic proxies require an interface. To proxy a concrete class you need subclass-based proxies (CGLIB/ByteBuddy), which is why Spring falls back to those for classes without interfaces.

## Annotation Processing (Compile Time)

`SOURCE`-retained annotations can be consumed at compile time by annotation processors (`javax.annotation.processing.Processor`) to generate code — how Lombok, MapStruct, Dagger, and the Java `AutoService` work. This is compile-time metaprogramming with zero runtime reflection cost, in contrast to runtime annotation reading.

## Performance and Trade-offs

Reflection is far slower than direct calls: it does access checks, boxing, and defeats many JIT optimizations. Costs can be mitigated by caching `Method`/`Field` objects and using `MethodHandles`/`VarHandle` (from `java.lang.invoke`), which are more JIT-friendly and the modern, faster alternative. Reflection also breaks encapsulation, weakens type safety, and complicates refactoring and native-image (GraalVM) builds, which need explicit reflection configuration.

## `MethodHandles` and `VarHandle` (Modern Alternative)

`java.lang.invoke` provides `MethodHandle` (typed, signature-polymorphic, JIT-optimizable invocation) and `VarHandle` (typed access to fields/arrays with memory-ordering modes, replacing much of `sun.misc.Unsafe` and `AtomicXFieldUpdater`). Prefer these over classic reflection in performance-sensitive code.

## Interview Q&A

**Q: What are the three retention policies and when is each used?**
A: `SOURCE` (compiler/processor only, discarded — e.g. `@Override`), `CLASS` (kept in bytecode, not loaded at runtime — the default), and `RUNTIME` (available to reflection). Only `RUNTIME` annotations are readable via reflection.

**Q: What is reflection and what can it do?**
A: An API to inspect and manipulate classes, methods, and fields at runtime — creating instances, reading/writing fields, invoking methods, and reading annotations, even for members not known at compile time.

**Q: What does `setAccessible(true)` do and what restricts it now?**
A: It disables access checks to reach private members. Under JPMS it requires the target package to be `open`, otherwise it throws `InaccessibleObjectException`.

**Q: How do dynamic proxies work?**
A: `Proxy.newProxyInstance` synthesizes a class implementing given interfaces; all method calls are dispatched to an `InvocationHandler`. It powers AOP and mocking. Interface-only — concrete classes need CGLIB/ByteBuddy.

**Q: Difference between `getMethods()` and `getDeclaredMethods()`?**
A: `getMethods()` returns public methods including inherited ones; `getDeclaredMethods()` returns all methods declared directly on the class (any access) but not inherited.

**Q: Runtime reflection vs annotation processing?**
A: Annotation processing consumes annotations at compile time to generate code (Lombok, MapStruct) with no runtime cost; reflection reads `RUNTIME` annotations while the program runs, with performance overhead.

**Q: Why is reflection slow and how do you mitigate it?**
A: It performs access checks and boxing and blocks inlining. Cache reflective objects, or use `MethodHandles`/`VarHandle` which are typed and JIT-friendly.

**Q: What are the downsides of reflection?**
A: It breaks encapsulation, loses compile-time type safety, hurts performance, complicates refactoring, and needs explicit configuration for GraalVM native images.

**Q: Can you read a `SOURCE`-retained annotation with reflection?**
A: No — it is discarded by the compiler and never present at runtime.

**Q: What are `MethodHandle` and `VarHandle`?**
A: `java.lang.invoke` primitives: `MethodHandle` is fast, typed, signature-polymorphic invocation; `VarHandle` gives typed field/array access with memory-ordering semantics, replacing much of `Unsafe`.

## Interview Notes

* Explain annotations as metadata and the role of `@Retention`/`@Target`.
* Stress that only `RUNTIME` retention is reflectively readable.
* Walk through reflective instantiation, field/method access, and `getX` vs `getDeclaredX`.
* Explain `setAccessible` and the JPMS `opens` requirement.
* Describe dynamic proxies (interface-based) vs CGLIB and where Spring uses each.
* Contrast compile-time annotation processing with runtime reflection, and cite `MethodHandles`/`VarHandle` as the faster modern option.
