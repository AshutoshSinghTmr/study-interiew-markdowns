# Java Concurrent Utilities

## The `java.util.concurrent` Package

`java.util.concurrent` (JUC) provides higher-level building blocks that make concurrency safer and more scalable than raw `Thread`/`synchronized`: executors and thread pools, futures, explicit locks, concurrent collections, atomics, and synchronizers. Prefer them over hand-rolled coordination.

## Executors and Thread Pools

An `ExecutorService` decouples task submission from thread management, reusing a pool of threads.

```java
ExecutorService pool = Executors.newFixedThreadPool(8);
Future<Integer> f = pool.submit(() -> compute());
pool.shutdown();                         // graceful: no new tasks, finish running
pool.awaitTermination(30, TimeUnit.SECONDS);
```

Factory options: `newFixedThreadPool` (bounded threads), `newCachedThreadPool` (elastic, unbounded — risky under load), `newSingleThreadExecutor` (serialized), `newScheduledThreadPool` (delayed/periodic), and `newVirtualThreadPerTaskExecutor` (Java 21). For production, construct a `ThreadPoolExecutor` directly to control the queue, max size, and rejection policy.

`ThreadPoolExecutor` parameters: core/max pool size, keep-alive, a `BlockingQueue` for pending tasks, a `ThreadFactory`, and a `RejectedExecutionHandler` (Abort, CallerRuns, Discard, DiscardOldest). An unbounded queue (as in `newFixedThreadPool`) can hide backpressure and cause OOM.

## `Future` and `CompletableFuture`

`Future` represents a pending result; `get()` blocks until done. Its limitation is that it can't be composed or completed externally.

`CompletableFuture` adds non-blocking composition, chaining, and combination:

```java
CompletableFuture
    .supplyAsync(() -> fetchUser(id), pool)          // run async
    .thenApply(User::profile)                        // transform
    .thenCompose(p -> loadPrefsAsync(p))             // flat-map another future
    .thenCombine(loadAdsAsync(), (prefs, ads) -> render(prefs, ads))
    .exceptionally(ex -> fallback(ex))               // recover
    .thenAccept(this::send);                         // consume
```

Use `thenApply` (map), `thenCompose` (flatMap), `thenCombine` (zip two), `allOf`/`anyOf` (fan-in), and `exceptionally`/`handle` (error handling). Always supply your own executor for `*Async` variants — the default common ForkJoinPool is shared and unsuitable for blocking work.

## Explicit Locks

`java.util.concurrent.locks` gives more control than `synchronized`:

* **`ReentrantLock`** — like `synchronized` but with `tryLock()` (with timeout), interruptible acquisition, and optional **fairness**. Must `unlock()` in a `finally`.
* **`ReadWriteLock` / `ReentrantReadWriteLock`** — many concurrent readers or one writer; good for read-heavy data.
* **`StampedLock`** — adds an **optimistic read** mode that avoids locking entirely when there's no contention, validating afterward. Not reentrant.

```java
lock.lock();
try { /* critical section */ } finally { lock.unlock(); }  // always unlock in finally
```

## Concurrent Collections

* **`ConcurrentHashMap`** — high-concurrency map using per-bin (CAS + fine-grained) locking; reads are lock-free. Atomic methods: `putIfAbsent`, `computeIfAbsent`, `merge`. No null keys/values. Iterators are weakly consistent.
* **`CopyOnWriteArrayList`/`CopyOnWriteArraySet`** — copy the backing array on every mutation; ideal for read-mostly data (e.g. listener lists) but expensive writes.
* **`BlockingQueue`** (`ArrayBlockingQueue`, `LinkedBlockingQueue`, `SynchronousQueue`) — thread-safe producer/consumer handoff with blocking `put`/`take`; the backbone of the producer-consumer pattern and thread pools.
* **`ConcurrentLinkedQueue`** — lock-free unbounded queue.

```java
BlockingQueue<Task> q = new LinkedBlockingQueue<>(1000);
// producer: q.put(task);   consumer: Task t = q.take();  (both block appropriately)
```

## Atomic Variables and CAS

The `java.util.concurrent.atomic` classes (`AtomicInteger`, `AtomicLong`, `AtomicReference`) provide lock-free, thread-safe updates via **compare-and-swap (CAS)** — a hardware instruction that atomically updates a value only if it still equals the expected one.

```java
AtomicLong counter = new AtomicLong();
counter.incrementAndGet();                 // atomic, no lock
ref.compareAndSet(expected, updated);      // CAS
```

