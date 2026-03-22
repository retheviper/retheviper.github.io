---
title: "Sorting in Various Languages"
date: 2021-11-10
translationKey: "posts/languages-comparsion-sorting"
categories: 
  - languages
image: "../../images/magic.webp"
tags:
  - kotlin
  - java
  - javascript
  - python
  - swift
  - go
---
I think we are now in an era where there is not much difference in what you can do no matter what programming language you choose, and you can just choose whatever programming language you like. Especially in an era where transpilers like [Kotlin/JS](https://kotlinlang.org/docs/js-overview.html) and frameworks like Flutter are appearing one after another, I think this trend will continue to accelerate.

However, while there are such changes, I think that there is now an increasing demand for the number of programming languages ​​that each programmer can handle. In actual work, the language used is fixed for various reasons, and it is necessary to be able to use something that you have never used before, and there are cases where a single engineer does not have a fixed position but implements it in various fields. It can also be said to be the era of the so-called [Polyglot](https://en.wikipedia.org/wiki/Polyglot_(computing)).

Therefore, I think it is important to at least understand the characteristics of various languages. And even if it's not for such a necessity, I also think that by being exposed to concepts in a language that you are not normally exposed to, you can deepen your understanding of the main language. In a sense, this is similar to the fact that the things that can be done in any language are not much different, but there are cases where new APIs and functions that incorporate concepts from other languages ​​are introduced, and such libraries appear.

Now, the introduction is long, but from now on, I would like to briefly compare how certain operations can be performed in various languages, and the characteristics of such cases. This time, we will be sorting the array.

## JavaScript

In JavaScript, you can sort arrays with [Array.prototype.sort()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort). So you can use code like the following. It's simple.

```javascript
> const a = [22, 1, 44, 300, 5000]
> a.sort()
[ 1, 22, 300, 44, 5000 ]
```

Also, if you want to create a new sorted array without changing the values of the original array, you can use the following method.

```javascript
> const a = [22, 1, 44, 300, 5000]
> const b = [...a].sort() // copy and sort `a`
> console.log(b)
[ 1, 22, 300, 44, 5000 ]
```

However, as some of you may have noticed, the sorted values are not as expected. If true, it would normally be `1, 22, 44, 300, 5000`. If you want to sort the values ​​in ascending order here, you will need to create your own sorting method. For example, there is a method like below.

```javascript
> const a = [22, 1, 44, 300, 5000]
> a.sort((a, b) => a - b)
[ 1, 22, 44, 300, 5000 ]
```

In this `sort()`, the following will happen depending on the result of the return value of `compareFunction` (two arguments, return value is number) passed as an argument.

- if less than 0, put index of a before b
- If 0, a and b are not changed
- if greater than 0, put index of b before a

If you are familiar with Java, you will immediately understand that this is the same as [Comparator](https://docs.oracle.com/javase/8/docs/api/java/util/Comparator.html). The shape of the arrow function is also similar to Java's Lambda, so I think you can adapt it without feeling too strange. Although it is quite simple, it is important that you need your own `compareFunction` for number type arrays, so you will need to be careful.

If you want to reverse the index of an array, just use [Array.prototype.reverse()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reverse). In this case, you don't need your own `compareFunction` even for a number array, so it's convenient.

```javascript
> const a = [22, 1, 44, 300, 5000]
> a.reverse()
[ 5000, 300, 44, 1, 22 ]
```

## Java

Next, let's take a look at Java. As mentioned earlier, you can easily implement the sorting method using `Comparator`, so they are basically the same. However, in the case of Java, there are various methods such as [List.sort()](https://docs.oracle.com/javase/8/docs/api/java/util/List.html#sort-java.util.Comparator-), [Collections.sort()](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#sort-java.util.List-), [Arrays.sort()](https://docs.oracle.com/javase/8/docs/api/java/util/Arrays.html#sort-int:A-), and [Stream.sorted()]. This means that you need to consider various options such as whether or not objects such as ollection and Array are `Immutable`, whether you should implement `Comparator` and [Comparable](https://docs.oracle.com/javase/8/docs/api/java/lang/Comparable.html) yourself, or whether you should use ones provided in the standard library.While there are various options, I think the easiest way is to use `Collections.sort()` or `Arrays.sort()`. When using this, primitive types and String Lists have the advantage of being able to be sorted with short code (and as a standard feature).

```java
jshell> var a = new ArrayList<>() {{ add(22); add (1); add(44); add(300); add(5000); }};
a ==> [22, 1, 44, 300, 5000]

jshell> Collections.sort(a);

jshell> System.out.println(a);
[1, 22, 44, 300, 5000]
```

Then `List.sort()` is easy. You need to pass `Comparator` as an argument, but if you want to sort in ascending or descending order, just call the already prepared method.

```java
jshell> var a = new ArrayList<>() {{ add(22); add (1); add(44); add(300); add(5000); }};
a ==> [22, 1, 44, 300, 5000]

jshell> a.sort(Comparator.naturalOrder());

jshell> System.out.println(a);
[1, 22, 44, 300, 5000]
```

By the way, the default sorting methods that can be used with `Comparator` are as follows.

- Ascending: [naturalOrder()](https://docs.oracle.com/javase/8/docs/api/java/util/Comparator.html#naturalOrder--)
- Descending: [reverseOrder()](https://docs.oracle.com/javase/8/docs/api/java/util/Comparator.html#reverseOrder--)
- Reverse order: [reversed()](https://docs.oracle.com/javase/8/docs/api/java/util/Comparator.html#reversed--)

`Comparator` can also be used as an argument for `Collections.sort()`. So, if you want to sort in descending order, you can use the following code.

```java
jshell> var a = new ArrayList<>() {{ add(22); add (1); add(44); add(300); add(5000); }};
a ==> [22, 1, 44, 300, 5000]

jshell> Collections.sort(a, Comparator.reverseOrder());

jshell> System.out.println(a);
[5000, 300, 44, 22, 1]
```

Alternatively, if you want to obtain a newly sorted result without changing the values of the original List, you can copy the original List, but another method is to use `Stream`.

```java
jshell> var a = List.of(22, 1, 44, 300, 5000);
a ==> [22, 1, 44, 300, 5000]
jshell> var b = a.stream().sorted().collect(Collectors.toList());
b ==> [1, 22, 44, 300, 5000]
```

Even when sorting by `Stream`, you can use `Comparator`.

```java
jshell> var a = List.of(22, 1, 44, 300, 5000);
a ==> [22, 1, 44, 300, 5000]
jshell> var b = a.stream().sorted(Comparator.reverseOrder()).collect(Collectors.toList());
b ==> [5000, 300, 44, 22, 1]
```

Also, if you want to sort the DTO List, you can consider having the DTO inherit `Comparable`, but in most cases, I think it would be better to implement `Comparator`, which clearly shows the sorting conditions. Even considering versatility and flexibility, I think it would be safer to use `Comparator`, since in the case of `Comparable`, the class needs to be modified when conditions change.

In the case of Array, you can use `Arrays.sort()` to sort (of course you can also use `Comparator`), and you can also convert it to List or Stream, so you can use the above method as is. So, there are more options, but I think it's a good idea to choose the method that is convenient (that suits your preferences). Personally, I think it would be better in terms of readability to pass `Comparator` to `Arrays.sort()`.

## Kotlin

As Kotlin provides a lot of Sytax Sugar, there are many sorting options to choose from. So, I tried to summarize it a little.| Order type | Sort result | fun | Notes |
|---|---|---|---|
| Natural | Caller | [Array/MutableList.sort()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sort.html) | Ascending |
| | | [Array/MutableList.sortDescending()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sort-descending.html) | Descending |
| | | [Array/MutableList.reverse()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/reverse.html) | Reverse order |
| | Array | [Array.sortedArray()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-array.html) | Ascending |
| | | [Array.sortedArrayDescending()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-array-descending.html) | Descending |
| | | [Array.reveredArray()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/reversed-array.html) | Reverse order |
| | List | [Array/List.sorted()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted.html) | Ascending |
| | [Array/List.sortedDescending()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-descending.html) | Descending |
| | [List/MutableList.asRevered()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/as-reversed.html) | Reverse order |
| Custom | Caller | [Array/MutableList.sortBy()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sort-by.html) | Ascending order, requires selector((T) -> R) |
| | [Array/MutableList.sortByDescending()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sort-by-descending.html) | Descending, requires selector((T) -> R) |
| | List | [Array/Iterable.sortedBy()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-by.html) | Ascending order, selector((T) -> R) required |
| | [Array/Iterable.sortedByDescending()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-by-descending.html) | Descending, requires selector((T) -> R) |
| | Array | [Array.sortedArrayWith()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-array-with.html) | [Comparator](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-comparator/) required |
| | List | [Array/Iterable.sortedWith()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-with.html) | [Comparator](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-comparator/) required |

It looks like there are quite a lot of options, but if you summarize them in a table like this, I think it's pretty easy to understand. If you know that sorting can be done based on several criteria, such as whether you need to write your own comparison processing, whether the sorted result is the original array, and whether it will be an Array or a List, I think you won't have to worry about which one to use.So, first, clarify what you want to do, then choose which API to use and write. The following is an example of creating a new sorted List from a List. These are for ascending order and descending order, respectively.

```kotlin
>>> val a = listOf(22, 1, 44, 300, 5000)
>>> val b = a.sorted()
>>> println(b)
[1, 22, 44, 300, 5000]
>>> val c = a.sortedDescending()
>>> println(c)
[5000, 300, 44, 22, 1]
```

Also, if you want to sort the data class array, you can use `sortBy` and `sortedBy`. What we need as an argument here is a selector of type `(T) -> R`, but it is easy to implement because we simply specify which criteria to sort on. See example below.

```kotlin
>>> data class Data(val number: Int)
>>> val a = listOf(Data(22), Data(1), Data(44), Data(300), Data(5000))
>>> val b = a.sortedBy { it.number }
>>> println(b)
[Data(number=1), Data(number=22), Data(number=44), Data(number=300), Data(number=5000)]
>>> val c = a.sortedByDescending { it.number }
>>> println(c)
[Data(number=5000), Data(number=300), Data(number=44), Data(number=22), Data(number=1)]
```

In addition, if you want to specify more complex comparison conditions, it is a good idea to implement `Comparator`, as in Java. It seems to be similar to Java, but it feels more simplified (and because of that, there are more options).

## Swift

In Swift, the only options seem to be to simply sort the original Collection or create a new sorted Collection. Nothing has changed much, but if you want to sort the original Collection, it will look like this:

```swift
  1> var a = [22, 1, 44, 300, 5000]
a: [Int] = 5 values {
  [0] = 22
  [1] = 1
  [2] = 44
  [3] = 300
  [4] = 5000
}
  2> a.sort()
  3> print(a)
[1, 22, 44, 300, 5000]
```

And if you want to create a new Collection it would be like this:

```swift
  1> let a = [22, 1, 44, 300, 5000]
a: [Int] = 5 values {
  [0] = 22
  [1] = 1
  [2] = 44
  [3] = 300
  [4] = 5000
}
  2> let b = a.sorted()
b: [Int] = 5 values {
  [0] = 1
  [1] = 22
  [2] = 44
  [3] = 300
  [4] = 5000
}
  3> print(b)
[1, 22, 44, 300, 5000]
```

However, what makes Swift sorting unique is when you specify how to sort. [sort()](https://developer.apple.com/documentation/swift/array/2296801-sort) and [sorted()](https://developer.apple.com/documentation/swift/array/2296815-sorted) now allow you to pass a function called `areInIncreasingOrder` as an argument, but while the return value of `compareFunction` and `Comparator` used in JavaScript, Java, and Kotlin was a number, the return value of `areInIncreasingOrder` is a Bool as a predicate type. Therefore, you can specify the sorting method in the following way.

```swift
let students: Set = ["Kofi", "Abena", "Peter", "Kweku", "Akosua"]
let descendingStudents = students.sorted(by: >)
print(descendingStudents) // "["Peter", "Kweku", "Kofi", "Akosua", "Abena"]"
```

Alternatively, if you want to sort based on the class field, you can use the following method.

```swift
struct Data { var number = 0 }

let datas = [Data(number: 1), Data(number: 3), Data(number: 4), Data(number: 2)]

let descending = datas.sorted { $0.number > $1.number }
dump(descending)
/**
  descending: [Data] = 4 values {
    [0] = {
      number = 4
    }
    [1] = {
      number = 3
    }
    [2] = {
      number = 2
    }
    [3] = {
      number = 1
    }
  }
*/
```

## Go

Perhaps because Go does not have generics, the package [sort](https://pkg.go.dev/sort) provides various funcs for sorting depending on the type of slice. For example:

- func Float64s(x []float64)
- func Ints(x []int)
- func Strings(x []string)

So, if it is a struct slice, you will need to select one of these to sort. For example:

```go
a := []int{22, 1, 44, 300, 5000}
sort.Ints(a)
fmt.Println(a) // [1 22 44 300 5000]
```

For struct, you can use the following methods: The sorting criteria is again `bool`.

```go
people := []struct {
  Name string
  Age  int
}{
  {"Gopher", 7},
  {"Alice", 55},
  {"Vera", 24},
  {"Bob", 75},
}
sort.Slice(people, func(i, j int) bool { return people[i].Name < people[j].Name })
fmt.Println(people) // [{Alice 55} {Bob 75} {Gopher 7} {Vera 24}]
```

What's interesting is that Go's sorting has a separate function called [sort.SliceStable()](https://pkg.go.dev/sort#SliceStable). This performs [stable sort](https://en.wikipedia.org/wiki/Sorting_algorithm#Stability), and its definition is described in Wiki as follows.

> The order of equivalent data before sorting is preserved even after sorting. In other words, the ranking positional relationship is always maintained in each state during sorting.

In other words, in the case of stable sorting, the original positional relationship (index) between elements with the same sorting standard value is guaranteed. Let's see what the results actually are.

```go
people := []struct {
  Name string
  Age  int
}{
  {"Alice", 25},
  {"Elizabeth", 75},
  {"Alice", 75},
  {"Bob", 75},
  {"Alice", 75},
  {"Bob", 25},
  {"Colin", 25},
  {"Elizabeth", 25},
}

sort.SliceStable(people, func(i, j int) bool { return people[i].Age < people[j].Age })
fmt.Println(people) // [{Alice 25} {Bob 25} {Colin 25} {Elizabeth 25} {Alice 75} {Alice 75} {Bob 75} {Elizabeth 75}]
```

As you can see in the code execution result, you can see that the original order of `Alice 25`, `Bob 25`, `Colin 25`, `Elizabeth 25` and `Alice 75`, `Bob 75`, `Elizabeth 75` is maintained while being sorted. If you use `sort.Slice()` here, it will look like this:

```go
sort.Slice(people, func(i, j int) bool { return people[i].Name < people[j].Name })
fmt.Println(people) // [{Alice 25} {Alice 75} {Alice 75} {Bob 75} {Bob 25} {Colin 25} {Elizabeth 75} {Elizabeth 25}]
```

Stable sorting is likely to have lower performance than non-stable sorting (because it takes into account the original index), so if there is no problem with sorting based on a single value, I feel that `sort.Slice()` is sufficient, but if not, stable sorting may need to be considered.

## Python

In Python, you can use [list.sort()](https://docs.python.org/3/library/stdtypes.html#list.sort) or [sorted()](https://docs.python.org/3/library/functions.html#sorted). It's pretty much the same in other languages, so I think you can guess just from the naming, but the former sorts the original list, and the latter creates a new list.

First, `list.sort()` can be used as follows. It's not much different from other languages.

```python
>>> a = [22, 1, 44, 300, 5000]
>>> a.sort()
>>> print(a)
[1, 22, 44, 300, 5000]
```

In contrast, `sorted()` can be used as follows.

```python
>>> a = [22, 1, 44, 300, 5000]
>>> b = sorted(a)
>>> print(b)
[1, 22, 44, 300, 5000]
```

In addition, by specifying parameters such as `key` and `reverse` in these functions, you can specify which criteria to sort, or whether to sort in reverse order. It's as simple as Python.

```python
class Data:
  def __init__(self, number):
    self.number = number
  def __repr__(self):
    return repr((self.number))
 
datas = [Data(1), Data(3), Data(2), Data(4)]
datas.sort(key=lambda data: data.number) # [1, 2, 3, 4]
sorted(datas, key=lambda data: data.number, reverse=True) # [4, 3, 2, 1]
```

## Extra: Stable sort

There was some talk about stable sorting in Go's sorting method, but the other languages compared here didn't have the option of using stable sorting or non-stable sorting like Go does, so I've created a table to show how stable sorting is handled in each language. See below.

| Language | stable | non-stable | Notes |
|---|---|---|---|
| Go | ⭕️ | ⭕️ | Can be selected by func |
| Java | ⭕️ | ⭕️ | Stream is non-stable |
| JavaScript | ⭕️ | ⭕️ | Depends on browser version |
| Python | ⭕️ | ❌ | |
| Kotlin | ⭕️ | ❌ | Stable even in Sequence |
| Swift | ❌ | ⭕️ | Expressed as "stability is not guaranteed" |

Many languages support stable sorting, but the specifications may differ slightly. For example, in the case of Java, sorting by Stream is not a stable sort, so if you want to guarantee stable sorting results, we recommend using an already sorted Collection. In the case of Kotlin, even when using a Sequence similar to Stream, stable sorting was supported, probably because it is `stateful`.

Also, in the case of JavaScript, it depends on the browser version, but most modern browsers support stable sorting. However, in the case of projects using JavaScript, IE may also be considered as a target browser, but IE does not support stable sorting, so I think you need to check.

In the case of Swift, they are still considering whether to make the default value when sorting stable, and it seems that the API is also considering whether to separate stable and non-stable values ​​like Go. It seems that they are also discussing which algorithm to use, so I don't think we can expect stable sorting for a while.

Kotlin and Python have stable sorting in all cases, so it's nice to have one less thing to worry about.

## Finally

This time I looked into sorting in various languages, but what do you think? Once data is sorted, subsequent elements can be accessed quickly, so I think this is necessary from a tuning perspective. When you look into the sorting APIs of various languages ​​like this, you can get a glimpse of the language's design philosophy and development process, which is interesting and educational. I learned a lot about stable sorting, which I personally didn't pay much attention to.

I would like to continue to compare the use of various languages, APIs, and the differences between languages ​​when doing the same thing. That is, if you have enough time and energy...!

See you soon!
