---
title: "What is Spring WebFlux?"
date: 2020-09-06
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - webflux
  - nonblocking
---

Spring was first announced in 2002, so almost 20 years have passed. Nowadays, we use Spring as a matter of course when talking about Java, and with the advent of Spring Boot, the problems of complicated environment construction and initial settings such as Spring MVC, Maven, and properties have become much easier. This is especially true in my case, but I often write applications using Spring Boot, Gradle, and yaml that don't contain any XML, and I find everything easy.

Spring has been making great progress, but for several years now there have been calls to improve the problems with Spring MVC, and from Spring 5.0 a new framework that is completely different from MVC has been introduced. That is Spring WebFlux, which I will introduce this time.

## What is the difference between Spring WebFlux and MVC?

Since it is a framework that has been rebuilt from scratch, there are many fundamental differences, so theoretically the following keywords can be cited.

- Asynchronous, non-blocking, reactive

From my (Code Monkey) level, the differences from MVC that I experience at the actual code level are as follows.

- Mono and Flux
- Function instead of Controller/Service
- Netty from Tomcat
- R2DBC from JPA/JDBC
- new abstraction class

Now, let's take a look at these differences one by one.

## theoretical story

## Asynchronous, non-blocking, reactive

Spring WebFlux is built on [Project Reactor](https://github.com/reactor/reactor-core), and its core idea is [Reactive Streams](https://www.reactive-streams.org). Reactive Streams is a standard API introduced in Java 9 as `java.util.concurrent.Flow`.

Reactive Streams is similar to the Observer pattern. Simply put, when an event occurs, you request a notification and receive the data. This act of requesting notification is called a subscription, and it happens through the exchange between a publisher that emits data and a subscriber that subscribes to it. Creating programs around this kind of event-driven model is what we call reactive programming.

And with Reactive Streams, this subscription exchange is done asynchronously and in a non-blocking way. That means you do not have to wait for one task to finish before doing something else. As a result, there may not be much difference compared with synchronous blocking code when running the same number of tasks [^1], but when thread count is the bottleneck, asynchronous non-blocking processing becomes advantageous.

Theoretically speaking, the more you go into it, the more it becomes clear, so what is the difference in actual code? Let's take a look.

## At the code level

## Mono and Flux

With Spring WebFlux, Mono and Flux are used as return values ​​(responses) in controller methods. With Spirng MVC, you would specify a JSP file as a string, and with REST API, you would specify an object to be returned as JSON. Of course, Mono and Flux are also output as JSON objects, but the way they are created is a little unique.

I mentioned earlier that the core idea behind Spring WebFlux is Reactive Stream. Mono and Flux implement Reactive Stream on the WebFlux side. Roughly speaking, Mono represents `0 or 1`, while Flux represents `0 to N`, but that does not necessarily mean `Collection = Flux`; depending on the situation, it can still be treated as Mono.

From its name, Reactive Stream seems to have some connection to Java 1.8's Stream API. In fact, the point of data creation and consumption is different [^2], but the similarly named methods and Lambda completion are similar. If you are already familiar with [RxJava](https://github.com/ReactiveX/RxJava) or [JOOL](https://github.com/jOOQ/jOOL), you should be able to adapt to the writing style without any problems, but for those who are not, it may be difficult to adapt.

For example, let's say you want to implement a simple GET method that returns a response 1 second after receiving a request. A REST API using Spring MVC would look like this:

```java
@GetMapping
@ResponseStatus(HttpStatus.OK)
public String getTest() {
    try {
        Thread.sleep(1000);
        return "task completed by mvc";
    } catch (InterruptedException e) {
        return "task failed by mvc";
    }
}
```

In WebFlux, create and return Mono as follows:

```java
@GetMapping
@ResponseStatus(HttpStatus.OK)
public Mono<String> getTest() {
    return Mono.delay(Duration.ofMillis(1000)).then(Mono.just("task completed by Mono"));
}
```

Recently, there are an increasing number of languages ​​and frameworks that are all declarative (for example, Flutter and SwiftUI), so this type of writing is not uncommon, but it may be a bit difficult for those who are used to traditional imperative programming. I myself like Stream and Lambda, but I'm not sure which is better: imperative type with nested methods or declarative type with long method chains...

## Function instead of Controller/Service

Another feature of WebFlux's code is that it allows you to create alternative classes for Controller and Service. Of course, you can use the Controller and Service classes as before, but you might want to try something new.

Creating a Controller/Service with Spring means developing it by relying on "metaprogramming" using annotations. Annotations are certainly useful, and it feels like you can do anything with them in Spring. However, development using annotations has the following problems.

- Cannot be verified by compiler
- Does not prescribe code behavior
- There are no standards for inheritance and extension rules.
- Can produce unintelligible and easily misunderstood code
- extremely difficult to test
- Difficult to customize

The reason is that using annotations ultimately means relying on Reflection. When using Reflection, bytecode is generated at runtime, so there's not much you can do at compile time. Reflection is certainly a powerful tool, but it also has other problems. For example, performance suffers and debugging is difficult. Because of this problem, [GraalVM](https://www.graalvm.org), which natively compiles Java code, may not support Reflection.

Anyway, a new feature introduced in WebFlux to solve such problems is `Function`. Yes, as you say, it's a function. You can create `Router` that corresponds to an existing Controller and `Handler` that corresponds to a Service, and write code as a functional model (Functinal). Of course, even if you write it functionally, you can use annotations (in fact, you can't do it without annotations...). For example, it would be written as follows.

