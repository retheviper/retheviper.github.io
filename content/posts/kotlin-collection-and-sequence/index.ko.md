---
title: "Sequence는 언제 유리할까"
date: 2021-06-13
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - stream
translationKey: "posts/kotlin-collection-and-sequence"
---

어떤 처리를 여러 방식으로 쓸 수 있다면, 무엇을 선택하는 것이 가장 좋은지 고민하게 됩니다. 보통은 가독성, 코드 길이, 예상되는 문제 같은 기준으로 비교해 결정합니다. 다만 방식 자체가 거의 비슷한 경우에는, 표면적인 차이보다 내부 동작을 먼저 봐야 합니다. 겉보기만으로는 차이가 잘 드러나지 않기 때문입니다.

이번 글에서는 Kotlin의 Collection 처리에서 자주 보게 되는 두 가지 방식, 즉 Collection의 함수를 직접 쓰는 경우와 Sequence로 바꿔서 처리하는 경우의 차이를 살펴보겠습니다.

## 처리 방식의 차이

Java에서는 Collection 요소를 처리하는 방법이 여러 가지 있습니다. 크게 보면 Java 8 이전의 루프 방식과 Java 8 이후의 Stream 방식으로 나눌 수 있습니다. 이 둘은 시작부터 사고방식이 다르기 때문에 코드 스타일도 많이 달라집니다. 같은 처리를 하더라도 아래처럼 모양이 완전히 달라집니다.

```java
// for 루프의 경우
List<String> filterEven() {
    List<Integer> list = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
    List<String> result = new ArrayList<>();
    for (Integer i : list) {
        if (i % 2 == 0) {
            result.add(i.toString());
            if (result.size() == 3) {
                break;
            }
        }
    }
    return result;
}

// Stream을 사용하는 경우
List<String> filterEvenStream() {
    return List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
            .stream()
            .filter(i -> i % 2 == 0)
            .map(i -> i.toString())
            .limit(3)
            .collect(Collectors.toList());
}
```

Stream 기반 코드는 여러 operation을 이어 붙이는 방식입니다. 이런 스타일은 함수형 프로그래밍 계열 언어에서 흔히 볼 수 있습니다. Kotlin도 마찬가지인데, 공식 문서에서는 부르는 이름이 조금씩 다르지만 여기서는 이런 조작 방식을 간단히 Functional function이라고 부르겠습니다.

