# Java Concurrency and Multithreading

## Processes and Threads

A thread is the unit of scheduling within a process; all threads of a JVM share the same heap (objects) but each has its own stack, program counter, and local variables. Concurrency lets you overlap work (I/O, CPU) but introduces the hard problems of **visibility**, **atomicity**, and **ordering**.

## Creating Threads

```java
Thread t1 = new Thread(() -> work());          // Runnable (no result)
t1.start();                                     // start() != run()

ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Integer> f = pool.submit(() -> compute()); // Callable (returns/throws)
```

`Runnable.run` returns nothing and can't throw checked exceptions; `Callable.call` returns a value and may throw. Prefer executors over raw `Thread`. Calling `run()` directly executes on the current thread — you must call `start()` to spawn a new one.

## Thread Lifecycle

`NEW → RUNNABLE → (BLOCKED | WAITING | TIMED_WAITING) → TERMINATED`.

* **RUNNABLE** — running or ready to run.
* **BLOCKED** — waiting to acquire a monitor lock.
* **WAITING** — in `wait()`, `join()`, or `park()` with no timeout.
* **TIMED_WAITING** — timed variants (`sleep(ms)`, `wait(ms)`).
* **TERMINATED** — `run` finished.

## `synchronized` and Intrinsic Locks

Every object has an intrinsic **monitor**. `synchronized` acquires it, giving **mutual exclusion** (one thread at a time) and, crucially, a **memory barrier** (visibility of changes). Locking is reentrant — a thread can re-enter a lock it already holds.

```java
private final Object lock = new Object();
private int count;

void inc() {
    synchronized (lock) {   // atomic + establishes happens-before
        count++;
    }
}
```

`synchronized` on an instance method locks `this`; on a static method it locks the `Class` object. Always lock on a private, final object to avoid external interference. Hold locks for the shortest time possible.

## The Java Memory Model (JMM)

The JMM defines when a write by one thread becomes visible to a read by another. Without synchronization, the compiler, JIT, and CPU may **reorder** operations and cache values in registers, so one thread may never see another's update.

The central concept is **happens-before**: if action A happens-before B, A's effects are visible to B. Sources of happens-before edges:

* program order within a single thread;
* unlocking a monitor happens-before a later lock of the same monitor;
* a write to a `volatile` happens-before a later read of it;
* `Thread.start()` happens-before the thread's actions; a thread's actions happen-before another thread's successful `join()`;
* `final` field freeze at construction end.

## `volatile`

`volatile` guarantees **visibility** and prevents reordering across the access, but **not atomicity** of compound actions. Reads/writes go to main memory, so a flag written by one thread is seen by others.

```java
private volatile boolean running = true;   // safe stop flag
public void stop() { running = false; }
public void loop() { while (running) { /* ... */ } }
```

`volatile` is right for a status flag or a safely-published reference, but `count++` is still a race (read-modify-write) even if `count` is `volatile` — use an atomic or a lock for that.

## `wait`/`notify` and Condition Signalling

`wait()`, `notify()`, and `notifyAll()` coordinate threads on an object's monitor and must be called while holding that monitor. `wait` releases the lock and suspends until notified; always wait in a **loop** on the condition to guard against spurious wakeups and stale conditions.

```java
synchronized (queue) {
    while (queue.isEmpty()) queue.wait();   // loop, not if
    return queue.remove();
}
```

Prefer higher-level tools (`BlockingQueue`, `Condition`, `CountDownLatch`) over hand-written wait/notify.

## Concurrency Hazards

* **Race condition** — result depends on unsynchronized interleaving (e.g. check-then-act, read-modify-write).
* **Deadlock** — threads each hold a lock the other needs. Prevent by acquiring locks in a global order, using timeouts (`tryLock`), or reducing lock scope.
* **Livelock** — threads keep reacting to each other and make no progress.
* **Starvation** — a thread never gets CPU/lock time (e.g. unfair scheduling).

```java
// Deadlock-prone: two threads lock A,B in opposite orders.
// Fix: always lock in a consistent order (e.g. by identity hash / id).
```

## Thread Safety Strategies

Ordered roughly from safest:

1. **Immutability** — immutable objects need no synchronization.
2. **Confinement** — keep state in one thread (thread-local, stack).
3. **Stack confinement / local variables** — inherently thread-safe.
4. **Synchronization** — guard shared mutable state with a lock or `synchronized`.
5. **Concurrent/atomic utilities** — `ConcurrentHashMap`, `AtomicLong`, etc.

`ThreadLocal` gives each thread its own copy of a variable (e.g. `SimpleDateFormat`), but must be cleaned up in thread pools to avoid leaks/stale data.

## `sleep`, `yield`, `join`, Interruption

* `sleep(ms)` pauses without releasing locks.
* `yield()` hints the scheduler to let others run (advisory).
* `join()` waits for another thread to finish.
* **Interruption** is cooperative: `interrupt()` sets a flag; blocking calls throw `InterruptedException`. Never swallow it — restore the flag (`Thread.currentThread().interrupt()`) or propagate. `Thread.stop()` is unsafe and deprecated.

## Interview Q&A

**Q: Difference between `Runnable` and `Callable`?**
A: `Runnable.run` returns void and can't throw checked exceptions; `Callable.call` returns a value and may throw. `Callable` is submitted to an executor and yields a `Future`.

**Q: What does `synchronized` guarantee?**
A: Mutual exclusion on an object's monitor plus a memory barrier that establishes happens-before, making prior writes visible to the next lock holder. It is reentrant.

**Q: What is the Java Memory Model and happens-before?**
A: The JMM specifies visibility and ordering of memory operations across threads. If A happens-before B, A's effects are visible to B. Edges come from locks, `volatile`, `start`/`join`, and program order.

**Q: What does `volatile` guarantee and not guarantee?**
A: It guarantees visibility and prevents reordering for that field, but not atomicity of compound operations like `count++`. Use it for flags/safe publication, not counters.

**Q: Why call `wait` in a loop?**
A: To re-check the condition after waking, guarding against spurious wakeups and against another thread having changed the state before this one resumes.

**Q: What causes a deadlock and how do you prevent it?**
A: Two+ threads each holding a lock the other needs (circular wait). Prevent with a global lock ordering, `tryLock` with timeout, or minimizing lock scope.

**Q: `start()` vs `run()`?**
A: `start()` schedules the thread and invokes `run()` on the new thread; calling `run()` directly executes synchronously on the current thread with no new thread.

**Q: What is a race condition? Give an example.**
A: A bug where correctness depends on thread interleaving — e.g. check-then-act or read-modify-write (`count++`) without synchronization.

**Q: How should you handle `InterruptedException`?**
A: Don't swallow it: either propagate it or restore the interrupt flag with `Thread.currentThread().interrupt()` so higher layers can respond.

**Q: What is `ThreadLocal` used for and what's the risk?**
A: Per-thread state (e.g. formatters, request context). In thread pools it can leak memory or carry stale data if not `remove()`d after use.

**Q: Which thread-safety strategy is best?**
A: Immutability first, then confinement, then synchronization/concurrent utilities — minimize shared mutable state.

## Interview Notes

* Explain the three concurrency problems: visibility, atomicity, ordering.
* Define happens-before and list the edges (locks, `volatile`, `start`/`join`, final fields).
* State exactly what `volatile` does and doesn't do; contrast with `synchronized`.
* Know the thread lifecycle states and `wait`/`notify` in a loop.
* Enumerate deadlock/livelock/starvation and deadlock prevention.
* Rank thread-safety strategies (immutability → confinement → synchronization) and handle interruption correctly.
