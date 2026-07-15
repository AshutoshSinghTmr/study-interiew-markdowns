# Java Functional Programming and Lambdas

## Lambdas and Functional Interfaces

A lambda is an anonymous function — a concise implementation of a **functional interface** (an interface with exactly one abstract method, the SAM). The compiler infers the target type from context.

```java
Runnable r = () -> System.out.println("run");
Comparator<String> byLen = (a, b) -> Integer.compare(a.length(), b.length());
```

`@FunctionalInterface` documents and enforces the single-abstract-method rule (default/static methods don't count). A lambda can only be assigned where a functional interface is expected — its "target type."

## The `java.util.function` Toolkit

The standard functional interfaces cover the common shapes:

| Interface | Abstract method | Shape |
| --- | --- | --- |
| `Supplier<T>` | `T get()` | () → T |
| `Consumer<T>` | `void accept(T)` | T → () |
| `Function<T,R>` | `R apply(T)` | T → R |
| `Predicate<T>` | `boolean test(T)` | T → boolean |
| `UnaryOperator<T>` | `T apply(T)` | T → T |
| `BinaryOperator<T>` | `T apply(T,T)` | (T,T) → T |
| `BiFunction<T,U,R>` | `R apply(T,U)` | (T,U) → R |

Primitive specializations (`IntFunction`, `ToIntFunction`, `IntPredicate`, ...) avoid boxing on hot paths.

## Method References

A method reference is shorthand for a lambda that just calls an existing method. Four kinds:

```java
Function<String,Integer> parse = Integer::parseInt;   // static
Supplier<List<String>> make = ArrayList::new;         // constructor
Function<String,Integer> len = String::length;        // instance of arbitrary object
Consumer<String> print = System.out::println;         // instance of a specific object
```

They improve readability when the lambda body is a single call, and are usually as efficient as the equivalent lambda.

## Capture and Effectively Final

A lambda may capture local variables, but only if they are **effectively final** (assigned once, never mutated). This is because a captured local is copied into the lambda instance; allowing mutation would create confusing aliasing across threads. Instance and static fields *can* be mutated (they're captured via the enclosing `this`), which is also why capturing `this` can leak the enclosing object.

```java
int factor = 3;                  // effectively final
Function<Integer,Integer> f = x -> x * factor;  // captures a copy
// factor = 4;                   // would make the above not compile
```

Non-capturing lambdas (no free variables) are cached as singletons by the runtime; capturing lambdas allocate per capture.

## Lambdas vs Anonymous Classes

Lambdas are not just syntactic sugar for anonymous classes:

* `this` in a lambda refers to the **enclosing** instance; in an anonymous class it refers to the anonymous instance.
* Lambdas don't introduce a new scope for names (no shadowing surprises).
* Lambdas compile to `invokedynamic` + `LambdaMetafactory` rather than a separate `.class` file per lambda, reducing class-count and enabling runtime optimization.

## How Lambdas Compile: `invokedynamic`

Instead of generating an anonymous inner class at compile time, `javac` emits an `invokedynamic` instruction whose bootstrap is `LambdaMetafactory`. On first execution it spins up an implementation class (or reuses a constant for non-capturing lambdas) and links the call site. This keeps class files small, defers the strategy to runtime, and lets the JVM optimize.

## Function Composition

Functional interfaces provide default methods to combine behaviours:

```java
Function<Integer,Integer> plus1 = x -> x + 1;
Function<Integer,Integer> times2 = x -> x * 2;
plus1.andThen(times2).apply(3);   // (3+1)*2 = 8
plus1.compose(times2).apply(3);   // (3*2)+1 = 7

Predicate<String> nonEmpty = s -> !s.isBlank();
Predicate<String> shortStr = s -> s.length() < 10;
nonEmpty.and(shortStr).negate();  // combinators
```

`Comparator` composition (`comparing`, `thenComparing`, `reversed`) is the most common real-world use.

## Higher-Order Functions and Closures

Methods that take or return functions are higher-order. Returning a function that captures state creates a closure — a function bundled with its captured environment.

```java
static Function<Integer,Integer> adder(int n) {
    return x -> x + n;            // closes over n
}
adder(10).apply(5);              // 15
```

## Practical Guidance

* Prefer the standard `java.util.function` interfaces over custom ones for interoperability.
* Keep lambdas short; extract to a named method (and use a method reference) when logic grows or needs testing.
* Avoid side effects in lambdas passed to streams — they should be pure for correctness under parallelism.
* Use primitive specializations in hot code to avoid autoboxing.
* Beware capturing `this` in long-lived lambdas — it can retain the enclosing object.

## Interview Q&A

**Q: What is a functional interface?**
A: An interface with exactly one abstract method (a SAM type), optionally with default/static methods. It is the target type a lambda or method reference can implement; `@FunctionalInterface` enforces the rule.

**Q: Why must captured locals be effectively final?**
A: Captured locals are copied into the lambda; permitting mutation would create ambiguous aliasing (especially across threads). Fields can still be mutated because they're reached through `this`.

**Q: How do lambdas differ from anonymous classes?**
A: `this` refers to the enclosing instance (not a new object), no new naming scope, and they compile via `invokedynamic`/`LambdaMetafactory` instead of a generated class file — often with a cached singleton for non-capturing lambdas.

**Q: What are the four kinds of method reference?**
A: Static (`Integer::parseInt`), constructor (`ArrayList::new`), instance method of an arbitrary object of a type (`String::length`), and instance method of a specific object (`out::println`).

**Q: Name the core functional interfaces and their methods.**
A: `Supplier.get`, `Consumer.accept`, `Function.apply`, `Predicate.test`, `UnaryOperator`/`BinaryOperator.apply`, `BiFunction.apply`.

**Q: How are lambdas implemented under the hood?**
A: `javac` emits `invokedynamic`; at first run `LambdaMetafactory` links a synthesized implementation. Non-capturing lambdas can be reused as singletons; no per-lambda class file is generated at compile time.

**Q: What is a closure?**
A: A function that captures and carries variables from its defining scope, e.g. a returned lambda that references a parameter of the enclosing method.

**Q: `andThen` vs `compose`?**
A: `f.andThen(g)` applies `f` then `g`; `f.compose(g)` applies `g` then `f`.

**Q: Why avoid side effects in stream lambdas?**
A: They break correctness under parallel execution and hurt readability; stream operations should be pure/stateless.

**Q: Why use primitive functional interfaces like `IntPredicate`?**
A: To avoid autoboxing overhead on performance-sensitive paths.

## Interview Notes

* Define a functional interface and the SAM rule; know `@FunctionalInterface`.
* Recall the standard interfaces and their single methods, plus method-reference kinds.
* Explain effectively-final capture and how it differs for fields vs locals.
* Contrast lambdas with anonymous classes (`this`, scope, `invokedynamic`).
* Explain composition (`andThen`/`compose`, comparator chaining) and closures.
* Note purity for parallelism and primitive specializations for performance.
