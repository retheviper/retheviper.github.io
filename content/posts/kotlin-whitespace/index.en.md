---
title: "A peek into Kotlin's String implementation ~ whitespace edition ~"
date: 2021-05-08
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - string
---
In Kotlin (JVM), the result of compilation is JVM bytecode. That's why libraries written in Java can be used as is in Kotlin. This is the same for Kotli libraries, so if you look at the standard library, there are quite a few that depend on Java features.

However, the fact that Kotlin becomes JVM bytecode when compiled does not simply mean that Kotlin is "Java written in a different way." This is partly because Kotlin has different language specifications from Java, but it is designed with the assumption that it will be compiled into JavaScript and native code as well as the JVM, so the standard library Standard library has different implementations depending on the platform. And even if it is a JVM, it does not use the Java API as is.

This structure of Kotlin becomes clear when you look at the internal source code. Some methods and classes in the standard library use the keywords `expect` and `actual`, which are similar to inheritance in Java. In Java, the method defined in `interface` is implemented and used in the class that inherits it in `override`. Similarly, in Kotlin, functions defined as `expect` are implemented in `actual` according to the platform.

Also, although Kotlin's standard library may seem similar to Java's at first glance, it is actually different in some cases. This is because the code implemented by `actual` is written in accordance with Kotlin. Therefore, it may be dangerous to assume that the Kotlin standard library is the same as Java.

This time, I would like to take a look at the functions related to string whitespace, focusing on the standard library source code.

## Determination of whitespace

One way to determine whether a string is meaningful (valid) data is to determine whether the string is just blank space. In other words, it checks to see if there is any data or just whitespace.

Judgment in such cases can be easily done using the Kotlin standard library. Kotlin basically provides the following two methods for String.

- `isEmpty()`
- `isBlank()`

Methods with the same names exist in Java 11 and later, so at first glance it may seem fine as is. However, in Kotlin, these methods are first called from `kotlin.text.Strings`. Since the Java API is not used as is, we can assume that the processing may be different.

In the former case, it simply checks whether the string contains no data at all. If you look at the actual source code, you can see that it only checks the length of the string.

By the way, in Java, `String` inherits from `CharSequence`, but in Kotlin, the inheritance relationship is the same even though the libraries are different. So, in Kotlin, even though it is a member of `String`, it is written as a `CharSequence` function.

```kotlin
public inline fun CharSequence.isEmpty(): Boolean = length == 0
```

In the latter case, it is determined whether the string includes whitespace. If you look at the code below, it should be clear what it is doing.

```kotlin
public actual fun CharSequence.isBlank(): Boolean = length == 0 || indices.all { this[it].isWhitespace() }
```

The implementation of `isWhitespace()` called by `isBlank()` is as follows.

```kotlin
public actual fun Char.isWhitespace(): Boolean = Character.isWhitespace(this) || Character.isSpaceChar(this)
```

Kotlin's `Char.isWhitespace()` will ultimately be determined using `Character.isWhitespace()` and `Character.isSpaceChar()`. This is a Java API that determines whether the former case applies to Unicode whitespace, and the latter case applies to Unicode spaces (such as line feed codes). As you can see here, unless it is a special case, it is better to use `isEmpty()` when checking strings.

## Delete whitespace

To simply determine whether a string contains meaningful data, it is best to use `isEmpty()` as mentioned above, but there are cases where the string contains not only whitespace but also meaningful data. In this case, you will want to remove the whitespace before and after.

In Java, there are `trim()` and `strip()` as methods to erase whitespace before and after a string. The former is old-fashioned and cannot detect full-width whitespace and has performance issues, so it is recommended to use the latter from Java 11 onwards.

However, in the case of Kotlin, the situation is slightly different. In Kotlin, you basically only use `trim()`. First, let's look at the implementation of `trim()`.

```kotlin
public inline fun String.trim(): String = (this as CharSequence).trim().toString()
```

First, as `String`, we will upcast to `CharSequence` and call that `trim()`. After that, simply return it with `toString()`.

Next, let's look at `trim()` on the `CharSequence` side, which is called by `String`.

