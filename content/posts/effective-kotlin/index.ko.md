---
title: "Effective Kotlin 읽기"
date: 2022-02-20
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - design pattern
translationKey: "posts/effective-kotlin"
---

오랜만에 책을 읽고, 그 감상을 간단히 남겨 보려고 합니다. 이전에는 주로 Java를 다뤘기 때문에 `Effective Java`를 읽으면서 제 코드를 자주 돌아보곤 했습니다. 지금은 Kotlin을 쓰고 있지만, 여전히 JVM 위에서 돌아가고 Spring도 함께 쓰고 있으니 기본적인 관점은 크게 다르지 않다고 생각했습니다. 다만 Kotlin을 1년 정도 써 보니, 언어가 달라지면 좋은 코드의 기준도 한 번쯤 다시 점검할 필요가 있겠다는 생각이 들었습니다.

그래서 `Effective Kotlin`을 읽어 봤고, 이번 글에서는 그 인상을 간단히 정리해 보려 합니다.

이 책은 나온 지 조금 지나서 인터넷에도 요약이나 발표 자료가 제법 보입니다. 예를 들어 가독성 챕터는 다른 블로그에도 잘 정리된 글이 있으니 함께 참고해도 좋겠습니다.

## 전반적인 인상

개인적으로 `Effective Java`는 어느 정도 실무 경험이 있는 사람에게 더 잘 맞는 책이라고 느꼈습니다. 예를 들어 `try-finally` 대신 `try-with-resources`를 써야 하는 이유나, `Stream`에서 부작용 없는 함수를 써야 하는 이유 같은 내용은 Java 언어 설계에 대한 배경지식이 어느 정도 있어야 더 잘 와닿습니다.

반면 `Effective Kotlin`은 초보자나 주니어 개발자에게도 훨씬 열려 있는 책입니다. 객체지향이 무엇인지부터 짚고 넘어가는 부분도 있고, 전반적으로는 깊은 원리 설명보다는 "실제로 이렇게 쓰는 편이 좋다"는 베스트 프랙티스 중심으로 구성돼 있습니다.

물론 Kotlin에서도 그대로 유효한 내용은 `Effective Java`와 겹칩니다. 예를 들어 객체 생성 시 팩터리 메서드를 고려하라는 조언 같은 것들이 그렇습니다.

다만 Kotlin의 변화 속도를 책이 완전히 따라가지는 못하는 부분도 있었고, 아주 숙련된 개발자에게는 다소 가볍게 느껴질 수도 있겠습니다. 전체적으로는 주니어부터 중급 초입 정도의 독자에게 더 잘 맞는 책이라는 인상이었습니다.

## 흥미로웠던 점

주니어에게 더 잘 맞는 책이라고는 했지만, 저 역시 아직 배우는 입장에 가깝기 때문에 흥미롭게 읽힌 부분이 많았습니다. 여기서는 그중 몇 가지를 간단히 소개해 보겠습니다.

### Single responsibility principle

