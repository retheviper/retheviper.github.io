---
title: "Using Stream Properly"
date: 2020-04-06
translationKey: "posts/java-stream"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - stream
  - java
---

Personally, I am not deeply versed in functional programming, but I do like Java 1.8's Stream API. I also like APIs such as Lambda and Optional, but it would not be an exaggeration to say that part of the reason I studied for my Java certification was to learn more about this Stream API.

While studying Stream, I started having questions. It is certainly a good API, but it is also quite different from traditional Java APIs. It is not hard to infer that there must be reasons for introducing it into Java, but what about the downside? And is the way I use Stream actually correct?

In this post, I want to share what I found while researching those questions on my own. Rather than presenting a single correct answer, please read this as one possible perspective.

## Is Stream a Silver Bullet?

The first question is simple: is Stream a silver bullet? In other words, can we rewrite every existing piece of code with Stream without any problem? And should we try to write all future code with Stream whenever possible?

When a new API appears and can do the same job as existing code, there is usually some reason for it. Java's NIO was one example. It was introduced to address limitations in traditional I/O, which could not fully use OS kernel features. Even so, NIO was not better than the old I/O in every situation. I suspected Stream might be similar.

The conclusion is that we do not need to rewrite everything with Stream. Here is why.

## It can be slower

Java 1.5 introduced the enhanced `for` loop in addition to the traditional `for` loop. Then Java 1.8 added `forEach()` together with Stream. However, `forEach()` and Stream can be slower than the enhanced `for` loop. Some benchmarks even reported that Stream was slower than the enhanced `for` loop even when using Parallel. The reason is simple: Stream does more work internally. Even converting an array into a Stream adds wrapping overhead.

The performance gap can be larger when working with primitive types than when working with objects. So there is no need to force every array into a Stream. It is better to use Stream or `forEach()` only when the code can still stay stable and clear. For primitive types, using the dedicated stream types such as `IntStream` is usually better.

There was also a view that the JVM had been optimized for the traditional `for` loop for a long time, and that Stream, having arrived much later, was still less optimized. That was written when Java 1.8 had just been released, so I do wonder how much of it still holds now that the version has moved forward quite a bit. Even so, I still think the traditional `for` loop is probably faster in many cases.

## It is hard to stop halfway

Unlike a normal `for` loop, Stream processing does not let you use `continue`, `break`, or `return` to skip part of the work or stop in the middle. Stream is basically designed with the assumption that it processes every element. So you need to choose the right Stream method for the job. For example, the following `for`-loop patterns map to these methods:

- When you want to process only matching elements
  - `filter()`
- When you want to collect into a Collection
  - `collect()`
- When you want to retrieve one element
  - `findAny()` / `findFirst()`

## You cannot use a loop variable

With the traditional `for` loop, you can use a loop variable to count how many times the loop has run. But with Stream, you cannot use a loop variable directly. Even if you place a variable outside the Stream, it will result in a compilation error.

If you want an integer loop variable, use `IntStream`. If you want to skip items based on an index, use `skip()`. Of course, in cases like this, the ordinary `for` loop is usually the better choice.

## The functional premise

There is a way to fake a loop variable inside Stream by using an external variable, for example with `AtomicInteger`. But if you have to go that far, there is little reason to use Stream at all, and it no longer fits the goals of functional programming.

One of the core concepts of functional programming is immutability. I wrote about [immutability before](../java-thoughts-of-immutable/), and the important point here is that data does not change. If the data does not change, processing happens by keeping the original data as-is and working on a copy instead.

That is also why the result of processing with Stream becomes a new instance. When iterating over a `List`, you can edit the original elements. But when processing with Stream, the result becomes a new `List` made up of changed elements. Even if you modify the original elements in an intermediate operation, the Stream is closed after the terminal operation and cannot be reused. That keeps the original data unchanged.

So whether Stream is appropriate depends first on what you want to do with the original data. Of course, even if you do not use Stream, there are many cases where a functional style is still possible. That said, it may be time to study functional programming first...

## Making Better Use of Stream

The next question is how to use Stream correctly and efficiently. Stream is not reusable once a terminal operation has been performed, so how can we apply different processing to the same data?

