---
title: "I wrote it in Kotlin ~Part 2~"
date: 2021-06-28
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
---

Following the previous post, I wrote about a number of Kotlin patterns again this time. Kotlin provides many functions in its standard library and language features, so simply using them well can greatly improve productivity and code quality. This time as well, I want to show how Java-like patterns can be implemented more efficiently in Kotlin.

Of course, in Kotlin, you can basically write code that works in Java style without any problems, but in many cases it is easier and shorter to write code that is unique to Kotlin, and you can save a lot of effort (and in most cases, the standard library implementation is higher quality than the code you wrote...), so I think it is well worth making this kind of effort.

So, this time I would like to introduce some Kotlin tricks that I researched.

## Create sequential data

We often create and use test data for unit tests, etc. I think there are various types of data that are needed in such cases, but there may be times when you want to create something that looks like multiple records numbered and arranged in order. For example, if you want to create data such as Data01, Data02, Data03...

In this case, I think it's common to create data in a loop and summarize it in a List. For example, let's say we have the following example.

```kotlin
// Create test data
fun createTestDatas(): List<String> {
    // Test data list
    val testDatas = mutableListOf<String>()
    
    // Add 10 records
    for (i in 0 until 10) {
        testDatas.add("Test$i")
    }

    // Convert to read-only and return
    return testDatas.toList()
}
```

However, this is rather similar to the Java method, so I would like to start by thinking about how to do the same thing with Kotlin-like code based on this.

## repeat

The first possible method is to simplify the loop. If we want to create a list of size 10, we will loop 10 times, so we will use the appropriate function. For example, there is [repeat](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/repeat.html). With `repeat`, the index is passed as a parameter within the scope, so you can easily

```kotlin
fun createTestDatas(): List<String> {
    val testDatas = mutableListOf<String>()

    // Repeat 10 times
    repeat(10) { 
        testDatas.add("Test$i")
    }

    return testDatas.toList()
}
```

The next thing I would like to consider is changing `MutableList` to `Immutable`. Although it may be fine as data to use for testing, it is not a good choice to leave data that does not need to be changed in `Mutable` as it is. Therefore, I would like to use a method that allows data creation to be done using `List` from the beginning.

