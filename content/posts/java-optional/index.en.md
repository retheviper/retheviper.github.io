---
title: "Escaping the Hell of Null Checks"
date: 2020-01-05
categories: 
  - java
image: "../../images/java.webp"
tags:
  - optional
  - java
---

If there is one exception that is most frequently encountered while building applications in Java, it is NullPointerException (NPE). I think it's quite difficult for people who are coming into programming for the first time to understand the difference between "blank" and "null," but even if you can understand null, there are many times when you have trouble with NPEs that appear in unexpected places. In a sense, it is no exaggeration to say that developing applications in Java is a battle against NPEs.

So this time, I will introduce a method to deal with NPE. Let's look for ways to check for nulls and write safe code.

## Is a null check sufficient?

Let's say we have a method that retrieves the name of a certain student. If you pass a data object as an argument, it will retrieve the school, grade, group, and student information in order from there, and finally return the student's name as a String. If you express this in code, you can express it as follows, for example.

```java
public String getStudentName(Data data) {
    School school = data.getSchool();
    Grade grade = school.getGrade();
    ClassName className = grade.getClassName();
    Student student = className.getStudent();
    return student.getName();
}
```

If we were to express this in more concise code, it would look like this:

```java
public String getStudentName(Data data) {
    return data.getSchool().getGrade().getClassName().getStudent().getName();
}
```

If this method works as intended, the signature and code alone should make the intent and result clear. However, as you can see, this code can throw an exception at any time.

What if the student's name field is Null? No, what if the student, group, grade, or school is Null? What if the argument is Null? Any of these can potentially become NPEs, making them extremely dangerous codes.

The first possible solution here would be to write code that checks for null in advance and moves on to the next process only if it is not null. If it is Null, you can write an appropriate process (or a default value) to make it work as intended.

Now, let's add null check processing to the above code and turn it into Null Safe code. For example, you could change it to:

```java
public String getStudentName(Data data) {
    if (data != null) {
        School school = data.getSchool();
        if (school != null) {
            Grade grade = school.getGrade();
            if (grade != null) {
                ClassRoom classRoom = grade.getClassRoom();
                if (classRoom != null) {
                    Student student = classRoom.getStudent();
                    if (student != null) {
                        String studentName = student.getName();
                        if (studentName != null) {
                            return studentName;
                        }
                    }
                }
            }
        }
    }
    return "Sato"; // Default value
}
```

The above code is nested too much and is extremely difficult to read. So if even one item is unchecked, you won't know. It also makes it much harder to fix the code. The purpose of the method was to respond to a simple request, ``I want to know the student's name,'' but now there are so many null checks that it's hard to understand what the logic is for.

Even if you separate the if statements to avoid nested processing, the result will not change much. See the code below.

```java
public String getStudentName(Data data) {
    if (data == null) {
        return "Sato"; // Default value
    }
    
    School school = data.getSchool();
    if (school == null) {
        return "Sato"; // Default value
    }

    Grade grade = school.getGrade();
    if (grade == null) {
        return "Sato"; // Default value
    }
    
    ClassRoom classRoom = grade.getClassRoom();
    if (classRoom == null) {
        return "Sato"; // Default value
    
    Student student = classRoom.getStudent();
    if (student == null) {
        return "Sato"; // Default value
    }

    String studentName = student.getName();
    if (studentName == null) {
        return "Sato"; // Default value
    }

    return studentName;
}
```

I tried to make it easier to read by eliminating nested if statements. However, with this method, there are actually too many returns, which is not a very good process.

How can I fix code like this? Java provides an API that can be used in such cases. This time's topic is Optional.

## Introducing Optional

Modern languages seem to address this problem up front by either disallowing null assignment in many cases or by providing APIs for values that may be null. Kotlin and Swift, for example, treat nullable values and optionals as first-class concepts, and Python also has APIs such as `Union` and `Optional`. In response to that trend, Java introduced `Optional` in Java 1.8.

Optional is a new (but not really new anymore, since it was introduced in Java 1.8) way to handle potentially null objects. Basically, it seems that it was created under the influence of functional languages.

