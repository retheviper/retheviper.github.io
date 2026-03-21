---
title: "Another Story About Loops"
date: 2020-11-30
categories: 
  - java
image: "../../images/java.webp"
tags:
  - stream
  - java
  - collection
  - foreach
---

Although Java is originally a procedural language, it has cleverly adopted the characteristics of a functional language and allowed them to coexist within the language. Personally, I admire functional programming, so I like to use Stream and Lambda in Java, and I'm also experimenting with Kotlin and Spring WebFlux.

Ever since Java 1.8, people have asked whether Streams can really replace every `for` loop. The reasons usually given for keeping `for` loops are performance, readability, and debugging difficulty. In other words, Streams do more complex work internally, can make exception paths harder to follow, and still rely on concepts such as [method chaining](https://en.wikipedia.org/wiki/Method_chaining) and lambdas that not everyone is comfortable with.

For the above reasons, I usually follow the rule that List = ArrayList and the loop is an extended For statement (occasionally, when creating a new instance by manipulating the elements of the List, use a Stream), but I suddenly started to question whether this is really a good idea. Java has also been updated to version 16, so isn't it a good time to switch to functional programming? Or maybe you want to test whether what you know is correct.

So, although it feels like it's too late, I tried various verifications and thought about it as a small benchmark (actually, I just wanted to run a benchmark).

## Loop method

This is a late introduction, but since this post is about a recent topic, I would like to talk about the details of the four loop statements related to Collection.

In many cases, I think that the method of looping through Collections and arrays is used as shown in the table below.

| Types | Scenes to use |
|---|---|
| For | When an index is needed |
| Extend For | If you don't need to do anything else |
| Iterator | Basically not used |
| forEach() | Basically not used |

I think the main basis for these choices is performance. Readability matters too, but performance is the easiest thing to measure directly. In an application that already satisfies its requirements, performance is often the only thing refactoring can visibly improve. If two implementations do the same work, the faster one is naturally attractive.

So, when I first learned about loop processing, I used the traditional For statement and While, but later I was taught that it is better to use the Extended For statement when working with Collections and arrays, but at that time I was given the rationale that ``For and Extended For are not much different in terms of performance, and Extended For is guaranteed to always loop for the number of elements.'' After all, it's a matter of thinking in terms of performance and then considering other things. It made sense to me, so I believed it myself and have been using extended For statements until now.

However, I had never really verified that assumption myself. Online, you can find explanations that an extended `for` statement is just a nicer form of an Iterator, or simply [syntactic sugar](https://en.wikipedia.org/wiki/Syntactic_sugar). 

The remaining question is `Stream` and `forEach()`. They have been around for a long time, but I still do not feel they are used as widely as classic loops. One reason is the perception that they are slower. Other reasons are that some developers do not see a strong reason to use them, or simply find them harder to read. Even when Java 1.8 first appeared, many benchmark posts concluded that the older style still had an advantage, at least for raw performance. Java has improved a lot since then, but the common understanding still seems to be that Streams and `forEach()` carry some unavoidable overhead because of their design.

However, it's probably not a good idea to think that way just because someone told you so. Also, as mentioned above, Java has already been updated to version 16, and most of the changes have been the addition of new features, but behind the scenes there may have been some invisible improvement in JVM or compiler tuning. It's a matter of whether you're used to writing functional code or not, but if it improves performance, there's no reason not to use a more modern method. Also, I think it is necessary to verify in advance whether the extended For statement is really suitable in all cases.

For the above reasons, I would like to first introduce the four loops used in verification and their benchmarks.

## For statement

First is the traditional form of the `for` statement. Some people call it `C-style`. It is the most basic and familiar form, but it can also feel old-fashioned. In many modern languages, this style is either discouraged or unavailable. The basic form is as follows.

```java
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}
```

As a micro-optimization, the length of the collection or array to be looped may be declared in advance. By doing this, there is no need to calculate the size of the Collection or array that is the target of the loop each time, so it is said that there is a slight performance advantage. (There is also talk that the compiler does this level of optimization on its own.)

```java
int size = list.size();
for (int i = 0; i < size; i++) {
    list.get(i);
}
```

The nice thing about this traditional For statement is that it works index-based, so you can do whatever you want with an index. For example, there may be the following cases:

```java
// Process only even-numbered indexes
for (int i = 0; i < list.size(); i += 2) {
    System.out.println(list.get(i));
}

// Find the index of elements that match a condition
for (int i = 0; i < list.size(); i++) {
    if (list.get(i).length() == 10) {
        System.out.println(i);
    }
}

// Compare the previous and current elements
for (int i = 1; i < list.size(); i += 2) {
    System.out.println("Length at index " + (i - 1) + ": " + list.get(i - 1).length() + ", length at index " + i + ": " + list.get(i).length());
}
```

However, a classic `for` loop does not guarantee that the index you wrote is actually safe. You might accidentally use `i - 1` when `i` starts at 0, or let `i` go past the size of the target collection and trigger an exception. Indexes can also turn into [magic numbers](https://en.wikipedia.org/wiki/Magic_number_(programming)), which hurts readability and increases the chance of bugs. If you really need index-based processing, you need to write it carefully.

## Extended For statement

This is a so-called `for-each` statement. I don't think there is anything easier to understand and safer than this to cycle through all the elements in a Colleciton/array.

```java
for (String element : list) {
    System.out.println(element);
}
```

Recently, this style has become standard not only in Java but in many other languages as well, though the syntax differs a little. Instead of thinking in terms of indexes, you limit the scope of the loop to the elements in the target collection or array. That usually leads to simpler and safer code.

Compared to traditional For statements, the problem with extended For statements is that indexes cannot be used in extended For statements. However, this does not mean that there is no way at all. If you really want to use an index in an extended For statement, you can either declare a constant outside the loop, or use `indexOf()` or `Collections.binarySearch()`, which can be used with Collections.

```java
// Using a counter
int i = 0;
for (String element : list) {
    System.out.println(element + " index: " + i);
    i++;
}

// Using indexOf()
for (String element : list) {
    System.out.println(element + " index: " + list.indexOf(element));
}

// Using Collections.binarySearch()
for (String element : list) {
    System.out.println(element + " index: " + Collections.binarySearch(values, value));
}
```

However, using `indexOf()` inside a loop is not a very good choice. The following is an implementation of `ArrayList.indexOf()`, but in the end it ends up searching for the index while looping through the Collection, so it is essentially a double loop. Therefore, if you absolutely need an index, you should use constants as much as possible or use the traditional For statement.

```java
// ArrayList.indexOf()
public int indexOf(Object o) {
    return indexOfRange(o, 0, size);
}

int indexOfRange(Object o, int start, int end) {
    Object[] es = elementData;
    if (o == null) {
        for (int i = start; i < end; i++) {
            if (es[i] == null) {
                return i;
            }
        }
    } else {
        for (int i = start; i < end; i++) {
            if (o.equals(es[i])) {
                return i;
            }
        }
    }
    return -1;
}
```

Note that even when using `binarySearch()` of `Collections`, the index is still searched while looping. Below is its implementation.

```java
// Collections.binarySearch()
public static <T> int binarySearch(List<? extends Comparable<? super T>> list, T key) {
    if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
        return Collections.indexedBinarySearch(list, key);
    else
        return Collections.iteratorBinarySearch(list, key);
}

private static <T> int indexedBinarySearch(List<? extends Comparable<? super T>> list, T key) {
    int low = 0;
    int high = list.size()-1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        Comparable<? super T> midVal = list.get(mid);
        int cmp = midVal.compareTo(key);

        if (cmp < 0)
            low = mid + 1;
        else if (cmp > 0)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found
}

private static <T> int iteratorBinarySearch(List<? extends Comparable<? super T>> list, T key) {
    int low = 0;
    int high = list.size()-1;
    ListIterator<? extends Comparable<? super T>> i = list.listIterator();

    while (low <= high) {
        int mid = (low + high) >>> 1;
        Comparable<? super T> midVal = get(i, mid);
        int cmp = midVal.compareTo(key);

        if (cmp < 0)
            low = mid + 1;
        else if (cmp > 0)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found
}
```

## Iterator

Iterator is something that I personally don't get used to (I don't want to use). Since any Collection can be extracted as an Iterator, it feels like the Iterator is more important than the Collection, and this is because we are correcting the standard writing style. At least with the extended For, it is clear which elements from which Collections are extracted and used, but with Iterator it is not clear.

Anyway, this Iterator has the feature of looping both for and while.

```java
// Using for
for (Iterator iterator = values.iterator(); iterator.hasNext(); ) {
    System.out.println(iterator.next());
}

// Using while
Iterator iterator = values.iterator();
while (iterator.hasNext()) {
  System.out.println(iterator.next());
}
```

The problem with using Iteratror is that it's not very intuitive to use. For example, consider the following example. It may be easy to misunderstand that `getFoo()` and `getBar()` are called from the same object.

```java
for (Iterator iterator = list.iterator(); iterator.hasNext(); ) {
    System.out.println(iterator.next().getFoo());
    System.out.println(iterator.next().getBar()); // Be careful
}
```

Interestingly, the bytecode for the extended For statement is code that uses an Iterator. Therefore, at least the extended For statement may be said to be a more advanced form than Iterator.

## forEach()

forEach() is a modern way of writing. It's not much different from the extended For statement, but it has the advantage of being able to use Lambda and method references. Also, like Kotlin's scope function, it may be good in the sense that the scope of processing is clear. Above all, I like that the cord is short.

```java
list.forEach(System.out::println)
```

The implementation has a simple structure of executing Lambda inside an extended For statement. Therefore, simply thinking about it, the performance may be inferior to the extended For statement. Below is the implementation of Iterable.

```java
// Iterable.forEach()
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```

However, in the case of ArrayList, the implementation is very different. Therefore, performance may vary significantly. Below is its implementation.

```java
// ArrayList.forEach()
public void forEach(Consumer<? super E> action) {
    Objects.requireNonNull(action);
    final int expectedModCount = modCount;
    final Object[] es = elementData;
    final int size = this.size;
    for (int i = 0; modCount == expectedModCount && i < size; i++)
        action.accept(elementAt(es, i));
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}

static <E> E elementAt(Object[] es, int index) {
    return (E) es[index];
}
```

## Verify with benchmark

Once again, I created a simple benchmark using JMH. Actually, I thought that if I declared it as a static final field, that object would be reused in all benchmarks, but apparently that was not the case. So this time, I properly initialized the field using the @Setup annotation. The actual code is below.

```java
@State(Scope.Thread)
public class LoopTest {

    private List<String> values;

    @Setup
    public void init() {
        final DecimalFormat format = new DecimalFormat("0000000");
        values = IntStream.rangeClosed(0, 9999999).mapToObj(format::format).collect(Collectors.toList());
    }

    @Benchmark
    public void indexLoop(Blackhole bh) {
        final int length = values.size();
        for (int i = 0; i < length; i++) {
            bh.consume(values.get(i));
        }
    }

    @Benchmark
    public void iteratorLoopFor(Blackhole bh) {
        for (Iterator iterator = values.iterator(); iterator.hasNext(); ) {
            bh.consume(iterator.next());
        }
    }

    @Benchmark
    public void iteratorLoopWhile(Blackhole bh) {
        final Iterator iterator = values.iterator();
        while (iterator.hasNext()) {
            bh.consume(iterator.next());
        }
    }

    @Benchmark
    public void extendedLoop(Blackhole bh) {
        for (String value : values) {
            bh.consume(value);
        }
    }

    @Benchmark
    public void forEachLoop(Blackhole bh) {
        values.forEach(bh::consume);
    }
}
```

And the benchmark results are as follows.

```dos
Benchmark                    Mode  Cnt   Score   Error  Units
LoopTest.indexLoop          thrpt   25  27.737 ± 0.475  ops/s
LoopTest.iteratorLoopFor    thrpt   25  26.968 ± 0.556  ops/s
LoopTest.iteratorLoopWhile  thrpt   25  27.250 ± 0.557  ops/s
LoopTest.extendedLoop       thrpt   25  13.186 ± 0.152  ops/s
LoopTest.forEachLoop        thrpt   25  12.479 ± 0.104  ops/s
```

As expected, the four loop styles produced different results. At least in this benchmark, the traditional `for` statement had the best performance. The gap is large enough that it makes me question the common advice that the performance difference is small enough to prefer the extended `for` statement by default.

But does that really mean we should always choose the fastest one?

## What I want to think about

If the results are the same, it is natural to prefer the faster implementation. At the company level, performance is directly tied to cost. But performance is not the only consideration in modern software. For example, if you built a web application in C or C++ only for speed, productivity would likely drop. If you optimize only for speed and ignore readability and maintainability, you may end up with [spaghetti code](https://en.wikipedia.org/wiki/Spaghetti_code).

So, it's true that there are many other factors to consider when developing an application, not just performance. Examples include Readability, Error-proneness, and Capability. Up until now, we have only talked about performance, but how about comparing the four loops from these perspectives?

## Thinking in terms of readability and possibility of errors

The extended For statement (forEach()) allows you to access the elements of the Collection itself. Conversely, it is possible with For statements and Iterators. So, if you want to do something about only the elements in a Collection or array that match a certain condition, it seems like you should use a For statement or Iterator rather than an extended For statement.

However, if you change your perspective, side effects may occur due to changes in the original object itself. In this case, being able to directly manipulate the original object is more of a disadvantage than an advantage. Therefore, the correct answer may be to extract only the elements that match the conditions from the given Collection/array and generate a new Collection/array instance. The extended For statement (forEach()) can be used to make this easier to understand. For example:

```java
// The original list is modified
public void filterFor(List<String> list) {
    for (int i = 0; i < list.size(); i++) {
        if (list.get(i).length() > 10) {
            list.remove(i);
        }
    }
}

// Does not affect the original list - for statement
public List<String> filterFor(List<String> list) {
    List<String> result = new ArrayList<>();
    for (int i = 0; i < list.size(); i++) {
        String element = list.get(i);
        if (element.length() <= 10) {
            result.add(element);
        }
    }
    return result;
}

// Does not affect the original list - extended for statement
public List<String> filterForEach(List<String> list) {
    List<String> result = new ArrayList<>();
    for (String element : list) {
        if (element.length() <= 10) {
            result.add(element);
        }
    }
    return result;
}

// Does not affect the original list - Stream.forEach()
public List<String> filterStream(List<String> list) {
    return list.stream().filter(element -> element.length() <= 10).collect(Collectors.toList());
}
```

I believe that good code is short and easy to understand. And code that is easy to understand should be less likely to cause bugs, no matter who maintains it. From that perspective, traditional For statements and Iterators may not be used anymore.

## Thinking from the perspective of processing capacity

Processing power is related to performance to some extent. So let's think about it again from a performance perspective. It may also be called compatibility and versatility. What I'm trying to say here is that no matter what kind of collection/array you have, you need to consider an implementation that guarantees a certain level of performance.

Let's say you want to implement a method that takes `List` as an argument and performs some processing in a loop. When choosing which of the four loop patterns I have mentioned so far, you need to take into consideration the fact that you do not know what the implementation class of the argument will be. This is because List is an interface with various implementation classes.

If you first declare List as an argument, the language specification allows any List implementation class. Therefore, the argument that comes in may be [ArrayList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/ArrayList.html), it may be [LinkedList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/LinkedList.html), and in the extreme, we can expect that it may be [AbstractList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/AbstractList.html) customized by the individual. In addition, based on Java 11, there are collection implementation classes that inherit java.util.List, such as [AbstractSequentialList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/AbstractSequentialList.html), [AttributeList](https://docs.oracle.com/en/java/javase/11/docs/api/java.management/javax/management/AttributeList.html), [CopyOnWriteArrayList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CopyOnWriteArrayList.html), [RoleList](https://docs.oracle.com/en/java/javase/11/docs/api/java.management/javax/management/relation/RoleList.html), [RoleUnresolvedList](https://docs.oracle.com/en/java/javase/11/docs/api/java.management/javax/management/relation/RoleUnresolvedList.html), [Stack](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Stack.html), and [Vector](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Vector.html), and all of these can be Lists, so any implementation must support them.

Of course, inheriting a certain interface in Java means that the preconditions and results of the process are clear, so even if the implementation class changes, the result of the process will not change significantly. However, you must first understand that the reason why there are multiple implementation classes for List is that the performance will be biased depending on the purpose of using them. This means that even under the same conditions, processing performance can vary greatly depending on the implementation class. The implementation class of List that is commonly used is ArrayList, but it can be expected that LinkedList will be used in some cases because of its inferior performance for things other than references. If so, what performs well with ArrayList may not necessarily be the same with LinkedList.

This is the reason why it cannot be said that this is definitely advantageous in terms of performance just by looking at the benchmarks conducted above. This is because the test data is created as a List using `Collectors.toList()`, but as you can see in the code below, an ArrayList is always generated.

```java
public static <T> Collector<T, ?, List<T>> toList() {
    return new CollectorImpl<>((Supplier<List<T>>) ArrayList::new, List::add,
                                (left, right) -> { left.addAll(right); return left; },
                                CH_ID);
}
```

So, I decided to also benchmark other implementation classes. However, it is impossible to test all implementation classes of List (in particular, AbstractList and AbstractSequentialList require separate implementation, CopyOnWriteArrayList has no meaning unless it is multi-threaded, RoleList and Vector are rarely used, and I don't think Stack will be used in a loop), so I just checked how it works in the case of LinkedList. Well, it's enough to have just one counterexample that is different from ArrayList.

Fortunately, Collectors can be specified with a Collection implementation via `toCollection()`. Therefore, you can change the List implementation from the benchmark code above by simply making the following modifications.

```java
// In the case of LinkedList
values = IntStream.rangeClosed(0, 9999999).mapToObj(format::format).collect(Collectors.toCollection(LinkedList::new));
```

In the case of LinkedList, performance tends to decrease rapidly as the number of elements increases. Therefore, we reduced the number of elements by two orders of magnitude compared to when using ArrayList and ran the benchmark. The results are below.

```bash
Benchmark                    Mode  Cnt    Score    Error  Units
LoopTest.indexLoop          thrpt   25    0.084 ±  0.005  ops/s
LoopTest.iteratorLoopFor    thrpt   25  854.459 ± 36.771  ops/s
LoopTest.iteratorLoopWhile  thrpt   25  839.233 ± 18.142  ops/s
LoopTest.extendedLoop       thrpt   25  659.999 ± 47.702  ops/s
LoopTest.forEachLoop        thrpt   25  780.463 ± 78.591  ops/s
```

You can see that the result is the exact opposite of ArrayList. In particular, the performance of loops using indexes is so low that it becomes unusable, and the performance of `forEach()` is higher than that of extended For statements, which is a surprising result. Although the numbers in this benchmark cannot be said to be absolute, we can infer from the results that it is dangerous to process all Lists with For statements just because the traditional For loop using an ArrayList index was the fastest. Therefore, we can conclude that it is necessary to choose a method that "gives good performance on average in any implementation class." (I think it's probably an extended For statement)

## lastly

It is difficult to write code that is optimal for all situations, and code written in the past will eventually need to be improved. Even though I don't have a long experience as an engineer, I am still surprised when I look at the code that was created before I joined the company. Although I managed to create something that worked, my mistakes were scattered everywhere, such as duplicated code and unnecessary instance creation.

So sometimes I think it would be a good experience to face the code that I wrote in the past and try to fix it. In particular, like this time, loop processing is one of the basics, but there were many times I just wrote it without really knowing which processing to choose (and without thinking about side effects), so this will serve as a reflection on that, and it will also serve as a study to find the basis for what I think. And benchmarking is surprisingly fun. It's also a good experience to prove your theory.

See you soon!
