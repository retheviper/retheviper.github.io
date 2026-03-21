---
title: "Using Reflection and Generics"
date: 2019-07-30
categories: 
  - java
image: "../../images/java.webp"
tags:
  - reflection
  - generic
  - java
---

What I learned through this job is how to implement my code when someone else uses it as a library. It was a very new experience for me, as I had never been involved only with writing scripts that perform certain functions, or with the UI used by end users and the logic that processes its content. Up until now, all I had to do was implement the functions I was responsible for and optimize them. However, libraries are basically used by people who can handle code, so the design is completely different.

And one of the difficult parts was being flexible. For example, while receiving and processing data, up until now the first prerequisite was that the data was mapped to the beans that I had implemented. In my current job, I had to be able to respond to that because I didn't know what kind of beans were going to come in.

First, you need to know how to receive classes and instances as arguments. At first I thought of using Object itself, but when I looked into it, I found something called `Generic`, so I decided to use that.

Next is the bean received as an argument using that Generic. If you only use beans that you designed, you need to know the data types of the fields that the beans have, and you can exchange data using Getters/Setters. However, if you didn't create it yourself, you won't know how to call the field data type or getter/setter methods. I was wondering how to deal with this, and the answer I found was `Reflection`.

I think this post is about how I used those two methods to "handle beans that I didn't create."

## Generic

Generic means to "generalize the type of data." It is also called a generic type. It is said to be used when setting the type used within a class from outside the class. Until now, I have mainly used `Object` when I wanted to process multiple data types and objects. In Java, when looking at some primitives (reference types), most of the data is treated as objects, so it cannot be changed to the highest type, Object.

However, if you use `Object` as an argument, you cannot know what fields exist inside the method. You also cannot simply access values through getters and setters. So instead of treating it as an opaque object, you need to inspect the instance directly.

If you use Generic here, it has the same functionality as Object (any instance or class can be received), but the concrete type is determined first, so there is no need to cast. If this is the case, you can expect better performance and safer behavior. Another benefit is that you can use Reflection to look inside a class or instance and retrieve methods and fields for objects whose structure you don't know.

First, let's look at how to receive an instance as an argument. Specify `T`[^1] as an argument. Now it can accept arguments of type Generic. In other words, the instance itself is the argument. However, if the argument is `T`, you need to declare `<T>` even before the return value of the method. Expressed in code, it looks like this:

```java
// Method that receives a bean instance
public <T> boolean isBean (T parameter) {
    // ...some processing
}

// Example
BeanObject beanObject = new BeanObject();

if (isBean(beanObject)) {
    // ...some processing
}
```

Generics can also be used inside Lists. For example, if you write the following, any type will be accepted.

```java
List<T> list = new ArrayList<>();
```

To accept the class itself in Generic instead of the instance, write as follows.

```java
// Method that receives a bean class
public boolean isBean (Class<?> parameter) {
    // ...some processing
}

// Bounded inheritance is also possible
public boolean isStringBean (Class<? extends String> parameter) {
    // ...some processing
}

// Example
if (isBean(BeanObject.class)) {
    // ...some processing
}
```

This completes the preparation for passing Generic arguments. Next, let's look into Reflection, which handles the arguments received in this way.

## Reflection

Reflection is an API that lets you access methods, constructors, fields, and other members without knowing the concrete class type in advance. Once you obtain them, you can use them much like normal class members: call methods, read return values, access fields, and inspect attached annotations.

Now, I will show you how to actually use Reflection with the code below. First, let's look at how to get a class from an instance, and then the code to get a constructor from the obtained class.

```java
public <T> boolean isBean (T object) {

    // Get the class from the instance
    Class<?> objectClass = object.getClass();
    // Create another instance from the class
    Object instance = objectClass.newInstance();

    // Get the package name of the class
    String package = objectClass.getPackage();
    // Get the class name including its package
    Stirng classNamePackageInvolved = objectClass.getName();
    // Get only the simple class name
    String className = objectClass.getSimpleName();   
    
    // Get public constructors as an array
    Constructor[] constructors = objectClass.getConstructors();
    // Get a specific public constructor
    Constructor constructor = objectClass.getConstructor(parameter1, parameter2, ...);
    // Create an instance from the retrieved constructor
    Object instance2 = constructor.newInstance();

}
```

