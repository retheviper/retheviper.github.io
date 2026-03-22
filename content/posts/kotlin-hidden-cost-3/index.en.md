---
title: "Kotlin's hidden costs - Part 3"
date: 2021-11-28
translationKey: "posts/kotlin-hidden-cost-3"
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---

This is the final article on Kotlin's hidden costs. The previous articles have been quite interesting, but this time I'm going to touch on the unique features of Kotlin, so I think the content will be even more in-depth, as I need to include an understanding of Kotlin itself.

This time's agenda is "delegation properties" and "loop using range." This article summarizes the contents of [Exploring Kotlin’s hidden costs - Part 3](https://bladecoder.medium.com/exploring-kotlins-hidden-costs-part-3-3bf6e0dbf0a4).

## Delegate property

[Delegated properties](https://kotlinlang.org/docs/delegated-properties.html) are [properties](https://kotlinlang.org/docs/properties.html) whose `getter` and `setter` are implemented by a delegate object. This allows you to create reusable custom properties.

```kotlin
class Example {
    var p: String by Delegate()
}
```

The delegate object must implement `getValue()` and `setValue()` to set and read properties. These functions then require property metadata (property name) and an object instance as arguments.

When a class is defined as a delegated property, the compiler generates code similar to the following:

```java
public final class Example {
   @NotNull
   private final Delegate p$delegate = new Delegate();
   // $FF: generated field
   static final KProperty[] $$delegatedProperties = new KProperty[]{(KProperty)Reflection.mutableProperty1(new MutablePropertyReference1Impl(Reflection.getOrCreateKotlinClass(Example.class), "p", "getP()Ljava/lang/String;"))};

   @NotNull
   public final String getP() {
      return this.p$delegate.getValue(this, $$delegatedProperties[0]);
   }

   public final void setP(@NotNull String var1) {
      Intrinsics.checkParameterIsNotNull(var1, "<set-?>");
      this.p$delegate.setValue(this, $$delegatedProperties[0], var1);
   }
}
```

Some static property metadata is added to the class. And each time a value is set and read, initialization by the constructor occurs.

## delegate instance

In the above sample, a new delegate instance is created to implement the property. This is what happens when delegation is stateful. An example would be using locally computed properties.

```kotlin
class StringDelegate {
    private var cache: String? = null

    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        var result = cache
        if (result == null) {
            result = someOperation()
            cache = result
        }
        return result
    }
}
```

Additionally, if additional parameters are passed to the constructor, a new delegate instance is required.

```kotlin
class Example {
    private val nameView by BindViewDelegate<TextView>(R.id.name)
}
```

If it is stateless and you just want to keep the instance and property name of the object that has already been passed, there is a way to add `object` to the delegate class and make it a singleton. For example:

```kotlin
object FragmentDelegate {
    operator fun getValue(thisRef: Activity, property: KProperty<*>): Fragment? {
        return thisRef.fragmentManager.findFragmentByTag(property.name)
    }
}
```

You can also extend and delegate existing objects. This means that `getValue()` and `setValue()` can also be defined as extension functions. Kotlin already uses the pattern of delegating as extension functions to `Map` and `MutableMap`. (property name is used as key)

If you want to keep and reuse multiple properties in a local delegate instance within a class, initialize the instance in the class's constructor.

Since Kotlin 1.1, you can also use [local delegated properties inside functions](https://kotlinlang.org/docs/delegated-properties.html#local-delegated-properties). In this case, the delegate can be initialized later.

Each delegated property defined in a class incurs additional overhead and metadata, so it is better to reuse properties as much as possible. You should also consider whether delegated properties are a good option if you have many items to define.

## generic delegation

Delegated functions can also be defined generically. So you can also define delegate classes as properties of different types.

```kotlin
private var maxDelay: Long by SharedPreferencesDelegate<Long>()
```

However, when using generic delegation for primitives as shown above, you need to be aware that `boxing` and `unboxing` occur when specifying and reading values. This happens even if the property is non-null.

Therefore, when defining a non-null primitive delegated property, it is better to avoid defining it generically.

## Standard delegation (lazy())

In Kotlin, standard features for delegation exist such as [Delegates.notNull()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/not-null.html), [Delegates.observable()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/observable.html) and [lazy()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/lazy.html).

`lazy()` is a function for read-only delegated properties. You can specify a lambda to initialize the properties when the first load occurs.

```kotlin
private val dateFormat: DateFormat by lazy {
    SimpleDateFormat("dd-MM-yyyy", Locale.getDefault())
}
```

This is a good performance and readability way to defer expensive initialization until the value is actually loaded.

However, it is important to note that `lazy()` is not an inline function, the lambda passed as an argument is compiled as a separate `Function` class, and the returned delegate object is also not inlined.

What is easy to overlook with the `lazy()` function is that the delegation type of the return value can be determined by the argument `mode`.

```kotlin
public fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)
public fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
        when (mode) {
            LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
            LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
            LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
        }
```

If `mode` is not specified, the default is `LazyThreadSafetyMode.SYNCHRONIZED`, which does the expensive `double-checked lock` to ensure that initialization blocks are executed safely across multiple threads.

If you know that only a single thread will have access to a property, you may want to lower unnecessary locks. In this case, you can use `LazyThreadSafetyMode.NONE`.

```kotlin
val dateFormat: DateFormat by lazy(LazyThreadSafetyMode.NONE) {
    SimpleDateFormat("dd-MM-yyyy", Locale.getDefault())
}
```

## Ranges

You can define a limited range of values ​​in [Ranges](https://kotlinlang.org/docs/ranges.html). This value can be specified as anything that is `Comparable`. Using this expression, we can implement the interface [ClosedRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-closed-range/).

## inclusion test

You can use `in` and `!in` to detect whether a specific value is included in the range using range.

```kotlin
if (i in 1..10) {
    println(i)
}
```

Range is optimized for non-null primitive types (`Int`, `Long`, `Byte`, `Short`, `Float`, `Double`, `Char`), so the compiled result is as follows.

```java
if(1 <= i && i <= 10) {
   System.out.println(i);
}
```

So there is no overhead or additional object allocation. But what if it's not primitive?

```kotlin
if (name in "Alfred".."Alicia") {
    println(name)
}
```

Prior to Kotlin 1.1.50, `ClosedRange` objects were always generated at compile time. However, starting from 1.1.50 it will look like this:

```kotlin
if(name.compareTo("Alfred") >= 0) {
   if(name.compareTo("Alicia") <= 0) {
      System.out.println(name);
   }
}
```

range can also be used in `when` conditional expressions. The readability is better than `if-else`.

```kotlin
val message = when (statusCode) {
    in 200..299 -> "OK"
    in 300..399 -> "Find it somewhere else"
    else -> "Oops"
}
```

However, when using range, when checking whether a specific value is included, there is a cost if there is a gap between the specified range and the code that uses it. For example, suppose you have the following code.

```kotlin
private val myRange get() = 1..10

fun rangeTest(i: Int) {
    if (i in myRange) {
        println(i)
    }
}
```

In this case, a `IntRange` object will be added when you compile.

```java
private final IntRange getMyRange() {
   return new IntRange(1, 10);
}

public final void rangeTest(int i) {
   if(this.getMyRange().contains(i)) {
      System.out.println(i);
   }
}
```

This is the same even if you define the property getter as `inline`. Therefore, it is better to write it directly in the test where range is used to prevent objects from being added. Also, if you want to use a non-primitive object, you can define it as a constant and reuse the `ClosedRange` instance.

## for loop

It is also a good choice to use a range of primitive types excluding `Float` and `Double` in a loop.

```kotlin
for (i in 1..10) {
    println(i)
}
```

There is no overhead in the compiled result.

```java
int i = 1;
for(byte var2 = 11; i < var2; ++i) {
   System.out.println(i);
}
```

If you want to loop in reverse order, you can use [downTo()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/down-to.html).

```kotlin
for (i in 10 downTo 1) {
    println(i)
}
```

Again, there is no overhead involved.

```java
int i = 10;
byte var1 = 1;
while(true) {
   System.out.println(i);
   if(i == var1) {
      return;
   }
   --i;
}
```

It's also nice to use [until](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/until.html) to loop below a certain value.

```kotlin
for (i in 0 until size) {
    println(i)
}
```

Previously it would have cost a little more, but starting with Kotlin 1.1.4 it will generate code like this:

```java
int i = 0;
for(int var2 = size; i < var2; ++i) {
   System.out.println(i);
}
```

However, there are other cases where optimization is not very effective. Let's say we have an example that uses [reversed()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/reversed.html).

```kotlin
for (i in (1..10).reversed()) {
    println(i)
}
```

The compiled code is not very clean.

```kotlin
IntProgression var10000 = RangesKt.reversed((IntProgression)(new IntRange(1, 10)));
int i = var10000.getFirst();
int var3 = var10000.getLast();
int var4 = var10000.getStep();
if(var4 > 0) {
   if(i > var3) {
      return;
   }
} else if(i < var3) {
   return;
}

while(true) {
   System.out.println(i);
   if(i == var3) {
      return;
   }

   i += var4;
}
```

A `IntRange` object is generated to redefine the range, and a `IntProgression` object is generated to arrange the elements in reverse order.

If more than one function is used to create `progression`, it will incur the overhead of creating more than one object.

The above rules are the same when using [step()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/step.html), and the situation does not change even if `step 1` is specified.

```kotlin
for (i in 1..10 step 2) {
    println(i)
}
```

Additionally, when the generated code reads the last value, additional processing takes into account the last element of the `IntProgression` object and the range specified by `step()`. In the sample above, the last element is `9`.

Therefore, when doing a loop using `for`, it is better to avoid overhead by using `..`, `downTo()`, and `until()` as much as possible.

## forEach loop

The results do not change much if you use the inline extension function `forEach()` for range instead of the `for` loop.

```kotlin
(1..10).forEach {
    println(it)
}
```

However, `forEach()` is not optimized only for `Iterable`. This means we need to create an iterator. So when compiled it will look like this:

```java
Iterable $receiver$iv = (Iterable)(new IntRange(1, 10));
Iterator var1 = $receiver$iv.iterator();

while(var1.hasNext()) {
   int element$iv = ((IntIterator)var1).nextInt();
   System.out.println(element$iv);
}
```

This is more expensive than previous samples. Because not only are we generating `IntRange` objects, we are also generating `IntIterator` objects. If it's not primitive it will cost more.

Therefore, if you need a loop using range, it is better to use `for` loop rather than `forEach()` to reduce overhead.

## collection index loop

The Kotlin standard library provides an array and `Collection` intex in an extended property called [indices](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/indices.html).

```kotlin
val list = listOf("A", "B", "C")
for (i in list.indices) {
    println(list[i])
}
```

The compiled results of `indices` show good optimization.

```java
List list = CollectionsKt.listOf(new String[]{"A", "B", "C"});
int i = 0;
for(int var2 = ((Collection)list).size(); i < var2; ++i) {
   Object var3 = list.get(i);
   System.out.println(var3);
}
```

`IntRange` object is not created. So what happens if you try to implement it yourself?

```kotlin
inline val SparseArray<*>.indices: IntRange
    get() = 0 until size()

fun printValues(map: SparseArray<String>) {
    for (i in map.indices) {
        println(map.valueAt(i))
    }
}
```

If you define it as an extended property and compile it, you'll notice that you end up with less efficient code. A `IntRange` object is created.

```kotlin
public static final void printValues(@NotNull SparseArray map) {
   Intrinsics.checkParameterIsNotNull(map, "map");
   IntRange var10000 = RangesKt.until(0, map.size());
   int i = var10000.getFirst();
   int var2 = var10000.getLast();
   if(i <= var2) {
      while(true) {
         Object $receiver$iv = map.valueAt(i);
         System.out.println($receiver$iv);
         if(i == var2) {
            break;
         }
         ++i;
      }
   }
}
```

In this case it is better to use the `until()` and `for` loops instead.

```kotlin
fun printValues(map: SparseArray<String>) {
    for (i in 0 until map.size()) {
        println(map.valueAt(i))
    }
}
```

## lastly

How was it? Personally, I have never used delegated properties much, so I learned a lot from understanding them in the first place. Also, with regard to range, it was customary in Java to define it as a field in the test class and reuse it in various functions, but I was a little shocked because I had no idea that it would be more expensive.

I also realized that I needed to properly understand the functions and APIs provided by Kotlin. Also, as I mentioned in [Occam's razor](https://en.wikipedia.org/wiki/Occam%27s_razor), I thought it was necessary to pursue logic and code that are as simple as possible. You can always check the code that has been decompiled to Java with `Tools > Kotlin > Show Kotlin Bytecode` in the IntelliJ menu, so it may be better to optimize code while checking how it will be converted in the latest version.

This month's posts were mostly just translations, rather than the usual ones where I introduce my experiences and hypotheses, but I personally think I gained quite valuable knowledge. If I have another opportunity to come up with something good, I would definitely like to introduce it to you.

See you soon!
