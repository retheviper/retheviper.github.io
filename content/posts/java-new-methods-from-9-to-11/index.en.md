---
title: "Tour of new methods from 9"
date: 2020-12-30
categories: 
  - java
image: "../../images/java.webp"
tags:
  - java
  - stream
  - collection
  - optional
  - string
---
I often work with Java 11 at work, but to be honest, when I look back at the code I've written, the reality is that I don't use many of the new methods added in Java 9. However, these new methods not only exist as syntactic sugar to hide redundancy, but also have better performance and functionality, so I think there are many that you should take a look at even if you don't use them right away.

2021 is the time when the next LTS version, 17, has been announced, so it feels like it's too late, but it's almost two years since I became an SE, so this time I've reflected on the code I wrote this year, and I've selected the methods that look good (and can be used often) from among the newly added methods from Java 9 to Java 11. This post will be a brief introduction to the selected methods.

In most cases, if you are in an environment where you can use these methods, you must have installed Java 11, so it may not make much sense, but please note that the version from which the method was introduced is written to the right of each method name.

## Stream

I think Stream is the key to Java 8. Java 9 improves various issues with Stream and provides methods that make it easier to use. Therefore, I think that even people who are only familiar with existing for loops can now easily get started.

## Iterate (9)

You may not be able to immediately understand the meaning of the method name `iterate()`, but this method allows you to write Stream processing using a syntax similar to a traditional For statement. In other words, the number of elements in the Stream can be determined by writing "initialization, loop continuation conditions, and counter variable updates." For example, you can write it like this:

```java
// Print 0 through 9
Stream.iterate(0, i -> i < 10, i -> i + 1).forEach(System.out::println);
```

This has the same meaning as the code below.

```java
for (int i = 0; i < 10; i++) {
    System.out.println(i);
}
```

However, since there is no restriction that the initialization value that can be specified with `iterate()` is a number (`T`), you can also do something like the following.

```java
// Print a triangle using A
Stream.iterate("A", s -> s.length() < 10, s -> s + "A").forEach(System.out::println);
```

You can also specify no continuation condition for the loop.

```java
// Print a triangle using A
Stream.iterate("A", s -> s + "A").forEach(System.out::println);
```

If I don't specify a continuation condition, won't it become an infinite loop? That seems to be the case. That's true, but Java 9 also adds a new method that allows you to specify an upper limit on the number of elements in a Stream. That's what I'm going to introduce next.

## takeWhile (9)

I previously wrote that ``you can't stop processing midway through'' as a problem with Stream, but by using the `takeWhile()` method introduced in Java 9, it is now possible to terminate processing midway. In the case of the existing `limit()`, there was a limit of ``the specified number of times'', but the difference here is that you can specify a Predicate type condition.

```java
// Print up to AAAAAAAAA
Stream.iterate("A", s -> s + "A")
    .takeWhile(s -> s.length() < 10)
    .forEach(System.out::println);
```

Therefore, if you have not written a continuation condition for `iterate()`, it is better to use `takeWhile()` to specify which condition the process will end under.

## dropWhile (9)

As you can guess from its name, `dropWhile()` is a method that has the exact opposite function to `takeWhile()`. This method returns the remaining elements from the Stream, excluding the elements that match the given condition.

```java
// Print starting from AAAAA
Stream.iterate("A", s -> s.length() < 10, s -> s + "A")
    .dropWhile(s -> !s.contains("AAAAA"))
    .forEach(System.out::println);
```

## ofNullable (9)

In Java 1.8 Stream, in order to add a Null element, it was necessary to first check whether the element was Null, and if it was Null, call `Stream.empty()`. This is the usual Java null check. For example:

```java
// Collect a Stream while checking elements for null
keyList.stream()
    .flatMap(k -> {
        Object value = map.get(k);
        return value != null ? Stream.of(value) : Stream.empty();
    })
    .collect(Collectors.toList());
```