이른바 [SOLID](https://ko.wikipedia.org/wiki/SOLID)에 접하는 파트입니다. Kotlin에서는 확장 함수를 사용하여 단일 책임의 원칙을 지킬 수 있다고 주장했습니다. 우선 다음과 같은 경우가 있다고 합시다.

```kotlin
class Student {
    // ...

    fun isPassing(): Boolean = 
        calculatePointsFromPassedCourses() > 15

    fun qualifiesForScholarship(): Boolean = 
        calculatePointsFromPassedCourses() > 30

    private fun calculatePointsFromPassedCourses(): Int {
        //...
    }
}
```

여기서 `isPassing()`은 `accreditations` 모듈에서, `qualifiesForScholarship()`은 `scholarship` 모듈에서 사용된다고 가정합니다. 그렇다면 `Student` 클래스가 이런 함수를 모두 직접 들고 있는 것이 단일 책임 원칙에 맞는가 하는 문제가 생깁니다.

그래서 책에서는 모듈별로 이런 함수를 확장 함수로 분리하는 편이 더 낫다고 설명합니다.

```kotlin
// scholarship module
fun Student.qualifiesForScholarship(): Boolean {
    /*...*/
}

// accreditations module
fun Student.calculatePointsFromPassedCourses(): Boolean {
    /*...*/
}
```

물론 `calculatePointsFromPassedCourses()` 자체를 클래스 밖으로 빼는 방법도 생각할 수 있습니다. 다만 그렇게 하면 이 두 메서드 전용의 private helper로 둘 수는 없습니다. 그래서

1. 모든 모듈에서 사용할 수 있는 공통 함수를 만들어 둡니다.
2. department별로 helper 함수를 만들어 둔다

같은 방법도 생각할 수 있습니다.

확장 함수의 장점은 인터페이스 구현이나 상속 없이도 기존 타입에 처리를 자연스럽게 추가할 수 있다는 점인데, 이런 식으로 쓰면 유스케이스별로 책임을 나누기 좋다는 생각이 들었습니다. 특히 확장 함수를 어디에 둘지, 가시성을 어떻게 제한할지까지 설계 포인트가 된다는 점이 개인적으로는 새롭게 느껴졌습니다.

### Consider defining a DSL for complex object creation

객체를 만들 때 복잡한 처리는 DSL을 사용합시다는 부분입니다. Kotlin에서 이미 제공한 예라면 HTML이 있네요. 다음과 같은 형태로 정의하게 됩니다.

```kotlin
body {
    div {
        a("https://kotlinlang.org") {
            target = ATarget.blank
            +"google"
        }
    }
    +"Some content"
}
```

실제로 Ktor 같은 프레임워크에서도 자주 쓰이는 방식이라서 수요가 있는 이유는 충분히 이해할 수 있었습니다. Kotlin은 고차 함수를 다루기 어렵지 않으니, 언어 차원에서도 DSL을 설계하기 좋은 편입니다.

다만 DSL 특유의 사용 규칙을 팀 안에서 합의하고, 그 규칙을 계속 유지하는 일은 생각보다 어렵겠다는 느낌도 들었습니다. 처음 설계할 때의 부담도 있고, 유지보수까지 생각하면 모든 프로젝트에 쉽게 권할 수 있는 접근은 아니라는 생각도 들었습니다.

그래도 언젠가 라이브러리나 프레임워크를 직접 만들게 된다면 한 번쯤 도전해 보고 싶은 주제입니다.

## 이미 공감하고 있던 부분

읽다 보면 "막연히 맞다고 생각하고 있었지만, 이유를 딱 정리해 본 적은 없던 내용"을 잘 문장으로 풀어 준 부분도 있었습니다. 어디선가 듣고 습관처럼 따르고 있던 원칙을 다시 확인하는 느낌이어서, 개인적으로는 이런 대목도 꽤 좋았습니다.

### Do not repeat common algorithms

표준 라이브러리로 해결할 수 있는 일반적인 알고리즘을 굳이 직접 구현하지 말라는 내용입니다. 이유는 대략 다음과 같습니다.

1. 호출하는 것이 코드를 작성하는 것보다 시간이 짧습니다.
2. 이름만 봐도 의도가 잘 드러납니다.
3. 코드를 이해하기 쉽습니다.
4. 이미 충분히 최적화되어 있는 경우가 많습니다.

저 역시 가능하면 표준 라이브러리를 적극적으로 쓰는 편이 좋다고 생각해 왔기 때문에, 이 부분은 바로 공감할 수 있었습니다. 직접 작성한 로직이 정말 잘 최적화됐는지 확신하기도 어렵고, 굳이 업무 핵심이 아닌 부분까지 손수 구현하는 일은 피하고 싶기 때문입니다.

이 책에서는 다음과 같은 코드를 올리고 있습니다. 자신의 로직을 쓴 경우입니다.

```kotlin
val percent = when {
    number > 100 -> 100
    number < 0 -> 0
    else -> number
}
```

위 코드는 [coerceIn()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/coerce-in.html)으로 더 간단하게 바꿀 수 있습니다.

```kotlin
val percent = number.coerceIn(0, 100)
```

Kotlin은 특히 표준 라이브러리에 유용한 함수가 많아서, 직접 로직을 쓰기 전에 먼저 어떤 API가 있는지 확인하는 편이 낫다고 느낀 적이 많았습니다. 이 책은 그런 판단에 왜 의미가 있는지도 잘 설명해 줘서 좋았습니다.

### Implementing your own utils

표준 라이브러리로 해결되지 않는 프로젝트 공통 처리는 유틸리티 함수로 만들어 두자는 내용입니다. 특히 클래스를 하나 더 만드는 대신 확장 함수로 만들면 아래 같은 장점이 있다고 설명합니다.

- 함수는 상태가 없기 때문에 부작용이 없다.
- 최상위 함수보다 어느 타입에 붙는 기능인지 더 분명합니다.
- 인자로 넘기는 것보다 타입에 붙은 형태가 더 직관적입니다.
- 객체에 함수를 결합하는 것보다 필요한 기능을 찾기 쉽습니다.
- 특정 타입에 묶여 있어서 어디에 둬야 하는지 고민이 줄어듭니다.

Java를 쓸 때는 저도 Singleton 패턴 유틸리티 클래스나 static 메서드를 자주 만들었습니다. Kotlin에서는 그런 클래스를 만들지 않고도 특정 타입에 함수를 자연스럽게 붙일 수 있어서 훨씬 편합니다.

예를 들어, 같은 일을 해도, 확장 함수로 쓰는 경우와 유틸리티 클래스를 만드는 경우의 코드는 이하와 같은 차이가 있습니다.

```kotlin
// 확장 함수를 사용하는 경우
val isEmptyByExtension = "Text".isEmpty()

// 유틸리티 클래스를 사용하는 경우
val isEmptyByUtilClass = TextUtils.isEmpty("Text")
```

유틸리티 클래스를 쓰면 먼저 "어느 유틸리티 클래스에 이 함수가 들어 있지?"를 떠올려야 합니다. 반면 확장 함수는 IDE 자동완성만으로도 바로 찾기 쉬워서 훨씬 직관적입니다.

게다가 특정 타입에만 붙일 수 있으니 사용 범위를 더 안전하게 제한할 수 있다는 점도 좋습니다. 확장 함수가 왜 자주 쓰이는지 다시 한 번 납득하게 된 부분이었습니다.

### 빌더 패턴

Kotlin에서는 [named arguments](https://kotlinlang.org/docs/functions.html#named-arguments)를 쓸 수 있으니 Builder 패턴이 굳이 필요 없다는 내용입니다. 물론 Kotlin에서도 Builder를 만들 수는 있지만, named parameter 쪽이 더 나은 이유는 다음과 같습니다.

- 더 짧습니다.
- 더 읽기 쉽습니다.
- 사용법이 단순합니다.
- 스레드 안전성 측면에서도 유리합니다.

저 역시 Java에서 익숙하게 쓰던 패턴이라 Kotlin에서도 Builder가 필요할 거라고 생각한 적이 있습니다. 하지만 지금은 대부분 필요 없다고 보는 쪽입니다. 위 이유들도 타당하지만, 무엇보다 Builder는 인스턴스를 만들 때 필수 파라미터가 다 채워졌는지 보장하기가 생각보다 까다롭습니다.

예를 들어 책에 나오는 빌더 패턴의 예가 있다고 가정 해 봅시다.

```kotlin
class Pizza private constructor(
    val size: String,
    val cheese: Int,
    val olives: Int,
    val bacon: Int
) {
    class Builder(private val size: String) {
        private var cheese: Int = 0
        private var olives: Int = 0
        private var bacon: Int = 0

        fun setCheese(value: Int): Builder = apply { cheese = value }

        fun setOlives(value: Int): Builder = apply { olives = value }

        fun setBacon(value: Int): Builder = apply { bacon = value }

        fun build() = Pizza(size, cheese, olives, bacon)
    }
}
```

이 빌더는 다음과 같이 사용할 수 있다고 생각합니다.

```kotlin
val villagePizza = Pizza.Builder("L")
        .setCheese(1)
        .setOlives(2)
        .setBacon(3)
        .build()
```

그런데 아래처럼 필수 값이 빠진 상태로도 빌드는 됩니다.

```kotlin
val villagePizza = Pizza.Builder("L").build()
```

만약 `cheese`, `olives`, `bacon`이 `0`을 허용하지 않는 설계라면 이 구조는 금방 애매해집니다. 기본값을 둘지, `null`을 허용할지, 강제로 체크할지 고민이 늘어나고 코드도 점점 복잡해집니다.

반면 named parameter를 사용하면 이런 문제를 훨씬 단순하게 풀 수 있습니다. 기본값이 없는 `val`이라면 필수 항목이라는 점도 자연스럽게 드러납니다.

```kotlin
val myFavorite = Pizza(
            size = "L",
            cheese = 5,
            olives = 5,
            bacon = 5
        )
```

### Consider factory functions instead of constructors

Java도 최근에는 여러 factory function을 통해 immutable 객체를 만들기 쉬워졌습니다. Kotlin 역시 생성자와 named parameter만으로 꽤 편리하게 객체를 만들 수 있지만, 그래도 factory function이 더 좋은 경우가 있다는 내용입니다. 이유는 다음과 같습니다.

- 함수에는 이름이 있으므로 객체가 어떻게 생성되는지 알 수 있습니다.
  - `ArrayList(3)`보다 `ArrayList.withSize(3)`을 이해하기 쉽습니다.
- 반환 값으로 하위 유형의 객체를 지정할 수 있습니다.
  - 상황에 따라 다른 구현을 돌려줄 수 있습니다.
- 호출될 때마다 새 객체를 만들 필요는 없습니다.
  - `Connections.createOrNull()`처럼 null을 반환할 수도 있습니다.
- 아직 실제로 존재하지 않는 객체를 표현할 수도 있습니다.
  - 프록시나 지연 생성 같은 패턴에도 응용할 수 있습니다.
- 객체 바깥에 두면서 가시성을 제어할 수 있습니다.
- `inline`으로 만들 수 있어서 [reified](https://kotlinlang.org/docs/inline-functions.html#reified-type-parameters)와도 잘 맞습니다.
- 복잡한 객체 생성 과정을 감추기 좋습니다.
- 슈퍼 클래스나 기본 생성자를 호출하지 않고 인스턴스를 생성할 수 있습니다.

이 부분도 읽으면서 꽤 납득했습니다. 저 역시 Service 계층 DTO와 Controller 응답 객체 사이 매핑에 factory function을 도입해 재사용성을 높였던 경험이 있어서, 그 방향이 괜찮았다고 다시 확인하게 됐습니다.

또 factory function을 어디에 둘 수 있는지에 대한 여러 방식도 함께 소개됩니다. 보통은 `companion object` 안에 두는 경우가 많겠지만, 상황에 따라 다른 선택지도 충분히 고려할 수 있어 보였습니다.

#### companion object

Java의 static 메서드와 같은 패턴. 가장 이해하기 쉽습니다. 다음과 같은 형태입니다.

```kotlin
class MyLinkedList<T>(
    val head: T,
    val tail: MyLinkedList<T>?
) {
    companion object {
        fun <T> of(vararg elements: T): MyLinkedList<T>? {
            /*...*/
        }
    }
}

// Usage
val list = MyLinkedList.of(1, 2)
```

factory function은 대체로 아래 같은 규칙으로 이름을 짓는다고 설명합니다.

##### from

하나의 인자를 받아 타입을 변환할 때 사용합니다.

```kotlin
val date: Date = Date.from(instant)
```

##### of

여러 인자를 받아 하나의 타입으로 묶을 때 사용합니다.

```kotlin
val faceCards: Set<Rank> = EnumSet.of(JACK, QUEEN, KING)
```

##### valueOf

`of`와 비슷한 의미로 쓰는 이름입니다.

```kotlin
val prime: BigInteger = BigInteger.valueOf(Integer.MAX_VALUE)
```

##### instance/getInstance

싱글턴이나 캐시된 인스턴스를 가져올 때 자주 쓰입니다. 같은 인자라면 같은 인스턴스를 돌려줄 수도 있습니다.

```kotlin
val luke: StackWalker = StackWalker.getInstance(options)
```

##### createInstance/newInstance

`instance` / `getInstance`와 비슷하지만, 이쪽은 보통 항상 새 인스턴스를 반환합니다.

```kotlin
val newArray = Array.newInstance(classObject, arrayLen)
```

##### getType

`instance` / `getInstance`와 비슷하지만, 다른 타입의 인스턴스를 돌려줄 때 쓰입니다.

```kotlin
val fs: FileStore = Files.getFileStore(path)
```

##### newType

`createInstance` / `newInstance`와 비슷하지만, 다른 타입의 인스턴스를 돌려줄 때 쓰입니다.

```kotlin
val br: BufferedReader = Files.newBufferedReader(path)
```

#### 확장

클래스에 `companion object`를 두고, 바깥에서 확장 함수로 factory function을 붙이는 방식입니다. 원래 클래스를 크게 건드리지 않아도 되고, 패키지 분리나 가시성 제어 같은 확장 함수의 장점도 함께 활용할 수 있습니다.

```kotlin
interface Tool {
    companion object { /*...*/ }
}

fun Tool.Companion.createBigTool( /*...*/ ): BigTool {
    //...
}
```

#### top-level

표준 라이브러리에 포함된 [listOf()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/list-of.html), [setOf()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/set-of.html), [mapOf()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/map-of.html) 같은 형태입니다.

이 방식은 자주 쓰는 타입이라면 꽤 편리합니다. 다만 IDE 자동완성에 많이 노출되기 때문에, 이름을 모호하게 지으면 오히려 혼란을 줄 수 있다는 점도 함께 짚고 있었습니다.

#### fake constructor

PascalCase 이름으로 함수를 만들어 생성자처럼 보이게 하는 방식입니다. Kotlin 표준 라이브러리에도 이런 예가 있습니다.

```kotlin
List(4) { "User$it" } // [User0, User1, User2, User3]
```

이것은 실제로는 아래와 같은 함수입니다.

```kotlin
public inline fun <T> List(
    size: Int,
    init: (index: Int) -> T
): List<T> = MutableList(size, init)

public inline fun <T> MutableList(
    size: Int,
    init: (index: Int) -> T
): MutableList<T> {
    val list = ArrayList<T>(size)
    repeat(size) { index -> list.add(init(index)) }
    return list
}
```

이 방식은 인터페이스에 대해 생성자처럼 보이는 진입점을 만들고 싶을 때나, `reified` 타입 파라미터를 활용하고 싶을 때 고려할 수 있습니다.

그 외에도 fake constructor을 만드는 방법이 있습니다.

```kotlin
class Tree<T> {

    companion object {
        operator fun <T> invoke(size: Int, generator: (Int) -> T): Tree<T> {
            // ...
        }
    }
}

// Usage
Tree(10) { "$it" }
```

다만 이 경우 constructor reference 쪽 문법은 오히려 더 복잡해질 수 있습니다.

```kotlin
// Constructor
val f: () -> Tree = ::Tree

// Fake Constructor
val d: () -> Tree = ::Tree

// Invoke in companion object
val g: () -> Tree = Tree.Companion::invoke
```

그래서 fake constructor를 쓴다면, 결국은 일반 함수 형태가 더 읽기 쉬울 수도 있겠다는 생각이 들었습니다.

#### factory class

별도 Factory 클래스를 두고 인스턴스를 돌려주는 방식도 있습니다. Java에서는 이런 접근이 익숙해서 Kotlin에서도 가능할까 싶었는데, 책에서는 "factory 클래스는 상태를 가질 수 있다"는 점을 장점으로 들고 있었습니다. 실제로 상황에 따라서는 꽤 쓸모가 있을 것 같았습니다.

```kotlin
data class Student(
    val id: Int,
    val name: String,
    val surname: String
)

class StudentsFactory {
    var nextId = 0
    fun next(name: String, surname: String) =
        Student(nextId++, name, surname)
}

val factory = StudentsFactory()
val s1 = factory.next("Marcin", "Moskala")
println(s1) // Student(id=0, name=Marcin, Surname=Moskala)
val s2 = factory.next("Igor", "Wojda")
println(s2) // Student(id=1, name=Igor, Surname=Wojda)
```

## 마지막으로

거칠게 정리하면, 이 책은 Kotlin 입문 이후 한 단계 더 나아가고 싶은 사람에게 꽤 괜찮은 안내서였습니다. 새로운 발견도 있었고, 평소 막연하게 좋다고 느끼던 습관들을 조금 더 명확한 언어로 확인할 수 있었다는 점도 좋았습니다.

다만 Kotlin이 아직 비교적 새로운 언어이고 여러 패러다임을 계속 흡수하는 중이어서인지, `Effective Java`처럼 아주 깊은 수준의 설계 논의까지는 다소 부족하다는 인상도 있었습니다. 그래도 그만큼 Kotlin이라는 언어 자체가 아직 더 발전할 여지가 많다는 뜻으로도 볼 수 있겠습니다.
