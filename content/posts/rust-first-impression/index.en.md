---
title: "Rust as seen by a Kotlin programmer"
date: 2022-03-27
translationKey: "posts/rust-first-impression"
categories: 
  - rust
image: "../../images/rust.webp"
tags:
  - rust
  - python
  - kotlin
  - java
  - go
---
It was about two years ago that I decided I wanted to start learning Rust. At the time, I was mainly working with Java and Python, so I thought there were some performance-critical areas that I couldn't handle, so I thought I needed to try a natively compiled language. If possible, I thought that if I tried writing an app in a language that handles pointers without GC, I would be able to deepen my understanding of Java, which is my main job.

So I thought of Go and Rust as candidates. However, regardless of public opinion, from my point of view, I feel that Go is a little out of line with the goals we are pursuing. Especially after changing jobs, I started using the languages ​​Go and Kotlin at the same time, and for better or worse, I felt like I was starting to understand what I wanted to do.

So, I decided it was time to try out Rust, which I had been considering as my next choice. This also made me want to put aside public opinion and see if it actually suits me. Recently, there is a concept that we are in the era of [Polyglot Programmer](https://medium.com/@guestposts_92864/what-is-a-polyglot-programmer-and-why-you-should-become-one-e5629bf720c2) who can handle various languages, and there is a reputation that many programming languages ​​have become similar while absorbing the good points of each other, but in my case, I would like to try a new language called Rust as a way to explore what is suitable for me.

So, this time, I would like to start by reading this [The Rust Programming Language](https://doc.rust-jp.rs/book-ja/title-page.html) and comment on the parts that I found interesting, comparing it to other programming languages ​​that I have experienced so far. The document is long and my understanding is still shallow, so I'll just introduce a part of it first.

## Loop

In Rust, in addition to the traditional `for` and `while`, there is also `loop`, which does not specify a loop condition (infinite loop). If you want to end the loop under certain conditions, you can do so by `break`. For example, it has the following form.

```rust
loop {
    // do something
}
```

In other languages, I think the most common form is `while(true)`. For example, in Python it would look like this:

```python
while true:
  # do something
```

The situation is no different for Kotlin and Java. It will look like this:

```java
while(true) {
    // do something
}
```

In the case of Kotlin, there is an extension function, so I thought why not define something called `loop`, but then the compiler wouldn't recognize it as a loop, so writing `break` would result in a compilation error. Therefore, it was not possible to create an extension function as shown below.

```kotlin
fun test() {
  loop {
    break // Compile error
  }
}

fun loop(doSomething: () -> Unit) {
  while(true) {
    doSomething()
  }
}
```

In Go, you can write a simple infinite loop by not writing a conditional expression in `for`.

```go
for {
    // do something
}
```

Personally, I think that `while(true)` and `for` without specifying a conditional expression are just conventions and do not lead to intuitive understanding, so I thought that adding the keyword `loop` would be easier to understand in terms of code readability. It's a small detail, but once it's set as a specification, it's difficult to change, so I feel that deciding what keywords to use is also important when designing a language.

## Array

In Rust, when extracting a part based on the index of an array, write it as follows. It needs to be defined using a reference (`&`), and the format of standard output is also a bit unique. Also, data extracted by specifying an index is defined as "another data type without ownership." It seems that the part of the array cut out here is called `slice` in Rust.

```rust
let arr = [0, 1, 2, 3, 4];
let slice = &arr[1..3];

println!("{:?}", slice); // [1, 2]
```

You can write code in a similar way in Python. Below is an example of code that behaves the same as above. However, the difference with Rust is that the data type of `slice` cut out here is also `list`.

```python
list = [0, 1, 2, 3, 4]
slice = list[1:3]
print(slice) # [1, 2]
```

In Kotlin, you can do the same by passing a [Range](https://kotlinlang.org/docs/ranges.html) object to the [List](https://kotlinlang.org/docs/collections-overview.html#list) function. A bit of a problem is that it is difficult to understand the range unless you remember how `Range` is written, which is unique to Kotlin. Fortunately, with the Intellij Idea 2021.3 update, you can now [Show hints](https://blog.jetbrains.com/idea/2021/10/intellij-idea-2021-3-eap-5/#inline_hints_for_ranges), so if you are using an earlier version, it is better to update.

```kotlin
val list = listOf(0, 1, 2, 3, 4)
val subList = list.slice((1..2))
println(subList) // [1, 2]
val subListUntil = list.slice((1 until 3))
println(subListUntil) // [1, 2]
```

In Java, you can do the same by passing a range of indexes to [List.subList()](https://docs.oracle.com/javase/8/docs/api/java/util/List.html#subList-int-int-).

```java
List<Integer> list = List.of(0, 1, 2, 3, 4);
List<Integer> subList = list.subList(1, 3);
System.out.println(subList); // [1, 2]
```

In Go, you can define it in exactly the same way as in Python. Also, it is the same as Rust that the view cut from the array by specifying the index range is called `slice`. However, unlike an array, Go's slice has a variable length.

```go
arr := []int{0, 1, 2, 3, 4}
slice := arr[1:3]
fmt.Println(slice) // [1 2]
```

## Immutability

In Rust, a variable declaration is basically just one `let` and is immutable. Of course, it is not impossible to define variables that can be changed, and you can reassign values ​​using the `mut` keyword as shown below. For example:

```rust
fn main() {
    let mut x = 5;
    println!("The value of x is: {}", x); // The value of x is: 5
    x = 6;
    println!("The value of x is: {}", x); // The value of x is: 6
}
```

In other programming languages, the mutability of a variable is often expressed using keywords when the variable is declared. Or, variables are basically mutable, and you use special keywords only when you want them to be immutable. However, in Rust, variables are fundamentally immutable. Is it safe to say that as a language without GC, some ingenuity to ensure memory safety appears here?

Of course, there are cases, such as in Python, where there is no distinction between variable declaration and reassignment.

```python
x = 5
print("The value of x is: {}".format(x)) # The value of x is: 5
x = 6
print("The value of x is: {}".format(x)) # The value of x is: 6
```

In Kotlin, immutable items are declared with `val`, and mutable items are declared with `var`.

```kotlin
val x = 5
x = 5 // Compile error
var y = 6
y = 7 // OK
```

For Java, it's the opposite of Rust. If `final` is not added, the structure is basically reassignable.

```java
int x = 5
x = 6 // OK
final int y = 6
y = 7 // Compile error
```

In the case of Go, there seems to be no way to make variables immutable. Therefore, reassignment is free, but I also feel like I want a keyword like Java's `final`.

```go
x := 5
x = 6
fmt.Println(x) // 6
```

## Shadowing

This is something I didn't expect at all, but the Rust documentation says that you can shadow variables. Reassignment is possible by adding the `mut` keyword, so I think that's fine, but I was told that it can be used to make a variable immutable while redefining it as a different data type.

Rust allows you to use shadowing to create code like this:

```rust
fn main() {
    let x = 5;
    let x = x + 1; // 6
    let x = x * 2; // 12

    println!("The value of x is: {}", x); // The value of x is: 12
}
```

You can do something similar with Python. If you write code with the same behavior as below, it will work without any problems. I think this is because there is no strict distinction between variable declaration and reassignment, but in terms of form it can be said to be exactly the same as Rust.

```python
x = 5
x = x + 1
x = x * 2
print("The value of x is: {}".format(x)) # The value of x is: 12
```

For Kotlin, shadowing is only possible in some cases. This happens when the argument of a function matches the name of the variable declared in that function.

```kotlin
fun shadow(value: Int) {
  val value = value + 1 // Name shadowed: value
  println(value) // `val` is printed instead
}
```

Shadowing is not possible in Java. However, the following forms are possible.

```java
class Clazz {
  private int value = 0;

  public void setValue(int value) {
    this.value = value;
  }
}
```

For Go, it's a little more complicated. In the example below, `x` is declared and assigned twice, but this is possible because the scopes are separated. You could say it's similar to the Kotlin case.

```go
x := 0
fmt.Println("Before the decision block, x:", x) // Before the decision block, x: 0

if true {
  x := 1
  x++
}
fmt.Println("After the decision block, x:", x) // After the decision block, x: 0
```

The behavior of the above code changes by reassigning `x` in the if statement as shown below.

```go
x := 0
fmt.Println("Before the decision block, x:", x) // Before the decision block, x: 0

if true {
  x = 1
  x++
}
fmt.Println("After the decision block, x:", x) // After the decision block, x: 2
```

In other languages, shadowing is discouraged as much as possible, but in Rust it is introduced as a feature, which is interesting. This also seems to be strongly related to "ownership," which I will discuss later.

## Ownership

When compared to other languages, one of the unique features of Rust is ownership. This was a pretty interesting concept since I've never worked with a language without GC. For example, in the case of Kotlin, when compiling with Native, it is said that [reference counting](https://blog.jetbrains.com/kotlin/2021/05/kotlin-native-memory-management-update/#kn-gc) is used. In the case of Java, Python, and Go, GC works to release memory occupied by unreferenced objects.

However, in Rust, all reference types except for scalar types such as constants, fixed-point numbers, logical values, and characters seem to have the principle that ``memory is freed once it is used'' and ``memory is freed when it goes out of scope.'' The distinction between reference types and scalar types reminds me of the relationship between primitive types and reference types in Java. The difference is that memory is freed more aggressively and aggressively.

Basically, I think it would be better to consider that variables that have been used once cannot be used twice, or that the value cannot be changed, but there were many other interesting things.

## Move

There is a concept related to ownership called move. It seems to depend on how variables and data actually interact. Let's take a look at the code below. There doesn't seem to be any problem.

```rust
let s1 = "hello";
let s2 = s1;

println!("s1 = {}, s2 = {}", s1, s2); // s1 = hello, s2 = hello
```

However, if you change the above `String literal` to `String`, a problem will occur. Take a look at the code below.

```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{}, world!", s1);
```

When I try to compile the above code, I get the following error:

```shell
error[E0382]: use of moved value: `s1`
 --> src/main.rs:4:27
  |
3 |     let s2 = s1;
  |         -- value moved here
4 |     println!("{}, world!", s1);
  |                            ^^ value used here after move
  |
  = note: move occurs because `s1` has type `std::string::String`,
which does not implement the `Copy` trait
```

In other words, the data in `s1` has been moved to `s2` and can no longer be used. So, if you want to guarantee the same data in two variables, you need to explicitly copy the values. For example:

```rust
let s1 = String::from("hello");
let s2 = s1.clone();
println!("s1 = {}, s2 = {}", s1, s2); // s1 = hello, s2 = hello
```

The reason for this is that in Rust, memory is freed when a variable moves out of scope, but if multiple variables use the same pointer, there is a risk of double freeing. Also, unlike `String literal`, `String` is not immutable, so I feel like this is intended to prevent the problem of changing the data in s2 by reassigning s1.

In fact, there are some language cases where such assignments are problematic. Let's look at Python for example. Since the two variables use the same pointer, you can see that the value of both has changed upon reassignment.

```python
s1 = "hello"
s2 = s1
print("s1 = {}, s2 = {}".format(s1, s1)) # s1 = hello, s2 = hello

s1 = "world"
print("s1 = {}, s2 = {}".format(s1, s1)) # s1 = world, s2 = world
```

Java treats String as immutable, so reassigning the value of s1 has no effect on s2. Kotlin behaves the same way when using JVM, probably because it basically generates JVM bytecode. See below.

```kotlin
var s1 = "hello"
val s2 = s1
println("s1 = $s1, s2 = $s2") // s1 = hello, s2 = hello

s1 = "world"
println("s1 = $s1, s2 = $s2") // s1 = world, s2 = hello
```

The same applies to Java as described above.

```java
var s1 = "hello";
var s2 = s1;
System.out.println(String.format("s1 = %s, s2 = %s", s1, s2)); // s1 = hello, s2 = hello

s1 = "world";
System.out.println(String.format("s1 = %s, s2 = %s", s1, s2)); // s1 = world, s2 = hello
```

Even in Go, variables cannot be defined as immutable, but safety is ensured against variables whose values may change through reassignment.

```go
s1 := "hello"
s2 := s1
fmt.Println(fmt.Sprintf("s1 = %s, s2 = %s", s1, s2)) // s1 = hello, s2 = hello


s1 = "world"
fmt.Println(fmt.Sprintf("s1 = %s, s2 = %s", s1, s2)) // s1 = world, s2 = hello
```

If you don't explicitly copy in Rust, the data itself will be moved, which is something you have to be careful about when coding, but fortunately it's a problem that can be confirmed at compile time, and I think it's not a good specification considering that it may correct habits that may cause unexpected behavior when working with other languages.

## Closure

In Rust, it is of course possible to define a closure as a function within a function, but it is written in the form `|val| val + x`. This is called a lambda in other languages. I feel like it's a somewhat unique writing style, but I think it's convenient compared to other languages ​​because types can be omitted. Of course, you can also express the type explicitly, so you can use it like the following.

```rust
fn main() {
    // When an i32 argument is required
    let closure_annotated = |i: i32| -> i32 { i + 1 };
    let closure_inferred  = |i     |          i + 1  ;

    let i = 1;
    println!("closure_annotated: {}", closure_annotated(i)); // closure_annotated: 2
    println!("closure_inferred: {}", closure_inferred(i)); // closure_inferred: 2

    // When there are no arguments
    let one = || 1;
    println!("closure returning one: {}", one()); // closure returning one: 1
}
```

In Python, you can write it as follows. Of course, you can define functions within functions, but I think it's more convenient to use lambda. However, [type hints can be used from 3.5](https://docs.python.org/3/library/typing.html), and if you want to reliably check for errors at compile time, I feel it would be better to write the type explicitly.

```python
closure = lambda x : x + 1
print(closure(1)) // 2
```

It is easy to define in Kotlin, but at least the type of the argument needs to be written. Or maybe you need to specify a type for the variable.

```kotlin
val closure = { x: Int -> x + 1 }
println(closure(1)) // 2
```

Java does not allow you to define methods within methods, and you must use `Functional Interface`, which was added in 1.8. Also, starting with version 10, type inference can be used with `var`, but there is a restriction that Functional Interface cannot be declared as var. It has the most restrictions compared to other languages.

```java
Function<Integer, Integer> closure = i -> i + 1;
System.out.println(closure.apply(1)); // 2
```

In Go, it is not impossible to define functions within functions, but you cannot write them like lambdas in other languages, and you can define them as anonymous functions. Also, since you need to specify the type, it's like defining a complete function except for the name and assigning it to a variable.

```go
closure := func(x int) int {
  return x + 1
}
fmt.Println(closure(1)) // 2
```

There is also another feature of Rust in closure. This is how to write a function that takes a closure as an argument. It is like using generic for closure and writing the closure inside the function using the keyword where. In other languages, the way you write it doesn't change much even if closure is an argument, but it's interesting that Rust has a completely different form. For example, the code would be as follows.

```rust
// Function that takes a closure `F` as an argument
fn apply_to_3<F>(f: F) -> i32 where
    // `F` is a closure that takes i32 and returns i32
    F: Fn(i32) -> i32 {
    f(3)
}

fn main() {
    let double = |x| 2 * x;
    println!("3 doubled: {}", apply_to_3(double)); // 3 doubled: 6
}
```

## Finally

I haven't read half of the documentation yet, and I haven't actually created any apps, so I'm fully aware that this post alone won't be enough, but I found a lot of interesting things while learning a different language for the first time in a while, so I thought I'd write my thoughts for now.

Rust has it that Rust's compiler is excellent, and that there are many moments where you can learn a lot just by building an app according to the compiler's instructions, and that the language itself is well designed, so I would like to continue blogging about what I noticed, felt, and learned while studying it. I don't want this project to end with just this post, so I'd like to do my best with this this year.

See you soon!
