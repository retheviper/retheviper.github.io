---
title: "Kotlin as seen by a Java programmer"
date: 2020-10-25
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---
It's been a long time since Kotlin became the official language for Android, but there are still many companies in the web application industry that use Java as a server-side language (such as my case), and it seems that many companies in the mobile industry also use Java. Since Java 9, Java has been rapidly updated using agile development, and has absorbed the characteristics of so-called modern languages, but the original design of the language is old, and some remnants of the past have not been discarded for the sake of compatibility, so it must be quite different from languages ​​designed with fundamentally different philosophies. Also, [Kotlin Native](https://kotlinlang.org/docs/reference/native-overview.html), which does not use JVM, has been announced, so I feel that it may be useful in more situations than Java in the future. (I wonder if [GraalVM](https://www.graalvm.org/) is ever used...)

In summary, I haven't received any formal training yet, and I've only tried using Spring WebFlux while creating a simple application, so I'm sure there are a lot of things I don't understand, but I'd like to briefly introduce my impressions from the perspective of a Java programmer.

## This was good

First of all, let's start with what I liked about using it. In conclusion, the parts that I thought were good were mostly as I had imagined (as I expected).

## It still feels modern

When I look at the code written in Kotlin, I get the feeling that it would be something like this in a modern language. I feel like it's necessary from the definition of a modern language, such as Swift, Kotlin, and Go. I'm not very familiar with other languages, but I feel like these languages ​​are somehow similar to Python. For example, abbreviations are often used in basic grammar such as `var` and `fun`, types are specified after a colon, there is no semicolon, and there are commonalities such as `in`, `Range`, and `is`. Another thing is not using language, but generally accessing fields directly without using Getters/Setters. (Thanks to you, it's also convenient that you don't need to use Lombok.)

However, even though it has such a modern feel, Kotlin still feels more like Java. I guess you could say it just feels like a more relaxed version of the strict Java. For example, in Python it is `elif`, but in Kotlin it is `else if`. It is a language that Java programmers can easily adapt to, not only because it is a JVM language, but also because of its basic syntax. For example, you can label for loops.If I were to compare it with Java, I would say that not only did it eliminate redundancy, but it also felt like it was fundamentally changing the design of the Java language. For example, this is the case with methods that handle Null and Mutable. Basically, in Kotlin, variables cannot be null, and objects that can be null need to be declared as such from the beginning, and even when dealing with objects that can be null, Safe Call is enforced to support nulls as much as possible at the compiler level (this is probably the case with all modern languages). And when you declare a Collection etc., an Immutable object is basically created unless you intentionally declare it Mutable. This alone makes me feel like I can avoid the most common NPEs found in Java. It may seem tedious to have to make every declaration and call with a question mark, but don't we all know that a compile error is much better than a runtime error?

Also, I'm glad that Kotlin also has the features that I personally thought were nice to use in Python. For example, there are multiple returns and named arguments. I think the former is especially great because it can present a clear return value using the Pair/Triple type. This gave me the impression that although it was modern, it had not sacrificed the stability and robustness of Java.

However, recent Java has come close to achieving these benefits. (Although it still feels a little slow)

## class = not file

In Java, it is common knowledge that one class per file. Of course, there are cases where you write an Inner Class, but as the name suggests, it is included in the class, so it can be complicated when creating an instance. But with Kotlin, you can write multiple pure classes in one file.

This allows you to collect similar classes in one file. For example, in a pattern where multiple similar classes such as DTO, DAO, and Entity fail, I feel that it would be better to collect them in one file so that the package would not become complicated. Actually, it may be a matter of my preference while trying out Kotlin.

I can't say that having the freedom to choose one or the other is necessarily a good thing, but since whether or not you write multiple classes in a file is a matter of texture and does not affect the coding style during implementation (I don't think anyone writes imports directly these days...), I think it can be cited as a good point.### Functions can be added freely with extension functions

One of the often cited disadvantages of Java is that it is too verbose. Having to write boilerplate code every time is bad from a productivity standpoint. Because of these aspects of Java, various design patterns have developed, IDEs automatically generate code, and libraries like Lombok that reduce the amount of code are popular. The framework development project that I participated in was ultimately aimed at reducing redundant code.

