---
title: "Kotlin은 어떻게 써야 할까"
date: 2022-12-30
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
translationKey: "posts/kotlin-how-to-kotlin-like"
---

제가 Java에서 Kotlin으로 넘어온 지도 벌써 2년 정도가 됐습니다. 그런데도 아직 Kotlin에서 할 수 있는 일은 무궁무진하고, 새로운 발견이 끊이지 않는다는 느낌이 듭니다. Kotlin 자체의 버전 업 속도도 빠르고 새로운 기능도 계속 추가되고 있으니, 이런 흐름은 앞으로도 한동안 계속될 것 같습니다.

IntelliJ를 쓰다 보면 Java 코드를 Kotlin으로 자동 변환해 주기도 하고, Java식 작성법을 그대로 가져와도 당장 큰 문제가 생기지 않는 경우가 많습니다. 하지만 그래도 결국은 "Kotlin답게 쓰고 싶다"는 생각이 들게 됩니다. 즉, Kotlin만의 장점을 살린 작성 방식을 고민하게 된다는 뜻입니다.

## Kotlin다운 작성 방식이란

"Kotlin답다"는 게 정확히 무엇인지는 사람마다 다르게 느낄 수 있습니다. 저는 기본적으로 Kotlin의 문법과 기능을 최대한 잘 활용하는 방향이라고 생각합니다. 표준 라이브러리 API, scope function, extension function 같은 요소를 적극적으로 활용하는 것이죠. 그렇게 하면 코드를 더 짧게 쓸 수 있고, 결과적으로 생산성도 높아질 가능성이 큽니다.

다만 이런 이야기를 개념적으로만 하면 다소 추상적이니, 예시 코드로 보는 편이 이해하기 쉽습니다. 예를 들어 아래와 같은 함수를 구현해야 한다고 해 보겠습니다.

```kotlin
fun numbers(): List<String> {
    // TODO
}
```

이 함수가 해야 할 일은 "0부터 10까지의 숫자를 문자열로 바꿔 리스트로 반환한다"는 것이라고 하겠습니다. 가장 직관적으로는 아래처럼 작성할 수 있습니다.

```kotlin
fun numbers(): List<String> {
    val list = mutableListOf<String>()
    var i = 0
    while (i < 10) {
        i++
        list.add(i.toString())
    }
    return list.toList()
}
```

하지만 여기서 [kotlin.collections.map()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/map.html)을 사용하면 같은 내용을 훨씬 간결하게 쓸 수 있습니다.

```kotlin
fun numbers(): List<String> =
    (0..10).map { it.toString() }
```

`map()` 같은 함수가 처음 나왔을 때는 직관적이지 않다는 이유로 "가독성이 떨어진다"는 평가도 있었습니다. 실제로 어떤 사람에게는 `while` 루프 쪽이 더 익숙하고, 그래서 더 읽기 쉽다고 느껴질 수도 있습니다. 다만 지금은 `map()`이 여러 언어에서 널리 쓰이고 있고, 그 의미도 어느 정도 공통 감각이 된 상태입니다. 그래서 오히려 "무엇을 하려는 코드인가"를 짧고 분명하게 보여 주는 쪽은 `map()` 쪽이라고 볼 수도 있습니다.

## 가독성이라는 관점

그렇다면 조금 더 복잡한 처리에서도 긴 코드가 항상 더 읽기 쉬울까요? 예를 들어 "리스트 안의 숫자를 모두 곱한 결과를 반환하는 함수"를 구현한다고 해 보겠습니다.

```kotlin
fun multiply(numbers: List<Int>): Int {
    if (numbers.isEmpty()) {
        throw IllegalAccessException("List is empty")
    }
    var acc = numbers[0]
    if (numbers.size == 1) return acc
    for ((index, value) in numbers.withIndex()) {
        if (index != 0) {
            acc *= value
        }
    }
    return acc
}
```

이 코드는 분명 원하는 결과를 만들어 냅니다. 하지만 가독성 면에서 보면 어떨까요? 함수 시그니처만 봐서는 의도를 파악하기 어렵고, 실제로 무슨 일이 벌어지는지 이해하려면 구현 전체를 처음부터 끝까지 읽어야 합니다.

