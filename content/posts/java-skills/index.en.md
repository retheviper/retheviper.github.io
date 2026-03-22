---
title: "Various Java Coding Techniques"
date: 2019-11-17
translationKey: "posts/java-skills"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - java
---

In this post, I am collecting a few coding techniques that are not exactly advanced tricks, but that I found useful or just neat while writing Java code.

## Converting a List with Stream

Suppose we have the following two classes. To keep the code short, let us assume Lombok is being used.

```java
@Getter
public class Item {

    private String value;
}

@Setter
public class Product {

    private String value;
}
```

For business purposes, let us assume that the `value` field in these two classes is the same. In that case, there may be situations where we need to read `value` from `Item` and copy it into `Product`. If there is only one object involved, this is not a big deal.

```java
public Product setProductValueFromItemValue(Item item) {

    Product product = new Product();
    product.setValue(item.getValue());
    return product;
}
```

If multiple fields need to be mapped, we could also use a library such as [ModelMapper](http://modelmapper.org/). It is convenient because it automatically maps matching Getter and Setter names.

```java
public Product setProductValueFromItemValue(Item item) {

    ModelMapper mapper = new ModelMapper();
    Product product = mapper.map(item, Product.class);
    return product;
}
```

But what if these objects are inside a `List` or `Map`? Usually you would map them one by one inside a `for` loop.

```java
public List<Product> itemListToProductList(List<Item> itemList) {

    List<Product> productList = new ArrayList<>();
    for (Item item : itemList) {
        productList.add(mapper.map(item, Product.class));
    }
    return productList;
}
```

Using Stream and Lambda lets us write this more simply.

```java
public List<Product> itemListToProductList(List<Item> itemList) {

    List<Product> productList = itemList.stream().map(item -> mapper.map(item, Product.class)).collect(Collectors.toList());
    return productList;
}
```

What it does is not very different from a `for` loop. It takes elements from the original list one by one, maps them into new objects, and collects them into a new `List`. The difference is that `map()` takes a Lambda, so it can handle not only simple mapping but also more complex processing. Even when doing the same thing, the code becomes shorter and easier to read.

## Making Collections Immutable

I talked about immutable classes in [a previous post](../java-thoughts-of-immutable/). This time, I want to look at how to make `List` and `Map` data inside a class immutable using `Collection`. The following code is an example for `List`.

```java
public List<Item> returnAsUnmodifiableList(List<Item> list) {

    return Collentions.unmodifiableList(list);
}
```

The same idea applies to `Map` if you wrap it with `Collentions.unmodifiableMap()`. These converted `List` and `Map` instances cannot be modified, so they are useful for configuration-like data. Just be careful: if `null` is passed in, a `NullPointerException` will occur. If the `List` you want to wrap may be `null`, you can substitute `Collection.emptyList()` instead.

If you want to modify an immutable `List` or `Map`, copy it into a new object.

```java
public List<Item> returnAsModifiableList(List<Item> list) {

    return new ArrayList<>(list);
}
```

That said, if you copy an object this way and then modify the data, you need to be careful because the original immutable `List` may still be affected depending on what was copied.

## Making a Custom Class Iterable

Suppose a certain class contains child elements as a `List`. For example, something like this.

```java
public class Container {

    private List<Baggage> baggages = new ArrayList<>();
}
```

Depending on the situation, you may want to take out all child elements and write a `for` loop over them. Usually that would look like this.

```java
public void printBaggageNames(Container container) {

    List<Baggage> baggages = container.getBaggages();
    for (Baggage baggage : baggages) {
        System.out.println(baggage.getName());
    }
}
```

But if you could use the class itself inside an enhanced `for` loop, it would be even more convenient. In other words, you would want to use it like this.

```java
public void printBaggageNames(Container container) {

    for (Baggage baggage : container) {
        System.out.println(baggage.getName());
    }
}
```

If that were possible, we would no longer need a Getter, and the code would become much simpler. Also, because we are not exposing the `List` itself, we no longer need to worry about making it immutable.

This can be implemented with `Iterable`.

```java
public class Container implements Iterable<Baggage> {

    private final List<Baggage> baggages = new ArrayList<>();

    @Override
    public Iterator<Baggage> iterator() {
        return baggages.iterator();
    }
}
```

With this, child elements can be used directly in an enhanced `for` loop from the parent class. Simple.

## Closing Thoughts

These were not exactly ultra-advanced tricks, but I collected a few techniques that seem likely to be useful somewhere if you remember them. They are things I actually use in real work, and I think they are the sort of details that help when you want to go beyond "it just needs to work." Small differences in these kinds of techniques add up to overall programming skill, don't they? Thinking that way, I want to keep writing posts like this whenever I learn something new.

See you next time!
