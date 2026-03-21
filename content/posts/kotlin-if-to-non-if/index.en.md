---
title: "Rethinking Branching with if"
date: 2022-11-20
categories:
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
---

In most languages, writing logic that runs only when a certain condition is met usually means using a branching construct such as `if`. Some languages treat it as an expression rather than a statement, and alternatives like `switch` or the ternary operator also exist, but the basic idea is still the same: write a condition and the logic that follows it.

As with any useful tool, branching can also be dangerous. `if` is easy to use and very clear in the first version of an implementation, but from a maintenance perspective it can become a poor choice. If conditions grow or change, it becomes harder to know whether all cases are covered, and unit testing can become more difficult.

At the very least, that is why we often want to simplify `if` logic or replace it with something else, such as a design pattern that avoids branching syntax altogether. The code may become a little harder to read, but it can also become better from a maintenance standpoint.

Of course, removing `if` completely is close to impossible, and there is no need to go that far. The tool itself is not the problem; the problem is how it is used. Here I want to focus only on how code written with `if` can be refactored. This is meant to be beginner-friendly.

## Example `if` Code

Start with the following function. Given a code and an amount, it returns a discount based on the code. The example is simplified, but logic like this may exist in an e-commerce promotion system.

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    return if (code == "Facebook") {
        (amount * 0.1).roundToInt()
    } else if (code == "Twitter") {
        (amount * 0.15).roundToInt()
    } else if (code == "Instagram") {
        if (amount < 1000) {
            amount
        } else {
            1000
        }
    } else {
        0
    }
}
```

Some readers may already know exactly how they would refactor this, but I want to look at it from several angles, so I will go through the options one by one.

## Refactoring the Function

Let us first think about how to change the logic inside the function. We can simplify code, improve readability, eliminate missing cases, factor out common logic, or split things apart when the unit of processing feels unclear.

## Standard Library

The function above has `if` nested inside another `if`. The deeper that nesting becomes, the harder it is to call the code good. So let us start there.

One way to reduce the nesting is to use the standard library. You could also split the logic into a separate function, but delegating to the standard library lowers the burden of this function itself.

Kotlin provides [coerceAtMost()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/coerce-at-most.html), which caps a value at the given maximum. That means the logic "return `amount` if it is 1000 or less, otherwise return 1000" can be simplified with this function:

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    return if (code == "Facebook") {
        (amount * 0.1).roundToInt()
    } else if (code == "Twitter") {
        (amount * 0.15).roundToInt()
    } else if (code == "Instagram") {
        amount.coerceAtMost(1000) // Keep the value within the threshold
    } else {
        0
    }
}
```

This is just the standard library being used, but one level of nesting disappears and the code becomes simpler. If the threshold ever changes, you only need to update one place.

It is a good idea to actively use the standard library wherever it fits, and when similar logic repeats, to extract it into your own reusable library.

## `when`

Another refactoring option is to replace `if` with `when` (similar to `switch` in other languages). `when` is still the same basic feature, so it is not always a better substitute. But when the conditions are uniform, it can be a good choice.

In the earlier example, the branching condition is only whether the `code` string has a certain value. Since there are no other conditions, using `when` keeps the branches based purely on strings and makes the code cleaner:

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    return when (code) { // Branch only by comparing the value of code
        "Facebook" -> (amount * 0.1).roundToInt()
        "Twitter" -> (amount * 0.15).roundToInt()
        "Instagram" -> amount.coerceAtMost(1000)
        else -> 0
    }
}
```

Just by changing `if` to `when`, the whole thing becomes shorter and easier to read. Looking at the condition itself and deciding whether it fits `when` can be a good choice in some situations.

## Extension Functions

Languages like Kotlin let you add methods to existing classes through extension functions. If the same logic repeats in more than one place, you should consider splitting it into a function, and in some cases that function can be an extension.

Here, the branching is based on whether `code` is `Facebook` or `Twitter`, but what we really want is to multiply `amount` by a percentage and return it. So it makes sense to define a helper for percentage calculations.

If that helper is only used here, a private function would be fine, but if you want it to be more reusable, an extension like the following is a good option:

```kotlin
// Extension function for calculating a percentage
infix fun Int.percentOf(amount: Int): Int = (amount * this / 100)

fun getDiscountAmount(code: String, amount: Int): Int {
    return when (code) {
        "Facebook" -> 10 percentOf amount // Return 10 percent of the amount
        "Twitter" -> 15 percentOf amount // Return 15 percent of the amount
        "Instagram" -> amount.coerceAtMost(1000)
        else -> 0
    }
}
```

Compared with the previous code, this version not only shares the calculation logic, but also makes the intention "calculate a percentage" much clearer.

## Map

There are cases where branching can be done without `if` or `when`. One example is using a `Map`. Since we want to apply a different multiplier based on `code`, we can define a map with `code` as the key and the percentage as the value:

```kotlin
// Codes and discount percentages
val discountPercentages = mapOf(
    "Facebook" to 10,
    "Twitter" to 15
)

