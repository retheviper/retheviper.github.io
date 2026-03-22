---
title: "I want Java to evolve like this"
date: 2020-02-03
translationKey: "posts/java-wishes"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - java
---
Java has long been well-regarded in the industry for its productivity, performance, and stability, and recent updates have added various features. I mainly use version 11 at work, but I think I'll be using it when the next LTS version, 17, is released. So even now, I always check the update history when a new version is announced, but I can't help but feel that the number of changes to the language specifications itself is smaller than the addition of new APIs.

In terms of productivity, which is one of the reasons why Java became popular, it is now inferior to Python, JavaScript, etc., and the advantage of ``easy to read code'' has suddenly changed to ``too verbose''. I like Java, and I haven't given up on the specifications of the language itself, but there are times when I use it and think that it's inconvenient and that I wish it were the same as other languages. In this post, I would like to talk about the inconveniences I experienced while using Java at work, and the areas I would like to improve compared to other languages. I think it will be a good learning experience for me to understand what Java is like compared to other languages.

## Improved Optional notation

Optional, which I introduced in [the previous post](../java-optional/), is one of the APIs that is widely used not only in Java but also in other languages. Rather, it seems that Java's Optional was introduced after being influenced by other languages. I use it a lot myself now, and I think it's very convenient, but there are still some things that I think are inconvenient compared to other languages.

I mentioned earlier that Java's code is more redundant than other languages, but let's first compare how it is in actual code. I will explain this with an example of reading a certain field of a complexly nested object and returning a default value if it is Null.

## Java

In Java code, wrap the first object with Optional, and chain map() to nested fields and methods to change the wrapping target one after another. And finally, if the target object is Null, use methods such as orElse() to set the default value.

```java
// Original object
SomeClass object;

// Deeply nested
Optional.ofNullable(object).map(obj::getProp1).map(prop1::getProp2).map(prop2::getProp3).orElse("default");
```

## C#

Although the language itself is similar to Java, C# is much more advanced than Java, perhaps because it is younger. The code is written in the same way here as in Kotlin. The only difference is that the object itself does not declare in advance the possibility that it may be null.

```csharp
object?.prop1?.prop2?.prop3? ?? "default";
```

## JavaScript

Optionals in JavaScript are also not much different from C#.

```js
object?.prop1?.prop2?.prop3? ?? "default"
```

## Swift

Swift is no different.

```swift
object?.prop1?.prop2?.prop3 ?? "default"
```

## Kotlin

Kotlin is also the same if you look at the Elvis operator's unique expression for specifying default values.

```kotlin
object?.prop1?.prop2?.prop3 ?: "default"
```

Compared to examples from other languages, Java's Optionals certainly seem redundant. Therefore, I think it would be a good idea to introduce Optional as a basic specification of the language in the future and make it possible to express it as ? instead of wrapping it. Introducing ? does not impair the readability of the code.

## Multiple Return Statements

According to the Java specification, only one object can be the return value of a method at any given time, but languages like Python allow multiple return values to be specified. Of course, to overcome Java's single-return-value constraint, it is often possible to return multiple objects or data in a Bean or Collection, so this is just syntactic sugar, but if there is a convenient method, you might want to use it.

## Java

If you want to receive multiple pieces of data as the processing results of a method, in Java you would use Beans and Collections, as mentioned earlier. Below is an example where the return value is multiple numbers.

```java
// Method with multiple return values
public List<Integer> multipleReturn() {
    return Arrays.asList(1, 2);
}

// Read the return values
List<Integer> data = multipleReturn();
```

## C#

In C#, you can obtain multiple pieces of data in a similar way to Java. In reality, there are apparently ways to use ref/out parameters, structs, and classes, but I think using Tuple is more convenient than Java. The way to write it differs depending on the version of C#, and while the old way of writing it is not much different from using Collection in Java, the new way of writing it is quite convenient. Below is the code for the two examples.

```csharp
// Method with multiple return values (before Java 7)
public Tuple<int, int> oldMultipleReturn()
{
    return Tuple.Create(1, 2);
}

// Get them as an object
var result = oldMultipleReturn();

// Method with multiple return values (Java 7 and later)
public (int, int) newMultipleReturn()
{
    return (1, 2);
}

// Get them as variables
(int one, int two) = newMultipleReturn();
```

### Python

In the Python example, you can write code in a similar way to C# 7 or later. It would be more convenient to retrieve all return values as a tuple or as individual variables within the same function.

```python
# Function with multiple return values
def multiple_return():
    return 1, 2

# Get each return value individually
a, b = multiple_return()
print(a) # 1

# Get all return values as a tuple
d = multiple_return()
print(d) # (1, 2)
```

## JavaScript

The writing style introduced in ES6 allows you to obtain multiple return values using code similar to Python.

