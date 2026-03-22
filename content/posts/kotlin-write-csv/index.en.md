---
title: "Converting a List of Data Classes to CSV"
date: 2022-08-27
translationKey: "posts/kotlin-write-csv"
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - csv
---

When writing an app, there are many cases where data is read or output in a format different from the format saved in the DB. A typical example is REST API, which is used in many backend applications. In many cases, the input value and return value of the API do not match the format stored in the DB, so the API receives parameters appropriately and returns a response according to the needs (convenience of the side that sent the request). At times, it may be necessary to summarize data in a format that is easily readable by humans. In such cases, you can assume various things such as Excel files, CSV, and PDF.

This post is also about how to implement the Kotlin side, especially when dealing with CSV as a human-readable file.

## Handling CSV with Kotlin

[kotlin-csv](https://github.com/doyaaaaaken/kotlin-csv/) is a Kotlin CSV reading/writing library, and you can use this library to easily handle CSV not only with JVM but also with Kotlin/JS. Furthermore, there is a library called [kotlin-grass](https://github.com/blackmo18/kotlin-grass), and in combination with `kotlin-csv`, you can easily organize CSV data as a data class List. It has a wide range of functions such as data formats that can be specified when reading and custom mapping options, making it a very easy-to-use library.

However, there is actually one problem when using `kotlin-csv`. As mentioned earlier, it is possible to output data to CSV, but just as a separate library was required to map to a data class when reading, additional processing is required to write a list of data classes. This is because the kotlin-csv writing method is as follows.

```kotlin
fun writeAll(rows: List<List<Any?>>, targetFile: File, append: Boolean = false) {
    open(targetFile, append) { writeRows(rows) }
}
```

Here, `rows` is the data used for writing, and its type is `List<List<Any?>>`. In other words, each row of the CSV must be represented as a `List`, and the entire CSV must be represented as a list of those rows. To write a list of data classes, you need to turn each object's fields into a row. CSV usually also includes a header row, so you need to prepare that first row separately as well.

Although it looks complicated at first glance, by using [reflection](https://kotlinlang.org/docs/reflection.html), you can obtain the data class field names and their values, so you can use them to change the data class List into a form suitable for this method. I will explain this in two steps: how to create the header, and how to change the data class value to a row.

## Create a Header from a Data Class

First, create the header. To create a header, it would be a good idea to get the field from the data class and get only the name of that field. If there is a field called `id`, the header will also be `id`. If you want to give a name different from the field name, you can consider using annotations, but first I would like to explain how to use the field name as is.

There are three ways to retrieve fields from a Kotlin data class. First of all, there is [KClass.members](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-class/members.html). However, this will bring all the members including the method. It's like below.

```kotlin
data class Data(val id: Int, val name: String)

val members = Data::class.members
println(members)
// [val Line_2.Data.id: kotlin.Int, val Line_2.Data.name: kotlin.String, fun Line_2.Data.component1(): kotlin.Int, fun Line_2.Data.component2(): kotlin.String, fun Line_2.Data.copy(kotlin.Int, kotlin.String): Line_2.Data, fun Line_2.Data.equals(kotlin.Any?): kotlin.Boolean, fun Line_2.Data.hashCode(): kotlin.Int, fun Line_2.Data.toString(): kotlin.String]
```

This is one method that can be used because the field name is shown, but in a data class, basically methods such as `equals()`, `hashCode()`, `copy()`, `toString()`, `componentN()` are available.
You need to filter these out. For example, like below.

```kotlin
val memberProperties = Data::class.members.filterNot { it.name.contains("component") || it.name == "copy" || it.name == "equals" || it.name == "hashCode" || it.name == "toString" }

println(memberProperties)
// [val Line_2.Data.id: kotlin.Int, val Line_2.Data.name: kotlin.String]
```

However, there is an easier way to extract only fields without filtering. The solution is to use [memberProperties](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect.full/member-properties.html).

```kotlin
val memberProperties = Data::class.memberProperties

println(memberProperties)
// [val Line_2.Data.id: kotlin.Int, val Line_2.Data.name: kotlin.String]
```

However, this method also has problems. This means that the retrieved fields will not be ordered as defined in the data class, but in alphabetical order. In the example below, you can see that the fields defined in the order name and age are now in the order age and name.

```kotlin
data class Person(val name: String, val age: Int)

val memberProperties = Person::class.memberProperties
println(memberProperties) // The order will not be name, age
// [val Line_11.Person.age: kotlin.Int, val Line_11.Person.name: kotlin.String]
```

If you really want to get the fields in the order they were defined, you can use the data class constructor. The first method is to use a constructor, which takes advantage of the fact that the data class is automatically created with parameters in the order in which the fields are defined. It will look like this:

```kotlin
val parameters = Person::class.primaryConstructor!!.parameters.mapNotNull { it.name }

println(parameters) // [name, age]
```

Although it is a somewhat brute-force method, I was able to obtain the field name to be used as a field. Next, let's look at when to use annotations.

### When using annotations

If you do not want to use the field name as it is as a header, you can use annotations. The method is to define an annotation with a String as a field and retrieve the annotation attached to that field when reading the header. For example, let's say you define the following annotation.

```kotlin
@Target(AnnotationTarget.PROPERTY)
annotation class CsvHeaderName(val value: String)
```

Annotations are used in data classes as shown below.

```kotlin
data class Person(
    @CsvHeaderName("Name")
    val name: String,
    @CsvHeaderName("Age")
    val age: Int
)
```

Then we will get the annotations of this data class. When you get a field with `memberProperties`, you get a list of annotations (because there can be multiple annotations) from that field, and then filter only the previously defined `CsvHeaderName`. All you have to do is check whether there is an annotation and decide which value to use. The code below is a sample.

```kotlin
val datas = listOf(Person("John", 20))

val headers = datas.first()!!::class.memberProperties.map { property ->
            val name = property.annotations.filterIsInstance<CsvHeaderName>().firstOrNull() // There may be no annotation
            name?.value ?: property.name // If the annotation is null, use the field name
        }

println(headers) // [Age, Name]
```

Even if you obtain it using `primaryConstructor` parameters, the method does not change much. In this case, we simply loop through the constructor parameters and look for matching fields. For example:

```kotlin
val fieldNames = datas.first()::class.primaryConstructor!!.parameters.mapNotNull { it.name }

val headers = fieldNames.mapNotNull { name ->
    // Process only the field that matches the parameter name
    datas.first()::class.memberProperties.find { it.name == name }?.let { property ->
        val headerName = property.annotations.filterIsInstance<CsvHeaderName>().firstOrNull()
        headerName?.value ?: property.name
    }
}
```

The header data is now ready. Next, just convert the data class to List as a line to be output below according to this header.

## Change data class to List

As we have already done with header processing, the processing does not change much when converting a data class to a List. The difference is that we just get the actual data from the field. Here, let's write the code assuming that the parameters are obtained from the constructor.

When retrieving a field value using reflection in Kotlin, it is no different from Java; all you need to do is pass the data class instance to the retrieved field. However, you need to take into account if the field is null. If it becomes null, that column itself will be skipped, and the column may be misaligned in the final output CSV data. Therefore, it is necessary to match the length of each line (List size) by specifying a blank String. It's like below.

```kotlin
datas.map { d ->
    fieldNames.mapNotNull { name ->
        d::class.memberProperties.find { it.name == name }?.let { field ->
            field.call(d) ?: "" // Read the field value; use a blank if it is null
            } 
        }
    }
```

However, when handling time and dates here, there may be cases where you want to use a formatter. For example, you may want to output the data as `LocalTime` in the app, but as a CSV such as `HH:mm`, or you may want to convert `LocalDate` to `yy/MM/dd`. Here, the format itself is simply using [DateTimeFormatter](https://docs.oracle.com/javase/8/docs/api/java/time/format/DateTimeFormatter.html), but the problem is determining what type the retrieved field is.

The field obtained by Kotlin reflection is of type [KProperty1](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-property1/). The problem here is how to get the original type. This class implements an interface called [KCallable](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-callable/), which has a property called [returnType](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-callable/return-type.html). Now you will be able to obtain the interface [KType](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-type/), so you will be able to make a decision using this.

However, as the name suggests, `KType` is an interface for Kotlin types. The classes I want to compare, such as `LocalDate` and `LocalTime`, are Java classes, so I can't compare them directly. Fortunately, Java classes can still be converted into a Kotlin `KType`, like this:

```kotlin
val localDateKType: KType = LocalDate::class.createType()
```

So, all you have to do is branch and use the appropriate formatter. It is as below.

```kotlin
datas.map { d ->
        fieldNames.mapNotNull { name ->
            d::class.memberProperties.find { it.name == name }?.let { field ->
                field.call(d)?.let {
                    when (field.returnType) { // Branch based on the field type
                        LocalDate::class.createType() -> dateFormatter.format(it as LocalDate)
                        LocalTime::class.createType() -> timeFormatter.format(it as LocalTime)
                        LocalDateTime::class.createType() -> dateTimeFormatter.format(it as LocalDateTime)
                        else -> it
                    }
                } ?: ""
            }
        }
    }
```

However, another thing to note here is that the KType of a nullable field will be different. In other words, in the above code, the formatter will not work for the following data class fields during branch processing.

```kotlin
// None of these fields would be formatted
data class DateTimes(
    val date: LocalDate?,
    val time: LocalTime?,
    val dateTime: LocalDateTime?
)
```

In this case, you can easily resolve this by specifying that it is nullable when creating `KType`. All you have to do is branch and check both. It's like below.

```kotlin
datas.map { d ->
        fieldNames.mapNotNull { name ->
            d::class.memberProperties.find { it.name == name }?.let { field ->
                field.call(d)?.let {
                    when (field.returnType) { // Format both nullable and non-nullable cases
                        LocalDate::class.createType(), LocalDate::class.createType(nullable = true) -> dateFormatter.format(it as LocalDate)
                        LocalTime::class.createType(), LocalTime::class.createType(nullable = true) -> timeFormatter.format(it as LocalTime)
                        LocalDateTime::class.createType(), LocalDateTime::class.createType(nullable = true) -> dateTimeFormatter.format(it as LocalDateTime)
                        else -> it
                    }
                } ?: ""
            }
        }
    }
```

All that remains is to combine the header with the list of values obtained from the data class and pass them to `writeAll()` in `kotlin-csv`. The values are already in `List<List<Any>>` form, but the header is a `List<String>`, so the header also needs to be wrapped in a list.

```kotlin
// Header
val headers = // omitted
// Actual data
val rows = datas.map { /** omitted */ }

csvWriter().writeAll(
            rows = listOf(headers) + rows,
            targetFile = targetFile
        )
```

Now the header will be written first, and the actual values stored in the data class fields will be written from the following row onward.

## Finally

This time, I was in trouble because the library I adopted was different from what I had expected, thinking, ``Since it's Kotlin, let's use a Kotlin-based library,'' but fortunately, when I was using Java, I had the experience of creating a library with a similar function using [Apache POI](https://poi.apache.org/), so I can say that I was able to put that knowledge to good use. At the time, I was just a budding engineer (I still think I am), so I remember having a lot of trouble, but now I feel like I was able to deal with it because of that experience, so I think it was a very rewarding experience.

I created a small library for the above code, so I would like to use it somewhere. I would like to make various improvements and be able to publish it on a place like Maven Repository later.

See you soon!
