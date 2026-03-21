---
title: "Should I Use Functional Endpoints with WebFlux?"
date: 2021-05-30
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - webflux
  - kotlin
  - rest api
---

Previously, I wrote [a post about Spring WebFlux](../spring-webflux/), where I briefly touched on the MVC pattern (`Controller`/`Service`) and `Functional Endpoint` (`Router`/`Handler`). To state the conclusion first, the introduction of Functional Endpoint can be said to be an attempt to improve the MVC pattern, which is a good pattern that has been used for a long time, but has problems such as performance and unsuitability for functional programming.

Now, just from that explanation, I feel like I should use Functional Endpoint as much as possible when using Spring WebFlux. But what actually happens? For example, if you use Controller/Service like traditional Spring MVC, is the performance really lower than when you use Router/Handler? Also, are there any "problems that did not exist in the MVC pattern" that should be considered when using Functional Endpoint? You can think of something like this.

So, this time I will use these two patterns to summarize some of the things you should consider when implementing a server-side application using Spring WebFlux.

## As a Programming Paradigm

Spring's MVC pattern is close to a paradigm based on traditional ideas such as metaprogramming using annotations and object orientation. Of course, there are features unique to the Spring Framework such as AOP and DI, but compared to the WebFlux code written in `Mono`/`Flux`, which is a realization of [Reactive Streams](https://www.reactive-streams.org), I feel that it is still closer to the traditional writing style.

I won't argue here about which is better between object-oriented and functional. Also, Java was originally designed as an object-oriented language, but since version 1.8 it has become possible to perform functional programming to some extent with the introduction of Functional Interface, and this is not much different with Kotlin. Therefore, choosing between Spring MVC and Spring WebFlux does not mean that you should write your code object-oriented or functionally.

However, since Spring WebFlux allows you to choose between the MVC pattern and Functional Endpoint, it is true that you can write in a way that suits either paradigm. I think the most important thing to decide on here is the agreement between the people writing the code. This is because programming paradigms have been developed primarily for the sake of programming efficiency.

Therefore, the criteria for judgment here should be pragmatic, such as "how easy it is to read" and "how quickly it can produce results." For example, if you want to improve the concurrency performance of an application that is already functioning as a service, and you think it would be better to write it as an MVC pattern because it will allow you to start up the service quickly, that is probably a good enough reason. Or, if there are many programmers who are already familiar with Functional Endpoint, you can actively introduce it. In other words, from my point of view, programming paradigms should be selected based on efficiency from the standpoint of practitioners.

Now, let's take a look at the code to see how the Controller/Service pattern and the Router/Handler pattern are actually different.

## When writing with MVC pattern

As the name suggests, Spring WebFlux's MVC pattern, or Controller/Service pattern, can be written in the same way as the existing Spring MVC. So, for example, if you write code like the following, you won't be able to tell much of the difference from Spring MVC.

```kotlin
@RestController
class HelloController(private val service: HelloService) {

    @GetMapping("/hello")
    fun hello(): ResponseEntity = 
        ResponseEntity
            .ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(service.getHello())
}

@Service
class HelloService {

    fun getHello(): String = "Hello Spring WebFlux"
}
```

However, with Spring WebFlux, in order to achieve complete non-blocking including DB connections, it is necessary to use a non-blocking compatible API such as [R2DBC]. This means that we need to use `Reactive Stream`, which necessarily means that we need to use its implementation, Mono/Flux.

So, for the time being, we have no choice but to obtain Mono/Flux from the Repository and write the code according to the Reactive Stream-specific writing style. The problem is that although the name gives the impression that Reactive Stream can be handled like a normal Java Stream, the actual processing is not so easy. For example, if you use an existing blocking-based API such as JPA or MyBatis, you would write code like the following in the Service method.

```kotlin
// Get member data and mail history by member ID
fun getMemberWithMailRecord(memberId: Int): MemberWithMailRecord {
    // Fetch member data
    val member = memberRepository.getMember(id = memberId) ?: throw RuntimeException("Can't find member")
    // Fetch the member's mail history
    val mailRecord = mailRecordRepository.getMailRecord(memberId = memberId) ?: throw RuntimeException("Can't find mailRecord")
    // Return both together
    return MemberWithMailRecord(
            member = member,
            mailRecord = mailRecord  
        )
}
```

However, if you use an API that returns Mono/Flux, the code will be as follows.

```kotlin
fun getMemberWithMailRecord(memberId: Int): Mono<MemberWithMailRecord> =
    memberRepository.getMember(id = memberId)
        .switchIfEmpty(Mono.error(RuntimeException("Can't find member")))
        .zipWith(mailRecordRepository.getMailRecord(memberId = memberId).switchIfEmpty(Mono.error(RuntimeException("Can't find mailRecord")))
        .map { MemberWithMailRecord(
              member = it.t1,
              mailRecord = it.t2
          )
        }
```

