---
title: "Comparing JVM and GraalVM Native Image in a Serverless Environment"
date: 2026-04-29
translationKey: "posts/serverless-jvm-graalvm-benchmark"
categories:
  - java
image: "../../images/java.webp"
tags:
  - java
  - jvm
  - graalvm
  - serverless
  - spring
  - ktor
  - quarkus
---

In a serverless environment, how quickly an application can respond to the first request matters a lot. Unlike a server where the process is always warm, cold start latency directly affects the user experience. Memory usage is also easy to connect directly to cost.

Previously, I wrote about Leyden, Loom, and related work in [The JVM Is Still Evolving](../jvm-evolution/). The JVM is not just an old runtime. It is still evolving in the direction of better startup time, warm-up, and memory efficiency. Since Java 25 has started to include improvements related to Leyden, this time I wanted to compare "how much difference GraalVM Native Image really makes" and "how far the Java 25 JVM can go" under serverless-style conditions.

The repository I used for the comparison is [Serverless](https://github.com/retheviper/Serverless). The HTML summary of the results is available as [serverless-summary.html](https://github.com/retheviper/Serverless/blob/main/docs/reports/serverless-summary.html).

## What I compared

This time, I compared applications with the same HTTP API and the same PostgreSQL access using the following configurations.

| Target | Runtime |
|---|---|
| Quarkus | JVM Java 25 |
| Quarkus | GraalVM Native Image Java 25 |
| Spring Boot | JVM Java 25 |
| Spring Boot | GraalVM Native Image Java 25 |
| Ktor | JVM Java 25 |
| Ktor | GraalVM Native Image Java 25 |

Ktor Kotlin/Native is also included separately, but it is not part of the database-backed comparison. The reason is simple: JDBC is a JVM API, so it cannot share the same PostgreSQL path as the JVM / GraalVM Native Image targets. For now, I think it should be treated as an experimental server-start measurement.

The API shape was aligned like this.

| Endpoint | Work |
|---|---|
| `GET /health` | JSON health response |
| `GET /api/items/{id}` | PostgreSQL point read and JSON object serialization |
| `GET /api/items?limit=50` | PostgreSQL ordered list read and JSON array serialization |
| `GET /api/items/report` | PostgreSQL aggregate query and JSON array serialization |

I did not want to measure only "the process started." I wanted to include database connection, JSON serialization, DI/component graph initialization, and HTTP server startup. So the main metric is `apiReadyMillis`, which measures the time until `/api/items/1` actually responds.

## Results

Looking at the medians from the HTML summary, the result was quite clear.

| Target | Median API ready ms | Peak RSS MiB | Median steady RSS MiB |
|---|---:|---:|---:|
| Quarkus GraalVM Native | 81 | 41.2 | 27.3 |
| Spring Boot GraalVM Native | 118 | 98.9 | 60.4 |
| Ktor GraalVM Native | 243 | 34.9 | 28.9 |
| Ktor JVM | 641 | 160.4 | 134.1 |
| Quarkus JVM | 710 | 151.1 | 129.1 |
| Spring Boot JVM | 1716 | 332.3 | 270.7 |

The most visible point is still the startup speed of GraalVM Native Image. Quarkus Native had a median of 81ms, Spring Native 118ms, and Ktor Native 243ms. The JVM versions use Java 25, so they should already be much better than older JVMs, but for cold starts the Native Image gap is still large.

Memory follows the same pattern. Ktor Native and Quarkus Native stayed around 30MiB steady RSS. On the other hand, the JVM versions of Ktor and Quarkus were around 130MiB, and Spring Boot was around 270MiB. In an environment like serverless, where allocated memory can affect pricing and concurrency, this is not a small difference.

## Differences by framework

Quarkus looked very strong for this kind of use case. With Native Image, both startup time and memory were small, and even the JVM version was much lighter than Spring Boot. Since Quarkus is designed to use build-time processing heavily, I expected it to work well with GraalVM Native Image, but the actual database-backed API ready numbers were also very good.

Spring Boot was the heaviest in the JVM version. I do not think this means Spring is bad. It is more natural to see the thickness of DI, auto configuration, and application context initialization showing up directly in cold start time. But once built as a Native Image, it dropped to 118ms, so the effect of Spring AOT and GraalVM is quite large.

Ktor on the JVM was close to Quarkus JVM, but with Native Image it was slower than Quarkus and Spring. In this Ktor Native target, I used the CIO engine and avoided relying on Ktor's runtime serializer lookup by explicitly calling `kotlinx.serialization` serializers. Also, to avoid Hikari native-image metadata issues, the database-backed target uses a small eager JDBC pool implemented inside the repository. In other words, the numbers include not only Ktor itself, but also implementation constraints needed for Native Image support.

## Implementation constraints

GraalVM Native Image was not just a matter of compiling a JVM application as-is.

There were several constraints this time as well.

- Quarkus native build cannot produce the normal JAR package and native executable in the same build
- Spring native build needs Spring AOT and the GraalVM Native Build Tools plugin
- Kotlin Spring applications need `kotlin-reflect` so Spring AOT can read Kotlin metadata
- Ktor GraalVM native needs the CIO engine
- Ktor native has parts where serializer lookup and Hikari are difficult to use as-is
- Kotlin/Native cannot use JDBC, so it belongs outside the database-backed JVM/GraalVM comparison

I think these points are important when reading the benchmark numbers. Native Image is clearly fast and light, but in exchange you need to care about build time, reflection, dynamic proxies, serializers, JDBC drivers, and framework integration.

Of course, Quarkus and Spring Boot absorb quite a lot of this. That is why a framework's Native Image support can directly become a difference in practicality. With a thinner framework like Ktor, there are more things to handle yourself, but it is also easier to see where things are happening.

## Difference from the previous five-stack comparison

Previously, in [Comparing five Spring and Ktor stacks](../spring-ktor-five-stack-benchmark/), I compared Spring MVC JDBC, Spring MVC virtual thread, Spring WebFlux R2DBC, Ktor JDBC, and Ktor R2DBC using the same PostgreSQL.

At that time, I mainly looked at throughput, so the main interest was "which stack is strong while continuously receiving requests." The result was that a traditional configuration like Spring MVC + JDBC was still quite strong, and async or R2DBC did not automatically win.

This time, I changed the viewpoint and looked at serverless-style cold start and memory. In other words, the metric is not steady-state req/s, but the time from process start until the first database-backed JSON response.

These are both performance comparisons, but they look at different things. For a long-running server, throughput and latency distribution are important. For serverless or something close to scale-to-zero, cold start and RSS matter a lot. So I think the previous result and this result are not contradictory. They are different axes.

## Environment can change the result

This measurement was done on arm macOS. Native binaries are host-specific, so the same gap may not appear on x64 Linux.

Real serverless environments are usually Linux, and results can change depending on CPU architecture, container runtime, filesystem, network, distance to PostgreSQL, memory limits, and process startup behavior. Native Image in particular is affected by the OS / architecture it was built for, so the final measurement should be done in an environment close to the deployment target.

Also, this benchmark uses `SERVERLESS_DEPENDENCY_COUNT` with a default of 180 to roughly model an application with a lot of DI. But real Spring applications can have more beans, auto configuration, configuration properties, external clients, security filters, metrics, and so on. As the DI target count grows, it may affect Spring Boot JVM startup time and memory more strongly, and it may also increase the complexity of reachability metadata and AOT processing on the Native Image side.

So these numbers are measurements for this specific condition. They should not be generalized directly to every Spring / Ktor / Quarkus application.

## Finally

Based on this result, if immediate response and memory efficiency matter in a serverless environment, GraalVM Native Image is a very strong option. In particular, Quarkus Native and Spring Native became ready in a very short time even including database connection and JSON response.

On the other hand, I do not think the JVM is a dead option. Even with the Java 25 JVM versions, Ktor and Quarkus reached API ready in under one second, and with improvements like Leyden continuing, the gap may shrink further. Considering JVM flexibility, ease of debugging, and compatibility with existing libraries, it is not as simple as always choosing Native Image.

Still, if we look only at cold start and memory numbers, Native Image has a clear advantage today. For a long-running server, we should look at throughput like in the previous five-stack comparison, but for serverless we need another axis of judgment.

In the end, it depends on the environment, how often cold starts happen, and how much memory cost matters. I want to keep improving [Serverless](https://github.com/retheviper/Serverless) as one piece of material for making that decision.

See you soon!
