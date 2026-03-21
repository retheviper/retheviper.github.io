---
title: "Kotlin as seen by a Java programmer ~Part 2~"
date: 2021-02-28
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---
This time, I decided to change jobs, and the language used at work will also change from Java to Kotlin. I have personally created a simple Spring WebFlux project in Kotlin, and although it is said that it is not difficult for Java programmers to migrate to Kotlin, I think it is still quite a challenge to change the language used for work. So, up until now I have mainly posted posts about Java, but from now on I would like to increase the number of posts about Kotlin.

As is well known, Kotlin is fully compatible with Java. It is a JVM language, and when compiled it becomes bytecode just like Java. However, I feel that writing code with a Java mindset defeats the purpose of moving to a different language like Kotlin. The more you use Kotlin, the more you realize that it is designed around a fundamentally different way of thinking from Java. At first, I thought the main goal was simply to reduce Java's verbosity, but after studying it seriously I realized there was much more to it.

This post was created based on the content of the [Kotlin for Java Developers](https://www.coursera.org/learn/kotlin-for-java-developers) lecture on [Coursera](https://www.coursera.org).

## Reducing redundancy

Java is still a good language and is still used in a wide range of fields, even though many languages have been released and are used at the enterprise level. Java remains a popular language, as evidenced by the [TIOBE index](https://www.tiobe.com/tiobe-index/), JetBrains' [The State of Developer Ecosystem](https://www.jetbrains.com/lp/devecosystem-2020), and Stackoverflow's [Developer Survey](https://insights.stackoverflow.com/survey/2020).

However, even though Java is still a popular language, this does not mean that it is overwhelmingly superior or easier to use than other languages. I think this is the case with any language, but one of the problems that is often pointed out with Java is that it is ``too verbose''. Even though there are many libraries and you can use excellent build tools like Maven and Gradle, the language specifications remain the same. In order to solve this problem, Java 9 introduces many features that are influenced by other languages ​​(for example, [instanceof pattern matching](https://blogs.oracle.com/javamagazine/pattern-matching-for-instanceof-in-java-14) and [record](https://blogs.oracle.com/javamagazine/records-come-to-java), etc.), but this is more like ``introducing the features of a different language with specifications tailored to Java'' rather than changing the design concept of the language itself, so it cannot be called a fundamental change. Therefore, the redundant code that has been written so far will remain and will continue to be used.

## Shorter code

Reducing verbosity can be simply defined as ``achieving the same result with shorter code.'' From that perspective, Kotlin's language specifications show traces of efforts to reduce Java's verbosity. For example, let's say we have the following code.

```java
public void updateWeather(int degrees) {
    String description;
    Color color;
    if (degrees < 10) {
        description = "cold";
        color = BLUE;
    } else if (degrees < 25) {
        description = "mild";
        color = ORANGE;
    } else {
        description = "hot";
        color = RED;
    }
}
```

Rewriting this in Kotlin would look like this:

```kotlin
fun updateWeather(degrees: Int) {
    val (description, color) =
        if (degrees < 10) {
            Pair("cold", BLUE)
        } else if (degrees < 25) {
            Pair("mild", ORANGE)
        } else {
            Pair("hot", RED)
        }
}
```

First, you can see that the two variables can be made shorter by expressing an object whose return value is `Pair`. And this code can also be modified into a shorter form using the `when` clause. The result is below.

```kotlin
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 10 -> Pair("cold", BLUE)
        degrees < 25 -> Pair("mild", ORANGE)
        else -> Pair("hot", RED)
    }
}
```

Furthermore, `Pair` can also be more easily expressed using `to`. Then the code will look like this:

```kotlin
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 10 -> "cold" to BLUE
        degrees < 25 -> "mild" to ORANGE
        else -> "hot" to RED
    }
}
```

You can see that the code is much simpler and clearer than the initial Java code. Even if you're using another language, you'll instantly understand what you're doing, and your code will be shorter and more efficient. I think this is where Kotlin focuses on reducing Java's redundancy and waste.

## Easy to write code

At first, when I briefly looked at Kotlin's syntax, my only impression was that `switch` had changed to `when` here, and there was no need to write `case`. However, if you look closely, you can see other differences from Java. For example, what you can read here is the following code from earlier.

- `when` clause can be used as an expression
- The condition of the `when` clause can only be targeted within the conditional expression.
- Define multiple values as return values in an expression and use them as
- Two objects can be combined into `Pair` with `to`In addition, Kotlin's `when` clause can be used as follows compared to Java's `switch`. It's easier to compare objects. For example, with the code below, you can easily compare two objects.

```kotlin
fun mix(c1: Color, c2: Color) =
    when (setOf(c1, c2)) {
        setOf(RED, YELLOW) -> ORANGE
        setOf(YELLOW, BLUE) -> GREEN
        setOf(BLUE, VIOLET) -> INDIGO
        else -> throw Exception("Dirty Color")
    }
```

If you were to write this in Java code, it would probably look something like this: Personally, I think that a lot of `else if` is not very readable code, nor is it clean from a writing perspective.

```java
private Color mix(Color c1, Color c2) {
    if (c1 == Color.RED && c2 == Color.YELLOW) {
        return Color.ORANGE;
    } else if (c1 == Color.YELLOW && c2 == Color.BLUE) {
        return Color.GREEN;
    } else if (c1 == Color.BLUE && c2 == Color.VIOLET) {
        return Color.INDIGO;
    } else {
        throw new RuntimeException("Dirty Color");
    }
}
```

What you can see here is that even if you do the same thing in Kotlin as in Java, the code is not only shorter but also easier to code. Of course, you might be able to do something similar in Java by creating another method, creating a `Comparable` object, or implementing a `Comparator` class. However, I'm not sure if I want to go that far.

Of course, starting with Java 12, you can also write code with a feel similar to Kotlin's `when`. It's also good that it can be used as an expression, that multiple conditions can be specified, and that it can be written in the same way as `Lambda`.

```java
var result = switch (month) {
    case JANUARY, JUNE, JULY -> 3;
    case FEBRUARY, SEPTEMBER, OCTOBER, NOVEMBER, DECEMBER -> 1;
    case MARCH, MAY, APRIL, AUGUST -> {
        int monthLength = month.toString().length();
        yield monthLength * 4;
    }
    default -> 0;
};
```

Looking at these changes, it may seem that Kotlin's appeal has been halved in terms of ``reducing redundancy,'' since Java is also introducing new features one after another with version upgrades. However, there is one more important point in Kotlin. It's the extensibility of the language itself.

## Being able to expand

I mentioned the extensibility of the language itself, but to put it simply, I'm referring to the extension function that I introduced before, namely `extension`. This is introduced as a major part of the Kotlin specifications. If you use this often, you will not only be able to add a method to a class without inheriting it, but by combining it with `infix` you can use it as if it were reserved.

In fact, in the coding problem of [Kotlin for Java Developers](https://www.coursera.org/learn/kotlin-for-java-developers), it is said that the results are checked using the following extension function written in `infix`.

```kotlin
infix fun <T> T.eq(other: T) {
    if (this == other) println("OK")
    else println("Error: $this != $other")
}
```

Using this `infix`, you will be able to write code like the following.

```kotlin
"ABC" eq "ABC"
```

Having these features is convenient for the user, but I think that as the language itself is updated, more convenient features will be added and it will also become cheaper. For example, the `to` that creates the `Pair` object mentioned earlier is created as a `infix` function like this. Convenient features like this will continue to be added and it will be easier to add them, which is certainly a good thing in terms of reducing development costs.

## Must be convenient enough

I think the features of reducing verbosity and being an extensible language are probably very effective features for JetBrains, the company that makes Kotlin. If you look at the Kotlin standard library, there are already many useful functions. For example, a simple loop could do something like this:

```kotlin
val list = listOf("A", "B", "C")

// Regular `for` loop
for (element in list) {
    println(element)
}

// `for` loop with an index
for ((index, element) in list.withIndex()) {
    println("$index: $element")
}

// `for` loop using only an index
for (index in list.indices) {
    println(index)
}
```

I briefly talked about the performance of Java's for statements in a previous post, where I concluded from benchmarking at JMH that Java uses traditional for statements when indexing is required, and that it is generally better to use extended for statements when indexes are not needed. However, if a language already provides a convenient method like this, it is also attractive because it allows you to use a convenient method without worrying about performance.

Another good thing is that if you need an index in forEach, you can use `forEachIndexed`. For example, you can write it like this:

```kotlin
// Regular `forEach` loop
list.forEach(::println)

// `forEach` loop with an index
list.forEachIndexed { index, element -> println("$index: $element") }
```

The fact that indexes can be easily retrieved is also good because if you want to retrieve all the indexes of the object being looped over, you don't have to specify a possible magic number like `0`. In Java, there are many cases where you declare it as a static final field or manage it as a separate constant...

In addition, there are functions that are provided in advance that allow you to easily check characters without regular expressions (such as `Char.isLetter()` and `Char.isDigit()`), elements can be inserted into `Map` with `Pair`, and operations like the Stream API can be performed immediately from iterable objects.I think it is attractive that there is certainly no need to worry about it compared to Java. Well, I think some people may see this as a disadvantage...

## Finally

I have talked about various features and benefits of Kotlin, but since I have not actually used Kotlin in my work yet, I think my knowledge is still only superficial. However, I think you can get a feel for the charm of Kotlin just from what I have introduced here.The language itself is attractive, but there are many other benefits to using Kotlin. For example, it is developed by JetBrains, so it is compatible with Intellij. Since it is a JVM language and is compatible with Java, it can absorb the development of Java as it is. It also means that it can be compiled to Native or JavaScript. Although other languages ​​have their own attractive points, I believe that if you are a Java programmer, it is worth trying Kotlin once. If you haven't touched Kotlin yet, please give it a try.

See you soon!
