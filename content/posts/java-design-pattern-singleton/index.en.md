---
title: "Design pattern, Singleton"
date: 2019-08-19
categories: 
  - java
image: "../../images/java.webp"
tags:
  - design pattern
  - java
---
I remember that memory was always a problem when using a PC. The first PC I ever used was one that my father used for work, and back then it used DOS as its operating system, so I had to change the memory settings whenever I wanted to play games. At that time, I didn't think it was an inconvenience, I just thought it would be nice to be able to play games.

However, as time passed and I was preparing for a presentation at a university, I realized that multitasking is difficult if you don't have enough memory. Nowadays, the SSD is the part of a PC that gives you the greatest performance improvement when you upgrade it, but I think that's because we're now in an era where you can stably secure CPU and memory. First of all, there was a time when you would think that if you didn't have enough memory, it would just be slow.

Then, when I became a programmer, the memory problem became a more realistic problem. For example, if you build a certain system and have multiple users use it, there is a possibility that you will run out of memory, which is a limited resource, even today, even though hardware is rapidly developing. Every time you create an object, the remaining memory continues to decrease.

Therefore, in terms of optimization, in order to save memory, we should suppress the creation of unnecessary objects. I was wondering if there was a way to do this, and it already exists. This is the Singleton pattern, which is the subject of this post.

## What is the Singleton pattern?

The Singleton pattern is a design pattern for creating classes that are instantiated only once within an application and used until the application terminates. In the case of beans, many instances are generated and used, each with different data, but since there is only one instance here, there is no dynamic field. Therefore, when you have immutable data that can be accessed from anywhere, or when you need to repeat a specific process, you may create a class using the Singleton pattern.

The good thing about having a class like this is the memory problem mentioned earlier. For example, global variables can be accessed from any class, so they may not look much different from Singleton. However, in the case of global variables, they are always in memory whether they are used or not, potentially causing them to be wasted. However, in the case of Singleton, you can generate it if you need it, and not if you don't. So you can save memory.At work, I mainly created Singleton classes as utility classes. When data needs to be processed repeatedly, creating an instance each time causes memory issues and tends to make the code unnecessary and redundant. By storing this data in different instances of beans and leaving the processing to the Singleton class, we were able to reduce the amount of code and save memory.

## Classic Singleton pattern

Now, I will show you how to create a Singleton class. There are various design patterns, and one of them, Singleton, can be realized in various ways. First, we will introduce the classic method.

The purpose here is to have only one instance, so even though it is possible to access an already generated instance from the outside, it is not possible to create an instance without permission. To do this, we need to restrict access to the constructor. Let's start with the code.

```java
// Make the class public so it can be accessed from outside
public class SingletonClass {

    // Make the constructor private so it cannot be accessed from outside
    private SingletonClass() {}
}
```

But this alone is not enough. You need to create an instance somewhere. Also, as mentioned earlier, the point of instantiation must be externally controllable. Therefore, we need to provide a method that can access the private constructor.

```java
public class SingletonClass {

    // Static field used to store the instance
    private static SingletonClass uniqueInstance;

    private SingletonClass() {}

    // Return the instance (create it first if it does not exist yet)
    public static SingletonClass getInstance() {
        if (uniqueInstance == null) {
            uniqueInstance = new SingletoneClass();
        }
        return uniqueInstance;
    }

    public void doSomething() {
        // ... ordinary methods
    }
}
```

First, declare a static field where you can store your instance. This will be used to obtain an instance of the Singleton class from the outside. The reason why it is only declared and no instance is created at this stage is to distinguish it from global variables.

Next, create a static method that can be accessed even if no instance has been created. From here we will get an instance of this Singleton class. Inside the method, set the instance field as the return value, and only do new if the instance has not been created.

Now you can use it from the outside like this:

```java
// Get the instance
SingletonClass singletonInstance = SingletonClass.getInstance();

// Use a method on the instance
singletonInstance.doSomething();
```

Now we have a Singleton class that can be used with the same instance from anywhere.

## Problems with the classic Singleton pattern

If you don't need to think about multithreading, you don't have to worry about it, but that's not the case with modern programming. Especially when creating a system and providing it as a service, the same class may be requested by multiple users.