I am not especially familiar with functional languages myself, but the way `Optional` is used feels very different from traditional Java, much like lambdas do. Instead of branching around a null check directly, you express the flow through chained methods, which leads to a very different style of code.

If so, why use such a foreign API? Let's first look at what Optional is and what features it has.

## Easy to use

There are methods such as map() and filter() that have similar functionality to Collection[^1] and Stream[^2], and you can also use Lambda as an argument, so if you are familiar with Collections and Streams, you can easily adapt.

In order to use Optional efficiently, you first need to be comfortable with method chaining [^3] and lambdas, so let us assume you can already use `java.util.function`. Please refer to [the previous post](../java-functional-interface/).

## You can tell just by looking at it

Optional is an API created to wrap objects and handle when the object is null. Therefore, if you have an object wrapped in Optional, you are making it clear that the object can be null. Therefore, you will be able to identify code that may be null just by looking at the return value.

## Improves readability

While fulfilling its original purpose of checking for nulls, it also makes the code more concise and easier to read. Optional can also be used if the objects you want to retrieve are nested. The first object is checked for nulls, and then the nested objects are checked for nulls.

## Let's change the null check with Optional

Now, let's take a look at the actual code and see how it is possible to check for null in Optional. The previous method can be changed as follows.

```java
public String getStudentName(Data data) {
    return Optional.ofNullable(data)
                    .map(Data::getSchool)
                    .map(School::getGrade)
                    .map(Grade::getClassRoom)
                    .map(ClassRoom::getStudent)
                    .map(Student::getName)
                    .orElse("Sato");
}
```

If you are new to Optional, you may not be able to understand what it is doing at first glance, but you will notice that the amount of code has been reduced and readability has improved. Of course, this does not mean that the null check is omitted. Optional allows for such a simple, easy-to-understand, and safe null check.

## Optional method

Optional imports Singleton's `java.util.Optional<T>`, wraps the object, and has methods that behave depending on whether the wrapped object is Null or not. Let's look at each of these methods one by one.

## empty()

Create an empty Optional. As the name suggests, an empty Optional is empty, meaning that the object wrapped inside it is Null.

```java
Optional<String> emptyName = Optional.empty(); // String is null
```

## get()

Used when retrieving an object wrapped in Optional.

```java
String value = "Sato";
Optional<String> name = Optional.of(value);

System.out.println(name.get()); // Sato
```

## of(T value)

Generates an Optional that wraps the object passed as an argument. However, if the argument object is Null, the result of get() will also be Null.

```java
String value = null;
Optional<String> name = Optional.of(value);

name.get(); // Null
```

## ofNullable(T value)

It is the same as of() in that it generates an Optional that wraps the object passed as an argument, but if the argument object is Null, it returns an Optional generated by empty().

```java
String value = null;
Optional<String> name = Optional.ofNullable(value);

name.get(); // Optional<String>
```

## map(Function<? super T, ? extends U> mapper)

This method is similar to map() of Collection and Stream. Used to safely check complex nested fields. Objects retrieved using map will automatically become a class wrapped in Optional.

```java
String name = "Sato";
Student sato = new Student(name);

Optional<Student> student = Optional.ofNullable(sato);
String nameOfSato = student.map(Student::getName).get(); // Optional<Student> -> Optional<String>
```

The :: expression used here is called a Method Reference, and it is a writing style that allows you to expect the same effect as general Lambda by simply writing the target reference and method. Lambda allows you to write existing code more concisely, but it also allows you to omit variable names in arguments. Use this when you want to know what you are pointing to without writing the variable name, and when you want to call only one method.

```java
// Typical lambda for printing an argument
Consumer<String> print = name -> System.out.println(name);

// The same thing using a method reference
Consumer<String> print = System.out::print;

// Instance creation
Supplier<String> supplier = String::new;
```

## filter(Predicate<? super T> predicate)

filter() is another method that is easy to use if you are familiar with Collection and Stream methods. Returns a value only if it matches the condition (True due to Predicate). It is used when you want to add some processing other than simply determining whether it is null or not.

