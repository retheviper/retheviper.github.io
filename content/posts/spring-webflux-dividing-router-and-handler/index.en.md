---
title: "A small study on WebFlux's Functional Enpoint"
date: 2021-08-30
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - webflux
  - kotlin
  - java
  - rest api
---

Last time, I wrote a post called [Should I Use Functional Endpoints with WebFlux?](../spring-webflux-router/). This time, instead of comparing `Controller`/`Service` with `Router`/`Handler`, I want to focus on how to structure an implementation when you choose the functional endpoint style.

I don't use WebFlux in actual work, so I think there are various patterns, but I think what you should consider when using `Functional Endpoint` is how to separate `Router Function` (hereinafter referred to as `Router`) and `Handler Function` (hereinafter referred to as `Handler`). Although `Router` and `Handler` are conceptually different, the application can be implemented without problems even if they are combined into one class, so it is more about the architecture of the application than the specifications and ideas of the framework.

So, this time I would like to think about the cases in which `Router` and `Handler` are separated and the cases in which they are not separated from several perspectives.

## Should Router and Handler be separated?

In the case of Spring MVC, it is common knowledge to clearly separate `Controller` and `Service`. This is true not only in terms of architecture, but also in terms of framework philosophy (design perspective).

Because of this precedent, even in the case of WebFlux, which also belongs to the Spring Framework, even if we introduce a new concept called `Functional Endpoint`, I think we tend to think that it is necessary to separate `Router` and `Handler`. At first glance, it appears that the correspondence relationship `Controller ≒ Router, Service ≒ Handler` holds true, and many sample codes that can be searched online are written with such a structure.

However, if you want to write an actual application with `Functional Endpoint`, there are some things you need to think about. For example, is it true that `Router` and `Handler` have a one-to-one correspondence with `Controller` and `Service`, respectively? If not, is there a need to follow the MVC pattern? How will it affect implementation? I think there are such things. So, this time I will talk about `Functional Endpoint` from these perspectives.

## About correspondence

Spring's [official documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-fn-overview) introduces WebFlux functional endpoints with sample code like the following.

```kotlin
val repository: PersonRepository = ...
val handler = PersonHandler(repository)

val route = coRouter { 
    accept(APPLICATION_JSON).nest {
        GET("/person/{id}", handler::getPerson)
        GET("/person", handler::listPeople)
    }
    POST("/person", handler::createPerson)
}

class PersonHandler(private val repository: PersonRepository) {

    // ...

    suspend fun listPeople(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun createPerson(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun getPerson(request: ServerRequest): ServerResponse {
        // ...
    }
}
```

Looking at the official sample where `Handler` is a separate class, it seems that the correspondence relationship `Controller ≒ Router, Service ≒ Handler` also holds true. It is important to note that unlike [@RestController](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/RestController.html) and [@Service](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/stereotype/Service.html), there are no `@Router` and `@Handler` annotations. This probably means that from the perspective of the Spring framework, there is no need to separate `Router` and `Handler`.

Therefore, I think it is difficult to conclude that the correspondence relationship `Controller ≒ Router, Service ≒ Handler` holds, at least from the perspective of application architecture.

So, what would happen if you actually used annotations to DI `Router` and `Handler`? I think a typical example would be something like the following:

```kotlin
@Configuration
class PersonRouter(private val handler: PersonHandler) {

    @Bean
    fun route(): RouterFunction<ServerResponse> =
        coRouter {
            accept(APPLICATION_JSON).nest {
                GET("/person/{id}", handler::getPerson)
                GET("/person", handler::listPeople)
            }
            POST("/person", handler::createPerson)
        }
}    

@Component
class PersonHandler(private val repository: PersonRepository) {

    // ...

    suspend fun listPeople(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun createPerson(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun getPerson(request: ServerRequest): ServerResponse {
        // ...
    }
}
```

