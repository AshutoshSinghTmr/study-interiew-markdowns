# Java Virtual Threads and Loom

## The Problem Loom Solves

Traditional Java threads are thin wrappers over **OS threads**: each costs ~1 MB of stack and a kernel scheduling slot, so a JVM can host only a few thousand. The classic workaround for scalable I/O — thread pools plus asynchronous/reactive code — scales well but is hard to write, debug, and read (callback chains, lost stack traces). Project Loom (delivered as **virtual threads** in Java 21, JEP 444) makes the simple *thread-per-request* model scale to millions.

## Platform Threads vs Virtual Threads

* A **platform thread** is a 1:1 wrapper over an OS thread — heavyweight and limited in count.
* A **virtual thread** is a lightweight thread managed by the JVM, not the OS. Many virtual threads are multiplexed onto a small pool of **carrier** (platform) threads. They are cheap to create (millions) and have small, growable stacks stored on the heap.

```java
Thread.startVirtualThread(() -> handle(request));   // one-off

try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var task : tasks) exec.submit(task);        // a virtual thread per task
}                                                    // close() awaits completion
```

## Mounting, Unmounting, and Carriers

When a virtual thread runs, it is **mounted** on a carrier platform thread. When it hits a blocking operation (I/O, `sleep`, lock wait), the JVM **unmounts** it — saving its stack (a "continuation") to the heap and freeing the carrier to run another virtual thread. When the blocking call completes, the virtual thread is rescheduled onto some carrier. Blocking a virtual thread is cheap; blocking a platform thread is not. The carrier pool is a `ForkJoinPool` sized to the number of CPU cores by default.

## Pinning: The Main Pitfall

A virtual thread cannot unmount in a few situations — it becomes **pinned** to its carrier, blocking that OS thread:

* inside a `synchronized` block/method that then blocks;
* during a native call (JNI).

Pinning defeats the scalability benefit. The fix is to replace `synchronized` around blocking operations with `ReentrantLock` (which cooperates with unmounting). (Later JDKs progressively remove the `synchronized` pinning limitation, but interviews still expect you to know it.) Diagnose with `-Djdk.tracePinnedThreads=full`.

## What Doesn't Change

* Virtual threads are still `java.lang.Thread`; existing APIs, `ThreadLocal`, debuggers, and stack traces work.
* They are **not faster** for CPU-bound work — you still need only as many runnable threads as cores. Their win is **throughput for blocking I/O**.
* **Don't pool virtual threads.** They're cheap and disposable; create one per task. Pooling them is an anti-pattern (`newVirtualThreadPerTaskExecutor` creates a fresh one per task, not a fixed pool).

## Structured Concurrency (Preview)

Structured concurrency (JEP 453, preview in 21) treats a group of related concurrent subtasks as a single unit of work with a clear scope, so they're started, joined, and cancelled together — eliminating leaks and orphan threads.

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<User> user  = scope.fork(() -> fetchUser(id));   // child virtual threads
    Subtask<Order> order = scope.fork(() -> fetchOrder(id));
    scope.join();                 // wait for both
    scope.throwIfFailed();        // propagate the first failure, cancel siblings
    return combine(user.get(), order.get());
}                                 // scope guarantees all children are done/cancelled
```

If one subtask fails, the others are cancelled; if the scope exits, no child outlives it. This gives concurrent code the same reliability and readability as structured (block-scoped) sequential code.

## Scoped Values (Preview)

**Scoped values** (JEP 446) are an immutable, inheritable alternative to `ThreadLocal`, designed for the massive scale of virtual threads. A value is bound for the dynamic extent of a `run`/`call` and automatically visible to child threads in a structured scope — without the mutability, unbounded lifetime, and cleanup problems of `ThreadLocal`.

```java
private static final ScopedValue<User> CURRENT = ScopedValue.newInstance();

ScopedValue.where(CURRENT, user).run(() -> handle());  // bound for this call tree
// inside: CURRENT.get()
```

## When to Use Virtual Threads

* **Use** for high-concurrency, I/O-bound, thread-per-request servers (HTTP handlers, DB/RPC calls) where blocking code is clearer than reactive code.
* **Don't bother** for CPU-bound work (use a sized pool) or low-concurrency apps.
* **Migrating**: replace `Executors.newFixedThreadPool(...)` request executors with `newVirtualThreadPerTaskExecutor()`, and swap `synchronized` around blocking calls for `ReentrantLock`.

## Interview Q&A

**Q: What is a virtual thread?**
A: A lightweight, JVM-managed thread multiplexed onto a small pool of carrier (platform) OS threads. It's cheap to create in the millions and blocks cheaply, enabling scalable thread-per-request code.

**Q: How do virtual threads differ from platform threads?**
A: Platform threads map 1:1 to OS threads (heavy, few). Virtual threads are many-to-few over carriers, with heap-stored growable stacks, created and blocked cheaply.

**Q: What are mounting and unmounting?**
A: A virtual thread runs while mounted on a carrier; on a blocking call the JVM unmounts it (saving its continuation to the heap) and frees the carrier, remounting later when the operation completes.

**Q: What is pinning and how do you avoid it?**
A: A pinned virtual thread can't unmount — it holds its carrier — when blocking inside `synchronized` or a native call. Avoid by using `ReentrantLock` instead of `synchronized` around blocking sections; diagnose with `-Djdk.tracePinnedThreads`.

**Q: Are virtual threads faster for CPU-bound work?**
A: No. They improve throughput for blocking I/O by allowing huge numbers of cheap threads. CPU-bound work is still limited by cores and should use a sized pool.

**Q: Should you pool virtual threads?**
A: No — they're cheap and disposable. Create one per task (e.g. `newVirtualThreadPerTaskExecutor`). Pooling them is an anti-pattern.

**Q: What is structured concurrency?**
A: A model that scopes related subtasks as one unit via `StructuredTaskScope`, so they're forked, joined, and cancelled together — preventing thread leaks and giving clear error propagation.

**Q: What are scoped values and why prefer them over `ThreadLocal`?**
A: Immutable, inheritable values bound for a bounded dynamic scope, visible to child threads without the mutability, unbounded lifetime, and cleanup burden of `ThreadLocal` — well suited to millions of virtual threads.

**Q: Do existing APIs and `ThreadLocal` still work with virtual threads?**
A: Yes — a virtual thread is still a `java.lang.Thread`. But `ThreadLocal` at massive scale is discouraged in favour of scoped values.

**Q: What kind of application benefits most?**
A: High-concurrency, I/O-bound servers written in a blocking thread-per-request style — an alternative to reactive/async programming with far simpler code.

## Interview Notes

* Explain the scalability limit of OS threads that Loom addresses.
* Define virtual vs platform threads and the carrier/mount/unmount model.
* Know pinning causes (`synchronized`, native) and the `ReentrantLock` fix.
* Stress: not faster for CPU-bound; don't pool; one virtual thread per task.
* Describe structured concurrency (`StructuredTaskScope`) and scoped values as the `ThreadLocal` replacement.
* Give a concrete migration: fixed pool → `newVirtualThreadPerTaskExecutor`.
