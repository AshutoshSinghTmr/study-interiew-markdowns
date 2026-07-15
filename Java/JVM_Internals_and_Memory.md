# Java JVM Internals and Memory

## JDK, JRE, and JVM

* **JVM (Java Virtual Machine)** — the abstract runtime that executes bytecode; a specification with many implementations (HotSpot, OpenJ9). It provides platform independence ("write once, run anywhere").
* **JRE (Java Runtime Environment)** — the JVM plus the core class libraries needed to *run* Java programs.
* **JDK (Java Development Kit)** — the JRE plus development tools (`javac`, `jar`, `javadoc`, `jlink`, `jshell`) needed to *build* them.

Since Java 11 there is no separate JRE download; you produce a runtime with `jlink`.

## Compilation and Bytecode

`javac` compiles `.java` source to **bytecode** (`.class` files) — a portable, stack-based instruction set independent of any CPU. The JVM loads and executes this bytecode. Because bytecode targets the JVM rather than hardware, the same `.class` runs on any platform with a compatible JVM.

```
Foo.java --javac--> Foo.class (bytecode) --JVM--> interpret + JIT-compile --> native execution
```

Inspect bytecode with `javap -c ClassName`.

## Class Loading: Loading, Linking, Initialization

A class is loaded lazily on first active use, in three phases:

1. **Loading** — a class loader reads the `.class` bytes and creates a `Class` object.
2. **Linking**:
   * *Verification* — bytecode is checked for safety and correctness.
   * *Preparation* — static fields are allocated and set to default values (0/null).
   * *Resolution* — symbolic references to other classes/methods are resolved (may be lazy).
3. **Initialization** — static initializers and static field assignments run, exactly once, in source order.

## The Class Loader Hierarchy and Delegation

Class loaders form a parent-first hierarchy:

* **Bootstrap** — loads core JDK classes (`java.*`); written in native code, shown as `null`.
* **Platform** (formerly Extension) — loads JDK modules beyond the core.
* **Application (System)** — loads classes from the application classpath/module path.

The **parent delegation model**: a loader first asks its parent to load a class, and only loads it itself if the parent can't. This ensures core classes (like `java.lang.Object`) are loaded once by the bootstrap loader and can't be spoofed. Frameworks (app servers, plugins) sometimes use custom loaders (occasionally child-first) for isolation and hot deployment.

## Runtime Data Areas

The JVM partitions memory into:

* **Heap** — shared across threads; holds all objects and arrays; managed by the garbage collector. Split into young and old generations.
* **Stack** — one per thread; holds **stack frames** (one per method call) with local variables, operand stack, and the return address. `StackOverflowError` on deep/infinite recursion.
* **Program Counter (PC) register** — per thread; the address of the current bytecode instruction.
* **Metaspace** — off-heap (native memory) storage for class metadata; replaced the fixed **PermGen** in Java 8, growing dynamically (bounded by `-XX:MaxMetaspaceSize`).
* **Native method stack** — for JNI/native calls.
* **Runtime constant pool** — per class; symbolic constants and references (part of method-area/metaspace).

## Object Memory Layout

A heap object has a **header** (mark word for hash/lock/GC state, and a class pointer), the instance fields (with alignment padding to 8-byte boundaries), and, for arrays, a length. `-XX:+UseCompressedOops` stores 64-bit references as 32-bit offsets under ~32 GB heaps, saving memory. This layout explains why object overhead matters for many small objects.

## Execution: Interpreter, JIT, and Tiered Compilation

The JVM starts by **interpreting** bytecode, then the **JIT compiler** compiles hot methods to optimized native code:

* **C1 (client)** — fast compilation, lighter optimization; good for startup.
* **C2 (server)** — aggressive optimizations for long-running code.
* **Tiered compilation** (default) uses C1 first (with profiling) then recompiles hot methods with C2, balancing startup and peak performance.

The JIT uses runtime profiles to inline methods, unroll loops, and eliminate dead code — often beating static AOT compilation. It can **deoptimize** back to the interpreter when a speculative assumption (e.g. a monomorphic call site) is invalidated.

