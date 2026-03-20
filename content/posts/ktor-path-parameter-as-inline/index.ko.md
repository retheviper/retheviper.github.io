---
title: "Path Parameter를 Inline Class로 다루기"
date: 2023-03-26
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - kotlin
  - ktor
  - exposed
translationKey: "posts/ktor-path-parameter-as-inline"
---

최근에는 사이드 프로젝트로 간단한 웹 애플리케이션을 만들고 있고, 서버 프레임워크로는 Ktor를 쓰고 있습니다. 그동안은 주로 Spring을 다뤄 왔기 때문에, 가끔 이렇게 다른 프레임워크로 뭔가 만들어 보는 것도 꽤 재미있습니다.

이번 글에서는 Ktor에서 Path Parameter를 받아 `Inline Class`로 다루는 방법을 정리해 보겠습니다. 아주 복잡한 기법은 아니지만, 조금만 정리해 두면 더 타입 안전하고 읽기 쉬운 코드를 만들 수 있습니다.
## 인라인 클래스란

Kotlin 1.6부터 [Inline Class](https://kotlinlang.org/docs/inline-classes.html)라는 기능이 추가되었습니다. Inline Class는 형식의 래퍼로 사용할 수 있는 기능입니다. 예를 들어, 다음과 같은 코드가 있다고 가정합니다. 플레이어가 있고, 그 플레이어마다의 성적을 표현한 것입니다.
```kotlin
// 플레이어
data class Player(val id: Int, val playerRecordId: Int)
// 플레이어 성적
data class PlayerRecord(val id: Int, val score: Int)
```

그리고 다음과 같은 방법이 있다고 가정합니다.
```kotlin
// 플레이어 성적을 기록한다
fun createPlayerRecord(val playerId: Int, val score: Int) {
    // ...
}
```

이 메서드를 호출할 때 플레이어의 ID와 점수를 전달해야 합니다. 그러나 이 코드에서는 플레이어의 ID와 스코어가 모두 Int형으로 되어 있기 때문에 플레이어의 ID를 스코어로 전달해 버리는 등의 실수가 발생할 수 있습니다. 따라서 다음과 같이 플레이어의 ID와 점수를 래핑한 형식을 정의하고 이를 사용하면 더 안전하게 코드를 작성할 수 있습니다.
```kotlin
// 플레이어 ID
@JvmInline
value class PlayerId(val value: Int)

// 점수
@JvmInline
value class Score(val value: Int)
```

두 개의 인라인 클래스를 정의하면 이전 함수를 다음과 같이 수정할 수 있습니다. 이렇게 되면, 파라미터를 잘못 지정하면 컴파일러가 에러를 토해 주기 때문에, 보다 안전하게 코드를 쓸 수 있군요.
```kotlin
// 플레이어 성적을 기록한다
fun createPlayerRecord(val playerId: PlayerId, val score: Score) {
    // ...
}
```

여기까지만 보면 "그냥 래퍼 클래스나 `data class`, 혹은 `typealias`와 뭐가 다른가?"라는 의문이 생길 수 있습니다. 인라인 클래스는 컴파일 과정에서 가능하면 원래 값처럼 다뤄지면서도, 코드 차원에서는 별도 타입으로 취급된다는 점이 핵심입니다. 그래서 타입 안전성을 확보하면서도 성능 부담은 상대적으로 적다는 장점이 있습니다.
## Ktor에서 Path Parameter 처리

Ktor에서 Path Parameter를 받으려면 다음과 같이 작성해야합니다.
```kotlin
routing {
    get("/{id}") {
        val id = call.parameters["id"]?.toInt()
    }
}
```

Path Parameter를 얻기 위해 [ApplicationCall](https://api.ktor.io/ktor-server/ktor-server-core/io.ktor.server.application/-application-call/index.html)에서 [Parameters](https://api.ktor.io/ktor-http/io.ktor.http/-parameters/index.html)에서 `{id}`에 지정된 값을 먼저 String으로 읽게 됩니다. 그리고 `toInt()`에서 Int로 변환하고 있습니다. 이제 Path Parameter를 Int로 받아 처리에서 사용할 수 있게 됩니다.
## Path Parameter를 받는 처리 개선

`call.parameters`을 이용한 샘플은 순식간에 보면 간단한 코드이므로 별로 개선의 여지가 없을까 생각할지도 모릅니다만, 실은 이 코드에는 몇가지 문제가 있습니다. 예를 들어, Int 변환시의 에러를 고려해야 하는군요. `toInt()`에서 Int로 변환 할 때 `null` 또는 `"abc"`과 같은 문자열이 전달되면 `NumberFormatException`이 발생합니다. 또한 `toInt()`에서 Int로 변환 할 때 `Int.MAX_VALUE`을 초과하는 값이 전달되면 `NumberFormatException`이 발생합니다. 이와 같이, Path Parameter를 받을 때는 반드시 `null` 체크나 `NumberFormatException`의 체크를 실시할 필요가 있습니다.
게다가 `ApplicationCall`은 `get()`이나 `post()` 같은 라우팅 블록 안에서만 다루게 되므로, 비슷한 처리와 예외 처리를 엔드포인트마다 반복하기 쉽습니다. 라우터 안에 이런 코드가 여러 번 반복되는 것은 좋지 않으니, 아래처럼 `ApplicationCall` 확장 함수로 공통화하는 편이 낫습니다.
```kotlin
fun ApplicationCall.getIdFromPathParameter(name: String): Int {
    val parameter = parameters[name] ?: throw IllegalArgumentException("id is required")
    val id = parameter.toIntOrNull() ?: throw IllegalArgumentException("id must be integer")
    return idInt
}
```

이렇게 하면 다음과 같이 오류 처리를 공통화할 수 있습니다. try-catch로 예외를 처리하고 있습니다만, 여기는 필요에 따라서 [Status Pages](https://ktor.io/docs/status-pages.html)에 의한 에러 핸들링을 추가하는 것이 좋을 것입니다.
```kotlin
routing {
   get("/{id}") {
       try {
           val id = call.getIdFromPathParameter("id")
       } catch (e: IllegalArgumentException) {
           call.respond(HttpStatusCode.BadRequest, e.message)
       }
   }
}
```

## Path Parameter를 Inline Class로 래핑

그런데, Path Parameter로부터 Int형으로 취득하는 처리를 공통화할 수 있었으므로, 다음은 Path Parameter를 Inline Class로 랩 하는 것으로, 보다 안전하게 코드를 쓰는 것을 생각해 봅시다. 첫째, 가장 쉬운 방법은 다음과 같이 확장 함수가 반환하는 값을 인라인 클래스로 만드는 것입니다. 방금 전의 함수라면 다음과 같습니다.
```kotlin
routing {
   get("/{playerId}") {
        val id = PlayerId(call.getIdFromPathParameter("playerId"))
   }
}
```

다만, Inline Class도 우선 코드상에서는 클래스의 취급이므로, 제네릭을 사용할 수도 있습니다. 그래서 앞의 함수를 제네릭을 사용한 것으로 만드는 방법도 생각할 수 있습니다. 이미지적으로는 다음과 같습니다.
```kotlin
routing {
   get("/{playerId}") {
        val id = call.getIdFromPathParameter<PlayerId>("playerId")
   }
}
```

이렇게 하면 `getIdFromPathParameter()`의 반환값을 `PlayerId`로 변환하는 처리를 `getIdFromPathParameter()`에서 수행할 수 있다. 또, 제네릭이기 때문에, 사용할 수 있는 형태를 특정의 Interface 에 제한해, ID 에 관한 Inline Class 가 그것을 구현하는 형태로 하면 보다 안전한 코드가 될 것입니다. 그래서 우선은 아래와 같이 ID계의 공통의 Interface를 정의합니다.
```kotlin
// ID 계열의 공통 Interface
interface Id(val value: Int)

// 플레이어 ID
@JvmInline
value class PlayerId(val value: Int) : Id

// 감독 ID
@JvmInline
value class DirectorId(val value: Int) : Id
```

그리고 다음과 같이 `getIdFromPathParameter()`을 제네릭으로 하여 `Id`을 구현한 클래스만을 받을 수 있도록 합니다.
```kotlin
// ID를 가져오는 확장 함수
inline fun <reified T: Id> ApplicationCall.getIdFromPathParameter(name: String): T {
    val parameter = parameters[name] ?: throw IllegalArgumentException("id is required")
    val id = id.toIntOrNull() ?: throw IllegalArgumentException("id must be integer")
    return T::class.java.getDeclaredConstructor(Int::class.java).apply { isAccessible = true }.newInstance(id)
}
```

수정은 간단하고, 지정된 `Id`타입의 인스턴스를 작성해, Path Parameter로부터 취득한 Int값을 랩 해 돌려주는 것뿐입니다. 이것으로 `Id`을 구현하는 Inline Class만 대응한다고 하는 제한도 걸면서, 형태의 안전성도 확보할 수 있게 됩니다.
다만 하나, Path Parameter로서의 변수명을 Inline Class 쪽에 companion object로서 갖게 해 공통화할 수 있으면 좋겠습니다만, 유감스럽지만 그것은 어려운 것 같습니다. interface의 companion object는 override 할 수 없고, Inline Class는 abstract class를 구현할 수 없기 때문입니다. 그래서 다른 방법으로 Path Parameter명의 지정을 할 수 있게 하면 확장 함수의 인수를 줄여 보다 단순한 것이 된다고 하는 개선의 여지가 아직 있을 것 같습니다.
## 마지막으로

오랜만에 Ktor를 다시 만지면서 글로 정리해 보니, 작은 주제라도 한 번 언어화해 두는 일이 꽤 도움이 된다는 걸 다시 느꼈습니다. REST API 구현만 오래 하다 보면 익숙한 방식에만 머물기 쉬운데, 이런 식으로 조금 다른 방향을 시도해 보는 과정에서 새로 보이는 점이 분명히 있습니다. 블로그를 시작한 지도 꽤 됐지만, 아직도 정리해 볼 주제는 계속 남아 있습니다.
나중에는 이 패턴을 조금 더 다듬어서, 라우팅 레벨에서 더 자연스럽게 쓸 수 있는 형태까지 고민해 보고 싶습니다.
