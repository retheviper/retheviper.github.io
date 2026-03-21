---
title: "Go as seen by a Java programmer"
date: 2021-03-21
categories: 
  - go
image: "../../images/go.webp"
tags:
  - go
  - java
  - kotlin
  - exception
---

It's a sudden post for this blog, but after changing jobs, I ended up using Go a little due to work commitments. I've always wanted to try Go or Rust, but I get a little scared when it comes to modifying an app written in a language I've never used before. So, you should know at least a little bit about Go. So, this time I would like to talk about what I felt after getting a little exposure to Go from the perspective of a Java programmer.

I think Go has many characteristics, but to be honest, the fact that it has a GC and no VM is not really a noticeable difference. Since there's no VM, I can only say that it's probably faster than some other languages.

As someone who actually uses the language to write code, I would like to focus more on the things you need to be aware of when writing code than on such features. First of all, how is it different from other languages ​​that I have come into contact with before? For example, what should I do when writing loops and conditions? Is it okay to write code as I have always done? What should I be careful about? This time, I would like to write about my impressions after actually touching Go a little from that perspective.

## Maybe you need to change your way of thinking

After playing around with Go a little bit, I realized that even the most basic parts require a very different approach than when writing Java. In my case, I have tried Python, JavaScript, TypeScript, and Kotlin in addition to Java, and I can write JavaScript and TypeScript in the same way as I write Java, and Kotlin is basically a simplified version of Java. Python is quite different, but I feel like I can write whatever code I want without any grammatical constraints, so I didn't mind the grammatical differences.

However, in the case of Go, things are a little different. This is because not only is the writing style slightly different from Java, but there are also differences at the functional level. I think the fact that they are different at the functional level means that it is not a good idea to write code that is simply a slight modification of Java code. So I thought maybe I needed to change my way of thinking. My impressions of Go from that perspective are as follows.

## Looks similar but isn't