This can be written in simpler code in Java 9. It doesn't feel much different from `ofNullable()` in `Optional`.

```java
keyList.stream()
  .flatMap(k -> Stream.ofNullable(map.get(k)))
  .collect(Collectors.toList());
```

## Collectors

The `Collectors` API provides a Collector for aggregating Stream elements, but the changes here seem to be mostly syntactic sugar. It mainly allows you to do things that could only be done with Stream, and to write concise code that would be quite long if you only used existing Collectors.

## filtering (9)

The same processing as `filter()` of `Stream` can now be performed with `Collector`. I think which one to use is a matter of preference, but it seems like it would be possible to standardize `Collector` itself.

```java
// List containing 0 through 9
List<Integer> numbers = Stream.iterate(0, i -> i < 10, i -> i + 1).collect(Collectors.toList());

// Stream.filter()
numbers.stream()
    .filter(e -> e > 5)
    .collect(Collectors.toList()); // 6, 7, 8, 9

// Collectors.filtering()
numbers.stream()
    .collect(Collectors.filtering(e -> e > 5, Collectors.toList()));  // 6, 7, 8, 9
```

## flatMapping (9)

As you can probably guess from the name, this is something that allows you to do flat mapping of elements when you change it to a Collection with `Collectors`. Specifically, please refer to the sample code below.

For example, suppose you have a class like the following:

```java
public class Selling {
   String clientName;
   List<Product> products;
}
public class Product {
   String name;
   int value;
}
```

And what should I do if I want to make this Selling list into a Map with clientName as the key and products as the value? For example, you can consider the following methods.

```java
Map<String, List<List<Product>>> result = operations.stream()
                .collect(Collectors.groupingBy(Selling::getClientName, Collectors.mapping(Selling::getProducts, Collectors.toList())));
```

However, the problem is that we end up putting `List<Product>` further into the List. This is inconsistent with the original purpose, causes unnecessary processing, and is inconvenient when exporting the value.If you want to change this to the form `Map<String, List<Product>>`, you can use the following method. So you're creating your own Collector.

```java
Map<String, List<Product>> result = operations.stream()
    .collect(Collectors.groupingBy(Selling::getClientName, Collectors.mapping(Selling::getProducts,
        Collector.of(ArrayList::new, List::addAll, (x, y) -> {
            x.addAll(y);
            return x;
        }))));
```

However, I think that creating a self-made Collector like this every time is not a very efficient method. Also, if you don't use your own Collector regularly, it may be a little difficult to understand just by looking at the code. So, if you change this with the newly added `flatMapping()`, it will look like the following. It's more concise.

```java
Map<String, List<Product>> result = operations.stream()
    .collect(Collectors.groupingBy(Selling::getClientName, Collectors.flatMapping(selling -> selling.getProducts().stream(), Collectors.toList())));
```

## toUnmodifiable (10)

Java 10 adds the following three methods to `Collectors`.

- `toUnmodifiableList()`
- `toUnmodifiableSet()`
- `toUnmodifiableMap()`

These methods allow you to easily (and with much shorter code) create Unmodifiable Collections without having to call an existing `Collections`.

```java
// Collections.unmodifiableList
List<Integer> collectionsUnmodifiable = Collections.unmodifiableList(Stream.iterate(0, i -> i < 10, i -> i + 1).collect(Collectors.toList()));

// Collectors.toUnmodifiableList
List<Integer> collectionsUnmodifiable = Stream.iterate(0, i -> i < 10, i -> i + 1).collect(Collectors.toUnmodifiableList());
```

The arguments are the same as the existing `toList()`, `toSet()`, and `toMap()` (only for `toMap()`, you need to specify the Key and Value mapping), so you can use it in the same way as existing methods.

## Collections