In any case, Kotlin seems to have been designed as a language out of reaction or reflection on these issues. I felt that the design of the language not only copied the features of recent modern languages, but also showed a strong desire to improve Java.

## Standard library is very convenient

This is related to the reason why extension functions are useful, but the functions in Kotlin's standard library are also useful from the same perspective. For example, the so-called `Scope Functions` functions such as [let](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/let.html), [with](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/with.html), [apply](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html), [run](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/run.html), and [also](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/also.html) are already famous.

These can be handled in Java by creating a separate utility class, defining private methods, or inheriting a specific class and then defining new functions with overrides, but it is still time-consuming and is something you don't want to do. Kotlin solves this in a more functional way. For example, let's take an example of let. Suppose we have a data class like below.

```kotlin
data class Member(val region: String, val name: String)
```

Create one instance of this data class. Then it will look like this:

```kotlin
var john = Member("Tokyo", "John")
```

Suppose that later we also add a variable called jake as an instance of Member. jake must always be in the same region as john.

```kotlin
var jake = Member("Tokyo", "Jake")
```

If we were to organize the code using the Java concept, it would look like this: The idea is to use the same instance for region.

```kotlin
var tokyo = "Tokyo"
var john = Member(tokyo, "John")
var jake = Member(tokyo, "Jake")
```

If you write this as code when using let, it will look like the following.

```kotlin
var jake = john.let {
    Member(it.region, "jake")
}
```

Even if you want to declare a common region as a separate variable, jake's region will have the same value as the region specified for john. And in a sense, I feel that the intention of ``john and jake share the same region'' is better expressed in the code. For now, we are just sharing a simple field, but if the number of variables or items to be processed increases, I think this way of writing may be more elegant than declaring constants each time. In that sense, it provides a fairly sophisticated method. If I tried to do the same thing in Java...I don't think I would want to do it that much.

## This is not good enough

It is true that Kotlin has focused attention on the various problems and inconveniences of Java and has solved many of them, but I do not feel that it is simply better than Java in every respect. However, just like the advantages, the problems and disadvantages of Kotlin listed here are personal opinions, so please use them only as a reference.

## vars and types

Those who come to it from a modern language may be tempted to say that concentrating variable declarations in `var` is an advantage. In fact, not only Kotlin but most languages used today, such as JavaScript and C#, support `var`, and Java also introduced `var` starting in version 10. Some languages, such as Python, do not even need a `var` declaration in the first place. I am not sure whether the appeal comes from making it explicit that something is a variable, or from the fact that some languages treat functions as first-class objects and therefore prefer function-like syntax, but it does feel like a trend.

Looking at this trend alone, it seems that we are simply saying, "We just need to know that a variable is a variable." However, I have doubts as to whether this `var` is really good. Maybe I'm too used to Java and can't accept new things, or maybe I just don't understand the merits of `var`, but I find myself thinking, "Isn't it better to specify the type so I know both the variable and the type?" Another reason I think this is that even among modern programming languages, there are moves to add type declarations to existing languages, such as TypeScript. The same is true of Python, which has allowed type declarations since 3.6. This itself seems to show a shift from "I only need to know that a variable is a variable" to "I should also know the type of the variable." However, the problem is that there is no way to declare variables with explicit types from the beginning, and type annotations end up being added on top of languages that only have `var`. Once you do that, the advantage of `var`, namely its brevity, is largely lost.

For example, a declaration of only `var` in Kotlin would be as follows.

```kotlin
var a = "this is string"
```

Then, specifying the type for `var` will look like this:

```kotlin
var b: String = "this is string"
```

The traditional way of writing Java is as follows. I think the code would be shorter this way, and it would be clear that it is a variable.

```java
String a = "this is string";
```

Also, strictly speaking, it may be natural not to add `var` to things that are not variables, but in Java, types are added to variables, return values, and arguments, whereas in Kotlin, there are different situations in which it is better to add `var` or types to these things, so isn't this one thing more strict than Java? I feel like that. For example:

