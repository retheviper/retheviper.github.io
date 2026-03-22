---
title: "Using the Japanese calendar with Kotlin"
date: 2021-12-05
translationKey: "posts/kotlin-japanese-era"
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---
There are times when it is necessary to process Japanese calendars, such as in forms. For example, you may want to write the name of an era or the year according to the Japanese calendar. In the case of Kotlin (JVM), it is easy to use Java APIs such as `Date` and `LocalDate` for the Western calendar, but since the Japanese calendar is required in only a few cases, it may be difficult to understand how to do it. So, this time I have summarized a little about how to handle the Japanese calendar in Kotlin.

## JapanseEra / JapaneseDate

Since Java 1.8, we have provided APIs called [JapaneseDate](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/chrono/JapaneseDate.html) that can handle dates in the Japanese calendar and [JapaneseEra](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/chrono/JapaneseEra.html) that can handle era names. Therefore, by creating an instance of `JapaneseDate` and retrieving `JapaneseEra` from it, you can easily retrieve era name information. The actual usage is as follows.

```kotlin
// Get the current JapaneseDate
val japaneseDate = JapaneseDate.now()

// Get the JapaneseEra
val japaneseEra = japaneseDate.era
```

In the case of `JapaneseDate`, like `LocalDate`, it inherits [ChronoLocalDate](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/chrono/ChronoLocalDate.html), so the method of creating an instance is not much different. So you can also do something like the following:

```kotlin
// Convert LocalDate to JapaneseDate
val japaneseDateFromLocalDate = JapaneseDate.from(LocalDate.now())

// Create a JapaneseDate from a specific date
val japaneseDateFromSpecificDate = JapaneseDate.of(2000, 12, 31)
```

## Write the era name in Japanese

I think there are two main things you want to do when dealing with the Japanese calendar. One is to treat the era name as a string, and the other is to treat the year in the Japanese calendar as a number. First, I will explain how to obtain the era name as a string.

