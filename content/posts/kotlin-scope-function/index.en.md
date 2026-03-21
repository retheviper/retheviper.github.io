---
title: "How to use Scope Function"
date: 2020-11-04
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - scope function
  - extension function
  - design pattern
---

I think there are many features that distinguish Kotlin from Java, but I think `Scope Function` is one of them. As I briefly touched on in a previous post, I believe that if used properly, these built-in functions can become a powerful weapon that not only allows you to write code that is more concise than Java, but also makes it easier to convey the creator's intentions. However, since it is still a new concept compared to other languages, it is difficult to know what the so-called best practices are, such as when to use it and how to use it.

I'm probably not the only one who thinks this way. If you search on the internet, you can find many articles about Scope Function, but I feel like most of them are just an introduction to how to use individual functions. So, I was wondering what exactly these functions are for, how to use them properly, and in what situations they should be used. When I looked into it, I found information in official documents and several blog articles, so I tried to organize them all together.

## First of all, what is Scope Function?

I mentioned Scope Function first, but what exactly is this? First of all, it seems necessary to know why these functions were given such names. The [official explanation](https://kotlinlang.org/docs/reference/scope-functions.html) says the following:

> The Kotlin standard library contains several whose sole purpose is to execute a block of code within the context of an object. When you call such a function on an object with a lambda expression provided, it forms a temporary scope. In this scope, you can access the object without its name. Such functions are called scope functions.

In short, these functions can be used when you want to limit processing to a specific object and execute a lambda in that context. Of course, this is not an entirely new form of function. In terms of syntax, it looks like you are simply calling a function on an object. However, once you look at the code, it becomes visually clearer that the processing is limited to a specific object because the code block creates a temporary scope. For example, the following code all do the same thing, but from the perspective of the reader, the latter one makes the scope of the processing clearer.

```kotlin
// Code without Scope Function
var john = Person("John", 20, Gender.Male)
john.doWarmingUp()
john.startToRun()

// Using Scope Function's let
var john = Person("John", 20, Gender.Male).let {
    it.doWarmingUp()
    it.startToRun()
    it
}
```

## What's the difference?

As mentioned above, we found that by limiting the scope of the code, the scope of processing becomes clear. However, this does not mean that we are ready to use Scope Function. In fact, there are other things to consider as well. This is because Scope Function has five functions: [with](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/with.html), [let](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/let.html), [apply](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html), [run](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/run.html), and [also](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/also.html). The existence of multiple functions means that you may need to choose a different function depending on the situation.

So what makes these functions different? The first thing to consider is the specifications. Scope Function internally executes the Lambda passed to it and returns the result. If so, the only difference in specifications is the objects handled by Lambda and the return values. First, the object to be processed (called a receiver) and Lambda are passed to the Scope Function as parameters. Each of these five Scope Functions differs in how you write access to the receiver passed here and what the return value will be after processing. If this is expressed as a table, it will look like this:

| Function name | Receiver access | Return value |
|---|---|---|
| with | this | final result |
| let | it | final result |
| apply | this | T |
| run | this | final result |
| also | it | T |

Also, the other four functions except with are also [Extension Function](https://kotlinlang.org/docs/reference/extensions.html). What is Extension Function? Simply add a function to the default class. In Java, this functionality can be achieved through class inheritance, creating a wrapper class, overriding, etc., but it is easy to define in Kotlin.

You may be wondering, "Even though it's a Scope Function, it's also an Extension Function?" The reason is simple. This is because even if a function is not defined as a function when the class is created, it can be called as if it were originally in the class. Any object can use Scope Functions without declaring them, as if they had been declared in advance, with the exception of with.

## Reference: it and this

`it` is used in Lambda with only one parameter. For example, in Java, even if there is only one parameter, you need to write it as follows unless you use [Method Reference](https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html).

```java
List<String> names = List.of("john", "jack");
// Predicate has only one argument, but
Optional<String> filtered = names.stream().filter(name -> "john".equals(name)).findFirst();
```

In Kotlin, the same situation can be expressed simply as `it` by omitting the parameter.

```kotlin
val names: List<String> = listOf("john", "jack")
// Omitted using it
val filtered = names.first { it == "o" }
```

After all, isn't it the same as `this`? You might think that, but the difference is that `it`'s scope is limited to Lambda parameters, and `this`'s scope can be local or global. This is because `this` points to the receiver itself, and without parameters it points outside the scope of Lambda. In other words, you cannot use it in Lambda without parameters, but you can use this.

## When should you use it?

Now that you know that Scope Function has five functions and how they differ from each other, you'll want to know how to use each one properly. There were a variety of opinions, so I tried to organize them into ones that made sense to me.

## with

`with` is not an Extension Function, so it can be used as a general function that receives an object as a parameter. Because of this feature, it can be used when it is necessary to perform similar processing on multiple objects. For example, you can use it when you don't want to separate methods on purpose in a for loop (because naming them is too troublesome, etc.). Also, since it is a Scope Function, it can also be used to clearly divide the scope of processing.

```kotlin
for (name in names) {
    println(name)
    with(name) {
        var rev = this.reversed()
        reversedName.add(rev)
    }
}
```

## let

This is used when you want to perform some processing using an object as a trigger. As `let` means, it is an image of holding the object and doing something with it. Also, the return value is the final result, so you can probably do something with it. Also, since it can be used like [Safe Call](https://kotlinlang.org/docs/reference/null-safety.html#safe-calls), it is also possible to specify the operation only when it is not Null. Therefore, it is better to use `let` for objects that can be null.

```kotlin
var name: String? = null
name?.let { println("name is not null") }
```

## apply

Use this when you want to return the receiver itself without using the receiver function in Lambda. That is, when you want to put a new value into a property of the receiver. A typical example is object initialization. Of course, just calling the constructor may be sufficient for initialization, but it seems to be useful when replacing the values ​​of the same object (for example, in the Configuration class). For example, consider the following cases.

```kotlin
if (devMode) {
    SomeConfig().apply {
        name = "devMode"
    }
}
```

## run

There are many people who say that this function should be avoided as much as possible since the other four functions are sufficient. Indeed, run can also be used as a normal function (`run {}`), so I don't really understand the difference between it and `with`. If you dare to use it, it seems to be when you override the value of the object. However, this can also be done with `let`, so there seems no need to use it. In many cases this was not recommended.

However, some people say that it is useful to use it to initialize objects. It's true that using `this` has the slight advantage that the code can be shorter than `let`, which uses `it`.

```kotlin
// Using run
var result1 = server.run {
    port = 8080
    get("/members:$port")
}

// Using let
var result2 = server.let {
    it.port = 8081
    get("/members:${it.port}")
}
```

## also

An object becomes a trigger and performs another process unrelated to that object. Therefore, even if the original object is Null, you can perform some processing when the object is called. By applying this, you can also use it like a conditional branch (ternary operator). For example, it looks like this.

```kotlin
var name: String? = null
name?.let { println("name is not null") } ?: also { println("name is null") }
```

## summary

[Kotlin Standard Library (let, run also, apply, with)](https://medium.com/@brijesh1794/kotlin-standard-library-let-run-also-apply-with-bb08473d29fd) presents the criteria for determining which of the five Scope Functions to use as a flowchart. Below is a simple translation of that flowchart. If you're having trouble deciding which one to use, it might be a good idea to write your code using these standards.

![Kotlin Select Scope Function](kotlin_select_scope_function.webp)

## application

The fact that Scope Function returns the receiver itself as a return value also means that it can be used as a builder pattern. Therefore, with appropriate combinations, you can also create method chains using Scope Functions. If you make good use of this, you will be able to write code with a fairly functional feel. Below is one such example.

```kotlin
// Chaining let
var three: Int = " abc ".let { it.trim() }.let { it.length }

// Chaining also
var jack: Person = Person("Jack", 30, Gender.MALE).also { println(it.name) }.also { it.doWarmingUp() }
```

## lastly

Actually, this Kotlin feature is not that new. Because there is [Groovy](http://groovy-lang.org), which is the same JVM language and provides functions like `with()` that also work. All I can think of is [Spock](http://spockframework.org) or Gradle...

However, I feel that the functions provided by Kotlin like this certainly convey the feeling that it is ``not new, but comfortable.'' Therefore, the number of Java programmers converting to Kotlin is probably increasing. Recently, languages ​​such as Python and JavaScript have become particularly popular (Kotlin doesn't even appear in the rankings of [TIOBE](https://www.tiobe.com/tiobe-index/)...), but it is a language that I would definitely recommend to anyone who wants a better balance between performance, stability, and comfortable development. So I want more people (including myself) to understand the appeal of Kotlin. I hope this post conveys a little of that.

See you soon!
