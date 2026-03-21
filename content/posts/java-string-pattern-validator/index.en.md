---
title: "Determine whether a string matches a pattern"
date: 2020-11-22
categories: 
  - java
image: "../../images/java.webp"
tags:
  - java
  - pattern
  - regular expression
---
In general, there are things that need to be considered in terms of business requirements and security for applications, so when creating a function, you may not only need to know whether it will work, but also some logic. Therefore, in order for the function to work, in certain cases, restrictions may be required, such as processing according to certain conditions when it works.

This post was also born out of such business requirements. The project I'm currently involved in is creating a Spring Boot-based application that runs on EC2. This app has a simple function where you specify the file data and the path to upload it to, and it uploads it to S3, and you are in charge of it.

Creating a working function is simple if you simply have an upload path and data. Spring has a framework called Spring Cloud, so you can already have a class called [ResourceLoader](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/ResourceLoader.html) to achieve file upload. Even if you don't use Spring Cloud, you can easily implement it using [AWS SDK](https://aws.amazon.com/sdk-for-java/). In fact, yesterday's version only requires the upload destination path and file, so it's not even an implementation.

However, when this function was called, it was necessary to check that the upload path passed was the "correct" one. In other words, there are fixed path rules when storing business files in S3, and this function required a check to see if the path passed as a parameter matched the specified pattern.

What should I use to create a function to check whether the passed path is "correct"? And how should I make it? I think there are various ways to do it, but first I would like to introduce how I implemented it.

## String pattern is a regular expression