```java
// Traditional pattern
public String getSato(Student student) {
    String name = student.getName();
    if (name != null && name.equals("Sato")) {
        return name;
    }
}

// filter
public String getSato(Student student) {
    return Optional.ofNullable(student)
            .filter(s -> s.getName().equals("Sato"))
            .map(Student::getName)
            .get();
}
```

Since there is only one Optional element, if the result of the condition specified in filter is false, subsequent methods will be ignored.

## isPresent()

A method for determining whether the class wrapped in Optional is Null. It's simple: True if it's not Null, False if it's Null.

```java
String name = "Sato";
Optional<String> studentName = Optional.ofNullable(name);
studentName.isPresent(); // true
```

## ifPresent(Consumer<? super T> consumer)

Write a method that executes only if the wrapped object is not null.

```java
Optional<String> name = Optional.ofNullable(student.getName());
name.ifPresent(n -> System.out.println(n));
```

## orElse(T other)

If the object passed as an argument is null, the default value will be used. If you use this method, you do not need to write get().

```java
String defaultName = "Sato";
Optional<String> name = Optional.ofNullable(student.getName());
String result = name.orElse(defaultName); // Use defaultName if student.getName() is null
```

## orElseGet(Supplier<? extends T> other)

If the object passed as an argument is Null, execute the Lambda specified as the default value and return the result. If you use this method, you do not need to write get().

```java
Optional<String> name = Optional.ofNullable(student.getName());
String result = name.orElseGet(() -> "No name for student " + student.getNumber()); // Run the lambda if student.getName() is null
```

## orElseThrow(Supplier<? extends X> exceptionSupplier)

Throws an exception if the object passed as an argument is null. If you use this method, you do not need to write get().

```java
Optional<String> name = Optional.ofNullable(student.getName());
String result = name.orElseThrow(BusinessException::new);
```

## Things to note with Optional

Optional is convenient and safe for null checking, but it is not necessary to change all processing related to null to optional in all situations. This section explains the things to be aware of when considering the introduction of Optional.

## be conscious of performance

As some of you may have already noticed, since Optional wraps objects, it inevitably leads to a decrease in performance. Therefore, it is not a good idea to use Optional in situations where there is a null check. A simple null check can be done without being optional, and it's fast.

When writing processing when an object is Null using Optional, it is better to use orElseGet() rather than orElse(). orElse() is always executed even if it is not null. On the other hand, orElseGet() is a Lazy[^4] method, so better performance can be expected.

However, in some cases (such as when you have a static default value as a field), it is better to use orElse(), so it is important to judge on the spot. Be sure to understand where and when an instance of the default value you want to return is created.

```java
// Bad example (creates an instance that is discarded when the value is not null)
public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return Optional.ofNullable(student).orElse(Student::new);
}

// Good example
public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return Optional.ofNullable(student).orElseGet(Student::new);
}
```

Also, if you expect Null or a fixed default value as the return value, Null checking may be better than Optional.

```java
// Bad example (when the default value is always fixed)
private static Student defaultStudent;

public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return Optional.ofNullable(student).orElse(defaultStudent);
}

// Good example
public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return student != null ? student : defaultStudent;
}
```

## Combination of isPresent() and get() is not allowed

After checking whether an object is Null with isPresent(), the code that retrieves the object with get() is no different from a normal Null check. If you want to use the default value, use orElseGet(), and if you want to use an exception, use orElseThrow().

```java
// Bad example
public String getStudent(String name) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));

    if (student.isPresent()) { // `value != null` would be clearer here
        return student.get();
    } else {
        throw new NullPointerException();
    }
}

// Good example
public String getStudent(String name) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));
    return student.orElseThrow(NullPointerException::new);
}
```

Also, if you want to process only if the object is not null, use ifPresent().

```java
// Bad example
public void adjustScore(String name, int score) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));
    
    if (student.isPresent()) {
        student.get().setScore(score);
    }
}

// Good example
public void adjustScore(String name, int score) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));
    student.ifPresent(s -> s.setScore(score));
}
```