fun getDiscountAmount(code: String, amount: Int): Int {
    // If a percentage is defined, apply it
    discountPercentages[code]?.let {
        return it percentOf amount
    }

    // If the code is not in the map
    return when (code) {
        "Instagram" -> amount.coerceAtMost(1000)
        else -> 0
    }
}
```

This approach does not cover every branch by itself. If `code` is not in the map, you still need a fallback. If you use closures, you can also include the `Instagram` logic in the map:

```kotlin
// Change the value type to (Int) -> Int
val discountRules = mapOf(
    "Facebook" to { amount: Int -> 10 percentOf amount },
    "Twitter" to { amount: Int -> 15 percentOf amount },
    "Instagram" to { amount: Int -> amount.coerceAtMost(1000) }
)

fun getDiscountAmount(code: String, amount: Int): Int {
    return discountRules[code]?.let { it(amount) } ?: 0
}
```

I would not say this is always better than branching, but if other functions also need the same discount rates, or if you want to keep shared values across multiple classes, this is a reasonable option. In that case, updating a single map keeps the rest of the logic consistent.

## OOP Thinking

So far I have only talked about changing the logic inside the function. Of course, there are more advanced approaches. From an OOP perspective, the function above is responsible for calculating the discount amount, but it also contains the definition of the discount itself and the calculation formula. That means responsibilities should be separated.

This kind of refactoring may look more complicated at first, but it also aligns with the OOP principles summarized as [SOLID](https://en.wikipedia.org/wiki/SOLID). In the long run, this approach is generally better for maintainability.

## Extract an Interface

First, we separate the "discount policy" into an `interface`. The classes implementing this interface will calculate the discount according to each policy.

```kotlin
interface DiscountPolicy {
    fun calculate(amount: Int): Int

    companion object {
        val NONE: DiscountPolicy = object : DiscountPolicy {
            override fun calculate(amount: Int): Int = 0
        }
    }
}
```

Then define a class for each code that implements the interface:

```kotlin
class FacebookDiscountPolicy : DiscountPolicy {
    override fun calculate(amount: Int): Int = 10 percentOf amount
}

class TwitterDiscountPolicy : DiscountPolicy {
    override fun calculate(amount: Int): Int = 15 percentOf amount
}

class InstagramDiscountPolicy : DiscountPolicy {
    override fun calculate(amount: Int): Int = amount.coerceAtMost(1000)
}
```

With those policies in place, `getDiscountAmount()` can be rewritten like this:

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    val discountPolicy = when (code) {
        "Facebook" -> FacebookDiscountPolicy()
        "Twitter" -> TwitterDiscountPolicy()
        "Instagram" -> InstagramDiscountPolicy()
        else -> DiscountPolicy.NONE
    }
    return discountPolicy.calculate(amount)
}
```

## Factory

Even though the interface extraction separates the discount policy itself, `getDiscountAmount()` still has the responsibility of creating the policy. That can also be split out into a separate role. A factory like this works well:

```kotlin
object DiscountPolicyFactory {
    fun getDiscountPolicy(code: String): DiscountPolicy {
        return when (code) {
            "Facebook" -> FacebookDiscountPolicy()
            "Twitter" -> TwitterDiscountPolicy()
            "Instagram" -> InstagramDiscountPolicy()
            else -> DiscountPolicy.NONE
        }
    }
}
```

In the end, `getDiscountAmount()` can be simplified like this. The amount of code increases because of the interface and factory, but the responsibility of this function becomes lighter, and it becomes easier to add or change discount policies later.

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    val discountPolicy = DiscountPolicyFactory.getDiscountPolicy(code)
    return discountPolicy.calculate(amount)
}
```

## Enum

Instead of using a factory to create discount policies, you can also use an enum. This is a way of extending `DiscountPolicy` and handling the policies as enum constants rather than classes:

```kotlin
enum class DiscountPolicies(private val code: String) : DiscountPolicy {
    FACEBOOK("Facebook") {
        override fun calculate(amount: Int): Int = 10 percentOf amount
    },
    TWITTER("Twitter") {
        override fun calculate(amount: Int): Int = 15 percentOf amount
    },
    INSTAGRAM("Instagram") {
        override fun calculate(amount: Int): Int = amount.coerceAtMost(1000)
    };

    companion object {
        fun fromCode(code: String): DiscountPolicy {
            return values().find { it.code == code } ?: DiscountPolicy.NONE
        }
    }
}
```

With that enum, `getDiscountAmount()` becomes:

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    val discountPolicy = DiscountPolicies.fromCode(code)
    return discountPolicy.calculate(amount)
}
```

In the enum version, `fromCode()` removes the need to branch on `code` directly, and adding a new policy is as easy as adding another enum constant. I think this is better than the factory approach.

## In Closing

As I said at the beginning, removing every `if` is close to impossible, and it is not even necessary. But you still need to watch the essence of the logic, its responsibility, and its readability carefully. It is worth writing something that works with `if` first, then later asking whether it can be improved in another way.

I do not always write clean code myself, but I still think it is important to step back sometimes and review code I have written with a beginner's mindset. Good code is always hard. But if you do the hard part early, you are less likely to regret it later.

See you again.
