---
title: "I tried Ktor"
date: 2021-07-18
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - kotlin
  - ktor
  - exposed
---
Kotlin is becoming popular as a server-side language, but even when using Kotlin, I think Spring is still often used as a web framework. The reason may be due to the circumstances of each company, but the general reason is that while Kotlin is familiar to Java engineers, this is not the case with frameworks, and there is no framework as tested as Spring. There are some companies that are still using Struts and are trying to migrate to Spring.

Kotlin is fully (almost) compatible with Java, so there is no big problem if you migrate an application written in Java to Kotlin as is. It is certainly advantageous that it is more productive than Java, and that you can use many libraries such as Spring as well as Jackson, Apache POI, JUnit, Jacoco, etc., and I think that is certainly a reason why companies are considering introducing Kotlin. Since there are a large number of Java engineers, one of the advantages is that recruiting engineers is cheaper.

However, if you use Kotlin, you may want to consider introducing a library or framework written in Kotlin in the long run. Even if the Byte code generated as a result of compilation is exactly the same as Java, the original source code is in a different language, so there may be inconveniences for the user code (client code), or it may not be optimized for Kotlin. Also, Kotlin can be compiled natively in addition to JVM, so if you want to create a native app, you will need to choose an API that does not depend on Java.

So, this time I would like to introduce Ktor, a web framework based on JetBrains, and Exposed, an ORM that can be used with Ktor, and compare it with Spring.

## Ktor

