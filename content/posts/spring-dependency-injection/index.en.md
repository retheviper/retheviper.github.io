---
title: "Spring DI should be done in the constructor"
date: 2019-12-09
translationKey: "posts/spring-dependency-injection"
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - dependency injection
  - java
  - spring
---
Speaking of Spring's representative features, there are many, but if I were to pick one, it would have to be DI[^1] by `@Autowired`. When I first encountered Spring, it seemed indescribably strange that objects could work without `new`. Once I learned that this is a design pattern, I thought it was even more amazing. After all, in order to write good code, it is necessary to ingenuity in various areas.

Anyway, DI is important and convenient, but I noticed something while using Spring Boot recently. Until now, `@Autowired` was declared in the field as usual, but in this case, it was sometimes attached to the constructor. Why do we attach some to fields and some to constructors? That's what I thought. In the end, I ended up adding all the annotations to the constructor, but I didn't really understand the difference between adding them to fields, so I did a little research.

In conclusion, it seems that in most cases it is better to attach `@Autowired` to the constructor rather than the field. And this (appending `@Autowired` to fields and constructors) is called "field injection" and "constructor injection" respectively. Let's explain these using code.

## Field Injection

First, let's assume that we have defined the following configuration class [^2] for injection.

```java
@Configuration
public class Mapper extends ModelMapper {

    @Bean
    public Mapper mapper {
        return new ModelMapper();
    }
}
```

Here we will use [ModelMapper](http://modelmapper.org/). As I wrote about ModelMapper in a previous post, it is a useful library that automatically maps beans with matching Getters/Setters.

Below is an example of a service class that registers a bean with Spring and adds the Autowired annotation to the field. This is called field injection.

```java
@Service
public class ItemServiceImpl implements ItemService {

    @Autowired
    private Mapper mapper;
}
```

When I first learned about Spring Framework, field injections like this were common. However, there is a fatal problem with field injection. This means that the program will work even if the field is Null. If the program runs even though the requirements necessary for the class to run are not in place, this is a bug. So field injection is not good.## Setter Injection

Actually, injection is also possible through Setter. Although it is not a very common method, this method is called setter injection, and expressed in code as follows.

```java
@Service
public class ItemServiceImpl implements ItemService {

    private Mapper mapper;

    @Autowired
    public void setMapper(Mapper mapper) {
        this.mapper = mapper;
    }
}
```

The problem with injection with setters is the same as with field injection. The program may run regardless of whether or not the required object is injected with the setter. Constructor injection, introduced next, can solve this problem.

## Constructor Injection

Injection using a constructor is called constructor injection, and the code looks like this:

```java
@Service
public class ItemServiceImpl implements ItemService {

    private Mapper mapper;

    @Autowired
    public ItemServiceImpl(Mapper mapper) {
        this.mapper = mapper;
    }
}
```

The good thing about constructor injection is that it blocks the possibility of problems like the ones mentioned above. This is more about the Java language specifications than Spring, but instances cannot be created for classes whose constructor does not meet the argument requirements. And the NullPointerException will no longer occur unless you inject Null.

Constructor injection also has the advantage of being able to prevent circular reference [^3] problems. With field injection and setter injection, problems cannot be discovered until the actual code is called, but if a circular reference occurs in constructor injection, a warning will be output when starting the Spring application.

Another problem with field injection is that you cannot unit test the class. There is no way to inject objects into fields that have the Autowired annotation. Once you use a setter, you will be able to inject it, but there is no reason to use a setter.

Another reason constructor injection is good is that you can declare fields final. Adding final to a field prevents the object from being modified within the class, making it safer.

## Finally

Until now, I used to use field injection as a matter of course, but after learning about the problems with field injection, I try to write code using constructor injection as much as possible. Even if you don't do it, IntelliJ always warns you to use constructor injection, and Spring's official documentation also mentions this. This will force us to change our previous perceptions.Speaking not only of Spring but also of Java coding, it is a Java specification that basically a constructor with no arguments is generated implicitly even if you do not write it. For Singleton classes and classes that would cause problems if they run without initializing arguments, constructors are intentionally written to prevent this. Therefore, it is important to always write the constructor explicitly. If you think about this, you will understand the importance of writing a constructor. After all, writing good code requires ingenuity in various areas!

[^1]: Dependency Injection. I won't go into depth as there are many detailed explanations on the internet, but to briefly explain the concept, it means creating an object externally and putting it in the code to make the object's dependencies independent from the code. Injected objects are code-independent, so they function the same no matter where they are called.
[^2]: If you add the Configuration annotation, it will be automatically recognized as a configuration class within Spring. If you define the object as a bean here, you will be able to do DI. Previously, there were cases where it was entered in an xml file.
[^3]: A circular reference means that multiple objects refer to each other. For example, if you create an instance of class A by referring to class B, and class B also references class A, you will end up in an infinite loop of repeating references to each other when creating an instance of either. The end of this infinite loop is StackOverflow.