이 함수에는 여러 단계의 정보가 들어 있습니다. 리스트가 비어 있으면 예외를 던지고, 요소가 하나면 그대로 반환하고, 그 외에는 첫 요소를 기준으로 나머지를 순회하며 계속 곱합니다. 즉, "숫자들을 모두 곱한다"는 본래 목적보다 구현 세부사항이 더 전면에 드러납니다.

반면 extension function과 `kotlin.collections`의 함수를 활용하면 같은 의도를 훨씬 직접적으로 표현할 수 있습니다.

```kotlin
fun List<Int>.multiply(): Int =
    reduce { acc, e -> acc * e }
```

물론 `reduce()`가 무엇을 하는지 모르면 처음에는 낯설 수 있습니다. 하지만 이 함수가 "여러 요소를 하나의 값으로 줄인다"는 의미라는 것을 알고 나면, 이 코드가 하려는 일은 오히려 더 명확합니다. 게다가 `List`의 확장 함수로 정의하면, 마치 원래부터 제공되던 메서드처럼 사용할 수 있다는 점도 장점입니다.

저는 이런 경우에 오히려 짧은 코드 쪽이 더 읽기 쉽다고 생각합니다. 길이를 줄였기 때문이 아니라, 구현 세부보다 의도가 더 앞에 나오기 때문입니다.

## 공수라는 관점

코드를 작성하는 데는 시간과 비용이 듭니다. 같은 기능을 매번 처음부터 구현하기보다, 공통화할 수 있는 부분은 분리해서 재사용하는 것이 일반적입니다. 그래서 수많은 라이브러리와 프레임워크가 존재하는 것이기도 합니다.

Kotlin의 기능을 잘 이해하고 적극적으로 활용하는 것도 본질적으로는 비슷합니다. 표준 라이브러리를 잘 쓰는 것은 일종의 재사용 전략이고, 그만큼 구현 비용을 줄여 줍니다. 앞서 본 것처럼 "리스트를 하나의 값으로 축약한다"는 처리를 매번 직접 구현한다면 꽤 많은 시간이 들 것입니다.

다른 예도 보겠습니다. 아래와 같은 데이터가 있다고 가정해 봅시다.

```kotlin
data class Data(val id: Int, val version: Int)

val list = listOf(
    Data(1, 1),
    Data(1, 2),
    Data(2, 1),
    Data(2, 2),
    Data(2, 3)
)
```

DB에 이력을 남겨야 하는 경우에는 version이나 순번 같은 컬럼을 두고 같은 ID의 데이터를 여러 건 저장하는 일이 종종 있습니다. 그리고 그중 최신 버전만 추려서 처리하고 싶을 때가 있습니다. 위 예시라면 `Data(1, 2)`와 `Data(2, 3)`만 필요하다는 뜻입니다.

처음부터 쿼리에서 최신 데이터만 가져올 수 있다면 좋겠지만, 외부 API 응답처럼 정제되지 않은 데이터를 다뤄야 하는 경우도 있습니다. 그런 상황에서 직접 필터링 로직을 구현하면 대략 이런 코드가 나올 수 있습니다.

```kotlin
fun maxVersions(list: List<Data>) {
    val result = mutableListOf<Data>()
    for ((index, value) in list.withIndex()) {
        if (index == 0) {
            result.add(value)
        } else {
            val last = result[result.size - 1]
            if (value.id == last.id) {
                if (value.version > last.version) {
                    result.remove(last)
                    result.add(value)
                }
            } else {
                result.add(value)
            }
        }
    }
    return result.toList()
}
```

가독성 문제는 잠시 제쳐 두더라도, 이런 코드를 처음부터 직접 작성하는 데는 적지 않은 공수가 듭니다. 익숙한 사람이라면 금방 쓸 수도 있겠지만, 처음 구현하는 사람이라면 로직을 짜는 시간에 더해, 기대한 대로 동작하는지 검증하는 시간까지 필요할 것입니다.

반면 표준 라이브러리를 활용하면 같은 작업을 훨씬 짧게 표현할 수 있습니다.

```kotlin
val maxVersions = list.groupBy { it.id }.map { it.value.maxBy { it.version } }
```

