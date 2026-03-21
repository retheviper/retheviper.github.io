---
title: "How to Write Kotlin Idiomatically"
date: 2022-12-30
categories:
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---

It has been about two years since I moved from Java to Kotlin. Even now, I still feel that Kotlin has an infinite number of things it can do, and that there is no end to the new things I can discover. The language itself is evolving quickly, and new features keep being added, so I expect that this stream of discoveries will continue for a while.

One thing I keep thinking about is that IntelliJ can automatically convert Java code for me, and even if I reuse Java-style code as-is, it rarely becomes a problem. Still, there is also a desire to write code that feels uniquely Kotlin. In other words, I end up wanting to write in a way that "feels like Kotlin."

## What Counts as Kotlin-like Writing?

What exactly does "writing like Kotlin" mean? First, we need a definition. There are many ways to look at it, but I basically think it means "making the most of Kotlin's specification and features." In other words, actively using the standard library APIs, scope functions, extensions, and so on. I believe that doing so shortens the time needed to write code and improves efficiency.

That said, simply describing the idea in words leaves it a little vague, so it is better to show examples with code. Suppose you need to implement the following function:

```kotlin
fun numbers(): List<String> {
    // TODO
}
```

Let us say the goal is to "convert the numbers from 0 to 10 into strings and return them as a list." In that case, one possible implementation would look like this:

```kotlin
fun numbers(): List<String> {
    val list = mutableListOf<String>()
    var i = 0
    while (i < 10) {
        i++
        list.add(i.toString())
    }
    return list.toList()
}
```

If we use [kotlin.collections.map()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/map.html), we can rewrite the same code like this:

```kotlin
fun numbers(): List<String> =
    (0..10).map { it.toString() }
```

When functions like `map()` first appeared, some people felt they were "hard to read" because their meaning was not immediately intuitive. Comparing the two snippets above, there are probably cases where a `while` loop feels easier to understand. Today, `map()` is used across many programming languages and its behavior is well understood, but for people who are used to classic code, "looping through something" may still feel clearer.

## Readability

But can we really say that longer code is easier to read even when the processing becomes more complex? Let us look at another example. Suppose we need to implement a function that "multiplies all numbers in a list and returns the result." A straightforward implementation might look like this:

```kotlin
fun multiply(numbers: List<Int>): Int {
    if (numbers.isEmpty()) {
        throw IllegalAccessException("List is empty")
    }
    var acc = numbers[0]
    if (numbers.size == 1) return acc
    for ((index, value) in numbers.withIndex()) {
        if (index != 0) {
            acc *= value
        }
    }
    return acc
}
```

This code accomplishes the goal, but what about readability? Unless you read not only the function signature but the whole body, it is not obvious what is happening.

If you read the processing step by step, the function says: "throw an error if the list is empty," "return the only element if there is just one," "otherwise continue," and "take the first element, then loop through the list and multiply everything except the first element." All of that is hidden in the implementation.

If we rewrite the same logic using an extension and functions from `kotlin.collections`, the code changes dramatically:

```kotlin
fun List<Int>.multiply(): Int =
    reduce { acc, e -> acc * e }
```

As with `map()`, people may not immediately know what `reduce()` does. But once you understand that it "reduces elements into a single value" and that the reduction here is multiplication, the intent becomes much easier to grasp. Compared with the earlier version, which required understanding several separate pieces of processing, I think this version is better in terms of readability.

Defining it as an extension on `List` is also nice because it can be used like a built-in method, and IDE completion should show it automatically.

## In Terms of Effort

Writing code takes effort. So even when implementing the same function, it is normal to separate reusable parts and avoid rewriting everything from scratch. That is why so many libraries and frameworks exist.

Understanding a language's features and using them well is fundamentally no different from using a library or framework. As with the earlier example, if we had to write the logic for "combining list elements into one" every time, it would take a lot of time.

Let me give another example. Suppose we have the following data:

```kotlin
data class Data(val id: Int, val version: Int)

val list = listOf(
    Data(1, 1),
    Data(1, 2),
    Data(2, 1),
    Data(2, 2),
    Data(2, 3)
)
```