You can manipulate the class itself, so if you know its contents, you can create and use new instances. Next, I'll show you how to get the fields.

## Get Field

```java
public boolean isBean (T object) {

    Class<?> objectClass = object.getClass();

    // Get public fields as an array
    Field[] fields = objectClass.getFields();
    // Get a specific public field
    Field field = objectClass.getField("fieldName");
    // Get all fields as an array (including non-public ones)
    Field[] declaredFields = objectClass.getDeclardFields();
    // Get a specific field (including non-public ones)
    Field declaredFiled = objectClass.getDeclaredField("fieldName");

    // What you can do with Field
    // Set a value on the field
    field.set(object, parameter);
    // Get the field value
    String fieldValue = field.get(object);
    // Get annotations on the field
    Annotation[] annotations = field.getAnnotations();
}
```

The important thing to note here is that when you retrieve a field, you use it from the class, but when you assign or retrieve the actual value, you use the instance as the target. This is the moment when it becomes clear that a class is a blueprint, and instances are generated using that blueprint. This may also be an advantage of using Reflection.

Next, let's look at the method.

## Get Method

The methods are not that different from those for fields. You can get methods from a class and do more with them. I have also provided the code below.

```java
public boolean isBean (T objectClass) {

    Class<?> objectClass = object.getClass();

    // Get public methods as an array
    Method[] methods = objectClass.getMethods();
    // Get a specific public method
    Method method = objectClass.getMethod("methodName", parameter1, parameter2, ...);
    // Get methods as an array (including non-public ones)
    Method[] declaredMethods = objectClass.getDeclaredMethods();
    // Get a specific method (including non-public ones)
    Method declaredMethod = objectClass.getDeclaredMethod("methodName", parameter1, parameter2, ...);

    // What you can do with Method
    // Get the method name
    String methodName = method.getName();
    // Invoke the method
    Object methodInvoked = method.invoke();
    // Get annotations on the parameters
    Annotation[] parameterAnnotations = method.getParameterAnnotations();
    // Get the parameters
    Parameter[] parameters = method.getParameters();
}
```

Annotations can be retrieved from classes and fields, but methods are unique in that you can also retrieve annotations for their arguments. You can also get the argument itself. Of course, the `Parameter` class can also perform various operations such as obtaining argument names.

## conclusion

In the end, once the path forward became clear, the solution was simple. I could read fields from the incoming instance and assign the values to `Object` without dealing with methods or constructors. I finished the task with a straightforward structure: branch with `instanceof`, cast as needed, and process the data. In simplified form, it looks like this:

```java
// Method that accepts a bean whose class is unknown
public <T> processSomething(T bean) {
    // Objects of multiple types
    String stringObject;
    Integer intObject;

    // Get the class and its fields
    Class<?> beanClass = bean.getClass();
    Field[] beanFields = beanClass.getDeclardFields();

    // Loop through fields and store values in matching typed objects
    for (Field field : beanFields) {
        // Make the field accessible in case it is private
        if (field.canAccess(bean)) {
            field.setAccessible(true);
        }

        // Get the field value and inspect its type
        Object value = field.get(bean);
        if (value instanceof String) {
            stringObject = (String)value;
        } else if (value instanceof Integer) {
            intObject = (int)value();
        }
    }
}
```

What do you think? It's easy to do if you know how. It could also be applied further to modify and return only part of the value of an instance. There are many aspects that can be used in many ways.

## lastly

Once you know the fields and methods like this, I feel like you can handle any instance or class that comes your way. I won't know for sure until I compile and run it, but I feel like I've gotten smarter. We hope that you will deepen your understanding of classes and instances through Reflection.

See you soon!

[^1]: Type. However, it seems that E (Element), K (Key), N (Number), and V (Value) can also be used.
