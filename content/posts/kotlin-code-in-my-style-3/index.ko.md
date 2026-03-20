---
title: "Kotlin으로 써 본 코드 3"
date: 2021-09-18
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
translationKey: "posts/kotlin-code-in-my-style-3"
---

Java에서 Kotlin으로 넘어온 입장에서 보면, Kotlin은 표준 라이브러리만으로도 꽤 많은 기능을 제공해서 Java보다 생산성이 높다고 느껴질 때가 많습니다. 다만 반대로, 어떤 함수를 언제 어떻게 써야 Kotlin다운 코드가 되는지는 여전히 고민되는 부분입니다. 그래서 이번에도 여러 방식으로 코드를 써 보면서, 그중 실무에서 써 볼 만하다고 느낀 것들을 정리해 보겠습니다.
## List 요소 스왑

List의 요소 순서를 바꾸는 방법은 정렬을 포함해 여러 가지가 있지만, 때로는 특정 두 요소만 맞바꾸고 싶을 때도 있습니다. 이럴 때 쓸 수 있는 간단한 확장 함수를 먼저 생각해 보았습니다.
### 인덱스를 알 때

스왑할 요소의 인덱스를 이미 알고 있다면, 결국 그 두 위치의 값을 바꾸면 됩니다. 이는 두 변수의 값을 교환하는 것과 같은 문제입니다. 가장 전통적인 방식은 다음과 같습니다.
```kotlin
var a = 10
var b = 20
var c = a

a = b
b = c
```

Kotlin스럽게 쓰고 싶다면 `also`를 활용하는 방법도 있습니다. 이렇게 하면 코드가 조금 더 짧아집니다.
```kotlin
var a = 10
var b = 20

a = b.also { b = a }
```

이와 같이, List의 요소를 스왑 하는 처리를 확장 함수로 쓰면(자) 다음과 같이 됩니다.
```kotlin
fun <T> List<T>.swapByIndex(indexFrom: Int, indexTo: Int): List<T> =
    toMutableList().apply {
        this[indexFrom] = this[indexTo].also { this[indexTo] = this[indexFrom] }
    }.toList()
```

### 인덱스를 모를 때

스왑할 요소를 알고는 있지만 인덱스를 모르는 경우도 있습니다. 이때도 결국 마지막에는 인덱스를 구해 값을 바꾸게 되므로, 먼저 인덱스를 찾는 단계만 추가하면 됩니다.
Kotlin에는 요소로 찾는 [indexOf](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/index-of.html)와 조건식으로 찾는 [indexOfFirst](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/index-of-first.html)가 있으니, 이를 앞서 만든 확장 함수와 조합하면 됩니다. 예를 들면 다음과 같습니다.
```kotlin
// indexOf(element)를 사용하는 경우
fun <T> List<T>.swapByElement(from: T, to: T): List<T> =
    swapByIndex(indexOf(from), indexOf(to))

// indexOfFirst(predicate)를 사용하는 경우
fun <T> List<T>.swapByCondition(from: (T) -> Boolean, to: (T) -> Boolean): List<T> =
    swapByIndex(indexOfFirst { from(it) }, indexOfFirst { to(it) })
```

## 시간을 숫자로

`java.time` 패키지의 `LocalDate`이나 `LocalDateTime`과 같은 객체는 코드에서 시간을 다루는 데 유용하지만 파일에 쓰는 등으로 포맷을 변경해야 할 때도 있습니다. 즉, `yyyy-MM-dd`이 아니라 `yyyyMMddhhmmss`과 같은 형태로 하고 싶은 경우가 있다는 것입니다. 이런 경우에는 간단하게 Int형으로 변경할 수 있는 확장 함수를 써 두면 편리하겠지요. 예를 들면 다음과 같은 것을 생각할 수 있습니다.
```kotlin
fun LocalDate.toInt(): Int = "$year$monthValue$dayOfMonth".toInt()

val date = LocalDate.of(2021, 12, 31) // 2021-12-31
println(date.toInt()) // 20211231
```

다만, 이렇게 하는 경우, 이하와 같이 달이나 일자가 한자리수의 것이 되어 버리는 경우도 있습니다.
```kotlin
val date = LocalDate.of(2021, 9, 1) // 2021-09-01
println(date.toInt()) // 202191
```

이 문제를 해결하려면 월과 일자를 먼저 두 자리 문자열로 맞춰야 합니다. 예를 들면 다음과 같습니다.
```kotlin
fun LocalDate.toInt(): Int = 
    "$year${monthValue.toString().padStart(2, '0')}${dayOfMonth.toString().padStart(2, '0')}".toInt()

val date = LocalDate.of(2021, 9, 1) // 2021-09-01
println(date.toInt()) // 20210901
```