Another concern I had, maybe not just for me, is that APIs which support method chaining can also encourage inefficient code. For example, you can convert a `Collection` or array to Stream and then use `forEach()`, but `Collection` already has `forEach()` without Stream. In such cases, `Collection.forEach()` feels like the obvious choice, but I was less sure about other scenarios.

So I looked into those two questions as well.

## Reuse

Stream can perform intermediate operations repeatedly, but once a terminal operation runs, it is closed and cannot be reused. That is because Stream is meant for processing data, not storing it.

Still, there are times when you want to use Stream on the same data more than once with different processing. In those cases, you can store the necessary data in a `Collection` first and convert it to Stream again whenever needed. For arrays, there is `Arrays.stream()` or `Stream.of()`, and for `Collection`, there is `stream()`. For example:

```java
// Collect the needed data in a List first
List<String> names = 
  Stream.of("Eric", "Elena", "Java")
  .filter(name -> name.contains("a"))
  .collect(Collectors.toList());

// 1st Stream
Optional<String> firstElement = names.stream().findFirst();
// 2nd Stream
Optional<String> anyElement = names.stream().findAny();
```

This is an example of collecting the data in a `List` first and then converting it back to Stream when needed. If the data is already in a `Collection` or array, you can call `stream()` as needed and perform different processing each time. You can also insert `peek()` to send data into a different `Collection`. Strictly speaking, this is more about how to create a Stream than about reuse, but the point is that one source of data can still produce multiple processing results.

## Write it shorter

As mentioned earlier, Stream is an API that supports method chaining, so it can easily lead to inefficient code. I collected a few more efficient alternatives by use case. I mainly use Eclipse, but in IntelliJ these are the kinds of suggestions it would usually offer.

- Use `Collection` methods

```java
// Collection forEach
collection.stream().forEach() 
  → collection.forEach()
  
// Collection to array
collection.stream().toArray() 
  → collection.toArray()
```

- Create a Stream

```java
// From array to Stream
Arrays.asList().stream() 
  → Arrays.stream() / Stream.of()

// Create an empty Stream
Collections.emptyList().stream() 
  → Stream.empty()

// Create an array with a range
IntStream.range(expr1, expr2).mapToObj(x -> array[x]) 
  → Arrays.stream(array, expr1, expr2)

// Create a Stream with a range
Collection.nCopies(count, ...) 
  → Stream.generate().limit(count)
```

- Check elements

```java
// Determine whether any matching element exists (1)
stream.filter().findFirst().isPresent() 
  → stream.anyMatch()

// Determine whether any matching element exists (2)
stream.map().anyMatch(Boolean::booleanValue) 
  → stream.anyMatch()

// Determine whether none of the elements match
!stream.anyMatch() 
  → stream.noneMatch()

// Determine whether all elements match
!stream.anyMatch(x -> !(...)) 
  → stream.allMatch()

// Sort and find the first value
stream.sorted(comparator).findFirst() 
  → Stream.min(comparator)
```

- Collect elements

```java
// Count elements
stream.collect(counting()) 
  → stream.count()

// Find the largest element
stream.collect(maxBy()) 
  → stream.max()

// Map elements into a different object
stream.collect(mapping()) 
  → stream.map().collect()

// Combine elements into one
stream.collect(reducing()) 
  → stream.reduce()

// Sum numeric elements
stream.collect(summingInt()) 
  → stream.mapToInt().sum()
```

- Process elements

```java
// Only change the element's state
stream.map(x -> {...; return x;}) 
  → stream.peek(x -> ...)
```

## Closing Thoughts

Even if you are not interested in functional programming, Stream itself is a very attractive API, and I think it is worth trying. When Java 1.8 was released, there were many complaints that it was slower and harder to read, but enough time has passed and Java has already moved on to much newer versions. It is probably time to try a more modern way of writing code with Stream.

Another benefit is that Stream lets you experience functional programming a little. Of course, Stream is not a perfect example of functional programming, but simply being able to experience a style beyond object-oriented programming has value. More than a few years have passed since functional programming concepts became mainstream, and since the programming world is always changing, it helps to keep an eye on current trends.

See you next time!
