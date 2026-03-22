---
title: "design pattern, builder"
date: 2019-07-14
translationKey: "posts/java-design-pattern-builder"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - design pattern
  - java
---
I once had a junior at university who got a job in Japan as a developer earlier than me, and I asked him what language or framework he should study. There were people around me who said I should learn C# and popular libraries like React and Node.js, but I wanted to hear the opinions of people who are using them in the field. He also said that you can change languages ​​and frameworks at any time once you have mastered your main language, and that the skills you really need are design patterns.

After that, I bought a book and saw some design patterns, but it was difficult for me to think about how to use those patterns on my own. When you write code by yourself, all you have to do is write code that you can understand, so you don't have the time to think that much. Also, since I tended to write code with only the end user in mind, I didn't have to write code that included too many patterns.

Now, I am involved in framework development in Java, and I have been instructed to write the code I purchased so that other people can use it, but I thought that I should just generate a Wrapper class based on DTO as usual, but after I tried implementing it, I realized that this was not possible. This is because there are dozens to hundreds of variable settings, and the arguments passed as arguments are not simply numbers or letters. If you were to use the code you created, you would probably think this is a bad idea.

When I was wondering how to improve it, the person who gave the instructions advised me to use the Builder pattern. This seems to be intuitive and can be used with minimal arguments. So I actually tried using it. And when I compared it to traditional DTO, I was impressed. I would like to explain what is different and why by comparing DTO and Builder patterns.

## Telescoping Constructor Pattern

A telescope is a telescope. What makes it telescope-like is that the shape of the constructor, which gradually extends, resembles the expansion and contraction of a telescope, hence the name. This is a pattern that is often used in Java Beans these days. When creating an object, you can adjust the number of variables that contain values ​​depending on the number of arguments. Just have multiple constructors with Method Overloading[^1].For example, let's say you want to embody the process of ordering coffee at a cafe using a Java class. There are various options such as cup size, hot or iced, whether to add syrup or cream. If you write this using the Telescoping Constructor Pattern, it will look like this:

```java
public class CoffeeOrder{

    private final String size; 
    private final boolean hot;
    private final boolean addCream; 
    private final boolean addSugar;
    private final boolean takeout;

    public CoffeeOrder(Stirng size, boolean hot){
        this(size, hot, false, false, false);
    }

    public CoffeeOrder(Stirng size, boolean hot, boolean addCream){
        this(size, hot, addCream, false, false);
    }

    public CoffeeOrder(Stirng size, boolean hot, boolean addCream, boolean addSugar){
        this(size, hot, addCream, addSugar, false);
    }

    public CoffeeOrder(Stirng size, boolean hot, boolean addCream, boolean addSugar, boolean takeout){
        this.size = size;
        this.hot = hot;
        this.addCream = addCream;
        this.addSugar = addSugar;
        this.takeout = takeout;
    }
}
```

Now we have one class that defines the order. Let's actually use it.

```java
public class Cafe{

    public static void main(String[] args){
        // Create an object for each order
        CoffeeOrder order_1 = new CoffeeOrder("tall", false);
        CoffeeOrder order_2 = new CoffeeOrder("grande", false, true);
        CoffeeOrder order_3 = new CoffeeOrder("venti", true, true, false);
        CoffeeOrder order_4 = new CoffeeOrder("tall", false, false, true, false);
    }
}
```

The bad thing about this pattern is that it is difficult to understand the meaning of the arguments when creating the object. Unless you actually look at the contents of the class, you won't understand the meaning of the consecutive `false` and `true`. For example, if you only want to include the size and syrup as arguments, you will need to create another constructor to match that. Another problem is that the more variables you have, the more constructors you need to provide. If the order options increase or decrease later, it's difficult to deal with that.

## Java Bean, DTO(VO)

When learning the concept of OOP[^2] in Java, the first thing I encountered was this `Java Bean`. This can also be considered a pattern. As you all know, this is a pattern for passing values ​​using `Getter` and `Setter`. Let's also create an order class.

```java
public class CoffeeOrder{

    private final String size; 
    private final boolean hot;
    private final boolean addCream; 
    private final boolean addSugar;
    private final boolean takeout;

    public CoffeeOrder(){}

    public setSize(String size){
        this.size = size;
    }

    public getSize(){
        return this.size;
    }

    public setHot(boolean hot){
        this.hot = hot;
    }

    public getHot(){
        return this.hot;
    }

    public setAddCream(boolean addCream){
        this.addCream = addCream;
    }

    public getAddCream(){
        return this.addCream;
    }

    public setAddSugar(boolean addSugar){
        this.addSugar = addSugar;
    }

    public getAddSugar(){
        return this.addSugar;
    }

    public setTakeout(boolean takeout){
        this.takeout = takeout;
    }

    public getTakeout(){
        return this.takeout;
    }
}
```

It may also include a pattern that receives arguments as a constructor, but the characteristics of a Java Bean are its getters and setters, so they are omitted here. Now, let's create an order with this as well.

