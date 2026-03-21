---
title: "I wrote it in Kotlin ~Part 1~"
date: 2021-03-28
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---
The other day I wrote a post about Go, but my main job is Kotlin, so if I find out something or have an idea about Kotlin, I'll write about it one by one. This time, I would like to briefly explain what kind of code I used to meet my business requirements while creating an API with Kotlin.

When I work as a server-side engineer, I sometimes feel that no matter what method is used to achieve the required functionality (this is before I talked about technology or architecture aspects such as GraphQL, REST APIs, and microservices), my work has become a pattern to some extent. In these cases, it may seem as if the logic is more important than the code. But on the other hand, since there are many similar logics, I think there is also a lot of room for improvement in order to write better code.

To be honest, I'm not very good at algorithms, so I feel like there are limits to what I can do to write efficient code. Maybe all you can do is write some working code and then refactor it to improve it little by little.

I do think there are things you can do to write better code. For example, in Java I was taught to add `final` to make objects as [immutable](https://en.wikipedia.org/wiki/Immutable_object) as possible because of reference-related concerns, but in practice, as shown by the [benchmark results](https://www.baeldung.com/java-final-performance?__s=m4suw1p9x2sbizbhxrew), that can also improve performance. Java and Kotlin both offer many useful APIs, and new ones keep getting added, so simply understanding how to use them well can help you write code that is both readable and performant.

So, this time I would like to introduce some of the code I wrote using Kotlin's API.

## Grouping lists

If there is a product information table in the DB, as well as a product attribute table, production area and store table, etc., depending on your business, you may want to see what products are sold by each store, or you may want to only want to see products that fit a specific product attribute.

In such cases, the API needs to return the data retrieved from the table, summarized based on specific columns. If you were to write this in code, it would mean returning the data obtained in `List` in `Map` using one of the attributes as a key. In Java, I think it would look like this:

```java
// Example DB data
List<User> list = List.of(
        new User("John", 20, "USA", "Programmer"),
        new User("James", 30, "Canada", "Sales"),
        new User("Jack", 35, "UK", "Programmer")
);
// Group by the user's job
Map<String, List<Pair>> map = list.stream()
        .collect(Collectors.groupingBy(User::getJob,
                Collectors.mapping(user -> new Pair(user.getAge(), user.getName()), Collectors.toList())));
// {James=[Pair(first=30, second=Sales)], John=[Pair(first=20, second=Programmer), Pair(first=35, second=Writer)]}

@Data
@AllArgsConstructor
static class User {
    private String name;
    private int age;
    private String address;
    private String job;
}

@Data
@AllArgsConstructor
static class Pair {
    private Object first;
    private Object second;
}
```

You can use Java's API as is in Kotlin, so you can do the same thing using `Stream` and `Collector` above. However, since I am using a different language, I would like to do the same thing using the API provided by Kotlin if possible.

In Kotlin, it is often possible to perform processing similar to a combination of `Stream` and `Collector` using only the functions provided by Collection, so in many cases it is sufficient to simply look for functions that are compatible with the Java API. In other words, all you need is something similar to `Collectors.groupingBy()` and `Collectors.mapping()`, which are key to the above processing, but you can combine those processes with `groupBy()`. So, if you change the above code in Kotlin, it will look like this: It feels refreshing in many ways.

```kotlin
// Example DB data
val list = listOf(
    User("John", 20, "USA", "Programmer"),
    User("James", 30, "Canada", "Sales"),
    User("Jack", 35, "UK", "Programmer")
  )
// Group into Map<String, List<Pair<Int, String>>> using job as the key
val map = list.groupBy({ it.job }, { it.age to it.name })
// {Programmer=[(20, John), (35, Jack)], Sales=[(30, James)]}

data class User(
    val name: String,
    val age: Int,
    val address: String,
    val job: String
)
```

## Change only the value of Map

In addition to the above processing, there may be other conditions. For example, let's say you have an example of calculating an amount. If an employee is supposed to receive wages for each project and the projects are managed by code, there may be cases where the wage payer only wants to know the total amount for the same project. In this case, you need to compile the data for each employee, and then add up only the amounts if there are any duplicates in the list of projects handled by that person.

In such a case, I think it would be most efficient to include such processing from the grouping stage, but since there is also a problem with threads (traversing through the map being generated), it may become quite complicated to write it in the actual code. So, first, we will take the form of combining `List` into `Map` and add further processing to the result.

In addition to `map()`, Kotlin's `Map` has functions such as `mapKeys()` and `mapValues()`, so you can map only the necessary parts. This time, I only want to change `value`, so I think using `mapValues()` is less wasteful and makes the intent clearer for the code reader. The code for further mapping using `mapValues()` looks like this:

```kotlin
data class User(val name: String, val id: Int, val amount: Int)

// Example DB data
val list = listOf(
    User("A", 1, 1000),
    User("A", 1, 2000),
    User("A", 2, 4000),
    User("B", 3, 5000)
)
// Group by name, then merge duplicate ids by summing the amounts
val map = list.groupBy({ it.name }, { it.id to it.amount })
    .mapValues {
        // Group by id
        it.value.groupBy { pair -> pair.first }
        // Keep the key and sum only the values
            .map { map -> map.key to map.value.sumBy { pair -> pair.second } }
    }
// {A=[(1, 3000), (2, 4000)], B=[(3, 5000)]}
```

Another way to combine `List` into `Map` is `groupingBy()`. When you use this function, the Collection changes to an object called [Grouping](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-grouping), and subsequent processing can be done using functions such as `aggregate()`, `reduce()`, `fold()`, and `eachCount()`. If you change the above code to use `Grouping`, it will look like this:

```kotlin
// Use Grouping.aggregate and then process the values after converting to Map
val map = list.groupingBy { it.name }
    .aggregate { _, accumulator: MutableList<Pair<Int, Int>>?, element, first ->
        // Create a MutableList for a new key
        if (first)
            mutableListOf(element.id to element.amount)
        // Otherwise, add the element to the existing list
        else
            accumulator?.apply { add(element.id to element.amount) }
    }.mapValues {
        it.value?.groupBy { pair -> pair.first }
            ?.map { pair -> pair.key to pair.value.sumBy { pair -> pair.second } }
    }
```

At first glance, `groupingBy()` may seem more complicated, but since you can stack values mapped using `accumulator`, it may be worth considering in some cases.

## Caching using Map

If the DB is frequently referenced, and the referenced data itself is not updated frequently, it may be a good idea to cache it within the application. In this case, it would be a good idea to declare `Map` with the parameter as a key and access the DB only if that key does not exist (and add it to `Map`). Since Java 1.8 has provided a method called `computeIfAbsent()`, it can be easily implemented. For example:

```java
// Example DB data
List<String> list = List.of("A", "B", "C");
// Cache map
Map<String, Boolean> map = new ConcurrentHashMap<>();
// Parameter
String element = "A";

// If the cache does not contain the parameter, look it up in the DB data, add it, and return it
Boolean exists = map.computeIfAbsent(element, key -> list.contains(element));
// Example using method reference
exists = map.computeIfAbsent(element, list::contains);
```

Since this is a feature provided in Java, it can of course be implemented in Kotlin in exactly the same way. However, according to Kotlin specifications, the code for `compute` is [written differently depending on whether it is Lambda or Method Reference](https://kotlinlang.org/docs/lambdas.html#instantiating-a-function-type), so you need to be careful about that. This is due to the specifications of Kotlin itself, but it may be difficult to understand at first if you are used to writing Java.

```kotlin
// Example DB data
val list = listOf("A", "B", "C")
// Cache map
val map = ConcurrentHashMap<String, Boolean>()
// Parameter
val element = "A"

// Lambda version
var exists = map.computeIfAbsent(element) { list.contains(element) } // false
// Method reference version
exists = map.computeIfAbsent(element, list::contains)
```

By the way, there is a method called `putIfAbsent()` that has a similar function, but the difference seems to be that in `computIfAbsent()`, the subsequent processing is performed only if there is no key in `Map`, but in `putIfAbsent()`, the process runs regardless of whether there is a key or not. Therefore, if you want to use it as a cache, it is better to use `computeIfAbsent()`.

## Finally

I've introduced some code I wrote, what do you think? I've just migrated to Kotlin, so there are a lot of things I don't understand, and there may actually be a smarter way to do it, but I personally think it's meaningful and fun to compare code with different languages ​​and look at the API source to think about how to write it in your own way, depending on the actual business requirements.

Therefore, research on how to write in Kotlin will continue. I feel like it's going to be difficult if I don't study up and try creating a simple API in Go, but...well, I'm sure it'll work out somehow.

See you soon!
