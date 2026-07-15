# Java Collections Framework

## Structure of the Framework

The Collections Framework is a unified architecture of interfaces, implementations, and algorithms rooted at `Iterable`:

```
Iterable
└── Collection
    ├── List   (ordered, indexed, duplicates)      -> ArrayList, LinkedList
    ├── Set    (no duplicates)                      -> HashSet, LinkedHashSet, TreeSet
    └── Queue/Deque (ends-based)                    -> ArrayDeque, PriorityQueue, LinkedList

Map (not a Collection)                              -> HashMap, LinkedHashMap, TreeMap, ConcurrentHashMap
```

`Map` sits outside `Collection` because it models key→value associations, not a group of elements. The framework separates the *contract* (interface) from the *implementation*, so you program to `List`/`Map` and swap implementations freely.

## `ArrayList` vs `LinkedList`

* **`ArrayList`** — backed by a resizable array. O(1) random access and amortized O(1) append; O(n) insert/remove in the middle (shifting). Cache-friendly and the default choice.
* **`LinkedList`** — a doubly linked list. O(1) insert/remove at the ends (and at a known node), but O(n) indexed access and poor cache locality due to node chasing.

In practice `ArrayList` wins almost always; `LinkedList` is rarely the right list and is mainly useful as a `Deque`. `ArrayList` grows by ~1.5×; presizing with `new ArrayList<>(capacity)` avoids repeated copies.

## `HashMap` Internals

`HashMap` stores entries in an array of **buckets** indexed by hash. Key mechanics:

* **Hashing** — the key's `hashCode()` is spread (`h ^ (h >>> 16)`) to mix high bits, then masked to a bucket index (`(n - 1) & hash`, since capacity is a power of two).
* **Collisions** — entries in the same bucket form a linked list. Since Java 8, a bucket with **> 8** entries (and table size ≥ 64) is **treeified** into a red-black tree, improving worst-case lookup from O(n) to O(log n). It untreeifies below 6.
* **Load factor & resize** — default load factor 0.75. When `size > capacity * loadFactor`, the table doubles and entries are **rehashed** into the larger table. Resizing is O(n) and momentarily costly.
* **Null** — `HashMap` permits one null key (bucket 0) and null values.

```java
Map<String, Integer> counts = new HashMap<>();
counts.merge("a", 1, Integer::sum);          // idiomatic counting
counts.computeIfAbsent("b", k -> new ...);   // lazy init
```

Correct `equals`/`hashCode` on keys is essential — a broken contract makes entries unfindable.

## `LinkedHashMap` and `TreeMap`

* **`LinkedHashMap`** — a `HashMap` plus a doubly linked list threading entries in insertion (or access) order, giving predictable iteration. With `accessOrder=true` and an overridden `removeEldestEntry`, it becomes a simple **LRU cache**.
* **`TreeMap`** — a red-black tree keeping keys **sorted** by natural order or a `Comparator`. O(log n) operations and navigation methods (`floorKey`, `ceilingKey`, `subMap`). Does not allow a null key.

## Set Implementations

`HashSet` wraps a `HashMap` (elements are keys), so it inherits O(1) average operations and no ordering. `LinkedHashSet` preserves insertion order; `TreeSet` keeps elements sorted with O(log n) operations and navigation. Membership relies on `equals`/`hashCode` (hash sets) or `compareTo`/`Comparator` (tree sets).

## Queues and Deques

* **`ArrayDeque`** — a resizable-array double-ended queue; the preferred stack and queue (faster than `Stack`/`LinkedList`, no synchronization). No null elements.
* **`PriorityQueue`** — a binary heap ordering elements by natural order or comparator; `poll()` returns the smallest. Not sorted on iteration — only the head is ordered.
* **`Stack`** — a legacy synchronized `Vector` subclass; avoid it in favour of `ArrayDeque`.

## Iterators: Fail-Fast vs Fail-Safe

