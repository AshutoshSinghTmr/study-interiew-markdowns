# Java Generics

## Why Generics Exist

Generics add compile-time type safety and eliminate casts by parameterizing types and methods over the types they operate on. Before generics, collections held `Object` and required manual casts that failed at runtime with `ClassCastException`. Generics move that error to compile time.

```java
List<String> names = new ArrayList<>();
names.add("a");
String s = names.get(0);   // no cast; adding an Integer won't compile
```

## Generic Classes and Methods

A type parameter is a placeholder (`T`, `E`, `K`, `V`) bound when the type is used.

```java
class Box<T> {
    private T value;
    T get() { return value; }
    void set(T value) { this.value = value; }
}

// generic method: type parameter declared before the return type
static <T> T firstOrNull(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}
```

A generic method infers its type argument from the call, so you rarely specify it explicitly. Generic constructors and generic static methods are also allowed.

## Bounded Type Parameters

Bounds constrain a type parameter to a supertype, enabling method calls on `T`.

```java
static <T extends Comparable<T>> T max(List<T> list) {
    T best = list.get(0);
    for (T t : list) if (t.compareTo(best) > 0) best = t;
    return best;
}
```

`extends` here means "is a subtype of" and applies to both classes and interfaces. Multiple bounds use `&`: `<T extends Number & Comparable<T>>` (at most one class, which must come first).

## Wildcards and PECS

A wildcard `?` represents an unknown type, used for flexible method parameters:

* **Upper-bounded** `? extends T` — an unknown subtype of `T`. You can **read** `T` from it but cannot add (except `null`). It is a *producer*.
* **Lower-bounded** `? super T` — an unknown supertype of `T`. You can **write** `T` into it but reads come back as `Object`. It is a *consumer*.
* **Unbounded** `?` — any type; read as `Object`, write only `null`.

The mnemonic is **PECS — Producer Extends, Consumer Super**:

```java
static <T> void copy(List<? extends T> src,   // produces T -> extends
                     List<? super T> dst) {    // consumes T -> super
    for (T t : src) dst.add(t);
}
```

`Collections.copy`, `Stream.max(Comparator<? super T>)`, and similar signatures follow PECS to maximize caller flexibility.

## Type Erasure

Generics are a compile-time feature. The compiler checks types, inserts casts, and then **erases** type parameters — replacing `T` with its leftmost bound (or `Object`) in the bytecode. At runtime there is no `List<String>`, only `List`.

Consequences:

* `list instanceof List<String>` is illegal; only `list instanceof List<?>` is allowed.
* `new T()`, `new T[]`, and `T.class` are illegal — the type isn't known at runtime.
* Two overloads that differ only by generic parameter (`f(List<String>)` vs `f(List<Integer>)`) clash — same erasure.
* A single `Box.class` object exists for all `Box<T>`.

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
a.getClass() == b.getClass();   // true — both erase to ArrayList
```

## Bridge Methods

Because of erasure, the compiler synthesizes **bridge methods** to preserve polymorphism across a generic supertype. When `class MyComparable implements Comparable<MyComparable>` defines `compareTo(MyComparable)`, the compiler adds a synthetic `compareTo(Object)` that casts and delegates, so virtual dispatch through the erased `Comparable` interface still works. You rarely see them, but they explain odd stack traces and reflection results.

## Generics and Arrays

Arrays are **covariant** and **reified** (they know their element type at runtime); generics are **invariant** and **erased**. Mixing them is unsafe, so generic array creation is forbidden.

```java
Object[] arr = new String[1];
arr[0] = 42;                     // compiles, throws ArrayStoreException at runtime

List<String>[] bad = new List<String>[1];  // compile error
```

Invariance means `List<String>` is *not* a `List<Object>`, even though `String` is an `Object` — precisely so you can't insert an `Integer` into a `List<String>` through an aliased reference.

## Heap Pollution and `@SafeVarargs`

A generic varargs parameter creates a covariant array under the hood, risking **heap pollution** (a variable of a parameterized type refers to an object of a different type). The compiler warns; if the method only reads the varargs and never stores the array elsewhere, annotate it `@SafeVarargs` to suppress the warning.

```java
@SafeVarargs
static <T> List<T> listOf(T... items) {
    return new ArrayList<>(Arrays.asList(items));
}
```

## Recursive Generic Bounds (Self-Referential)

The pattern `<T extends Comparable<T>>` or the CRTP-style `class Builder<T extends Builder<T>>` lets a type refer to itself, enabling fluent builders that return the concrete subtype and comparisons restricted to the same type.

## Recovering Runtime Type Information

Because parameters are erased, APIs that need the runtime type accept a **type token** (`Class<T>`) or a `TypeReference`/super type token trick (as libraries like Jackson use) to capture the full parameterized type through an anonymous subclass, whose generic superclass *is* retained in the class file.

```java
static <T> T parse(String json, Class<T> type) { /* uses type at runtime */ }
```

## Interview Q&A

**Q: What problem do generics solve?**
A: Compile-time type safety and removal of casts. Type errors that used to appear as runtime `ClassCastException` are caught by the compiler.

**Q: What is type erasure?**
A: The compiler removes type parameters after checking, replacing them with their bound (or `Object`) and inserting casts. No generic type information exists at runtime for parameterized types.

**Q: Explain PECS.**
A: Producer Extends, Consumer Super. Use `? extends T` for a source you read from and `? super T` for a sink you write to, maximizing API flexibility.

**Q: Why can't you write `new T()` or `T[]`?**
A: Erasure means `T` is unknown at runtime, so the JVM cannot allocate the correct type. Pass a `Class<T>` factory/token or a supplier instead.

**Q: Why are generics invariant while arrays are covariant?**
A: Arrays are reified and covariant, which allows unsafe stores caught only at runtime (`ArrayStoreException`). Generics are invariant so such errors are impossible — you can't treat `List<String>` as `List<Object>`.

**Q: What is a bridge method?**
A: A synthetic method the compiler generates to preserve overriding/polymorphism after erasure, e.g. a `compareTo(Object)` that delegates to `compareTo(MyType)`.

**Q: What is heap pollution and when do you use `@SafeVarargs`?**
A: Heap pollution is when a parameterized variable references an object of a different type, possible via generic varargs arrays. `@SafeVarargs` asserts a method doesn't leak/store the array unsafely, suppressing the warning.

**Q: Can two methods differ only by their generic parameter type?**
A: No — `f(List<String>)` and `f(List<Integer>)` have the same erasure `f(List)` and clash.

**Q: What does `<T extends Number & Comparable<T>>` mean?**
A: Multiple bounds: `T` must be a subtype of both `Number` and `Comparable<T>`. At most one bound may be a class and it must be listed first.

**Q: How do frameworks recover generic types at runtime despite erasure?**
A: Via type tokens (`Class<T>`) or super type tokens — an anonymous subclass whose parameterized superclass is retained in the class file and read through reflection.

## Interview Notes

* Distinguish generic classes, generic methods, and bounded parameters.
* Explain PECS with a `copy`-style signature.
* Be fluent on erasure and its consequences (no `new T()`, no `instanceof List<String>`, overload clashes, one `Class` object).
* Contrast reified covariant arrays with erased invariant generics and why invariance is safer.
* Mention bridge methods, heap pollution/`@SafeVarargs`, and type tokens for runtime type recovery.
