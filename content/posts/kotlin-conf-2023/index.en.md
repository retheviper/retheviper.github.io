---
title: "A Summary of KotlinConf'23"
date: 2023-04-14
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - kotlinconf
  - compose
  - ios
---
[KotlinConf](https://kotlinconf.com/) was held again this year. I don't attend or watch keynotes with any interest every year, but Jetbrains products such as K2 Compiler and KMM seem to be quite popular these days, so I decided to watch them. There were many more interesting presentations than I expected, so I thought I would briefly summarize them this time.

Now, I will introduce what was talked about in each session.

## K2 Compiler

First, let's talk about [K2 Compiler](https://blog.jetbrains.com/kotlin/2021/10/the-road-to-the-k2-compiler/), which is scheduled to be adopted in Kotlin 2.0. It has been announced since 2021, and is scheduled to not only improve compiler performance but also provide features such as plug-in support. Development is currently progressing, and even now that Kotlin 1.8 has been released, many parts are still completed.

Here, we first presented a graph showing the difference in compiler performance between Kotlin 1.8 and 2.0. Naturally, it seems that the speed has increased considerably with 2.0. What used to take 20 seconds under the same environment has now been reduced to 10 seconds.

![Compilation time comparison](compilation-time-comparison.webp)

Kotlin has been adopted as the official language for Android, and I remember hearing that builds were slowing down among developers who migrated from Java to Kotlin, so this is a very happy result. Also, as the performance of the compiler improves, compilation with Intellij will also become faster, so I think it will become a more comfortable development environment.

Also, Kotlin 1.9 is scheduled to be released in the second half of this year, and there are no plans for a version like 1.10 after that. In other words, after 1.9, it will immediately become 2.0. And since 2.0 maintains backward compatibility with 1.9, if it can be compiled with 1.9, it will also be compiled with 2.0.

Of course, I don't think many companies are rushing to update their languages, and since we don't know what problems will arise from changing the compiler itself, it will probably take quite a while before it is actually adopted. However, I personally want to try out various side projects, so I'm looking forward to being able to use K2 Compiler from next year.

## Most requested features

You can discuss Jetbrains products on the site [Youtrack](https://www.jetbrains.com/youtrack/), and this is a session to explain how we will respond to the most requested functions in the future. Let's take a look at each request one by one.

### Static Extensions

[KT-11968](https://youtrack.jetbrains.com/issue/KT-11968/Research-and-prototype-namespace-based-solution-for-statics-and-static-extensions) states that they would like to be able to add static methods and properties to Java classes, like Companion objects.

For example, until now it was not possible to define a Companion object in a Java class and use it without creating an instance as shown below.

```kotlin
File.open("data.txt")
fun File.Companion.open(name: String)
```

This can be written as follows using the keyword `static`.

```kotlin
fun File.static.open(name: String)
```

Personally, I've had bugs caused by the difference in the MAX_VALUE threshold for dates between MySQL and JVM, and I wanted to add a separate property to LocalDate, but I couldn't add a static property to a Java class, so I gave up, so this is a very welcome change.

### Collection Literals

[KT-43871](https://youtrack.jetbrains.com/issue/KT-43871/Collection-literals) As the name suggests, we would like to be able to create collection literals.

For example, until now, collection literals were not supported at the language level, so I think it was often written in the following way.

```kotlin
cmdArgs = listOf("-language-version", "2.0")
```

This means that you will be able to write collection literals as shown below.

```kotlin
val skip: PersistentSet<Int> = [0, 1]
val skip2 = PersistentSet [0, 1]
```

Personally, I would like to see the scope of the `const` keyword expanded further, but this is not a bad change either. In particular, I think that arrays used in annotations etc. are actually literals in many cases, so there may be room for various other uses.

### Name-Based Destructuring

[KT-19627](https://youtrack.jetbrains.com/issue/KT-19627) states that they would like the variable name and field name to match when declaring decomposition.

For example, let's say you declare the decomposition as follows.

```kotlin
data class Person(
    val firstName: String,
    val lastName: String
)

val (firstName, lastName) = Person("John", "Doe")
```

In the above code, `firstName` becomes `John`, `lastName` becomes `Doe`. It also matches the actual field name of the data class, so there is no problem. However, what if you make a mistake and do the following?

```kotlin
val (lastName, firstName) = Person("John", "Doe")
```

In this case, `firstName` becomes `Doe`, and `lastName` becomes `John`, contrary to the intention. In order to avoid such mistakes, many people are taking measures such as introducing [inline class](https://kotlinlang.org/docs/inline-classes.html) to define a type for each field, or not using the decomposition declaration itself, but in the future, in order to avoid such mistakes, the compiler will judge whether the variable name and field name match at the time of decomposition declaration and assign a value.

Personally, I think it's pretty amazing, but there are also a lot of concerns. I'm not sure if it will work only when the variable name and field match, so I would like to see how it actually works.

### Context Receivers

[KT-10468](https://youtrack.jetbrains.com/issue/KT-10468/Context-receivers-multiple-receivers-on-extension-functions-properties) states that if a function requires a context, it would be possible to pass the context using a separate keyword rather than as a parameter.

For example, suppose you have a function like the following:

```kotlin
fun processRequest(context: ServiceContext, request: ServiceRequest) {
    val data = request.loadData(context)
}
```

The above function takes a context as an argument, and then passes that context to a different function. Naturally, the called function also needs a context as an argument. It's like below.

```kotlin
fun ServiceRequest.loadData(context: ServiceContext): Data { /** ... */ }
```

In this case, the more other functions are called within a function, the more context needs to be passed to those functions as well. Therefore, you can now use a separate keyword as shown below to pass the context without adding any arguments.

```kotlin
context(ServiceContext)
fun processRequest(request: ServiceRequest) {
    val data = request.loadData()
}

context(ServiceContext)
fun ServiceRequest.loadData(): Data { /** ... */ }
```

I still don't know what the criteria for the context that can be passed to this `context` keyword is, or how to call the context in a function, but I think it's a cleaner feeling.

### Explicit Fields

[KT-14663](https://youtrack.jetbrains.com/issue/KT-14663/Support-having-a-public-and-a-private-type-for-the-same-property) states that they would like to avoid having to define public properties for private properties.

For example, if you want to refer to a private property from the outside, you might write it as follows. This means that while the property remains private, its value is provided by another property that can only be viewed externally.

```kotlin
private val _applicationState = MutableStateFlow(State())
val applicationState: StateFlow<State>
    get() = _applicationState
```

It seems that this can be written as follows.

```kotlin
val applicationState: StateFlow<State>
    field = MutableStateFlow(State())
```

It seems to be quite convenient as it reduces the number of lines considerably and eliminates the need to define similar properties. I think there are many patterns like this, especially when managing the state with properties like in Compose, so I thought it could be used effectively.

## Kotlin Notebooks

In this session, there was a new announcement called Kotlin Notebooks. Currently, you can use Kotlin with [Jupyter Notebook](https://jupyter.org/), but it feels like similar functionality was developed specifically for Kotlin. Jupyter itself is famous, and I've already introduced many of its features in the video, so I'll just post a screenshot of it rather than an explanation. The following usage examples are introduced.

![Prototyping](kotlin-notebook-example-1.webp)

![Learning to use language (AI support)](kotlin-notebook-example-2.webp)

![Verification of the algorithm (visualization of sorting)](kotlin-notebook-example-3.webp)

![Data investigation](kotlin-notebook-example-4.webp)

![Data analysis](kotlin-notebook-example-5.webp)

![Plot generation](kotlin-notebook-example-6.webp)

In addition, the introduction reveals that it can auto-complete, support online code sharing, and can sort tables and change the order of columns.

## Google@KotlinConf

This session had people from Google giving presentations, but they mainly talked about metrics, such as how many Android apps use Kotlin and Compose. There was also talk that Google is actively using Kotlin, and Google Workspace also uses Kotlin Multiplatform to write business logic.Another thing to note is that Gradle's default setting for Android development is now Kotlin DSL. If you're developing with Kotlin, I thought it would be more convenient than Groovy, so I'm glad about this change.

However, although Google also uses Kotlin, there are competitors such as Flutter and Go, so I was curious about what direction they would take in the future. Of course, since it was a Kotlin conference, I didn't touch on such topics, but I wanted to pay more attention to Google's future policies and the market share of Compose Multiplatform.

## Crossplatform

This session included the announcement of Compose's iOS support and the current introduction of Mutiplatform. You continued to talk about how Compose is compatible with many libraries at the Alpha stage for iOS and the Beta stage for Multiplatform. It's true that Kotlin was developed from the beginning with the goal of being able to be used in areas other than the JVM, but I felt like that roadmap was finally becoming a reality more than 10 years after its introduction.

Personally, I felt that intellij and Compose were a better fit than Xcode and SwiftUI, so I'm very happy that I can now develop for iOS. In particular, since the development of AppCode was scheduled to end this year, I was thinking that I would have to learn SwiftUI in order to develop for Mac and iOS, so the timing is perfect. I've recently been working on a side project to develop web, mobile, and desktop apps using only Kotlin, so I'm thinking of trying it out right away.

## Finally

I watched the video purely because I wanted to know the current state of K2 Compiler, and I was happy to see so many unexpected announcements. Personally, I wanted to be able to do what I needed in one language, so I felt like I made the right choice in choosing Kotlin.

I don't think the market share on the server side is that high yet, but as Kotlin continues to be used in a variety of fields, I think it has a lot of potential to grow as a language, so I'm looking forward to seeing it in the future.

I've just introduced the KotlinConf keynote video, but there are many other videos available on the official channel, so if you're interested, please check them out.

See you soon!