`LongAdder`/`DoubleAdder` outperform `AtomicLong` under high contention by striping across cells. CAS can suffer the **ABA problem** (value changes A→B→A); `AtomicStampedReference` adds a version stamp to detect it.

## Synchronizers

* **`CountDownLatch`** — a one-shot gate; threads `await()` until a counter reaches zero (`countDown()`). Wait for N tasks to finish.
* **`CyclicBarrier`** — reusable; a fixed number of threads wait for each other at a barrier point, then all proceed (optionally running a barrier action).
* **`Semaphore`** — permits limiting concurrent access to a resource (e.g. a connection pool of size N).
* **`Phaser`** — a flexible, reusable, multi-phase barrier with dynamic party registration.
* **`Exchanger`** — two threads swap objects at a rendezvous.

## Fork/Join Framework

`ForkJoinPool` executes divide-and-conquer tasks (`RecursiveTask`/`RecursiveAction`) using **work-stealing**: idle threads steal subtasks from busy threads' deques, maximizing utilization. It powers parallel streams. Split until subtasks are small enough, then compute directly.

```java
class SumTask extends RecursiveTask<Long> {
    protected Long compute() {
        if (small()) return computeDirectly();
        var left = new SumTask(/* half */); left.fork();   // async
        var right = new SumTask(/* half */);
        return right.compute() + left.join();               // combine
    }
}
```

## Choosing the Right Tool

* Counter under contention → `LongAdder`; simple atomic → `AtomicInteger`.
* Shared map → `ConcurrentHashMap`; read-mostly list → `CopyOnWriteArrayList`.
* Producer/consumer → `BlockingQueue`.
* Limit concurrency → `Semaphore`; wait for completion → `CountDownLatch`.
* Async composition → `CompletableFuture`; task execution → `ExecutorService`.

## Interview Q&A

**Q: Why use an `ExecutorService` instead of creating threads?**
A: It reuses a bounded pool, decouples submission from execution, provides lifecycle management (`shutdown`), queueing, and rejection policies — avoiding the cost and risk of unbounded thread creation.

**Q: `Future` vs `CompletableFuture`?**
A: `Future.get()` blocks and can't be composed; `CompletableFuture` supports non-blocking chaining (`thenApply`/`thenCompose`), combination (`thenCombine`/`allOf`), and error handling (`exceptionally`/`handle`).

**Q: How does `ConcurrentHashMap` achieve concurrency?**
A: Fine-grained per-bin locking with CAS for updates and lock-free reads (Java 8+), instead of locking the whole map. It disallows nulls and has weakly-consistent iterators.

**Q: What is CAS and the ABA problem?**
A: Compare-and-swap atomically updates a value only if it equals the expected value, enabling lock-free algorithms. ABA is when a value changes A→B→A and CAS wrongly succeeds; `AtomicStampedReference` adds a version to detect it.

**Q: When is `CopyOnWriteArrayList` appropriate?**
A: Read-mostly scenarios (e.g. listener registries) where writes are rare, since each write copies the whole array.

**Q: `CountDownLatch` vs `CyclicBarrier`?**
A: A latch is one-shot: threads wait until a count hits zero. A barrier is reusable: a fixed set of threads wait for each other, then all proceed, and it resets.

**Q: What is a `Semaphore` for?**
A: Limiting the number of threads accessing a resource concurrently (permits), e.g. throttling to N connections.

**Q: What is work-stealing in the fork/join framework?**
A: Idle worker threads take tasks from the tails of busy workers' deques, balancing load for divide-and-conquer parallelism; it underlies parallel streams.

**Q: Why prefer `LongAdder` over `AtomicLong`?**
A: Under high contention `LongAdder` stripes updates across multiple cells, reducing CAS contention, and sums them on read — much higher throughput for counters.

**Q: Why avoid the common ForkJoinPool for blocking tasks?**
A: It's a shared, small pool sized to CPU cores; blocking tasks starve it and degrade everything using it (including parallel streams). Supply a dedicated executor.

## Interview Notes

* Explain `ThreadPoolExecutor` parameters (core/max, queue, rejection) and the danger of unbounded queues.
* Contrast `Future` with `CompletableFuture` and know the composition methods.
* Compare `ReentrantLock`/`ReadWriteLock`/`StampedLock` with `synchronized`.
* Describe `ConcurrentHashMap`, `CopyOnWriteArrayList`, and `BlockingQueue` use cases.
* Explain CAS, ABA, and `LongAdder` vs `AtomicLong`.
* Match synchronizers (`CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`) to scenarios and describe fork/join work-stealing.
