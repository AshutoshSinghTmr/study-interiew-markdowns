# Java Exceptions and Error Handling

## The Throwable Hierarchy

All errors and exceptions descend from `java.lang.Throwable`:

```
Throwable
├── Error                (serious, unrecoverable: OutOfMemoryError, StackOverflowError)
└── Exception
    ├── RuntimeException  (unchecked: NullPointerException, IllegalArgumentException, ...)
    └── (other)          (checked: IOException, SQLException, ...)
```

* **`Error`** signals conditions a program should not try to catch (JVM/resource failures).
* **Checked exceptions** (subclasses of `Exception` excluding `RuntimeException`) must be declared or handled — the compiler enforces it.
* **Unchecked exceptions** (`RuntimeException` and subclasses) need no declaration and usually indicate programming bugs.

## Checked vs Unchecked

Checked exceptions represent recoverable, expected conditions at an API boundary (a file missing, a network drop) that the caller should consciously handle. Unchecked exceptions represent programming errors (bad argument, null dereference, illegal state) that are better fixed than caught.

The pragmatic guidance most teams follow: use unchecked exceptions for most application error signalling to avoid `throws` clutter and leaky abstractions, reserving checked exceptions for genuinely recoverable, caller-actionable conditions. Never declare `throws Exception` broadly — it destroys the compiler's ability to reason about failures.

## `try`, `catch`, `finally`

```java
try {
    process();
} catch (IOException e) {
    log.error("io failed", e);        // handle or translate
    throw new ServiceException(e);    // preserve cause
} finally {
    cleanup();                        // always runs (except System.exit / JVM crash)
}
```

`finally` always executes — even after a `return` in `try`/`catch`. A `return` or `throw` inside `finally` **overrides** any pending return/exception from the `try` block, silently swallowing it — a classic bug, so never return or throw from `finally`.

## Multi-catch

Catch several unrelated exception types in one block when handling is identical. The caught variable is implicitly `final`, and the types must not be in a subclass relationship.

```java
try {
    parseAndStore();
} catch (IOException | SQLException e) {   // one handler, two types
    throw new DataAccessException(e);
}
```

## try-with-resources and `AutoCloseable`

Any resource implementing `AutoCloseable` (or `Closeable`) is closed automatically at the end of the block, in **reverse** order of opening, whether or not an exception occurs. This replaces error-prone nested `finally` blocks.

```java
try (var in = Files.newInputStream(path);
     var out = Files.newOutputStream(dst)) {   // closed in reverse: out, then in
    in.transferTo(out);
}
```

Since Java 9 you can list already-effectively-final resource variables in the header. try-with-resources also fixes the "lost exception" problem via suppression.

## Suppressed Exceptions

If the `try` body throws and a resource's `close()` also throws, the close exception would previously mask the real one. try-with-resources instead **suppresses** the close exception, attaching it to the primary via `addSuppressed`; retrieve them with `getSuppressed()`.

```java
try { throw new RuntimeException("primary"); }
// if close() also throws, that becomes a suppressed exception on "primary"
```

## Exception Chaining and Translation

Wrap a low-level exception in a higher-level, domain-meaningful one while preserving the original as the **cause**. This keeps abstraction layers clean without losing the stack trace.

```java
catch (SQLException e) {
    throw new RepositoryException("save failed", e);  // e is the cause
}
```

Always pass the cause (constructor or `initCause`). Losing it makes production debugging far harder.

## Custom Exceptions

Define custom exceptions to convey domain meaning and carry contextual data. Extend `RuntimeException` for unchecked (the common choice) or `Exception` for checked.

```java
public class InsufficientFundsException extends RuntimeException {
    private final BigDecimal shortfall;
    public InsufficientFundsException(BigDecimal shortfall) {
        super("short by " + shortfall);
        this.shortfall = shortfall;
    }
    public BigDecimal shortfall() { return shortfall; }
}
```

Provide a constructor accepting a `cause` so chaining works. Don't create an exception per method — model meaningful failure categories.

## Performance and Stack Traces

Constructing an exception is relatively cheap except for `fillInStackTrace`, which walks the stack — the dominant cost. Never use exceptions for ordinary control flow (e.g. to end a loop). For extremely hot, expected-failure paths you can override `fillInStackTrace` to skip trace capture, but this is a rare optimization. Prefer returning `Optional`/result types over throwing for expected "not found" outcomes.

## Best Practices

* Catch the most specific type you can handle; never swallow with an empty `catch`.
* Always log or rethrow with the cause; don't do both blindly (avoid double-logging).
* Don't catch `Throwable`/`Error` unless you have a specific reason (e.g. top-level supervisor).
* Fail fast: validate arguments early and throw `IllegalArgumentException`/`NullPointerException` (via `Objects.requireNonNull`) at boundaries.
* Clean up with try-with-resources, not manual `finally`.
* Keep `throws` clauses honest and narrow.

```java
void deposit(Account acct, BigDecimal amt) {
    Objects.requireNonNull(acct, "acct");
    if (amt.signum() <= 0) throw new IllegalArgumentException("amt must be > 0");
    // ...
}
```

## Interview Q&A

**Q: What is the difference between checked and unchecked exceptions?**
A: Checked exceptions (subclasses of `Exception`, not `RuntimeException`) must be declared or handled at compile time and model recoverable conditions. Unchecked exceptions (`RuntimeException`) need no declaration and usually indicate bugs.

**Q: What is the difference between `Error` and `Exception`?**
A: `Error` indicates serious, typically unrecoverable JVM/resource conditions you shouldn't catch; `Exception` indicates conditions an application can reasonably anticipate and handle.

**Q: Does `finally` always run?**
A: Yes, except on `System.exit()`, a JVM crash, or an infinite loop/daemon thread kill. Avoid `return`/`throw` in `finally` because they override the pending result and swallow exceptions.

**Q: How does try-with-resources work and why prefer it?**
A: It auto-closes any `AutoCloseable` in reverse order at block end, even on exception, and suppresses close-time exceptions instead of masking the primary — eliminating leaky, error-prone `finally` blocks.

**Q: What are suppressed exceptions?**
A: When both the body and a resource `close()` throw, the close exception is attached to the primary via `addSuppressed` and retrievable with `getSuppressed()`, so the original error isn't lost.

**Q: What is exception chaining/translation?**
A: Wrapping a low-level exception in a domain exception while passing the original as the cause, preserving the stack trace across abstraction layers.

**Q: When should you create a custom exception?**
A: When you need domain-specific meaning or contextual data. Model failure categories, extend `RuntimeException` (usually), and include a cause-accepting constructor.

**Q: Should you use exceptions for control flow?**
A: No. Exceptions are costly (mainly `fillInStackTrace`) and obscure intent. Use return values, `Optional`, or result objects for expected outcomes.

**Q: What does `Objects.requireNonNull` give you?**
A: A fail-fast null check that throws a clear `NullPointerException` at the boundary with a helpful message, instead of a confusing NPE deep in the call stack.

**Q: Why avoid catching `Throwable`?**
A: It also catches `Error`s like `OutOfMemoryError` you can't sensibly recover from, and can hide serious problems. Catch specific exceptions instead.

## Interview Notes

* Draw the `Throwable` hierarchy and place checked/unchecked/Error correctly.
* Explain the checked-vs-unchecked design trade-off and why broad `throws Exception` is bad.
* Know the `finally` override pitfall and why not to return/throw from it.
* Explain try-with-resources ordering plus suppressed exceptions.
* Emphasize chaining with a cause and fail-fast validation with `requireNonNull`.
* State that `fillInStackTrace` is the main cost and exceptions aren't for control flow.