First, the file upload path is a string and must follow a specific pattern. To determine whether a string matches a required pattern, use a [regular expression](https://en.wikipedia.org/wiki/Regular_expression). So we define the "correct" upload path pattern in advance as a regular expression, then check whether the given parameter matches it. Java provides several ways to perform regex-based matching, so you need to decide which one to use. For example:

1. Use `Pattern.matches()`
1. Obtain and use `Matcher` from `Pattern`
1. Obtain and use `Predicate` from `Pattern`
1. Use `String.matches()`

And these methods can be used as per the code below.

```java
// Example regular expression
String patternRegex = "^[0-9]*$";
// String to validate with the regular expression
String value = "123456789"; 

// Using Pattern.matches()
boolean patternMatches = Pattern.matches(patternRegex, value);

// Using Matcher
Pattern pattern = Pattern.compile(patternRegex);
Matcher matcher = pattern.matcher(value);
boolean matcherFind = matcher.find(); // Partial match
boolean matcherMatches = matcher.matches(); // Full match

// Using Predicate
Pattern pattern = Pattern.compile(patternRegex);
boolean matcherFind = matcher.asPredicate().test(value); // Partial match
boolean matcherMatches = matcher.asMatchPredicate().test(value); // Full match

// Using String.matches()
boolean stringMatches = value.matches(patternRegex);
```

If you use `Matcher` or `Predicate` here, you can select partial match, so it seems that you have no choice but to use these in case of partial match. However, if you need an exact match, what criteria should you use to choose one? If they all produce similar results, you'll want to choose the more efficient method. And in this case, the consideration is performance. In other words, which method can be used to obtain the determination result most quickly.

## If everything is the same, it's performance

As mentioned above, there are various ways to determine whether a string matches a given regular expression pattern, so I would like to measure which is the fastest. The so-called stopwatch method (subtracting the time at the start of processing from the time at the end of processing) is easy, but I wanted to make a more accurate comparison, so I created a benchmark using [JMH](https://openjdk.java.net/projects/code-tools/jmh) provided by Openjdk. It's nice that it also measures several times during warm-up, probably to prevent the slow startup problem that is specific to Java from affecting the measurements.

The code used to actually perform the benchmark is as follows.

```java
public class RegexTest {

    private static final String PATTERN_REGEX = "^[0-9]*$";

    private static final DecimalFormat DECIMAL_FORMAT = new DecimalFormat("0000000");

    private static final Pattern PATTERN = Pattern.compile(PATTERN_REGEX);

    private static final Predicate PREDICATE = PATTERN.asPredicate();

    private static final Predicate MATCH_PREDICATE = PATTERN.asMatchPredicate();

    private static final List<String> VALUES = IntStream.rangeClosed(0, 9999999).mapToObj(DECIMAL_FORMAT::format).collect(Collectors.toList());

    @Benchmark
    public void patternMatches(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(Pattern.matches(PATTERN_REGEX, value));
        }
    }

    @Benchmark
    public void matcherFind(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(PATTERN.matcher(value).find());
        }
    }

    @Benchmark
    public void matcherMatches(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(PATTERN.matcher(value).matches());
        }
    }

    @Benchmark
    public void predicate(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(PREDICATE.test(value));
        }
    }

    @Benchmark
    public void matchPredicate(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(MATCH_PREDICATE.test(value));
        }
    }

    @Benchmark
    public void stringMatches(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(value.matches(PATTERN_REGEX));
        }
    }
}
```

And the measurement results are as follows. The order of the method names is different in the actual output, but it is changed to match the order in the code.

```dos
Benchmark                  Mode  Cnt  Score   Error  Units
RegexTest.patternMatches  thrpt   25  0.591 ± 0.013  ops/s
RegexTest.matcherFind     thrpt   25  1.525 ± 0.022  ops/s
RegexTest.matcherMatches  thrpt   25  1.481 ± 0.030  ops/s
RegexTest.predicate       thrpt   25  2.050 ± 0.182  ops/s
RegexTest.matchPredicate  thrpt   25  1.733 ± 0.236  ops/s
RegexTest.stringMatches   thrpt   25  0.609 ± 0.005  ops/s
```

Based on this result, it can be said that it is certainly better to use `Matcher` or `Predicate` in terms of performance. However, the benchmark results show that `Predicate` has the best performance even including errors, but `Pattern.asPredicate()` was introduced in Java 1.8 and `Pattern.asMatchPredicate()` was introduced in Java 11, so you need to choose the appropriate one according to the JDK version.However, I would like to know not only the results but also the reasons. `Matcher` and `Predicate`, which had good performance, have one thing in common: they created instances in advance during testing. Therefore, in the case of `Pattern.matches()` and `String.matches()`, which have low performance, we can speculate that the slowness may be due to the fact that an instance is created every time a method is called. Let's take a look at the code to see if it actually works.

First, regarding `Pattern.matches()`, the actual code is as follows.

```java
// Pattern.matches
public static boolean matches(String regex, CharSequence input) {
    Pattern p = Pattern.compile(regex);
    Matcher m = p.matcher(input);
    return m.matches();
}
```

You can see that instances of `Pattern` and `Matcher` are created each time the method is called (in fact, if you follow the code for `Pattern.compile()` and `Pattern.matcher()`, you can see that instances are created). So it's natural that they will be delayed.

Now, let's check the code for `String.matches()` as well. The actual code is below.

```java
// String.matches
public boolean matches(String regex) {
    return Pattern.matches(regex, this);
}
```

Again, this is just a call to `Pattern.matches()`, so it's slow. The only difference is that unlike `Pattern`, it is not a static method because it requires its own instance to be compared, but this does not affect performance, so I think the benchmark results were within the error range.

## Create the actual Validator

Now that we know that `Matcher` and `Predicate` have an advantage in performance, we can use this to create a validator that determines whether the passed path is acceptable. In my current project, I use Java 11, so I chose `Predicate`.

Since there are multiple path patterns, specify the pattern as an array or list. Also, since the judgment is made using `Predicate`, it is a good idea to create an instance with a specified pattern in advance, and when judgment is necessary, loop through the array or list of patterns and return whether there is a match. Based on this requirement, the actual code is as follows (although the regular expression for the path is different from the actual work).

```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class StorageValidator {

    /**
     * Allowed path patterns
     */
    private static final List<Predicate<String>> PATH_PATTERN_MATCHERS = List.of(
            createMatcher(
                    "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"),
            createMatcher(
                    "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail_backup\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"));

    /**
     *
     * Determines whether the given string is a valid file upload path that can be used in SPL.
     *
     * @param path the string to validate (file path)
     * @return validation result
     */
    public static boolean isValidUploadPath(String path) {
        return PATH_PATTERN_MATCHERS.stream().anyMatch(predicate -> predicate.test(path));
    }

    /**
     *
     * Returns a {@link Predicate}-based pattern matcher object from the given regular expression.
     *
     * @param pattern regular expression
     * @return generated pattern matcher object
     */
    private static Predicate<String> createMatcher(String pattern) {
        return Pattern.compile(pattern).asMatchPredicate();
    }
}
```

Now you can determine whether the passed path matches the expected pattern. It's easy.

## Extra: Why not write it in Kotlin?

It's not really related to the content of this post, but just out of curiosity, I converted a Validator created in Java to Kotlin code. Fortunately, intellij has a convenient feature that allows you to convert code written in Java to Kotlin, making it easy. Jetbrain is the company that created Kotlin in the first place, so I think this feature is meant to popularize Kotlin, but it also makes it easier for Java programmers to get started with Kotlin.

It seems that `static final` fields are treated as `companion object` in Kotlin, so the code itself doesn't feel that different. However, in Kotlin, there is a method (`any`) that can be called directly from Collection without calling `stream()`, and `List.of()` can be replaced with `listOf()`, but automatic conversion does not do that much, so you have to change those parts yourself. The completed code is as follows.

```kotlin
class StorageValidator {

    companion object {
         private val PATH_PATTERN_UPLOAD = listOf( // Valid patterns for image storage paths
            Pattern.compile(
                "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"
            )
                .asMatchPredicate(),
            Pattern.compile(
                "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail_backup\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"
            )
                .asMatchPredicate()
        )
    }

    fun isValidUploadPath(path: String): Boolean {
        return PATH_PATTERN_UPLOAD.any { predicate -> predicate.test(path) }
    }
}
```

## Finally

Actually, it's not that difficult to create a function that uses regular expressions to determine whether a string matches a pattern. If anything, I think it may be difficult to write the regular expression itself. However, recently there are many online tools such as [RegExr](https://regexr.com), [RegEx Testing](https://www.regextester.com), and [regular expressions 101](https://regex101.com) that allow you to create regular expressions while testing them on the spot, so if you take your time, I think you can create as many patterns as you want.

Personally, the code I wrote was short and simple, but I think it was quite an interesting task as it gave me a chance to think about various things (in terms of efficiency) after a long time. If such requirements continue to exist in the future, I would like to try a different method. See you soon!
