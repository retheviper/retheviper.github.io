---
title: "About Oracle JavaSE 8 Gold"
date: 2020-03-29
categories: 
  - java
image: "../../images/java.webp"
tags:
  - ocjp
  - certification
  - java
---
This time I took the Java SE 8 Gold exam. I took Silver at the end of September last year, so it's been about half a year since I took the exam. Previously, when I took the Silver exam, I wrote about my impressions on [post](../java-se-8-silver/), so I would like to write about my impressions after taking the Gold exam this time as well.

## Generics

The problem with Generics was often when the type was not clear. For example, what to do if you want to receive only classes that inherit from a certain class as arguments. This is a problem that becomes difficult if you don't understand the difference between `<? super X>` and `<? extends X>`.

Also, the problem with Generics is that `Comparator<T>` and `Comparable<T>` have also appeared, and it becomes a difficult problem unless you understand how they are different and where they are needed. Personally, this was an API I didn't use much, so this was a problem I struggled with at first. The more you understand, the better, but I'm not sure if you'll use it much in the future...But I feel like it would be useful to know, so it's an API I want to remember.

## Lambda

Lambda, which is an important feature of Java 1.8, has many problems. Lambda doesn't just make it easier to write methods; it's also an important API that is related to other important APIs such as functional interfaces and Streams that can be used, so if you don't understand this, you can't really understand Java 1.8.

I felt that many of the problems related to Lambda could not be solved without understanding the functional interface conditions (SAM) and the types of interfaces provided by the java.util.function package. There are also questions regarding Method Reference. If you have a solid understanding of Lambda, you'll understand that it's just written differently, but if you don't, you might not even be able to read the code.

## Stream

Stream has a lot of problems. It is one of the important APIs added in Java 1.8, and its character is slightly different from previous Java APIs, so it was a great learning experience for me, who had only been using it based on feeling. Up until now, I had only used stream() and collect() at most, but after this exam, I feel like there will be more places where I can use Stream.

Personally, I don't think it's necessary to change to Stream if your existing code is working without problems, but I think it's a nice API that lets you get a taste of functional programming. For example, the questions asked about the difference between intermediate operations and terminal operations, and how to use each method, and compared to meaningless questions like ``Choose the correct method for API from the following'' when you list the names of those methods, I liked the feeling that the questions were difficult to solve unless you understood the characteristics of the API itself and how to use it. I was especially happy to see good methods like peek(), sorted(), and Collectors.groupingBy().

## I/O

Basically, there were many problems related to NIO, which was added from Java 1.7. There were also many questions regarding methods such as lines(), list(), and walk(), probably because this is also connected to Stream. Also, I think it was a good thing because there was also an issue with try-with-resource, which also appeared in 1.7.

In my case, I mostly use NIO's Files class for file I/O, so I'm familiar with it, so I think the fact that it provides a safe API without having to go through the trouble of calling close() will have a positive impact on my future habits.

Personally, I was happy because I had created a class that inherited AutoCloseable at work.

## Thread

There seemed to be a relatively large number of problems related to concurrency. It was a good opportunity to learn important APIs such as AtomicInteger and ExecutorService, but since it covers Runnable and Callable, I thought it would have been better to cover CompletableFuture as well. CyclicBarrier was released, but it doesn't seem to be used much these days. Threading wasn't that difficult, and I felt that many of the problems could be solved if you properly memorized the signatures of ForkJoinPool, CyclicBarrier, Runnable, and Callable.

## Temporal

There were a few issues with APIs such as LocalDateTime, but the percentage was low. This API was also covered in Silver, so I felt there were many questions here about formatters and exception patterns. ZoneId also appeared, but it did not feel especially important.

## Locale

This percentage is also low. However, knowing how to read properties is necessary not only when dealing with Locale, but also when dealing with environment variables, etc., so I think it is worth remembering. When it comes to questions about API signatures, there is plenty of information available on the internet, so is there really a need to memorize them? But...

## OthersThere were some problems with JDBC, but since it is not used much these days, the number of problems was small. In my case, I use Mybatis and JPA for DB processing in Java, so I don't bother using JDBC, but I think it's enough to understand that this is basically how Java accesses and processes DBs, so I think the problem was appropriate.

Also, up to Silver, I feel like there are a lot of questions asking whether you have properly memorized the Java API, but from Gold, there were some questions about class design, including questions about encapsulation and how to implement singleton classes, and questions about recursion processing using Files.walk(), etc., so I can't imagine that the people taking the Gold exam don't have that kind of knowledge.

One thing I regret is that there is no mention of Optional, which is used in a similar way (method chaining) and can also be used when acquiring elements with Stream. It was introduced in Java 1.8 like the others, so why? That's what it feels like.

## Finally

Last year, the Java SE 11 qualification also appeared, so I thought it would have been better to choose that from the beginning, but according to the exam introduction on Oracle's official website, it seems that the content of the exam hasn't changed much other than the module system introduced with Java 9.

Both 8 and 11 are LTS versions, so when the next LTS, 17, is released, perhaps the entitlements will also be updated. Personally, I think the new features introduced in grades 12 through 14 are more convenient, so if you're taking the exam for study purposes like me and already have a grade 8 qualification, upgrading to grade 11 may not be of much benefit. Of course, 11 is better for those who are about to obtain the qualification for the first time. I didn't even know that, so I took the 8th grade anyway...

In summary, I am happy to have obtained the qualification, but most of all I am satisfied because the Java qualification, which I started to study, provided me with more useful knowledge in my work than I expected. Many of the questions can be solved simply by memorizing the patterns well, but if you want to deepen your understanding of Java APIs, I don't think you'll be disappointed in taking the exam. The test fee is high, but...
