# Java Streams API

## What a Stream Is

A `Stream` is a lazily-evaluated pipeline over a source (collection, array, generator, I/O). It is **not** a data structure: it holds no storage, doesn't mutate its source, and is **single-use** (consumed once). Streams express *what* to compute declaratively, leaving *how* to the library.

```java
List<String> names = people.stream()
    .filter(p -> p.age() >= 18)
    .map(Person::name)
    .sorted()
    .toList();               // Java 16+ immutable list
```

## Pipeline Anatomy

A pipeline has three parts:

1. **Source** — `collection.stream()`, `Stream.of(...)`, `Arrays.stream(...)`, `IntStream.range(...)`, `Stream.iterate/generate`, `Files.lines(...)`.
2. **Intermediate operations** — return a new stream and are **lazy**: `map`, `filter`, `sorted`, `distinct`, `limit`, `peek`, `flatMap`. Nothing runs until a terminal op.
3. **Terminal operation** — triggers execution and produces a result or side effect: `collect`, `toList`, `reduce`, `forEach`, `count`, `findFirst`, `anyMatch`.

## Laziness and Fusion

Intermediate operations are deferred and **fused**: elements flow one at a time through the whole chain rather than materializing intermediate collections. This enables short-circuiting and efficiency.

```java
Optional<Integer> first = Stream.iterate(1, n -> n + 1)  // infinite
    .map(n -> n * n)
    .filter(n -> n > 50)
    .findFirst();            // stops as soon as one element qualifies
```

Because a stream is lazy and single-pass, an infinite source works as long as a short-circuiting terminal op (`findFirst`, `limit`, `anyMatch`) bounds it.

## `map` vs `flatMap`

`map` transforms each element 1:1. `flatMap` transforms each element into a stream and **flattens** the results into one stream — used to unnest collections or combine.

```java
List<String> words = lines.stream()
    .flatMap(line -> Arrays.stream(line.split(" ")))   // Stream<String[]> -> Stream<String>
    .distinct()
    .toList();
```

## Reduction and `collect`

`reduce` folds elements into a single value with an identity and an associative accumulator:

```java
int total = nums.stream().reduce(0, Integer::sum);
```

`collect` performs a **mutable reduction** into a container using a `Collector`. `Collectors` provides the common recipes:

```java
Map<Dept, List<Employee>> byDept =
    emps.stream().collect(Collectors.groupingBy(Employee::dept));

Map<Dept, Double> avgSalary =
    emps.stream().collect(Collectors.groupingBy(
        Employee::dept, Collectors.averagingDouble(Employee::salary)));

String csv = names.stream().collect(Collectors.joining(", ", "[", "]"));
Map<Boolean, List<Employee>> parts =
    emps.stream().collect(Collectors.partitioningBy(e -> e.salary() > 100_000));
```

Key collectors: `toList`/`toSet`/`toMap`, `groupingBy`, `partitioningBy`, `joining`, `counting`, `summingInt`, `averagingDouble`, `mapping`, `reducing`, and downstream composition.

## Primitive Streams

`IntStream`, `LongStream`, and `DoubleStream` avoid boxing and add numeric operations (`sum`, `average`, `summaryStatistics`, `range`). Convert with `mapToInt`/`boxed`.

```java
IntSummaryStatistics stats = emps.stream()
    .mapToInt(Employee::age)
    .summaryStatistics();       // count, min, max, sum, average in one pass
```

## Stateless vs Stateful, Short-Circuiting

* **Stateless** ops (`map`, `filter`) process each element independently.
* **Stateful** ops (`sorted`, `distinct`, `limit`) may need to see other elements or buffer, which limits parallel efficiency (`sorted` must see everything).
* **Short-circuiting** ops (`findFirst`, `anyMatch`, `limit`) can terminate early without processing the whole source.

## Parallel Streams

`parallelStream()` (or `.parallel()`) splits the source across the common ForkJoinPool. It can speed up CPU-bound work on large, easily-splittable sources (arrays, `ArrayList`) with stateless, associative operations — but is often *slower* or *wrong* when misused.

Pitfalls:

* Operations must be **stateless** and side-effect-free; sharing mutable state causes races.
* `reduce`/collector combiners must be **associative**.
* Poorly-splittable sources (`LinkedList`, I/O) parallelize badly.
* It uses the shared common pool, so blocking tasks can starve the whole application.
* Ordering-sensitive ops (`findFirst`, `forEachOrdered`) add coordination cost.

Measure before parallelizing; default to sequential.

## Common Pitfalls

* A stream can be consumed only once — reusing one throws `IllegalStateException`.
* `peek` is for debugging, not for side-effecting logic; it may be skipped under optimization.
* `Collectors.toMap` throws on duplicate keys unless you pass a merge function.
* Don't mutate the source collection during a stream operation.
* `forEach` order is undefined on parallel streams; use `forEachOrdered` if order matters.

## Interview Q&A

**Q: What is a stream and how does it differ from a collection?**
A: A stream is a lazy, single-use pipeline for computation, not storage. It doesn't hold or mutate data; a collection is an in-memory data structure you can traverse repeatedly.

**Q: What are intermediate vs terminal operations?**
A: Intermediate ops (`map`, `filter`, `sorted`) are lazy and return a new stream; terminal ops (`collect`, `reduce`, `forEach`, `count`) trigger execution and produce a result. Nothing runs without a terminal op.

**Q: What does laziness enable?**
A: Operation fusion (elements flow one at a time), short-circuiting, and processing of infinite sources bounded by ops like `limit`/`findFirst`.

**Q: `map` vs `flatMap`?**
A: `map` is a 1:1 transform; `flatMap` maps each element to a stream and flattens all of them into one stream (unnesting).

**Q: `reduce` vs `collect`?**
A: `reduce` folds into a single immutable value with an associative function; `collect` is a mutable reduction into a container via a `Collector` (grouping, joining, toMap, etc.).

**Q: When do parallel streams help, and what are the risks?**
A: They help for large, splittable sources with CPU-bound, stateless, associative operations. Risks: races from shared state, non-associative reducers, common-pool starvation from blocking, and overhead that can make them slower than sequential.

**Q: Why are `sorted` and `distinct` special?**
A: They are stateful — they may buffer or need to see other elements — which reduces parallel efficiency and can require full materialization.

**Q: What happens if you reuse a stream?**
A: It throws `IllegalStateException`; streams are single-use. Create a new stream from the source instead.

**Q: How do primitive streams help?**
A: `IntStream`/`LongStream`/`DoubleStream` avoid boxing and offer numeric ops like `sum`, `average`, and `summaryStatistics`.

**Q: Why can `Collectors.toMap` throw?**
A: On duplicate keys it throws `IllegalStateException` unless you provide a merge function to resolve collisions.

## Interview Notes

* Describe the source → intermediate → terminal pipeline and laziness/fusion.
* Distinguish `map` vs `flatMap` and `reduce` vs `collect` with `Collectors`.
* Know the main collectors: `groupingBy`, `partitioningBy`, `joining`, `toMap`, downstream collectors.
* Explain stateless/stateful/short-circuiting ops and their parallel implications.
* Be candid about parallel-stream pitfalls (state, associativity, common pool) and "measure first."
* Note single-use semantics, `peek` caveats, and `toMap` duplicate-key handling.
