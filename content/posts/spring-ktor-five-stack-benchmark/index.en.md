---
title: "Spring vs. Ktor: Comparing Five Backend Configurations"
date: 2026-03-20
translationKey: "posts/spring-ktor-five-stack-benchmark"
categories:
  - kotlin
image: ../../images/kotlin.webp
tags:
  - spring
  - ktor
  - benchmark
  - r2dbc
  - jdbc
  - virtual-thread
  - kotlin
---

Recently, since [Exposed](https://www.jetbrains.com/exposed/) 1.0 has officially been able to handle R2DBC, it has become easier to configure the entire system in Ktor, including DB access, in a non-blocking manner. Originally, Ktor can be written based on Coroutine, so if you have R2DBC here, you might be a little hopeful that it will be able to achieve fairly high concurrent processing performance while maintaining a Kotlin-like configuration.

On the other hand, Spring has also become much easier to handle virtual threads since Spring Boot 3 series. In that case, I am also concerned about how to compare the orthodox configuration such as conventional Spring MVC + JDBC and the reactive configuration such as WebFlux + R2DBC. Furthermore, if you add Ktor + JDBC and Ktor + R2DBC to that, it will become much easier to see the differences between frameworks, serializers, ORMs, and drivers.

So this time, I compared the following five configurations under the same conditions using the same PostgreSQL and the same HTTP scenario.

## Benchmark Targets

The following five configurations were compared this time.

| Configuration | Main stack |
|---|---|
| Spring MVC JDBC (platform) | Tomcat, Jackson, jOOQ, JDBC, HikariCP, platform threads |
| Spring MVC JDBC (virtual) | Tomcat, Jackson, jOOQ, JDBC, HikariCP, virtual threads |
| Spring WebFlux R2DBC | Netty, Jackson, jOOQ SQL builder, R2DBC PostgreSQL, R2DBC pool |
| Ktor JDBC | Netty, kotlinx.serialization, Koin, Exposed JDBC, HikariCP |
| Ktor R2DBC | Netty, kotlinx.serialization, Koin, Exposed R2DBC, R2DBC PostgreSQL |

Here, with Spring MVC, the application itself is the same, only enabling/disabling the virtual thread. On the Ktor side, by separating the JDBC version and the R2DBC version, we have made it easier to see whether it is a "difference in Ktor itself" or "a difference in R2DBC + Exposed R2DBC."

Personally, I quite like the configurations provided by JetBrains official libraries like Ktor + kotlinx.serialization + Exposed. The amount of code is relatively easy to keep down, and it feels like it's written in Kotlin. However, likes and dislikes and performance are two different things, so the starting point for this article is to take a look at the numbers.

## Test scenario and measurement method

This time, rather than simply looking at "which one is faster," we have divided the scenarios into several types to make it easier to identify which layer makes the difference. Light response without using DB, simple DB read, large payload, search and aggregation-like processing, and write and contention.

The intent of the scenario is roughly as follows.

| Classification | Scenario | Things to see |
|---|---|---|
| non-DB | `GET /health` | The lightest HTTP route. Differences in routing and minimal response processing are likely to appear |
| small payload | `GET /api/books/bench/small-json` | Small JSON without DB. Easy to see the simplicity of the framework and serializer |
| large payload | `GET /api/books/bench/large-json?limit=200` | View large JSON assembly and serialization load |
| point read | `GET /api/books/{id}` | Typical CRUD read that reads one item by primary key |
| list read | `GET /api/books?limit=50` | Light list acquisition |
| large list read | `GET /api/books?limit=500` | Materialization and response generation when the number of retrieved items is large |
| search read | `GET /api/books/search?q=Book&limit=20` | conditional read |
| aggregate-like read | `GET /api/books/bench/aggregate-report?limit=50` | aggregate-like read path |
| write | `POST /api/checkouts` | Normal inventory subtraction + checkout creation |
| write contention | `POST /api/checkouts` (`bookId=1` fixed) | Conflict when updating the same row many times |
| distributed write | `POST /api/checkouts` (random bookId) | Write with contention distributed to some extent |

In particular, the `checkout` scenarios need to be interpreted carefully. Internally, they execute a write transaction that goes through "read the book -> check inventory -> decrement inventory -> insert checkout." Unlike a simple JSON response, those paths are heavily affected by connection pools, transactions, and lock contention.

The measurement conditions are as follows.

- Start PostgreSQL 17 in a container every time and initialize it for each run
- All run the same HTTP scenario from the same benchmark runner
- Warm up 1 second, measurement 3 seconds
- There are two patterns for the number of concurrent executions: 128 and 256.
- Spring MVC has a maximum of 32 HikariCP, Spring WebFlux and Ktor R2DBC have a maximum R2DBC pool of 32, and Ktor JDBC has a maximum of 32 HikariCP

This time, I wanted to compare the entire stack, so I measured not only the framework, but also the serializer, DI, ORM, and drivers. In other words, this is not a comparison of "pure Ktor vs Spring", but rather a comparison of "the overall difference when this combination is actually assembled".

## Measurement results

For detailed numbers, it's best to look at the HTML summary introduced at the end. These were the main trends:

### 1. WebFlux is strong on small JSON, but Spring MVC is stronger on large JSON

Spring WebFlux R2DBC had the highest performance on `smallJson`, at about 29,035 req/s with concurrency 256. There is no DB access in this path, so the lightness of the framework and serialization path shows up directly, and the Netty-based stack seems to benefit from that.

However, when it comes to `largeJson`, the story changes. Looking at concurrency 256, Spring MVC virtual was about 3,319 req/s, Spring MVC platform about 3,015 req/s, Ktor JDBC about 2,545 req/s, and Ktor R2DBC about 1,093 req/s. As has been said before, large JSON with `kotlinx.serialization` still seems to be at a disadvantage compared to Jackson. It does quite well on small payloads, but the gap becomes clearer as the payload grows.

### 2. Spring MVC was stronger than expected on DB reads

Looking at the main read scenarios, the two Spring MVC configurations ranked near the top for most of `bookById`, `bookList`, `bookList500`, and `bookSearch`. At concurrency 256, the results looked like this:

| Scenario | 1st place | 2nd place | Ktor JDBC | Ktor R2DBC |
|---|---|---|---|---|
| `bookById` | Spring MVC virtual 12,868 | Spring MVC platform 10,052 | 7,730 | 6,173 |
| `bookList` | Spring MVC virtual 8,182 | Spring MVC platform 6,808 | 5,473 | 3,537 |
| `bookList500` | Spring MVC virtual 2,340 | Spring MVC platform 1,875 | 1,410 | 728 |
| `bookSearch` | Spring MVC virtual 8,281 | Spring MVC platform 7,237 | 5,654 | 4,763 |

At first, I thought that Ktor R2DBC, which is based on Coroutine and includes R2DBC, would be very strong at least in reading. However, the result was rather the opposite; the fairly orthodox Spring MVC + JDBC was a very strong reference point.

Virtual threads were also interesting. They were consistently ahead on the DB-read scenarios, but I still would not call them "always faster than platform threads." `aggregateReport` stayed roughly neck-and-neck, and WebFlux still won the very light non-DB paths. Virtual threads are clearly useful, but they are not magic that makes everything faster the moment you enable them.

### 3. Ktor JDBC was respectable, but Ktor R2DBC struggled

Looking only at the Ktor side, the gap between the JDBC and R2DBC versions was still quite clear. At concurrency 256, for example, `bookList500` was 1,410 req/s for Ktor JDBC and 728 req/s for Ktor R2DBC, while `largeJson` was 2,545 req/s for Ktor JDBC and 1,093 req/s for Ktor R2DBC. Even in `checkoutCreate`, Ktor JDBC reached 3,302 req/s and Ktor R2DBC 3,416 req/s, which is basically a draw rather than evidence that switching to R2DBC improves performance overall.

Looking at this difference, it appears that the bottleneck in Ktor is not simply "Ktor itself", but rather lies in the asynchronous DB access path including Exposed R2DBC. While Spring WebFlux is also very strong in non-DB light paths, there were not many situations where it clearly outperformed Spring MVC with JDBC in DB reads. I think it's natural to think that, at least for CRUD-centric workloads like this one, Reactive Streams-based DB access has not led to the straightforward advantage that was expected.

### 4. For hotspot scenarios, req/s alone is not very meaningful

`checkoutHotspot` is a little special, as they are constantly reducing the inventory of the same book, so there are a lot of failures midway through. Therefore, you need to carefully look at how much "effective success" you are producing rather than pure req/s.

Looking at the numbers, Ktor JDBC is still highest at concurrency 256 with about 3,374 req/s, but it also records 10,182 failures. Spring WebFlux is at 2,755 req/s with 8,321 failures, and Ktor R2DBC is at 1,723 req/s with 5,226 failures. In other words, "fast" here still includes a lot of "failing fast", so this scenario should be treated differently from the normal ones.

### 5. Ktor JDBC stood out on `aggregateReport`

In `aggregateReport`, Ktor JDBC had the highest result at 1,443 req/s at concurrency 128 and 1,453 req/s at concurrency 256. However, regarding this scenario, there is a note that the Ktor side is not yet on the exact same DB `GROUP BY` path as the Spring side, so it still seems better to view this as a directional result only.

## What I Took Away from It

The most impressive thing about this result was that the expectation that ``since it is non-blocking, it should produce high RPS'' did not turn out to be true. Of course, non-blocking is important and can be very useful for certain workloads. However, in relatively orthodox CRUD-centered benchmarks like this one, traditional configurations like Spring MVC + JDBC are still quite strong.

Also, while `kotlinx.serialization` performed quite well with small JSON, it seemed to be easily different from Jackson with large JSON. This is a trend that has been noted for some time, but this difference was confirmed to some extent in this measurement.

Regarding R2DBC, at least under these conditions, it is difficult to say that it is "fast enough to clearly replace JDBC." Although it is a beautiful concept, my impression is that in real applications with DB access, you also need to evaluate the extra overhead and locking behavior that come with it.

## Finally

Speaking of personal preference, Ktor, `kotlinx.serialization`, and Kotlin native configurations like Exposed are quite attractive. The code style is consistent, and it feels good to write. However, now that the cost of writing the code itself is lower than before with the help of AI, I think it would be better to focus more on "how fast it will run in the end" than just "how much code can be written with less code" than before.

In that sense, orthodox configurations like Spring MVC are still quite effective if performance is important. On the other hand, if you prioritize code volume, readability, and a Kotlin-like feel, I think Ktor is still a good choice. In fact, Ktor JDBC remained reasonably competitive on reads and some write paths, so it can still be a practical option depending on the kind of application you are building.

For detailed measurement results, I think it would be best to have a look at [README](https://github.com/retheviper/Benchmark/blob/main/README.md) in the Benchmark repository and [five-stack-summary.html](https://github.com/retheviper/Benchmark/blob/main/docs/reports/five-stack-summary.html) published there. The overview, purpose, stacks to be compared, and measurement methods are summarized in the README, so please refer to it if you want to follow individual numbers or trends for each scenario.
