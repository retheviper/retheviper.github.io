---
title: "I tried SwiftUI ~ Part 1 ~"
date: 2022-07-31
categories: 
  - swift
image: "../../images/swift.webp"
tags:
  - swift
  - swiftui
  - gui
  - kotlin
  - compose
---
Looking back on my career so far, my work experience has been mostly on the back end, and I have never been involved in screen implementation. However, in order to create a standalone app, whether it's web, mobile, or desktop, you need a screen, so I'm always thinking that someday I'll need to be able to implement the screen side as well.

When it comes to creating screens, there are many factors to consider, such as the engineering path you want to follow, the kind of company you want to work for, and the languages you already know. In my case, I am familiar with Kotlin, so I chose [Jetpack Compose](https://developer.android.com/jetpack/compose) because it can handle web, mobile, and desktop applications. I want to study [SwiftUI](https://developer.apple.com/xcode/swiftui/) because I often use Apple products such as Mac, iPhone, and iPad, and it seems relatively easy to get started with Swift from Kotlin.

Now, after deciding on the language and framework, it's time to put it into practice. The [official tutorial](https://developer.apple.com/tutorials/swiftui) was very comprehensive, so I would like to start by summarizing what I found impressive about Swift and SwiftUI while recommending it. Of course, I have never been involved in the implementation of mobile apps in my work, so I think the content will be poor, but I hope it will be helpful for those who are like me who are thinking of trying Swift from a Kotlin background, those who are new to GUI from a backend-only career, or those who are interested in either Kotlin or Swift.

## Swift

Let's start with the language itself. I've heard that Kotlin and Swift are very similar, but it's hard to know exactly what they are until you try them. The expression "similar" means that there are commonalities, so the commonalities that can be cited will vary depending on what criteria you use.

For example, commonalities can be found in the fact that they are OOP-oriented from a language design perspective, have functional elements, and have GC. Alternatively, it can also mean that the keywords and writing styles are similar in language usage. In detail, you can also point out that you don't have to use semicolons.

So, first of all, I would like to explain how Swift compares to Kotlin based on my experience while going through the above tutorial.

### Something similar to Kotlin

First, I'll start by saying what I feel is similar to Kotlin. Even if they are similar, they are just a ``skin feel'', so strictly speaking, many of them have different specifications, but the standard here is ``How closely can you reproduce what was done with Kotlin'' (though this is just my personal opinion), so please use it as a reference.

### Computed properties

First, let's talk about properties. When defining a data class in Kotlin, properties can be defined in two ways:

```kotlin
data class Student(
    val age: Int
) {
    val isAdult: Boolean
        get() = age >= 18
}
```

Here, `age` is a simple value that is fixed when creating an instance, but `isAdult` is defined to return the result of the process defined as a getter (whether `age` is 18 or higher). Properties that involve such processing could also be defined in Swift through [Computed Properties](https://docs.swift.org/swift-book/LanguageGuide/Properties.html#ID259). If you want to perform similar processing, you can define it as follows.

```swift
struct Student {
    var age: Int

    var isAdult: Bool {
        get { return age >= 18 }
    }
}
```

I haven't mentioned anything yet, but I feel like I've started to understand the meaning of "Kotlin and Swift are similar". This is true of the specification of being able to handle properties that involve processing, but also of type definitions, and the fact that the code can be read with a similar feeling even though the keywords are slightly different.

However, there are still some differences. I think this is because Swift can use structs like Go and Rust for data classes. Of course, Swift also has classes, so it seems that you will choose one depending on your purpose. In that respect, it seems similar to Kotlin's use of data classes and classes separately.

### Extension

Next up is expansion. Kotlin allows you to define methods and properties outside of an object. These are called extension functions and extension properties, and can be defined as follows.

```kotlin
val Student.isUnderAge: Boolean
    get() = age < 18
```

This is a usage method presented in [Effective Kotlin](https://www.amazon.com/dp/B08WXCRVD2), which I have previously mentioned on this blog. If different processing is required depending on the use case or domain, you can use an extension like this to restrict access by defining methods and properties per package instead of putting everything into a class. You can do something similar in Swift, but the way you write it is a little different. If you were to implement a property like the one above in the same way in Swift, it would look like this:

```swift
extension Student {
  var isUnderAge: Bool {
    get { return age < 18 }
  }
}
```

As mentioned above, [Extension](https://docs.swift.org/swift-book/LanguageGuide/Extensions.html) exists as a separate keyword in Swift, and you can add functions and properties as if you were defining a new class or struct. Personally, I think it's similar to Rust's [method](https://doc.rust-jp.rs/book-ja/ch05-03-method-syntax.html), and the fact that it's grouped into one `extension` feels more organized than Kotlin, so I think it's nice. In the case of Kotlin, if there are multiple extensions for one object, it looks a little dirty...

### String Interpolation

This is also the case with Java, and in many languages, when you want to combine a string and a value stored in another variable into a single string, you often use `format()` or convert them to strings and combine them. Of course you can use it like that in Kotlin, but with [string templates](https://kotlinlang.org/docs/basic-types.html#string-templates) you can easily embed different values in a string. For example, something like the following.

```kotlin
val world = "World"
println("Hello, $world")
```

Swift also has [String Interpolation](https://docs.swift.org/swift-book/LanguageGuide/StringsAndCharacters.html#ID292), so you can do the same thing. The writing style is slightly different, but the functionality is almost the same.

```swift
let world = "World"
print("Hello, \(world)!")
```

### Arguments

In Kotlin, you can easily implement overloading by setting default values for function parameters, and you can also handle cases where the parameters are not passed.

```kotlin
// Print the string the specified number of times
fun printHello(string: String, times: Int = 1) {
    repeat(times) {
        println("Hello, $string")
    }
}

printHello("world") // Can call the function without specifying times
```

You can also use [Named Arguments](https://kotlinlang.org/docs/functions.html#named-arguments) in various situations, such as when clarifying which parameter you want to specify a value for, or when you want to specify a value regardless of the order of parameters defined in a function. For example, [joinToString()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/join-to-string.html) has six parameters such as `separator`, `limit`, and `truncated`, but default values are specified, and named arguments can be used as follows.

```kotlin
listOf("A", "B", "C", "D").joinToString(prefix = "[", postfix = "]")
```

Named Arguments are optional in Kotlin, and basically, just like in Java, there is no problem just passing the values in the order of the parameters defined in the function. However, in Swift, this is reversed, and when creating an instance of sturct or calling a function, parameters must basically be passed in a form like Named Arguments. It seems that this is read as [Argument Label](https://docs.swift.org/swift-book/LanguageGuide/Functions.html#ID166).

```swift
func printHello(string: String) {
    print("Hello \(string)!")
}

printHello(string: "world") // Specify string with the function argument label
```

However, like Kotlin, you can specify a default value, in which case you can omit the parameter.

```swift
func printHello(string: String, times: Int = 1) {
    var count = 1
    repeat { // Similar to Kotlin's do-while loop
        print("Hello \(string)!")
        count += 1
    } while (count <= times)
}

printHello(string: "world") // times is omitted
```

You can also omit the Argument Label by using an underscore.

```swift
func printHello(_ string: String, times: Int = 1) {
    var count = 1
    repeat {
        print("Hello \(string)!")
        count += 1
    } while (count <= times)
}

printHello("world") // string is omitted
```

From the side that defines the function, it may not seem very similar, but as the side that calls it, I think the unique feature is that the code can be written in a very similar way.

### Range

In Kotlin, you can easily define a range of numbers using [rangeTo()](https://kotlinlang.org/docs/ranges.html#:~:text=values%20using%20the-,rangeTo(),-function%20from%20the). This function is defined as an [operator](https://kotlinlang.org/docs/keyword-reference.html#operators-and-special-symbols), so you can easily use it with `..`. With a range defined like this, you can do various things such as get the minimum and maximum values and convert it to a list.

```kotlin
// Define the range
val min = 10
val max = 20
val range = min..max

// Get the minimum and maximum values
println(range.start) // 10
println(range.endInclusive) // 20

// Convert the range to a list
val intList = range.toList() // [10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
```

You can also define ranges in Swift using the [range operator](https://docs.swift.org/swift-book/LanguageGuide/BasicOperators.html#ID73). It is also similar in shape and is written as `...`. It feels almost the same, except that Swift uses one more dot than Kotlin and exposes the minimum and maximum values under different property names.

```swift
// Define the range
let min = 10
let max = 20
let range = min...max

// Get the minimum and maximum values
print(range.lowerBound) // 10
print(range.upperBound) // 20

// Convert Range to Array
let array = Array(range) // [10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
```

However, this is easy to miss from the code above, but the two languages handle range types a little differently. In Kotlin, the return value of `rangeTo()` becomes something like `IntRange` or `LongRange` depending on the original value type, and the minimum and maximum values you read from properties use that same underlying type.

By contrast, the `min...max` form used here becomes a Swift [ClosedRange](https://developer.apple.com/documentation/swift/closedrange). In this example, `lowerBound` and `upperBound` are still `Int`, but Swift represents the range itself as a generic type such as `ClosedRange<Int>` rather than a dedicated name like `IntRange` or `LongRange`.

## Exclusive to Swift

Up until now, I have been talking about how much of the code can be written in the same way as with Kotlin from the perspective of a Kotlin user, but from now on I would like to summarize some of the things that I thought were a little different.

### Omitted in method/property calls

In Kotlin, `this` can be omitted if it is clear that it refers to itself, such as [apply()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html). It's like below.

```kotlin
data class Student(
    val name: String,
    var age: Int = 0
)

val studentA = Student(name = "A").apply { age = 18 } // Student(name=A, age=18)
```

In this way, except for specific cases such as when using `this` or explicitly importing a target, in Kotlin, the basic rule is to indicate which class member is being called in a format like `Class.method()`.

However, in the case of Swift, the situation is a little different. It's more relaxed, and if the target is clear based on the compiler, it feels like it can be abbreviated to something like `.method()`. The following is an excerpt of the code presented in the SwiftUI tutorial, and you can see that `filter` is an enum called `FilterCategory`, so it is used in the form of `.all` in the ternary operator.

```swift
struct LandmarkList: View {
    @State private var filter = FilterCategory.all

    enum FilterCategory: String, CaseIterable, Identifiable {
        case all = "ALL"
        case lakes = "Lakes"
        case rivers = "Rivers"
        case mountains = "Mountains"
        
        var id: FilterCategory { self }
    }

    var title: String {
        let title = filter == .all ? "Landmarks" : filter.rawValue
        return showFavoritesOnly ? "Favorite \(title)" : title
    }
}
```

### Protocol

Swift has something called [Protocol](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html), which can be used in much the same way as the Java or Kotlin interface. I don't think there is much of a difference up to this point, but I think the difference you can experience is that when you actually define a struct, class, enum, etc., you need to adopt a protocol as needed.

For example, if you define one data class in Kotlin, the following members will be automatically added.

- equals()
- hashCode()
- toString()
-componentN()
-copy()

However, such members are generally not added to Swift's structs, classes, enums, etc. Therefore, if there are necessary members, it is necessary to adopt and implement the relevant protocols. For example, if you want to use a hash value, use [Hashable](https://developer.apple.com/documentation/swift/hashable),
Use [Codable](https://developer.apple.com/documentation/swift/codable) to convert to JSON, [Identifiable](https://developer.apple.com/documentation/swift/identifiable) if you want to loop with a list, [CaseIterable](https://developer.apple.com/documentation/swift/caseiterable) if you want to loop over all enum cases, and [Equatable](https://developer.apple.com/documentation/swift/equatable) if you want to compare identity.

Of course, Java and Kotlin still require interfaces and annotations where appropriate, but in Swift the conveniences you are used to from Kotlin may not be available when you define structs, classes, and similar types, so you should be careful here.

### some

In Swift, there are some keywords that have a slightly different feel. To explain the keyword, let's first assume that we have the following protocol and struct definitions.

```swift
protocol Something {
    func define()
}

struct GoodThing: Something {
    func define() {
        print("It's good thing!")
    }
}
```

If you have code like the one above, you can use some slightly unique keywords in variable type declarations and function return values. It's called `some`. When actually used, the code will be as follows.

```swift
var good: some Something = GoodThing()

func returnSomething() -> some Something {
    return GoodThing()
}
```

From this alone, I don't know what the keyword `some` is. If we bring in the concept of Kotlin here, it is similar to `<T extends Something>`. If you have experience with Kotlin or Java, I think this will give you a good idea of what these types mean. In other words, `some` indicates an instance that satisfies a certain protocol. In Swift, even if an object satisfies that requirement, it may not be possible to use the protocol directly as a variable type or function return value. In that case, you can avoid the problem by using `some`. It is similar to using an interface in Java or Kotlin regardless of the specific implementation. Thanks to this keyword, in SwiftUI, if [View](https://developer.apple.com/documentation/swiftui/view) is satisfied, it can be treated as any component that makes up the screen.

However, even though it is conceptually the same as handling an interface, the sense of writing the code is completely different, so I think you have to be careful here.

### Compiler Control Statements

Swift has a specification called [Compiler Control Statements](https://docs.swift.org/swift-book/ReferenceManual/Statements.html#ID538), which allows you to specify processing at compile time. For example, in the SwiftUI tutorial, there are cases where one application is implemented and used to realize different functions depending on the OS. For example:

```swift
// Use notifications when running on watchOS
#if os(watchOS)
WKNotificationScene(controller: NotificationController.self, category: "LandmarkNear")
#endif

// Use settings when running on macOS
#if os(macOS)
Settings {
    LandmarkSettings()
}
#endif
```

In the case of Kotlin, there may be situations in which such settings are necessary when implementing an app on Android, but in my experience with backends, I had not had many cases where I controlled the compiler with code, so it was quite a new feeling.

## Finally

How was it? I was going to talk about SwiftUI, but the amount of time I spent talking about Swift turned out to be quite large, so I think I'll talk about SwiftUI in the next post. However, there were many interesting things about Swift alone, so I was able to gain a lot of experience by going through the tutorials, so I think I made a good choice.

Also, I feel that there are some similarities between Kotlin and Swift, so I feel that if you have experience with one, it will be easier to get started with the other. I'm a little happy when I think that this is probably because they have background knowledge, or schemata in foreign language education (which is my major). I'm glad I chose Kotlin after all.

See you soon!
