---
title: "Use a functional interface"
date: 2019-08-06
translationKey: "posts/java-functional-interface"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - functional interface
  - lambda
  - java
---
This time, as always, it's the knowledge I gained through work. I created a certain Iterable class and needed to run a For loop. This wasn't that difficult. First, let the field contain a list. All we had to do was implement Iterable and create a method whose return value is an Iterator. However, the problem was ``creating a class that has a function that terminates the loop according to rules specified by the user'' in the middle of the loop of that class. In other words, in order to end a loop, there is a method with an argument that accepts the element being looped as an argument, and that method has data that serves as a reference for each instance.

## Use methods as fields?

Does it have data like a field while functioning as a method? Should the user be able to specify it further? It was a difficult order, so I hesitated for a moment, but I was advised to use function, so I looked into it. I see, if you use this, you can expect the functionality of a method while declaring it as a field. I would like to apply it right away and show you how it works first.

First, prepare an Iterable class. This is the target of the loop.

```java
// A class that holds Line objects as a list and can be iterated
public class Factory implements Iterable<Line> {

    private List<Line> lines = new ArrayList<>();

    public Iterator<Line> iterator() {
        return this.lines.iterator();
    }
}
```

The following is an example of using this Factory class to perform a loop. In the middle of the loop, a judgment is made at `isEnd(Line)` of the `Rule` class. If something in the Line instance meets the condition, the return value will be True and the loop will exit.

```java
// Iterable class
Factory factory = new Factory();

// Process Line objects inside Factory with a for loop
for (Line line : factory) {
    // Use the Rule class method that determines when the loop ends
    if (Rule.isEnd(line)) {
        break;
    }
    // ... some processing
}
```

In the case of the Rule class that performs the judgment here, it is as follows.

```java
public class Rule {

    // Holds the matching rule as a field
    private Predicate<Line> endRule = line -> line.isBroken();

    // Method that checks whether the Line argument matches the Predicate
    public boolean isEnd(Line line) {
        return endRule.test(line);
    }

    public class RuleBuilder {

        // The internals are just a normal builder

        private Predicate<Line> endRule;

        public RuleBuilder endRule(Predicate<Line> endRule) {
            this.endRule = endRule;
            return this;
        }
    }
}
```

First of all, what is a Predicate and can it be done with just Lambda? This is the code you might think. But it's working properly. Why? I didn't know much about it until now, but it was a magic created by `java.util.function` added in Java8. I thought that fields were used to hold data, but I didn't know that the data could also be used for methods.

Now, I would like to introduce what `java.util.function` is, including the various interfaces included in it.

## Functional Interface

It seems that the various interfaces included in `java.util.function` are called functional interfaces. Lambda, which was added in Java 8, is said to be an embodiment of an interface that has only one abstract method to implement, but the term "interface that has only one abstract method to implement" is a functional interface.It's difficult to express in words, but the point is one. An interface that can be filled with Lambda. There are various types, and each has slightly different characteristics, but it's actually not that difficult, just the one you choose depends on what you want it to do. If I were to say it's more difficult, I'd say Lambda...

Anyway, let's introduce these functional interfaces one by one.

## Function

Function is, as it sounds, a typical function. You declare it with arguments and a return value. To execute it, you call `apply()`. Looking at the code, it looks like this:

```java
// Example where Integer is the argument and String is the return value
Function<Integer, String> function = number -> String.valueof(number);

// Execute the Function
String result = function.apply(12);
```

### BiFunction

In addition to Function, there are several functional interfaces that are prefixed with "Bi". The difference is, as you can see, there are two arguments. Others are almost the same as the original.

```java
// Example where two Strings are arguments and Integer is the return value
BiFunction<String, String, Integer> biFunction = (string1, string2) -> Integer.parseInt(string1) + Integer.parseInt(string2);

// Execute the BiFunction
int result = biFunction.apply("1", "2");
```

## Predicate

This is what I introduced earlier. `Predicate` has the meaning of "predicate". As the name suggests, it is something like "state whether an argument is True or False". It takes one argument and returns a Boolean. The execution is `test`.

```java
// Example with a String argument
Predicate<String> predicate = string -> string.isEmpty();

// Execute the Predicate
boolean result = predicate.test("not empty");
```

### BiPredicate

A Predicate with two arguments.

```java
// Example with a String argument
BiPredicate<String, Integer> biPredicate = (string, number) -> string.equals(Integer.toString(number));

// Execute the BiPredicate
boolean result = biPredicate.test("1", 1);
```

## Consumer

`Consumer` does exactly what its name suggests: it consumes a value. It accepts arguments but has no return value (`void`). You execute it with `accept()`.

```java
// Example with a String argument
Consumer<String> consumer = string -> System.out.println(string);

// Execute the Consumer
consumer.accept("receive!");
```

### BiConsumer

A Consumer with two arguments.

```java
// Example with String and Integer arguments
BiConsumer<String, Integer> biConsumer = (string, number) -> System.out.println(string + "：" + number);

// Execute the BiConsumer
biConsumer.accept("The chance of profit this year is", 0);
```

## UnaryOperator

`Unary` means "unary". `Operate` has the meaning of acting. It gives the impression that the argument and return value are the same, and that the argument is manipulated in some way before being returned.

```java
UnaryOperator<String> uOperator = string -> string + " is completed";

// Execute the UnaryOperator
String result = uOperator.apply("If you input this text");
```

### BinaryOperator

Operator with two arguments.

```java
BinaryOperator<String> biOperator = (string1, string2) -> string1 + string2 + " is not";

// Execute the BinaryOperator
String result = biOperator.apply("I am", " fine");
```

## Supplier

`Supply` means "supply". It is the exact opposite of a Consumer, which has no arguments and only a return value. The execution will be `get`. Since there are no arguments, there is no interface like BiSupplier.

```java
Supplier<String> supplier = () -> "For example, a string is returned without arguments!";

// Execute the Supplier
String result = supplier.get();
```

## Finally

It's been several years since Java 8 was released, and Java has already been updated to version 12. However, there are still many situations in which Java 8 is used, and you want to write code that takes full advantage of Java 8's features as much as possible. Both Lambda and Stream are difficult, but I would like to be able to use them somewhere like Function and be able to do things that I couldn't do before.

I learned a lot this time as well. The world of Java is still wide and deep!
