---
title: "Java 프로그래머가 본 Kotlin 2"
date: 2021-02-28
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
translationKey: "posts/kotlin-basics"
---

이직을 하면서 업무 언어도 Java에서 Kotlin으로 바뀌었습니다. 개인적으로 Kotlin으로 간단한 Spring WebFlux 프로젝트를 만들어 본 적은 있었지만, 실제 업무 언어가 바뀌는 일은 또 다른 문제였습니다. Java 개발자가 Kotlin으로 넘어가는 일이 어렵지 않다고들 하지만, 막상 실무에서 본격적으로 쓰기 시작하면 생각보다 배울 것이 많습니다. 아마 앞으로는 Java보다 Kotlin 관련 글을 더 자주 쓰게 될 것 같습니다.

Kotlin은 Java와 호환되지만, 그렇다고 Java 감각만으로 쓰면 Kotlin을 쓰는 의미가 줄어듭니다. Kotlin은 단순히 Java의 장황함을 줄인 언어가 아니라, 코드를 바라보는 방식 자체를 조금 다르게 설계한 언어에 가깝습니다. 처음에는 문법만 간결해진 언어처럼 보였지만, 공부를 이어갈수록 차이가 훨씬 넓다는 걸 느꼈습니다.
이 글은 [Coursera](https://www.coursera.org)의 [Kotlin for Java Developers](https://www.coursera.org/learn/kotlin-for-java-developers) 강의를 바탕으로 정리했습니다.
## 중복을 줄이는 것

Java는 여전히 좋은 언어이고, 엔터프라이즈 환경에서도 폭넓게 쓰입니다. Java가 여전히 많이 사용된다는 사실은 [TIOBE index](https://www.tiobe.com/tiobe-index/), JetBrains의 [The State of Developer Ecosystem](https://www.jetbrains.com/lp/devecosystem-2020), Stack Overflow의 [Developer Survey](https://insights.stackoverflow.com/survey/2020)에서도 확인할 수 있습니다.
하지만 Java가 널리 쓰인다고 해서, 그 자체가 다른 언어보다 압도적으로 뛰어나거나 더 쓰기 쉽다는 뜻은 아닙니다. Java에서 자주 지적되는 문제 중 하나는 바로 중복이 많다는 점입니다. 라이브러리와 빌드 도구는 잘 갖춰져 있지만, 언어 사양 자체는 그만큼 크게 바뀌지 않았습니다. Java 9 이후로 여러 개선이 들어오긴 했지만, 기존 코드에 남아 있는 장황함을 완전히 없애지는 못했습니다. 그래서 중복 코드는 여전히 남아 있고, 앞으로도 꽤 오래 마주치게 될 것입니다.
### 코드가 짧아짐

중복을 줄인다는 것은 결국 같은 결과를 더 짧고 덜 지루한 코드로 얻는다는 뜻입니다. 이 관점에서 보면 Kotlin은 Java의 중복을 줄이기 위해 꽤 집요하게 설계된 언어처럼 보입니다. 예를 들어 다음과 같은 코드가 있다고 해봅시다.
```java
public void updateWeather(int degrees) {
    String description;
    Color color;
    if (degrees < 10) {
        description = "cold";
        color = BLUE;
    } else if (degrees < 25) {
        description = "mild";
        color = ORANGE;
    } else {
        description = "hot";
        color = RED;
    }
}
```

이것을 Kotlin으로 다시 작성하면 다음과 같습니다.
```kotlin
fun updateWeather(degrees: Int) {
    val (description, color) =
        if (degrees < 10) {
            Pair("cold", BLUE)
        } else if (degrees < 25) {
            Pair("mild", ORANGE)
        } else {
            Pair("hot", RED)
        }
}
```

먼저 두 변수를 `Pair`를 반환하는 표현식으로 묶어서 더 짧게 만들 수 있습니다. 여기에 `when`을 쓰면 코드가 한층 더 단순해집니다. 결과는 다음과 같습니다.
```kotlin
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 10 -> Pair("cold", BLUE)
        degrees < 25 -> Pair("mild", ORANGE)
        else -> Pair("hot", RED)
    }
}
```

또한 `Pair`는 `to`를 쓰면 더 자연스럽게 표현할 수 있습니다. 그러면 코드는 다음처럼 더 간결해집니다.
```kotlin
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 10 -> "cold" to BLUE
        degrees < 25 -> "mild" to ORANGE
        else -> "hot" to RED
    }
}
```

첫 번째 Java 코드와 비교하면 훨씬 읽기 쉽고, 의도도 바로 보입니다. 다른 언어를 쓰던 사람도 무슨 일을 하는지 금방 파악할 수 있고, 코드도 더 짧고 명료해집니다. 이런 점이 Kotlin이 Java의 중복과 낭비를 줄이는 데 강한 이유라고 생각합니다.
### 코드를 더 쉽게 작성할 수 있다

처음 Kotlin 문법을 봤을 때는 `switch`가 `when`으로 바뀌고 `case`를 쓰지 않아도 되는 정도로만 느꼈습니다. 그런데 조금 더 들여다보면 Java와 다른 점이 훨씬 많습니다. 조금 전 예제만 봐도 다음과 같은 특징이 있습니다.
- `when`을 표현식으로 사용할 수 있다
- 조건을 더 유연하게 적을 수 있다
- 여러 값을 한 번에 반환값으로 만들 수 있다
- `to`로 두 객체를 `Pair`로 묶을 수 있다

그 밖에도 Kotlin의 `when`은 Java의 `switch`보다 훨씬 유연합니다. 객체 비교도 쉽게 할 수 있어서, 다음처럼 두 객체를 기준으로도 간단히 분기할 수 있습니다.
```kotlin
fun mix(c1: Color, c2: Color) =
    when (setOf(c1, c2)) {
        setOf(RED, YELLOW) -> ORANGE
        setOf(YELLOW, BLUE) -> GREEN
        setOf(BLUE, VIOLET) -> INDIGO
        else -> throw Exception("Dirty Color")
    }
```

이걸 굳이 Java로 쓰면 아마 다음처럼 될 것입니다. 개인적으로는 `else if`가 많아질수록 읽기도 쓰기도 점점 답답해집니다.
```java
private Color mix(Color c1, Color c2) {
    if (c1 == Color.RED && c2 == Color.YELLOW) {
        return Color.ORANGE;
    } else if (c1 == Color.YELLOW && c2 == Color.BLUE) {
        return Color.GREEN;
    } else if (c1 == Color.BLUE && c2 == Color.VIOLET) {
        return Color.INDIGO;
    } else {
        throw new RuntimeException("Dirty Color");
    }
}
```

결국 Kotlin은 같은 일을 해도 더 짧고 더 직접적으로 쓰게 해줍니다. 물론 Java에서도 다른 메서드를 만들거나 `Comparable`, `Comparator`를 활용하면 비슷한 결과를 낼 수는 있습니다. 하지만 그만큼의 수고를 들여야 한다는 점은 분명합니다.
물론 Java 12 이후에는 Kotlin의 `when`에 가까운 방식으로도 코드를 쓸 수 있습니다. 표현식으로 쓸 수 있고, 여러 조건을 한 번에 다룰 수 있다는 점도 좋아졌습니다.
```java
var result = switch (month) {
    case JANUARY, JUNE, JULY -> 3;
    case FEBRUARY, SEPTEMBER, OCTOBER, NOVEMBER, DECEMBER -> 1;
    case MARCH, MAY, APRIL, AUGUST -> {
        int monthLength = month.toString().length();
        yield monthLength * 4;
    }
    default -> 0;
};
```

이런 변화를 보면, "중복을 줄인다"는 면에서는 Java도 버전업을 통해 꾸준히 따라오고 있습니다. 그래서 Kotlin의 매력이 줄어든 것처럼 보일 수도 있습니다. 그런데 Kotlin에는 그보다 더 중요한 장점이 하나 있습니다. 바로 언어 자체의 확장성입니다.
## 확장이 가능하다는 것

언어의 확장성이라고 하면 결국 확장 함수, 즉 `extension`을 말합니다. Kotlin에서는 이 기능이 꽤 중요한 축으로 다뤄집니다. 잘 쓰면 단순히 "기존 클래스에 메서드를 덧붙인다" 수준을 넘어서 `infix`와 조합해 훨씬 읽기 좋은 문법처럼 쓸 수 있습니다.
실제로 [Kotlin for Java Developers](https://www.coursera.org/learn/kotlin-for-java-developers)의 과제에서도 `infix`로 만든 확장 함수를 써서 결과를 확인하게 되어 있습니다.
```kotlin
infix fun <T> T.eq(other: T) {
    if (this == other) println("OK")
    else println("Error: $this != $other")
}
```

이 `infix`을 사용하면 다음과 같은 코드를 작성할 수 있습니다.
```kotlin
"ABC" eq "ABC"
```

이런 특징은 사용하는 쪽에서도 편하지만, 언어가 성장하면서 더 편한 기능이 계속 추가될 수 있다는 점이 더 큽니다. 예를 들어 앞서 본 `to`도 `infix` 함수의 형태로 제공됩니다. 이런 방식이 쌓이면 개발 비용을 줄이는 데에도 분명 도움이 됩니다.
## 이미 충분히 유용한 것

중복을 줄이고 확장하기 쉬운 언어라는 점은 Kotlin을 만드는 JetBrains 입장에서도 강점일 것입니다. 표준 라이브러리만 봐도 이미 유용한 함수가 많습니다. 예를 들어 간단한 루프는 다음처럼 쓸 수 있습니다.
```kotlin
val list = listOf("A", "B", "C")

// 일반적인 for문
for (element in list) {
    println(element)
}

// 인덱스를 포함한 for문
for ((index, element) in list.withIndex()) {
    println("$index: $element")
}

// 인덱스만 사용하는 for문
for (index in list.indices) {
    println(index)
}
```

예전에 Java `for`문의 성능을 다룬 글에서도 비슷한 이야기를 한 적이 있습니다. Java에서는 인덱스가 필요하면 전통적인 `for`를 쓰고, 그렇지 않으면 확장 `for`를 쓰는 쪽이 일반적으로 좋다는 결론이었습니다. 그런데 Kotlin은 이런 편한 사용법을 언어 차원에서 바로 제공하니, 상황에 따라 굳이 성능과 편의 사이에서 오래 고민하지 않아도 됩니다.
그리고 `forEach`를 쓰면서도 인덱스가 필요하면 `forEachIndexed`를 쓸 수 있다는 점도 좋습니다. 예를 들면 다음과 같습니다.
```kotlin
// 일반적인 forEach문
list.forEach(::println)

// 인덱스를 포함한 forEach문
list.forEachIndexed { index, element -> println("$index: $element") }
```

인덱스를 쉽게 얻을 수 있다는 것은, 루프 대상의 전체 인덱스를 다룰 때 굳이 `0` 같은 매직 넘버를 직접 관리하지 않아도 된다는 뜻이기도 합니다. Java였다면 이런 값을 상수로 따로 빼 두는 경우가 많았을 겁니다.
그 밖에도 정규 표현식 없이 문자 판정을 할 수 있는 함수가 미리 준비되어 있고(`Char.isLetter()`, `Char.isDigit()` 등), `Map`에는 `Pair`를 그대로 넣을 수 있으며, iterable 객체에도 Stream API처럼 빠르게 접근할 수 있습니다. 확실히 Java보다 "어떻게 쓸지"를 덜 고민하게 해주는 언어입니다. 물론 그걸 단점으로 느끼는 사람도 있겠지만요.
## 마지막으로

여기까지 여러 특징과 장점을 정리했지만, 저 역시 아직 실무에서 아주 오래 써 본 단계는 아닙니다. 그래도 이 정도만으로도 Kotlin의 매력은 충분히 느껴진다고 생각합니다.

언어 자체의 완성도도 높지만, JetBrains가 만들었다는 점에서 IntelliJ와의 궁합이 좋고, JVM 언어라 Java의 발전도 그대로 흡수할 수 있으며, Native나 JavaScript로도 확장할 수 있다는 점도 큽니다. 다른 언어에도 각자의 장점은 있지만, Java 개발자라면 Kotlin은 한 번쯤 진지하게 써 볼 만한 언어입니다.