As introduced above, first we need to obtain an instance of `JapaneseDate`, and then we need to obtain `JapaneseEra` held by that object. After that, you can obtain the string by specifying [TextStyle](https://docs.oracle.com/javase/8/docs/api/java/time/format/TextStyle.html) and [Locale](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Locale.html) to the function called `JapaneseEra.getDisplayName()`. Think of the former as an enumeration constant that specifies the character output type, and the latter as a language specification.

For `TextStyle`, the values ​​are as follows: In other languages, the output may vary considerably depending on what you specify, but I think `FULL` and `NARROW` are sufficient for Japanese.

| Constant | Output example |
|---|---|
| `FULL` | Showa |
| `FULL_STANDALONE` | Showa |
| `NARROW` | S |
| `NARROW_STANDALONE` | S |
| `SHORT` | Showa |
| `SHORT_STANDALONE` | Showa |

In the case of `Locale`, the result is the same whether you specify `Locale.JAPAN` or `Locale.JAPANESE`. However, the implementation is as follows, so it seems better to use `Locale.JAPAN` as much as possible.

| Locale | Setting of BaseLocale to be created |
|---|---|
| `JAPAN` | `language = ja, region = JP` |
| `JAPANESE` | `language = ja` |

The following is an example of passing these constants and obtaining the era name as a string.

```kotlin
val today = JapaneseDate.now()
val era = today.era

// Get the era name in kanji
val eraName = era.getDisplayName(TextStyle.FULL, Locale.JAPAN) // Reiwa
```

You may want to include not only the era name but also the year. What you can use in that case is [DateTimeFormatter](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/format/DateTimeFormatter.html). This is also possible because `JapaneseDate` inherits from `ChronoLocalDate`, just like `LocalDate`.

```kotlin
// Format the date in Japanese
val formatter = DateTimeFormatter.ofPattern("Gy\u5e74", Locale.JAPAN)
val todayString = formatter.format(JapaneseDate.now()) // Reiwa 3
```

If you cannot use `LocalDate` or `JapaneseDate` and have no choice but to use `java.util.Date`, such as when using Java 1.8 or earlier versions, you can obtain the year name and year using the following method.

```kotlin
val format = SimpleDateFormat("Gy\u5e74", Locale("Ja", "JP", "JP"))
val year = format.format(Date()) // Reiwa 3
```

When using `java.util.Date`, you need to specify up to the third argument `variant` to `Locale`, so you cannot use one defined as an existing enumerated type.

Also, if you set it to something like `Locale.ENGLISH`, you may get a mixed result that combines `AD` with a Japanese-style date string even if you are using `JapaneseDate`.

## Use ligatures

For era names, you may want to obtain and use ligatures in Unicode. In that case, I think it would be a good idea to define and retrieve a Unicode Map etc. as shown below. It would also be a good idea to define extension functions etc.

```kotlin
val eraUnicodeMap = mapOf(
    JapaneseEra.MEIJI to "\u337e", // ㍾
    JapaneseEra.TAISHO to "\u337d", // ㍽
    JapaneseEra.SHOWA to "\u337c", // ㍼
    JapaneseEra.HEISEI to "\u337b", // ㍻
    JapaneseEra.REIWA to "\u32ff" // ㋿
)

val era = JapaneseDate.now().era
// Get the era name as a ligature
val eraUnicode = eraUnicodeMap[era] // ㋿
```

In the above sample, `JapaneseEra` is an enumeration type, so it is used as a key, but `JapaneseEra` also has numerical information, so there may be a way to use that. The numerical values ​​for each value are as follows.

| JapaneseEra | Numerical value |
|---|---|
| MEIJI | -1 |
| TAISHO | 0 |
| SHOWA | 1 |
| HEISEI | 2 |
| REIWA | 3 |

In the case of March from 2021 to 2022, it is Reiwa 3, so I think it is easy to misunderstand that the value of `JapaneseEra.REIWA.value` is the fiscal year. Please note that the actual year information is in `JapaneseDate`.

## Display the year in numbers

Since `JapaneseEra` is a class of enumeration constants used to obtain the era name, it itself does not have the date information of `JapaneseDate`. Therefore, the only information you can refer to is the era name to which the original `JapaneseDate` belongs.

Therefore, the year as a number must be obtained by passing the enumeration type [ChronoField](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/temporal/ChronoField.html) to `JapaneseDate.get()`.

```kotlin
val today = JapaneseDate.of(2010, 12, 31) // Heisei 22

// Get the year as an Int
val year = today.get(ChronoField.YEAR) // 2010
val yearOfHeisei = today.get(ChronoField.YEAR_OF_ERA) // 22
```

This is because `JapaneseDate`, unlike `LocalDate`, cannot directly obtain `year` with a getter. If you actually look inside the object, `LocalDate` holds the year, month, and date as int and short fields, while `JapaneseDate` has `LocalDate` and `yearOfEra` of int type, and you can only obtain `yearOfEra` through `get(ChronoField.YEAR_OF_ERA)`. I think the reason why no getter is provided is probably because there are two concepts: `LocalDate` and `yearOfEra`. Of course, since it's Kotlin, you can easily create a getter by writing an extension function.

Also, if you are using `LocalDate` as a date object, the year in the Western calendar will be returned even if you pass `ChronoField.YEAR_OF_ERA`, so first check whether you are using `JapaneseDate` to use the Japanese calendar.

## Display year as two digit characters

Strictly speaking, this has nothing to do with the Japanese calendar, but when obtaining and using the year, you may want to consistently treat it as a two-digit string with a "0" at the end. If you obtain the year through `JapaneseDate`, it will be in the `Int` type, so the numbers between 1 and 9 will be single digits, but if you want to display this as 01 to 09, you can use the following method.

### Use DecimalFormat

One is to use the Java API [DecimalFormat](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/text/DecimalFormat.html). Personally, I prefer this method because it allows me to specify the range of decimal points etc. in an easy-to-understand manner.

```kotlin
val today = JapaneseDate.now() // Reiwa 3

// Specify a format for displaying numbers
val decimalFormat = DecimalFormat("00")
val year = decimalFormat.format(today) // 03
```

### Use String.format

Another way is to use [String.format()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/format.html), a feature of Kotlin's standard library. If you are looking at performance, I think this method is better.

```kotlin
val today = JapaneseDate.now() // Reiwa 3

// Specify a format for displaying numbers
val year = "%02d".format(today) // 03
```

## Extra: kotlinx-datetime

Kotlin originally did not have an API to handle dates and times, but since 2020, it has provided [kotlinx-datetime](https://github.com/Kotlin/kotlinx-datetime). Therefore, we have created an official API that can handle dates even when it does not run on JVM, such as Kotlin/JS and Kotlin/Native, but there are some concerns, so I think it is necessary to consider before introducing this.

## Pre-release stage

`kotlinx-datetime` is still in the pre-release stage, and `v0.3.1` will be released in October 2021. Therefore, there may be various bugs or things may not work as expected. Also, it can't be helped because it is still under development, but the functionality provided at the moment is less compared to the `java.time` API, and it is not possible to easily calculate era names. For now, you can think of it as providing only the bare minimum of functionality.

## For multiple platforms

The Kotlin standard library and the library provided as `kotlinx` are implemented with multi-platform considerations in mind, so they have the advantage of being able to be used in the same way on different platforms, but they can also have disadvantages. In fact, the `kotlinx-datetime` JVM implementation internally relies on the `jata.time` API, so if you only use the JVM, you may not need to install it.

Also, the fact that the implementation differs depending on the platform means that unexpected exceptions may occur somewhere, or cases may occur where the expected results may not be obtained.## java.time concerns

In `JapaneseEra`, era names before the Meiji era (Keio, etc.) cannot be used, but I think the reason for this is probably due to the historical background that the Gregorian calendar was used in the Japanese calendar from the Meiji era. Also, in `JapaneseDate`, if a date before 1873 (January 1, 1873) is specified, the following exception will occur.

```shell
Exception in thread "main" java.time.DateTimeException: JapaneseDate before Meiji 6 is not supported
	at java.base/java.time.chrono.JapaneseDate.<init>(JapaneseDate.java:333)
	at java.base/java.time.chrono.JapaneseDate.of(JapaneseDate.java:257)
```

Therefore, the method introduced here may not be usable in cases such as simply creating a form, but also in calculating dates for historical research.

Also, you need to be careful because there are cases where `JapaneseEra.REIWA` cannot be obtained and an error may occur due to issues such as the JDK version. Even in this case, there is no problem in obtaining the value of `value`, so it seems better to avoid using the constant as is for branching decisions, even though it may slightly reduce readability. (I don't know the exact reason, but...)

## Finally

How was it? This is a summary of what I started researching out of curiosity, but it is also a process that is actually necessary in my day job, so I think this was a good opportunity to think about how this can be implemented as an extension function.

Also, regarding the Java API, there was [a good article summarizing Java-version support for era changes and new era names](https://qiita.com/yamadamn/items/56e7370bae2ceaec55d5), so if you are interested, it is worth looking up.

See you soon!
