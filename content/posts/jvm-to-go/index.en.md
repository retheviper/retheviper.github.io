---
title: "Things to get hooked on when using Go for people with JVM language experience"
date: 2022-04-17
categories: 
  - go
image: "../../images/go.webp"
tags:
  - java
  - kotlin
  - go
---
It's been a year since I started using Go and Kotlin at a new job, even though I had no experience with Java, Python, or even a little JavaScript. Modern programming languages ​​tend to evolve in a convergent manner, so if you can learn one language, you will be able to learn or read other languages ​​as well.

However, different languages ​​mean different design philosophies, so there are cases where code written with the expectation of the same result does not turn out exactly as expected. I think there are two main reasons for this: inertia: ``This is how it was in that language, so it's probably the same in this language,'' and the belief that ``this is a special specification of this language.'' (Actually, that's what I did)

So, this time I would like to introduce some of the pitfalls when using Go as an engineer with a Java/Kotlin background. This is just my personal experience, but I hope it will be of some help to those who are writing code in Go from now on.

## time

In Go, [time](https://pkg.go.dev/time) exists as a standard library for handling time, and it can be used as follows.

```go
// Get the current time
now := time.Now()
// Get a specific time (2022-04-01 12:30:00 +0000 UTC)
someDay := time.Date(2022, 4, 1, 12, 30, 0, 0, time.UTC)
```

In Java/Kotlin, there is a corresponding API called [java.time](https://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html). Compared with Go, the classes are divided not only by "time" but also by more specific units.

```java
// Year
Year year = Year.now();
// Year and month
YearMonth yearMonth = YearMonth.now();
// Year, month, and day
LocalDate date = LocalDate.now();
// Time
LocalDateTime time = LocalDateTime.now();
```

However, this is not the only difference. Naturally, the implementation of the library differs depending on the language, so the processing results may also differ in some cases. Typically, when dealing with time in units of "months", depending on the implementation, there are cases where Go does not provide the intended range. For example, suppose you have the following code.

```go
func getOneMonthBefore(t *time.Time) time.Time {
    // Return one month earlier by passing -1 for the month
    return t.AddDate(0, -1, 0)
}
```

The code seems to have no problems, but in some cases the following problems occur.

```go
date := time.Date(2022, 3, 31, 0, 0, 0, 0, time.UTC)
oneMonthBefore := getOneMonthBefore(&date) // 2022-03-03
```

The reason the above code results in "March 3rd" instead of "February 28th" is because the processing is done as follows.

1. From `2022-03-31` to `2022-02-31` one month ago
2. Since the date `2022-02-31` does not exist, correct the date from the last day of February.
3. Add the date by the difference of 31 days from the 28th, the last day of February.

Therefore, this kind of result can be obtained ``if there are fewer dates in the previous month than the reference month''. However, if it is one month before the end of the month, the intention is to expect `2022-02-28`. It is quite possible that the processing that a person thinks and the result calculated by the actual code are different, but the [AddDate](https://pkg.go.dev/time#Time.AddDate) document does not specifically mention that the processing will be like the one above, so I think there is a possibility of misunderstanding.

Also, in the case of `LocalDate` in Java/Kotlin, the result becomes `2022-02-28` as expected, so engineers with Java/Kotlin experience may unconsciously write code that relies on that behavior. By the way, the reason the `LocalDate` code gives a different result from Go is that [minusMonths()](https://docs.oracle.com/javase/8/docs/api/java/time/LocalDate.html#minusMonths-long-) ultimately calls the following method.

```java
private static LocalDate resolvePreviousValid(int year, int month, int day) {
    switch (month) {
        case 2:
            day = Math.min(day, IsoChronology.INSTANCE.isLeapYear(year) ? 29 : 28);
            break;
        case 4:
        case 6:
        case 9:
        case 11:
            day = Math.min(day, 30);
            break;
    }
    return new LocalDate(year, month, day);
}
```

Therefore, if you want to have the date one month before March 31st become February 28th in Go, I think you should consider the following two methods.

1. Add processing that takes into account leap years and the end of each month, just like LocalDate
2. If the month obtained by `AddDate()` is the same month as the reference time, return the last day of the previous month.

In the former case, it can be achieved by calculating the month and then adding the same process as `resolvePreviousValid()` above, and in the latter case, it is possible to obtain the last day as shown below, so please refer to it.

```go
date := date.AddDate(2022, 3, 0, 0, 0, 0, 0, time.UTC) // If you specify day 0 in March, it becomes February 28
```

## map

There are two ways to declare variables in Go:

```go
// Declare only the type
var intSlice []int
// Declare with initialization
stringSlice := make([]string, 10)
```

The problem is that depending on how you declare it, problems can arise when adding elements. Let's look at the slice example first. There is no problem in adding elements using [append](https://pkg.go.dev/builtin#append) whether it is declared with `var` or initialized with `make()`.

```go
var intSlice []int
// Add a value to the slice
intSlice = append(intSlice, 1) // [1]
```

However, in the case of map, problems may occur if you declare it with `var`. The code below will result in a nil pointer.

```go
var stringMap map[string]string
stringMap["A"] = "a" // panic: assignment to entry in nil map
```

This is because variables declared with `var` are basically nil. It is natural that an error will occur because I tried to add an element to a nil map, but on Goland (Intellij) no warning is displayed and the compilation passes successfully, so I don't know whether this code will work until I actually run it. In fact, if you handle slices first, you can append even nil files, so I think it's easy to think, "Is this okay in Go?"Even in Java and Kotlin, it is not possible to add elements to a Map that has not generated an instance, but I think this is a case that is considered to be a ``speciality of Go'' rather than a Java or Kotlin week, so I think it is something to be careful about.

## switch

Go's [switch](https://gobyexample.com/switch) is very similar to Java. However, although they are similar in shape, there is a crucial difference. First, let's take a look at Java's switch. Suppose you have code like the following:

```java
int i = 1;
switch (i) {
    case 0:
       System.out.println("zero");
    case 1:
       System.out.println("one");
    case 2:
       System.out.println("two");
    default:
       System.out.println("else");
}
```

In Java's switch, unless you explicitly write `break`, even if it branches to a case that matches the condition, it will also flow in the cases below it. So, the result of running the above code will be as follows.

```text
one
two
else
```

In Kotlin, it becomes a `when` expression, and processing ends by executing the code block that matches the condition even without `break`. For example, let's say you write code like the following.

```kotlin
val i = 1
when (i) {
    0 -> println("zero")
    1 -> println("one")
    2 -> println("two")
    else -> println("else")
}
```

You can see that the execution result is different from Java. This is because it is omitted and the processing is `break` for each branch.

```text
one
```

Let's take a look at what happens in the case of Go's switch. It looks similar to Java in form, but is the result the same?

```go
i := 1
switch i {
case 0:
    fmt.Println("zero")
case 1:
    fmt.Println("one")
case 2:
    fmt.Println("two")
default:
    fmt.Println("else")
}
```

The result of running the above code is the same as in Kotlin. In other words, it will print "one". This is because Go's switch also does `break` for each branch, just like Kotlin. So if you want the same result as in Java, you need to add `fallthrough` and explicitly say to proceed to the next branch. It is as below.

```go
i := 1
switch i {
case 0:
    fmt.Println("zero")
    fallthrough
case 1:
    fmt.Println("one")
    fallthrough
case 2:
    fmt.Println("two")
    fallthrough
default:
    fmt.Println("else")
}
```

If you have experience with Java, you may end up omitting `fallthrough`, thinking that the behavior will be the same since the styles are similar. Please be careful here, as the language is different and the usage is also different.

## if

In Go, if the condition of the if statement seems strange, it will not compile. For example, take a look at the example below.

```go
type Role int

const (
    SystemAdmin = 1
    Operator    = 2
    Developer   = 3
)

type User struct {
    Name string
    Role Role
}

// Return an error if the user is neither SystemAdmin nor Developer
func checkRunnableUser(u User) error {
    if u.Role != SystemAdmin || u.Role != Developer {
        return errors.New("user is not runnable")
    }
    return nil
}

func Test_checkRunnableUser(t *testing.T) {
    u := User{Name: "John", Role: Operator}
    err := checkRunnableUser(u)
    if err != nil {
        t.Errorf("unexpected error: %s", err)
    }
}
```

If you try to compile the above code, you can see that Goland (Intellij) shows a warning in the condition and when you compile it shows the error message `suspect or: u.Role != SystemAdmin || u.Role != Developer`. As you can see from the error message, this is because the condition of the if statement is incorrect. In order to satisfy the requirement that "User's Role is only allowed if it is SystemAdmin or Developer", it is necessary to use `and` instead of `or`. Therefore, if you modify the condition of the if statement as shown below, it will work as intended, and no warnings or compile errors will occur in the IDE.

```go
// Return an error if the user is neither SystemAdmin nor Developer
func checkRunnableUser(u User) error {
    if u.Role != SystemAdmin && u.Role != Developer {
        return errors.New("user is not runnable")
    }
    return nil
}
```

In the case of Java, Intellij will display a warning that the condition is suspicious, but like Kotlin, no errors will occur at compile time. So you will be able to run it, but if you don't check the part where the warning appears, you won't realize that the logic is wrong until you actually run it.

```java
enum Role {
    SYSTEM_ADMIN, OPERATOR, DEVELOPER
}

record User(String name, Role role) {}

static void checkRunnableUser(User user) {
    if (user.role() != Role.SYSTEM_ADMIN || user.role() != Role.DEVELOPER) {
        throw new IllegalArgumentException("user is not runnable");
    }
}

public static void main(String[] args) {
    checkRunnableUser(new User("John", Role.SYSTEM_ADMIN));
}
```

However, the problem is that when I write the same process in Kotlin, Intellij does not issue any warnings and compiles successfully. As in the Java case, the code will cause an error at runtime, but since there are no warnings, I think it's easy to miss something about why it's working as intended if you don't look at the code carefully.

```kotlin
enum class Role(val value: Int) {
    SystemAdmin(1),
    Operator(2),
    Developer(3)
}

data class User(val name: String, val role: Role)

fun checkRunnableUser(user: User) {
    if (user.role != Role.SystemAdmin || user.role != Role.Developer) {
        throw IllegalAccessException("user is not runnable")
    }
}

fun main() {
    checkRunnableUser(User(name = "John", role = Role.Operator))
}
```

I think the Go compiler is certainly better in that it can detect errors in advance at compile time. However, if you are used to Kotlin or Java, I think that rather than noticing that the conditions are wrong, you may not notice the true nature of the problem, such as "Maybe it's because I'm using `const`" or "Should I use switch?"

I think this is an appropriate example of how, while it is most important to write conditions correctly in the first place, problems can arise if you write code in a different language using habits formed in another language.

## Range loop

There is only a for statement in Go's loop, but there is also a [Range](https://go.dev/tour/moretypes/16) loop in addition to the traditional form that uses index. It can be used in the following formats.

```go
var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

for i, v := range pow {
    fmt.Printf("2**%d = %d\n", i, v)
}
```

If you were to write code that would produce the same result in Kotlin, it would probably look like this: You can tell at a glance.

```kotlin
// kotlin
val pow = listOf(1, 2, 4, 8, 16, 32, 64, 128)

for ((i, v) in pow.withIndex()) {
    println("2**$i = $v")
}
```

However, Go has pointers, and there are cases where problems occur when trying to use pointers in a Range loop. For example, let's take the following example.

```go
var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

// Store references to pow in a new slice
var ppow []*int
for _, v := range pow {
    ppow = append(ppow, &v)
}

// Print the values of ppow
for i, v := range ppow {
    fmt.Printf("2**%d = %d\n", i, *v)
}
```

As expected, each `ppow` should contain a reference to `1, 2, 4, 8, 16, 32, 64, 128`. However, when I run this code, all values ​​are actually the same as `128`. This is because all values ​​referenced without a Range loop will refer to the same address.

Therefore, when processing a loop using a slice that uses pointers, you need to reassign `v` as shown below or refer to it by index to avoid the problem.

```go
var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

// Store references to pow in a new slice
var ppow []*int
for _, v := range pow {
    v := v // Copy v here
    ppow = append(ppow, &v)
}

// Print the values of ppow
for i, v := range ppow {
    fmt.Printf("2**%d = %d\n", i, *v)
}
```

If you accidentally use a Range loop, you may run into problems when using slices that use pointers to process loops, so be careful.

## Finally

I've given a few examples, but since I haven't had much experience writing apps in Go, and I don't have a deep understanding of the language, I think there's a possibility that I'll run into various problems in the future. I would like to post it on my blog like this again. I'm happy because I have something to write about on my blog, but I'm not happy about it because posting from a failure ends up being painful for me...

Anyway, all of the problems I have listed here are things I have experienced myself, but I think the important thing is to use your own background knowledge when trying a different language, but also to not let that bias become your bias, and not to get ahead of yourself. It's like you have to be careful when touching new things, not just Go.

See you soon!
