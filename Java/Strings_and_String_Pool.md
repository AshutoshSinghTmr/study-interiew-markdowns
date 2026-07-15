# Java Strings and String Pool

## Strings Are Immutable

`String` is immutable: once created, its contents never change. Every "modifying" operation (`concat`, `substring`, `replace`, `toUpperCase`) returns a **new** `String`. Immutability brings major benefits:

* **Thread safety** — shareable without synchronization.
* **Safe caching/interning** — a value can be pooled and reused.
* **Hashcode caching** — `String` caches its hash, making it an excellent `HashMap` key.
* **Security** — a string passed to a file path or network call can't be mutated after validation.

```java
String s = "abc";
s.concat("d");     // returns "abcd" but doesn't change s
System.out.println(s);   // still "abc"
```

## The String Pool (Interning)

The JVM maintains a **string pool** (a table of unique string instances) so identical string literals share one object. String literals are automatically interned; strings built at runtime are not, unless you call `intern()`.

```java
String a = "hello";              // literal -> pooled
String b = "hello";              // same pooled object
a == b;                          // true

String c = new String("hello"); // new heap object, not pooled
a == c;                          // false
a == c.intern();                 // true — intern() returns the pooled instance
```

The pool lives in the **heap** (since Java 7; it was in PermGen before), so it participates in normal GC. Interning saves memory for many duplicate strings but has a lookup cost, so intern deliberately, not everywhere.

## `==` vs `equals`

`==` compares references; `equals` compares character content. Because literals are pooled, `==` on literals can appear to work — then fail for runtime-built strings. **Always use `equals` for content comparison.** A useful idiom to avoid NPEs is `"CONST".equals(userInput)`.

## `String` vs `StringBuilder` vs `StringBuffer`

Because `String` is immutable, repeated concatenation in a loop creates many throwaway objects (O(n²) work). Use a mutable builder instead:

* **`StringBuilder`** — mutable, **not** thread-safe, fast; the default for building strings.
* **`StringBuffer`** — mutable and synchronized (thread-safe), slower; rarely needed today.

```java
// Bad: quadratic, many temporaries
String r = "";
for (String p : parts) r += p;

// Good: linear, one buffer
StringBuilder sb = new StringBuilder();
for (String p : parts) sb.append(p);
String r2 = sb.toString();
```

## How Compile-Time and Runtime Concatenation Differ

A single expression like `"a" + b + "c"` is optimized by the compiler. Since Java 9 (JEP 280), `javac` emits an **`invokedynamic`** call to `StringConcatFactory` rather than explicit `StringBuilder` code, letting the runtime pick the best strategy. Constant expressions (`"a" + "b"`) are folded at compile time into a single pooled literal `"ab"`. The performance problem is only **repeated** concatenation across statements/loops — that still needs an explicit `StringBuilder`.

## Compact Strings

Since Java 9, `String` uses a `byte[]` plus a coder flag instead of a `char[]` (JEP 254, "compact strings"). Strings containing only Latin-1 characters use one byte per character instead of two, roughly halving memory for typical ASCII-heavy applications — transparently, with no code changes.

## Text Blocks

Text blocks (Java 15) are multi-line literals delimited by `"""`, preserving newlines and reducing escaping — ideal for JSON, SQL, and HTML. Incidental leading whitespace is stripped based on the closing delimiter's indentation.

```java
String json = """
    {
      "name": "Ada",
      "role": "engineer"
    }
    """;
```

## Useful `String` APIs

`isBlank`, `strip`/`stripLeading`/`stripTrailing` (Unicode-aware, unlike `trim`), `lines()` (stream of lines), `repeat(n)`, `formatted(...)`, `chars()`, and `String.join(...)`. Prefer `strip` over `trim` for correct Unicode whitespace handling.

## Performance and Memory Notes

* Reuse a single `StringBuilder` in loops; presize it (`new StringBuilder(expectedLen)`) to avoid regrowth.
* Avoid unnecessary `intern()` — it adds pool lookups and can retain memory.
* `substring` (since Java 7) copies characters (no shared backing array), so it no longer leaks the parent string.
* Don't hold huge strings you only need a slice of; copy the slice.

## Interview Q&A

**Q: Why is `String` immutable?**
A: For thread safety, safe sharing/pooling, cached hash codes (great map keys), and security (validated strings can't be mutated). Every mutating method returns a new string.

**Q: What is the string pool?**
A: A heap-based table of unique strings. Literals are auto-interned so identical literals share one object; `intern()` returns the pooled instance for a runtime-built string.

**Q: Why does `new String("x") == "x"` return false?**
A: `new String(...)` creates a distinct heap object, not the pooled literal. `==` compares references; use `equals` or `intern()` for pooled identity.

**Q: `StringBuilder` vs `StringBuffer`?**
A: Both are mutable; `StringBuilder` is faster and not synchronized (single-thread use), `StringBuffer` is synchronized/thread-safe but slower. Prefer `StringBuilder`.

**Q: Is `"a" + b + "c"` slow?**
A: No — a single expression is optimized (via `invokedynamic`/`StringConcatFactory` since Java 9). Only repeated concatenation across loops/statements is O(n²); use an explicit `StringBuilder` there.

**Q: What are compact strings?**
A: Since Java 9, `String` stores a `byte[]` with a coder flag, using one byte per char for Latin-1 content — roughly halving memory for ASCII-heavy text, transparently.

**Q: Why use `equals` instead of `==` for strings?**
A: `==` compares references and only coincidentally matches for pooled literals; `equals` compares content, which is what you almost always want.

**Q: What are text blocks?**
A: Multi-line string literals (Java 15) using `"""` that preserve formatting and reduce escaping, with incidental indentation stripped.

**Q: `strip` vs `trim`?**
A: `strip` (Java 11) is Unicode-whitespace-aware; `trim` only removes characters ≤ U+0020. Prefer `strip`.

**Q: Does `substring` still leak memory?**
A: Not since Java 7 — it copies the characters into a new array instead of sharing the parent's backing array.

## Interview Notes

* Explain immutability and its four benefits (thread safety, pooling, cached hash, security).
* Describe the heap-based string pool and `intern()`, and the `==` vs `equals` trap.
* Contrast `StringBuilder` (fast) vs `StringBuffer` (synchronized) and the loop-concatenation pitfall.
* Note single-expression concat is optimized via `invokedynamic` since Java 9.
* Mention compact strings, text blocks, and `strip` vs `trim`.
