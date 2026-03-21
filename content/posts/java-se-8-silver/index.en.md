---
title: "About Oracle JavaSE 8 Silver"
date: 2019-10-06
categories: 
  - java
image: "../../images/java.webp"
tags:
  - ocjp
  - certification
  - java
---

This time, I took the Oracle certified JavaSE 8 Silver exam. I was going to be using Java for a while at work and wanted to see what level my ability was at, so I took the test and luckily, I passed. If you search for Java Silver, there are many blogs about how those who passed the exam studied and how difficult it was, so I won't write about it here.

Still, the reason why I write about qualification exams as a post is that there is a disconnect between qualifications and the knowledge required for actual work. I think some people first obtain qualifications before starting work, while others, like me, gain experience in business etiquette before taking the exam. So, even if you are not in exactly the same situation as me, I think that the problems you are having trouble with on tests probably overlap to some extent. As a result, I think that what I am saying is not much different from what many other people have written...

If someone who is about to take the JavaSE Silver exam finds this post, I hope that the information will be of some use to them, so I would like to share my experience with what I found surprising.

## overall impression

There are many cases where knowledge that is not used in the job is required. However, I had the impression that it was not all useless knowledge. There were some problems that I thought that once I learned them, I could put them to use in my work. Broadly speaking, there are questions that require simple knowledge of APIs, and questions that require the ability to read and understand code.

What I found surprisingly difficult was the problem of not being able to depend on the IDE. Typical examples include how to write imports and finding compilation errors. I don't think you would write Java using a simple text editor without using an IDE in your actual work, but such questions are not uncommon in exams, so you should carefully read the code presented in the questions.

Now, I would like to share my thoughts on the individual items on the test.

## data type

I think the three most commonly used data types are int/double/String. In the test, there were no questions asking about the range of byte or float types, but these questions may appear in some questions. For example, the question of choosing which data types cannot be used in the conditional statement of a switch statement. If you take Bronze, you will be asked questions about data types in more detail, so if you are taking Silver after Bronze, it may be a little difficult if you are starting from Silver like me.

Another type of problem related to data types is type conversion (implicit versus non-implicit type conversion). This was also a problem where it was surprisingly difficult to know the correct answer unless you clearly memorized the characteristics of byte, short, long, and float.

## String/Stringbuilder

There were many problems with String and StringBuilder methods. However, this is something that I don't usually pay much attention to, but since String is an Immutable object, you need to remember that using methods such as replace and substring will return a new instance. For example, the following problems exist.

```java
// Problem of guessing the output string

String s = "this is string";
s.substring(0, 1);
System.out.println(s);
```

Since the original String itself has not changed, it is natural that it will be output as originally declared, but if you do not read the code carefully, it is easy to misunderstand that the characters cut out by substring are being output. (Maybe it's just me...)

## array

I didn't often use arrays, and mostly used Lists, so I had a hard time with this problem. Problems such as how to declare an array and how to process the elements in the array have often come up, but since there are various ways to declare an array, the choices seem a little dubious.

Of course, List is basically mutable and not thread-safe, so depending on the case, it may be necessary to introduce arrays in business. I think it's worth remembering, so I wanted to make sure to remember about arrays.

## Access modifiers/inheritance

There is a question that asks the difference between default and protected. I think it's important to keep in mind that even if you've always taken package structure and inheritance into consideration in your implementation, most of the time you're only using public and private, and it's easy to forget the others.

There was also an issue where access modifiers were important keywords in inheritance issues. For example, you might be presented with code that narrows down the scope of a method that is being overridden from a superclass, and asked what would happen if that code were executed. Even if you think that the process will run normally and select the result as the answer, the correct answer may be "Compile error".

I got the impression that there were quite a lot of problems regarding inheritance. There are many variations, such as the difference between an interface and an abstract, the method of inheritance, the difference between an instance and a reference, and casting, and although I am familiar with all of them because they are used in practice, I got the impression that the questions themselves were trying to lead to mistakes in a way that was a little more complex than just my knowledge. You need to read the code carefully.

## label

This was my first time encountering labels, so I didn't know anything about them. However, I think that using a double loop can significantly improve performance, so I think it's a good idea to keep in mind not only for testing. However, I think there may not be much use for it other than loops.

There was a small problem where I wrote an if statement without a labeled loop statement and asked what the result would be. For example:

```java
// Problem asking what the output will be

int num = 0;

x:
for (int i = 0; i < 10; i++) {
    if (i % 5 == 0) {
        continue x;
    }

    System.out.println(num);
    num++;
}
```

I think this question is more like asking whether you understand the difference between a conditional statement and continue/break than the label itself, but what is a label anyway? Once that happens, you will no longer be able to answer. Of course, there was also the question of what could be labeled as knowledge.

## exception

Issues arise regarding the characteristics of Java exceptions and their behavior in try-catch statements. You may create custom exceptions in your work, so if you understand the difference between Exception and RuntimeException, I don't think it will be that big of a problem.However, what you might miss in the code presented is things like ``Is there a throws declaration?'' and ``Does it happen if you declare a superclass exception with catch and then write a subclass again and get a compile error?'' Note that if you catch a higher-level exception, you don't need to write lower-level exceptions.

If you're in an environment where you can use an IDE, you don't need to worry too much about this, so I think it's surprisingly easy to overlook.

## Lambda

In JavaSE Silver, we often hear about APIs unique to Java8. Same goes for Lambda. However, most Lambda-related problems require knowledge about correct writing methods and types of functions. Even if you're not used to it, it's okay if you remember it.

As for how to write it, it was a question of whether the parentheses and return were omitted correctly, and like exceptions, it was easy to miss because it would immediately result in a compile error in an IDE.

## LocalDate / LocalTime / LocalDateTime

This was a difficult problem for me because the only date-related APIs I had used up until now were `java.util.Date` and `java.sql.Date`. This is also Immutable like String, so similar problems will occur. For example, the problem is as follows.

```java
// Problem asking what the output will be

LocalDate date = LocalDate.now();
date.plusDays(5);
System.out.println(date);
```

Just like with String, the only thing you need to remember about Local APIs is that the methods that manipulate the value return a new instance. However, there are problems with the output format of time and date, so I had to keep this in mind. Looking at the API, it is possible to convert to `java.sql.Data`, but there were no problems with conversion.

## others

When I looked at the question book, I found that I only selected one question even though it was a multiple-choice question, and I thought it would be a problem if this happened in the real test.However, in the actual test, single-choice questions are selected using radio buttons, and multiple-choice questions are selected using checkboxes, so I didn't have to worry about that. If only one check box was selected, a warning was displayed before moving on to the next question.

Although some loop processing may require direct calculation, most of the questions can be answered immediately without taking much time. I don't think you'll ever run out of time, so I think it's an easy qualification to pass if you remember the characteristics of the API you're being asked about.

However, on the other hand, I think there are many cases where people like me, who are used to work and forget or don't know the basics, get hooked. As mentioned earlier, you are unlikely to run out of time, so be sure to read the questions that provide code carefully. There are actually only three answers to the problems presented with code: ``The result of the process is output,'' ``A compilation error occurs,'' and ``An exception is thrown during execution,'' so I think it's a good idea to first think about which one applies to you.

So, I hope all of you who are taking the exam from now on will pass!