If the class is complex and it takes time to generate instances, or if instances are requested at almost the same time, the classic Singleton pattern may not be able to prevent multiple instances from being generated. In this case, it may not work as originally designed and an unexpected exception may occur.

Of course, several methods have been proposed to solve these problems, but these solutions also have disadvantages. First, let's take a look at the methods available and the disadvantages of each.

## To open the multithreading problemThere are probably other methods, but the following are ways to generate a thread-safe Singleton class.

1. Synchronize instance generation
2. Use Double-Checked Locking
3. Leave it to the JVM class loader

First, there is a simple way to synchronize instance generation. The solution is to add `synchronized` to the `getInstance()` method to get the instance. There is not much difference when looking at the code, so if you have a class with the classic Singleton pattern, this is the easiest method to apply.

```java
public class SingletonClass {

    private static SingletonClass uniqueInstance;

    private SingletonClass() {}

    // Synchronize the method that provides the instance
    public static synchronized SingletonClass getInstance() {
        if (uniqueInstance == null) {
            uniqueInstance = new SingletoneClass();
        }
        return uniqueInstance;
    }
}
```

However, the problem with `synchronized` is performance. It seems that the processing speed can be more than 100 times slower, so if you are using multithreading for performance reasons, it is not very desirable.

The next method is to double check. If the instance is null, synchronize. With this method, there is no need to synchronize every time (since the instance is not null from the second time onwards), so performance will not decrease except for the first time.

```java
public class SingletonClass {

    // Use volatile to keep the behavior stable
    private volatile static SingletonClass uniqueInstance;

    private SingletonClass() {}

    // Double-check the instance
    public static SingletonClass getInstance() {
        if (uniqueInstance == null) {
            synchronized (SingletonClass.class){
                if (uniqueInstance == null) {
                    uniqueInstance = new SingletoneClass();
                }
            }
        }
        return uniqueInstance;
    }
}
```

The reason for using volatile declaration is to prevent variables from entering the CPU's cache memory. Program data is first read from the hard disk and placed in the system memory, but then when processing is performed by the CPU, it may be further placed in the CPU's cache memory.

Recently, there are many systems equipped with multiple CPUs, so if an instance is stored in the cache memory of a different CPU, you will not know whether an instance has been generated or not. By placing a field in system memory with a volatile declaration, instance creation is performed more stably. However, there is still the problem of having to experience performance degradation due to synchronization at least once.

The last method is to create an instance when the JVM starts. With this method, instances cannot be reliably controlled from the outside, and instances are always generated, so multithreading problems can be avoided.

```java
public class SingletonClass {

    // Declare instance creation at the field level
    private static SingletonClass uniqueInstance = new SingletonClass();

    private SingletonClass() {}

    // No instance check is needed anymore
    public static SingletonClass getInstance() {
        return uniqueInstance;
    }
}
```

Since the JVM instantiates the class when it is loaded, static fields cannot be accessed by any thread. However, this is not much different from declaring it as a global variable, so the problem remains that the instance remains created in memory even if it is not used. The difference, of course, is that unlike global variable declarations, instances are unique. With global variables, even if you declare a field static, you can declare it again in a different class. In the first place, it is often difficult to know what is contained in global variables, so it is better not to use them too often.

## Is it okay to declare all methods and fields static?

Of course there is another way. However, this method is only effective if the initialization process is extremely simple (for example, there are no fields). This is possible if the class itself has a simple structure and the methods simply process and return data input from the outside. It's not a method that can't actually be used, but it's no longer a good method when you consider future functional expansion.

## Finally

The Singleton pattern is widely used and certainly an attractive way to design classes. However, there are issues that must be devised to avoid multi-threading issues, and since it is the only instance, there are also aspects that must be taken into account when handling fields. If an instance is being used in one thread and you put data in a field and then try to access it in another thread, problems may occur.

The Singleton class also has problems considering the OOP principle that "one class has only one responsibility." Even though you are in charge of processing something, you have the dual responsibility of managing the instance yourself. And since the constructor is private, there is a problem that subclasses cannot be created. If you change the constructor to public or protected to create a subclass, you may have a dilemma that it will no longer be a Singleton.

Still, classes created using the Singleton pattern have a certain appeal. As long as you manage your instances properly, you can call it up and use it anywhere. If you have a class that needs to be always in memory, this is a pattern you might want to consider.