Kotlin에는 Collection에도 이런 operation이 있고, Kotlin판 Stream처럼 쓸 수 있는 [Sequence](https://kotlinlang.org/docs/sequences.html)도 있습니다. 게다가 Java의 Stream도 그대로 쓸 수 있으니, 기능적으로는 비슷한 선택지가 세 가지나 됩니다. 그래서 같은 일을 여러 방식으로 표현할 수 있지만, 오히려 무엇을 써야 할지 고민이 생깁니다. 예를 들어 같은 처리는 Kotlin에서 다음처럼 쓸 수 있습니다.

```kotlin
// Collection의 경우
fun filterEven(): List<String> = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10).filter { it %2 == 0 }.map { it.toString() }.take(3)

// Sequence를 사용하는 경우
fun filterEvenSequence: List<String> = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10).asSequence().filter { it %2 == 0 }.map { it.toString() }.take(3).toList()

// Java Stream API를 사용하는 경우
fun filterEvenStream(): List<String> = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10).stream().filter { it %2 == 0 }.map { it.toString() }.limit(3).collect(Collectors.toList())
```

겉보기에는 차이가 크지 않습니다. 로직도 크게 바뀌지 않으니, 다음에 신경 쓰게 되는 것은 결국 성능입니다. 특히 Sequence가 더 빠르다는 이야기를 들으면, 자연스럽게 Sequence를 쓰는 편이 좋아 보입니다.

하지만 여기서 몇 가지 질문이 남습니다. 정말 Sequence가 항상 유리하다면, 왜 Collection에서 함수를 호출할 때 내부에서 자동으로 Sequence로 바꾸지 않을까요? 왜 굳이 `asSequence()`를 직접 호출해야 할까요? 이 질문은 결국 Sequence의 장점이 특정 조건에서만 나타난다는 뜻일 수 있습니다. 그래서 이번에는 성능 관점에서 Collection과 Sequence의 차이를 살펴보려 합니다.

### Lazy evaluation

Kotlin의 Sequence는 원래 Java의 Stream과 같은 이름이 될 예정이었다고 합니다. 이름만 비슷한 것이 아니라 실제 동작도 꽤 닮아 있습니다. 핵심은 [Lazy evaluation](https://ko.wikipedia.org/wiki/%E9%81%85%E5%BB%B6%E8%A9%95%E4%BE%A1)입니다. 필요한 시점까지 처리를 미루는 방식이라고 보면 됩니다.

Sequence를 쓰면 이 Lazy evaluation 덕분에 성능이 좋아진다고 자주 말합니다. 즉, 지금 당장 모든 것을 처리하지 않고, 필요한 순간에만 처리해서 Collection보다 효율적일 수 있다는 뜻입니다.

그런데 단순히 처리를 늦춘다고 해서 왜 성능이 좋아지는지는 바로 와닿지 않을 수 있습니다. 우리가 익숙한 루프는 보통 전체 요소를 순회하면서 차례대로 처리하는 방식이기 때문입니다.

그래서 Sequence가 더 빠르다고 해도, 성능은 입력 크기와 처리 방식에 따라 얼마든지 달라집니다. 무턱대고 모든 코드를 Sequence로 바꾸는 것은 위험합니다. 그렇다면 Collection과 Sequence가 실제로 어떻게 다른지, 코드와 실행 결과를 같이 보겠습니다.

#### Eager evaluation의 Collection

Collection의 함수형 처리는 Eager evaluation이라고 볼 수 있습니다. Lazy evaluation의 반대로, 필요 여부와 상관없이 먼저 결과를 만들어 둡니다. 이렇게 하면 이미 계산된 결과를 재사용할 수 있다는 장점이 있습니다.

Collection에서 함수형 함수를 쓰면, 각 단계마다 전체 요소를 한 번씩 처리하게 됩니다. 예를 들어 다음 코드를 보겠습니다. `onEach()`는 처리 흐름을 눈으로 보기 위한 것입니다.

```kotlin
listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    .filter { it %2 == 0 }
    .onEach { println("Found even: $it") }
    .map { it.toString() }
    .onEach { println("Now $it is String") }
    .take(3)
    .onEach { println("$it has taken") }
```

실행 결과는 다음과 같습니다.

```bash
Found even: 2
Found even: 4
Found even: 6
Found even: 8
Found even: 10
Now 2 is String
Now 4 is String
Now 6 is String
Now 8 is String
Now 10 is String
2 has taken
4 has taken
6 has taken
```

즉 Collection에서는 다음 순서로 동작합니다.

1. `filter`로 조건에 맞는 요소를 골라 새 List를 만든다
1. 그 List에 `map`을 적용해 또 다른 List를 만든다
1. 마지막에 `take`를 적용한다

이것을 그림으로 표현하면 다음과 같습니다.

![Kotlin List Processing](kotlin_list_processing.webp)
*출처: Kotlin 공식 문서 - [Sequences](https://kotlinlang.org/docs/sequences.html#iterable)*

##### Collection의 opreation

Collection의 내부 구현은 어떨까요. 여기서는 `map()`을 보겠습니다.

```kotlin
public inline fun <T, R> Iterable<T>.map(transform: (T) -> R): List<R> {
    return mapTo(ArrayList<R>(collectionSizeOrDefault(10)), transform)
}
```

`mapTo()`에 새로 만든 `ArrayList`와 Lambda를 넘기고 있습니다. `collectionSizeOrDefault()`는 대상이 `Collection`이면 실제 크기를 쓰고, 아니면 기본값을 쓰는 함수입니다.

```kotlin
internal fun <T> Iterable<T>.collectionSizeOrDefault(default: Int): Int = if (this is Collection<*>) this.size else default
```

`mapTo()`는 원래 Collection을 순회하면서 결과를 새 List에 추가합니다.

```kotlin
public inline fun <T, R, C : MutableCollection<in R>> Iterable<T>.mapTo(destination: C, transform: (T) -> R): C {
    for (item in this)
        destination.add(transform(item))
    return destination
}
```

즉, 함수형 함수를 하나 호출할 때마다 리스트를 한 번 더 만들고, 또 한 번 더 순회하게 됩니다. 위 예시에서는 루프가 여러 번 일어나고 리스트도 계속 새로 만들어집니다. `onEach()`를 빼더라도 순회는 여전히 여러 번입니다.

이 차이 때문에 Sequence가 더 빠르다는 말은, 중간 결과 리스트를 줄이고 순회 횟수도 줄일 수 있다는 뜻에 가깝습니다. 그렇다면 Sequence는 실제로 어떻게 동작할까요.

#### Lazy evaluation의 Sequence

Collection은 `asSequence()`를 호출해 손쉽게 Sequence로 바꿀 수 있습니다. 다만 실제 실행은 Java Stream처럼 종단 처리가 있어야 일어납니다. 이것이 바로 Lazy evaluation입니다.

```kotlin
listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    .asSequence() // Sequence로 변환
    .filter { it %2 == 0 }
    .onEach { println("Found even: $it") }
    .map { it.toString() }
    .onEach { println("Now $it is String") }
    .take(3)
    .onEach { println("$it has taken") }
    .toList() // Collection으로 다시 변환한다(종단 처리에서 실행)
```

실행 결과는 다음과 같습니다. Collection과 결과는 같지만, 처리 순서는 다릅니다.

```bash
Found even: 2
Now 2 is String
2 has taken
Found even: 4
Now 4 is String
4 has taken
Found even: 6
Now 6 is String
6 has taken
```

여기서 중요한 점은 8과 10은 아예 처리되지 않았다는 것입니다. Collection에서는 단계별로 전체를 다 처리한 뒤 다음 단계로 넘어가지만, Sequence는 한 요소에 대해 필요한 모든 처리를 끝내고 나서 다음 요소로 넘어갑니다.

정리하면 다음과 같습니다.

1. 요소 하나에 `filter`를 적용한다
1. 조건에 맞으면 다음 처리로 넘어간다
1. 그 요소에 `map`을 적용한다
1. `take`까지 처리한다
1. 다음 요소로 이동한다

이것을 그림으로 보면 다음과 같습니다.

![Kotlin Sequence Processing](kotlin_sequence_processing.webp)
*출처: Kotlin 공식 문서 - [Sequences](https://kotlinlang.org/docs/sequences.html#sequence)*

구조가 다르니 구현도 다를 수밖에 없습니다. 이제 `map()` 구현을 보겠습니다.

##### Sequence에서의 operation

Sequence의 `map()`은 중간 처리이며, 새 Collection을 만들지 않습니다.

```kotlin
public fun <T, R> Sequence<T>.map(transform: (T) -> R): Sequence<R> {
    return TransformingSequence(this, transform)
}
```

내부에서는 `TransformingSequence`라는 새 Sequence를 만들어 반환합니다.

```kotlin
internal class TransformingSequence<T, R>
constructor(private val sequence: Sequence<T>, private val transformer: (T) -> R) : Sequence<R> {
    override fun iterator(): Iterator<R> = object : Iterator<R> {
        val iterator = sequence.iterator()
        override fun next(): R {
            return transformer(iterator.next())
        }

        override fun hasNext(): Boolean {
            return iterator.hasNext()
        }
    }

    internal fun <E> flatten(iterator: (R) -> Iterator<E>): Sequence<E> {
        return FlatteningSequence<T, R, E>(sequence, transformer, iterator)
    }
}
```

결국 Sequence는 요소 하나를 기준으로 필요한 처리를 하나씩 진행합니다. 그래서 Collection처럼 매 단계마다 전체 리스트를 다시 만드는 비용을 줄일 수 있습니다. 요소 수가 많고 operation이 많은 경우에는 Sequence가 더 유리해 보입니다.

## Stateless

Java Stream도 마찬가지지만, Sequence는 상태를 가지지 않는다는 점이 특징입니다. 여기서 상태가 없다는 것은 요소의 수나 순서 같은 정보를 직접 보관하지 않는다는 뜻입니다. Iterator 기반이기 때문입니다. 이 때문에 처리 종류에 따라서는 오히려 Collection보다 불리할 수 있습니다.

앞서 본 예시에서 Sequence의 종단 처리로 `toList()`를 호출했습니다. 이는 상태가 없는 것을 상태가 있는 형태로 바꾸는 작업입니다. 단순하게 생각하면 Mutable List를 만들고 요소를 하나씩 `add()`하면 됩니다. 실제 구현도 그렇게 되어 있습니다.

```kotlin
public fun <T> Sequence<T>.toList(): List<T> {
    return this.toMutableList().optimizeReadOnlyList()
}
```

먼저 Mutable List로 만든 뒤, 읽기 전용 리스트로 최적화합니다.

```kotlin
public fun <T> Sequence<T>.toMutableList(): MutableList<T> {
    return toCollection(ArrayList<T>())
}
```

`ArrayList`를 만든 뒤 `toCollection()`에 넘깁니다.

```kotlin
public fun <T, C : MutableCollection<in T>> Sequence<T>.toCollection(destination: C): C {
    for (item in this) {
        destination.add(item)
    }
    return destination
}
```

즉 Sequence의 요소를 하나씩 List에 넣고 있는 셈입니다. 단순하지만, 여기서 중요한 건 "List에 요소를 더해 간다"는 사실입니다.

Sequence는 자신의 크기를 미리 알 수 없기 때문에 List를 만들 때도 크기를 추측해야 합니다. 그리고 Mutable List는 필요할 때마다 더 큰 배열을 새로 만들고 기존 값을 복사하는데, 요소가 많아질수록 이 비용이 커집니다.

복사가 끝난 뒤에도 배열 크기가 실제 요소 수보다 클 수 있습니다. 그럴 경우 메모리도 낭비되고, 읽기 전용 리스트로 정리하는 과정도 필요합니다. `optimizeReadOnlyList()`가 마지막에 호출되는 이유가 바로 그것입니다.

```kotlin
internal fun <T> List<T>.optimizeReadOnlyList() = when (size) {
    0 -> emptyList()
    1 -> listOf(this[0])
    else -> this
}
```

이런 이유로 Sequence를 사용해 처리한 뒤 다시 Collection으로 모으면, 요소가 많을수록 오히려 성능이 떨어질 가능성이 있습니다. 반대로 Collection은 이미 크기를 알고 있으므로 이런 추가 비용이 적습니다. 따라서 어떤 방식을 고를지는 함수형 함수를 몇 번 쓰는지뿐 아니라 요소 수까지 같이 봐야 합니다.

다만 요소 수가 많더라도, 종단 처리가 단순하면 Sequence가 유리할 수도 있습니다. 예를 들어 `forEach()`나 `onEach()`처럼 요소별로만 처리하는 경우입니다.

상태가 필요한 연산도 있습니다. 예를 들면 다음과 같습니다.

- 어떤 요소가 포함되어 있는지 알아야 하는 경우
  - `distinct()`
  - `average()`
  - `min()`
  - `max()`
  - `take()`
- 요소의 순서를 알아야 하는 경우
  - `indexOf()`
  - `mapIndexed()`
  - `flatMapIndexed()`
  - `elementAt()`
  - `filterIndexed()`
  - `foldIndexed()`
  - `forEachIndexed()`
  - `reduceIndexed()`
  - `scanIndexed()`

이런 처리는 Sequence에서 어떻게 동작할까요. 예로 `sorted()`를 보겠습니다.

```kotlin
public fun <T : Comparable<T>> Sequence<T>.sorted(): Sequence<T> {
    return object : Sequence<T> {
        override fun iterator(): Iterator<T> {
            val sortedList = this@sorted.toMutableList()
            sortedList.sort()
            return sortedList.iterator()
        }
    }
}
```

Sequence를 한 번 List로 바꾸고 정렬한 뒤, 다시 Sequence로 되돌려 반환합니다. 결국 `toMutableList()`를 호출하는 것이므로, 사실상 `toList()`와 비슷한 비용이 듭니다. 따라서 상태가 필요한 연산은 요소 수가 많을수록 Collection보다 불리해질 수 있습니다.

반대로 상태가 필요 없는 경우라면, 중간 결과 리스트를 만들지 않는 Sequence가 여전히 유리합니다.

## 마지막으로

정리하면, Collection과 Sequence 중 무엇을 고를지는 "어떤 처리를 하는가"에 따라 달라집니다.

| 조건 | 추천 |
|---|---|
| 처리가 복잡하다 | Sequence |
| 처리 결과로 Collection이 필요하다 | Collection |
| 반복만 한다 | Sequence |
| 처리에 상태가 필요하다 | Collection |
| 요소 수가 많다 | Sequence |
| 요소 수가 적다 | Collection |

물론 실제 상황에서는 조건이 여러 개 겹치기도 합니다. 그래서 결국은 필요한 처리가 무엇인지 잘 보고 고르는 수밖에 없습니다. 많은 경우에는 일단 Collection으로 시작해도 큰 문제는 없습니다.

이번에는 Kotlin의 Sequence를 살펴봤습니다. 더 깊게 이해하고 싶다면 Sequence 사용 시점을 잘 정리한 [이 글](https://typealias.com/guides/when-to-use-sequences)도 참고할 만합니다. 또 Java Stream은 `parallelStream()`을 쓸 수 있다는 점에서 Sequence와는 또 다른 선택지가 됩니다. 병렬 처리가 유리한 경우라면 Stream까지 함께 검토하는 편이 좋습니다.
