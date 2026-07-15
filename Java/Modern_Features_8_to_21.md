# Java Modern Features (8 to 21)

## Release Cadence and LTS

Since Java 9, a new version ships every six months, with a **Long-Term Support (LTS)** release every few years: **8, 11, 17, 21**. Most shops run an LTS. Knowing which feature landed where is a common interview filter, so this file groups the highlights and shows the modern idioms.

## Java 8 — The Functional Revolution

The most impactful release for everyday code:

* **Lambdas** and **method references** (`x -> x + 1`, `String::length`).
* **Streams API** for declarative data processing.
* **`Optional<T>`** to model absence explicitly.
* **Default/static interface methods**.
* **`java.time`** (JSR-310) date/time API.
* `CompletableFuture`, `Collectors`, `forEach`.

```java
Optional.ofNullable(user)
    .map(User::email)
    .filter(e -> e.contains("@"))
    .orElse("none");
```

## Java 9–10 — Modules and `var`

* **JPMS (Project Jigsaw, Java 9)** — the module system with `module-info.java`.
* **Collection factory methods** — `List.of`, `Set.of`, `Map.of` (immutable).
* **Private interface methods**; `Stream` additions (`takeWhile`, `dropWhile`, `iterate` with predicate).
* **`var`** local variable type inference (Java 10) — infers the static type from the initializer, reducing boilerplate without losing type safety.

```java
var users = new ArrayList<User>();      // inferred ArrayList<User>
for (var e : map.entrySet()) { /* ... */ }
```

`var` is only for locals with an initializer (not fields, params, or `null`); the type is still static and fixed.

## Java 11 (LTS) — Modern Baseline

* **`String` methods** — `isBlank`, `strip`, `lines`, `repeat`.
* **`Files.readString`/`writeString`**.
* **HTTP Client** (`java.net.http`) standardized — async, HTTP/2.
* **`var` in lambda parameters**; running a single source file directly (`java File.java`).

## Java 14–16 — Records, Pattern Matching, Text Blocks

* **`switch` expressions** (standard in 14) — arrow form returns a value, no fall-through.
* **Records** (16) — transparent immutable data carriers.
* **Pattern matching for `instanceof`** (16) — test and bind in one step.
* **Text blocks** (15) — multi-line string literals.
* **Helpful NullPointerExceptions** (14) — messages name the exact null variable.

```java
// switch expression
String size = switch (day) {
    case SATURDAY, SUNDAY -> "weekend";
    default -> "weekday";
};

// pattern matching for instanceof
if (obj instanceof String s && !s.isBlank()) {
    use(s);                     // s is in scope, already cast
}
```

## Java 17 (LTS) — Sealed Types

* **Sealed classes/interfaces** — restrict the permitted subtypes for closed hierarchies.
* Pattern matching maturing; the previous LTS most projects target.

```java
sealed interface Shape permits Circle, Square {}
```

## Java 21 (LTS) — The Current Landmark

* **Virtual threads** (JEP 444) — lightweight threads for massive I/O concurrency.
* **Pattern matching for `switch`** (JEP 441) — match on type patterns with exhaustiveness and guards.
* **Record patterns** (JEP 440) — destructure records directly in patterns.
* **Sequenced collections** — `SequencedCollection`/`Map` with `getFirst`/`getLast`/`reversed`.
* Preview: **structured concurrency**, **scoped values**, string templates.

```java
sealed interface Shape permits Circle, Rectangle {}
record Circle(double r) implements Shape {}
record Rectangle(double w, double h) implements Shape {}

double area(Shape s) {
    return switch (s) {                                  // exhaustive, no default
        case Circle(double r)          -> Math.PI * r * r;   // record pattern
        case Rectangle(double w, double h) -> w * h;
    };
}

Object o = "hi";
String desc = switch (o) {
    case Integer i when i > 0 -> "positive int";        // guarded pattern
    case String str            -> "string of " + str.length();
    default                    -> "other";
};
```

## `Optional` Idioms and Anti-Patterns

`Optional` communicates "maybe absent" for **return types**. Use `map`, `filter`, `orElse`, `orElseGet`, `orElseThrow`, and `ifPresentOrElse`. Anti-patterns: `Optional` fields or method parameters, and calling `get()` without checking. Use `orElseGet` (lazy) over `orElse` (eager) when the fallback is expensive.

## Putting It Together

Records + sealed types + pattern matching form Java's answer to **algebraic data types**: model a closed set of variants as sealed records, then process them with an exhaustive `switch` the compiler verifies — concise, type-safe, and refactor-friendly. Combined with virtual threads, modern Java writes clearer concurrent and data-oriented code than the Java 8 style.

## Interview Q&A

**Q: Which versions are LTS?**
A: Java 8, 11, 17, and 21. New feature releases ship every six months; LTS releases are the ones most organizations run in production.

**Q: What did Java 8 introduce?**
A: Lambdas, the Streams API, `Optional`, default/static interface methods, `java.time`, and `CompletableFuture` — the functional-programming shift.

**Q: What is `var` and where can't you use it?**
A: Local variable type inference (Java 10). The type is inferred from the initializer but remains static. Not allowed for fields, method parameters, return types, or without an initializer/`null`.

**Q: What are records and what do they generate?**
A: Immutable data carriers (Java 16) that auto-generate constructor, accessors, `equals`, `hashCode`, and `toString`; they're implicitly final and can't extend classes.

**Q: What are sealed classes for?**
A: Restricting which types may extend/implement a type (Java 17), enabling closed hierarchies and exhaustive `switch` without a default.

**Q: What is pattern matching for `switch` and record patterns?**
A: Java 21 features letting `switch` match on type patterns (with guards and exhaustiveness) and destructure records inline, e.g. `case Circle(double r) ->`.

**Q: What's the difference between switch statements and switch expressions?**
A: Switch expressions (arrow form) return a value, don't fall through, and can be exhaustive; statements use `case:`/`break` with fall-through risk.

**Q: What are the headline Java 21 features?**
A: Virtual threads, pattern matching for `switch`, record patterns, and sequenced collections (plus preview structured concurrency and scoped values).

**Q: `orElse` vs `orElseGet`?**
A: `orElse(value)` always evaluates its argument; `orElseGet(supplier)` evaluates lazily only when the `Optional` is empty — preferred for expensive fallbacks.

**Q: What are text blocks?**
A: Multi-line string literals (Java 15) delimited by `"""` that preserve formatting and reduce escaping, useful for JSON/SQL/HTML.

## Interview Notes

* Map key features to versions: 8 (lambdas/streams/Optional/java.time), 10 (`var`), 11 (LTS, HTTP client, String methods), 16 (records, `instanceof` patterns), 17 (sealed), 21 (virtual threads, switch patterns, record patterns).
* Know `var` constraints and that types stay static.
* Explain records + sealed + pattern matching as algebraic data types with exhaustive `switch`.
* Contrast switch expressions vs statements.
* Use `Optional` for returns only; prefer `orElseGet` for costly fallbacks.