New methods in the Collections API allow for a fairly modern writing style. If languages ​​like Kotlin have tried to avoid the verbosity of Java, the new methods added to Java seem to have embraced it in a way that suits Java even more. (Actually, maybe that was the only way...)

## Factory Method (9)

In Java 9, Collections can now be created using [factory methods](https://en.wikipedia.org/wiki/Factory_method_pattern). In terms of usage, it feels similar to the existing `Arrays.asList()`.

```java
// Create a List
List<String> list = List.of("A", "B", "C");

// Create a Set
Set<Integer> set = Set.of(1, 2, 3);
```

In the case of Map, you can create an instance by arranging keys and values in order, but you can also define entries.

```java
// Define entries as key-value pairs
Map<String, String> map = Map.of("foo", "a", "bar", "b", "baz", "c");

// Define an entry
Map<String, String> map = Map.ofEntries(
  Map.entry("foo", "a"),
  Map.entry("bar", "b"),
  Map.entry("baz", "c"));
```

A characteristic of Collections created using these factory methods is that they are Unmodifiable objects from the beginning. So, for example, you can use it to define constants in a field as a Collection when starting an application. In other words, it can replace existing code such as:

```java
// Most basic approach
Set<String> set = new HashSet<>();
set.add("foo");
set.add("bar");
set.add("baz");
set = Collections.unmodifiableSet(set);

// Double-brace initialization 
Set<String> set = Collections.unmodifiableSet(new HashSet<String>() {
    {
        add("foo");
        add("bar");
        add("baz");
    }
});
```

Also, the Collection created using this factory method has the following characteristics, so it is important to use it as necessary.

- Become Immutable (Unmodifiable)
- Null elements cannot be specified
- If the element is Serializable, the Collection will also be Serializable

### copyOf (10)

A method called `copyOf()` has been added to List, Set, and Map. You can make an Unmodifiable copy by passing each Collection as an argument.

```java
// Source list
List<String> original = ...

// Copy it
List<String> copy = List.copyOf(original);
```

## Optional

Is Optional actively used? In my case, I rarely use Optional myself other than what Stream returns. Since there are many restrictions, I feel that it is difficult to use unless complex null checking is required. However, the methods added in 9 and 10 have made it quite convenient to use, so it may be useful once in a while.

## or (9)

This method is executed when the content of Optional is Null. What is different from the existing `orElse()` and `orElseGet()` is that this returns another Optional instead of the contents of the Optional. It takes Supplier as an argument.

```java
String string = null;
Optional<String> optional = Optional.ofNullable(string);
System.out.println(optional.or(() -> Optional.of("default")).get()); // "default"
```

## orElseThrow (10)

This is a branch that throws an exception if the content of Optional is Null. Null Optional originally throws `NoSuchElementException`, but you can use this if you want to throw a customized exception based on business logic etc. It takes Supplier as an argument.

```java
String string = null;
Optional<String> optional = Optional.ofNullable(string);
String throwing = optional.orElseThrow(RuntimeException::new); // RuntimeException
```

## ifPresentOrElse (9)

This method allows branching by specifying two actions depending on whether the content of Optional is Null or not. By specifying Consumer as the first argument, write the process to be performed when the content is not Null, and as the second argument, write the process to be performed when the content is Null by specifying Runnable.

```java
Optional<String> hasValue = Optional.of("proper value");
hasValue.ifPresentOrElse(v -> System.out.println("the value is " + v), () -> System.out.println("there is no value")); // the value is proper value

Optional<String> hasNoValue = Optional.empty();
hasNoValue.ifPresentOrElse(v -> System.out.println("the value is " + v), () -> System.out.println("there is no value")); // there is no value
```

## stream (9)

This method changes Optional to Steam with one element or Null (`Stream.empty()`). Originally, it was Optional when getting elements from Stream, so it's only natural that a method like this was added. Is there any point in changing it to Stream even though there are many elements and only one? Since it can be combined with other Streams, there seems to be room for various uses.

```java
Optional<String> optional = Optional.of("value");
Stream<String> stream = optional.stream();
```

## StringFor the String API, there have been quite a few changes, mainly in Java 11. Not only web applications but also modern applications often handle strings, so this kind of change is a welcome change.

## repeat (11)

Repeats a string a specified number of times. If you want to simply repeat the same string, this method is better because it can be easily used without StringBuilder or StrinbBuffer.

```java
String a10 = "A".repeat(10); // "AAAAAAAAAA"
```

## strip (11)

Up until now, `trim()` has been used in many cases to exclude spaces before and after a string, but from Java 11, `strip()` has been added and can be used in place of `trim()`. The difference between these two is that the "blank" defined in each method is different. `trim()` did not take Unicode into consideration, so it only supported half-width spaces, but `strip()` targets all whitespace specified in Unicode, so it can also support full-width spaces and line breaks. The `Character.isWhitespace()` method determines which characters are treated as whitespace, so please refer to [that JavaDoc](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/Character.html#isWhitespace(char)).

```java
String stripped = "\n  hello world  \u2005".strip(); // "hello world"
```

Also, `strip()` removes all leading and trailing spaces, but if you want to remove just one side of the string, you can also use `stripLeading()`, which only removes from the front, or `stripTrailing()`, which removes only from the end.

```java
String stripLeading = "\n  hello world  \u2005".stripLeading(); // "hello world   "
String stripTrailing = "\n  hello world  \u2005".stripTrailing(); // "\n  hello world"
```

I think there are enough reasons to use `strip()` just from the explanation so far, but there is actually one more reason. It's performance. In terms of performance, `strip()` is said to be [5 times faster](https://stackoverflow.com/questions/53640184/why-is-string-strip-5-times-faster-than-string-trim-for-blank-string-in-java) than `trim()`, so `strip()` should be used instead of `trim()` if possible.

## isBlank (11)

There is already a method called `isEmpty()`, but the difference between this method and `isBlank()` is similar to the relationship between `trim()` and `strip()`. Similarly, compared to `isEmpty()`, `isBlank()` supports Unicode, so it can support Whitespace in more cases and has better performance.

```java
boolean isEmpty = "\n    \u2005".isEmpty(); // false
boolean isBlank = "\n    \u2005".isBlank(); // true
```

## lines (11)

Returns `Stream<String>`, which is a string divided based on line feed codes (`\n`, `\r`, `\r\n`).

```java
String multipleLine = "first\nsecond\nthird";
long lines = multipleLine.lines().filter(String::isBlank).count(); // 3
```

## Predicate.not (11)

This is a method to determine whether the result of Predicate defined in Lambda or Method Reference is False. It is simply a negation of true, but since the argument of this method is a Predicate, the advantage is that it can be expressed more simply using Lambda or Method Reference.

```java
// Using a negated conditional expression
list.stream()                       
    .filter(m -> !m.isPrepared())
    .collect(Collectors.toList());

// Using Predicate.not()
list.stream()                          
    .filter(Predicate.not(Man::isPrepared))
    .collect(Collectors.toList());
```

## Finally

When the next LTS, Java 17, is released in 2021, I think many of the sites currently using Java 11 will migrate to Java 17. Versions 12 to 16 include various APIs, functions, and JVM improvements, which have already been introduced on many blogs, but we do not fully understand what changes will be made to the existing APIs. So, in line with the release of Java 17, I would like to organize and introduce the new methods 12 to 17 once again. You can learn a lot just by doing this, and it feels like you'll be learning more techniques that you can use in your work.

This also marks the end of this year's posting. It's been a tough year, but we managed to reach the end of the year. During that time, many people visited this blog. This is a blog written by a budding engineer who is only at a junior level, so it may not be very useful for gathering information, but I appreciate you taking the time to read my post. Starting next year, I would like to collect more interesting and better information and post it on my blog.

See you soon!
