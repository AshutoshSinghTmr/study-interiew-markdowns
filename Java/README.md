# Java Interview Preparation

A standalone, revision-ready knowledge base for core Java interviews. Each file is a self-contained topic in a consistent format:

* concise, internals-aware prose with focused code examples;
* an **Interview Q&A** section of rapid question → answer flashcards for active recall;
* an **Interview Notes** section listing the key points to be able to discuss.

**Target version:** Java 21 (LTS) — records, sealed types, pattern matching, virtual threads. Differences in Java 8/11/17 are called out where they still come up in interviews.

## Table of Contents

- [How to Revise](#how-to-revise)
- [Topics](#topics)
  - [Language & OOP](#language--oop)
  - [Collections & Functional](#collections--functional)
  - [Concurrency](#concurrency)
  - [JVM & Memory](#jvm--memory)
  - [Platform & Modern Java](#platform--modern-java)

## How to Revise

1. Read a topic top-to-bottom once for understanding.
2. On later passes, cover the answers and drill the **Interview Q&A** flashcards.
3. Use the **Interview Notes** as a final pre-interview checklist per topic.

## Topics

### Language & OOP

| Topic | Focus |
| --- | --- |
| [OOP and Class Design](OOP_and_Class_Design.md) | Encapsulation/inheritance/polymorphism, interfaces vs abstract classes, `record`, `sealed`, nested classes |
| [Object Fundamentals](Object_Fundamentals.md) | `equals`/`hashCode` contract, `Comparable`/`Comparator`, immutability, defensive copies, `Object` methods |
| [Generics](Generics.md) | Type parameters, bounded types, wildcards, PECS, type erasure, bridge methods, heap pollution |
| [Exceptions and Error Handling](Exceptions_and_Error_Handling.md) | Checked vs unchecked, hierarchy, try-with-resources, multi-catch, suppressed exceptions, best practices |
| [Annotations and Reflection](Annotations_and_Reflection.md) | Meta-annotations, retention/target, reflection API, `setAccessible`, dynamic proxies, annotation processing |

### Collections & Functional

| Topic | Focus |
| --- | --- |
| [Collections Framework](Collections_Framework.md) | Hierarchy, `HashMap`/`ArrayList`/`TreeMap` internals, load factor/treeify, fail-fast vs fail-safe |
| [Functional Programming and Lambdas](Functional_Programming_and_Lambdas.md) | Functional interfaces, method references, capture rules, closures, `invokedynamic` |
| [Streams API](Streams_API.md) | Pipeline model, laziness, `Collectors`, primitive/parallel streams, `flatMap`, short-circuiting |

### Concurrency

| Topic | Focus |
| --- | --- |
| [Concurrency and Multithreading](Concurrency_and_Multithreading.md) | Thread lifecycle, `synchronized`/monitors, `volatile`, Java Memory Model, deadlock/livelock |
| [Concurrent Utilities](Concurrent_Utilities.md) | Executors, `CompletableFuture`, locks, concurrent collections, atomics/CAS, synchronizers, fork/join |
| [Virtual Threads and Loom](Virtual_Threads_and_Loom.md) | Platform vs virtual threads, carrier threads, structured concurrency, scoped values, pinning |

### JVM & Memory

| Topic | Focus |
| --- | --- |
| [JVM Internals and Memory](JVM_Internals_and_Memory.md) | JDK/JRE/JVM, bytecode, class loading & delegation, runtime data areas, JIT, escape analysis |
| [Garbage Collection](Garbage_Collection.md) | Reachability, generational GC, G1/ZGC/Shenandoah, STW, tuning, reference types, leak diagnosis |

### Platform & Modern Java

| Topic | Focus |
| --- | --- |
| [Modern Features 8 to 21](Modern_Features_8_to_21.md) | `var`, records, sealed, pattern matching, switch expressions, text blocks, `Optional`, version timeline |
| [Strings and String Pool](Strings_and_String_Pool.md) | Immutability, string pool/interning, `StringBuilder`, compact strings, concatenation internals |
| [IO and NIO](IO_and_NIO.md) | Byte/char streams, buffering, NIO buffers/channels/selectors, `Path`/`Files`, serialization risks |
| [Date and Time API](Date_and_Time_API.md) | `java.time` types, `Instant`/zones, `Duration`/`Period`, formatting, legacy `Date`/`Calendar` migration |
| [Modules (JPMS)](Modules_JPMS.md) | `module-info.java`, `requires`/`exports`/`opens`, strong encapsulation, `jlink`/`jdeps`, migration |