The first thing that stands out is the grammar. Of course, the overall structure is not that different from the so-called C family of programming languages, but compared to Java there are many differences beyond syntax. For example, Go has a similar construct to the [Python assignment expression](https://docs.python.org/3/whatsnew/3.8.html#assignment-expressions), also called the `Walrus Operator`; `if` conditions can omit parentheses; imports are written as string literals; and there are no classes or keywords like `public` and `private`. It feels different not only when writing code, but also in how packages and application architecture should be designed compared with the Java and Kotlin I was used to.

I mentioned various differences, but let's simply take the code and compare it. For example, let's say you have the following code. There is a class that is responsible for calculations regarding numbers, and it determines whether the passed argument is an odd or even number and outputs the result to the standard output.

```java
package simple.math;

public Class Calculator {

    public void judge(int number) {
        boolean result = number %2 == 0;
        if (result) {
            System.out.println(number + " is even");
        } else {
            System.out.println(number + " is odd");
        }
    }
}
```

Let's convert this into Go code. For example, I think it would look like this:

```go
package simple.math

import "fmt"

func Judge(number int) {
    condition := number % 2 == 0
    if condition {
        fmt.Println(number, " is even")
    } else {
        fmt.Println(number, " is odd")
    }
}
```

Although it may not seem like there is much difference at first glance, there are some differences in details that you must be careful about. No matter how much the performance of an IDE has improved, if you write code that is completely different from the specifications of the language, it will not present the correct code. For example, if there are multiple imports, it will look like this:

```go
import (
    "fmt"
    "math"
)
```

In Python, you can also group imports in one line, and specify Alias ​​in `from` and `as`. However, the fundamental difference with Go is that Go itself can also manage packages like Maven or Gradle, so you can also import github packages. For example, you can import the Go web framework [Gin](https://github.com/gin-gonic/gin) with the code below.

```go
import "github.com/gin-gonic/gin"
```

Also, when declaring a variable using `:=`, it is possible only within a function, so it is necessary to understand the specification that when declaring a variable at the package level, it is necessary to use `var` normally. On the other hand, `var` declarations are not required for function arguments; types must be declared. This is very different from Java, where you need to write the type of the variable in any case (although it has become possible to use `var` with initialization in recent Java). Because of these small differences, it may be difficult to write code as if it were Java without an understanding of Go etiquette.

## Capital letters have a function

I'm sure each company has different rules, but in my experience, whether the language is Java or JavaScript, the following rules are often used when writing.

- Class and interface names are PascalCase
- Fields, methods, variables, and arguments are camelCase

Sometimes when I write code in Python, I use snake_case or kebab-case for URLs, but I often write code according to this rule even in private. And this is just a rule set by humans, so you don't have to follow it.

However, in Go, the meaning changes depending on whether a name uses PascalCase or camelCase. The exact difference depends on whether the first letter is uppercase or lowercase. This convention replaces `public` and `private`. Put simply, fields and functions that can be referenced from other packages start with an uppercase letter, and those that cannot start with a lowercase letter.

For example, take a look at the following. This is the code presented in [A Tour of Go](https://tour.golang.org). Here is an example of importing the `math` package and outputting the predefined `π` as standard output.

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    fmt.Println(math.pi)
}
```

The above code appears to be fine, but when executed it outputs the following error message:

```bash
./prog.go:9:14: cannot refer to unexported name math.pi
```

This means that it cannot be referenced from outside. So, if you change to the correct code, you need to modify the `main` function as follows.

```go
func main() {
    fmt.Println(math.Pi)
}
```

It seems that the name that starts with a capital letter and is defined so that it can be referenced from the outside is called [Exported Names](https://go-tour-jp.appspot.com/basics/3). Go doesn't have classes, so you import a package and only reference items defined in `Export Names` in the `.go` files that reside within that package. Therefore, it feels very different from the Java etiquette, where you create a class, generate an instance of that class, and call fields and methods that start with a lowercase letter.

## existence of a pointer

As any programmer knows, the question of whether or not a pointer exists is such that it affects each sense of the code considerably. In languages ​​that don't have pointers, such as Java and Kotlin, there is a GC between classes and methods, but Go has GC, so I don't think there is a memory problem like in C or C++, but you won't know until you use it directly.

Of course, it is possible to create objects that can be accessed anywhere by declaring them with `public static` in Java, or by adding the `Autowired` annotation in Spring. In Kotlin, you first need to define something similar to a class called `companion object`, but as a caller, the code is not much different from Java.

However, such `static` is generally used as a `constant` in Java and Kotlin. The use of `Autowired` is not much different from a singleton, and it is common to expect it to store a fixed value or always perform the same operation (unlike an idempotent function). Pointers are different because you may expect to rewrite their values directly.

I have yet to experience a language that handles pointers in earnest, and I haven't written much code that utilizes pointers in Go, so all I can say here is what I said above, but I think it will take some time for people like me who have only experience with languages ​​that don't have pointers to get used to it. There seems to be a lot of trial and error involved.

## Unique exception handling

When I look at code written in Go, I think what stands out most is the exception handling part. The languages ​​I have experience with (Java, Python, JavaScript, TypeScript, Kotlin) have a specification called the `try-catch` block for exception handling. Although there were slight differences in each language, the basic idea of ​​enclosing the place where an exception can occur in a block and then processing it remains the same. For example, it is common to handle exceptions with code like the following:

```java
public static void main(String[] args) {
    int result = 0;
    try {
        result = divide(1, 0);
    } catch (Exception e) {
        if (e instanceof ArithmeticException) {
            System.out.println("Do not divide by 0");
            return;
        }
    }
    System.out.println(result);
}

private static int divide(int numerator, int denominator) {
    return numerator / denominator;
}
```

However, Go does not have that kind of built-in exception flow. Instead, it is common practice to define the expected result and the error as return values for every function, and then have the caller check whether the error is `nil` after invoking it. If an error exists, the caller handles it. It's difficult to explain in words, so let's look at the actual code. Rewritten in Go style, the example becomes:

```go
func main() {
    result, err := divide(1, 0)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(result)
}

func divide(numerator int32, denominator int32) (int32, error) {
    if (denominator != 0) {
        return numerator / denominator, nil
    } else {
        return 0, errors.New("Do not divide by 0")
    }
}
```

In this way, if an error occurs in the function, it will be returned as is (in the code above, an error is intentionally created), and the caller will check whether there is an error and branch. This seems to have the advantage of ``clarifying the location where the error occurred.'' Indeed, the more code a `try-catch` block surrounds, the more you may not know which code could cause an exception. Functionality for handling exceptions gets mixed up with code that doesn't raise exceptions, and it becomes confusing. From that perspective, Go's approach has the advantage of separating errors and logic.

However, with Go's etiquette, an error check is performed every time a function is called, so it feels a little strange to have to write the same code every time. For example, I see code like the one below, what do you think? Maybe there's a smarter way and I just don't know about it...

```go
// Function that processes data by calling several other functions
func doSomething() (string, error) {
    // Call function 1
    result1, err1 := someFunction1()
    // Return an error if function 1 fails
    if err1 != nil {
        return "", err
    }
    // Call function 2 if function 1 succeeds
    result2, err2 := someFunction2(result1)
    // Return an error if function 2 fails
    if err2 != nil {
        return "", err
    }
    // Call function 3 if function 2 succeeds
    result3, err3 := someFunction3(result2)
    // Return an error if function 3 fails
    if err3 != nil {
        return "", err
    }
    // Call function 4 if function 3 succeeds
    result4, err4 := someFunction4(result3)
    // Return an error if function 4 fails
    if err4 != nil {
        return "", err
    }
    // ...and so on
}
```

## compiler is strict

If a compilation error occurs, you might think that you don't need to worry about it because the IDE will notify you, but what you might be surprised to find is how strict the compiler is. Personally, I often use interactive tools like `jShell` to verify code, but Go doesn't have that, so I try running code written in Vim in the terminal, and use [The Go Playground](https://play.golang.org/). And with this method, you can't expect much support like an IDE, so you often get compile errors.

However, there are various causes for compilation errors, and I think Go is particularly severe compared to other languages. For example, suppose you have the following code.

```go
package main

import (
    "fmt"
)

func main() {
}
```

When I run this in the terminal or The Go Playground, I get an error message like the one below.

```bash
./prog.go:4:2: imported and not used: "fmt"
```

The error is that the imported package is not being used. Furthermore, suppose you run the following code.

```go
package main

import (
    "fmt"
)

func main() {
    var result string
    fmt.Println("this is test program")
}
```

The result of running the above code is as follows.

```bash
./prog.go:8:9: result declared but not used
```

This time, the error is that the variable `result` is not used. In this way, if there are imports or variables that are not used in Go, an error will occur, so it can be said that it is stricter than in other languages. Therefore, you need to be very careful when modifying using Vim without plugins. Even with an IDE, it may be a little troublesome. (It might be a good idea to set it to delete unused packages and variables at the same time as linting.)

## lastly

There are many other minor differences, but that's all I can say for now. The characteristics of Go mentioned here from the perspective of a Java programmer may actually be just ``no problems once you get used to it.'' However, getting used to something can be quite difficult if there are things that you are already used to.

For example, in human languages, it is said that speakers of German find English easier to learn because the two languages are so similar. Conversely, [languages such as Chinese and Japanese are said to be the hardest for native English speakers](https://www.businessinsider.com/the-hardest-languages-to-learn-2014-5) because of differences in vocabulary, writing systems, and sentence structure. Programming languages also imitate human language to some extent, so the closer a new language is to what you already know, the easier it tends to be to learn. From that perspective, moving from Java to Go may look easy, but it can still be surprisingly difficult.

Of course, there are ex-Java programmers out there who find Go easier! I think there are many people who say so. Maybe I just can't keep up with it...
