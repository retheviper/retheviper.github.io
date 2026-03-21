---
title: "Kotlin으로 써 본 코드 2"
date: 2021-06-28
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
translationKey: "posts/kotlin-code-in-my-style-2"
---

[Kotlin으로 써 본 코드 1](../kotlin-code-in-my-style-1)에 이어, 이번에도 Kotlin에서 자주 쓰게 되는 작은 기능들을 정리해 보려 합니다. Kotlin은 표준 라이브러리와 언어 차원에서 제공하는 기능이 꽤 많아서, 이런 것들을 얼마나 잘 알고 있느냐에 따라 생산성과 코드 품질이 꽤 달라집니다. 이번 글도 Java식 코드를 Kotlin에서 어떻게 더 간결하게 바꿀 수 있는지에 초점을 맞췄습니다.

물론 Kotlin이라고 해서 Java 스타일로 쓰면 안 되는 것은 아닙니다. 그래도 Kotlin에 맞는 표현으로 바꾸면 더 짧고, 더 분명하고, 종종 더 안전한 코드를 만들 수 있습니다. 그런 이유로 이번에도 실제로 써 본 기능 몇 가지를 소개합니다.

## 순번 데이터 만들기

유닛 테스트를 작성하다 보면 테스트용 데이터를 순서대로 만들어야 할 때가 있습니다. 예를 들어 `Data01`, `Data02`, `Data03`처럼 번호가 붙은 데이터를 만들고 싶은 경우입니다.

보통은 루프로 리스트를 만들고, 마지막에 모아서 반환하는 식으로 작성합니다. 예를 들면 다음과 같습니다.

```kotlin
// 테스트용 데이터를 생성한다
fun createTestDatas(): List<String> {
    // 테스트용 데이터 목록
    val testDatas = mutableListOf<String>()
    
    // 10개의 데이터를 추가한다
    for (i in 0 until 10) {
        testDatas.add("테스트$i")
    }

    // read-only로 변환해 반환한다
    return testDatas.toList()
}
```

하지만 Kotlin답게 쓰려면, 이런 루프를 조금 더 간단하게 줄일 수 있습니다.

### repeat

가장 먼저 떠올릴 수 있는 방법은 `repeat`입니다. 반복 횟수가 정해져 있다면 그에 맞는 함수를 쓰는 편이 좋습니다.

```kotlin
fun createTestDatas(): List<String> {
    val testDatas = mutableListOf<String>()

    // 10번 반복한다
    repeat(10) { 
        testDatas.add("테스트$i")
    }

    return testDatas.toList()
}
```