하지만 이것만으로 끝나지는 않습니다. `LocalDate`뿐 아니라 `LocalDateTime`, `YearMonth` 등 `java.time` 패키지의 다른 타입에도 비슷한 처리가 필요할 수 있기 때문입니다.
다행히 이 타입들은 공통적으로 [Temporal](https://docs.oracle.com/javase/jp/8/docs/api/java/time/temporal/Temporal.html) 계열로 다룰 수 있으니, `Temporal`에 확장 함수를 두는 방식도 생각해 볼 수 있습니다. 다만 각 타입이 표현하는 시간 범위가 다르므로, 여기서는 `toString()`으로 문자열을 만든 뒤 숫자만 추출하는 쪽이 더 범용적입니다.
```kotlin
fun Temporal.toDigit(): Long = toString().filter { it.isDigit() }.toLong()

val yearMonth = YearMonth.of(2021, 8) // 2021-08
println(yearMonth.toDigit()) // 202108
val dateTime = LocalDateTime.of(2021, 10, 2, 10, 10, 10) // 2021-10-02T10:10:10
println(dateTime.toDigit()) // 20211002101010
```

문자열에서 숫자로 바꾸는 과정은 [toInt](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/to-int.html)나 [toLong](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/to-long.html)만 써도 충분하지만, `CharSequence`를 거치면 루프가 한 번 더 생깁니다. 성능만 보면 직접 변환하는 쪽이 유리하겠지만, 이 정도 길이의 날짜 문자열이라면 크게 신경 쓸 수준은 아닐 가능성이 큽니다.
## 요소의 일부를 합산

List의 값을 하나로 집계하고 싶은 (합산 값을 내고 싶은) 경우가 있습니다. `sum`을 사용해도 좋지만, 이것은 원래 요소가 숫자가 아니면 어렵습니다. 예를 들어 요소가 다음과 같은 클래스가 되는 경우는 어떻게 하면 좋을까요.
```kotlin
data class Data(
    val name: String,
    val amount: Int,
    val price: Int
)
```

### 합산하려는 값이 하나인 경우

합산하고 싶은 값이 하나만의 경우는, `sumOf`로 합산하고 싶은 값만을 지정하면 됩니다. 다음은 `Data` 클래스의 `amount`만 합산하고 싶을 때 사용할 수 있는 방법입니다.
```kotlin
val list = listOf(Data("data1", 10, 100), Data("data2", 20, 200))
val totalAmount = list.sumOf { it.amount }
```

### 합산하려는 값이 둘 이상인 경우

여기서 `amount`뿐만 아니라 `price`도 합산하고 싶은 경우는 어떻게 하면 좋을까요. 마찬가지로 `sumOf`을 `price`에도 사용하면 구현할 수 있지만 동일한 List에 대해 루프가 두 번 발생하는 것보다 효율적이지는 않습니다. 이러한 경우에는 솔직하게 각각의 합산값을 변수로서 선언해 두고 `forEach`루프 안에서 값을 더해 가는 것이 효율이 좋을 것입니다. 예를 들면 다음과 같습니다.
```kotlin
var totalAmount = 0
var totalPrice = 0

list.forEach {
    totalAmount += it.amount
    totalPrice += it.price
}
```

다른 방법으로는 [fold](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html)를 사용할 수 있습니다. `fold`는 `reduce`와 비슷하지만 초기값을 줄 수 있다는 차이가 있습니다. 이 점을 활용하면 `Data` 리스트를 두 개의 합계값(`Pair`)으로 한 번에 접어 넣을 수도 있습니다. 예를 들면 다음과 같습니다.
```kotlin
val (totalAmount, totalPrice) = list.fold(0 to 0) { acc, value ->
    (acc.first + value.amount) to (acc.second + value.price)
}
```

`fold`를 쓸 때 합산할 값이 세 개라면 [Triple](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-triple/)도 사용할 수 있습니다. 다만 이런 방식은 합계값을 `val`로 깔끔하게 받을 수 있다는 장점이 있는 대신, 루프마다 새 인스턴스가 만들어집니다. 합산 대상이 많아질수록 비용도 커질 수 있으니 상황에 맞게 고르는 편이 좋습니다.
## 마지막으로

Java를 오래 쓰다가 Kotlin으로 넘어온 입장에서는, 지금도 무심코 Java식 코드를 쓰고 있지 않은지 자주 돌아보게 됩니다. 결국 "Kotlin다운 코드"라는 것도 한 번에 몸에 배는 것이 아니라, 언어가 권장하는 스타일을 계속 익히면서 조금씩 바뀌어 가는 것에 가깝다고 생각합니다.

그래서 앞으로도 Kotlin다운 코드를 어떻게 쓸지 계속 고민해 볼 생각입니다. 특히 이 시기에는 Java 17도 나왔기 때문에, 새 API와 Kotlin 스타일을 함께 보면서 어떤 식으로 활용할 수 있을지 더 살펴보고 싶었습니다.
