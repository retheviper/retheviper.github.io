---
title: "Let's use Enums"
date: 2019-10-27
translationKey: "posts/java-enum"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - enum
  - java
---
Enums in Java are interesting. It's attractive because it's easy to read and can hold multiple values ​​at the same time. Therefore, it is useful when multiple classes need to handle common code values ​​or when linking to a DB. I am also actively using it in my current job. Also, since it functions as a complete class, you can not only use it for constants, but also write processes, so I think it can be used in a wide range of ways.

So this time, I would like to summarize my experience and research on how to use Java's Enums and their benefits.

## Easy to read

The fact that the code is easy to read means that it is advantageous for maintenance and repair. Personally, I think that the programming stages should proceed in the following order: ``Create something that works,'' → ``Separate common (duplicate) processing using methods and classes,'' → ``Make code that is easy to understand for others.'' And all the better if every step has been made from the initial design.

Let's say that a certain code value exists as an item in a table. Depending on the type of DB, the item may be `boolean`, but it may also be `char(1)`. In these cases, the code value may have not only two states, but also three or more states. As an example, let's assume that we have the state in Java as a String and record it in the DB as `char(1)`. In that case, you would need code like the following.

```java
// Example method that converts a String status to a DB code value
public String toCodeValue(String status) {

    if (status.equals("TRUE")) {
        return "0";
    } else {
        return "1";
    }
}
```

The code that uses this method looks like this:

```java
// Set the status on the item object
public void setStatusTrue(Item item) {

    item.setStatus("TRUE");
}
```

It is possible to hardcode `status` to a constant. Constants can also be held within regular classes. However, in such a case, you first need to know in which class the constant is defined. If you don't know if a constant has been defined, it will be difficult to modify it, and you may end up creating the same constant multiple times in different classes.

The disadvantage of this code is that you may not be able to check what the state is until you see the result of the process. What if `status` specified somewhere is a third string that is neither `TRUE` nor `FALSE`? In that case, the only option would be to increase the number of `if` branches.

Let's change this to code that uses Enum. The code should look like this:

```java
public enum StatusEnum {
    
    TRUE("0"),

    FALSE("1");

    private Integer codeValue;

    StringEnum(Integer codeValue) {
        this.codeValue = codeValue;
    }

    public Integer getCodeValue() {
        return codeValue;
    }
}
```

Just set each code value to the constant name and prepare the field, constructor, and getter. As you can see from the structure, such an Enum class can be easily created using Lombok annotations.

```java
@Getter
@AllArgsConstructor
public enum StatusEnum {
    
    TRUE("0"),

    FALSE("1");

    private Integer codeValue;
}
```

The code to actually use the created Enum is as follows.

```java
// Set the status on the item object
Item item = new Item();
item.setStatus(StatusEnum.TRUE.getCodeValue()); // Becomes "0"
```

By creating an Enum, the value you want to put in becomes clear. Also, since Enum is an independent class, it can be easier to manage by storing it in separate packages. Even if the code value increases later, you can just modify that Enum. It is also possible to declare an Enum itself as a field. In that case, it will look like this:

```java
// Example with an Enum field
@Data
public Item {

    private StatusEnum status;
}
```

If the field is an Enum, specifying the value is easier.

```java
Item item = new Item();
item.setStatus(StatusEnum.TRUE); // Saved as TRUE

// Code for extracting the code value
String itemCodeValue = item.getStatus().getCodeValue();
```

## Can have multiple values

Isn't it that easy to read? There is a possibility that you may think so. This may certainly be the case if there is only one value per constant. However, the good thing about Enums is that a constant can have multiple values.

For example, let's say you are using two or more DBs and the same items are managed in `char(1)` in one table and `boolean` in the other. Enum allows you to manage both as a single constant. Let's check what that means with the code below.

```java
@Getter
@AllArgsConstructor
public enum MultiValueEnum {
    
    Y("0", true),

    N("1", false);

    private Integer charValue;

    private boolean booleanValue;
}
```

I think it's easy to understand when you look at the code, but depending on which getter you use, the same constant returns a code value of a different data type. If you were to use this in actual code, it would look like this:

```java
// When the DB column is char(1)
Item item = new Item();
item.setStatus(MultiValueEnum.Y.getCharValue()); // "0"

// When the DB column is boolean
item.setStatus(MultiValueEnum.Y.getBooleanValue()); // true
```

The fact that multiple values can be specified means that arrays and Lists can also be used, right? Some people may think so. The conclusion is yes. I didn't know this from the beginning, but when I looked into it, it seems that if you use [Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html), you can have a List as a value, and you can also compare it with the value in that List. The code in this case would look like this:

```java
@Getter
@AllArgsConstructor
public enum ListValueEnum {
    
    Y(Arrays.asList("Good","Excellent")),

    N(Arrays.asList("Bad", "Unavailable")),

    NULL(null);

    private List<String> codeValueList;

    // Check whether the List contains the value
    public boolean hasCodeValue(String codeValue){
        return codeValueList.stream().anyMatch(code -> code.equals(codeValue));
    }

    // Return the matching constant if a value in the List matches ("Good" returns ListValueEnum.Y)
    public ListValueEnum findByValue(String codeValue) {
        return Arrays.stream(ListValueEnum.getCodeValueList())
        .filter(listValueEnum -> listValueEnum.hasCodeValue(codeValue))
        .findAny().orElse(NULL); // Return Null if nothing matches
    }
}
```

I've only used Stream in some parts, so I haven't thought about how to use it like this, but this is code that could be used somewhere.

Also, if the Enum used as a field has no particular value, you can use the following annotation.

```java
@Data
public class Item {

    // Using StringEnum.YES as-is
    @Enumerated(EnumType.STRING)
    private StringEnum codeValue1;
}
```

## Class-like usage

Since Enum is a class, there are of course ways to add processing to it. It may seem difficult to have a process, but just think of the process as a method. Please refer to [previous post](../java-functional-interface/) regarding Lambda for information on how to fieldize methods.

```java
@AllArgsConstructor
public enum CalculateEnum {

    // Specify the field with a lambda
    TYPE_1(num -> num),

    TYPE_2(num -> num + 10);

    private Function<Integer> calculate;

    // Pass a value and get the processed result back
    pulbic Integer calculate(Integer number) {
        return calculate.apply(number);
    }
}
```

The code for using this type of Enum is as follows.

```java
// Get the processed result
Integer type_1 = CalculateEnum.TYPE_1.calculate(10); // 10

Integer type_2 = CalculateEnum.TYPE_2.calculate(10); // 20
```

## FinallyIf common parts are duplicated, it becomes difficult to manage and understand their values, and unnecessary code increases. I wanted to tell you that this can be overcome with Enum.What do you think? However, I think it is a question that you should think carefully about whether it is necessary to make constants into Enums. In some cases, it may be better to manage it as a table. And if it is only used within a specific class, there is no need to create an Enum.

Still, if you understand and remember how to use the API in this way, I think there will be times when you can use it somewhere. At first, I thought of Lambda as just "difficult to read code," but after learning about Function, I started using it actively. I believe that being a true programmer requires not only knowledge, but also being able to utilize that knowledge in the right place. First of all, it may be difficult to make that decision, but I think there are some things you can see by having knowledge first. So, if I have new knowledge, I will continue to share it on this blog.