Even though we're doing the same thing, I'm worried about whether the code is easy to write and read. In addition, in situations where more complex processing is required, such as when you want to convert the data obtained with Flux to a List, or when you want to obtain more Mono from the obtained Mono data, the writing becomes increasingly complex.

Fortunately, Kotlin has [Coroutines](https://kotlinlang.org/docs/coroutines-overview.html), which makes it easier to write complex statements like this. After applying Corutines, the above code will become as follows even when using Mono/Flux.

```kotlin
suspend fun getMemberWithMailRecord(memberId: Int): MemberWithMailRecord {
    val member = memberRepository.getMember(id = memberId).awaitFirstOrNull() ?: throw RuntimeException("Can't find member")
    val mailRecord = mailRecordRepository.getMailRecord(memberId = memberId).awaitFirstOrNull() ?: throw RuntimeException("Can't find mailRecord")
    return MemberWithMailRecord(
            member = member,
            mailRecord = mailRecord  
        )
}
```

When using Coroutines, it is generally necessary to specify the scope, but in reality it seems to be a good idea to define up to the Controller methods as `suspend`. However, when migrating an existing project from Spring MVC to WebFlux, if you change many processes to suspend methods, it may become difficult to fix the unit tests, so be careful.

## When writing with Functional Endpoint

Next is the code when using Functional Endpont. Annotations have been cited as a problem with the MVC pattern, but you can also use annotations with Router/Handler, and depending on the application architecture, I think it's common to have to separate classes and manage them with annotations. Therefore, in such cases, the Router may be registered as `@Bean`, and the Handler may also be defined as `@Component`. In that case, the code would be as follows.

```kotlin
@Configuration
class HelloRouter(private val handler: HelloHanlder) {

    @Bean
    fun hello(): router {
        GET("/hello", handler::getHello)
    }
}

@Component
class HelloHandler {

    fun getHello(request: ServerRequest): Mono<ServerResponse> =
        ServerResponse
            .ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(Mono.Just("Hello Spring WebFlux"))
}
```

A feature of using Functional Endpoint is that the Router only serves to connect the endpoint and Handler, so you will need to write the process for receiving [ServerRequest](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerRequest.html) and returning [ServerResponse](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerResponse.html) in Handler. In the MVC pattern, you can freely specify ResponseEntity or a self-defined class as the return value of RestController, and considering that there are many patterns in Service that are only responsible for business logic, this is quite unique.

Because ServerRequest and ServerResponse are used in this way, the business logic part of Handler is one layer deeper than Service. You write logic in the body of ServerResponse and return it. For example, the code would be as follows.

```kotlin
fun getMember(request: ServerRequest): Mono<ServerResponse> =
    ServerResponse
        .ok()
        .contentType(MediaType.APPLICATION_JSON)
        .body(memberRepository.getMember(id = request.PathVariable("id"))
            .switchIfEmpty(Mono.error(RuntimeException("Can't find member")))
            .zipWith(mailRecordRepository.getMailRecord(memberId = request.PathVariable("id")).switchIfEmpty(Mono.error(RuntimeException("Can't find mailRecord")))
            .map { MemberWithMailRecord(
                member = it.t1,
                mailRecord = it.t2
                )
            }
        )

```

Of course, you can still use Coroutines in this case. If you use Corutines, you can write it like this:

```kotlin
suspend fun getMember(request: ServerRequest): Mono<ServerResponse> {
    val member = memberRepository.getMember(id = memberId).awaitFirstOrNull() ?: throw RuntimeException("Can't find member")
    val mailRecord = mailRecordRepository.getMailRecord(memberId = memberId).awaitFirstOrNull() ?: throw RuntimeException("Can't find mailRecord")
    return ServerResponse
                    .ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .bodyValueAndAwait(
                        Mono.just(
                            MemberWithMailRecord(
                                member = it.t1,
                                mailRecord = it.t2
                            )
                        )
                    }
}
```

## Performance

One of the problems with the MVC pattern is annotations. Using annotations inevitably means using reflection, which naturally leads to a drop in performance. Looking at this, it seems that using Functional Endpoint is more advantageous in terms of performance than the MVC pattern in WebFlux. But what about the reality?

[Spring's official documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-programming-models) describes Functional Endpoints as `lightweight`, but it does not say much more about performance. Most comparisons focus on Spring MVC versus Spring WebFlux, and I could not find many examples comparing the MVC pattern and Functional Endpoints within WebFlux itself. So I ran a simple benchmark to check the difference between the two patterns.

As a benchmark tool, we could have used Jmeter, but it also allows measurements with short commands, so this time we used the [Apache HTTP server benchmarking tool](https://httpd.apache.org/docs/2.4/en/programs/ab.html) to conduct the test.

## Code Used

What we want to know when measuring performance is whether performance changes depending on the implementation pattern, so we purposely excluded DB connections. The sample application created for comparison consists of a common logic that simply generates data, and a set of Controller/Service and Router/Hanlder that simply return it.

### common

The logic itself for generating data is the same, and we simply generate a List using a loop.

```kotlin
// Utility for generating and returning fixed data
object DataCreationUtil {

    // From January 1, 1970 to December 31, 2021
    var data: List<Person> = (0..18992)
        .map {
            Person(
                id = it.toLong(),
                name = "Person${it}",
                birth = LocalDate.ofEpochDay(it.toLong()),
                joined = LocalDate.ofEpochDay(it.toLong() + 10000)
            )
        }.toList()
}

// Generated data
data class Person(
    val id: Long,
    val name: String,
    val birth: LocalDate,
    val joined: LocalDate
)
```

### Controller/Service Implementation

I think you can understand the MVC pattern just by reading the code, so I will omit the explanation.

```kotlin
@RestController
class PerformanceTestController(
    private val service: PerformanceTestService
) {

    @GetMapping("/performance-controller")
    fun getData(): ResponseEntity<List<Person>> =
        ResponseEntity
            .ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(service.getData())
}

@Service
class PerformanceTestService {

    fun getData(): List<Person> = service.getData()
}
```

### Router/Handler Implementation

The difference in Functional Endpoint compared to the MVC pattern is that everything that can be called processing is written in the Handler.

```kotlin
@Configuration
class PerformanceTestRouter(private val handler: PerformanceTestHandler) {

    @Bean
    fun route() = router {
        GET("/performance-router", handler::getData)
    }
}

@Component
class PerformanceTestHandler {

    fun getData(request: ServerRequest): Mono<ServerResponse> =
        ServerResponse
            .ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(Flux.fromIterable(DataCreationUtil.data))
}
```

## test results

The test was conducted under the following conditions.

- Number of users set to 5000, requests per user set to 50
- Separate tests for each pattern, taking into account warm-up time
  - Test after restarting the server
  - Test loops 10 times

The actual script I used is as follows. We run this after the server starts, and after 10 loops, we restart and test the Functional Endpoint.

```bash
#!/bin/bash

for i in {1..10}
do
 ab -n 5000 -c 50 http://localhost:8080/performance-controller
done
```

However, even after doing this, there were still deviations in the test results for each lap, so here we will introduce the results that apply to the intermediate values. The results are as follows, but to state the conclusion first, I think the difference in performance between the MVC pattern and Functional Endpoint can be described as "within the margin of error."

### Controller/Service results

```bash
Server Software:        
Server Hostname:        localhost
Server Port:            8080

Document Path:          /performance-controller
Document Length:        1440242 bytes

Concurrency Level:      50
Time taken for tests:   24.989 seconds
Complete requests:      5000
Failed requests:        0
Total transferred:      7201590000 bytes
HTML transferred:       7201210000 bytes
Requests per second:    200.09 [#/sec](mean)
Time per request:       249.892 [ms](mean)
Time per request:       4.998 [ms](mean, across all concurrent requests)
Transfer rate:          281433.26 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   1.4      0      11
Processing:    29  248  79.8    242     516
Waiting:       24  192  60.8    185     442
Total:         29  249  79.4    243     516

Percentage of the requests served within a certain time (ms)
  50%    243
  66%    275
  75%    295
  80%    309
  90%    354
  95%    394
  98%    430
  99%    449
 100%    516 (longest request)
```

### Router/Handler results

```bash
Server Software:        
Server Hostname:        localhost
Server Port:            8080

Document Path:          /performance-router
Document Length:        1440257 bytes

Concurrency Level:      50
Time taken for tests:   25.541 seconds
Complete requests:      5000
Failed requests:        0
Total transferred:      7201775000 bytes
HTML transferred:       7201285000 bytes
Requests per second:    195.76 [#/sec](mean)
Time per request:       255.410 [ms](mean)
Time per request:       5.108 [ms](mean, across all concurrent requests)
Transfer rate:          275360.22 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   3.2      0     151
Processing:    33  253  80.4    246     612
Waiting:       28  194  59.8    184     475
Total:         39  254  80.0    247     613

Percentage of the requests served within a certain time (ms)
  50%    247
  66%    286
  75%    302
  80%    312
  90%    361
  95%    398
  98%    441
  99%    459
 100%    613 (longest request)
```

## documentation

Next, from a documentation perspective, I feel that it is still too early to introduce Functional Endpoint. Documentation here does not refer to comments like JavaDoc or Kdoc, but refers to [Swagger](https://swagger.io), which is often used these days.

Recently, it is easy to document APIs using [SpringFox](http://springfox.github.io/springfox) etc., but it is not so easy with Functional Endpoint. Some of you may already know the reason, but the reason is that the endpoint and Handler processing are simply passed as arguments to the Router, and the Handler argument is fixed to ServerRequest, and the return value is fixed to ServerResponse.

Of course, even when using ServerRequest and ServerResponse, you can treat it as the data you want. For example, how to take parameters from a request is as follows.

```kotlin
// Get a member by the ID passed in the path variable
suspend fun getMember(request: ServerRequest): ServerResponse {
    // Read the path variable
    val id = request.pathVariable("id")
    // ...
}

// Create a new member from the data passed in the request body
suspend fun createMember(request: ServerRequest): ServerResponse {
    // Map the request body to a class
    val form = request.bodyToMono(MemberCreateForm::class.java).awaitFirst()
    // ...
} 
```

However, useful APIs like SpringFox that automatically document the API probably use reflection. Therefore, it may be difficult to judge what kind of processing is actually being performed by Handler.

Fortunately, Spring developers seem to be aware of this issue, and using [springdoc-openapi](https://springdoc.org) allows API documentation even with Functional Endpoint. However, even in this case, there is still a problem. This is because it does not automate API documentation, it just "provides a means for documentation". Therefore, you need to specify annotations for each Router and Handler as shown below.

```kotlin
@Bean
@RouterOperations(
    value =
        [
            RouterOperation(
                path = "/api/v1/members",
                beanClass = MemberHandler::class,
                beanMethod = "listMember",
                method = [RequestMethod.GET],
                produces = [MediaType.APPLICATION_JSON_VALUE]
            ),
            RouterOperation(
                path = "/api/v1/members/{id}",
                beanClass = MemberHandler::class,
                beanMethod = "getMember",
                method = [RequestMethod.GET],
                produces = [MediaType.APPLICATION_JSON_VALUE]
            ),
            RouterOperation(
                path = "/api/v1/members",
                beanClass = MemberHandler::class,
                beanMethod = "createMember",
                method = [RequestMethod.POST],
                produces = [MediaType.APPLICATION_JSON_VALUE]
            ),
            RouterOperation(
                path = "/api/v1/members/{id}",
                beanClass = MemberHandler::class,
                beanMethod = "updateMember",
                method = [RequestMethod.PUT],
                produces = [MediaType.APPLICATION_JSON_VALUE]
            ),
            RouterOperation(
                path = "/api/v1/members/{id}",
                beanClass = MemberHandler::class,
                beanMethod = "deleteMember",
                method = [RequestMethod.DELETE],
                produces = [MediaType.APPLICATION_JSON_VALUE]
            ),
        ]
    )
fun routeMember() = coRouter {
    GET("/api/v1/members") { handler.listMember() }
    GET("/api/v1/members/{id}", handler::getMember)
    POST("/api/v1/members", handler::createMember)
    PUT("/api/v1/members/{id}", handler::updateMember)
    DELETE("/api/v1/members/{id}", handler::deleteMember)
}
```

As you can see, the annotations for documentation are longer than the actual code. It's good that we now have a way to use Swagger in Functional Endpoint, but it's still inconvenient compared to the MVC pattern. So if documentation is important to you, you probably shouldn't use Functional Point yet.

## lastly

This time, I compared Spring WebFlux's MVC pattern and Functional Endpoint from the viewpoints of code writing, performance, and documentation. Spring WebFlux was also announced in 2017, so this year is already the fifth year, but I feel that there are still many things that need to be complemented compared to the MVC pattern. In my case, I feel that it would be better to choose Functional Endpoint in order to take advantage of WebFlux's main concepts of non-blocking and functional programming, but I feel that there is no need to do so just yet. This is especially true when it comes to writing enterprise applications. Of course, you need to think about whether you should introduce WebFlux itself in the first place.

Still, I think there are many possibilities for the implementation method called Functional Endpoint. Even if it is not Spring WebFlux, it is widely used in modern web frameworks. For example, [jooby](https://jooby.io), which had the best performance among Java frameworks in TechEmpower's [Benchmark](https://www.techempower.com/benchmarks), supports the MVC pattern and the Script Route pattern, which is very similar to Functional Endpoint, and [Ktor](https://ktor.io), a web framework for Kotlin developed by JetBrains, does not have the MVC pattern, but also supports Functional Endpoint. It only supports Routing, which is very similar to Endpoint. Therefore, other frameworks such as [Express](https://expressjs.com/ja) and [Gin](https://github.com/gin-gonic/gin) provide similar API implementation methods, so if you have the time, it might be a good learning experience to try them out and get used to them. Also, functional programming is likely to be widely used in the future, so it might be a good idea to have it and practice with it.

See you soon!
