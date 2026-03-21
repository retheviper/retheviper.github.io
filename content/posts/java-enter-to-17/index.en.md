---
title: "What has changed in Java 17"
date: 2021-09-25
categories:
  - java
image: "../../images/java.webp"
tags:
  - java
  - kotlin
---

This month saw the release of a new LTS version, Java 17. I think there are still many projects using Java 1.8, but since Java 1.8 will be supported until 2022 and Java 11 until 2023, I think it will be necessary to migrate to Java 17 in any case. It seems that migrating from Java 8 was quite difficult, especially since modules were introduced in Java 9, but migrating from Java 11 is said to be less so, so it wouldn't be a bad idea to take a look at what has changed in Java 17.

At present, most of the famous JDKs such as [Eclipse Temurin](https://adoptium.net) (formerly AdoptOpenJDK) and [Zulu](https://www.azul.com/downloads/) have either completed the release of version 17 or are in the process of supporting it. Also, [Oracle JDK 17 is now free](https://www.oracle.com/java/technologies/downloads/#java17), so choosing this may not be a bad option.

Also, in addition to making it free of charge and diversifying the JDK, the lawsuit between Google and Oracle ended with Google's victory, so now that it is possible to use Java 17 on Android, there may be more situations in which Java 17 can be used in the future. In fact, it's still a long way off, but when using Spring, it seems that Java 17 will be the baseline for Spring 6 in 2022 (https://spring.io/blog/2021/09/02/a-java-17-and-jakarta-ee-9-baseline-for-spring-framework-6). Therefore, even in the field where Java 11 has not been adopted, I think there is a possibility of switching to Java 17, taking into account the support period, etc.

So, this time I will talk about what has changed in Java 17, but I would like to trace the changes from two main points of view: language specifications such as the addition of new reserved words and new writing methods, and newly added APIs. Depending on the project, there may be cases of migrating from Java 1.8 to Java 17, but this blog covers the changes and new APIs that occurred between 9 and 11, and I think there are many other sites that can be used as reference, so this time I will omit the changes from 8 to 11 and will only cover the changes between 11 and 17.

## language specs

## New macOS Rendering Pipeline (17)

For a long time, macOS used [OpenGL](https://www.opengl.org/) for Java 2D rendering such as Swing, but from 10.14 OpenGL became `deprecated` while introducing the new [Metal framework](https://developer.apple.com/metal/).

Therefore, on the Java side, [Project Lanai](https://openjdk.java.net/projects/lanai/) was underway to implement a new graphics rendering pipeline that uses Metal, but it was introduced from 17 under the name [New macOS Rendering Pipeline](https://openjdk.java.net/jeps/382). You don't often use GUI in Java, do you? You might think so, but there are rumors that even Java-based IDEs like intellij can improve performance when drawing on the screen. However, intellij basically uses [Jetbrains Runtime](https://confluence.jetbrains.com/display/JBR/JetBrains+Runtime), which does not currently support Java 17, so you will have to wait a little while.

## macOS/AArch64 Port (17)

From 17 onwards, we have made it compatible with new Macs equipped with Apple Silicon, such as M1 (https://openjdk.java.net/jeps/391). In some cases, other JDKs such as [Zulu](https://www.azul.com/downloads/) supported it independently, but now that it is supported by OpenJDK (OracleJDK), I think that other JDKs that are based on this, such as [Eclipse Temurin](https://adoptium.net/) and [Microsoft Build of OpenJDK](https://www.microsoft.com/openjdk), can also be used naturally on ARM-based Macs.

## Record (17)

`Record`, which was introduced as a preview in version 14, became stable and officially introduced in version 17. It sets the specified fields to `private final` and automatically generates constructors, `getter`, `toString`, `hashcode`, `equals`, and so on. At first I thought it was something like [@Data](https://projectlombok.org/features/Data) in `Lombok`, but it is actually closer to [@Value](https://projectlombok.org/features/Value). Values can only be passed in the constructor and cannot be changed later. That is similar to Kotlin's `data class`, where the field is specified as `val`. So, if you look at the actual usage example, it will look like this:

```java
// Define a record
record MyRecord(String name, int number) {}

// Create an instance
MyRecord myRecord = new MyRecord("my record", 1);

// Read the fields
String myRecordsName = myRecord.name();
int myRecordsNumber = myRecord.number();
```

Kotlin supports [Named Arguments](https://kotlinlang.org/docs/functions.html#named-arguments), but Java does not yet have such a feature, so if there are a lot of fields with `Record`, I feel like it will be difficult to tell which is which. On the other hand, if you want to use `Record` on the Kotlin side, you can think of a way to deal with this by creating some kind of wrapper class. Alternatively, it would be better to define a DTO with `setter` or use the builder pattern.

Also, in `Record`, the `getter` name is also the same as the field name, but when customizing the automatically generated constructor, the way it is written is slightly different.

```java
record MyRecord(String name, int number) {
    // Example of adding validation to the constructor
    public MyRecord {
        if (name.isBlank()) {
            throw new IllegalArgumentException();
        }
    }
}
```

In addition, even if you define it as `Record`, `Class` will actually be created, so you can do something like the following.

- Add constructor
- Override `getter`
- Define `Record` as an inner class
- implement the interface

Also, [isRecord](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Class.html#isRecord()) is added to determine whether the class is `Record` even in `Reflection`.

## Text Blocks (15)

For a long time in Java, in order to use HTML, JSON, SQL, etc. as literals, it was necessary to use escapes, string concatenation, etc. This was not very good in terms of readability and made it difficult to modify the code. For example, if you were to express HTML, you would probably do something like the following.

```java
String html = "<html>\n" +
              "    <body>\n" +
              "        <h1>This is Java's new Text block!</h1>\n" +
              "    </body>\n" +
              "</html>\n";

String query = "SELECT \"EMP_ID\", \"LAST_NAME\" FROM \"EMPLOYEE_TB\"\n" +
               "WHERE \"CITY\" = 'INDIANAPOLIS'\n" +
               "ORDER BY \"EMP_ID\", \"LAST_NAME\";\n";
```

Fortunately, since version 15, [Text Blocks](https://openjdk.java.net/jeps/378) have been introduced, making it possible to define strings that are easy and highly readable like in other languages. When you use this, you don't have to worry about escaping, so it can be used effectively in various fields even if you don't have multiple lines. If you change the above code using `Text Blocks`, it will look like this:

```java
String html = """
              <html>
                  <body>
                      <h1>This is Java's new Text block!</h1>
                  </body>
              </html>
              """;

String query = """
               SELECT "EMP_ID", "LAST_NAME" FROM "EMPLOYEE_TB"
               WHERE "CITY" = 'INDIANAPOLIS'
               ORDER BY "EMP_ID", "LAST_NAME";
               """;
```

In Kotlin, you can do the same thing with exactly the same writing style, so I will omit it here.

## Sealed Class (17)

[sealed classes](https://openjdk.java.net/jeps/409) introduced in Preview from JDK 15 is now Stable. By changing `class` or `interface` to `sealed`, you can limit the classes and interfaces that can extend and implement it. By doing this, you can protect classes and interfaces that you do not want to be extended by libraries etc. There also seems to be talk that in the future, when a child class of a class defined as `sealed` is specified as `case` of `switch`, the compiler will check whether all cases are specified. The following is an example of a `sealed` class using the `permits` keyword to specify which classes it can inherit from.

```java
public abstract sealed class Shape permits Circle, Rectangle, Square, WeirdShape { }
```

[Sealed Classes](https://kotlinlang.org/docs/sealed-classes.html) also exists in Kotlin, but in order to change `interface` to `sealed`, you need to use version 1.5 or later, and it does not specify the classes and interfaces that can be extended and implemented, and the specification is that classes and interfaces defined as `sealed` cannot be extended or implemented outside of compiled modules. So, the way to write it is as follows. It's easier.

```kotlin
sealed interface Error

sealed class IOError(): Error

class FileReadError(val f: File): IOError()
class DatabaseError(val source: DataSource): IOError()

object RuntimeError : Error
```

Also, in Java, like `Record`, [isSealed](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Class.html#isSealed()) is added to determine whether this class is `sealed`.

## Switch Expressions (14)

[Switch Expressions](https://openjdk.java.net/jeps/361) was introduced in Preview from Java 12, and has been Stable since Java 14. This is an improved version of the previous `switch`, and now you can do the following.

- `case` can be specified all at once
- `case` processing can be written in a lambda-like manner
- You can use `switch` as an expression by using the process of `case` as a return value.

For example, let's say we want to implement a method that looks at the value of an enum called `day` and returns an int value. The traditional method would look like this:

```java
int numLetters;
switch (day) {
    case MONDAY:
    case FRIDAY:
    case SUNDAY:
        numLetters = 6;
        break;
    case TUESDAY:
        numLetters = 7;
        break;
    case THURSDAY:
    case SATURDAY:
        numLetters = 8;
        break;
    case WEDNESDAY:
        numLetters = 9;
        break;
    default:
        throw new IllegalStateException("Wat: " + day);
}
```

The above process can be written in the new `switch` as follows.

```java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY                -> 7;
    case THURSDAY, SATURDAY     -> 8;
    case WEDNESDAY              -> 9;
};
```

In Kotlin, it should look like this:

```kotlin
val numLetters = when (day) {
    Day.MONDAY, Day.FRIDAY, Day.SUNDAY -> 6
    Day.TUESDAY -> 7
    Day.THURSDAY, Day.SATURDAY -> 8
    Day.WEDNESDAY -> 9
}
```

Also, `when` has features such as being usable without an argument and allowing branching to be based on conditional statements, so I think it is easier to use than Java's `switch`. However, with Java version upgrades, the functions described below have been added, so I think there is a possibility that various improvements will be made in the future like Kotlin.

## Pattern Matching for instanceof (16) / switch (17)

Starting with Java 14, [Pattern Matching for instanceof](https://openjdk.java.net/jeps/394) was introduced, and it became Stable in Java 16. Until now, after determining the instance type of an object using `instanceof`, it was necessary to perform a cast as shown below to perform processing appropriate to that instance type.

```java
static String formatter(Object o) {
    String formatted = "unknown";
    if (o instanceof Integer) {
        formatted = String.format("int %d", (Integer) i);
    }
    // ...
}
```

It's tedious to have to make further casts once you know which instance it is, and if you make a mistake, it could cause an exception. So, you can now use `Pattern Matching` to eliminate the cast. If you specify the variable name to be cast in the conditional statement using `instanceof`, you can use the automatically cast variable as is in `if`. So you will be able to do things like the following.

```java
static String formatter(Object o) {
    String formatted = "unknown";
    if (o instanceof Integer i) {
        formatted = String.format("int %d", i);
    } else if (o instanceof Long l) {
        formatted = String.format("long %d", l);
    } else if (o instanceof Double d) {
        formatted = String.format("double %f", d);
    } else if (o instanceof String s) {
        formatted = String.format("String %s", s);
    }
    return formatted;
}
```

Furthermore, starting with version 17, [Pattern Matching for switch](https://openjdk.java.net/jeps/406) has been introduced as a preview. By using this, you can write simpler processing using `switch` statements without `instanceof`. By combining this with `Switch Expressions` introduced earlier, the above process can be changed to the following. You can see that it's pretty simple.

```java
static String formatterPatternSwitch(Object o) {
    return switch (o) {
        case Integer i -> String.format("int %d", i);
        case Long l    -> String.format("long %d", l);
        case Double d  -> String.format("double %f", d);
        case String s  -> String.format("String %s", s);
        default        -> o.toString();
    };
}
```

## Packaging Tool (16)

[Packaging Tool](https://openjdk.java.net/jeps/392) has been introduced to generate executable binaries. This is a feature that combines the Java runtime, libraries, and executable files for each OS into one package. Including the Java runtime means that it can be executed regardless of the Java version of the OS, so it may be a useful feature if you want to fix the Java version or start multiple applications using different versions of Java.

## API

Starting with Java 17, there is a tab that allows you to view only the list of newly added APIs from the API documentation. This time we will only be looking at the things added after 11, but if a new LTS version is released in the future, you will be able to check the ones added after 17 here. You can check the list of new APIs from [here](https://download.java.net/java/early_access/jdk17/docs/api/new-list.html).

I think it would be difficult to explore the details of all APIs here, so I would like to introduce some that I personally find interesting.

## @Serial (14)

The annotation [Serial](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/Serial.html) has been added to the `java.io` package. This is a class that implements [Serializable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/Serializable.html), and it seems to have a function that changes the serialization mechanism to `@Override`. For example, you can do something like:

```java
class SerializableClass implements Serializable {

    @Serial
    private static final ObjectStreamField[] serialPersistentFields;

    @Serial
    private static final long serialVersionUID;

    @Serial
    private void writeObject(ObjectOutputStream stream) throws IOException {}

    @Serial
    private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException {}

    @Serial
    private void readObjectNoData() throws ObjectStreamException {}
    
    @Serial
    Object writeReplace() throws ObjectStreamException {}

    @Serial
    Object readResolve() throws ObjectStreamException {}
}
```

By adding this annotation, it is also possible to catch errors at compile time. For example, if you use this annotation on a member of a class like the one below, a compilation error will occur.

- Classes that do not implement Serializable
- Classes that have no effect on Serialize, like Enum
- Classes that inherit from [Externalizable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/Externalizable.html)

The addition of such annotations may have some impact on the implementation of libraries such as Jackson and Gson.

## String

Even if they are the same string, Java uses `java.lang.String` and Kotlin uses `kotlin.text.String`, so when using Kotlin, I don't think you will use Java's API much (Also, String-related APIs in Java are often `deprecated` in Kotlin). So, here I will mainly introduce the new API and the methods that can be used to perform similar processing in Kotlin.

### formatted (15)

In Java, you can use `String.format()` to format strings. In many cases, it is said that using format for strings has better performance than using `+`, and this is what I often used.

```java
String name = "formatted string";

// Before Java 15
String formattedString = String.format("this is %s", name);

// Java 15 and later
String newFormattedString = "this is %s".formatted(name);
```

With Kotlin, you can use [String.format](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/format.html) and [String Templates](https://kotlinlang.org/docs/basic-syntax.html#string-templates).

```kotlin
val name = "formatted string"

// Format
val formattedString = "this is %s".format(name)

// String Template
val templateString = "this is $name"
```

### indent (12)

[indent](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/String.html#indent(int)) inserts the amount of white space specified by the argument into the target string. Since the argument is of type `int`, you can also reduce the white space by passing a negative number.

```java
String nonIndent = "A";
// Add indentation
String indented10 = nonIndent.indent(10); // "          A"
// Remove indentation
String indented5 = indented10.indent(-5); // "     A"
```

In the case of Kotlin, there are [prependIndent](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/prepend-indent.html) for adding indentation and [replaceIndent](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/replace-indent.html) for replacing, etc., and the parameters to be passed are also strings, so the usage is slightly different from that of Java.

```kotlin
val nonIndent = "A"
// Add indentation
val prepended = nonIndent.prependIndent("     ") // "     A"
// Replace indentation (or add it if none exists)
val replaced = prepended.replaceIndent("|||||") // "|||||A"
```

### stripIndent (15)

When handling multi-line strings with `Text Block`, if you add arbitrary indentation for readability reasons in the source code, it may be difficult to handle them as actual data. Here, the one to remove the indentation is [stringIndent](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/String.html#stripIndent()).

In Kotlin, [trimIndent](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/trim-indent.html) plays the same role.

### transform (12)

It is a simple API that executes [Function](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/function/Function.html) on a string. It seems to be useful when you need conditional processing that is not possible with `replace`. The implementation is quite simple.

```java
public <R> R transform(Function<? super String, ? extends R> f) {
    return f.apply(this);
}
```

In Kotlin, you can use higher-order functions such as `map`, `filter`, and `reduce` for strings, so you can also use these. Or I think you can do the same thing by defining an extension function like the one below.

```kotlin
fun <R> String.transform(f: (String) -> R): R = f(this)
```

### translateEscapes (15)

This function converts some escaped characters into literals. I think this will be easier to understand if you look at the code.

```java
String string = "this\\nis\\nmutli\\nline";
String escapeTranslated = string.translateEscapes() // "this\nis\nmutli\nline"
```

Previously, I would have had to write my own processing, such as combining `Matcher` with a regular expression, or rely on a library, so it would be nice to be able to do something like this. The escape characters that are converted are as follows.

| Escape | Name | Translation |
|---|---|---|
| `\b` | backspace | U+0008 |
| `\t` | horizontal tab | U+0009 |
| `\n` | line feed | U+000A |
| `\f` | form feed | U+000C |
| `\r` | carriage return | U+000D |
| `\s` | space | U+0020 |
| `\"` | double quote | U+0022 |
| `\'` | single quote | U+0027 |
| `\\` | backslash | U+005C |
| `\0 - \377` | octal escape | code point equivalents |
| `\<line-terminator>` | continuation | discard |

Since there is no similar API in Kotlin, it seems better to write your own processing if necessary. (I don't know about the library...)

## Map.Entry.copyOf (17)

Make a copy of `Map.Entry`. The copied entries will be data that has nothing to do with the original Map. The following sample code is presented from the [official documentation](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.Entry.html).

```java
var entries = map.entrySet().stream().map(Map.Entry::copyOf).toList();
```

By the way, you can copy `Map` itself using [copyOf](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.html#copyOf(java.util.Map)), which was added in version 10.

```java
var copiedMap = Map.copyOf(map);
```

In Kotlin, you can copy `Entry` as follows. The type will be `List<MutableMap.MutableEntry<K, V>>`.

```kotlin
// Using Map.Entry
val entriesJava = map.entries.map { Map.Entry.copyOf(it) }

// Using Kotlin's Map.Entry
val entriesKotlin = map.entries.toSet()
```

Also, you can copy `Map` in Kotlin as follows.

```kotlin
val copiedMap = map.toMap()
```

## Stream

### mapMulti (16)

From version 16, a method called [mapMulti](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html#mapMulti(java.util.function.BiConsumer)) has been added to Stream. Basically, it is a process of ``applying a 1:N transformation to the elements of a Stream and returning the result as a Stream'', so it is similar to [flatMap](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html#flatMap(java.util.function.Function)), but in the following cases it is said to be better than using `flatMap`.

- When reducing elements
- If you have difficulty converting elements to Stream

First, let's consider using `flatMap` for a Collection in which objects are nested. In the case of reducing the number of elements, it is necessary to first expand all elements with `flatMap` and then use `filter` to select only the elements that meet the condition. To expand the elements here, we have to convert all the elements to `Stream`, so we will create an instance of `Stream` for every group of elements. Also, if the objects are nested, you will need to define how to convert each individual element to `Stream` in the process.

The problem is that creating an instance of `Stream` every time creates overhead, and if the elements are objects of various types, it is difficult to write the process to convert them to `Stream`. For example, let's say we have a List like the one below.

```java
List<Object> numbers = List.of(List.of(1, 2L), 3, List.of(4, 5L, 6), List.of(7L), 8L);
```

What should I do if I want to extract only `Integer` from this List and create a separate List? First, if you were to use `flatMap`, you would probably write the following processing.

```java
List<Integer> integers = list.stream()
        .flatMap( // Convert elements into a Stream
                it -> {
                    if (it instanceof Iterable<?> l) {
                        return StreamSupport.stream(l.spliterator(), false);
                    } else {
                        return Stream.of(it);
                    }
                })
        .filter(it -> it instanceof Integer) // Keep only Integers
        .map(it -> (Integer) it) // Cast from Object to Integer
        .toList();
```

When processing this using `mapMulti`, it will be as follows. It's now simpler.

```java
class MultiMapper {
    static void expandIterable(Object e, Consumer<Integer> c) {
        if (e instanceof Iterable<?> i) {
            i.forEach(ie -> expandIterable(ie, c));
        } else if (e instanceof Integer i) {
            c.accept(i);
        }
    }
}

List<Integer> integers = list.stream().mapMulti(MultiMapper::expandIterable).toList();
```

Other methods such as [mapMultiToInt](https://docs.oracle.com/en/java/javase/17/docs/api/new-list.html#:~:text=java.util.stream.Stream.mapMultiToInt(BiConsumer%3C%3F%20super%20T%2C%20%3F%20super%20IntConsumer%3E)), [mapMultiToLong](https://docs.oracle.com/en/java/javase/17/docs/api/new-list.html#:~:text=java.util.stream.Stream.mapMultiToLong(BiConsumer%3C%3F%20super%20T%2C%20%3F%20super%20LongConsumer%3E)), and [mapMultiToDouble](https://docs.oracle.com/en/java/javase/17/docs/api/new-list.html#:~:text=java.util.stream.Stream.mapMultiToDouble(BiConsumer%3C%3F%20super%20T%2C%20%3F%20super%20DoubleConsumer%3E)) have also been added, so it may be more convenient to use these when dealing with numbers. For example, if you write the above `mapMulti` as `mapMultiToInt`, it will be as follows.

```java
class MultiMapper {
    static void expandIterable(Object e, IntConsumer c) {
        if (e instanceof Iterable<?> i) {
            i.forEach(ie -> expandIterable(ie, c));
        } else if (e instanceof Integer i) {
            c.accept(i);
        }
    }
}

List<Integer> integers = list.stream().mapMultiToInt(MultiMapper::expandIterable).boxed().toList();
```

The return value of `mapMultiToInt` is [IntStream](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/IntStream.html), so there are slight differences such as calling [boxed](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/IntStream.html#boxed()) to convert it to `Stream<Integer>`, `Consumer` changes to `IntConsumer`, and the type specification of `mapMulti` changes.

Kotlin does not treat `flatMap` as `Stream` in the first place, so you need to think about the process from a different perspective. Fortunately, Kotlin Collection has various APIs, so it's not that difficult. For example, if you want to aggregate elements based on object instances, you can write code like the following.

```kotlin
val list = listOf(listOf("A", 'B'), "C", setOf("D", 'E', "F"), listOf('G'), 'H')

val result: List<String> = list.flatMap {
    if (it is Iterable<*>) {
        it.filterIsInstance<String>()
    } else {
        listOf(it).filterIsInstance<String>()
    }
} // [A, C, D, F]
```

However, in Java, if you create a List with `List.of(1, 2L)`, 1 is treated as int and 2L is treated as Long, but in Kotlin, `listOf(1, 2L)` becomes `List<Long>`, so you need to be careful about the original type.

```kotlin
val list = listOf(listOf(1, 2L), 3, setOf(4, 5L, 6), listOf(7L), 8L)

val result = list.flatMap {
    if (it is Iterable<*>) {
        it.filterIsInstance<Int>()
    } else {
        listOf(it).filterIsInstance<Int>()
    }
} // [3]
```

### toList(16)

It is a method that feels like it was created as syntactic sugar for ``summarize into a list'', which is frequently used as a terminal process for a Stream. It feels like Java has adopted Kotlin's functionality here. The List generated as a result of processing is `Unmodifiable`.

```java
List<String> list = List.of("a", "B", "c", "D");

// Old
List<String> upper = list.stream().map(String::toUpperCase).collect(Collectors.toUnmodifiableList());

// New
List<String> lower = list.stream().map(String::toLowerCase).toList();
```

In Kotlin, the result of calling a higher-order function on a Collection is basically a `Unmodifiable` List, but it can also be converted to `stream` and used, so it may be useful in some cases.

## Collectors.teeing (12)

A method called [teeing](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Collectors.html#teeing(java.util.stream.Collector,java.util.stream.Collector,java.util.function.BiFunction)) has been added to Collectors to join two `Collector`. By the way, `Tee` seems to have the meaning of ``T-joint'' that connects two water pipes and makes them one. The argument is two `Collector` and `BiFunction` which is the process to combine them.

For example, let's say we have the following `Stream`.

```java
record Member(String name, boolean enabled) {}

/**
* [
*    Member[name=Member1, enabled=false],
*    Member[name=Member2, enabled=true],
*    Member[name=Member3, enabled=false],
*    Member[name=Member4, enabled=true],
* ]
*/
Stream<Member> members = IntStream.rangeClosed(1, 4).mapToObj(it -> new Member("Member" + it, it % 2 == 0));
```

If you use `teeing` to divide this into two Lists based on `enabled` of `Member`, it will look like the following.

```java
/**
* [
*    [
*       Member[name=Member2, enabled=true],
*       Member[name=Member4, enabled=true]
*    ],
*
*    [
*       Member[name=Member1, enabled=false],
*       Member[name=Member3, enabled=false]
*    ]
* ]
*/
List<List<Member>> result = members.collect(
        Collectors.teeing(
                Collectors.filtering(
                        Member::enabled,
                        Collectors.toList()
                ),
                Collectors.filtering(
                        Predicate.not(Member::enabled),
                        Collectors.toList()
                ),
                (list1, list2) -> List.of(list1, list2)
        )
);
```

In Kotlin, there is no need to do `collect` in the first place, so it is better to use the higher-order functions of `Collection`. (I think it's easier to understand if you do that in Java as well...)

## lastly

What did you think? As expected, it was difficult to organize all the changes, so I tried to highlight some of the most noticeable changes, but it's still quite a large amount. However, one thing that is certain is that Java 17 is a more modern version of the language than 11, so it seems well worth introducing if you are using Java for projects. Also, it seems that Java 15 has improved performance due to improvements in G1GC compared to Java 11 (https://www.optaplanner.org/blog/2021/01/26/HowMuchFasterIsJava15.html), so it is good in terms of performance.

Even if you are using Kotlin, there may not be much benefit if you only look at the API, but as long as you are using JVM, you will be able to get benefits such as improved performance, so I think it is a good idea to consider introducing it. Also, various interesting APIs may appear one after another in the next LTS.

See you soon!
