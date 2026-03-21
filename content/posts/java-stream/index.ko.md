---
title: "Stream을 더 잘 쓰기"
date: 2020-04-06
categories:
  - java
image: "../../images/java.webp"
tags:
  - stream
  - java
translationKey: "posts/java-stream"
---

개인적으로 함수형 프로그래밍에 아주 익숙한 편은 아니지만, Java 1.8의 Stream API는 꽤 좋아합니다. Lambda나 Optional도 좋아하지만, 제가 Java 자격을 공부했던 이유 중 하나도 Stream을 더 제대로 이해해 보고 싶어서였습니다.

그렇게 Stream을 공부하다 보니 몇 가지 의문이 생겼습니다. Stream은 분명 좋은 API지만, 전통적인 Java 코드 스타일과는 결이 꽤 다릅니다. 도입된 이유가 있을 텐데 반대로 단점은 없을까, 그리고 제가 쓰고 있는 방식이 과연 적절한가 하는 생각이 들었습니다.

이번 글에서는 이런 의문에 대해 직접 찾아본 내용을 정리해 보겠습니다. 정답을 단정하기보다는, Stream을 바라보는 한 가지 관점 정도로 읽어 주시면 좋겠습니다.

## Stream은 만능인가

먼저 Stream은 만능인지 생각해 봤습니다. 기존 코드를 전부 Stream으로 바꿔도 괜찮을까요? 앞으로는 무조건 Stream으로 쓰는 게 맞을까요?

새 API가 등장하고 기존 코드와 같은 일을 할 수 있다면 분명 이유가 있습니다. Java의 NIO도 그랬습니다. 기존 I/O보다 장점이 있어 등장했지만, 모든 상황에서 더 낫다고 할 수는 없었습니다. Stream도 비슷할 수 있다고 생각했습니다.

결론부터 말하면, 모든 코드를 Stream으로 바꿀 필요는 없습니다. 이유를 나눠서 보겠습니다.

### 성능이 떨어질 수 있다

Java 1.5에는 전통적인 `for`문 외에 확장 `for`문이 들어왔고, 1.8에는 `forEach()`와 Stream이 추가됐습니다. 하지만 `forEach()`와 Stream은 확장 `for`문보다 느릴 수 있습니다. 어떤 벤치마크에서는 Parallel을 써도 확장 `for`문보다 성능이 낮았다는 결과도 있었습니다. 이유는 간단합니다. Stream은 내부에서 처리해야 할 일이 더 많기 때문입니다. 배열을 Stream으로 바꿀 때도 래핑 비용이 추가됩니다.

특히 객체보다 primitive 타입을 다룰 때 성능 차이가 더 커질 수 있습니다. 그래서 무조건 배열을 Stream으로 바꿔야 하는 것은 아닙니다. 안정적으로 코드를 쓸 수 있을 때만 Stream이나 `forEach()`를 쓰는 편이 좋습니다. primitive 타입을 다룰 때는 `IntStream`처럼 전용 타입을 쓰는 것이 더 낫습니다.

JVM이 오랫동안 전통적인 `for`문에 최적화돼 있었고, Stream은 상대적으로 늦게 들어왔기 때문에 성능이 밀린다는 이야기도 있습니다. 지금은 버전이 많이 올라왔지만, 그래도 전통적인 `for`문이 아직은 더 빠르지 않을까 생각합니다.

### 중간에 멈추기 어렵다

Stream은 일반적인 `for`문처럼 `continue`, `break`, `return`으로 중간에 끊기 어렵습니다. 기본적으로 모든 요소를 처리하는 전제로 설계됐기 때문입니다. 그래서 목적에 맞게 메서드를 구분해서 써야 합니다.

- 조건에 맞는 요소만 처리: `filter()`
- Collection으로 모으기: `collect()`
- 한 개만 꺼내기: `findAny()`, `findFirst()`

### 루프 변수를 쓸 수 없다

전통적인 `for`문에서는 루프 변수를 써서 몇 번째 반복인지 셀 수 있습니다. 하지만 Stream에서는 루프 변수를 직접 쓰기 어렵습니다. 외부 변수로 돌리면 컴파일 에러가 납니다.

숫자 기반으로 처리하려면 `IntStream`을 쓰거나, 필요한 경우 `skip()`을 써야 합니다. 이런 경우는 그냥 전통적인 `for`문이 더 맞는 경우가 많습니다.

### 함수형의 전제

Stream 안에서 외부 변수를 흉내 내는 방법이 아주 없는 것은 아닙니다. `AtomicInteger`를 쓰는 식입니다. 하지만 그렇게까지 할 거면 Stream을 쓸 이유가 약합니다. 함수형 프로그래밍의 목적에도 맞지 않습니다.

