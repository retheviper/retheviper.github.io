---
title: "Let's deal with the year and month"
date: 2021-04-27
translationKey: "posts/kotlin-year-month"
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - spring
---

In Kotlin (Java), you can process dates and times with classes in the `java.time` package. For example, there are [LocalDateTime](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/LocalDateTime.html) and [LocalDate](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/LocalDate.html). On the server side, you can use these classes to input dates and times into the DB, and to set the validity period of authentication tokens. There are also [Period](https://docs.oracle.com/javase/8/docs/api/java/time/Period.html) and [Duration](https://docs.oracle.com/javase/8/docs/api/java/time/Duration.html), which can also handle "period".

However, what should I do if I want to handle units such as "years and months"? For example, if you set a period such as "February to April" when inquiring about account deposit and withdrawal details, including unnecessary "days" and "hours" is not very efficient and may cause bugs in some cases. In such cases, you will need to consider either a method such as treating the data as a definite "year and month" or expressing it as a number.

So, this time I would like to talk a little about how to handle these years and months.

## year and month to year and month

Handling years and months also means that you want to be able to separate them into two pieces of data: "year" and "month" at any time. Here we will explain how to handle "Year/Month" by dividing it into "Year" and "Month" using two methods.

## As YearMonth

`LocalDate` and `LocalDateTime` can basically handle dates in [ISO-8601](https://www.iso.org/iso-8601-date-and-time-format.html) format. Of course, you can specify other formats using [DateTimeFormatter](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/format/DateTimeFormatter.html), but the only difference is the format of the data being handled, and essentially "year, month, and day" is the basic format.

The fact that dates are handled in the `ISO-8601` "Year Month Day" format also means that when creating a REST API with Spring, if the request value adheres to the `ISO-8601` format, it will be automatically converted to the `LocalDateTime` or `LocalDate` format. For example, let's say we have the following request JSON.

```json
{
    "id": "1",
    "date": "2021-04-01"
}
```

On the Spring side, you can convert the request date to `LocalDate` with the code below.

```kotlin
// Request body
data class DateRequest(val id: Int, val date: LocalDate)

// Controller
@PostMapping("/date")
fun date(@RequestBody request: DateRequest) {
    // ...
}
```

You can do the same for year and month by changing `LocalDate` to [YearMonth](https://docs.oracle.com/javase/8/docs/api/java/time/YearMonth.html). For example, suppose you receive the following request.

```json
{
    "id": "1",
    "yearMonth": "2021-04"
}
```

Now just change `yearMonth` to `YearMonth`. It will look like this:

```kotlin
// Request body
data class YearMonthRequest(val id: Int, val yearMonth: YearMonth)

// Controller
@PostMapping("/year-month")
fun yearMonth(@RequestBody request: YearMonthRequest) {
    // ...
}
```

The advantage of using `YearMonth` is that it is an object that belongs to the `java.time` package like `LocalDateTime` and `LocalDate`, so it is compatible with them and can be freely converted between them. For example, you can use it like this:

```kotlin
>>> val yearMonth = YearMonth.now() // Get the current year and month
>>> println(yearMonth)
2021-04
>>> val localDate = yearMonth.atDay(1) // Convert the year/month into a LocalDate by specifying the day
>>> println(localDate)
2021-04-01
```

In addition, `YearMonth` provides many useful methods related to time, so it may be useful when you need to do date-related processing to meet various requirements, rather than simply handling years and months as numbers. For example, the following functions are provided:

```kotlin
>>> val yearMonth = YearMonth.of(2021, 5)
>>> println(yearMonth)
2021-05
>>> println(yearMonth.getYear()) // Get the year
2021
>>> println(yearMonth.getMonth()) // Get the month enum
MAY
>>> println(yearMonth.getMonthValue()) // Get the month as a number
5
>>> println(yearMonth.isLeapYear()) // Check whether it is a leap year
false
>>> println(yearMonth.atEndOfMonth()) // The last day of the month as LocalDate
2021-05-31
```

## as a number

It seems the most beautiful way to receive and process it as `YearMonth`, but depending on the situation, there may be cases where it is better (or only possible) to simply receive it as `Int` type. For example, the following request is sent.

```json
{
    "id": "1",
    "yearMonth": 202104
}
```

It would be easier if they were separate items like `year` and `month`, but if the year and month are sent as one `Int` type data like this, you have no choice but to create a process to extract the year and month yourself. For example, you could write an extension function like the following.

```kotlin
// Extract the year
fun Int.extractYear(): Int = this / 100
// Extract the month
fun Int.extractMonth(): Int = this % 100
```

When you run the actual code, you can confirm that it works as intended.

```kotlin
>>> fun Int.extractYear(): Int = this / 100
>>> 202104.extractYear()
res4: kotlin.Int = 2021
>>> fun Int.extractMonth(): Int = this % 100
>>> 202104.extractMonth()
res6: kotlin.Int = 4
```

However, the problem is that the value passed as a parameter is just a `Int` type, so it may not be the expected value. It is always necessary to check whether data is sent in the form of `YYYYMM`.

In such a case, with the above code, it is unclear whether `yearMonth` in the request is in the correct year and month format. Therefore, it would be safer to include a validation check using a regular expression. For example, you can use code like the following:

```kotlin
fun Int.toYearMonth(): Pair<Int, Int> =
    if (Regex("^(19|20)\\d{2}(0[1-9]|1[012])").matches(this.toString()))
        this / 100 to this % 100
    else
        throw IllegalArgumentException("cannot convert")
```

The above function can be used as follows. It feels good because it's easy to use.

```kotlin
>>> val (year, month) = 202104.toYearMonth()
>>> println(year)
2021
>>> println(month)
4
```

I used `Pair` as the return value to split the original value into two `Int`, but `YearMonth` may be better in some cases. In that case, you can use code like the following.

```kotlin
fun Int.toYearMonth(): YearMonth =
    if (Regex("^(19|20)\\d{2}(0[1-9]|1[012])").matches(this.toString()))
        YearMonth(this / 100, this % 100)
    else
        throw IllegalArgumentException("cannot convert")
```

## year and month to year and month

Now, let's consider the process of concatenating "year" and "month" to create "year/month". This is a form that combines two `Int` to make one `Int`(YYYYMM). There are two possible methods here. There are two methods: using `YearMonth` and converting it to a string before processing.

## YearMonth

First, when using `YearMonth`, it is best to pass the year and month as arguments as they are, and then convert them to `Int`. However, `YearMonth` is basically `ISO-8601` format, so in April 2021 it will be `2021-04` and cannot be converted to `Int`. So, we will first change it to `String`, then delete `-` and convert it to `Int`. The above processing results in code like the following.

```kotlin
fun toYearMonth(year: Int, month: Int): Int = 
    YearMonth.of(year, month).toString().replace("-", "").toInt()
```

## in string

If you want to process it with a string, you can simply use [String templates](https://kotlinlang.org/docs/basic-types.html#string-templates), but please note that the month has a range of 1 to 12, so if you simply connect the year and month using template, you could end up with something like `20214`. Therefore, use `padStart()` and add `0` at the beginning if the month is from 1 to 9. After that, just convert it to `Int`. This will result in code like this:

```kotlin
fun toYearMonth(year: Int, month: Int): Int = "${year}${month.toString().padStart(2, '0')}".toInt()
```

Since these methods require two arguments, they can also be defined as `infix` (I think this is a matter of preference).

```kotlin
>>> infix fun Int.toYearMonthWith(month: Int): Int = "${this}${month.toString().padStart(2, '0')}".toInt()
>>> 2021 toYearMonthWith 5
res10: kotlin.Int = 202105
```

## lastly

What did you think? It wasn't a very difficult code, so I felt like I needed to make an article about it, but personally, this was the first time I learned about the existence of a class called `YearMonth`, and I wanted to write code (extension function) unique to Kotlin, so I wanted to share what I tried. If you need to handle years and months in Kotlin or Java, I hope this will be of some help.

See you soon!
