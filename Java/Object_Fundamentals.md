# Java Object Fundamentals

## The `Object` Root

Every class implicitly extends `java.lang.Object`, inheriting a small but critical API: `equals`, `hashCode`, `toString`, `getClass`, `clone`, `finalize` (deprecated for removal), and the concurrency primitives `wait`/`notify`/`notifyAll`. Interviewers probe these because getting `equals`/`hashCode` wrong silently breaks hash-based collections.

## Identity vs Equality

`==` compares references (identity) for objects and values for primitives. `equals` compares logical equality as defined by the class. By default `Object.equals` is identity (`==`), so unless overridden, two distinct objects are never equal.

```java
String a = new String("hi");
String b = new String("hi");
a == b;        // false — different objects
a.equals(b);   // true  — same content
```

## The `equals` Contract

A correct `equals` must be:

* **reflexive** — `x.equals(x)` is true;
* **symmetric** — `x.equals(y) == y.equals(x)`;
* **transitive** — if `x.equals(y)` and `y.equals(z)` then `x.equals(z)`;
* **consistent** — repeated calls return the same result absent state changes;
* **non-null** — `x.equals(null)` is false.

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;                 // fast path
    if (!(o instanceof Money m)) return false;  // type check + pattern bind
    return amount == m.amount && currency.equals(m.currency);
}
```

The symmetry trap: comparing across an inheritance boundary (a subclass adding a field) usually violates symmetry or transitivity. The common fixes are to make the class `final`, use `getClass()` comparison instead of `instanceof`, or favour composition. Records generate a correct component-wise `equals` for you.

## The `hashCode` Contract

`hashCode` must be consistent with `equals`: **equal objects must have equal hash codes.** Unequal objects *may* share a hash code (a collision) but ideally spread out. Violating this breaks `HashMap`/`HashSet`: an object stored under one hash cannot be found if its hash changes or disagrees with `equals`.

```java
@Override
public int hashCode() {
    return Objects.hash(amount, currency);  // combines fields, null-safe
}
```

Always override `hashCode` whenever you override `equals`, using the **same fields**. For a single field prefer `Objects.hashCode(field)`; for several, `Objects.hash(...)` (note it boxes into an array, so hot paths may hand-roll `31 * result + field`).

## Mutable Keys Pitfall

If you use a mutable object as a `HashMap` key and then mutate a field that participates in `hashCode`, the entry becomes unreachable — it lives in the bucket for its old hash. This is why immutable value objects (or records) make the best keys.

## `toString`

Override `toString` to return a concise, informative representation for logs and debugging. Records and Lombok generate it; otherwise include the class name and key fields. Avoid leaking secrets (passwords, tokens) in `toString`.

## `Comparable` vs `Comparator`

* `Comparable<T>` defines a type's **natural ordering** via `compareTo`; the type owns the ordering (e.g. `Integer`, `String`).
* `Comparator<T>` is an **external** ordering strategy, letting you sort the same type many ways without touching it.

```java
record Person(String name, int age) {}

people.sort(Comparator.comparingInt(Person::age)
                      .thenComparing(Person::name));       // multi-key
