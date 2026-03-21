---
title: "Spring WebFlux, after a little experience"
date: 2020-12-20
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - webflux
  - r2dbc
  - reactor
  - loom
---
I've been using Spring MVC for a long time, so recently I've been trying to create a simple app using the combination of Kotlin + Spring WebFlux (I'll omit the introduction of Spring WebFlux itself since I covered it in the previous post). It was already three years ago that Spring WebFlux was introduced (it will soon be four years), so even though it has been a long time, I think that it is not widely used. So even if you look it up on the internet, you might not find anything that might be helpful.

This is probably because there are still many things to consider before Spring WebFlux can be fully implemented. For example, the technology as a framework is not yet mature. We must also take into consideration the fact that there are still many people who are not familiar with how to write Reactor, which is the core of this framework (although it is called Reactive, it is also slightly different from [RxJava](https://github.com/ReactiveX/RxJava)). From a company's perspective, there are risks in introducing new technology so quickly, and there is not much benefit in terms of having to consider the learning costs for engineers.

And as I introduced last time, there is a problem in terms of performance that simply converting an existing Spring MVC project to Spring WebFlux does not provide much benefit. If so, is it okay to introduce it from a new project? You might think that, but there are many cases where startups and venture companies are not familiar with the JVM language in the first place (this may be my personal impression, but I think such companies tend to use Python, Ruby, and JavaScript), so it may not be a consideration in the first place. Also, even if Spring WebFlux can be introduced because there are engineers who are familiar with the JVM language, it is still not free from the problems of ``untested'' and ``incurring learning costs'' mentioned above.At least for these reasons, I think it is still difficult to introduce Spring WebFlux at the enterprise level. However, since the future of Spring probably lies in WebFlux, there is a possibility that development will increasingly focus on WebFlux from now on, so it may be necessary to learn how to write Reactor even now. Also, even if it's not from the perspective of a single framework called Spring, improving concurrent processing performance through asynchronous processing is an important element in developing web applications that are frequently used by a large number of users, so you may need to at least get used to its concepts, ideas, and how to write code. In that sense, I would like to talk about my experience of writing my own application using Spring WebFlux and what I felt from it.

## Writing a Spring WebFlux app

Spirng WebFlux allows you to write code using the existing Controller + Service pattern. So, at first glance, I think it gives the impression that it can be operated in parallel with an existing Spring MVC project, or easily migrated to WebFlux by simply rewriting the existing code. However, I feel like this is not actually the case. In my case, I first created a simple CRUD sample (GitHub repository [here](https://github.com/retheviper/springwebfluxsample)), and then applied this to create a minor clone for [Mr. Adjustment](https://chouseisan.com).

Here, the sample used Spring Data JPA, but since I wanted to write a full-fledged non-blocking application with WebFlux, I decided to introduce [R2DBC](https://r2dbc.io/) and configure DB access asynchronously. The reason is that if even one synchronous process occurs within an asynchronous process, it is no longer asynchronous. Therefore, the ORM needs to use R2DBC accordingly.

Fortunately, R2DBC is not much different from Spring Data JPA or Spring Data JDBC in terms of usage, and mapping with the DB was easy by simply creating an Interface-style Repository and a DTO-style object (in Kotlin, a Data class was sufficient). Also, compared to JPA, there are fewer annotations, so it is easier to define objects as tables (just add `@Id`). And unlike Spring Data JDBC, it also automatically generates a query from the method name, so I thought it was easy at first. However, this is the first time I've used this technology, so there's no way it'll work. I ran into one problem.

## Cannot join

Personally, I believe that any technology has an overwhelming advantage over the existing technology until it reaches a stable period after a certain period of time, and there is always one or more factors that make it difficult to immediately switch to it (in other words, there are elements that make it difficult to abandon the legacy). In that sense, I think that the part where Spring Data R2DBC is not sufficient to replace existing ORMs right away is that there is no way to automatically perform joins (it is not possible to define relationships between tables as objects in advance).When using Spring Data JPA or JDBC, you can easily define relationships between tables (such as [@OneToOne](https://docs.oracle.com/javaee/6/api/javax/persistence/OneToOne.html)) by using annotations, and table relationships can be easily defined in code. However, this is a feature that is not yet supported by R2DBC. In this situation, I think there is a way to add the `@Query` annotation to the repository and directly define a method with SQL that includes Join, or to combine the two objects in the application.

In the former case, the code coverage would be impossible (and personally, even if the performance is good, I don't think the queries would be too complicated from a maintenance perspective), so I decided to go with the latter method. I thought it was an easier method because the object and repository are now 1:1, and even if a table is modified later, you only have to modify the object that applies to that table. However, that decision also had its problems. This is because the object returned by R2DBC as a result of SQL execution was not the object itself, but `Mono` or `Flux`.

### block() dilemma

Since the objects obtained from the repository are `Mono` and `Flux`, we need to think about how to combine (join) the two objects.

The easiest way is to use blocking `Mono` or `Flux`. `Mono` and `Flux` already have a method called `block()`, which can be used in synchronous code. For example, in Spring MVC, it is recommended to use WebClient rather than RestTemplate, so it is not impossible or strange for asynchronous and synchronous to coexist.

The problem is, if you go that route, you lose the benefits of async. This is because the great concurrency performance that asynchronous provides can end up becoming synchronous code if there is blocking at any point. In that case, the existing Spring MVC and ORM will suffice, and there will be no need to use WebFlux or R2DBC. So I decided to try another method.

### Mono + Mono or Flux + Flux

The best course of action here is to find a processing method that is suitable for asynchronous processing. So after doing some research, I found out that there is a way to combine two `Mono` or `Flux` as an array-like object. The answer is surprisingly simple; you can connect two `Mono` or `Flux` by using a method called `zipWith()`. The concatenated `Mono` or `Flux` will become a tuple according to Kotlin standards, so all you need to do is specify the tuple index (the concatenation source will be `t1` and the concatenation destination will be `t2`) from `map()` and use it. For example, the code below retrieves an object called `Paticipant` from the repository, then retrieves `CandidateParticipants` and combines it.

```kotlin
fun getParticipant(participantId: Long): Mono<ParticipantDto> =
    repository.findById(participantId)
        .zipWith(candidateParticipantHandler.getåCandidateParticipantsByParticipantsId(participantId).collectList())
        .map { mapDto(it.t1, it.t2) }
```

However, in order to join tables using this method, you need an environment where you can obtain two Mono and Flux. Therefore, if you use a method that issues a Select (such as a Get API), you need to carefully design and reflect the relationship between tables in the object. In other words, it is necessary to be able to obtain all objects to be joined using one key. After getting the first object, wouldn't it be better to introduce it with another key that that object has? I've thought about it, but unfortunately, as far as I know, it doesn't seem to be easy. This is because such an approach requires the following steps:

1. Query the join source table with the key and get the data as `Mono` or `Flux`
1. Call `map()` of the obtained `Mono` or `Flux`, extract further keys, and query the join destination table.
1. Combine join source and join destination tables

At first glance, there seems to be no problem, but at the stage where you query the joined table, another `Mono` or `Flux` is obtained, so the problem becomes how to retrieve the original object. If we use `block()` again, everything we've done will be ruined. So, although it is quite inconvenient, I think this is the only way to join tables using code at the moment.

## Is this the best?

In this way, I was able to somehow solve the problem I had with WebFlux. However, I personally feel uncomfortable with this approach. And I was able to find out the reason from Ron Pressle's [explanation](https://www.infoq.com/presentations/continuations-java/), the leader of Project Loom. According to him, current asynchronous programming has three problems:

## Easy to lose track of control flow

When writing code asynchronously, I sometimes feel like I forget what logic and purpose I was writing the code for. This is probably because we tend to care more about "non-blocking etiquette" than about the "business logic" needed to achieve the original purpose of the application. When writing code asynchronously, even simple conditional branching and repetition can make the code quite complex, making it easy to lose track of the flow of control. I don't have much experience with this as someone who has only used Java, but those who have experience writing asynchronous code with JavaScript will understand this. (There is also the famous callback hell problem...)

For example, as shown in this post, take a look at what we do to join two tables. If it were synchronous, all you had to do was get the joined data from the beginning, or declare two objects in sequence and process them. If you try to change even a simple process to asynchronous like this, you will end up writing code that is compatible with the asynchronous format rather than thinking about what you want to do with the code, so you may not understand the original purpose of what you are trying to do in the first place.

## Losing context

Async makes it very difficult to follow the stack trace when an exception occurs. This is because in the case of synchronization, one thread processes one request, so looking at the history left behind by that thread is sufficient to track what was executed and what the results were. However, with asynchronous processing, one request is processed across multiple threads, so if you just follow the history of one thread, you won't know what's going on.

## Code Contagion

When it comes to writing code asynchronously, you may end up eliminating the concept of synchronization from your entire application. This is because, as I mentioned earlier, if there is even one synchronous process mixed in with the asynchronous process, the entire process will become synchronous. Therefore, it becomes quite difficult to have synchronous and asynchronous coexistence in one application, and as a result, there are cases where asynchronous code "infects" other code. For example, in the WebFlux example, it is almost impossible to mix it with synchronous code, so the only option is to put the object in `Mono` or `Flux` (change it to asynchronous), connect it as a tuple with `zipWith()`, and process it with `map()` or `flatMap()`. If you take the opposite approach (such as bringing out the contents of `Mono` in `block()`), there is a problem that it would be better to write the code synchronously from the beginning.

## Still need async

The above are probably problems that everyone who has experienced asynchronous technology has encountered at least once, and I think many people can relate to them. However, despite these inconveniences, there is still a need for asynchronous programming. This is especially true as the current trend is that concurrent processing performance has become important for many web applications. In fact, [Google Analytics](https://support.google.com/analytics/answer/4589209?hl=ja) quotes KissMetrics and says, ``A 1 second delay in page response reduces conversions by 7%'' and ``47% of consumers expect web pages to load within 2 seconds.'' However, it can only be said that asynchronous can meet these demands. Therefore, we are trying to compensate for the shortcomings of asynchronous, such as trying `async`/`await`/`promise` in languages ​​like JavaScript and C#, and introducing something called [Coroutine](https://kotlinlang.org/docs/reference/coroutines-overview.html) in Kotlin.

In particular, Java uses OS threads directly, so it can process only a few thousand requests at the same time. Therefore, many asynchronous libraries have been created to overcome the limitations of the thread-based language itself. However, the library still had its limitations, so measures at the JVM level are currently being considered. It is called [Project Loom](https://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html), which I introduced earlier.

## The answer is Project Loom

Project Loom separates existing threads into virtual, lightweight threads called Fibers to improve concurrent processing performance, and at the same time, aims to "make asynchronous programming feel like synchronous programming" and eliminate the previous phenomenon where code changes depending on whether it is asynchronous or synchronous. It is said that Fiber can generate up to several million fibers, and the usage is not much different from existing threads, so there will be less code modification. Also, even if you write it in a synchronous manner, it will be processed internally as if you were using `async`/`await`, so there will be no problems like callback hell.Also, since Kotlin's Coroutine is only supported at the compiler level, it cannot fundamentally solve the three problems mentioned above, such as ``losing context'' and ``code contagion,'' but Project Loom is supported at the JVM level, so it has the advantage of eliminating all of these problems. It is also attractive that by changing the existing synchronous API to asynchronous, it can be adapted quickly without changing much in the client code.

Even if "asynchronous performance" is desired in many cases, many programmers are accustomed to "synchronous writing", so if Proejct Loom is officially released, I think that many of the existing projects using Coroutine, `async`/`await`/`promise`, and Reactice Stream will change to code using Fiber.

## Finally

In any field, the transition period is the most confusing and painful. And in programming, I think the paradigm of asynchronous programming is exactly that. Many geniuses have been working hard to overcome the shortcomings of asynchronous programming in a variety of languages, libraries, and frameworks, and I feel like we are finally starting to see the fruits of their efforts. Personally, I don't dislike the idea behind Reactive Steam and the concept of "reactive programming," but it's a way of writing that's so different from traditional programming that I personally wonder if it's worth the effort to accept the inefficiency of learning a completely different language for a specific purpose. Maybe that's just because I haven't really tried writing asynchronous code yet. However, this is also a problem that can be solved if there is a technology like Project Loom that allows asynchronous programming with a synchronous feel.

However, Project Loom is not a completed technology yet, and is still under development, so there is a possibility that limitations and problems will be discovered later. And even if it is adopted in WebFlux etc. in the future, we don't know when it will be released, so until then we have no choice but to use Reactive Stream or Coroutine, and it would be impossible to rewrite all the existing code later to match the release of Loom. I'd like it to come out as soon as possible, but it doesn't seem like it's going to be released within the next two years, so it might end up being completely different from what it is now.

Even so, it is attractive to replace the JVM itself and make asynchronous programming easier. Lately I've been hooked on Kotlin, but it's also fun to see how Java is changing. I feel like I can see a new world again.

See you soon!