```js
// Function with multiple return values
funtion multipleReturn() {
    return {
        first: 1,
        second: 2
    }
}

// Get each return value individually
var (first, second) = multipleReturn()
```

## Swift

Swift, like Optional, is not much different from JavaScript.

```swift
// Function with multiple return values
func multipleReturn() -> (Int, Int) {
    return (1, 2)
}

// Get each return value individually
let (first, second) = multipleReturn()
```

## Kotlin

In Kotlin, there are Pair and Triple, and they are easy to use.

```kotlin
// Function with multiple return values
fun multipleReturn(): Pair<Int, Int> {
    return 1 to 2
}

// Get each return value individually
val (first, second) = multipleReturn()
```

Writing code in Java has the advantage of making it easier to understand the role of methods, but Python is more convenient in that you can use return value objects and data as variables. The ability to define methods with multiple return values like this seems to be a feature of all modern programming languages. Will it be introduced to Java someday?

## Specify argument type or

Sometimes I think it would be useful to be able to specify multiple argument types for a single method. In Java, this is achieved using overloading.

## Java

```java
public void doSomething(String value) {
    // Handle the String case
}

public void doSomething(int value) {
    // Handle the int case
}
```

Of course, you can also declare the argument type as Object and use instancesof internally to determine it. However, if it is the former, there is a problem that the amount of code will be too large compared to what you want to do, and if it is the latter, the behavior may become strange if an Object other than the intended type is passed.

## TypeScript

TypeScript solves this by making it easy to specify multiple argument types.

```ts
function checkString(v: string | number) {
    if (typeof v === "string") {
        return true
    }
    return false
}
```

Up until now, when only the type of argument was different, I had overloaded it and written only common processing using private methods, but I think this will make it easier to understand the code. This is one of the features I would like to see introduced.

## Ternary Operator with Throw

If there are only two possible results, I think it is more convenient to use the ternary operator rather than if, as the code will be shorter and more convenient. However, Java's ternary operator cannot throw an exception. If the conditional expression does not apply, you have no choice but to write code like the following.

```java
// If x is 0 then number is also 0; otherwise throw an exception
int number = x -> {
    if (x == 0) {
        return x;
    } else {
        throw new RuntimeException();
    }
}
```

If you want to forcibly throw an exception using a ternary operator, there are ways to do it as follows.

```java
// Method with a generic return value that only throws an exception
public <T> T throwSomeException() {
    throw new RuntimeException();
}

// Call the method in the else branch
int number = x == 0 ? x : throwSomeException();
```

Personally, I think it's a waste of space to use if statements for results with only two choices, and there's no need to forcefully create a method to use a ternary operator, so I thought it would be nice if the ternary operator could simply throw an exception in the case of else. And after looking into it, it seems like it can be done in other languages.

## C#

In C#, there are two ways to do this. First, in versions before 7, this can be achieved by executing a Func that throws an exception in the case of else. And after 7, it seems that you can throw with the ternary operator normally. It's exactly the shape I wanted.

```csharp
// Before Java 7
int number = x == 0 ? x : new Func<int>(() => { throw new Exception(); })();

// Java 7 and later
int number = x == 0 ? x : throw new Exception();
```

## Kotlin

In Kotlin, there is no ternary operator, and the only thing you have to do is use if-else, making it simpler.

```kotlin
val i = if (x == 0) x else throw Exception("error")
```

## JavaScript

JavaScript allows you to throw exceptions in a similar way to C# before 7.

```js
var number = (x == 0) ? x : (function() { throw "error" }());
```

I don't want to go to the trouble of running a function and throwing an exception with a ternary operator, but just knowing that there is a way to do this is quite interesting. I think it would be great if Java could be written in a way similar to C# 7 and later.

## Access modifier extension

Public, private, and protected are commonly used access modifiers in Java. However, when creating a self-made library, there are times when I wish there was something in between public and private. For example, when you put together a Jar, you can restrict access so that it cannot be accessed outside of the Jar.

Modules were introduced in Java 9, but I don't want to use them as much as possible because of the problems I experienced...I think it would be nice if there was a modifier that would allow access even if different packages are in the same project. Also, if it is protected, it can be referenced in subpackages. Access modifiers that can only be accessed within the same module are provided as Internal in C#, Swift, and Kotlin, so it would be nice to see them introduced in Java as well.

## Finally

Looking at the recent update history of Java, useful features continue to be introduced one after another. In particular, version 14 introduced `record`, which has the same basic role as Lombok's `@Data`. The next LTS version will be 17, so there is still room for more improvement. Java 1.8 already had many useful features, but I hope it will continue to absorb good ideas from other languages and evolve.

Also, it was a good learning experience for me to research how things are done in other languages. TypeScript in particular is a language that I've been paying attention to recently, so if I have a chance, I'd like to try it out and compare it with Java. See you soon!
