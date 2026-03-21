---
title: "A Late Look at String Operations"
date: 2021-02-08
categories: 
  - java
image: "../../images/java.webp"
tags:
  - java
  - string
---

This is already the third entry in my "late to the topic" series. That series has returned again, this time with another benchmark in tow. The theme this time is string manipulation in Java, and in particular I want to talk about concatenation and splitting. What initially got me interested was the fact that splitting strings seems to be handled only by `String.split()`, while concatenation has several choices such as `String.join()` and `Collectors.joining()`. When the same thing can be done through multiple APIs, sometimes it is just [syntactic sugar](https://en.wikipedia.org/wiki/Syntactic_sugar), but sometimes the implementation is completely different. That is especially common in languages like Java that have been used for a long time.

Even when something is close to simple syntactic sugar, you sometimes still need to choose carefully based on the surrounding code, readability, and similar concerns. For example, the `transferTo()` method on `InputStream` that I introduced earlier falls into that category. So when using an API, it is often worth checking how it is actually implemented if you want to write good code.

With that said, let us get into the main topic. First, concatenation.

## Concatenating

String concatenation comes in many forms. There are APIs such as `String.concat()` and `String.format()`, and it would be hard to assume and verify every possible scenario. So this time I want to focus on one specific case: joining an array or `Collection` of strings into a single string using a delimiter.

In Java, the main options for delimiter-based string concatenation are the following. After comparing the characteristics and actual usage of each API, I will also run a benchmark as usual to measure performance. (I am excluding the simple `+` case, since I do not think it is a good option.)

- `String.join()`
- `StringJoiner`
- `StringBuffer`
- `StringBuilder`
- `Collectors.joining()`

## StringBuffer || StringBuilder

Of course, there are cases where you simply want to concatenate multiple strings that are already in a `Collection` or array. But in most cases where string concatenation is needed, there is usually some rule involved. For example, joining values with a dash (`-`), underscore (`_`), or comma (`,`). In those cases, using `StringBuffer` or `StringBuilder` is a little disadvantageous compared with the others, because you have to be very careful with the code to avoid leaving a delimiter at the end. A case like that looks like this:

```java
// Example using StringBuffer
List<String> list = List.of("A", "B", "C");
String delimiter = ", ";

StringBuffer buffer = new StringBuffer();
// Append each List element and the delimiter
for (String string : list) {
    buffer.append(string);
    buffer.append(delimiter);
}

String result = buffer.toString(); // A, B, C,
```

If you deliberately want to avoid adding a delimiter at the end, you would probably need code like this:

```java
List<String> list = List.of("A", "B", "C");
String delimiter = ", ";
int limit = list.size() - 1;

StringBuffer buffer = new StringBuffer();
// Append each List element and the delimiter (up to before the last index)
for (int i = 0; i < limit; i++) {
    buffer.append(list.get(i));
    buffer.append(delimiter);
}
// Append the last element
buffer.append(list.get(limit));

String result = buffer.toString(); // A, B, C
```

Compared with this problem, the other approaches (`String.join()`, `StringJoiner`, `Collectors.joining()`) do not place a delimiter after the last element, so they have the advantage of letting you write simpler code. So the conclusion is that `StringBuffer` and `StringBuilder` are not great choices from a readability standpoint when you need to join multiple strings using a specific rule.

## StringJoiner

Because `StringBuffer` and `StringBuilder` concatenate strings in a loop, you might think they are useful when you also need other processing inside the loop, such as conditionals. Even in those cases, though, `StringJoiner` is still the better choice. Its usage is almost the same, and it never leaves a trailing delimiter without extra work. The following is the most basic way to use `StringJoiner`.

```java
List<String> list = List.of("A", "B", "C");
// Create an instance with a delimiter
StringJoiner joiner = new StringJoiner(", ");
// Then add elements one by one
for (String string : list) {
    joiner.add(string);
}
String result = joiner.toString(); // A, B, C
```

`StringJoiner` also lets you specify a prefix and suffix. If you do, those values are added to the beginning and end of the resulting string.

```java
List<String> list = List.of("A", "B", "C");
// Specify delimiter, prefix, and suffix
StringJoiner joiner = new StringJoiner(", ", "[", "]");
for (String string : list) {
    joiner.add(string);
}
String result = joiner.toString(); // [A, B, C]
```

Even just looking at the usage, it is clear that `StringJoiner` is simpler than `StringBuffer` or `StringBuilder` when you want to join strings with a delimiter.

### Extra: StringJoiner implementation

As a side note, let us look at how `StringJoiner` is implemented. First, `add()`: interestingly, it behaves a lot like `ArrayList`. The `StringJoiner` class keeps a `String[]` field, and each time `add()` is called, it creates a larger copy.

```java
public StringJoiner add(CharSequence newElement) {
    final String elt = String.valueOf(newElement);
    if (elts == null) {
        elts = new String[8];
    } else {
        if (size == elts.length)
            elts = Arrays.copyOf(elts, 2 * size);
        len += delimiter.length();
    }
    len += elt.length();
    elts[size++] = elt;
    return this;
}
```

In `toString()`, you can see that it loops over the `String[]` field and joins elements with the delimiter. What is a little unusual is that, probably for performance reasons, it packs the string into a `char[]` first and then creates a new `String` instance to return it.

```java
public String toString() {
    final String[] elts = this.elts;
    if (elts == null && emptyValue != null) {
        return emptyValue;
    }
    final int size = this.size;
    final int addLen = prefix.length() + suffix.length();
    if (addLen == 0) {
        compactElts();
        return size == 0 ? "" : elts[0];
    }
    final String delimiter = this.delimiter;
    final char[] chars = new char[len + addLen];
    int k = getChars(prefix, chars, 0);
    if (size > 0) {
        k += getChars(elts[0], chars, k);
        for (int i = 1; i < size; i++) {
            k += getChars(delimiter, chars, k);
            k += getChars(elts[i], chars, k);
        }
    }
    k += getChars(suffix, chars, k);
    return new String(chars);
}
```

## String.join()

`String.join()` can be seen as syntactic sugar, much like `InputStream.transferTo()`. The actual code looks like this:

```java
public static String join(CharSequence delimiter,
        Iterable<? extends CharSequence> elements) {
    Objects.requireNonNull(delimiter);
    Objects.requireNonNull(elements);
    StringJoiner joiner = new StringJoiner(delimiter);
    for (CharSequence cs: elements) {
        joiner.add(cs);
    }
    return joiner.toString();
}
```

Apart from the null checks on the arguments, it is just `StringJoiner` without a prefix or suffix. So if you want shorter code, this is more convenient than using `StringJoiner` directly.

## Collectors.joining()

When using `Stream` for string concatenation, the appeal is that you can chain together processing such as `filter()`, `map()`, and `peek()`. Personally, I like Stream-based processing because the role, purpose, and scope of effect are all easy to see. That said, as I mentioned in a previous post, Stream is often slower than traditional loops, so you should choose appropriately depending on the situation.

So what does the implementation look like inside `Collectors.joining()`? In the end, it just uses `StringJoiner` internally. Compared with `String.join()` and `StringJoiner`, the only real differences are the Stream-based code shape and the performance overhead that comes with it.

```java
public static Collector<CharSequence, ?, String> joining(CharSequence delimiter,
                                                            CharSequence prefix,
                                                            CharSequence suffix) {
    return new CollectorImpl<>(
            () -> new StringJoiner(delimiter, prefix, suffix),
            StringJoiner::add, StringJoiner::merge,
            StringJoiner::toString, CH_NOID);
}
```

## toString()

Actually, when dealing with a `Collection` such as `List<String>`, there is an even easier way to build a string: call `toString()`. That gives you a comma-separated string right away. The downside is that the result includes `[]` at the beginning and end, so depending on the case you may need to remove them or extract the inner text with `substring()`. The following sample uses `substring()` to cut out only the text inside the brackets.

```java
List<String> list = List.of("A", "B", "C");
String toString = list.toString(); // [A, B, C]
String result = toString.substring(1, toString.length() - 1); // A, B, C
```

If the delimiter is not a comma, you can also first convert the `Collection` to a string with `toString()` and then call `replace()` to swap the delimiter afterward. But this is a very inefficient approach. Looking at the `replace()` implementation, you can see that it ends up building a new string with `StringBuilder` in a loop anyway. The actual code is below.

```java
public String replace(CharSequence target, CharSequence replacement) {
    String tgtStr = target.toString();
    String replStr = replacement.toString();
    int j = indexOf(tgtStr);
    if (j < 0) {
        return this;
    }
    int tgtLen = tgtStr.length();
    int tgtLen1 = Math.max(tgtLen, 1);
    int thisLen = length();

    int newLenHint = thisLen - tgtLen + replStr.length();
    if (newLenHint < 0) {
        throw new OutOfMemoryError();
    }
    StringBuilder sb = new StringBuilder(newLenHint);
    int i = 0;
    do {
        sb.append(this, i, j).append(replStr);
        i = j + tgtLen;
    } while (j < thisLen && (j = indexOf(tgtStr, j + tgtLen1)) > 0);
    return sb.append(this, i, thisLen).toString();
}
```
