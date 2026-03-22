---
title: "The JVM Is Still Evolving"
date: 2025-07-08
translationKey: "posts/jvm-evolution"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - jvm
  - java
  - kotlin
  - spring
---
As modern programming environments embrace lightweight runtimes such as Go and Rust, some are beginning to question the direction of the JVM. This is also the case for me, and especially in cloud environments, there are many cases where JVM startup time and memory usage are issues. So in a serverless environment, I sometimes choose Python or Node.js purely for startup time.

However, the JVM continues to have room for performance improvements through new technology. This article reviews major ongoing projects, including Leyden and Loom, and considers how JVM performance can be improved in the future.

## From JIT to AOT + PGO

[JIT (Just-In-Time Compiler)](https://en.wikipedia.org/wiki/Just-in-time_compilation) is a technology that compiles bytecode into native code at runtime.
While it is highly dependent on the ability to generate optimized code at runtime, the undefined performance due to the warm-up period [^1] and "jitter[^2]" cannot be ignored.

[Graal](https://www.graalvm.org) was designed to address those issues. It is a high-performance JIT compiler intended as a replacement for C2, and it reduces JIT instability through optimizations such as inlining, escape analysis, and vectorization.

For example, a [Twitter case study](https://www.oracle.com/a/ocom/docs/graalvm-twitter-casestudy-constellation.pdf) reported an 8-11% reduction in CPU usage with Graal JIT, and a Kotlin microbenchmark reported a [+18% performance improvement](https://martijndwars.nl/2020/02/24/graal-vs-c2.html).

## Leyden achieves both quick startup and stability

[Project Leyden](https://openjdk.org/projects/leyden/) aims to radically reduce the startup and warm-up times of JVM applications.

Traditional JIT-based optimization is an "adaptive" optimization method that incrementally increases performance by collecting profile information early in the runtime, but Leyden is replacing it with "static" optimization.

To support that direction, Leyden introduces the idea of "condensers," archives that pre-store information such as:

* List of classes used and linking information
* Initial state of heap (AppCDS + heap snapshot)
* Profile information (PGO)
* Compiled code

As a result, it is expected that the performance will be similar to that after JIT immediately after application startup, and jitter will be kept to a minimum.

For example, the [Quarkus](https://quarkus.io/) team reported the following improvements based on Leyden's initial implementation:

* Reduce boot time by over 50%
* Achieves response speed close to Native Image while retaining JVM flexibility
* Up to 30% reduction in memory usage

Leyden's related JEPs include [JEP 483](https://openjdk.org/jeps/483) (class loading/linking presave) and [JEP 515](https://openjdk.org/jeps/515) (PGO support), which will be gradually included in JDK 25 and later.

## ZGC, Shenandoah, and LXR

GC is a factor that greatly affects JVM performance. The recently introduced [ZGC](https://wiki.openjdk.org/display/zgc/Main) and [Shenandoah GC](https://wiki.openjdk.org/display/shenandoah/Main) aim for "pause-less GC", and ZGC achieves GC pauses of at least <1ms even on JVMs with large heaps.

The latest [LXR GC](https://www.steveblackburn.org/pubs/papers/lxr-pldi-2022.pdf) (research stage) shows performance that is 9 times better than ZGC, and there are reports that it reduces tail latency by 30 times and improves throughput by 6 times.

[Cassandra Benchmark](https://www.datastax.com/blog/apache-cassandra-benchmarking-40-brings-heat-new-garbage-collectors-zgc-and-shenandoah) shows Shenandoah's impact on real-world services, such as reducing p99 latency by 77%.

These are important for reducing performance jitter while preserving throughput, from small services to vast JVM graphs.

## Memory Efficiency with Valhalla

[Project Valhalla](https://openjdk.org/projects/valhalla/) aims to reduce dependence on heap objects and improve cache locality by bringing value classes to the JVM.

Normal objects are scattered across the heap and carry object headers, which increases GC pressure and cache misses. Value classes, by contrast, can be inlined like primitive structs, improving memory efficiency.

Kotlin data classes are likely to benefit significantly from Valhalla, and some experiments suggest [multi-fold improvements](https://www.reddit.com/r/java/comments/1dnhgut/i_actually_tested_jdk_20_valhalla_here_are_my/) in paths such as arrays of values and vector-heavy computation.

## Achieving scalable I/O concurrency with Loom

[Project Loom](https://openjdk.org/projects/loom/) is an attempt to use virtual threads to handle large numbers of concurrent connections while keeping traditional blocking I/O code intact.

What is important here is the relationship with existing parallel processing.

Traditional Java (and Spring MVC) was an architecture that consumed one OS thread per request.
However, this quickly leads to performance degradation due to thread exhaustion and context switching when the number of simultaneous connections increases.

[Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html) was introduced to solve this problem.
WebFlux is a non-blocking asynchronous model based on [Reactor](https://projectreactor.io/) that makes it possible to handle a large number of requests with a small number of threads.

However, the trade-off was that developers were forced to understand `Mono` and `Flux` and build asynchronous pipelines, making debugging difficult. Fortunately, Kotlin allows you to write asynchronous processing easily by using the `suspend` function, but Java still requires complex code.

Loom's virtual thread, introduced here, is revolutionary in that it has the scalability of WebFlux, but can be written using traditional ``synchronous code.'' The code itself also remains largely the same as the existing thread-based code.

Virtual threads were officially introduced in Java 21, and [Spring MVC's `VirtualThreadTaskExecutor`](https://docs.spring.io/spring-framework/reference/integration/virtual-threads.html) is a representative example. They can be introduced into traditional servlet-based applications with relatively small changes, and some benchmarks report [lower latency and higher throughput than WebFlux](https://github.com/chrisgleissner/loom-webflux-benchmarks).

Some application servers such as Netty and Tomcat are also supporting Loom, and I think virtual threads will become the standard option in the future.

## Summary

Below is a summary of the major JVM improvements introduced in this article.

* [Graal](https://www.graalvm.org/) and [Leyden](https://openjdk.org/projects/leyden/) reduce JIT instability and warm-up time
* [ZGC](https://wiki.openjdk.org/display/zgc/Main), [Shenandoah](https://wiki.openjdk.org/display/shenandoah/Main), [LXR GC](https://arxiv.org/abs/2210.17175) realize low-latency GC on heap
* [Valhalla](https://openjdk.org/projects/valhalla/) improves memory execution efficiency
* Improved I/O scalability with [Loom](https://openjdk.org/projects/loom/)

## Finally

While the JVM has a long history, its limitations have often been talked about in recent years.

But projects like Leyden and Loom mentioned here are trying to reimagine Java's value not just as compatibility, but as a performance foundation for today's needs.

In particular, languages ​​other than Java, such as Kotlin and Scala, can take advantage of these JVM improvements, so we can look forward to future developments.

Application development paradigms and performance infrastructure are still changing, and we don't know what will happen in the future, but I hope that technological innovations like this will continue.

See you soon!

[^1]: The initial stage necessary for JIT to perform optimization.
[^2]: Variation in execution time