For function arguments, the type must be specified.

```kotlin
fun getMember(request: ServerRequest): Mono<ServerResponse> =
        ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(repository.findById(request.pathVariable(id).toLong()).map { MemberDto(it.username, it.name) })
            .switchIfEmpty(notFound().build())
```

And the return value of a function can be omitted due to type inference.

```kotlin
fun getMember(request: ServerRequest) =
        ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(repository.findById(request.pathVariable(id).toLong()).map { MemberDto(it.username, it.name) })
            .switchIfEmpty(notFound().build())
```

However, this is only when the function is written using Single Expression. When explicitly writing return, omitting the return value will result in a compile error. For example, in the following cases.

```kotlin
// This is a compile error (the return value becomes Unit)
fun getMember(request: ServerRequest) {
      return ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(repository.findById(request.pathVariable(id).toLong()).map { MemberDto(it.username, it.name) })
            .switchIfEmpty(notFound().build())
}
```

Also, in the case of data class, you need to add `val` or `var` to the field. However, it is not necessary when declaring a general class.

```kotlin
// data class requires val or var
data class MemberDto(val username: String, val name: String)

// Not needed for a regular class
class MemberEntity(username: String, name: String)
```

Usually, a compilation error occurs, so until you get used to it, you have no choice but to review the code according to Kotlin's rules, but if you are just getting started with Java, this part can be quite confusing. (Maybe it's just me...)

## Dependencies

If you want to use Kotlin in your project, you will need to add and use the standard library. However, if you only add `kotlin-stdlib` here, you will not be able to use some features added after Java 1.7 (such as AutoCloseable). So, if you want to use the features of Java 1.7 or later, you need to add `kotlin-stdlib-jdk7` or `kotlin-stdlib-jdk8` to your dependencies.

Personally, I thought it was due to something like the lawsuit between Oracle and Google, and they purposely created their own packages to avoid copyright, but it seems that the reason is actually to support the Module system introduced in Java 9. So it seems that `kotlin-stdlib-jre7` was replaced by `kotlin-stdlib-jdk7` and `kotlin-stdlib-jre8` was replaced by `kotlin-stdlib-jdk8`.

Anyway, in order to use these standard libraries, you only need to register them once using a package manager that manages dependencies such as Maven or Gradle, so it may not be that troublesome, but since there is `kotlin-stdlib-jre8` for example, I think that it may be a disadvantage because you will have to spend time figuring out which one to choose and which one is necessary at first. For example, even without `kotlin-stdlib-jdk7`, you can use JDK 1.7 features other than AutoCloseable, but it seems quite troublesome to check whether to add dependencies depending on whether AutoCloseable is used in the project you are about to create or an existing project.

And the fact that there is a separate standard library compatible with JDK7 and JDK8 means that there is a possibility that new standard libraries such as JDK 17 will be added when Java is updated in the future. 7 (1.7) and 17 are often confused...Also, there are various dependencies packages other than JDK (such as `kotlin-reflect`), so depending on the project configuration, you may need to be very careful when introducing Kotlin. In a sense, I think this is the reason why Kotlin is not widely used for purposes other than creating Android apps, even though it has ample potential as a post-Java tool.

## Finally

I briefly divided it into Pros and Cons and wrote about what I felt about Kotlin. Actually, I haven't tried it in a full-scale project yet, so I haven't even touched on features that look good like [Type aliases](https://kotlinlang.org/docs/reference/type-aliases.html) and [inline class](https://kotlinlang.org/docs/reference/inline-classes.html). However, I feel that the more I use it, the more attractive it becomes. So, in my personal opinion, if you are already using Java, you should seriously consider migrating to Kotlin. It's easy for Java programmers to get used to, it's more productive, and it's also 100% compatible with Java. Considering the level of maturity of the language and the fact that it's compatible with the JVM and can also be linked with Native and JavaScript, I think it's the perfect language for Post Java. (Sorry for other JVM languages...)

In that sense, why don't you all start using Kotlin now?
