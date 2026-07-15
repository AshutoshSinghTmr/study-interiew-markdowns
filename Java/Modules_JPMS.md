# Java Modules (JPMS)

## What the Module System Is

The **Java Platform Module System (JPMS)**, delivered in Java 9 (Project Jigsaw), adds a layer of structure above packages: a **module** is a named, self-describing collection of packages and resources that explicitly declares what it **requires** from others and what it **exports** to others. It brings **strong encapsulation** and **reliable configuration** to large applications and to the JDK itself, which was modularized into ~`java.base` and many other modules.

## The Problems It Solves

* **JAR/classpath hell** — the flat classpath had no notion of dependencies, allowed split packages, and failed at runtime (`NoClassDefFoundError`) rather than startup.
* **Weak encapsulation** — `public` meant public to everyone; internal APIs (`sun.*`, `com.sun.*`) were used and abused. Modules hide non-exported packages even if their classes are `public`.
* **Monolithic runtime** — the whole JDK loaded regardless of need; modules enable custom, minimal runtimes via `jlink`.

## `module-info.java`

A module is declared by a `module-info.java` at the source root:

```java
module com.example.orders {
    requires com.example.core;            // depend on another module
    requires transitive com.example.api;  // re-export: my consumers also read api
    requires static lombok;               // compile-time only (optional at runtime)

    exports com.example.orders.api;                 // public to everyone
    exports com.example.orders.internal to com.example.web; // qualified export

    opens com.example.orders.model;       // deep reflection (e.g. Jackson/JPA) at runtime
    opens com.example.orders.dto to com.fasterxml.jackson.databind;

    uses com.example.orders.spi.PaymentProvider;                  // ServiceLoader consumer
    provides com.example.orders.spi.PaymentProvider
        with com.example.orders.impl.StripeProvider;             // service provider
}
```

## The Directives

* **`requires`** — declares a dependency on another module. `requires transitive` re-exports that dependency to your consumers (implied readability). `requires static` is a compile-time-only dependency.
* **`exports`** — makes a package's public types accessible to other modules. A **qualified** `exports ... to` limits access to named modules only.
* **`opens`** — grants **deep reflective** access (including private members via `setAccessible`) at runtime without exposing the package at compile time — needed by frameworks like Jackson, Hibernate, and Spring. `open module` opens every package.
* **`uses` / `provides ... with`** — declare a `ServiceLoader` consumer and provider, the module-aware SPI mechanism.

## Strong Encapsulation

The core rule: a type in a package is accessible to another module **only if** that package is `exported`, regardless of whether the type is `public`. Internal packages are inaccessible by default. Reflection is likewise blocked unless the package is `opens`ed — this is why upgrading to modules often surfaces `InaccessibleObjectException` from libraries doing reflection, fixed by `opens`.

## Module Path vs Classpath

* On the **classpath**, JARs are treated the old way (no encapsulation, split packages tolerated).
* On the **module path**, JARs are resolved as modules with enforced dependencies and encapsulation, validated at **startup** (the module graph is resolved before `main`), giving fail-fast, reliable configuration.

## Automatic and Unnamed Modules

Migration is gradual because of two bridging concepts:

* **Automatic module** — a plain (non-modular) JAR placed on the module path becomes a module whose name is derived from the JAR (or its `Automatic-Module-Name` manifest entry). It exports all its packages and can read all others, letting modular code depend on not-yet-modularized libraries.
* **Unnamed module** — everything on the **classpath** belongs to the unnamed module, which reads every module and exports everything. This preserves backward compatibility: legacy apps run unchanged on the classpath.

A recommended **bottom-up** migration modularizes leaf libraries first; **top-down** starts with the app as an automatic-module consumer.

## Tooling: `jlink`, `jdeps`, `jmod`

* **`jdeps`** — analyzes dependencies and flags use of internal/`JDK` APIs; helps plan modularization.
* **`jlink`** — assembles a **custom runtime image** containing only the modules your app needs, producing a small, self-contained runtime (great for containers).
* **`jmod`** — packages modules that can include native code/config for `jlink`.

```bash
jlink --add-modules com.example.orders --output runtime   # minimal custom JRE
```

## When Modules Matter (and When Not)

Full JPMS adoption pays off for large platforms, libraries, and applications needing strong encapsulation, reliable dependencies, or minimal runtime images. Many application teams run on the classpath (unnamed module) and still benefit indirectly because the **JDK is modularized** (smaller footprint, `jlink`, enforced encapsulation of internal APIs). Know both realities.

## Interview Q&A

**Q: What is JPMS and why was it introduced?**
A: The Java Platform Module System (Java 9) adds named modules that declare their dependencies and exported packages, providing strong encapsulation, reliable startup-time configuration, and a modular JDK — solving classpath hell and weak encapsulation.

**Q: What does `module-info.java` contain?**
A: The module name plus directives: `requires` (dependencies), `exports` (accessible packages), `opens` (deep reflection), and `uses`/`provides` (services).

**Q: `exports` vs `opens`?**
A: `exports` makes a package's public types accessible at compile and run time; `opens` grants runtime deep reflective access (including private members) for frameworks, without compile-time exposure.

**Q: What is `requires transitive`?**
A: Implied readability — modules that require your module automatically read the transitively-required module too, so you can re-export a dependency that appears in your public API.

**Q: What is strong encapsulation in modules?**
A: A `public` type is only accessible to other modules if its package is exported; non-exported packages are hidden, and reflection is blocked unless the package is opened.

**Q: Automatic module vs unnamed module?**
A: An automatic module is a plain JAR on the module path (name derived from the JAR, exports everything, reads all) enabling migration; the unnamed module is everything on the classpath (reads all, exports all) preserving legacy behaviour.

**Q: Module path vs classpath?**
A: The module path enforces module boundaries and resolves the dependency graph at startup (fail-fast); the classpath is the legacy flat model without encapsulation or dependency checks.

**Q: What does `jlink` do?**
A: Builds a minimal custom runtime image containing only the required modules, yielding a small, self-contained JRE — ideal for containers and serverless.

**Q: Why do libraries need `opens`?**
A: Frameworks like Jackson, Hibernate, and Spring use reflection (including private members) for serialization/injection; without `opens`, they throw `InaccessibleObjectException`.

**Q: Do you have to modularize your app to use Java 9+?**
A: No — apps run on the classpath in the unnamed module unchanged. Modularization is optional and adopted where its guarantees add value.

## Interview Notes

* Explain the problems JPMS solves: classpath hell, weak encapsulation, monolithic runtime.
* Know the directives: `requires`/`transitive`/`static`, `exports`(+qualified), `opens`, `uses`/`provides`.
* Distinguish `exports` (compile+runtime access) from `opens` (runtime reflection).
* Contrast module path (enforced, startup-resolved) with classpath (unnamed module).
* Explain automatic vs unnamed modules for migration.
* Cite `jdeps`/`jlink`/`jmod` and the reason libraries need `opens`.