다음으로 생각할 수 있는 것은 `MutableList`를 처음부터 쓰지 않는 방법입니다. 테스트용 데이터처럼 수정할 필요가 없는 값이라면, 처음부터 `List`로 만드는 편이 더 자연스럽습니다. 여기에는 두 가지 방법이 있습니다. 하나는 크기를 지정한 `List`를 만드는 것이고, 다른 하나는 범위, 즉 [Range](https://kotlinlang.org/docs/ranges.html#range)를 사용하는 것입니다.

### List

우선 크기를 지정한 `List`를 만드는 방법을 보겠습니다. 인스턴스를 만들 때 크기와 초기화 함수를 함께 넘기면 원하는 길이의 리스트를 바로 만들 수 있습니다.

```kotlin
fun createTestDatasByList(): List<String> =
    List(10) { "테스트$it" }
```

실제로는 앞서 본 루프를 조금 더 간단하게 쓴 것과 다르지 않습니다. 구현도 결국 내부에서 `repeat`를 돌리는 형태라서, 문법 설탕에 가깝습니다.

```kotlin
@SinceKotlin("1.1")
@kotlin.internal.InlineOnly
public inline fun <T> List(size: Int, init: (index: Int) -> T): List<T> = MutableList(size, init)

@SinceKotlin("1.1")
@kotlin.internal.InlineOnly
public inline fun <T> MutableList(size: Int, init: (index: Int) -> T): MutableList<T> {
    val list = ArrayList<T>(size)
    repeat(size) { index -> list.add(init(index)) }
    return list
}
```

`List`를 쓰면 초기화 함수에서 `it`을 이용해 원하는 패턴을 만들 수도 있습니다. 예를 들면 이런 식입니다.

```kotlin
List(5) { "Test${ it * 2 }" }
// [Test0, Test2, Test4, Test6, Test8]

List(5) { (it * 2).let { index -> "$index는 짝수" } }
// [0는 짝수, 2는 짝수, 4는 짝수, 6는 짝수, 8는 짝수]
```

다만 이 방식은 결과적으로 `MutableList`를 만들기 때문에, 최종적으로는 다시 `toList()`로 바꿔야 할 수도 있습니다.

### Range

다른 방법으로는 `Range`를 쓰는 방식이 있습니다. 숫자 범위를 한 번에 만들 수 있어서, 이 경우도 꽤 깔끔합니다.

```kotlin
// Range를 사용해 테스트 데이터를 만든다
fun createTestDatasByRange(): List<String> =
    (0..10).map { "테스트%it" }
```

`List`와 달리 `Range`는 [IntRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-int-range), [LongRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-long-range), [CharRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-char-range) 등으로 나뉘어 있어서, 숫자나 문자를 다루는 작업에 꽤 잘 맞습니다.

성능도 보통 `List`보다 `Range` 쪽이 좋게 나오는 경우가 많습니다. 아래처럼 벤치마크하면, 대부분 `Range` 쪽이 더 빠르게 측정됩니다.

```kotlin
import kotlin.system.measureTimeMillis

data class Person(val name: String, val Num: Int)

fun main() {
    benchmark { list() }
    benchmark { range() }
}

fun benchmark(function: () -> Unit) =
    println(measureTimeMillis { function() })
    
fun list() =
    List(200000) { Person("person$it", it) }
    
fun range(): List<Person> =
    (0..200000).map { Person("person$it", it) }
```

다만 `Range`는 기본적으로 1씩 증가하므로, `step`을 세밀하게 조정하는 작업에는 덜 유연할 수 있습니다. 상황에 따라 `List`와 `Range` 중 무엇이 맞는지 골라 쓰면 됩니다.

## 검증

검증 코드에서는 파라미터 값을 확인해야 할 때가 많습니다. Kotlin은 `nullable`과 non-null 타입이 나뉘어 있어서 Java보다 `null` 실수를 줄이기 쉽지만, 그래도 비즈니스 로직 차원에서 직접 검증해야 하는 경우는 여전히 있습니다.

우선 Java 스타일로 쓰면 보통 이런 식이 됩니다.

```kotlin
fun doSomething(parameter: String): String {
    if (parameter.isBlank()) {
        throw IllegalArgumentException("문자열이 비어 있습니다")
    }

    val result = someRepository.find(parameter)

    if (result == null) {
        throw IllegalStateException("결과가 null입니다")
    }
    return result
}
```

비슷한 역할을 Swift의 `guard`처럼 분리해서 쓰고 싶다면, Kotlin에서는 `requireNotNull`과 `checkNotNull`을 사용할 수 있습니다.

```swift
func doSomething(parameter: String) throws -> String {
    guard !parameter.isEmpty else {
        throw ValidationError.invalidArgument
    }

    guard let result = someRepository.find(parameter) else {
        throw ValidationError.notFound
    }
    return result
}
```

Kotlin에서는 다음처럼 쓸 수 있습니다.

```kotlin
fun doSomething(parameter: String?): String {
    val checkedParameter = requireNotNull(parameter) {
        "문자열이 null입니다"
    }

    val result = someRepository.find(checkedParameter)

    return checkNotNull(result) {
        "결과가 null입니다"
    }
}
```

`requireNotNull`은 인수가 `null`이면 `IllegalArgumentException`을 던지고, 그렇지 않으면 non-null 값으로 돌려줍니다. `checkNotNull`은 비슷하지만 `IllegalStateException`을 던진다는 점이 다릅니다. 그래서 입력 검증과 상태 검증을 나눠서 쓰기 좋습니다.

비슷하게 `require`도 있습니다. 이 함수는 `null`뿐 아니라 조건식 자체를 검증할 수 있어서, 숫자 범위 같은 것도 쉽게 확인할 수 있습니다.

```kotlin
fun doSomething(parameter: Int) {
    require(parameter > 100) {
        "$parameter는 너무 큽니다"
    }

    // ...
}
```

또 다른 방법으로는 [Elvis operator](https://kotlinlang.org/docs/null-safety.html#elvis-operator)를 사용할 수 있습니다. 이 방식은 `null`일 때 기본값을 넣거나 예외를 바로 던지는 식으로 자주 씁니다.

```kotlin
fun doSomething(parameter: String?): String {
    val checkedParameter = parameter ?: "default"

    val result = someRepository.find(checkedParameter)

    return result ?: throw CustomException("결과가 null입니다")
}
```

## List 분할

조건에 맞는 데이터만 골라내고 싶을 때는 `filter`를 쓰면 됩니다. 그런데 한 리스트를 두 갈래로 나누고 싶다면 어떻게 해야 할까요? 예를 들어 조건에 맞는 요소와 그렇지 않은 요소를 각각 다른 리스트로 만들고 싶은 경우입니다.

가장 먼저 떠오르는 방법은 두 번 순회하는 것입니다. 하지만 이 방법은 효율이 좋지 않습니다.

```kotlin
val origin = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// 홀수를 추출
val odd = origin.filter { it % 2 != 0 }
// 짝수를 추출
val even = origin.filter { it % 2 == 0 }
```

루프를 한 번만 돌리고 싶다면, 미리 빈 리스트를 준비한 뒤 분기 처리하는 방법도 있습니다.

```kotlin
val origin = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// 홀수와 짝수 목록을 선언해 둔다
val odd = mutableListOf<Int>()
val even = mutableListOf<Int>()

// 루프 처리
origin.forEach {
    if (it % 2 != 0) {
        odd.add(it) // 홀수 목록에 추가
    } else {
        even.add(it) // 짝수 목록에 추가
    }
}
```

하지만 이런 용도라면 Kotlin 표준 라이브러리의 `partition`을 쓰는 편이 훨씬 낫습니다. `partition`은 리스트를 조건에 맞는 것과 아닌 것으로 바로 나눠 줍니다. 반환값도 `Pair<List<T>, List<T>>`라서, [destructuring-declaration](https://kotlinlang.org/docs/destructuring-declarations.html)과 함께 쓰면 더 짧게 끝납니다.

```kotlin
val origin = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
val (odd, even) = origin.partition { it % 2 != 0 } // 조건에 일치하는 것과 일치하지 않는 것으로 목록을 분리한다
```

## 마지막으로

Kotlin은 편리한 기능이 정말 많은 언어입니다. 그래서 API를 제대로 알고 쓰느냐에 따라 코드 품질 차이가 꽤 크게 납니다. 버전업도 빠르고 새로운 기능도 계속 추가되니, 따라가는 습관도 중요합니다.

그래도 하나씩 익혀 가다 보면 Kotlin으로 할 수 있는 일이 계속 늘어난다는 점이 재미있습니다. 공부할수록 도움이 되는 언어를 쓰는 건 꽤 기분 좋은 일입니다. 이 시리즈도 그런 흐름으로 계속 이어 갈 생각입니다.
