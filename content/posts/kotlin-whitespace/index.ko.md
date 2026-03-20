---
title: "Kotlin String의 whitespace 처리 들여다보기"
date: 2021-05-08
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - string
translationKey: "posts/kotlin-whitespace"
---

Kotlin/JVM은 컴파일 결과가 JVM 바이트코드가 되기 때문에, Java로 만든 라이브러리를 그대로 활용할 수 있습니다. 그래서 Kotlin 표준 라이브러리를 들여다보면 Java 기능에 기대는 부분도 적지 않습니다.
하지만 그렇다고 해서 Kotlin을 단순히 "문법만 다른 Java"라고 볼 수는 없습니다. Kotlin은 JVM뿐 아니라 JavaScript와 네이티브 환경까지 염두에 두고 설계된 언어라서, 표준 라이브러리도 플랫폼별로 구현이 나뉘어 있습니다. 그리고 JVM 타깃이라 해도 Java API를 그대로 노출하는 방식만 쓰는 것은 아닙니다.
이 구조는 Kotlin 내부 코드에서 `expect`와 `actual` 키워드를 보면 분명하게 드러납니다. 공통 인터페이스처럼 기대 동작을 선언하고, 실제 구현은 플랫폼별로 따로 두는 방식입니다. 그래서 표면상 Java와 비슷해 보여도, 실제 동작이나 구현은 Kotlin 쪽에서 다시 설계된 경우가 꽤 있습니다.
이번에는 그 예시로 문자열의 whitespace 처리 기능을 살펴보겠습니다. 표준 라이브러리 소스를 기준으로 `isBlank()`, `trim()` 같은 메서드가 실제로 어떻게 동작하는지 정리해 보겠습니다.
## whitespace 판정

문자열이 실제로 의미 있는 값을 담고 있는지 확인할 때 가장 먼저 보는 기준 중 하나는, 이 문자열이 비어 있는지 혹은 공백뿐인지 여부입니다.
Kotlin 표준 라이브러리에는 이를 위한 메서드가 기본으로 들어 있습니다.
- `isEmpty()`
- `isBlank()`

Java 11 이후에도 같은 이름의 메서드가 있지만, Kotlin에서는 먼저 `kotlin.text` 쪽 구현을 보게 됩니다. 즉, Java API를 그냥 그대로 부른다고 생각하면 안 됩니다.
먼저 `isEmpty()`는 문자열 길이만 확인합니다. 실제 소스를 보면 판단 기준이 단순히 `length == 0`이라는 것을 알 수 있습니다.
참고로 Kotlin에서도 `String`은 `CharSequence` 계열로 다뤄지기 때문에, 이런 함수가 `String`이 아니라 `CharSequence` 확장으로 구현돼 있는 경우가 많습니다.
```kotlin
public inline fun CharSequence.isEmpty(): Boolean = length == 0
```

후자의 경우는, 캐릭터 라인에 whitespace까지 포함하고 있는지 어떤지를 판정합니다. 아래의 코드를 보면 무엇을 하고 있는지 명확할 것입니다.
```kotlin
public actual fun CharSequence.isBlank(): Boolean = length == 0 || indices.all { this[it].isWhitespace() }
```

`isBlank()`에서 호출하는 `isWhitespace()`은 다음과 같은 구현입니다.
```kotlin
public actual fun Char.isWhitespace(): Boolean = Character.isWhitespace(this) || Character.isSpaceChar(this)
```

Kotlin의 `Char.isWhitespace()`는 결국 `Character.isWhitespace()`와 `Character.isSpaceChar()`를 함께 사용해 판단합니다. 전자는 일반적인 Unicode whitespace를, 후자는 공백 문자 범주를 다룹니다. 그래서 단순히 비어 있는지만 확인하고 싶다면 `isEmpty()`를, 공백만 있는 경우까지 걸러야 한다면 `isBlank()`를 쓰는 편이 자연스럽습니다.
## whitespace 삭제

비어 있는지 여부만 확인하면 끝나는 경우도 있지만, 실제로는 앞뒤 공백을 걷어 낸 뒤 값을 써야 하는 경우가 더 많습니다.
Java에서는 이런 작업에 `trim()`과 `strip()`을 구분해서 사용합니다. 하지만 Kotlin에서는 상황이 조금 다릅니다. 보통은 `trim()`만으로도 충분합니다. 왜 그런지 구현을 따라가 보겠습니다.
```kotlin
public inline fun String.trim(): String = (this as CharSequence).trim().toString()
```

우선 `String.trim()`은 내부에서 `CharSequence.trim()`을 호출한 뒤 다시 문자열로 변환합니다. 실제 핵심 로직은 `CharSequence` 쪽에 있습니다.
```kotlin
public fun CharSequence.trim(): CharSequence = trim(Char::isWhitespace)
```