people.sort(Comparator.comparing(Person::name).reversed());
```

`compareTo`/`compare` must return negative/zero/positive and should be **consistent with `equals`** (`compareTo == 0` iff `equals`) — otherwise sorted sets and maps like `TreeSet`/`TreeMap` behave surprisingly, because they use comparison, not `equals`, to decide membership. Never implement comparison with `a - b` on ints (overflow); use `Integer.compare(a, b)`.

## Immutability

An immutable object cannot change after construction. Recipe:

1. make the class `final` (or use a record);
2. make all fields `private final`;
3. do not expose setters;
4. defensively copy mutable inputs in the constructor and mutable outputs in getters.

```java
public final class Period {
    private final Date start;                 // Date is mutable — copy it
    public Period(Date start) { this.start = new Date(start.getTime()); }
    public Date getStart() { return new Date(start.getTime()); } // copy out
}
```

Immutable objects are inherently thread-safe, cacheable, and safe as map keys. Prefer immutable types (`java.time`, records, `String`) precisely because they remove whole classes of bugs.

## Defensive Copying

Returning or storing a reference to a caller-supplied mutable object lets external code mutate your internals, breaking encapsulation and invariants. Copy on the way in *and* out. For collections, wrap with `List.copyOf(...)` (an unmodifiable copy) or `Collections.unmodifiableList(new ArrayList<>(src))`.

## Cloning and Copy Strategies

`Object.clone` is widely considered broken: it requires implementing `Cloneable` (a marker interface), performs a **shallow** copy (nested mutable objects are shared), bypasses constructors, and interacts awkwardly with `final` fields. Prefer alternatives:

* a **copy constructor**: `new ArrayList<>(source)`;
* a **static factory**: `List.copyOf(source)`;
* for deep copies, copy each mutable field explicitly or serialize/deserialize (slow).

Shallow vs deep matters: a shallow copy shares referenced sub-objects, so mutating one copy's inner object affects the other.

## Boxing, Caching, and `equals`

Autoboxing wraps primitives in objects. The Integer cache holds `-128..127`, so `==` on small boxed values can appear to work and then fail outside the range — always compare wrapper objects with `equals`, not `==`.

```java
Integer x = 127, y = 127;   x == y; // true  (cached)
Integer p = 128, q = 128;   p == q; // false (new objects)
```

## Interview Q&A

**Q: What is the relationship between `equals` and `hashCode`?**
A: If two objects are `equals`, they must have the same `hashCode`. The reverse need not hold (collisions allowed). Breaking this makes objects unfindable in hash-based collections.

**Q: What are the five clauses of the `equals` contract?**
A: Reflexive, symmetric, transitive, consistent, and non-null.

**Q: Why can subclassing break `equals`?**
A: A subclass that adds a significant field usually violates symmetry or transitivity when compared with its superclass. Fixes: make the class final, use `getClass()` comparison, or use composition.

**Q: What breaks if you mutate a field used in `hashCode` after using the object as a map key?**
A: The entry stays in the bucket for the old hash, so lookups with the mutated object fail — the entry effectively leaks. Use immutable keys.

**Q: `Comparable` vs `Comparator`?**
A: `Comparable` defines a single natural ordering inside the type (`compareTo`); `Comparator` is an external, swappable ordering (`compare`), composable with `thenComparing`/`reversed`.

**Q: Why should `compareTo` be consistent with `equals`?**
A: Sorted collections (`TreeSet`/`TreeMap`) decide equality by comparison result, not `equals`. Inconsistency causes "duplicate" or missing elements.

**Q: How do you make a class immutable?**
A: Make it `final`, all fields `private final`, no setters, and defensively copy mutable inputs/outputs — or use a `record` for the common case.

**Q: What is defensive copying and why does it matter?**
A: Copying mutable objects on input and output so external code cannot mutate your internals. It preserves encapsulation and invariants (especially for `Date`, arrays, and collections).

**Q: Why is `Object.clone` discouraged?**
A: It needs `Cloneable`, does a shallow copy, bypasses constructors, and is error-prone. Prefer copy constructors or static factories.

**Q: Why can `==` on `Integer` sometimes return true and sometimes false?**
A: The Integer cache reuses objects for `-128..127`, so `==` is true there but false for larger values. Always use `equals` for wrappers.

## Interview Notes

* State the `equals`/`hashCode` contracts precisely and always override them together with the same fields.
* Explain the subclass symmetry trap and its fixes.
* Contrast `Comparable` vs `Comparator` and note consistency with `equals`.
* Give the immutability recipe and explain why immutable objects are thread-safe and good keys.
* Explain defensive copying and shallow vs deep copy; know why `clone` is avoided.
* Mention the Integer cache as the reason to never `==`-compare wrappers.