Unlike `Contoller`, which requires the class itself to be registered as `@Component`, [RouterFunction](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/RouterFunction.html) is `Functional Interface`, so the method that implements it must be registered as `@Bean`. And since `@Congifuration` is generally in charge of registering `@Bean` in the application in Spring, the Router annotation naturally follows as well. `Handler` will normally be registered as `@Component`.

In this example, you can tell that `Router` and `Handler` are separated by looking at the class structure and implementation, but if you only look at the annotations the design can feel a little unusual. Since the implementation itself is not especially difficult, it would seem possible to create matching annotations for each part, so why is it structured this way? The [official documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-programming-models) gives the following explanation.

> The big difference with annotated controllers is that the application is in charge of request handling from start to finish versus declaring intent through annotations and being called back.

In other words, the difference between a "Contoller with annotations" and a "Functional Endpoint" is that the former "expresses callbacks and intent with annotations," while the latter "is responsible for handling requests from start to finish." Since there are such differences in terms of programming models, it may be natural that there are no annotations. As a result, it seems that the correspondence relationship `Controller ≒ Router, Service ≒ Handler` does not apply, at least from the perspective of a programming model.

## In terms of dispersion of responsibility

Looking at the annotation implementation, it becomes clearer that what separates `@Controller` and `@Service` is the framework architecture and philosophy. The implementation of each annotation is as follows.

```java
@Target(value=TYPE)
@Retention(value=RUNTIME)
@Documented
@Component
public @interface Controller

@Target(value=TYPE)
@Retention(value=RUNTIME)
@Documented
@Component
public @interface Service
```

Since both have the same implementation, in extreme terms, even if you add `@Service` to `Controller`, they are functionally the same. In `@Service`, the following comment clearly states that the reason for the existence of this annotation is that it is "based on design patterns."

> Indicates that an annotated class is a "Service", originally defined by Domain-Driven Design (Evans, 2003) as "an operation offered as an interface that stands alone in the model, with no encapsulated state."
May also indicate that a class is a "Business Service Facade" (in the Core J2EE patterns sense), or something similar. This annotation is a general-purpose stereotype and individual teams may narrow their semantics and use as appropriate.

Therefore, from an application design perspective, `Controller` has only the responsibility of receiving requests, returning responses, and connecting endpoints to `Service`, and `Service` can be considered to have the responsibility of processing business logic. From the same perspective, `Router` and `Handler` could also be written to have the same obligations, albeit without annotations.

However, the problem is the definition of "responsible for handling requests from start to finish." If you look closely at the sample code above, all Handler methods take [ServerRequest](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerRequest.html) as an argument and the return value is [ServerResponse](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerResponse.html). This means that even if `Router` and `Handler` are purposely divided into separate classes, requests and responses will be processed by `Handler`.

At this point, you may be thinking, "Couldn't we simply change the arguments and return type of `Handler`, just like with `Controller` and `Service`?" However, that would go directly against the framework's design. The JavaDoc for `ServerRequest` and `ServerResponse` explicitly states that `ServerRequest` and `ServerResponse` are handled by [HandlerFunction](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/HandlerFunction.html), as shown below.

```kotlin
/**
 * Represents a server-side HTTP request, as handled by a {@code HandlerFunction}.
 *
 * <p>Access to headers and body is offered by {@link Headers} and
 * {@link #body(BodyExtractor)}, respectively.
 *
 * @author Arjen Poutsma
 * @author Sebastien Deleuze
 * @since 5.0
 */
public interface ServerRequest {

/**
 * Represents a typed server-side HTTP response, as returned
 * by a {@linkplain HandlerFunction handler function} or
 * {@linkplain HandlerFilterFunction filter function}.
 *
 * @author Arjen Poutsma
 * @author Juergen Hoeller
 * @author Sebastien Deleuze
 * @since 5.0
 */
public interface ServerResponse {
```

As you can see from the above, WebFlux is designed to handle `ServerRequest` and `ServerResponse` with `HandlerFunction`. Therefore, the question of whether `Handler` handles "only" business logic, like the existing `Service`, is whether it is possible as an implementation.

