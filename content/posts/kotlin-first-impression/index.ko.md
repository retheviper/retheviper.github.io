---
title: "Java 프로그래머가 본 Kotlin"
date: 2020-10-25
categories:
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
translationKey: "posts/kotlin-first-impression"
---

Kotlin이 Android의 공식 언어가 된 지도 꽤 오래됐지만, 웹 애플리케이션 업계에서는 여전히 서버 사이드 언어로 Java를 쓰는 곳이 많습니다. 저 역시 그런 환경에서 일하고 있고, 모바일 쪽도 아직 Java를 쓰는 사례가 적지 않은 것 같습니다. Java도 9 이후부터는 릴리스 속도를 높이면서 이른바 모던 언어의 특징을 많이 흡수하고 있지만, 언어 자체의 설계가 오래됐고 하위 호환성 때문에 예전 방식의 흔적도 쉽게 버리지 못하고 있습니다. 그런 점에서 보면, 처음부터 다른 철학으로 설계된 언어와는 확실히 결이 다릅니다. 여기에 [Kotlin Native](https://kotlinlang.org/docs/reference/native-overview.html)처럼 JVM 밖을 겨냥한 흐름까지 보면, 앞으로는 Java보다 Kotlin이 더 활약할 장면이 많아질 수도 있겠다는 생각이 들기도 합니다. ([GraalVM](https://www.graalvm.org/)은 실제로 어디까지 쓰이게 될지 궁금하네요.)

물론 저는 아직 Kotlin을 정식으로 깊게 배운 것은 아니고, Spring WebFlux로 간단한 애플리케이션을 만들면서 직접 써 본 정도입니다. 그래서 놓친 부분도 많겠지만, 지금까지 만져 보며 느낀 점을 Java 개발자 관점에서 가볍게 정리해 보겠습니다.

## 이 점은 좋았다

먼저 직접 써 보면서 좋다고 느낀 부분부터 적어 보겠습니다. 전반적으로는 "기대했던 장점이 실제로도 꽤 잘 느껴진다"는 인상이었습니다.

### 확실히 현대적인 언어 같다

Kotlin 코드를 처음 보면, "요즘 언어는 이런 느낌이구나"라는 인상을 받게 됩니다. 물론 "모던한 언어"가 정확히 무엇이냐는 별도 논의가 필요하겠지만, 적어도 Swift, Kotlin, Go 같은 언어들을 떠올리게 하는 분위기가 있습니다. 제가 여러 언어에 아주 익숙한 편은 아니지만, 이런 언어들은 어딘가 Python과도 비슷한 감각이 있습니다. `var`, `fun`처럼 짧은 키워드를 쓰고, 타입은 콜론 뒤에 붙이며, 세미콜론이 필수가 아니고, `in`, `Range`, `is` 같은 표현이 자연스럽게 들어가는 식입니다. 언어 자체의 문법은 아니지만, Getter/Setter보다 프로퍼티 접근을 더 자연스럽게 여기는 문화도 비슷한 느낌을 줍니다. 덕분에 Lombok에 덜 의존해도 된다는 점도 편리합니다.

그렇다고 Kotlin이 Java와 아주 멀리 떨어져 있는 느낌은 아닙니다. 오히려 Java를 조금 더 유연하게 다듬은 언어에 가깝다는 인상을 받았습니다. 예를 들어 Python에서는 `elif`를 쓰지만 Kotlin은 `else if`를 그대로 씁니다. JVM 언어라는 점 때문만이 아니라, 기본 문법 자체도 Java 개발자라면 비교적 빨리 적응할 수 있게 되어 있습니다. `for` 루프에 라벨을 붙일 수 있는 것도 그런 예 중 하나입니다.

Java와 비교했을 때 더 인상적이었던 부분은, 단순히 문법을 줄여 놓은 수준이 아니라 Java의 설계 자체를 다시 생각한 흔적이 보인다는 점입니다. 대표적인 예가 `null`과 변경 가능성(`mutable`)을 다루는 방식입니다. Kotlin에서는 기본적으로 변수가 `null`이 될 수 없고, `null` 가능성이 있는 값은 처음부터 명시해야 합니다. `null` 가능한 값을 다룰 때도 safe call을 강제하는 식으로 컴파일 단계에서 최대한 실수를 막아 주려는 방향이 강합니다. 컬렉션도 마찬가지로, 굳이 `Mutable`을 명시하지 않으면 기본적으로 불변 객체 쪽으로 유도됩니다. 이것만으로도 Java에서 흔히 보게 되는 NPE를 꽤 많이 줄일 수 있겠다는 느낌이 들었습니다. 물론 선언할 때마다 신경 써야 해서 번거롭다고 느낄 수는 있지만, 런타임 오류보다 컴파일 오류가 훨씬 낫다는 데는 대부분 동의할 것입니다.

또 개인적으로 Python에서 좋다고 느꼈던 기능이 Kotlin에도 있어서 반가웠습니다. 예를 들면 Multiple Return(복수 반환값)이나 Named Argument(이름 붙은 인수) 같은 것들입니다. 특히 Pair, Triple처럼 반환 형태를 명확히 표현할 수 있다는 점은 꽤 마음에 들었습니다. 이런 부분을 보면 Kotlin은 현대적인 언어의 편의성을 가져오면서도, Java가 갖고 있던 안정성과 단단함을 아주 쉽게 버리지는 않았다는 인상을 줍니다.

물론 이런 장점들은 최근 Java도 꽤 많이 따라오고 있긴 합니다. 다만 아직은 Kotlin 쪽이 한발 앞서 있다는 느낌입니다.

### 클래스 하나가 파일 하나일 필요는 없다

Java에서는 "한 파일에 한 클래스"가 거의 상식처럼 자리 잡고 있습니다. 물론 내부 클래스를 쓸 수는 있지만, 말 그대로 클래스 안에 들어가는 구조라 순수하게 나란히 놓인 여러 클래스를 담는 느낌과는 다릅니다. 반면 Kotlin은 일반 클래스도 하나의 파일에 여러 개 정의할 수 있습니다.

이 점은 생각보다 편리합니다. 예를 들어 DTO, DAO, Entity처럼 성격이 비슷한 클래스를 묶어서 관리하고 싶을 때, 관련 타입을 한 파일에 모아 둘 수 있기 때문입니다. 꼭 정답이라고 말할 수는 없지만, 적어도 패키지 안에 파일이 과하게 흩어지는 것을 줄이는 데는 도움이 됩니다. 결국 취향의 문제일 수도 있지만, 이런 선택권이 있다는 점 자체는 장점이라고 생각합니다.

물론 선택지가 많다고 해서 언제나 좋은 것은 아닙니다. 그래도 파일 안에 클래스를 몇 개 둘지는 코드 스타일의 영역에 가깝고, 실제 구현 방식에 큰 제약을 주는 요소는 아닙니다. 그런 점에서 Kotlin의 유연함은 꽤 반갑게 느껴졌습니다.

### 확장 함수는 꽤 강력하다

Java의 단점으로 자주 언급되는 것이 지나친 장황함, 즉 `verbose`함입니다. 흔히 말하는 boilerplate 코드를 매번 반복해서 써야 한다는 점은 생산성 면에서도 불리합니다. 그래서 Java 진영에서는 다양한 디자인 패턴이 발전했고, IDE의 자동 생성 기능이나 Lombok 같은 도구도 널리 쓰이게 됐습니다. 저 역시 프레임워크 개발에 참여했을 때, 결국 핵심 목표 중 하나는 반복 코드를 줄이는 것이었습니다.

Kotlin은 이런 문제에 대한 반작용으로 설계된 언어처럼 느껴집니다. 단순히 최근 언어의 유행을 따라간 것이 아니라, Java에서 불편했던 점을 실제로 개선하려는 의지가 설계에 녹아 있다는 인상이 강했습니다.

### 표준 라이브러리가 정말 편하다

확장 함수가 편리한 이유와도 연결되는데, Kotlin의 표준 라이브러리에 들어 있는 여러 함수 역시 같은 맥락에서 유용합니다. 대표적으로 `Scope Functions`라고 부르는 [let](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/let.html), [with](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/with.html), [apply](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html), [run](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/run.html), [also](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/also.html)가 있습니다.

Java에서도 비슷한 일을 못 하는 것은 아닙니다. 유틸리티 클래스를 따로 만들거나, private 메서드를 정의하거나, 특정 클래스를 상속해서 메서드를 추가하는 식으로 우회할 수는 있습니다. 다만 그런 방식은 번거롭고, 작은 목적을 위해 너무 많은 구조를 도입하게 될 때가 많습니다. Kotlin은 이런 문제를 훨씬 함수형에 가까운 방식으로 풀어 줍니다.

예를 들어 `let`을 보면 다음과 같은 `data class`가 있다고 가정할 수 있습니다.

```kotlin
data class Member(val region: String, val name: String)
```

여기서 하나의 인스턴스를 만들면 이렇게 됩니다.

```kotlin
var john = Member("Tokyo", "John")
```

그리고 나중에 `jake`라는 `Member` 인스턴스를 하나 더 만든다고 해 보겠습니다. 이때 `jake`는 항상 `john`과 같은 `region`을 가져야 합니다.

```kotlin
var jake = Member("Tokyo", "Jake")
```

Java스럽게 정리하면 보통 이렇게 공통 값을 따로 빼 두게 됩니다.

```kotlin
var tokyo = "Tokyo"
var john = Member(tokyo, "John")
var jake = Member(tokyo, "Jake")
```

그런데 `let`을 사용하면 이렇게 표현할 수 있습니다.

```kotlin
var jake = john.let {
    Member(it.region, "jake")
}
```

이 방식은 공통 값을 별도 변수로 빼지 않아도 `jake`의 `region`이 `john`의 값을 그대로 따른다는 점을 코드 안에서 자연스럽게 드러냅니다. 어떤 의미에서는 "john과 jake가 같은 region을 공유한다"는 의도가 더 잘 드러난다고도 볼 수 있습니다. 예시는 단순하지만, 공유해야 할 값이 많아질수록 이런 방식이 훨씬 우아하게 느껴집니다. 같은 일을 Java에서 억지로 흉내 내는 쪽이 오히려 더 번거로워 보일 정도입니다.

## 이 점은 아쉬웠다

Kotlin이 Java의 불편함을 많이 해소해 준 것은 사실이지만, 그렇다고 모든 면에서 Java보다 더 낫다고 단정하기는 어렵다고 느꼈습니다. 아래 내용은 어디까지나 개인적인 인상에 가깝습니다.

### `var`와 타입 표기의 애매함

현대적인 언어에 익숙한 사람이라면, 변수 선언이 `var` 중심으로 정리되는 것을 장점으로 볼 수도 있습니다. 실제로 Kotlin뿐 아니라 JavaScript, C# 같은 여러 언어가 `var`를 사용하고 있고, Java도 10부터 `var`를 도입했습니다. Python처럼 아예 변수 선언 키워드가 없는 언어도 있죠.

다만 저는 `var`가 정말로 더 좋은 방식인지 아직은 조금 의문이 있습니다. Java에 너무 익숙해서일 수도 있겠지만, 타입을 앞에 적는 전통적인 방식은 "이 값이 변수이고, 타입이 무엇인지"를 한 번에 보여 준다는 점에서 여전히 장점이 있다고 느낍니다.

이런 생각을 하는 이유 중 하나는, 오히려 최근 언어들이 타입 정보를 더 강조하는 방향으로 가고 있기 때문입니다. TypeScript는 JavaScript 위에 타입 시스템을 얹었고, Python도 3.6 이후 타입 힌트를 적극적으로 쓰기 시작했습니다. 즉 "변수라는 사실만 알면 된다"에서 "타입도 같이 보이는 편이 낫다"로 이동하는 흐름도 분명히 있습니다. 그런 관점에서 보면, `var`만으로 시작한 뒤 결국 타입 주석을 다시 붙이게 되는 상황은 약간 애매합니다.

예를 들어 Kotlin에서는 이렇게 쓸 수 있습니다.

```kotlin
var a = "this is string"
```

타입을 명시하면 이렇게 됩니다.

```kotlin
var b: String = "this is string"
```

반면 Java식으로 쓰면 다음과 같습니다.

```java
String a = "this is string";
```

이 경우에는 오히려 Java 쪽이 더 짧고, 타입도 한눈에 들어옵니다. 물론 이는 취향의 문제일 수 있지만, 적어도 "무조건 `var`가 더 낫다"는 말에는 쉽게 동의하기 어렵습니다.

또 Kotlin은 타입 표기 규칙이 문맥마다 조금씩 다르게 느껴집니다. Java에서는 변수든 반환값이든 매개변수든 기본적으로 타입을 적는 방식이 거의 통일되어 있습니다. 그런데 Kotlin에서는 어떤 곳은 타입을 써야 하고, 어떤 곳은 추론에 맡길 수 있고, 어떤 곳은 `val`/`var`를 꼭 붙여야 하는 식으로 규칙이 나뉘어 있습니다. 익숙해지면 문제없겠지만, Java에서 넘어오면 처음에는 꽤 헷갈립니다.

예를 들어 함수 인자에는 타입 지정이 필요합니다.

```kotlin
fun getMember(request: ServerRequest): Mono<ServerResponse> =
        ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(repository.findById(request.pathVariable(id).toLong()).map { MemberDto(it.username, it.name) })
            .switchIfEmpty(notFound().build())
```

하지만 함수 반환형은 single expression이면 생략할 수 있습니다.

```kotlin
fun getMember(request: ServerRequest) =
        ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(repository.findById(request.pathVariable(id).toLong()).map { MemberDto(it.username, it.name) })
            .switchIfEmpty(notFound().build())
```

반대로 `return`을 명시하는 블록 본문에서는 반환형을 생략하면 컴파일 오류가 납니다.

```kotlin
// 컴파일 오류: 반환형을 쓰지 않으면 Unit으로 추론된다
fun getMember(request: ServerRequest) {
      return ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(repository.findById(request.pathVariable(id).toLong()).map { MemberDto(it.username, it.name) })
            .switchIfEmpty(notFound().build())
}
```

또 `data class`에서는 프로퍼티에 `val`이나 `var`를 붙여야 하지만, 일반 클래스 생성자 인자에는 꼭 필요하지 않습니다.

```kotlin
// data class에서는 val 또는 var가 필요하다
data class MemberDto(val username: String, val name: String)

// 일반 클래스에서는 생성자 인자만 선언할 수도 있다
class MemberEntity(username: String, name: String)
```

이런 규칙은 결국 컴파일러가 잡아 주기 때문에 적응만 하면 되지만, 처음 배울 때는 Java보다 오히려 더 헷갈리는 부분도 있다고 느꼈습니다.

### 의존성 구성이 조금 번거롭다

Kotlin을 프로젝트에 넣으려면 기본적으로 표준 라이브러리를 의존성에 추가해야 합니다. 그런데 여기서 `kotlin-stdlib`만 넣으면 Java 1.7 이후에 추가된 일부 기능, 예를 들면 `AutoCloseable` 등을 바로 쓰지 못하는 경우가 있습니다. 그래서 JDK 버전에 따라 `kotlin-stdlib-jdk7`이나 `kotlin-stdlib-jdk8` 같은 의존성을 추가해야 합니다.

처음에는 Oracle과 Google의 소송 같은 배경 때문에 저작권 문제를 피하려고 일부러 별도 패키지를 만든 건가 싶었는데, 실제로는 Java 9의 모듈 시스템 대응과 관련된 이유라고 하더군요. 그래서 예전의 `kotlin-stdlib-jre7`, `kotlin-stdlib-jre8`이 각각 `kotlin-stdlib-jdk7`, `kotlin-stdlib-jdk8`으로 대체된 흐름이 있었다고 이해하고 있습니다.

물론 Maven이나 Gradle을 쓰면 한 번 설정해 두는 것으로 끝나는 문제일 수도 있습니다. 하지만 처음 접하는 입장에서는 "대체 어느 것을 넣어야 하나", "내 프로젝트에는 어느 패키지가 필요한가"를 파악하는 데 시간을 꽤 쓰게 됩니다. 예를 들어 `kotlin-stdlib-jdk7`이 없어도 JDK 1.7 기능 전체를 못 쓰는 것은 아니고, 일부 기능만 영향을 받는 식이라 더 헷갈립니다. 프로젝트에 `AutoCloseable`이 필요한지에 따라 의존성을 추가해야 하는지 판단하는 것도 생각보다 번거롭습니다.

그리고 JDK 7, JDK 8 대응 표준 라이브러리가 따로 존재한다는 사실은, Java가 계속 버전 업될 때 비슷한 고민이 더 생길 수 있다는 뜻이기도 합니다. 여기에 `kotlin-reflect` 같은 별도 의존성까지 더해지면, 프로젝트 구조에 따라서는 Kotlin 도입을 조금 더 신중하게 검토해야 할 수도 있습니다. 어떤 의미에서는 Kotlin이 Post Java 후보로서 충분한 잠재력을 갖고 있으면서도, Android 외 영역에서 생각보다 천천히 확산된 이유 중 하나가 이런 초기 진입 장벽일 수도 있겠다는 생각도 들었습니다.

## 마지막으로

장점과 아쉬운 점을 나눠 Kotlin에 대한 첫인상을 정리해 봤습니다. 아직 실제 대형 프로젝트에서 깊게 써 본 것은 아니라 [Type aliases](https://kotlinlang.org/docs/reference/type-aliases.html)나 [inline class](https://kotlinlang.org/docs/reference/inline-classes.html) 같은 기능은 제대로 활용해 보지도 못했습니다. 그래도 지금까지 만져 본 범위에서는, Kotlin이 단순히 "Java를 조금 줄여 쓴 언어"는 아니라는 점은 분명했습니다.

개인적으로는 이미 Java를 쓰는 팀이라면 Kotlin 도입을 충분히 검토해 볼 만하다고 봅니다. Java 개발자라면 비교적 빠르게 적응할 수 있고, 생산성 면에서도 얻는 것이 분명하기 때문입니다. JVM 기반이면서도 Native와 JavaScript 방향까지 함께 보고 있다는 점을 생각하면, Kotlin은 앞으로도 계속 눈여겨볼 만한 언어입니다.
