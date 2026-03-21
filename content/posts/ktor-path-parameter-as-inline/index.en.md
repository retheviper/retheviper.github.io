---
title: "Receiving Path Parameters as Inline Classes"
date: 2023-03-26
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - kotlin
  - ktor
  - exposed
---
Recently, I've been creating a simple web application as a side project, and I'm using Ktor as the server-side framework. I've been using Spring for a long time, so it's fun to try making something with a different framework from time to time.

Well, the theme of this article is how to receive Path Parameter in Ktor and treat it as an Inline Class. Although it does not require advanced technology, I would like to introduce what I tried because I thought it might be a way to write code that is more type-safe and convenient to use.

## What is Inline Class?

From Kotlin 1.6, a feature called [Inline Class](https://kotlinlang.org/docs/inline-classes.html) has been added. Inline Class is a feature that can be used as a wrapper for types. For example, suppose you have the following code. There are players, and the performance of each player is expressed.

```kotlin
// Player
data class Player(val id: Int, val playerRecordId: Int)
// Player record
data class PlayerRecord(val id: Int, val score: Int)
```

Suppose we have a method like the following:

```kotlin
// Record a player's score
fun createPlayerRecord(val playerId: Int, val score: Int) {
    // ...
}
```

You must pass the player's ID and score when calling this method. However, in this code, both the player ID and score are of type Int, so it is possible to make a mistake such as passing the player ID as the score. Therefore, you can write code more safely by defining a type that wraps the player ID and score and using that as shown below.

```kotlin
// Player ID
@JvmInline
value class PlayerId(val value: Int)

// Score
@JvmInline
value class Score(val value: Int)
```

By defining two Inline Classes, the previous function can be modified as follows. This way, if you specify the wrong parameter, the compiler will issue an error, allowing you to write code more safely.

```kotlin
// Record a player's score
fun createPlayerRecord(val playerId: PlayerId, val score: Score) {
    // ...
}
```

Just looking at this, what is the difference between Inline Class and Java wrapper class or normal data class? A question may arise. There are also existing features like typealias. However, an Inline Class has the characteristic that while it is treated as a primitive type at compile time, it behaves like a wrapper class at runtime. Therefore, Inline Classes have the advantage of ensuring type safety while having less impact on performance.

## Handling Path Parameters with Ktor

In Ktor, in order to receive a Path Parameter, you need to write it as follows.

```kotlin
routing {
    get("/{id}") {
        val id = call.parameters["id"]?.toInt()
    }
}
```

To obtain the Path Parameter, the value specified in `{id}` in [Parameters](https://api.ktor.io/ktor-http/io.ktor.http/-parameters/index.html) from [ApplicationCall](https://api.ktor.io/ktor-server/ktor-server-core/io.ktor.server.application/-application-call/index.html) is first read as a String. Then, it is further converted to Int using `toInt()`. Now you can receive the Path Parameter as an Int and use it in the process.

## Improving Path Parameter Handling

At first glance at the sample using `call.parameters`, you may think that there is not much room for improvement since it is a simple code, but there are actually several problems with this code. For example, you need to consider errors during Int conversion. When converting to Int with `toInt()`, if a string such as `null` or `"abc"` is passed, `NumberFormatException` will occur. Also, when converting to Int with `toInt()`, if a value exceeding `Int.MAX_VALUE` is passed, `NumberFormatException` will occur. In this way, when receiving a Path Parameter, you must always check `null` and `NumberFormatException`.

Also, `ApplicationCall` can only be called within functions like `get()` and `post()`. This means that you need to write similar processing (error handling, etc.) for each endpoint. I don't want the same process to be repeated multiple times in the router, so I would like to make this code common. Therefore, it would be a good idea to define an extension function for `ApplicationCall` as shown below to standardize the process of acquiring Path Parameters.

```kotlin
fun ApplicationCall.getIdFromPathParameter(name: String): Int {
    val parameter = parameters[name] ?: throw IllegalArgumentException("id is required")
    val id = parameter.toIntOrNull() ?: throw IllegalArgumentException("id must be an integer")
    return id
}
```

By doing this, you can standardize error handling as shown below. Exceptions are handled using try-catch, but it would be a good idea to add error handling using [Status Pages](https://ktor.io/docs/status-pages.html) if necessary.

```kotlin
routing {
   get("/{id}") {
       try {
           val id = call.getIdFromPathParameter("id")
       } catch (e: IllegalArgumentException) {
           call.respond(HttpStatusCode.BadRequest, e.message)
       }
   }
}
```

## Wrap Path Parameters with Inline Classes

Now that we have standardized how to retrieve an `Int` from a path parameter, the next step is to make the code safer by wrapping that value in an inline class. The simplest approach is to have the extension function return an inline class directly, as shown below.

```kotlin
routing {
   get("/{playerId}") {
        val id = PlayerId(call.getIdFromPathParameter("playerId"))
   }
}
```

Because an inline class still behaves like a class in code, you can also make this generic. Conceptually, it would look like this:

```kotlin
routing {
   get("/{playerId}") {
        val id = call.getIdFromPathParameter<PlayerId>("playerId")
   }
}
```

In this way, the process of converting the return value of `getIdFromPathParameter()` to `PlayerId` can be performed in `getIdFromPathParameter()`. Also, since it is generic, it would be safer to limit the types that can be used to a specific Interface and have an Inline Class for ID implement it. So, first of all, define a common ID-based interface as shown below.

```kotlin
// Common interface for IDs
interface Id {
    val value: Int
}

// Player ID
@JvmInline
value class PlayerId(val value: Int) : Id

// Director ID
@JvmInline
value class DirectorId(val value: Int) : Id
```

Then, make `getIdFromPathParameter()` generic as shown below so that it can only receive classes that implement `Id`.

```kotlin
// Extension function to get an ID
inline fun <reified T: Id> ApplicationCall.getIdFromPathParameter(name: String): T {
    val parameter = parameters[name] ?: throw IllegalArgumentException("id is required")
    val id = parameter.toIntOrNull() ?: throw IllegalArgumentException("id must be integer")
    return T::class.java.getDeclaredConstructor(Int::class.java).apply { isAccessible = true }.newInstance(id)
}
```

The fix is easy: just create an instance of the specified `Id` type, wrap the Int value obtained from the Path Parameter, and return it. This makes it possible to ensure type safety while restricting it to only support Inline Classes that implement `Id`.

The only thing I would like is to have the variable name for the Path Parameter in the Inline Class as a companion object so that it can be shared, but unfortunately it seems difficult to do so. This is because companion objects of interfaces cannot be overridden, and inline classes cannot implement abstract classes. Therefore, there seems to be room for improvement by allowing the Path Parameter name to be specified in another way, which would reduce the arguments of the extension function and make it simpler.

## Finally

I used Ktor for the first time in a while and wrote a blog post about it, but what did you think? I've been only implementing Rest APIs for a long time, so I feel like I'll be in a state where I won't be able to discover anything unless I try something new in my private life. It's been five years since I started blogging, and while I feel like there's still a lot to learn, I also feel that it's not easy to put it into words.

Even if I have to update my blog less frequently, I will try my best to write better articles next time. See you soon!
