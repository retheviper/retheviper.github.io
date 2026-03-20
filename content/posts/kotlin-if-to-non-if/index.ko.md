---
title: "if 분기를 어떻게 줄일까"
date: 2022-11-20
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
translationKey: "posts/kotlin-if-to-non-if"
---

대부분의 언어에서 조건에 따라 다른 처리를 해야 할 때 가장 먼저 떠오르는 것은 `if`입니다. 언어에 따라 `switch`나 삼항 연산자 같은 다른 선택지가 있더라도, 결국 "조건에 따라 분기한다"는 본질은 크게 다르지 않습니다.
문제는 `if`가 너무 익숙한 만큼, 처음에는 간단해 보여도 나중에는 유지보수를 어렵게 만들 수 있다는 점입니다. 조건이 계속 늘어나거나 바뀌기 시작하면, 모든 경우를 빠짐없이 다루고 있는지 확인하기가 어려워지고 테스트도 함께 복잡해집니다.
그래서 경우에 따라서는 `if` 자체를 더 단순하게 만들거나, 아예 다른 구조로 바꿔 분기 구문을 줄이는 편이 나을 때도 있습니다. 물론 모든 `if`를 없애는 것은 불가능하고, 그럴 필요도 없습니다. 중요한 것은 `if`를 무조건 피하는 것이 아니라, 더 나은 구조가 있는지 한 번쯤 돌아보는 일입니다. 이번 글에서는 그런 리팩터링 아이디어를 몇 가지 정리해 보겠습니다.
## if 문 예제

우선 다음 함수를 보겠습니다. 코드와 금액을 받아, 코드 종류에 따라 할인 금액을 계산해 돌려주는 예입니다. 매우 단순화한 코드지만, 쇼핑몰 프로모션처럼 비슷한 처리는 실제로도 자주 볼 수 있습니다.
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

이미 어떻게 리팩터링하면 좋을지 감이 오는 분도 있을 것 같습니다. 그래도 여기서는 한 가지 정답만 제시하기보다, 여러 관점에서 코드를 어떻게 바꿀 수 있는지 차례대로 보겠습니다.
## 함수 리팩터

먼저 함수 내부에서 할 수 있는 리팩터링부터 보겠습니다. 코드를 더 단순하게 만들거나, 중복을 줄이거나, 책임이 섞여 있는 부분을 분리하는 식으로 접근할 수 있습니다.
### 표준 라이브러리

위 함수는 `if` 안에 또 `if`가 들어 있는 구조입니다. 이런 중첩은 깊어질수록 읽기 어려워집니다. 그래서 먼저 이 부분부터 줄여 보겠습니다.

방법 중 하나는 표준 라이브러리를 쓰는 것입니다. 꼭 라이브러리가 아니더라도 별도 함수로 분리할 수는 있지만, 이미 적절한 함수가 있다면 그쪽에 맡기는 편이 더 단순합니다.

Kotlin에는 [coerceAtLeast()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/coerce-at-least.html)가 있습니다. 전달한 값을 최소값 기준으로 보정해 주는 함수입니다. 여기서는 `amount`가 1000보다 작으면 `amount`를, 그렇지 않으면 1000을 돌려주는 로직으로 단순화할 수 있습니다.
```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    return if (code == "Facebook") {
        (amount * 0.1).roundToInt()
    } else if (code == "Twitter") {
        (amount * 0.15).roundToInt()
    } else if (code == "Instagram") {
        amount.coerceAtLeast(1000) // 임계값을 넘지 않는 값이 된다
    } else {
        0
    }
}
```

표준 라이브러리만 썼을 뿐인데 중첩 하나가 바로 사라졌습니다. 이런 수정은 코드도 짧아지고, 나중에 임계값을 바꿔야 할 때도 한 곳만 보면 된다는 장점이 있습니다. 비슷한 처리가 반복된다면 직접 함수로 빼거나, 이미 있는 표준 라이브러리 함수로 치환하는 습관이 꽤 도움이 됩니다.
### when

또 다른 방법은 `if`를 `when`으로 바꾸는 것입니다. `when`이 항상 더 좋은 것은 아니지만, 조건이 한 변수의 값 비교에 집중돼 있다면 `when`이 더 읽기 쉬운 경우가 많습니다.

여기서는 분기 조건이 전부 `code` 문자열 비교뿐입니다. 다른 부가 조건이 없으니 `when`으로 옮기면 훨씬 명확해집니다.
```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    return when (code) { // code 값을 비교하기만 하는 분기
        "Facebook" -> (amount * 0.1).roundToInt()
        "Twitter" -> (amount * 0.15).roundToInt()
        "Instagram" -> amount.coerceAtLeast(1000)
        else -> 0
    }
}
```