Ktor is a lightweight web framework for microservices developed at JetBrains. There are various things written in the introduction on the [official homepage](https://ktor.io), but I think it has the following characteristics in particular compared to Spring.

## LightweightAlthough Spring is said to be lightweight, it starts up slowly, so as an implementer, I don't feel like it's very lightweight. I think that the slow startup of applications written with Spring is due to the architecture of the framework itself, which loads various annotations at startup and completes DI, settings, etc. Therefore, there are known techniques to shorten startup speed, such as by setting the DI object to Late Init.

However, Ktor starts up quite quickly. I haven't benchmarked an app of the same size using both Spring and Ktor, so I won't go into exact numbers, but from my experience it's several times faster. For example, a Spring WebFlux application that implements In-memory H2 and basic CRUD took about 2.7 seconds to start on my PC.

```bash
2021-07-18 15:08:25.150  INFO 29047 --- [main] c.r.s.SpringWebfluxSampleApplicationKt   : Started SpringWebfluxSampleApplicationKt in 2.754 seconds (JVM running for 3.088)
```

When I implemented the Ktor app with the same configuration, it took less than 1 second to start.

```bash
2021-07-18 15:09:29.683 [main] INFO  ktor.application - Application started in 0.747 seconds.
```

This is probably due to the fact that it basically does not do DI, does not use many annotations (doesn't use Reflection), and Ktor itself has only the minimum necessary functions to create a REST API.

Faster application startup is an advantage in terms of reducing the time it takes to test, but it is also suitable for serverless applications. I have experience using AWS Lambda and Azure Functions, but I have never considered using Java or Kotlin in this case. With serverless, the app is not always running, so it has to be started every time a request occurs. Therefore, Spring, which is slow to start, was not considered in the first place. When using Ktor, the startup speed can be significantly reduced, so if the startup speed of the JVM is acceptable, I think it is at a level where you can consider implementing a serverless architecture.

## Expandable

This is related to the fact that Ktor is lightweight, but if there is a necessary function, you will need to add a plugin (module) or implement it yourself. The code will look like this:

```kotlin
fun main() {
    embeddedServer(Netty, port = 8080, host = "127.0.0.1") {
        install(CORS)
        install(Compression)
        install(Sessions) {
            cookie<MyCookie>("MY_COOKIE")
        }
        install(ContentNegotiation) { // kotlinx.serialization
            json()
        }
    }.start(wait = true)
}
```

Therefore, I think that immediately after introducing Ktor, there may be some disadvantages in terms of module management and development speed. In particular, features such as [role-based authorization](https://en.wikipedia.org/wiki/Role-based_access_control), which are basically provided by Spring Security, are not yet available as official plugins, so in some cases you have no choice but to implement them yourself. Personally, I think modularization itself becomes an advantage once you get used to it, but it is a disadvantage compared to Spring in the early stages of implementation. Also, Ktor does not support DI and there is no official JetBrains module, so if you want DI you need to add [Injekt](https://github.com/IVIanuu/injekt), [Kodein](https://kodein.org/Kodein-DI/?6.3/ktor), [Koin](https://insert-koin.io), and so on as dependencies. Depending on the architecture, DI may not be necessary and `object` can be used instead, so I think you need to think carefully before deciding on the architecture.

## Coroutine compatible

Spring WebFlux was also like that, but recently many web frameworks have asynchronous and non-blocking support. Even now that PaaS has become popular and infrastructure can be easily built and hardware has become cheaper, if there is a place where performance can be improved with software, I think it is well worth it. If so, I think it would be a good choice to introduce a framework that supports asynchronous and non-blocking.

In Ktor, to implement routing, you will call functions such as [route](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/route.html) of [Route](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/-route/index.html), or [get](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/get.html), [post](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/post.html), [put](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/put.html), and [delete](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/delete.html). This is very similar to Spring WebFlux's Router/Hanlder Function. Expressed in code, it looks like this:

```kotlin
routing {
    get("/hello") {
        call.respondText("Hello")
    }
}
```

We will then implement the body of the function for each Http method, which is basically `suspend`. This means that the code becomes asynchronous even if the implementer is not aware of it. In the case of Spring WebFlux, it was easy to implement using Coroutine, but I feel that Ktor's unique advantage is that you don't even have to be aware of `suspend`.

## Test

Testing is possible using `ktor-server-test-host`, `kotlin-test`, JUnit, etc. I think there are various ways to write unit tests in Spring, but the way you write them is just more Kotlin-like, and basically the way you test them doesn't change much. For example, to test the response of `Get`, you can write code like the following:

```kotlin
@Test
fun getMember() {
    withTestApplication(Application::module) {
        handleRequest(HttpMethod.Get, "api/v1/web/members/$id").apply {
            assertEquals(
                actual = response.status(),
                expected = HttpStatusCode.OK
            )
            assertEquals(
                actual = response.content,
                expected = Json.encodeToString(
                    MemberResponse(
                        id = id,
                        userId = userId,
                        name = name
                    )
                ),
            )
        }
    }
}
```

## Exposed

A typical ORM written in Kotlin that can be used with Ktor is [Exposed](https://github.com/JetBrains/Exposed). As with Java's [jOOQ](https://www.jooq.org), the good thing about using SQL DSL is that you can use it as if you were writing a query in code (although the SQL is automatically generated by interpreting the DSL). For example, the code to retrieve records from a table called User is as follows.

```kotlin
val userInUsa: List<User> = transaction {
    UserTable.select {
        UserTable.deleted eq false
    }.map {
        User(
            id = it[UserTable.id],
            name = it[UserTable.name],
            country = it[UserTable.country]
        )
    }.filter {
        it.country = Country.USA
    }
}
```

Also, Exposed allows you to use the DAO pattern, so if you write a query using the DAO pattern, you can do something like the following. It seems to be usable in a similar way to JPA and R2DBC. (I think the disadvantages are probably the same)

```kotlin
val userInGermany: List<User>  = transaction {
    User.find { (UserTable.country eq Country.GERMANY) and (UserTable.deleted eq false)}
}
```

Also, a feature of Exposed is that by defining the table as code, it can be reflected in the DB. Up until now, I have often used [Liquibase](https://www.liquibase.org) and [Flyway](https://flywaydb.org) to manage the shape of the DB, but personally, considering cases where there is a discrepancy between the table definitions of the actual DB and the application, I think it is much better to define it in the code like this from the perspective of the data owner. I think it will be particularly convenient for development in cases where table definitions are frequently modified or there are many microservices.

The Exposed table definition can be done as follows.

```kotlin
object Member : IntIdTable() {
    val userId: Column<String> = varchar(name = "user_id", length = 16)
    val name: Column<String> = varchar(name = "name", length = 16)
    val password: Column<String> = varchar(name = "password", length = 255)
    val deleted: Column<Boolean> = bool("deleted")
    val createdBy: Column<String> = varchar("created_by", 16)
    val createdDate: Column<LocalDateTime> = datetime("created_date")
    val lastModifiedBy: Column<String> = varchar("last_modified_by", 16)
    val lastModifiedDate: Column<LocalDateTime> = datetime("last_modified_date")
}
```

The actual SQL that will be issued will be as follows.

```sql
CREATE TABLE IF NOT EXISTS "MEMBER" (ID INT AUTO_INCREMENT PRIMARY KEY, DELETED BOOLEAN NOT NULL, CREATED_BY VARCHAR(16) NOT NULL, CREATED_DATE DATETIME(9) NOT NULL, LAST_MODIFIED_BY VARCHAR(16) NOT NULL, LAST_MODIFIED_DATE DATETIME(9) NOT NULL, USER_ID VARCHAR(16) NOT NULL, "NAME" VARCHAR(16) NOT NULL, PASSWORD VARCHAR(255) NOT NULL)
```

In the case of JPA and R2DBC, by defining an Auditable class and having entities inherit from it, it was possible to share columns and link with Spring Security, but you can also do something similar with Exposed.

```kotlin
abstract class Audit : IntIdTable() {
    val deleted: Column<Boolean> = bool("deleted")
    val createdBy: Column<String> = varchar("created_by", 16)
    val createdDate: Column<LocalDateTime> = datetime("created_date")
    val lastModifiedBy: Column<String> = varchar("last_modified_by", 16)
    val lastModifiedDate: Column<LocalDateTime> = datetime("last_modified_date")
}

object Member : Audit() { // Creates the table including the Audit columns
    val userId: Column<String> = varchar(name = "user_id", length = 16)
    val name: Column<String> = varchar(name = "name", length = 16)
    val password: Column<String> = varchar(name = "password", length = 255)
}
```

If you are used to MyBatis, etc., it may take some time to adapt, but basically, except for table definition, you will be writing almost all SQL statements using Kotlin code, so I think it will be convenient.

## Finally

That's all for my impressions and introduction after implementing a simple CRUD app with Ktor + Exposed. In summary, I thought it was a configuration that was specialized for microservices, as I was able to write fairly quick code and the performance was good. Also, as mentioned at the beginning, it is good that it is a pure Kotlin framework. In the introduction of Ktor, we emphasize that it is based on Kotlin Multiplatform and can deploy applications to any platform, so I think it can be used in a variety of places.

There are still some modules and functions that are lacking compared to Spring and other Java libraries, but in addition to Exposed, we are also developing Kotlin-based libraries such as ORMs such as [Ktorm](https://www.ktorm.org), and Intellij has strong support for Ktor, so we can expect continued development. Personally, even though I don't feel comfortable using it at work, I definitely want to consider installing it when I want to create my own apps. I'm happy to see that more and more things are being done with Kotlin.

See you soon!