## not used in the field

In the first place, it seems that Optional is not intended to be used as a field. This is because Optional does not inherit from Serializable. Therefore, if you use Optional as a field in DTO etc., problems may occur before the null check.

```java
// Bad example
@Data
public class Student implements Serializable{

    private Optional<String> name; // Cannot be serialized
}

// Good example
@Data
public class Student implements Serializable{

    private String name; // Can be serialized
}
```

## not used in arguments

When you use an Optional as an argument to a method or constructor, you must generate an Optional as an argument each time you call it. Also, the code becomes complicated because the Null check logic is included internally as Optional. In this case, it is inconvenient because it is difficult to know what processing is being done internally and whether it is working as expected.

Therefore, it is easier to use and expect the intended processing by using ordinary objects as arguments to methods and constructors and checking for nulls.

```java
// Bad example
public class Student {

    private String name;

    public Student(Optional<String> name) { // Requires Optional every time an instance is created
        this.name = name.orElseThrow(NullPointerException::new); // Null checking and assignment are hidden inside Optional
    }
}

// Good example
public class Student {

    private String name;

    public Student(String name) {
        this.name = name; // Straightforward behavior
    }
}
```

## Not used in Collection

Elements in a collection usually do not need to contain null values unless you force them to, and many collection APIs already provide their own null-handling behavior. Since a collection contains many elements, wrapping each one in `Optional` also introduces avoidable overhead. For that reason, avoid using `Optional` as an element type whenever possible. As with fields and arguments, it also makes every add and read operation more cumbersome.

```java
// Bad example
List<Optional<String>> names = new ArrayList<>();
names.add(Optional.ofNullable(name1)); // Must wrap every element before adding it

// Good example
List<String> names = new ArrayList<>();
names.add(name1);
```

## Collection is Collection

If Collection is a method that returns a value of Null, it is often better to return an empty Collection using Collections.emptyList() or Collections.emptyMap(). Collection is

Also, if you are using Spring Data JPA, if the return value is Null, it will automatically generate an empty List, so there is no need to use Optional.

```java
// Bad example
public Optional<List<Student>> listStudent() {
    List<Student> students = this.repository.listStudent();
    return Optional.ofNullable(students);
}

// Good example
public List<Student> listStudent() {
    List<Student> students = this.repository.listStudent();
    return students != null ? students : Collections.emptyList();
}
```

## int/long/double is not wrapped in Optional

Variations of Optional also provide classes for some primitive types. This is the case with int/long/double. These are better wrapped in OptionalInt, OptionalLong, OptionalDouble.

```java
// Bad example
Optional<Integer> count = Optional.of(100);
int countNum = count.get();

// Good example
OptionalInt count = OptionalInt.of(100);
int countNum = count.getAsInt();
```

## lastly

Even if your current Java version is old, many companies have already moved to Java 1.8 or later because of official support and ecosystem changes. As a Java developer, it is worth becoming familiar with the important APIs introduced in Java 1.8. In that sense, `Function` and `Optional` are both APIs worth learning. Java became popular in part because it improved developer productivity, so it makes sense to learn the APIs that continue pushing it in that direction.

Java is already a mature language, but it has continued to evolve quickly in response to trends such as functional programming. We now live in an era full of languages that are both fast and pleasant to write, so it is reasonable to wonder how long Java can keep adapting. The JVM is still a powerful platform, but alternatives such as LLVM continue to grow. Even so, if you stay current with modern Java and learn its newer APIs, adapting to other languages will usually become easier as well.

[^1]: Refers to an object with multiple elements such as List, Set, Map, etc.
[^2]: Rather than InputStream or OutputStream used for file input/output, it is a Java 1.8 API that allows you to execute a specific method (mainly Lambda) while cycling through the elements of a Collection one by one.
[^3]: Since the return value is itself, the method can be written in a chain many times. The Builder pattern is a typical example of method chaining.
[^4]: Lazy in programming refers to a system in which a certain process is executed only when it is called, rather than always. Processing starts only when necessary, reducing unnecessary processing.