`if`를 `when`으로 바꾸는 것만으로도 코드가 짧아지고 구조가 더 잘 보입니다. 결국 중요한 건 "이 분기가 한 값을 기준으로 나뉘는가"를 보는 일입니다. 그렇다면 `when`이 꽤 좋은 선택지가 됩니다.
### 확장 함수

Kotlin에서는 확장 함수로 기존 타입에 메서드를 추가할 수 있습니다. 비슷한 처리가 여러 곳에서 반복된다면 함수로 분리하고, 경우에 따라서는 확장 함수 형태로 두는 것도 괜찮습니다.

여기서 `Facebook`, `Twitter` 분기가 실제로 하는 일은 결국 "금액에 특정 퍼센트를 곱한다"는 것입니다. 그렇다면 이 계산을 따로 함수로 뽑는 편이 좋습니다.

이 로직이 이 파일 안에서만 쓰인다면 `private` 함수여도 괜찮지만, 더 넓게 써 보고 싶다면 아래처럼 확장 함수로 둘 수도 있습니다.
```kotlin
// 퍼센트를 구하는 확장 함수
infix fun Int.percentOf(amount: Int): Int = (amount * this / 100)

fun getDiscountAmount(code: String, amount: Int): Int {
    return when (code) {
        "Facebook" -> 10 percentOf amount // 10퍼센트 값을 반환한다
        "Twitter" -> 15 percentOf amount // 15퍼센트 값을 반환한다
        "Instagram" -> amount.coerceAtLeast(1000)
        else -> 0
    }
}
```

앞선 코드와 비교하면, 확장 함수로 공통 처리를 뽑아낸 덕분에 중복이 줄었고 "퍼센트를 계산한다"는 의도도 더 직접적으로 드러납니다.
### Map

`if`나 `when` 없이도 분기 비슷한 구조를 만들 수 있습니다. 대표적인 방법이 `Map`을 활용하는 방식입니다. 여기서는 `code`를 key로, 할인 규칙을 value로 두면 됩니다. 예를 들면 다음과 같습니다.
```kotlin
// 코드와 할인율
val discountPercentages = mapOf(
    "Facebook" to 10,
    "Twitter" to 15
)

fun getDiscountAmount(code: String, amount: Int): Int {
    // 할인율이 정의되어 있으면 곱한다
    discountPercentages[code]?.let {
        return it percentOf amount
    }

    // Map에 없는 code인 경우
    return when (code) {
        "Instagram" -> amount.coerceAtLeast(1000)
        else -> 0
    }
}
```

다만 이 방식만으로는 모든 분기를 다 표현하기는 어렵습니다. `code` 값이 `Map`의 key에 없을 때 처리가 따로 필요하기 때문입니다. 이럴 때는 값을 함수로 바꿔 두면 `Instagram` 같은 예외 규칙도 `Map` 안에 넣을 수 있습니다.
```kotlin
// Value를 (Int) -> Int로 바꾼다
val discountRules = mapOf(
    "Facebook" to { amount: Int -> 10 percentOf amount },
    "Twitter" to { amount: Int -> 15 percentOf amount },
    "Instagram" to { amount: Int -> amount.coerceAtLeast(1000) }
)

fun getDiscountAmount(code: String, amount: Int): Int {
    return discountRules[code]?.let { it(amount) } ?: 0
}
```

`Map`이 항상 조건 분기보다 좋은 것은 아닙니다. 그래도 코드별 할인 규칙을 여러 함수나 클래스에서 함께 참고해야 한다면, 이렇게 공통 데이터를 한곳에 모으는 방식이 더 잘 맞을 수 있습니다. 이 경우에는 `Map` 하나만 수정해도 전체 규칙이 함께 바뀐다는 장점이 있습니다.
## OOP적인 사고방식

