---
title: "I wrote it in Kotlin ~Part 3~"
date: 2021-09-18
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---
Speaking from the perspective of someone who has migrated from Java to Kotlin, Kotlin provides a variety of functions in the standard library alone, so it can be said that productivity is considerably higher than Java, but on the other hand, there may be times when you don't know how to use the functions effectively, or you don't know how to write a process that is ``Kotlin-like.'' So this is already my third post, and this time too I'll try writing a lot of code in Kotlin and share some that look good.

## Swap List elements

There are various ways to change the order of the elements in a List, including sorting, but there may be times when you want to swap two elements (exchanging indexes). I tried to think of extension functions that can be used in situations like this.

## If you know the index

If you know the index of the element you want to swap, just swap the index. Swapping the index here is no different from swapping the values ​​of two variables. Traditionally, there are two ways to exchange variable values:

```kotlin
var a = 10
var b = 20
var c = a

a = b
b = c
```

A slightly more Kotlin-like method is to use `also`. That way, the process required would be much simpler:

```kotlin
var a = 10
var b = 20

a = b.also { b = a }
```

Similarly, if you write the process of swapping List elements using an extension function, it would look like this:

```kotlin
fun <T> List<T>.swapByIndex(indexFrom: Int, indexTo: Int): List<T> =
    toMutableList().apply {
        this[indexFrom] = this[indexTo].also { this[indexTo] = this[indexFrom] }
    }.toList()
```

## If you don't know the index

There may be cases where you don't know the index of the element you want to swap, but since you'll end up swapping values with the index, I think it's best to just add the process of extracting the index first.

There are two ways to obtain the index: [indexOf](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/index-of.html), which is obtained by passing an element, and [indexOfFirst](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/index-of-first.html), which is obtained by passing a Predicate. All you have to do now is pass the index obtained using these methods to the extension function you implemented earlier. For example, you can implement the following.

```kotlin
// Case using indexOf(element)
fun <T> List<T>.swapByElement(from: T, to: T): List<T> =
    swapByIndex(indexOf(from), indexOf(to))

// Case using indexOfFirst(predicate)
fun <T> List<T>.swapByCondition(from: (T) -> Boolean, to: (T) -> Boolean): List<T> =
    swapByIndex(indexOfFirst { from(it) }, indexOfFirst { to(it) })
```

## Turn time into numbers

Objects like `LocalDate` and `LocalDateTime` from the `java.time` package are useful for working with time in code, but there are times when you need to change the format, such as when writing to a file. This means that you might want it to look like `yyyyMMddhhmmss` instead of `yyyy-MM-dd`. In such cases, it would be useful to write an extension function that can easily change the type to Int. For example, consider the following:

```kotlin
fun LocalDate.toInt(): Int = "$year$monthValue$dayOfMonth".toInt()

val date = LocalDate.of(2021, 12, 31) // 2021-12-31
println(date.toInt()) // 20211231
```

However, when doing this, there are cases where the month and date become single digits, as shown below.

```kotlin
val date = LocalDate.of(2021, 9, 1) // 2021-09-01
println(date.toInt()) // 202191
```

To solve this problem, you first need to convert the month and date to two-digit strings. For example, you can do something like:

```kotlin
fun LocalDate.toInt(): Int = 
    "$year${monthValue.toString().padStart(2, '0')}${dayOfMonth.toString().padStart(2, '0')}".toInt()

val date = LocalDate.of(2021, 9, 1) // 2021-09-01
println(date.toInt()) // 20210901
```

However, even this is not perfect. If you want to use not only `LocalDate` but also other objects belonging to the `java.time` package such as `LocalDate`, `LocalDateTime`, `YearMonth`, etc., you will need to write similar extension functions for all objects.

Fortunately, `LocalDate`, `LocalDateTime`, and `YearMonth` commonly inherit the interface [Temporal](https://docs.oracle.com/javase/8/docs/api/java/time/temporal/Temporal.html), so the problem can be solved by adding an extension function to `Temporal`.

And since the time range handled by these implementation classes differs for each object, the implementation also needs to be changed. All of these objects represent time as numbers, so first convert it to a string with `toString` and then extract only the numbers. Since `String` inherits from `CharSequence`, it would be a good idea to extract only the numbers with `filter`. Then you can use the following methods:

```kotlin
fun Temporal.toDigit(): Long = toString().filter { it.isDigit() }.toLong()

val yearMonth = YearMonth.of(2021, 8) // 2021-08
println(yearMonth.toDigit()) // 202108
val dateTime = LocalDateTime.of(2021, 10, 2, 10, 10, 10) // 2021-10-02T10:10:10
println(dateTime.toDigit()) // 20211002101010
```

When converting to a number in String format, the loop only occurs once with [toInt](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/to-int.html) or [toLong](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/to-long.html), but when treating it as a CharSequence, the loop occurs twice, so the former should be better in terms of performance, but when dealing with time, the loop is not that long, so I don't think it's worth worrying about.

## Add some of the elements

There are cases where you want to aggregate the values in a List into one (to produce a total value). You can also use `sum`, but this is difficult if the elements are not numbers in the first place. For example, what should we do in the case where the element has the following class?

```kotlin
data class Data(
    val name: String,
    val amount: Int,
    val price: Int
)
```

## When there is only one value to add upIf there is only one value you want to add up, you only need to specify the value you want to add up in `sumOf`. The following is a method that can be used when you want to add up only `amount` of `Data` class.

```kotlin
val list = listOf(Data("data1", 10, 100), Data("data2", 20, 200))
val totalAmount = list.sumOf { it.amount }
```

## If there are multiple values you want to add up

What should I do if I want to add up not only `amount` but also `price`? It can be implemented by also using `sumOf` for `price`, but it is not very efficient as the loop occurs twice for the same List. In this case, it would be more efficient to simply declare each sum value as a variable and add the values ​​in the `forEach` loop. For example:

```kotlin
var totalAmount = 0
var totalPrice = 0

list.forEach {
    totalAmount += it.amount
    totalPrice += it.price
}
```

Another method is to use [fold](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html). `fold` is similar to `reduce`, with the difference that you can specify an initial value, but the type of this initial value can be specified to be different from the elements of List, as in [reduceTo](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/reduce-to.html). The result of executing the function will be of the same type as initial, so by applying this, you can also convert the list of `Data` to two values ​​(`Pair`) to `reduce`. For example, the above process can be implemented with a one-liner using `fold` as shown below.

```kotlin
val (totalAmount, totalPrice) = list.fold(0 to 0) { acc, value ->
    (acc.first + value.amount) to (acc.second + value.price)
}
```

When using `fold`, if there are three values you want to add up, you can use [Triple](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-triple/), and if there are even more values, you can probably create a dedicated class. However, in this case, there is an advantage that the summed value can be declared as `val`, but an instance is created for each loop, so the more items you want to sum up, the more likely the performance will not be so good, so you will need to choose the appropriate one depending on the situation.

## Finally

What did you think? I've always written code in Java, so even now that I've completely switched to Kotlin, I still sometimes feel like I end up writing code that looks like Java. Going back to the origins, I think we need to consider what "Java-like code" and "Kotlin-like code" are in the first place, but even so, I think it's true that if the language is different, you need to change your coding style to match that language. I feel that by doing so, I will be able to write better code.

So, I will continue to work on ways to write Kotlin-specific code that are unique to Kotlin. Especially since Java 17 was released this month, I would like to take a look at the list of new APIs and think about how they can be used in Kotlin.

See you soon!