```java
@Configuration
public class Router {

    @Bean
    public RouterFunction<ServerResponse> route(final Handler handler) {
        return nest(
                path("/api/v1/web/members"),
                RouterFunctions.route(GET("/"), handler::listMember)
                        .andRoute(POST("/"), handler::createMember)
        );
    }
}

@Component
public class Handler {

    private final MemberRepository repository;

    @Autowired
    public Handler(final MemberRepository repository) {
        this.repository = repository;
    }

    public Mono<ServerResponse> listMember(final ServerRequest request) {
        return ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(Flux.fromIterable(repository.findAll()), Member.class);
    }

    public Mono<ServerResponse> createMember(final ServerRequest request) {
        return ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(request.bodyToMono(Member.class)
                        .flatMap(member -> Mono.fromCallable(() -> repository.save(member))
                                .then(Mono.just(member))), Member.class);
    }
}
```

Now that it's functional, it seems like it's become easier to understand, or it's become more difficult...

## Netty from Tomcat

Spring WebFlux's default application server is Netty. The reason, which can be easily inferred, is that [Netty](https://netty.io) was created based on the idea of ​​non-blocking from the beginning. Tomcat is of course synchronous and blocking, so it seems that there are the following differences when compared with Netty.

- Tomcat: requests and threads are 1:1
- Netty: requests and threads are N:1

Of course, you can also use Tomcat instead of Netty like Spring MVC. For example, in Gradle you would write something like the following:

```groovy
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-webflux:2.3.3.RELEASE') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-reactor-netty'
    }
    implementation('org.springframework.boot:spring-boot-starter-tomcat:2.3.3.RELEASE')
}
```

## R2DBC from JPA/JDBC

This is also a similar story with application servers. Traditional ORMs such as JPA/JDBC are blocking, so let's change to [R2DBC](https://r2dbc.io), which supports non-blocking. As was the case with NIO, it seems that using R2DBC with blocking can improve performance in some cases.

## new abstraction class

WebFlux also changes the abstraction classes used in Spring MVC. This is also to support writing as a function and non-blocking.

| Type | Spring MVC | Spring WebFlux |
|---|---|---|
| Request | HttpServletRequest | ServerHttpRequest |
| Response | HttpServletResponse | ServerHttpResponse |
| Call other APIs | RestTemplate | WebClient |

In the case of WebClient, RestTemplate is changed to `deprecated`, so you need to consider installing it even if you use Spring MVC. In fact, it is said that performance may be improved compared to Resttemplate even when used with Spring MVC.

## Where should I use it?

Now, we have taken a quick look at the features of WebFlux, but what do you think? The way it is written is quite different, and it is based on Reactor, which is completely different from the servlet-based MVC, so implementing Spring WebFlux is quite a pain. In fact, it cannot be used at the same time as Spring MVC (even if Dependency is forcibly added, MVC takes precedence and WebFlux functions may not work), and other frameworks such as Spring Security must be rebuilt for Spring WebFlux, so if there are many existing systems or libraries developed based on Spring MVC, the scope of the impact cannot be measured.

In terms of performance, non-blocking is strong under the condition that ``there are more requests than the specified number of threads.'' So changing to non-blocking doesn't mean that performance with a single thread will improve. [^3]

However, you can consider implementing WebFlux in the following cases.

1. Create a completely new service from scratch
1. When there are multiple services and there are many calls between services (microservices)
1. For BFF[^4]

## lastly

What I was trying to summarize briefly ended up becoming quite long, but I think I was able to learn a lot thanks to you. WebFlux has been around for a few years now, and as RestTemplate is scheduled to be deprecated, there may come a day when you will eventually need to migrate everything to the WebFlux foundation. Lately, the keywords asynchronous, functional, and reactive have become very popular.

Static type languages ​​were first created, then dynamic type languages ​​were also created, and when I see phenomena such as TypeScript returning to the static world, I wonder if the day may come when we will transition from functional to imperative language again. However, these paradigms are not absolute, so I feel that every talented programmer needs to be able to use them appropriately and in a timely manner. The road to learning as an engineer is truly endless.

[^1]: Actually, it seems that the functional type is a little slower when there is no bottleneck due to the number of threads. In reality, implementing a functional API is more complicated. However, this difference seems to be enough to be considered as a trade-off when considering factors such as code readability and ease of implementation.
[^2]: Streams are synchronous, so data is produced and consumed at the same time. However, with Reactive Stream, it cannot be said that data production will immediately lead to data consumption. Because it's asynchronous.
[^3]: In fact, some say that WebFlux is a little slower than MVC when it comes to processing with a single thread. I feel like it's similar to the reason why Stream is slower than a for loop...
[^4]: Abbreviation for Back-end For Front-end. A backend that is a type of microservice that brings together multiple endpoints and creates a unique object. The front end only needs to call one endpoint.