지금까지는 단순히 함수 내부의 처리를 어떻게 바꾸어 가는지에 대해 이야기했습니다만, 보다 고도의 방법도 물론 있습니다. OOP의 사고방식으로 파악하면, 방금 전의 함수는 「할인액을 구한다」책임이 있습니다만, 그 중에서 「할인」의 정의 그 자체와, 그 계산식까지 가지고 있는 것입니다. 그래서 책임을 분리해 나갈 필요가 있네요.
이 수정에 처리는 일견보다 복잡한 것이 되어 간다고 느끼는 경우도 있을까 생각합니다만, 이것은 OOP에 원칙인 [SOLID](https://ko.wikipedia.org/wiki/SOLID)를 고려한 것도 있습니다. 장기적인 관점에서 보면, 이러한 방법을 취하는 것이 더 유지 보수에 적합하게 될 것입니다.
### 인터페이스 추출

먼저 '할인 정책'을 `insterface`으로 분리합니다. 이 할인 정책를 구현하는 클래스에서 실제의 정책에 따른 할인액을 계산하는 이미지입니다.
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

그리고는 이 `interface` 를 구현하는 클래스를 코드별로 정의해 둡니다.
```kotlin
class FacebookDiscountPolicy : DiscountPolicy {
    override fun calculate(amount: Int): Int = 10 percentOf amount
}

class TwitterDiscountPolicy : DiscountPolicy {
    override fun calculate(amount: Int): Int = 15 percentOf amount
}

class InstagramDiscountPolicy : DiscountPolicy {
    override fun calculate(amount: Int): Int = amount.coerceAtLeast(1000)
}
```

이렇게 할인 정책을 정의하면 `getDiscountAmount()`은 다음과 같이 바뀔 수 있습니다.
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

### 공장

방금 전의 `interface`추출로 할인 정책 자체는 분리할 수 있었지만, `getDiscountAmount()`에서는 아직 「할인 정책을 생성한다」라고 하는 책임을 가지고 있습니다. 이것도 또 다른 역할로서 분리할 수 있을 것이다. 여기에서는 아래와 같이 할인 정책을 생성하는 `Factory` 를 정의해 두면 좋을 것입니다.
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

궁극적으로 `getDiscountAmount()`은 다음과 같이 수정할 수 있습니다. `interface`의 추출이나 `Factory`의 작성으로 코드의 양은 증가했지만, 이 함수의 책임은 보다 가벼워져, 할인 정책의 추가나 수정이 필요한 경우에서도 유연한 대응을 할 수 있게 되었습니다.
```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    val discountPolicy = DiscountFactory.getDiscountPolicy(code)
    return discountPolicy.calculate(amount)
}
```

### Enum

할인 정책을 생성하기 위해 `Factory`을 사용하는 대신 `Enum`을 사용할 수도 있습니다. 방금 전의 `DiscountPolicy`을 상속해, 클래스가 아니고 열거 정수로서 취급하는 방법입니다. 예를 들어 다음과 같은 것을 정의할 수 있습니다.
```kotlin
enum class DiscountPolicies(private val code: String) : DiscountPolicy {
    FACEBOOK("Facebook") {
        override fun calculate(amount: Int): Int = 10 percentOf amount
    },
    TWITTER("Twitter") {
        override fun calculate(amount: Int): Int = 15 percentOf amount
    },
    INSTAGRAM("Instagram") {
        override fun calculate(amount: Int): Int = amount.coerceAtLeast(1000)
    };

    companion object {
        fun fromCode(code: String): DiscountPolicy {
            return values().find { it.code == code } ?: DiscountPolicy.NONE
        }
    }
}
```

위의 `Enum`을 사용하는 경우 `getDiscountAmount()`은 다음과 같습니다.
```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    val discountPolicy = DiscountPolicies.fromCode(code)
    return discountPolicy.calculate(amount)
}
```

`Enum`을 사용하면 `fromCode()` 안에서 조건 분기를 길게 늘어놓지 않아도 되고, 할인 정책을 추가할 때도 열거 상수만 늘리면 되니 확장하기가 더 쉽습니다. 이런 경우에는 `Factory`보다 더 잘 맞는 선택일 수 있습니다.
## 마지막으로

처음에도 말했듯이, 모든 `if`를 없애는 것은 불가능에 가깝고 그럴 필요도 없습니다. 다만 어떤 `if`가 책임을 너무 많이 떠안고 있는지, 더 읽기 쉬운 구조로 바꿀 수 있는지는 한 번쯤 점검해 볼 필요가 있습니다. 처음에는 `if`로 빠르게 동작하는 코드를 만들더라도, 나중에 다른 구조로 다듬을 수 있다면 그 편이 더 나을 수 있습니다.
저 역시 항상 깔끔한 코드를 쓰는 것은 아닙니다. 그래도 가끔은 초보자처럼 다시 코드를 읽고, 더 나은 구조가 없는지 돌아보는 습관이 중요하다고 생각합니다. 좋은 코드는 늘 어렵지만, 유지보수 비용을 먼저 줄여 두는 쪽이 결국은 더 편합니다.
