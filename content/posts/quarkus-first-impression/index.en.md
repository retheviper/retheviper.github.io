---
title: "I tried Quarkus"
date: 2021-10-24
translationKey: "posts/quarkus-first-impression"
categories: 
  - quarkus
image: "../../images/quarkus.webp"
tags:
  - java
  - kotlin
  - ktor
---

Although Spring MVC is a good framework, there are criticisms that it is not well suited to the recently popular [microservice](https://en.wikipedia.org/wiki/Microservices) style. Reasons cited include slow app startup, large size, and high memory usage. If your app starts slowly, you cannot expect changes to take effect quickly. If your app is large and uses a lot of memory, the more instances you run, the higher the cost will be. This is not only a microservices issue either. It is the same reason we deliberately use interpreted languages like JavaScript and Python in serverless apps.

So how can these problems be avoided? Considering the problem in the first place, it shouldn't be a problem that is entirely limited to Spring. It is true that Spring's startup time is not fast compared to other frameworks, and there are problems with the amount of memory used, but as long as you are using a language based on the JVM, this seems to be an unavoidable problem to some extent. This is because when using the JVM language, it is necessary to consider the proportion that the JVM occupies in the startup time, size, and memory usage of the application.

However, this does not mean that there are no ways to solve these problems. This time we will introduce [Quarkus](https://quarkus.io), which has been developed as an answer to this problem.

## What is Quarkus?

Quarkus is a web framework for Java created by Red Hat, famous for [RHEL](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux). I think the explanation on the official website is more accurate than anything else, so please check the text below.

> A Kubernetes Native Java stack tailored for OpenJDK HotSpot and GraalVM, crafted from the best of breed Java libraries and standards.

The true nature of this framework is that you can create Java applications as `Kubernetes Native`. Although it is explained as Java, of course other JVM languages ​​such as Kotlin can also be used, so you can consider installing it even if you are using such a language.

I am curious about the phrase `Kubernetes Native` here, but I do not think it simply means that the framework specializes in building containers. Spring Boot 2.3 already introduced [Docker image creation](https://spring.io/blog/2020/08/14/creating-efficient-docker-images-with-spring-boot-2-3), and you can containerize Java applications with libraries like [Jib](https://github.com/GoogleContainerTools/jib). There are plenty of other ways to create containers too. So it seems natural to think that `Kubernetes Native` here means it is designed specifically for Kubernetes.

So what exactly makes it called `Kubernetes Native`? From an infrastructure perspective, `Kubernetes Native` seems to refer to an architecture that is complete with Kubernetes alone. The fact that it is completed with Kubernetes means that it is optimized accordingly. From an application perspective it shouldn't be much different. First of all, Quarkus is introduced as having the following features.

- Native compilation is possible
- Fast startup speed
- Low memory usage

Being able to compile natively means that there is no need to use the JVM, which eliminates all three problems listed above. If so, it would be quite advantageous not only for microservices and serverless deployments, but also for container-based deployment. It is said that even when using JVM, it has an advantage in startup speed and memory usage compared to other frameworks, so if this is true, there will be sufficient benefits even if you do not compile natively.

## When you actually touch it

All of the features listed are attractive, but there are some things you won't understand until you actually use the framework. So, I made a small sample, tried it out, and would like to tell you a little about my impressions.

## startup speed

### For Spring Boot

The startup speed of Spring varies greatly depending on the class being DIed, so here I tried starting an application from [Spring initializr](https://start.spring.io) with only the following items set.

- Project: Gradle
- Language: Kotlin
- Spring Boot: 2.5.5
- Packaging: War
- Java: 11
- Dependencies: None

And locally, I'm starting it using Oracle JDK 17. Maybe it's just my imagination, but I feel like it starts up faster than when I was using Java 11. In particular, when I started the app with the above settings, I got the following result. (Local machine information is deleted)

```bash
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.5.5)

2021-10-17 19:10:21.472  INFO 48844 --- [           main] com.example.demo.DemoApplicationKt       : Starting DemoApplicationKt using Java 17 on Local.
2021-10-17 19:10:21.475  INFO 48844 --- [           main] com.example.demo.DemoApplicationKt       : No active profile set, falling back to default profiles: default
2021-10-17 19:10:22.058  INFO 48844 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2021-10-17 19:10:22.067  INFO 48844 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2021-10-17 19:10:22.068  INFO 48844 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.53]
2021-10-17 19:10:22.117  INFO 48844 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2021-10-17 19:10:22.118  INFO 48844 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 607 ms
2021-10-17 19:10:22.335  INFO 48844 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2021-10-17 19:10:22.342  INFO 48844 --- [           main] com.example.demo.DemoApplicationKt       : Started DemoApplicationKt in 1.129 seconds (JVM running for 1.396)
```

It takes 1.396 seconds to start the JVM and 1.129 seconds to start the application. I think this is probably the fastest startup time on my machine since there are no dependencies. If this were to become an actual business app, it could take more than 10 seconds just to start the app. 10 seconds is not much of a problem for a single startup, but when testing locally, the app starts for each test.

### For Quarkus Native

Now, let's take a look at the case of Quarkus. First, I tried building it using [GraalVM](https://www.graalvm.org/) since it says it can be compiled natively. The actual build can be run as a Gradle task (although it requires specific parameters) and is simple. And here are the results of running it:

```bash
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2021-10-23 19:24:18,395 INFO  [io.quarkus](main) quarkus-sample 1.0.0-SNAPSHOT native (powered by Quarkus 2.3.0.Final) started in 0.018s. Listening on: http://0.0.0.0:8080
2021-10-23 19:24:18,397 INFO  [io.quarkus](main) Profile prod activated. 
2021-10-23 19:24:18,397 INFO  [io.quarkus](main) Installed features: [cdi, config-yaml, kotlin, resteasy-reactive, resteasy-reactive-jackson, smallrye-context-propagation, vertx]
```

It took 0.018 seconds. Although the structure of the built project is simple, the startup speed is certainly fast. I think this can certainly be used not only for microservices, but also for serverless applications with many requests.

### For Quarkus JVM

As with Spring, I tried starting it using Oracle JDK 17. Quarkus has a development mode that allows you to make modifications while the server is running, but here I intentionally created a Jar and started it. As a side note, it is interesting that in Spring, if you want to include all dependencies, it is called a war, but in Quarkus it is called an uber-jar.

```bash
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2021-10-23 19:20:59,897 INFO  [io.quarkus](main) quarkus-sample 1.0.0-SNAPSHOT on JVM (powered by Quarkus 2.3.0.Final) started in 0.761s. Listening on: http://0.0.0.0:8080
2021-10-23 19:20:59,905 INFO  [io.quarkus](main) Profile prod activated. 
2021-10-23 19:20:59,906 INFO  [io.quarkus](main) Installed features: [cdi, config-yaml, kotlin, resteasy-reactive, resteasy-reactive-jackson, smallrye-context-propagation, vertx]
```

This time it took 0.761 seconds. It is certainly several dozen times slower than Native, but it is still faster than Spring.

I think that if the app starts up faster in this way, unit tests will be faster even when developing locally, so you can check them immediately. In particular, when writing test cases using [RestTemplate] or [WebTestClient](https://spring.pleiades.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/reactive/server/WebTestClient.html) in Spring, you have to start the app for each test, so the more test cases you have, the more time it takes. Therefore, if you write a similar test in Quarkus, it will probably save you a lot of time if it starts up quickly.

## Easy migration from Spring

One of the good things about Quarkus, which I didn't really pay attention to at first, is that it's easy to migrate from Spring. When developing a new app or changing the framework of an existing app, there are many things and viewpoints to consider when selecting technology, but I think some of them include ``How can we reduce man-hours?'' and ``Is it easy to recruit engineers?'' From this perspective, it would be difficult for both the company and the engineers if the technology is completely new to current engineers, or if it is a technology that is not used much in the industry. I think these problems make it difficult for companies to introduce new technology.

Therefore, although it is a new technology, it is similar to something commonly used in the industry, which can be said to be a considerable advantage because it can reduce the learning cost for engineers. Now, let's look at the actual code and see what would happen if you migrated Spring code to Quarkus.

### For Spring

First, let's say we have an API that retrieves data for a record called Person by passing an ID in a query parameter. With Spring, I think the code would be something like the following.

```kotlin
@RestController
@RequestMapping("/api/v1/person")
class PersonController(private val service: PersonService) {
    @GetMapping
    fun getPerson(id: Int): PersonResponse {
        return service.getPerson(id).let { PresonResponse.from(it) }
    }
}

@Service
class PersonService(private val repository: PersonRepository) {
    fun getPerson(id: Int): PersonDto {
        return repository.findById(id).let { PersonDto.from(it) }
    }
}
```

### For Ktor

Before looking at the Quarkus code, I would like to first see what happens when the same code is written in [Ktor](https://ktor.io/), which I have introduced in this blog. This will allow you to compare Spring and choose a completely different framework.

Ktor is also a good framework, but the design philosophy of the framework itself is different from Spring, so there are many things to consider when migrating an existing application. For example, it basically doesn't support DI, so you need to install a separate library.

The following is an example of Ktor that implements the same API as Spring above by introducing [Koin](https://insert-koin.io/) as a DI library. You can see that they have quite different structures.

```kotlin
fun main() {
    // Configure DI
    val personModule = module {
        single { PersonService(get()) }
        single { PersonRepository() }
    }

    // Install Koin into the app
    install(Koin) {
        modules(personModule)
    }

    // Configure routing as a module
    configureRouting()
}

// Register the controller in the router
fun Application.configureRouting() {
    routing {
        personController()
    }
}

fun Route.personController() {
    // Inject the service
    val service: PersonService by inject()
    get("/api/v1/person") {
        return service.getPerson(id).let { PresonResponse.from(it) }
    }
}

// Service
class PersonService(private val repository: PersonRepository) {
    fun getPerson(id: Int): PersonDto {
        return repository.findById(id).let { PersonDto.from(it) }
    }
}
```

### For Quarkus

Next, let's take a look at the code when creating an API with Quarkus. There are two ways to create an API in Quarkus: [RESTEasy](https://resteasy.github.io) and [Reactive Routes](https://quarkus.io/guides/reactive-routes), but there is no big difference in creating the app no ​​matter which one you use, so here I would like to introduce an implementation using RESTEasy. First, take a look at the code below.

```kotlin
@Path("/api/v1/person")
class PersonController(private val service: PersonService) {
    @GET
    fun getPerson(id: Int): PersonResponse {
        return service.getPerson(id).let { PresonResponse.from(it) }
    }
}

@ApplicationScoped
class PersonService(private val repository: PersonRepository) {
    fun getPerson(id: Int): PersonDto {
        return repository.findById(id).let { PersonDto.from(it) }
    }
}
```

Comparing the Spring code, you can see that they can be implemented in almost the same way, just using different types of annotations. Therefore, there is no need to rethink the architecture as in the case of Ktor, and migration becomes easy. Another advantage of using RESTEasy is that it is easy to create Reactive APIs. For Reactive, you will need to use [Mutiny](https://smallrye.io/smallrye-mutiny/), but you can think of the concept of Uni/Multi as having a 1:1 correspondence with Mono/Flux, so anyone who has used Spring WebFlux or other Reactive Streams will be able to adapt quickly.

```kotlin
@Path("/api/v1/person")
class PersonController(private val service: PersonService) {
    @GET
    fun getPerson(id: Int): Uni<PersonResponse> {
        return Uni.createFrom().item(service.getPerson(id).let { PresonResponse.from(it) })
    }
}
```

I think writing methods like Ktor and Spring WebFlux's Router Function have some good points, but many people are probably used to writing methods like Spring MVC, and there are no particular problems with writing them that way, so even if it's a new framework, I think being able to write code in the same way as an existing framework is one of the selling points that makes it easier to choose that framework. For example, JavaScript frameworks like [NestJS](https://nestjs.com/) allow you to write code similar to Spring MVC, but this is probably to appeal to engineers who have experience with Spring.

From this perspective, it seems to be a good idea since engineers who already have experience with Spring can migrate to Quarkus right away, and existing Spring projects can be easily migrated as well.

## concern

After actually using Quarkus, I found the above to be the best, but while building an app with Native, I felt that there were some concerns. For example:

## Native builds are slow

It is certainly a good thing that the startup speed is faster with Native, but the problem is that the build itself is slow. Of course, building as Native means compiling all your code as machine code from the beginning. JVM bytecode is the same in all environments, but machine code is not, so it's only natural that it takes time to generate code tailored to that platform. For example, when building a project used for local testing as a native image, it took the following time.

```bash
./gradlew build -Dquarkus.package.type=native

> Task :quarkusBuild
building quarkus jar

[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]    classlist:   2,311.58 ms,  1.19 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336](cap):   3,597.91 ms,  1.19 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]        setup:   5,450.32 ms,  1.19 GB
19:22:21,827 INFO  [org.jbo.threads] JBoss Threads version 3.4.2.Final
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336](clinit):     779.71 ms,  4.33 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336](typeflow):  14,308.32 ms,  4.33 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336](objects):  16,140.38 ms,  4.33 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336](features):   1,145.40 ms,  4.33 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]     analysis:  33,857.15 ms,  4.33 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]     universe:   1,718.32 ms,  5.14 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336](parse):   2,635.36 ms,  5.14 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336](inline):   7,363.76 ms,  5.99 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336](compile):  26,314.40 ms,  6.15 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]      compile:  40,954.87 ms,  6.15 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]        image:  10,493.47 ms,  6.15 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]        write:   2,111.59 ms,  6.15 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]      [total]:  97,207.01 ms,  6.15 GB

BUILD SUCCESSFUL in 1m 43s
```

I think this could become quite a bottleneck if you are building with CI or need to frequently modify and deploy your app. I don't think it's a problem if you have enough machine power or don't really care about the time it takes to deploy, but if startup speed is important to you, the longer it takes to build, the more you'll end up with an equal trade-off. In this case, I think it would be better to measure the time it takes to build as a Jar or the time it takes from build to startup using another framework before making a decision.

## peak performance

It is generally accepted as common sense that Java (JVM language) has inferior performance compared to languages ​​such as C and C++. However, this is not the case in all situations. In some cases, the reasons are unrelated to the language itself, such as appropriate algorithms and application design, but there are also cases when considering the characteristics of the language. This is because the characteristics of the JVM language compiler are different from languages ​​that generate native code such as C and C++.

It stands to reason that languages ​​that generate machine code from the beginning have superior performance compared to JVM languages ​​that require a virtual machine to convert bytecode to machine code again. In fact, this is reflected in the numbers, and it seems that when Java first appeared, it received a lot of criticism due to performance issues. Nowadays, Java is often used when creating server-side applications where performance is relatively important, but this is often said to be due to ``developments in hardware.''

However, this does not mean that JVM applications are always slower than native ones. That is because there is not only [AOT](https://en.wikipedia.org/wiki/Ahead-of-time_compilation), which compiles everything up front, but also [JIT](https://en.wikipedia.org/wiki/Just-in-time_compilation), which compiles only when needed.

It is said that JVM uses JIT to analyze and optimize bytecodes and generate machine code to produce code with better performance. Optimization here includes reducing overhead by including frequently used methods and constants. If we were to convert code with such optimizations into machine code, we would naturally expect better performance. However, IT does not work in all cases. Compilation requires time and machine power, so it is wasteful to convert code that will only be used once into machine code. Therefore, the code compiled by JIT is determined by the frequency of use of that code. When testing with [JMH](https://github.com/openjdk/jmh), which is often used in Java microbenchmarks, we warm up in advance to improve benchmark accuracy through JIT optimization.

### When I verified it

I was curious about how much the runtime performance actually differs depending on whether it is Native or JVM, so I created a Service that simply creates and returns 100,000 pieces of data in a loop and measured the processing time. Here, the time to response varied greatly each time the API was called, probably because [Multi](https://smallrye.io/smallrye-mutiny/getting-started/creating-multis) was used as the return value of the Controller, so please understand that what we are measuring is the "time taken to generate data by the for loop" rather than the time from request to response.

GraalVM CE 21.3.0 (OpenJDK 11) was used for native build and Jar execution, and the processing time was measured by outputting the values ​​obtained with Kotlin's [measureTimeMillis](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.system/measure-time-millis.html) to the log.

First, here are the results when starting with Native.

```bash
2021-10-23 17:12:14,061 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-6) measured time was 89 ms
2021-10-23 17:12:15,630 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-1) measured time was 52 ms
2021-10-23 17:12:17,079 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-15) measured time was 106 ms
2021-10-23 17:12:18,174 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-5) measured time was 49 ms
2021-10-23 17:12:19,523 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-11) measured time was 50 ms
2021-10-23 17:12:20,468 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-4) measured time was 50 ms
2021-10-23 17:12:21,739 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-7) measured time was 124 ms
2021-10-23 17:12:23,113 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-12) measured time was 53 ms
2021-10-23 17:12:24,073 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-13) measured time was 49 ms
2021-10-23 17:12:25,308 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-2) measured time was 53 ms
```

Also, the following is the result when started with JVM.

```bash
2021-10-23 17:10:32,240 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-8) measured time was 163 ms
2021-10-23 17:10:35,057 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-6) measured time was 33 ms
2021-10-23 17:10:39,418 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-11) measured time was 40 ms
2021-10-23 17:10:42,211 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-3) measured time was 25 ms
2021-10-23 17:10:44,149 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-10) measured time was 38 ms
2021-10-23 17:10:46,283 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-2) measured time was 24 ms
2021-10-23 17:10:48,262 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-20) measured time was 22 ms
2021-10-23 17:10:49,854 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-12) measured time was 26 ms
2021-10-23 17:10:51,552 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-23) measured time was 23 ms
2021-10-23 17:10:52,967 INFO  [com.ret.dom.ser.MemberService](vert.x-eventloop-thread-7) measured time was 51 ms
```

You can see that the JVM takes a long time on the first execution, probably due to the involvement of JIT, and then the processing speed becomes significantly faster. There is a possibility that performance can be further improved by upgrading GraalVM's compiler, but the same is true for JVM, so if peak runtime performance is absolutely important, you may want to consider using JVM.

## lastly

Actually, I think it is necessary to measure memory usage more accurately, but I have already covered that in [another article](https://medium.com/swlh/springboot-vs-quarkus-a-real-life-experiment-be70c021634e), so I will omit it here. In conclusion, Quarkus does use less memory including the heap, but we confirmed that Spring Boot is better in terms of maximum CPU usage and latency. However, that may simply be because Quarkus has a shorter history.

At first glance, it seems that it can be called Kubernetes native. If you build Native, the size of the app will be larger (about twice as much) compared to Jar, but the good thing is that you don't need JDK. The size of JDK is approximately 300MB based on AdoptOpenJDK. If there is only one instance, this is not likely to be a problem, but if the number of instances increases, the storage size required for JDK alone will increase by the multiplier, so I think you may want to use Native.

Another attractive feature is that you can combine various libraries and frameworks, and you can use Spring Security as is. Any engineer with experience with Spring will likely get used to it quickly, so from a company's perspective, I think it will be relatively easier to recruit engineers than when using other frameworks.

Spring WebFlux and Ktor were good, but we're now in an era where new strong players have appeared and it's difficult to decide which one to use. I'd really like to try [Rocket](https://rocket.rs/), but I'm not sure if I'll be able to do it within this year...

See you soon!
