# Java Garbage Collection

## Automatic Memory Management

Java frees developers from manual `malloc`/`free` by automatically reclaiming unreachable objects. The garbage collector (GC) runs in the background, finding objects no longer reachable from live roots and returning their memory to the heap. This eliminates most memory leaks and dangling-pointer bugs — but not all (see leaks below).

## Reachability and GC Roots

An object is **live** if it is reachable from a **GC root**; otherwise it is eligible for collection. GC roots include:

* local variables and operands on thread stacks;
* active threads;
* static fields of loaded classes;
* JNI references.

The collector traverses the object graph from these roots (mark phase); everything not marked is garbage. Reachability, not reference counting, is used — so reference **cycles** (A↔B with no external references) are still collected.

## The Generational Hypothesis

Empirically, **most objects die young**. Generational GC exploits this by splitting the heap:

* **Young generation** — new objects; collected frequently and cheaply by **minor GC**. Subdivided into **Eden** and two **survivor** spaces (S0/S1).
* **Old (tenured) generation** — objects that survive several minor GCs are **promoted** here; collected by **major/full GC**, which is more expensive.

A minor GC copies survivors between survivor spaces, bumping their age; once an object's age crosses a threshold it is tenured. Copying collection of the young gen is fast because it only touches live objects and compacts as it goes.

## Stop-The-World Pauses

Many GC phases require a **stop-the-world (STW)** pause where application threads are halted so the collector can work on a consistent heap. Minor GCs are short; full GCs can be long. Modern collectors minimize STW by doing most work concurrently with the application. Pause time, throughput, and footprint are the three competing goals you tune between.

## The Collectors

* **Serial GC** (`-XX:+UseSerialGC`) — single-threaded, simple; good for small heaps/containers and single-core.
* **Parallel GC** (`-XX:+UseParallelGC`) — multi-threaded, throughput-oriented; longer pauses. The old default for batch workloads.
* **G1 GC** (`-XX:+UseG1GC`) — the **default since Java 9**. Divides the heap into equal **regions**, collects those with the most garbage first ("garbage first"), and targets a configurable pause goal (`-XX:MaxGCPauseMillis`). Balances throughput and latency for large heaps.
* **ZGC** (`-XX:+UseZGC`) — a low-latency, concurrent, region-based collector with sub-millisecond pauses that stay flat as the heap scales to terabytes. Uses colored pointers and load barriers.
* **Shenandoah** (`-XX:+UseShenandoahGC`) — another concurrent low-pause collector (OpenJDK), doing concurrent compaction via Brooks/load barriers.
* **Epsilon** — a no-op collector for performance testing (allocates, never reclaims).

Choose Parallel for max throughput/batch, G1 for a balanced default, ZGC/Shenandoah when low and predictable latency matters most.

## Minor, Major, and Full GC

* **Minor GC** — collects the young generation; frequent and fast.
* **Major GC** — collects the old generation.
* **Full GC** — collects the entire heap (young + old, sometimes metaspace); the most disruptive and a common cause of latency spikes. Frequent full GCs signal memory pressure or a leak.

## Reference Types

`java.lang.ref` exposes GC interaction beyond strong references:

* **Strong** — the default; prevents collection while reachable.
* **Soft** (`SoftReference`) — collected only under memory pressure; useful for memory-sensitive caches.
* **Weak** (`WeakReference`) — collected at the next GC once weakly reachable; the basis of `WeakHashMap` for canonicalizing maps/listener registries.
* **Phantom** (`PhantomReference`) — enqueued after finalization for scheduling precise post-mortem cleanup (a safer replacement for `finalize`), used with a `ReferenceQueue` and `Cleaner`.

## Memory Leaks in a GC Language

GC only reclaims *unreachable* objects, so leaks happen when references are unintentionally retained:

* forgotten entries in long-lived collections (static maps, caches without eviction);
* unremoved listeners/callbacks;
* `ThreadLocal`s not cleared in pooled threads;
* `ClassLoader` leaks (a single retained class keeps the whole loader and its classes alive).

Diagnose with heap dumps (`jmap`/`jcmd`), then analyze dominators and retained size in Eclipse MAT or VisualVM to find what keeps objects alive.

## Tuning and Diagnostics

Common flags: `-Xms`/`-Xmx` (initial/max heap), `-Xmn` (young size), `-XX:MaxGCPauseMillis` (G1 goal), `-XX:+UseG1GC`/`UseZGC`, and GC logging via `-Xlog:gc*`. General guidance: right-size the heap (too small → frequent GC; too large → long pauses/wasted RAM), prefer the collector matching your latency/throughput goal, and always **measure with GC logs and JFR** rather than guessing. `finalize()` is deprecated for removal — use `Cleaner`/try-with-resources instead.

## Interview Q&A

**Q: How does Java decide an object can be collected?**
A: By reachability — if it can't be reached from any GC root, it's garbage. This handles reference cycles that reference counting cannot.

**Q: What are GC roots?**
A: Thread-stack locals, active threads, static fields of loaded classes, and JNI references — the starting points for liveness tracing.

**Q: What is the generational hypothesis and how does GC use it?**
A: Most objects die young, so the heap is split into a frequently, cheaply collected young generation (Eden + survivors) and an old generation for long-lived objects collected less often.

**Q: What is a stop-the-world pause?**
A: A phase where all application threads are paused so the GC can work on a consistent heap. Minimizing STW time is the goal of modern concurrent collectors.

**Q: What is the default collector and how does G1 work?**
A: G1 (default since Java 9). It divides the heap into regions and collects those with the most garbage first, aiming for a target pause time — balancing latency and throughput on large heaps.

**Q: When would you use ZGC or Shenandoah?**
A: For latency-sensitive services needing very low, predictable pauses (sub-millisecond) that stay flat even with very large heaps; they do most work concurrently.

**Q: Minor vs major vs full GC?**
A: Minor collects the young gen (frequent, fast); major collects the old gen; full collects the whole heap and is the most disruptive — frequent full GCs indicate pressure or a leak.

**Q: Explain the reference types.**
A: Strong (never collected while reachable), soft (collected under memory pressure — caches), weak (collected at next GC — `WeakHashMap`), phantom (post-finalization cleanup via `ReferenceQueue`/`Cleaner`).

**Q: Can you leak memory in Java? Give examples.**
A: Yes — via retained references: unbounded caches/static collections, unremoved listeners, uncleared `ThreadLocal`s in pools, and classloader leaks. GC can't reclaim reachable objects.

**Q: How do you diagnose a memory leak?**
A: Capture a heap dump (`jmap`/`jcmd`) and analyze retained size/dominators in a tool like Eclipse MAT to find what's holding the objects, plus GC logs for trends.

**Q: Why is `finalize()` discouraged?**
A: It's unpredictable, delays reclamation, can resurrect objects, and hurts performance. It's deprecated for removal — use `Cleaner` or try-with-resources.

## Interview Notes

* Explain reachability from GC roots and why cycles are still collected.
* Describe the generational heap (Eden/survivors/old) and promotion.
* Compare Serial/Parallel/G1/ZGC/Shenandoah by throughput vs latency, and know G1 is the default.
* Distinguish minor/major/full GC and what frequent full GCs imply.
* Explain the four reference types and their use cases.
* Give concrete leak sources and the heap-dump diagnosis workflow; know `finalize` is replaced by `Cleaner`.
