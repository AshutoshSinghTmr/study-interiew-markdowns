# Java OOP and Class Design

## The Four Pillars

Java is a class-based, object-oriented language built on four principles:

* **Encapsulation** — bundle state with the behaviour that operates on it, and hide internals behind a stable API. Fields are usually `private`; access is mediated by methods so invariants are enforced in one place.
* **Inheritance** — an `is-a` relationship where a subclass reuses and specializes a superclass. Java allows single class inheritance but multiple interface inheritance.
* **Polymorphism** — one reference type, many runtime behaviours. Method calls dispatch to the actual object's overriding method at runtime (dynamic dispatch).
* **Abstraction** — expose *what* an object does, not *how*, via interfaces and abstract classes.

## Encapsulation and Access Modifiers

Access widens in this order: `private` → package-private (default) → `protected` → `public`.

| Modifier | Same class | Same package | Subclass (other pkg) | World |
| --- | --- | --- | --- | --- |
| `private` | ✅ | ❌ | ❌ | ❌ |
| default | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

Encapsulation is not just getters/setters — it means exposing behaviour, not raw state. Returning mutable internals (a `List`, a `Date`) breaks encapsulation unless you return a copy or unmodifiable view.

## Inheritance, Overriding, and Overloading

**Overriding** replaces a superclass method with the same signature; it is resolved at runtime by the object's actual type. **Overloading** is multiple methods with the same name but different parameter lists, resolved at compile time by the static types of the arguments.

```java
class Animal { String sound() { return "..."; } }
class Dog extends Animal {
    @Override String sound() { return "woof"; }   // override: runtime dispatch
}

Animal a = new Dog();
a.sound(); // "woof" — dynamic dispatch on the runtime type Dog
```

Rules for a valid override: same name and parameters, a covariant (same or narrower) return type, no broader checked exceptions, and access that is the same or wider. Always use `@Override` so the compiler catches accidental overloads.

`static` methods and fields are **hidden**, not overridden — they are resolved by the reference's compile-time type. Private and `final` methods cannot be overridden.

## Interfaces vs Abstract Classes

Both provide abstraction, but they answer different questions.

* An **interface** defines a capability (a contract) and supports multiple inheritance of type. Since Java 8 it can carry `default` and `static` methods; since Java 9, `private` helper methods. It holds no instance state (only `public static final` constants).
* An **abstract class** models a partially built type with shared state and constructors, using a single-inheritance `is-a` relationship.

```java
interface Payable {
    BigDecimal amount();
    default boolean isFree() { return amount().signum() == 0; } // default method
}

abstract class Employee {
    private final String name;          // shared state — only a class can hold this
    protected Employee(String name) { this.name = name; }
    abstract BigDecimal salary();       // subclasses must implement
}
```

Choose an interface for a role many unrelated types can play; choose an abstract class when subtypes share state and construction logic. When a class implements two interfaces with the same `default` method, the compiler forces you to override and disambiguate (optionally via `Interface.super.method()`).

## Composition Over Inheritance

Inheritance is tight coupling: subclasses depend on superclass implementation details and the hierarchy is fixed at compile time. Prefer **composition** — hold a collaborator and delegate — for flexible, testable designs.

```java
class Logger { void log(String msg) { /* ... */ } }

class OrderService {
    private final Logger logger;                 // composed, not inherited
    OrderService(Logger logger) { this.logger = logger; }
    void place() { logger.log("order placed"); } // delegation
}
```

The classic pitfall of inheritance is the fragile base class problem: a superclass change silently breaks subclasses. Composition also enables runtime behaviour swapping (the Strategy pattern) that inheritance cannot.

## Records (Java 16+)

A `record` is a transparent carrier for immutable data. The compiler generates a canonical constructor, `private final` fields, accessors, `equals`, `hashCode`, and `toString`.

```java
record Point(int x, int y) {
    Point {                                  // compact constructor: validation
        if (x < 0 || y < 0) throw new IllegalArgumentException("negative");
    }
    double distance(Point o) {
        return Math.hypot(x - o.x, y - o.y);
    }
}
```

Records are implicitly `final`, cannot extend a class (they extend `java.lang.Record`), and their components are final. They are ideal for DTOs, map keys, and value objects. You can add methods, static factories, and implement interfaces, but you cannot add instance fields beyond the components.

## Sealed Classes (Java 17+)

Sealing restricts which types may extend or implement a type, giving you a closed, known hierarchy — perfect for modelling algebraic data types and enabling exhaustive `switch`.