여기서는 오버로드된 다른 `trim()`에 `Char::isWhitespace`를 전달하고 있습니다. 즉, 어떤 문자를 잘라낼지 판단하는 기준을 함수로 넘기는 구조입니다. 이어서 실제 루프가 들어 있는 `trim(predicate)`를 보면 다음과 같습니다.
```kotlin
public inline fun CharSequence.trim(predicate: (Char) -> Boolean): CharSequence {
    var startIndex = 0
    var endIndex = length - 1
    var startFound = false

    while (startIndex <= endIndex) {
        val index = if (!startFound) startIndex else endIndex
        val match = predicate(this[index])

        if (!startFound) {
            if (!match)
                startFound = true
            else
                startIndex += 1
        } else {
            if (!match)
                break
            else
                endIndex -= 1
        }
    }

    return subSequence(startIndex, endIndex + 1)
}
```

실제 동작은 꽤 단순합니다. 왼쪽에서 시작해 공백이 아닌 문자를 찾고, 그다음에는 오른쪽에서 거꾸로 공백이 아닌 문자를 찾을 때까지 이동합니다. 구조는 간단하지만 꽤 효율적입니다.
그리고 이 과정에서 사용하는 기준이 `isWhitespace()`이므로, Kotlin의 `trim()`은 Unicode 공백 문자까지 충분히 고려합니다. 그래서 Kotlin/JVM에서는 Java처럼 굳이 `strip()`을 따로 찾지 않아도 되는 경우가 많습니다.
물론 상황에 따라 앞쪽만, 혹은 뒤쪽만 정리하고 싶을 수도 있습니다. 그럴 때는 아래 메서드를 쓰면 됩니다.
```kotlin
val string = "  string  "

// 왼쪽만 trim
println(string.trimStart()) // "string  "

// 오른쪽만 trim
println(string.trimEnd()) // "  string"
```

이 메서드들도 인자로 조건 함수를 받을 수 있어서, 공백이 아닌 다른 기준으로 잘라내고 싶을 때 응용할 수 있습니다.
또 앞뒤 공백이 아니라 특정 접두사나 접미사를 제거하고 싶다면 아래 메서드들이 더 잘 맞습니다.
```kotlin
val string = "--hello--"

// prefix만 제거
println(string.removePrefix("--")) // "hello--"

// suffix만 제거
println(string.removeSuffix("--")) // "--hello"

// 앞뒤를 제거
println(string.removeSurrounding("--")) // "hello"
```

### 줄바꿈 삭제

문자열 앞뒤의 개행은 `trim()`으로 충분하지만, 문자열 내부에 들어 있는 줄바꿈을 정리해야 할 때도 있습니다. 예를 들어 JSON을 로그에 한 줄로 남기고 싶거나, 아래 같은 multiline string을 한 줄 문자열로 바꾸고 싶은 경우입니다.
```kotlin
val string = """
    Hello
    World
"""
```

IntelliJ는 이런 문자열에 `trimIndent()`를 자동으로 붙여 주기도 하지만, 이 메서드는 들여쓰기만 정리할 뿐 내부 개행까지 없애 주지는 않습니다. 이런 경우에는 Kotlin이나 Java 모두 전용 메서드가 딱 있는 편은 아니라서, 직접 치환 로직을 써 주는 쪽이 보통입니다.
```kotlin
fun String.stripLine() = replace(System.lineSeparator(), " ")
```

다만 Java도 13부터 [Text Block](https://openjdk.java.net/jeps/355)을 도입했기 때문에, 앞으로는 Java 쪽 API에도 비슷한 방향의 메서드가 더 추가될 가능성이 있습니다.
## 마지막으로

처음에 언급한 `expect`와 `actual`은 [Kotlin Multiplatform](https://kotlinlang.org/docs/multiplatform.html)의 핵심 개념이기도 합니다. 같은 API를 유지하면서도 플랫폼별로 구현을 다르게 둘 수 있기 때문에, 표준 라이브러리 동작을 이해할 때도 꽤 중요한 힌트가 됩니다. Kotlin/JVM만 쓰더라도 한 번쯤 알아 둘 가치가 있습니다.
문자열 처리 쪽은 [JetBrains 공식 YouTube 영상](https://youtu.be/n4WBip822A8)에서도 간단히 다루고 있으니, Kotlin 내부 구현이 궁금하다면 같이 참고해 볼 만합니다.
그리고 앞에서 Kotlin에서는 `strip()`이 꼭 필요하지 않다고 했는데, 실제로 Kotlin 1.5.0 시점에는 `strip()`이 `deprecated` 상태였고 아래와 같은 안내가 붙어 있었습니다.
> 'strip(): String!' is deprecated. This member is not fully supported by Kotlin compiler, so it may be absent or have different signature in next major version

이런 사례를 보면 Kotlin이 Java와 같은 생태계를 공유하더라도, 세부 동작이나 권장 API는 항상 같지 않다는 점을 알 수 있습니다. 그래서 Java에서 Kotlin으로 옮겨올 때는 문법뿐 아니라 표준 라이브러리의 실제 동작까지 한 번 확인해 두는 편이 안전합니다.