* **Fail-fast** iterators (`ArrayList`, `HashMap`, ...) track a `modCount`; structural modification during iteration (other than via the iterator's own `remove`) throws `ConcurrentModificationException`. This is a best-effort bug detector, not a guarantee.
* **Fail-safe** (weakly consistent) iterators (`ConcurrentHashMap`, `CopyOnWriteArrayList`) iterate over a snapshot or tolerate concurrent changes without throwing, though they may not reflect the latest state.

```java
for (var it = list.iterator(); it.hasNext();) {
    if (drop(it.next())) it.remove();   // safe removal during iteration
}
list.removeIf(this::drop);              // cleaner equivalent
```

## Utility Classes and Immutable Collections

`Collections` provides algorithms and views (`sort`, `binarySearch`, `unmodifiableList`, `synchronizedMap`, `emptyList`); `Arrays` bridges arrays and lists (`Arrays.asList`, fixed-size and backed by the array). Java 9 factory methods `List.of`, `Set.of`, `Map.of` create compact **immutable** collections that reject nulls and throw on mutation; `List.copyOf` makes a defensive immutable copy. Note `Arrays.asList` returns a fixed-size list that reflects and writes through to the array.

## Choosing a Collection

* Need indexed, ordered, duplicates → `ArrayList`.
* Need uniqueness, no order → `HashSet`; sorted → `TreeSet`; insertion order → `LinkedHashSet`.
* Need key→value, no order → `HashMap`; sorted keys → `TreeMap`; insertion/access order or LRU → `LinkedHashMap`.
* Need FIFO/stack/deque → `ArrayDeque`; priority ordering → `PriorityQueue`.
* Need concurrency → `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue` (see Concurrent Utilities).

## Interview Q&A

**Q: How does `HashMap` work internally?**
A: Keys are hashed (with bit-spreading) into buckets of a power-of-two array. Collisions chain in a list, treeified to a red-black tree beyond 8 entries (table ≥ 64). It resizes (doubles + rehashes) when size exceeds capacity × load factor (0.75).

**Q: Why does `HashMap` treeify buckets?**
A: To bound worst-case lookup at O(log n) instead of O(n) when many keys collide (e.g. from poor or adversarial hashing).

**Q: `ArrayList` vs `LinkedList` — which and when?**
A: `ArrayList` almost always: O(1) random access, cache-friendly, amortized O(1) append. `LinkedList` only for heavy end insertion/removal as a deque, but it has O(n) indexing and poor locality.

**Q: What is the load factor?**
A: The fullness threshold (default 0.75) at which the table resizes. Lower reduces collisions but wastes space; higher saves space but increases collisions.

**Q: What is fail-fast vs fail-safe iteration?**
A: Fail-fast iterators throw `ConcurrentModificationException` on structural change during iteration (via `modCount`). Fail-safe/weakly-consistent iterators (concurrent collections) iterate a snapshot without throwing.

**Q: How do you remove elements while iterating safely?**
A: Use the iterator's own `remove()`, or `Collection.removeIf(...)`. Never modify the collection directly inside a for-each.

**Q: Why is `Map` not part of `Collection`?**
A: It models key→value pairs, not a single group of elements, so it has a distinct interface (though its `keySet`/`values`/`entrySet` are collection views).

**Q: What happens if a key's `hashCode`/`equals` is broken?**
A: Entries may land in the wrong bucket or be considered unequal, so `get`/`containsKey` fail to find them — silent data loss.

**Q: `HashMap` vs `TreeMap` vs `LinkedHashMap`?**
A: `HashMap` is unordered O(1) average; `TreeMap` is sorted O(log n) with navigation; `LinkedHashMap` keeps insertion/access order and can back an LRU cache.

**Q: Why prefer `ArrayDeque` over `Stack`?**
A: `Stack` extends the legacy synchronized `Vector`; `ArrayDeque` is faster, unsynchronized, and the modern choice for both stack and queue use.

**Q: What do `List.of` and `List.copyOf` return?**
A: Compact immutable collections that reject null elements and throw `UnsupportedOperationException` on modification; `copyOf` makes a defensive immutable copy of an existing collection.

## Interview Notes

* Sketch the interface hierarchy and why `Map` is separate.
* Explain `HashMap` internals: hashing/spreading, buckets, treeify at 8, load factor 0.75, resize/rehash.
* Compare `ArrayList` vs `LinkedList` on complexity and cache behaviour.
* Distinguish fail-fast (modCount, CME) from fail-safe iterators and safe removal patterns.
* Map requirements to implementations, including sorted/ordered/LRU/concurrent variants.
* Mention immutable factory methods and the `equals`/`hashCode` dependency for keys.