However, from the perspective of ``dispersion of responsibility,'' the idea of ​​dividing classes according to responsibility is not wrong. Therefore, it is possible to consider cases where classes in charge of business logic are operated separately from `Handler`, but I think it is not necessary that the criteria for separating classes should be `Router` and `Handler`.

## in terms of testing

When creating unit tests in Java using JUnit etc., the tests themselves are created for each use case, but I think there are many cases where these tests are grouped together for each class. So, when writing unit tests from the same perspective, if `Router` and `Handler` are separated, the unit tests will naturally be separated by that unit.

The problem with doing this, however, is that the test itself may not be very meaningful. First of all, `Router` will only have the role of simply connecting the endpoint and `Handler`, so the test will be limited to ``Are `HadlerFunction` being called as expected?'' And in the case of `Handler`, there is a problem that it receives `ServerRequest` and emits `ServerResponse`, which makes testing very difficult.

The reason why receiving `ServerRequest` and emitting `ServerResponse` is a problem is because it is difficult to generate an instance of `ServerRequest`, and even in the case of `ServerResponse`, it is difficult to extract the response body. So, I think it will be done with [WebTestClient](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/reactive/server/WebTestClient.html), but when using `WebTestClient`, the actual API will be called using endpoints and HTTP methods, so as a result, the intention to test `Handler` will have to extend to the test of `Router`. This not only makes it difficult to compile test cases for each class, but also makes testing only for `Router` virtually meaningless.

## So what should I do?

From the three viewpoints discussed so far, there is not much reason to make `Router` and `Handler` separate classes, and rather it seems that various problems will arise with separate classes. However, I think this does not necessarily mean that it is necessary to separate the routing and business logic for endpoints. As I mentioned earlier, I think it would be better to just not use `Router` and `Handler` as the criteria for separating classes. For example, suppose you have the following code.

```kotlin
@Configuration
class PersonRouter(private val repository: PersonRepository) {

    @Bean
    fun route(): RouterFunction<ServerResponse> =
        coRouter {
            GET("/person/{id}") {
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(
                        repository.findById(it.pathVariable("id").toLong())
                            .map { record ->
                                PersonDto(record.id, record.name)
                            }
                    ).awaitSingle()
                }
        }
}
```

In `Handler`, there is not much that can be called business logic other than the part where `Body` is created. Therefore, it seems like it would be a good idea to separate only `Body` and leave it to another class (`Service`). For example:

```kotlin
@Configuration
class PersonRouter(private val service: PersonService) {

    @Bean
    fun route(): RouterFunction<ServerResponse> =
        coRouter {
            GET("/person/{id}") {
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(service.getPersonById(it.pathVariable("id").toLong()))
                    .awaitSingle()
                }
        }
}

@Service
class PersonService(private val repository: PersonRepository) {

    suspend fun getPersonById(id: Long): Mono<PersonDto> =
        repository.findById(id)
                  .map { PersonDto(it.id, it.name) }
}
```

By doing this, you will no longer have to access `Repository` directly from `Router`, and the various problems mentioned above will be resolved.

## Finally

Although it is possible to separate business logic using the method presented here, there are still questions that remain if you take that method. Is this really `Functional`? `Functional Endpoint` is explained as `Lambda-based`, but since `Lambda` is not used, isn't the shape different from the design intention? In the first place, since it is a framework with a different concept from Spring MVC, wouldn't a different approach than the existing one be required? And so on.

It is quite difficult to judge these issues, but personally, I think it is difficult to always apply new methodologies just because the technology is new, and I don't think it is wrong to follow a good existing system if it exists. The official Spring documentation states, ``If you have a Spring MVC application that works fine, there is no need to change,'' but similarly, if there is an existing, verified architecture, I think it's not a bad idea to apply it as is. Well, if we were to use the MVC pattern when introducing WebFlux in the first place, there would be no reason to worry about such things...In fact, I feel that if this kind of programming model increases, new architectures will be created in the future. That's what I felt while writing this post.

See you soon!