함수형 프로그래밍에는 불변성이라는 개념이 있습니다. 이전에 [인스턴스를 Immutable하게 만들기 위한 고민](/ko/posts/java-thoughts-of-immutable/)에서 다룬 적이 있지만, 중요한 건 데이터가 바뀌지 않으면 원본은 그대로 두고, 처리 결과는 복사본으로 다룬다는 점입니다. Stream으로 처리한 결과가 새 인스턴스로 나오는 것도 그 때문입니다.

List를 직접 돌리면 원본을 수정할 수 있지만, Stream으로 처리하면 새 List를 만들게 됩니다. 중간 연산을 하더라도 종단 연산이 끝나면 Stream은 닫혀 재사용할 수 없습니다. 즉, 원본을 바꾸지 않는 방식에 더 가깝습니다.

그래서 Stream을 쓸지 말지는 원본 데이터를 어떻게 다룰지부터 정해야 합니다. 물론 Stream을 쓰지 않더라도 함수형 스타일로 더 잘 쓸 수 있는 경우도 많습니다.

## Stream을 더 잘 쓰기

다음은 Stream을 어떻게 더 잘 쓸 수 있느냐는 질문입니다. Stream은 한 번 종단 연산을 하면 재사용할 수 없습니다. 그렇다면 같은 데이터에 대해 여러 처리를 하고 싶을 때는 어떻게 해야 할까요?

또 하나는, 메서드 체이닝이 가능한 API는 비효율적으로 쓰기 쉬운데, 이를 어떻게 정리하느냐는 고민이었습니다. Collection 자체에도 `forEach()`가 있고, 굳이 Stream으로 바꾸지 않아도 되는 경우가 있습니다.

### 재사용

Stream은 중간 연산은 여러 번 할 수 있지만, 종단 연산이 나오면 닫히고 다시 쓸 수 없습니다. 애초에 데이터 저장용이 아니라 처리용이기 때문입니다.

그렇다고 같은 데이터를 여러 방식으로 처리하지 못하는 것은 아닙니다. 필요한 데이터를 미리 Collection에 담아 두고, 필요할 때마다 다시 Stream으로 바꿔 쓰면 됩니다.

```java
List<String> names =
        Stream.of("Eric", "Elena", "Java")
                .filter(name -> name.contains("a"))
                .collect(Collectors.toList());

Optional<String> firstElement = names.stream().findFirst();
Optional<String> anyElement = names.stream().findAny();
```

데이터를 한 번 모아 두고, 필요한 처리를 할 때마다 다시 Stream으로 쓰는 방식입니다. `peek()`를 끼워서 다른 Collection에 넣는 것도 가능합니다.

### 짧게 쓰기

Stream은 체이닝이 편한 만큼 비효율적으로 쓰기 쉬웠습니다. 그래서 자주 보이는 변환을 정리해 보면 이런 식입니다.

- Collection 메서드 사용

```java
collection.stream().forEach()
  → collection.forEach()

collection.stream().toArray()
  → collection.toArray()
```

- Stream 만들기

```java
Arrays.asList().stream()
  → Arrays.stream() / Stream.of()

Collections.emptyList().stream()
  → Stream.empty()

IntStream.range(expr1, expr2).mapToObj(x -> array[x])
  → Arrays.stream(array, expr1, expr2)

Collections.nCopies(count, ...)
  → Stream.generate().limit(count)
```

- 요소 판정

```java
stream.filter().findFirst().isPresent()
  → stream.anyMatch()

stream.map().anyMatch(Boolean::booleanValue)
  → stream.anyMatch()

!stream.anyMatch()
  → stream.noneMatch()

!stream.anyMatch(x -> !(...))
  → stream.allMatch()

stream.sorted(comparator).findFirst()
  → Stream.min(comparator)
```

- 요소 모으기

```java
stream.collect(counting())
  → stream.count()

stream.collect(maxBy())
  → stream.max()

stream.collect(mapping())
  → stream.map().collect()

stream.collect(reducing())
  → stream.reduce()

stream.collect(summingInt())
  → stream.mapToInt().sum()
```

- 요소 처리

```java
stream.map(x -> {...; return x;})
  → stream.peek(x -> ...)
```

## 마지막으로

함수형 프로그래밍에 익숙하지 않아도 Stream 자체는 꽤 매력적인 API입니다. Java 1.8이 처음 나왔을 때는 성능도 애매하고 읽기도 어렵다는 비판이 많았지만, 지금은 버전도 많이 올라왔습니다. 이제는 Stream으로 조금 더 현대적인 스타일을 써 볼 만한 시점이라고 생각합니다.

Stream을 통해 함수형 프로그래밍을 맛볼 수 있다는 점도 장점입니다. 물론 Stream이 완전한 함수형 프로그래밍의 예라고 보기는 어렵지만, 객체지향뿐 아니라 다른 스타일의 프로그래밍 트렌드를 경험해 볼 수 있다는 것만으로도 의미가 있습니다. 프로그래밍 세계는 계속 변하니, 최근 트렌드 정도는 알고 있는 편이 좋습니다.
