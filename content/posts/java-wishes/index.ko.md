---
title: "Java가 이렇게 진화하면 좋겠다"
date: 2020-02-03
categories:
  - java
image: "../../images/java.webp"
tags:
  - java
translationKey: "posts/java-wishes"
---

Java는 오래도록 생산성, 성능, 안정성 면에서 좋은 평을 받아 왔고, 최근에는 버전 업을 통해 다양한 기능이 추가되고 있습니다. 업무에서는 주로 11 버전을 쓰고 있지만, 다음 LTS인 17이 나오면 그쪽으로 옮겨 갈 것 같습니다. 새 버전이 나오면 변경 이력을 챙겨 보긴 하지만, 언어 사양 자체가 바뀌는 경우는 새 API 추가보다 훨씬 적습니다.

Java의 장점이던 생산성은 지금은 Python이나 JavaScript에 비해 아쉬운 면이 있습니다. 예전에는 "코드가 읽기 쉽다"가 장점이었지만, 어느새 "장황하다"는 인식이 더 강해졌습니다. 저는 Java를 좋아하지만, 쓰다 보면 불편한 점도 있고 다른 언어처럼 바뀌었으면 하는 부분도 있습니다. 이번 글에서는 업무에서 Java를 쓰며 느낀 아쉬움과, 다른 언어를 보며 개선되면 좋겠다고 생각한 부분을 정리해 보려 합니다.

## Optional 표기 개선

이전 글에서 소개한 `Optional`은 Java뿐 아니라 다른 언어에도 널리 있는 API입니다. 오히려 Java가 다른 언어의 영향을 받아 도입한 기능에 가깝다고 볼 수도 있습니다. 저도 자주 쓰고 있고 꽤 편리하다고 생각하지만, 다른 언어와 비교하면 여전히 아쉬운 부분이 있습니다.

복잡하게 중첩된 객체에서 특정 필드를 읽고, `null`이면 기본값을 돌려주는 예로 비교해 보겠습니다.

### Java

Java에서는 `Optional`을 감싸고 `map()`을 이어 붙인 뒤 `orElse()`로 기본값을 줍니다.

```java
SomeClass object;

Optional.ofNullable(object)
        .map(obj -> obj.getProp1())
        .map(prop1 -> prop1.getProp2())
        .map(prop2 -> prop2.getProp3())
        .orElse("default");
```

### C#

C#은 Java보다 훨씬 간단합니다. 오브젝트가 `null`일 수 있다는 점만 먼저 선언하지 않는다는 차이가 있습니다.

```csharp
object?.prop1?.prop2?.prop3 ?? "default";
```

### JavaScript

JavaScript도 비슷합니다.

```js
object?.prop1?.prop2?.prop3 ?? "default"
```

### Swift

Swift도 크게 다르지 않습니다.

```swift
object?.prop1?.prop2?.prop3 ?? "default"
```

### Kotlin

Kotlin 역시 Elvis 연산자만 보면 같은 계열입니다.

```kotlin
object?.prop1?.prop2?.prop3 ?: "default"
```

다른 언어와 비교해 보면 Java의 `Optional`은 확실히 장황합니다. 언젠가는 언어 기본 문법으로 흡수돼서 `?`처럼 더 간단하게 표현할 수 있으면 좋겠습니다.

## Multiple Return

Java는 메서드 반환값을 하나만 가질 수 있지만, Python 같은 언어는 여러 값을 바로 돌려줄 수 있습니다. 물론 Java에서도 Bean이나 Collection으로 우회할 수 있지만, 더 편한 방식이 있다면 쓰고 싶어집니다.

### Java

Java에서는 보통 Collection이나 Bean으로 반환합니다.

```java
public List<Integer> multipleReturn() {
    return Arrays.asList(1, 2);
}

List<Integer> data = multipleReturn();
```

### C#

C#은 예전 방식과 새 방식이 꽤 다릅니다. 옛날에는 Java처럼 Tuple을 썼지만, 최신 방식은 훨씬 보기 좋습니다.

```csharp
// 예전 방식
public Tuple<int, int> oldMultipleReturn() {
    return Tuple.Create(1, 2);
}

var result = oldMultipleReturn();

// 새로운 방식
public (int, int) newMultipleReturn() {
    return (1, 2);
}

(int one, int two) = newMultipleReturn();
```

### Python

Python은 더 자연스럽습니다.

```python
def multiple_return():
    return 1, 2

a, b = multiple_return()
print(a)

d = multiple_return()
print(d)
```

### JavaScript

ES6 이후에는 비슷한 느낌으로 쓸 수 있습니다.

```js
function multipleReturn() {
    return {
        first: 1,
        second: 2
    };
}

const { first, second } = multipleReturn();
```

### Swift

Swift도 깔끔합니다.

```swift
func multipleReturn() -> (Int, Int) {
    return (1, 2)
}

let (first, second) = multipleReturn()
```

### Kotlin

Kotlin은 `Pair`와 `Triple`이 있어서 쓰기 쉽습니다.