여기서 `groupBy()`, `map()`, `maxBy()`는 각각 자주 재사용되는 의미를 가진 함수입니다. `groupBy()`는 그룹화를, `map()`은 형태 변환을, `maxBy()`는 기준값이 가장 큰 요소 선택을 담당합니다. 이런 함수들을 조합할 수 있으면 더 복잡한 처리도 비교적 짧고 안정적으로 작성할 수 있습니다.

그래서 표준 라이브러리를 이해하는 일은 단순히 "멋있게 코드를 쓰는 방법"이 아니라, 실제 구현 공수를 줄이는 데도 큰 도움이 된다고 생각합니다.

## 주의할 점

여기까지는 Kotlin의 기능을 적극적으로 활용하면 가독성과 공수 두 측면에서 이점이 있다는 이야기를 했습니다. 하지만 어떤 선택이든 장점만 있는 것은 아닙니다.

당연히, 어떤 기능이든 "쓸 수 있으니까 쓴다"는 식으로 남용하면 오히려 역효과가 날 수 있습니다. 예를 들어 아래처럼 `Data`를 `Request`로 매핑해야 하는 상황을 생각해 보겠습니다.

```kotlin
data class Data(
    val id: Int?,
    val version: Int?
)

data class Request(
    val id: Int,
    val version: Int
)
```

`Data` 쪽은 nullable이지만 `Request`는 nullable이 아닙니다. 실무에서는 비즈니스적으로는 null이 나오면 안 되더라도, 입력 구조나 외부 시스템 사정 때문에 nullable로 받을 때가 종종 있습니다. 이런 경우 아래처럼 구현할 수 있습니다.

```kotlin
data class Request(
    val id: Int,
    val version: Int
) {
    companion object {
        fun from(data: Data): Request {
            return Request(
                id = requireNotNull(data.id) {
                    "${Data::id.name} should not be null."
                },
                version = requireNotNull(data.version) {
                    "${Data::version.name} should not be null."
                }
            )
        }
    }
}
```

이 구현 자체는 꽤 자연스럽습니다. `companion object`에서 매핑을 담당하고, `requireNotNull()`로 검증도 함께 수행합니다. 다만 여기서 한 번 생각해 볼 만한 부분은 에러 메시지를 만들기 위해 `Data`의 필드명을 reflection으로 가져오는 방식입니다.

정말 이 정도를 위해 reflection을 써야 할까요? 에러 시점에만 동작한다고 해도, 꼭 그렇게까지 해야 하는지는 고민해 볼 필요가 있습니다.

이처럼 언어 기능을 잘 활용하는 것과, 기능을 아무 생각 없이 끌어다 쓰는 것은 다릅니다. `map()`, `reduce()`, `groupBy()` 같은 함수도 매우 유용하지만, 각각 내부적으로 순회를 수행하고 새로운 컬렉션을 만들 수 있다는 점을 이해하고 있어야 합니다. 메서드 체인이 길어질수록 성능에 영향을 줄 수도 있고, 오히려 코드의 의도가 흐려질 수도 있습니다. 경우에 따라서는 별도 함수로 분리하는 편이 더 좋은 설계일 수도 있습니다.

그래서 저는 이런 기능들을 "대충 알고 습관적으로 쓰는 것"에는 위험이 있다고 생각합니다. 어떤 구현이 가장 적절한지는 늘 한 번쯤 고민해 볼 필요가 있습니다.

## 마지막으로

예전에도 "Kotlin으로 써 보았다"는 느낌의 글을 몇 번 올린 적이 있고, 이번 글도 큰 틀에서는 그 연장선에 있습니다. 어떤 의미에서는 여기서 다룬 내용이 아주 새로운 이야기라기보다, 프로그래머라면 기본적으로 익혀 두어야 할 내용처럼 보일 수도 있습니다.

그럼에도 굳이 글로 남기는 이유는, 결국 기본이 가장 중요하다고 생각하기 때문입니다. 경험이 쌓이면 분명 성장하는 부분도 있지만, 반대로 좋지 않은 습관이 굳어져 쉽게 고치기 어려워지는 경우도 있습니다. 그래서 한 번쯤은 기본으로 돌아가 지금의 내 코드를 다시 바라보는 일이 필요하다고 느낍니다.

조금 늦었지만, 2022년 글은 이 글로 마무리하게 됐습니다. 내년에는 저에게도, 그리고 이 블로그를 찾아와 주시는 분들에게도 성장하는 한 해가 되면 좋겠습니다.