If you want to keep history in a database, you may store version numbers or branch numbers and insert multiple records with the same ID. In that case, you may also want to work only with the latest data. In the example above, that would mean fetching only `Data(1, 2)` and `Data(2, 3)`.

It would be nice if you could retrieve only the latest records through the query itself, but responses from external APIs are not always filtered that way. So let us write logic that filters records by the maximum version. One possible implementation would be:

```kotlin
fun maxVersions(list: List<Data>) {
    val result = mutableListOf<Data>()
    for ((index, value) in list.withIndex()) {
        if (index == 0) {
            result.add(value)
        } else {
            val last = result[result.size - 1]
            if (value.id == last.id) {
                if (value.version > last.version) {
                    result.remove(last)
                    result.add(value)
                }
            } else {
                result.add(value)
            }
        }
    }
    return result.toList()
}
```

Leaving readability aside for a moment, how much effort does this kind of code take? Once you are used to it, it might be easy, but from the perspective of someone writing it for the first time, it seems like a lot of work. If you include verifying that the function behaves correctly, the effort increases even more. Even when you are familiar with it, it is not obvious that you can always write this kind of code perfectly.

Now compare that with an implementation using the standard library:

```kotlin
val maxVersions = list.groupBy { it.id }.map { it.value.maxBy { it.version } }
```

`groupBy()`, `map()`, and `maxBy()` are all useful on their own: one creates a `Map` of values grouped into lists, one transforms elements into another form, and one finds the maximum element in a list. Because these helpers are useful in many places, learning to combine them lets you write more complex processing much more easily. That is why understanding the standard library can improve efficiency so much.

## Cautions

So far I have argued that making use of Kotlin's features improves readability and reduces effort. But, of course, every advantage comes with a downside.

Naturally, any feature can backfire if you use it too aggressively just because it is available. For example, consider the case where you have two data classes and need to map `Data` to `Request`.

```kotlin
data class Data(
    val id: Int?,
    val version: Int?
)

data class Request(
    val id: Int,
    val version: Int
)
```

`Data` contains nullable fields, while `Request` does not. In some systems, fields may be nullable for implementation reasons even if they are never null in the business logic. In that case, how should the mapping be done? One possible implementation is:

```kotlin
data class Request(
    val id: Int,
    val version: Int
) {
    companion object {
        fun from(data: Data): Request {
            return Request(
                id = requireNotNull(data.id) {
                    "${Data::id.name} should not be null."
                },
                version = requireNotNull(data.version) {
                    "${Data::version.name} should not be null."
                }
            )
        }
    }
}
```

Here, a companion object is used to create the `Request` instance while mapping the data. `requireNotNull()` is used for validation, and an error message is produced if the value is null.

So far so good, but the problem is the error message. It tells us which parameter was null by pulling the field name from `Data` via reflection and embedding it as a string. Even if this is intentional when an error occurs, was it really worth using reflection, which is slower?

When you use language features like this, you need to judge the right place for them carefully. Functions such as `map()`, `reduce()`, and `groupBy()` are excellent helpers that make simple implementations easy, but if you look at how they work internally, each one performs its own loop and creates a new collection. That means chaining several of them can affect performance. They can also hurt readability or pack too many separate responsibilities into a single function.

So I think it is important to understand that using these features out of vague habit or inertia carries risks, and that you should always think carefully about what kind of implementation you actually want.

## In Closing

I have already written a few posts titled something like "trying Kotlin this way," showing different ways to write Kotlin code. This article is not that far removed from those. In a sense, what I described here is basic knowledge every programmer should already have, so it may feel a little old-fashioned.

The reason I still wrote it as an article is that I believe the basics matter most. As you gain experience, you develop better techniques, but you can also pick up bad habits that are difficult to fix later. I think it is worth stepping back to the basics now and then, comparing them with how you write today, as a way to check yourself. The end of the year is a good time for that too.

I am a little late, but this is the final post of 2022. I hope that next year will be one of growth for both me and everyone who visits here.

See you again soon.