There are two options here: you can declare `List` with a specified size from the beginning, or you can specify the loop range, that is, [Range](https://kotlinlang.org/docs/ranges.html#range).

## List

First, let's look at how to create `List` with a specified size. When creating an instance, you can easily create elements of the specified size by passing the size and initializer for the element as arguments. For example, the code introduced above can be changed as follows using `List`.

```kotlin
fun createTestDatasByList(): List<String> =
    List(10) { "Test$it" }
```

This method is actually not fundamentally different from the method introduced earlier. The implementation is as follows, so you can see that it can be used as syntax sugar.

```kotlin
@SinceKotlin("1.1")
@kotlin.internal.InlineOnly
public inline fun <T> List(size: Int, init: (index: Int) -> T): List<T> = MutableList(size, init)

@SinceKotlin("1.1")
@kotlin.internal.InlineOnly
public inline fun <T> MutableList(size: Int, init: (index: Int) -> T): MutableList<T> {
    val list = ArrayList<T>(size)
    repeat(size) { index -> list.add(init(index)) }
    return list
}
```

When using `List`, it is also convenient to be able to configure `step` depending on what function is passed as `itnit`. For example, you can do something like:

```kotlin
List(5) { "Test${ it * 2 }" }
// [Test0, Test2, Test4, Test6, Test8]

List(5) { (it * 2).let { index -> "$index is even" } }
// [0 is even, 2 is even, 4 is even, 6 is even, 8 is even]
```

However, the resulting instance of `List` is `MutableList`, so if you want to make the generated data read-only, you will need to convert it with something like `toList()`.

## Range

Now let's try another method again. In Kotlin, you can easily create a `Range` object by simply specifying a range of numbers. When using `Range`, the above code can be changed as follows.

```kotlin
// Create test data using Range
fun createTestDatasByRange(): List<String> =
    (0..10).map { "Test$it" }
```

Unlike `List`, `Range` has [IntRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-int-range), [LongRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-long-range), [CharRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-char-range), etc., and it is also good that you can easily arrange it by adjusting the numbers and letters of the argument.

Also, in general, `Range` seems to have better performance than `List`. When I benchmarked the code below, I was able to confirm that `Range` is usually about twice as fast as `List`.

```kotlin
import kotlin.system.measureTimeMillis

data class Person(val name: String, val Num: Int)

fun main() {
    benchmark { list() }
    benchmark { range() }
}

fun benchmark(function: () -> Unit) =
    println(measureTimeMillis { function() })
    
fun list() =
    List(200000) { Person("person$it", it) }
    
fun range(): List<Person> =
    (0..200000).map { Person("person$it", it) }
```

One thing to keep in mind is that in the case of `Range`, the value basically increases by 1, so you cannot use the conditions of `step` like `for` and `List`. So you need to think about which one to use depending on the situation.

## Check

You may need to check the value of a parameter, such as during validation. In Kotlin, `Nullable` objects and non-`Nullable` objects are separated, so unlike Java, if `null` is passed as an argument, a compile error will occur, but depending on the business logic, you may need to check other things, so you have no choice but to write your own checks in code.

First, if you follow the familiar Java method, you can write code like the following. This example includes checking the function's arguments and its return value.

```kotlin
fun doSomething(parameter: String): String {
    if (parameter.isBlank()) {
        throw IllegalArgumentException("The string is empty")
    }

    val result = someRepository.find(parameter)

    if (result == null) {
        throw IllegalStateException("The result is null")
    }
    return result
}
```

Let's take a look at an example using a slightly different language. In the case of Swift, which is said to be very similar to Kotlin, it seems common to use [Guard Statement](https://docs.swift.org/swift-book/ReferenceManual/Statements.html#grammar_if-statement) here. It's nice to have an expression for checking, which separates the business logic and checking. I haven't used Swift much, so it may not be a good example, but the code you can imagine is as follows.

```swift
func doSomething(parameter: String) throws -> String {
    guard !parameter.isEmpty else {
        throw ValidationError.invalidArgument
    }

    guard let result = someRepository.find(parameter) else {
        throw ValidationError.notFound
    }
    return result
}
```

Similarly, in Kotlin, if the checking expression and business logic can be separated, the meaning of the code should become clearer. How can we achieve that with Kotlin? For example, consider the following.

```kotlin
fun doSomething(parameter: String?): String {
    val checkedParameter = requireNotNull(parameter) {
        "The string is null"
    }

    val result = someRepository.find(checkedParameter)

    return checkNotNull(result) {
        "The result is null"
    }
}
```

[requireNotNull](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/require-not-null.html) throws `IllegalArgumentException` if the passed argument is `null`, otherwise returns the argument as `non-null` type. This is convenient because not only does it clearly check for `null`, but there is no need to check further. It's also nice to be able to specify a message when `IllegalArgumentException` occurs as `lazy message`.

[checkNotNull](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/check-not-null.html) is functionally the same as `requireNotNull`, but the exception thrown in `null` is `IllegalStateException`. So, you can use these two separately depending on your needs.

Another option that can be used is [require](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/require.html). By passing a conditional expression, you can do things other than checking `null`. Therefore, you can also check the range of `Int` type data as shown in the code below.

```kotlin
fun doSomething(parameter: Int) {
    require(parameter > 100) {
        "$parameter is too large"
    }

    // ...
}
```

There is also another method using [Elvis operator](https://kotlinlang.org/docs/null-safety.html#elvis-operator). In this case, instead of just throwing an exception as in the case of `null`, you can write alternative processing, so there is room for various uses. For example, you can do something like the following.

```kotlin
fun doSomething(parameter: String?): String {
    val checkedParameter = parameter ?: "default"

    val result = someRepository.find(checkedParameter)

    return result ?: throw CustomException("The result is null")
}
```

## Split List

If you want to extract data that matches a certain condition from a List, you can do so by using an operation like `filter`. But what if there are two conditions? More precisely, when you want to separate one list into two lists: elements that match the specified conditions and elements that do not.

In this case, I think there is a way to loop twice as shown below, but this is not very efficient.

```kotlin
val origin = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// Extract odd numbers
val odd = origin.filter { it % 2 != 0 }
// Extract even numbers
val even = origin.filter { it % 2 == 0 }
```

One way to reduce loops is to branch within the loop on a list that has been declared in advance. For example:

```kotlin
val origin = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// Declare the odd and even lists in advance
val odd = mutableListOf<Int>()
val even = mutableListOf<Int>()

// Loop processing
origin.forEach {
    if (it % 2 != 0) {
        odd.add(it) // Add to the odd list
    } else {
        even.add(it) // Add to the even list
    }
}
```

Fortunately, Kotlin's standard library provides a method that is perfect for this situation. The operation is [partition](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/partition.html). Using this opreation, the elements of the original list will be divided into those that match the condition and those that do not.

Also, the return value of `partition` is `Pair<List<T>, List<T>>`, so combining it with [destructuring-declaration](https://kotlinlang.org/docs/destructuring-declarations.html) makes the code much shorter. The actual code looks like this, and it's pretty smart.

```kotlin
val origin = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
val (odd, even) = origin.partition { it % 2 != 0 } // Split into matching and non-matching items
```

## lastly

Kotlin is convenient, but since the language itself provides many conveniences (features), I feel that compared to other languages, the quality of the code depends more on whether or not you can use the API correctly. Furthermore, updates are quick and features are added one after another, so it's important to catch up.

But I do feel like the more I try to improve what Kotlin can do one by one, the more things I can do. It's great to be able to use a language that becomes more powerful the more you study it. So, my series on writing with Kotlin will continue.

See you soon!