```java
public class Cafe{

    public static void main(String[] args){
        // Set the order contents through setters
        CoffeeOrder order_1 = new CoffeeOrder();
        order_1.setSize("tall");
        order_1.setHot(false);

        CoffeeOrder order_4 = new CoffeeOrder();
        order_2.setSize("grande");
        order_2.setHot(false);
        order_2.setAddCream(false);
    }
}
```

Now you can set values for each individual item, and it will be clearer what kind of order you are placing by looking at each setter. Also, even if the number of variables increases, you only need to prepare getters and setters accordingly.

However, when completing one order, there is a problem that as the number of options increases, the code becomes untitled and long. Right now we're only using 5 fields, but what if there were 20 or 30 options? It takes a lot of time to write them down one by one. It was this part that I failed. So I will use Builder to solve this problem.

## Builder Pattern

```java
public class CoffeeOrder{

    private final String size; 
    private final boolean hot;
    private final boolean addCream; 
    private final boolean addSugar;
    private final boolean takeout;

    public CoffeeOrder(Stirng size, boolean hot, boolean addCream, boolean addSugar, boolean takeout){
        this.size = size;
        this.hot = hot;
        this.addCream = addCream;
        this.addSugar = addSugar;
        this.takeout = takeout;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder{

        private final String size; 
        private final boolean hot;
        private final boolean addCream; 
        private final boolean addSugar;
        private final boolean takeout;

        public Builder(){}

        public CoffeeOrder size(String size){
            this.size = size;
            return this;
        }

        public CoffeeOrder hot(boolean hot){
            this.hot = hot;
            return this;
        }

        public CoffeeOrder addCream(boolean addCream){
            this.addCream = addCream;
            return this;
        }

        public CoffeeOrder addSugar(boolean addSugar){
            this.addSugar = addSugar;
            return this;
        }

        public CoffeeOrder takeout(boolean takeout){
            this.takeout = takeout;
            return this;
        }

        public CoffeeOrder build() {
            return new CoffeeOrder(size, hot, addCream, addSugar, takeOut);
        }
    }
}
```

Inner Class is also included, and it looks like it's complicated, but when you actually use it, it's not. Let's take another look at what happens when we use a Builder class like this.

```java
public class Cafe{

    public static void main(String[] args){
        // Create the order and use the builder
        CoffeeOrder order_1 = new CoffeeOrder();
        order_1.Builder().size("tall").hot(false).build();

        CoffeeOrder order_2 = new CoffeeOrder().Builder()
                            .size("grande")
                            .hot(false)
                            .addCream(true)
                            .takeout(true)
                            .build();
    }
}
```

It is used in a similar way to Setter, allowing you to handle all complex options in just one go. You can also complete orders at the same time as they are generated. This is because the builder uses itself as the return value and can call methods consecutively. With this, you can handle any number of variables!

## Use LombokIt seems that the above pattern can be set using just annotations using [Lombok](https://projectlombok.org). For example, constructors can be `@NoArgsConstructor`, `@RequiredArgsConstructor`, and `@AllArgsConstructor`. For Java Beans, it seems that getters and setters can be created by adding `@Data`. Also, for Builder, it says it can be done with `@Builder`. Below is an example using Lombok.

```java
import lombok.Builder;

@Builder
public class CoffeeOrder{

    private final String size; 
    private final boolean hot;
    private final boolean addCream; 
    private final boolean addSugar;
    private final boolean takeout;
}
```

By the way, if you set it to `@Builder(toBuilder = true)`, you will be able to directly access the Builder from `CoffeOrder.builder()` when creating a new instance. Also, when inheriting the value of an existing instance, you can use `order_1.toBuilder()`. In reality, the code will look like this:

```java
import lombok.Builder;

@Builder(toBuilder = true)
public class CoffeeOrder{

    private final String size; 
    private final boolean hot;
    private final boolean addCream; 
    private final boolean addSugar;
    private final boolean takeout;

    // ... basic builder

    public CoffeeOrderBuilder toBuilder() {
        return new CoffeeOrderBuilder().size(this.size).hot(this.hot).addCream(this.addCream)
                                       .addSugar(this.addSugar).takeout(this.takeout);
    }
}
```

When using Builder, if you want to inherit the fields of the parent class as is, you can add `@Builder.Default` to the field and it will be inherited as is. It has many other useful features, such as being able to specify access levels by attaching it to fields, so I definitely want to use it.

## Finally

It is important to understand the types of design patterns and their structures, but I think the most important thing is to use the right person in the right place. Even if I had known about the Builder pattern from the beginning, I don't think I would have tried to use it if I hadn't been told to use it. So from now on, I would like to study the design patterns themselves and consider when they can be used.

See you soon!

[^1]: A notation for creating multiple methods with the same name by changing the number and types of arguments.
[^2]: Object Oriented Programming.It is also called object-oriented programming. Rather than treating code as a process that flows from top to bottom (procedural programming), it is a programming paradigm that is established as data exchange between isolated objects.