## Key JIT Optimizations

* **Method inlining** — replace a call with the callee's body (the most impactful optimization).
* **Escape analysis** — if an object never escapes a method, the JVM can allocate it on the stack or eliminate it (scalar replacement), reducing heap/GC pressure and even removing locks (lock elision).
* **Loop unrolling, dead-code elimination, branch prediction** — standard speedups guided by profiles.

## `invokedynamic` and Method Handles

Bytecode has four classic call instructions (`invokevirtual`, `invokestatic`, `invokespecial`, `invokeinterface`) plus **`invokedynamic`**, which defers linking a call site to a runtime **bootstrap method**. It powers lambdas (`LambdaMetafactory`) and string concatenation (Java 9+), enabling flexible, JIT-friendly linkage decided at first execution.

## AOT, CDS, and Native Image

* **Class Data Sharing (CDS/AppCDS)** — memory-maps a pre-parsed class archive to speed startup and share metadata across JVMs.
* **AOT / GraalVM native image** — compiles the application ahead of time into a standalone native executable with near-instant startup and low memory, at the cost of build complexity and limited runtime reflection (needs configuration). Great for serverless/CLI.

## Diagnostics

`jps`, `jstack` (thread dumps), `jmap`/`jcmd` (heap dumps, histograms), `jstat` (GC stats), JFR (Java Flight Recorder), and JMC are the standard tools for inspecting a live JVM's memory, threads, and hot spots.

## Interview Q&A

**Q: Difference between JDK, JRE, and JVM?**
A: The JVM executes bytecode; the JRE is the JVM plus core libraries to run apps; the JDK is the JRE plus development tools (`javac`, etc.) to build them.

**Q: What are the phases of class loading?**
A: Loading (read bytes, create `Class`), Linking (verify → prepare static fields to defaults → resolve references), and Initialization (run static initializers once).

**Q: What is the parent delegation model and why does it matter?**
A: A class loader delegates to its parent before loading a class itself, so core classes are loaded once by trusted loaders and can't be replaced by user code — a key security/consistency property.

**Q: What lives on the heap vs the stack?**
A: The heap holds objects/arrays shared by all threads and is GC-managed; each thread's stack holds per-call frames with locals and operands. Deep recursion overflows the stack; too many live objects exhaust the heap.

**Q: What replaced PermGen and why?**
A: Metaspace (Java 8), which stores class metadata in native memory and grows dynamically, avoiding the frequent `OutOfMemoryError: PermGen space` from the old fixed-size region.

**Q: How does the JIT improve performance? What is tiered compilation?**
A: It compiles hot bytecode to native code using runtime profiles (inlining, escape analysis, loop unrolling). Tiered compilation starts with C1 for fast startup then recompiles hot methods with C2 for peak throughput.

**Q: What is escape analysis?**
A: A JIT analysis that detects objects not escaping a method, enabling stack allocation/scalar replacement and lock elision, reducing heap and GC pressure.

**Q: What is `invokedynamic` used for?**
A: A bytecode instruction that links a call site at runtime via a bootstrap method; it implements lambdas and string concatenation flexibly and efficiently.

**Q: Why is Java platform-independent?**
A: Source compiles to portable bytecode targeting the JVM abstraction; any platform's JVM executes the same bytecode, isolating programs from hardware/OS differences.

**Q: What is GraalVM native image and its trade-offs?**
A: AOT compilation to a native executable with near-instant startup and low memory, at the cost of longer builds and restricted runtime reflection/dynamic loading requiring explicit configuration.

## Interview Notes

* Cleanly separate JDK/JRE/JVM and describe the javac → bytecode → JIT flow.
* Recite the three class-loading phases and the parent delegation model plus its security role.
* Enumerate runtime data areas (heap, stack, PC, metaspace, native) and what errors each produces.
* Explain interpreter + tiered JIT (C1/C2), inlining, and escape analysis.
* Mention metaspace replacing PermGen, compressed oops, and `invokedynamic`.
* Know CDS/AOT/native-image trade-offs and the core diagnostic tools.