```java
sealed interface Shape permits Circle, Rectangle {}
record Circle(double r) implements Shape {}
record Rectangle(double w, double h) implements Shape {}

double area(Shape s) {
    return switch (s) {                       // no default needed — exhaustive
        case Circle c    -> Math.PI * c.r() * c.r();
        case Rectangle r -> r.w() * r.h();
    };
}
```

Each permitted subtype must be `final`, `sealed`, or `non-sealed`. Combined with records and pattern matching, sealed types let the compiler verify that a `switch` covers every case.

## Nested, Inner, Local, and Anonymous Classes

* **Static nested class** — a class scoped inside another with no reference to an enclosing instance. Use for helpers logically tied to the outer type.
* **Inner (non-static) class** — holds an implicit reference to the enclosing instance (`Outer.this`); it can therefore access the outer object's fields but can also leak it (a memory-leak risk).
* **Local class** — declared inside a method; captures effectively-final local variables.
* **Anonymous class** — a one-shot subclass/implementation created inline; largely superseded by lambdas for functional interfaces.

```java
Runnable r = new Runnable() {              // anonymous class
    @Override public void run() { System.out.println("run"); }
};
Runnable lambda = () -> System.out.println("run"); // lambda equivalent
```

## Constructors, Initialization Order, and `this`/`super`

Object construction follows a fixed order: static initializers (once, at class load) → superclass constructor → instance initializer blocks and field initializers in source order → the constructor body. A constructor may delegate with `this(...)` or invoke a superclass constructor with `super(...)`, and that call must be the first statement.

Calling an overridable method from a constructor is dangerous: the subclass override runs before the subclass fields are initialized, observing `null`/default values.

## Polymorphism in Depth

Dynamic dispatch is implemented via a per-class method table (vtable): the JVM looks up the actual type's method at call time. This is why an `Animal` reference to a `Dog` calls `Dog.sound()`. Fields, `static` methods, and `private` methods are **not** polymorphic — they bind to the compile-time type. Understanding this split (late binding for overridden instance methods, early binding for everything else) explains most "why did it call the parent version?" surprises.

## Interview Q&A

**Q: What is the difference between overloading and overriding?**
A: Overloading is compile-time (static) polymorphism — same name, different parameters, resolved by argument static types. Overriding is runtime polymorphism — same signature in a subclass, resolved by the object's actual type.

**Q: When would you choose an interface over an abstract class?**
A: Use an interface for a capability many unrelated types can implement and when you need multiple inheritance of type. Use an abstract class when subtypes share state, fields, or constructor logic in an `is-a` hierarchy.

**Q: What does a `record` generate, and what are its limits?**
A: A canonical constructor, `private final` fields, accessors, `equals`, `hashCode`, and `toString`. It is implicitly `final`, cannot extend a class, and cannot declare extra instance fields — it is a transparent immutable data carrier.

**Q: What problem do sealed classes solve?**
A: They restrict the set of permitted subtypes, producing a closed hierarchy. This enables exhaustive `switch` without a `default` and models fixed sets of variants safely.

**Q: Why prefer composition over inheritance?**
A: Composition avoids tight coupling and the fragile base class problem, allows runtime behaviour changes (Strategy), and is easier to test. Inheritance locks you into a fixed hierarchy and exposes superclass internals.

**Q: Can you override a `static` method?**
A: No. Static methods are hidden, not overridden — they bind to the compile-time (reference) type. Only instance methods participate in dynamic dispatch.

**Q: What is the object initialization order?**
A: Static initializers (once) → superclass constructor → instance field initializers and instance blocks in source order → constructor body.

**Q: Why is calling an overridable method from a constructor risky?**
A: The subclass override executes before the subclass's own fields are initialized, so it may see uninitialized (default/null) state.

**Q: What is the difference between a static nested class and an inner class?**
A: A static nested class has no reference to an enclosing instance; an inner (non-static) class holds an implicit `Outer.this` reference, so it can access outer state but can also prevent the outer instance from being garbage collected.

**Q: How does Java resolve a diamond of default methods?**
A: If two implemented interfaces provide the same default method, the class must override it; it can delegate explicitly with `InterfaceName.super.method()`.

## Interview Notes

* Explain the four pillars with a concrete example of each.
* Be precise about overriding rules (covariant return, no broader checked exceptions, wider-or-equal access) and use `@Override`.
* Contrast interface vs abstract class along state, multiple inheritance, and default methods.
* Know why composition beats inheritance and name the fragile base class problem.
* Describe what records generate and where sealed types + records + pattern matching combine for exhaustive `switch`.
* Recall the initialization order and the "don't call overridable methods in constructors" rule.