```kotlin
public fun CharSequence.trim(): CharSequence = trim(Char::isWhitespace)
```

Here you can see that we are passing `isWhitespace()` as a method reference to the other overloaded `trim()`. Since `Boolean` is the return value, we can infer that the argument is `Predicate`. Next, check the `trim(predicate)` that is being called here. The code is as follows.

```kotlin
public inline fun CharSequence.trim(predicate: (Char) -> Boolean): CharSequence {
    var startIndex = 0
    var endIndex = length - 1
    var startFound = false

    while (startIndex <= endIndex) {
        val index = if (!startFound) startIndex else endIndex
        val match = predicate(this[index])

        if (!startFound) {
            if (!match)
                startFound = true
            else
                startIndex += 1
        } else {
            if (!match)
                break
            else
                endIndex -= 1
        }
    }

    return subSequence(startIndex, endIndex + 1)
}
```

At this point, the actual processing has finally begun. It searches for whitespace from the left (start) to the right while looping through `CharSequence`, and when it finds a non-whitespace character for the first time, it repeats while looping from the right (end) to the left. It's a surprisingly simple but efficient process.

The decision criterion for this process is `isWhitespace()`, but as we confirmed earlier, this ultimately calls the Java API, so we can infer that `trim()` is sufficient to remove whitespace and spaces defined in Unicode. Therefore, unlike Java, there seems to be no need to use `strip()`.

Also, `trim()` removes whitespace before and after a string, but in some cases you may want to use it separately for only the front and only the rear. At that time, you can do the following:

```kotlin
val string = "  string  "

// Trim left only
println(string.trimStart()) // "string  "

// Trim right only
println(string.trimEnd()) // "  string"
```

These methods can also pass `Predicate` as an argument, so you can use that if you need to write other conditions yourself.

In addition, the following methods are provided if you want to delete specific characters (prefix, suffix) before and after the text that are not whitespace.

```kotlin
val string = "--hello--"

// Remove prefix only
println(string.removePrefix("--")) // "hello--"

// Remove suffix only
println(string.removeSuffix("--")) // "--hello"

// Remove both prefix and suffix
println(string.removeSurrounding("--")) // "hello"
```

## Remove line breaks

If there are line breaks before and after the string, `trim()` is sufficient, but there may be cases where the string contains line breaks and you want to change it. For example, if you want to output JSON to the log in one line, or if you want to combine the following Multiline String into one line.

```kotlin
val string = """
    Hello
    World
"""
```

Intellij automatically adds `trimIndent()`, but this is only related to indentation and does not trim line breaks inside. In this case, there is no corresponding method in either Kotlin or Java, so you have no choice but to write the process yourself. For example, you could use code like the following:

```kotlin
fun String.stripLine() = replace(System.lineSeparator(), " ")
```

However, [Text Block](https://openjdk.java.net/jeps/355) has been introduced in Java since version 13, so we may be able to expect methods like the above to be added to the Java API in the future.

## Finally

We first talked about `expect` and `actual`, but these keywords are the most important concepts in [Kotlin Multiplatform](https://kotlinlang.org/docs/multiplatform.html). Since the purpose is to make it possible to share code written in Kotlin across various platforms, it is natural to have this structure. So, I think this is a keyword that you need to understand for the time being, not only for Kotlin/JVM but also for those who want to try something else. It's a little unique, but the substance is simple, so it's easy to understand.

Also, Kotlin's Strings are briefly explained in [Video on JetBrains official YouTube channel](https://youtu.be/n4WBip822A8), so if you are developing with Kotlin, you may want to refer to it at least once.

In addition, I said that there is no need to use `strip()`, but in fact, even in the latest version of Kotlin, 1.5.0, `strip()` has been changed to `deprecated`, and there is a comment like the following, so it is better not to use it until it is officially supported in the next version.

> 'strip(): String!' is deprecated. This member is not fully supported by Kotlin compiler, so it may be absent or have different signature in next major version

As you can see in this case, I think there are some aspects in which Kotlin cannot be said to be 100% compatible with Java. So if you're moving from Java to Kotlin (whether it's the actual code or your own skills), you may need to read the Standard Library description carefully.

See you soon!