```kotlin
fun multipleReturn(): Pair<Int, Int> {
    return 1 to 2
}

val (first, second) = multipleReturn()
```

Java는 메서드 역할이 분명하다는 장점이 있지만, 값을 그대로 변수처럼 다루는 면에서는 Python 방식이 더 편합니다. 이런 기능은 이제 현대 언어라면 거의 기본처럼 갖고 있는 것 같습니다. Java에도 언젠가는 들어올까요?

## 인수 타입 확장

하나의 메서드에서 인수 타입을 여러 개 지정할 수 있으면 편할 때가 있습니다. Java는 오버로딩으로 이 문제를 해결합니다.

### Java

```java
public void doSomething(String value) {
    // String 처리
}

public void doSomething(int value) {
    // int 처리
}
```

물론 `Object`로 받고 내부에서 `instanceof`로 나눌 수도 있습니다. 하지만 전자는 코드가 너무 길어지고, 후자는 예상하지 못한 타입이 들어왔을 때 거동이 이상해질 수 있습니다.

### TypeScript

TypeScript는 union 타입으로 더 간단하게 풀 수 있습니다.

```ts
function checkString(v: string | number) {
    if (typeof v === "string") {
        return true;
    }
    return false;
}
```

타입만 다른 경우 오버로딩과 private 메서드 조합으로 처리하던 것보다 훨씬 읽기 쉽습니다. Java에도 이런 표현이 더 넓게 들어오면 좋겠습니다.

## Throw 가능한 삼항 연산자

결과가 두 가지뿐이라면 `if`보다 삼항 연산자가 더 짧습니다. 그런데 Java의 삼항 연산자는 `throw`를 직접 쓰기 어렵습니다. 조건식에 맞지 않을 때는 결국 아래처럼 써야 합니다.

```java
// x가 0이면 number도 0, 0이 아니면 예외를 던진다
int number = x -> {
    if (x == 0) {
        return x;
    } else {
        throw new RuntimeException();
    }
}
```

삼항 연산자에서 억지로 예외를 던지려면 아래 같은 방법이 있긴 합니다.

```java
// 일반적인 반환값을 가진 뒤 예외만 던지는 메서드
public <T> T throwSomeException() {
    throw new RuntimeException();
}

// else에서 메서드를 호출한다
int number = x == 0 ? x : throwSomeException();
```

저는 `if`가 단순히 두 갈래를 처리하기 위한 것이라면 코드가 길어지는 느낌이 있고, 굳이 도우미 메서드를 만들고 싶지는 않았습니다. 삼항 연산자에서 바로 예외를 던질 수 있으면 좋겠다고 생각했습니다.

### C#

C#은 이 부분이 잘 되어 있습니다.

```csharp
// 예전 방식
int number = x == 0 ? x : new Func<int>(() => { throw new Exception(); })();

// 최신 방식
int number = x == 0 ? x : throw new Exception();
```

### Kotlin

Kotlin은 애초에 삼항 연산자가 없고 `if-else`가 표현식이어서 깔끔합니다.

```kotlin
val i = if (x == 0) x else throw Exception("error")
```

### JavaScript

JavaScript도 함수로 감싸면 비슷하게 처리할 수 있습니다.

```js
var number = (x == 0) ? x : (function() { throw "error" }());
```

굳이 이렇게까지 하고 싶지는 않지만, 다른 언어들은 이런 표현이 가능하다는 사실만으로도 흥미롭습니다. Java도 언젠가 이런 쪽이 더 자연스러워졌으면 합니다.

## 접근 제한자 확장

Java는 `public`, `private`, `protected`를 주로 씁니다. 하지만 직접 라이브러리를 만들 때는 `public`과 `private` 사이 어딘가의 제한이 있으면 좋겠다고 느낍니다. 예를 들어 Jar 안에서는 보이지만 외부에서는 보이지 않게 하고 싶을 때가 있습니다.

Java 9에서 모듈이 도입됐지만, 저는 모듈 때문에 겪은 문제도 있어서 가능하면 쓰고 싶지 않습니다. 같은 프로젝트 안에서는 패키지가 달라도 접근할 수 있는 제한자, 혹은 같은 모듈 안에서만 보이는 제한자가 있으면 좋겠습니다. C#, Swift, Kotlin의 `internal` 같은 개념이 Java에도 있으면 꽤 편할 것 같습니다.

## 마지막으로

최근 Java 업데이트를 보면 편리한 기능이 계속 추가되고 있습니다. 14에서는 `record`로 Lombok의 `@Data`와 비슷한 선언도 가능해질 예정이었고, 이후 버전에서도 이런 변화는 계속 이어졌습니다. 1.8도 충분히 좋은 언어였지만, 앞으로는 다른 언어가 가진 장점도 더 적극적으로 흡수하면 좋겠다고 생각합니다.

무엇보다 다른 언어가 같은 문제를 어떻게 풀고 있는지 비교해 보는 일 자체가 꽤 좋은 공부였습니다. 결국 "Java가 부족하다"는 이야기라기보다, 더 좋아질 여지가 아직 많다는 이야기로 받아들이면 될 것 같습니다.
