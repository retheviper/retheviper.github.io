---
title: "Read Effective Kotlin"
date: 2022-02-20
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - design pattern
---

This time I read a book for the first time in a while, so I thought I'd write a little bit about my thoughts on it. Before changing jobs, I mainly worked with Java, so I read [Effective Java](https://www.amazon.co.jp/Effective-Java-%E7%AC%AC3%E7%89%88-%E3%82%B8%E3%83%A7%E3%82%B7%E3%83%A5%E3%82%A2%E3%83%BB%E3%83%96%E3%83%AD%E3%83%83%E3%82%AF-ebook/dp/B099922DML/ref=sr_1_1?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=1OT3QRYCGG9BB&keywords=Effective+Java&qid=1645265607&sprefix=effective+kotlin%2Caps%2C753&sr=8-1) and looked back at the code I had written. After changing jobs, I started using a different language called Kotlin, but it's still a language that runs on the JVM, and the framework I'm currently using hasn't changed from Spring, so I thought it would be a good idea to basically write code from the same perspective. However, now that it's been a year since I first started using Kotlin, I'm starting to think that if the language is different, I may need to revisit the weekly schedule when writing code.

Then, I just discovered a book called [Effective Kotlin](https://www.amazon.co.jp/Effective-Kotlin-Best-practices-English-ebook/dp/B08WXCRVD2/ref=sr_1_1?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=2HVT6TZYJL65A&keywords=effective+kotlin&qid=1645264802&sprefix=effective+kotli%2Caps%2C230&sr=8-1), so I immediately read it. And in this post, I will write about various things about it.

By the way, this book itself has been published for a while, so I was able to occasionally find its contents and PDF materials on the Internet. For example, the chapter on "readability" in this book is well organized on [this blog], so please refer to it.

## overall impression

Personally, I think `Effective Java` is a book for advanced users, and many parts were difficult to understand unless you had some experience writing applications in Java. For example, "`try-finally`
``It's better to replace it with `try-with-resource`'' and ``How to write functions without side effects with `Stream`'' are introduced, but these require some basic knowledge of the design and specifications of the Java language.

In comparison, `Effective Kotlin` has quite a lot of content for beginners. For example, there was content about object orientation in the first place. However, this may be because I decided that a title with `Effective Java` in mind would be meaningless (I also mention `Effective Java` in the preface), and there are many other things that are written as ``best practices''.

Of course, there are some things that are valid in Kotlin that overlap with `Effective Java`. For example, when creating an instance of an object, it is better to write a factory method.

However, I felt that I couldn't keep up with the speed of Kotlin updates (this is also a limitation of the publication), and I felt that the content for advanced users was somewhat insufficient, so it feels more like a book for juniors.

## interesting

Even though it was said to be aimed at juniors, since I am still a junior myself, there were parts of it that I thought were interesting. I would like to introduce some of them here.

## Single responsibility principle

This is the part that touches on so-called [SOLID](https://en.wikipedia.org/wiki/SOLID). Kotlin argued that extension functions can help preserve the single-responsibility principle. First, suppose we have the following case.

```kotlin
class Student {
    // ...

    fun isPassing(): Boolean = 
        calculatePointsFromPassedCourses() > 15

    fun qualifiesForScholarship(): Boolean = 
        calculatePointsFromPassedCourses() > 30

    private fun calculatePointsFromPassedCourses(): Int {
        //...
    }
}
```

Here, `isPassing()` is used in a module called `accreditations`, and `qualifiesForScholarship()` is used in a module called `scholarship`. Then, the question is, is it okay for the class `Student` to have these functions as a single responsibility?

Therefore, it is better to define these functions as extension functions for each module.

```kotlin
// scholarship module
fun Student.qualifiesForScholarship(): Boolean {
    /*...*/
}

// accreditations module
fun Student.calculatePointsFromPassedCourses(): Boolean {
    /*...*/
}
```

Or you can think of a way to get `calculatePointsFromPassedCourses()` out. However, in this case, these two methods cannot be used as private methods. that's why,

1. Create common functions that can be used in any module
2. Create a helper function for each department

There are also other possible methods.

Indeed, if you think about it, the good thing about extension functions is that they allow you to easily add processing without implementing an interface or inheriting from a superclass, so I think it's a good idea to use them like this because you can separate processing by use case. Personally, a new discovery for me was that by using extension functions, you can control the package in which the function is placed and its visibility.

## Consider defining a DSL for complex object creation

This is the part where you should use a DSL for complex processing when creating objects. An example that Kotlin already provides is HTML. It will be defined in the following way.

```kotlin
body {
    div {
        a("https://kotlinlang.org") {
            target = ATarget.blank
            +"google"
        }
    }
    +"Some content"
}
```

It's certainly something that's often used in frameworks like Ktor, so I thought there might be some demand for it. It's not difficult to create higher-order functions in Kotlin, so it's definitely a challenge.

However, it seems difficult to establish a DSL-specific writing style and share that writing style with engineers, as well as the initial design and maintenance, so I was a little doubtful whether it would be suitable for the current era where applications are required to be downsized.

Personally, if I were to create a library or framework of some kind, I would like to try it.

## Well, that's what I thought

There was also a part where he kindly organized what I thought was the case (or what I had heard somewhere and forgotten the theoretical part, but which had unconsciously become a habit) as a sentence. So I guess you could say that I was able to reconfirm my thoughts once again.

## Do not repeat common algorithms

This is the part that says, "Don't write your own code for general algorithms that can be solved by the standard library." The reason is as follows.

1. Calling takes less time than writing code
2. It has an easy-to-understand name
3. Code becomes easier to understand
4. Optimization is effective

I myself thought it would be better to use the standard library as much as possible, so this point immediately made sense to me. I didn't know if the processing I had written was optimized or not, and I didn't want to touch logic for anything other than business use.

This book includes the following code: This is when you write your own logic.

```kotlin
val percent = when {
    number > 100 -> 100
    number < 0 -> 0
    else -> number
}
```

The above code can be simplified using [coerceIn()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/coerce-in.html). For example:

```kotlin
val percent = number.coerceIn(0, 100)
```

Kotlin has a lot of good functions, especially in the standard library, so I personally feel that in many cases it's better to check what APIs are available rather than writing your own logic. And I think it's good that you wrote a reason why you can understand that.

## Implementing your own utils

In addition to problems that can be solved with the standard library, common processing necessary for the project should be created as a utility function. It seems that creating a utility as an extension function instead of a class has the following advantages.

- Functions have no state, so they have no side effects.
- Compared to top-level functions, they have a fixed type and are easier to use.
- The form attached to the class is more intuitive than the argument.
- It's easier to find the functionality you need than organizing functions into objects.
- Since it is subordinated to a specific class, there is no need to worry about whether it is a parent class or a child class.

It's true that when I was using Java, I used the so-called Singleton Pattern to create utility classes, define classes that could be used in DI, and write static methods. With Kotlin, you can add functions to specific classes without using utility classes, making it more convenient to use.

For example, even though they do the same thing, there are differences in the code when writing an extension function and when creating a utility class, as shown below.

```kotlin
// Using extension functions
val isEmptyByExtension = "Text".isEmpty()

// Using a utility class
val isEmptyByUtilClass = TextUtils.isEmpty("Text")
```

When using a utility class, you must first think about which utility class's functions you will use. In comparison, extension functions are more intuitive because you can quickly find the function you want using auto-completion in the IDE.

Another good thing is that it can be added only to specific classes, so it can be used more safely. I guess you could say that I was able to rediscover that extension functions have many uses.

## Builder pattern

This is the part where you can use [named arguments](https://kotlinlang.org/docs/functions.html#named-arguments) in Kotlin and don't need the Builder pattern. Although it is not technically impossible to use the Builder pattern in Kotlin, the following are reasons why it is better to use named parameters:

- shorter
- more beautiful
- Simple to use
- thread safe

I myself have wondered if the Builder pattern is necessary in Kotlin since I used it in Java, but I came to the conclusion that it is not necessary. The reasons listed above are of course valid, but the reason was that the Builder pattern makes it difficult to determine whether all required parameters are present when creating an instance.

For example, let's say you have an example of the Builder pattern from a book.

```kotlin
class Pizza private constructor(
    val size: String,
    val cheese: Int,
    val olives: Int,
    val bacon: Int
) {
    class Builder(private val size: String) {
        private var cheese: Int = 0
        private var olives: Int = 0
        private var bacon: Int = 0

        fun setCheese(value: Int): Builder = apply { cheese = value }

        fun setOlives(value: Int): Builder = apply { olives = value }

        fun setBacon(value: Int): Builder = apply { bacon = value }

        fun build() = Pizza(size, cheese, olives, bacon)
    }
}
```

I think this builder can be used in the following ways.

```kotlin
val villagePizza = Pizza.Builder("L")
        .setCheese(1)
        .setOlives(2)
        .setBacon(3)
        .build()
```

However, you can still build in the following cases.

```kotlin
val villagePizza = Pizza.Builder("L").build()
```

If `cheese`, `olives`, `bacon` are not configured to allow `0`, it will be difficult to fix this. Or, if the parameter is a complex object, you should set a default value, or add a forced null check (`!!`), etc. It just becomes more complicated.

However, it's a problem that can be easily solved using named parameters. If `val` does not specify a default value, it is easy to understand that it is a required item.

```kotlin
val myFavorite = Pizza(
            size = "L",
            cheese = 5,
            olives = 5,
            bacon = 5
        )
```

## Consider factory functions instead of constructors

Java has recently introduced various factory functions, making it easier to create immutable objects. Although creating constructors and generating instances using named parameters are useful in Kotlin, there are still cases where factory functions are better. The reason is as follows.

- Functions have names, so we know how objects are created.
  - `ArrayList.withSize(3)` is easier to understand than `ArrayList(3)`
- Subtype objects can be specified as return values
  - The specific implementation can be shaped differently depending on the time and situation.
- It does not create a new object each time it is called.
  - Can also return null like `Connections.createOrNull()`
- Can provide objects that don't yet exist
  - It can be applied to create objects that work without a proxy.
- You can control visibility by creating it outside the object.
- Since it can be `inline`, it can also be [reified](https://kotlinlang.org/docs/inline-functions.html#reified-type-parameters)
- Saves effort for objects that are complex to instantiate
- Instances can be created without calling the superclass or primary constructor

I also understood this while reading it. Especially in my case, I thought I could have improved the reusability of the code by introducing a factory function in the mapping between objects such as DTO in the Service layer and Response in the Controller layer, so I now think it was a good decision.

In addition, the following methods were presented as ways to create a factory function. Generally speaking, it is often defined within a companion object, but you may want to consider other methods if necessary.

### companion object

A pattern similar to Java's static methods. This is the easiest to understand. It has the form below.

```kotlin
class MyLinkedList<T>(
    val head: T,
    val tail: MyLinkedList<T>?
) {
    companion object {
        fun <T> of(vararg elements: T): MyLinkedList<T>? {
            /*...*/
        }
    }
}

// Usage
val list = MyLinkedList.of(1, 2)
```

It was also explained that factory functions are generally named according to the following rules.

#### from

When passing one parameter and changing the type (returning an instance of itself)

```kotlin
val date: Date = Date.from(instant)
```

#### of

When passing multiple parameters and converting them to a bundled type

```kotlin
val faceCards: Set<Rank> = EnumSet.of(JACK, QUEEN, KING)
```

#### valueOf

Redundant form of `of`

```kotlin
val prime: BigInteger = BigInteger.valueOf(Integer.MAX_VALUE)
```

#### instance / getInstance

Obtaining a Singleton instance (if the parameters are the same, the same instance will always be returned)

```kotlin
val luke: StackWalker = StackWalker.getInstance(options)
```

#### createInstance / newInstance

instance/getInstance are similar but always return a new instance

```kotlin
val newArray = Array.newInstance(classObject, arrayLen)
```

#### getType

Similar to instance/getInstance, but when returning an instance of a different type

```kotlin
val fs: FileStore = Files.getFileStore(path)
```

#### newType

Similar to createInstance / newInstance, but when returning an instance of a different type

```kotlin
val br: BufferedReader = Files.newBufferedReader(path)
```

### extension

Define a companion object in the class, and add a factory function using an extension function from outside. You don't have to modify the original class, and you can take advantage of the features of extension functions, such as package and visibility control.

```kotlin
interface Tool {
    companion object { /*...*/ }
}

fun Tool.Companion.createBigTool( /*...*/ ): BigTool {
    //...
}
```

### top-level

Things like [listOf()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/list-of.html), [setOf()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/set-of.html), and [mapOf()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/map-of.html) included in the standard library.

Although it is convenient for frequently used types because they are easy to use, it is necessary to be careful when naming them, as they may cause confusion if they appear in the IDE's auto-completion.

### fake constructor

It uses Pascal Case to make a function look like a constructor. Kotlin's standard libraries include the following:

```kotlin
List(4) { "User$it" } // [User0, User1, User2, User3]
```

This is actually a function like this:

```kotlin
public inline fun <T> List(
    size: Int,
    init: (index: Int) -> T
): List<T> = MutableList(size, init)

public inline fun <T> MutableList(
    size: Int,
    init: (index: Int) -> T
): MutableList<T> {
    val list = ArrayList<T>(size)
    repeat(size) { index -> list.add(init(index)) }
    return list
}
```

This seems to be something to consider when you need to create a constructor for an interface or when you need arguments of type `reified`.

There are other ways to create a fake constructor.

```kotlin
class Tree<T> {

    companion object {
        operator fun <T> invoke(size: Int, generator: (Int) -> T): Tree<T> {
            // ...
        }
    }
}

// Usage
Tree(10) { "$it" }
```

However, in this case, there seems to be a problem with the constructor reference, which makes the code complicated.

```kotlin
// Constructor
val f: () -> Tree = ::Tree

// Fake Constructor
val d: () -> Tree = ::Tree

// Invoke in companion object
val g: () -> Tree = Tree.Companion::invoke
```

Therefore, if you are going to use a fake constructor, it seems better to define it as a function.

### factory class

The method is to create a separate class called Factory and return an instance. In Java, there are cases where you do something like that with an interface (like [List.of()](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/List.html#of(E))), but is it ok to do it in Kotlin? A question arose. In conclusion, since "factory classes can have state," it may be worth considering in some cases. This may be more useful than you think.

```kotlin
data class Student(
    val id: Int,
    val name: String,
    val surname: String
)

class StudentsFactory {
    var nextId = 0
    fun next(name: String, surname: String) =
        Student(nextId++, name, surname)
}

val factory = StudentsFactory()
val s1 = factory.next("Marcin", "Moskala")
println(s1) // Student(id=0, name=Marcin, Surname=Moskala)
val s2 = factory.next("Igor", "Wojda")
println(s2) // Student(id=1, name=Igor, Surname=Wojda)
```

## lastly

This is a rough summary, but the above is my impression of the knowledge I gained from this book. I made some new discoveries, and I felt like I was getting other people's explanations to confirm that my habits weren't wrong, so it was quite interesting.

However, perhaps because Kotlin is still a new language and has absorbed various paradigms, I feel that there is not enough discussion on high-level etiquette like `Effective Java`, which I feel is a bit disappointing. Well, I'm cheekily imagining that the fact that I've come to think this way is in itself proof that I've grown a little.

See you soon!
