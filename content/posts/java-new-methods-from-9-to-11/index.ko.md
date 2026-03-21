---
title: "Java 9~11에서 추가된 메서드 정리"
date: 2020-12-30
categories: 
  - java
image: "../../images/java.webp"
tags:
  - java
  - stream
  - collection
  - optional
  - string
translationKey: "posts/java-new-methods-from-9-to-11"
---

업무에서는 Java 11을 쓰는 경우가 많지만, 막상 제가 작성한 코드를 돌아보면 Java 9 이후에 추가된 메서드를 적극적으로 활용하고 있다고 말하기는 어렵습니다. 새 메서드라고 해도 단순한 문법 설탕만 있는 것은 아니고, 기능적으로 더 편하거나 성능상 이점이 있는 경우도 적지 않습니다. 그래서 지금 당장 쓰지 않더라도 한 번쯤 정리해 둘 가치는 충분하다고 느꼈습니다.

게다가 당시에는 다음 LTS인 Java 17 출시도 가까워지고 있었습니다. 그래서 이번 글에서는 제 코드 습관을 되돌아보는 의미도 겸해, Java 9부터 11까지 추가된 메서드 중 실무에서 써 볼 만한 것들을 추려 간단히 정리해 보려 합니다.

대부분은 이미 Java 11 환경이라면 사용할 수 있는 메서드들이므로 아주 새로운 내용은 아닐 수 있습니다. 그래도 각 메서드 이름 옆에 어느 버전에서 들어왔는지 함께 적어 둘 테니 참고가 되면 좋겠습니다.
## 스트림

Stream은 Java 8을 대표하는 변화 중 하나였습니다. Java 9에서는 여기에 몇 가지 메서드가 더 추가되면서, 기존보다 더 자연스럽게 쓰거나 아쉬웠던 부분을 보완할 수 있게 됐습니다.
### Iterate (9)

`iterate()`는 이름만 보면 바로 감이 오지 않을 수도 있지만, 전통적인 `for`문과 비슷한 형태로 Stream 요소를 만들어 낼 수 있는 메서드입니다. 즉, 초기값, 계속 조건, 다음 값을 만드는 규칙을 지정해 스트림을 구성합니다. 예를 들면 다음과 같습니다.
```java
// 0~9를 출력
Stream.iterate(0, i -> i < 10, i -> i + 1).forEach(System.out::println);
```

즉, 다음 코드와 동일한 의미를 갖습니다.
```java
for (int i = 0; i < 10; i++) {
    System.out.println(i);
}
```

물론 초기값이 꼭 숫자일 필요는 없습니다. 제네릭 타입이기 때문에 아래처럼 문자열에도 쓸 수 있습니다.
```java
// A로 삼각형을 출력
Stream.iterate("A", s -> s.length() < 10, s -> s + "A").forEach(System.out::println);
```

또한 루프의 연속 조건을 지정하지 않을 수도 있습니다.
```java
// A로 삼각형을 출력
Stream.iterate("A", s -> s + "A").forEach(System.out::println);
```

그렇다면 계속 조건을 생략하면 무한 루프가 되는 것 아닌가 싶을 수 있습니다. 실제로 그렇습니다. 대신 Java 9에는 이런 스트림을 중간에서 끊을 수 있는 메서드도 함께 추가됐습니다. 바로 다음의 `takeWhile()`입니다.
### takeWhile (9)

`takeWhile()`은 조건이 깨질 때까지 요소를 가져오고, 그 시점에서 스트림 처리를 멈춥니다. 기존 `limit()`이 "몇 개까지"만 지정할 수 있었다면, `takeWhile()`은 조건식으로 경계를 정할 수 있다는 점이 다릅니다.
```java
// AAAAAAAAA까지 출력
Stream.iterate("A", s -> s + "A")
    .takeWhile(s -> s.length() < 10)
    .forEach(System.out::println);
```

그래서 `iterate()`에서 종료 조건을 직접 넣지 않았다면, `takeWhile()`로 어디서 멈출지 명시해 두는 편이 읽기 좋습니다.
### dropWhile (9)

`dropWhile()`은 이름 그대로 `takeWhile()`의 반대편에 있는 메서드입니다. 조건에 맞는 요소를 앞에서부터 버리고, 그 이후 요소들만 남깁니다.
```java
// AAAAA부터 출력
Stream.iterate("A", s -> s.length() < 10, s -> s + "A")
    .dropWhile(s -> !s.contains("AAAAA"))
    .forEach(System.out::println);
```

### ofNullable (9)

Java 8에서는 null일 수 있는 값을 스트림에 넣으려면 먼저 null 여부를 확인하고, null이면 `Stream.empty()`를 반환해야 했습니다. 흔히 보던 Java식 null 체크입니다. 예를 들면 다음과 같습니다.
```java
// 요소의 null 체크를 포함한 Stream collect
keyList.stream()
    .flatMap(k -> {
        Object value = map.get(k);
        return value != null ? Stream.of(value) : Stream.empty();
    })
    .collect(Collectors.toList());
```

Java 9에서는 이를 더 간단하게 쓸 수 있습니다. 느낌으로는 `Optional.ofNullable()`에 가깝습니다.
```java
keyList.stream()
  .flatMap(k -> Stream.ofNullable(map.get(k)))
  .collect(Collectors.toList());
```

## Collectors

`Collectors` API 쪽 변화는 주로 "기존에 길게 써야 했던 코드를 더 짧고 직접적으로 표현할 수 있게 해 준다"는 방향에 가깝습니다. Stream에서만 하던 작업을 Collector 쪽으로 옮길 수 있게 된 경우도 있습니다.
### filtering (9)

`Stream.filter()`와 비슷한 처리를 Collector 안에서도 할 수 있게 해 주는 메서드입니다. 취향 차이도 있겠지만, Collector 자체를 조합하거나 재사용하고 싶을 때는 꽤 유용합니다.
```java
// 0~9까지의 리스트
List<Integer> numbers = Stream.iterate(0, i -> i < 10, i -> i + 1).collect(Collectors.toList());

// Stream.filter()
numbers.stream()
    .filter(e -> e > 5)
    .collect(Collectors.toList()); // 6, 7, 8, 9

// Collectors.filtering()
numbers.stream()
    .collect(Collectors.filtering(e -> e > 5, Collectors.toList()));  // 6, 7, 8, 9
```

### flatMapping (9)

이름 그대로, `Collectors` 안에서 `flatMap`에 해당하는 동작을 할 수 있게 해 주는 메서드입니다. 예를 들어 아래 같은 클래스가 있다고 해 보겠습니다.
```java
public class Selling {
   String clientName;
   List<Product> products;
}
public class Product {
   String name;
   int value;
}
```

이 `Selling` 목록을 "`clientName`을 key로, `products`를 value로 하는 Map"으로 만들고 싶다면, 우선 이런 코드를 떠올릴 수 있습니다.
```java
Map<String, List<List<Product>>> result = operations.stream()
                .collect(Collectors.groupingBy(Selling::getClientName, Collectors.mapping(Selling::getProducts, Collectors.toList())));
```

하지만 이렇게 하면 `List<Product>`가 다시 List 안에 들어가서 `List<List<Product>>` 형태가 됩니다. 원하는 결과와 다르고, 이후 사용도 불편해집니다.
이를 `Map<String, List<Product>>`로 만들려면 직접 Collector를 조합해야 했습니다. 예를 들면 다음과 같습니다.
```java
Map<String, List<Product>> result = operations.stream()
    .collect(Collectors.groupingBy(Selling::getClientName, Collectors.mapping(Selling::getProducts,
        Collector.of(ArrayList::new, List::addAll, (x, y) -> {
            x.addAll(y);
            return x;
        }))));
```

다만 매번 이런 식으로 Collector를 직접 조합하는 건 번거롭고, 익숙하지 않으면 읽기도 쉽지 않습니다. 여기서 `flatMapping()`을 쓰면 같은 목적을 훨씬 간단하게 표현할 수 있습니다.
```java
Map<String, List<Product>> result = operations.stream()
    .collect(Collectors.groupingBy(Selling::getClientName, Collectors.flatMapping(selling -> selling.getProducts().stream(), Collectors.toList())));
```

### toUnmodifiable (10)

Java 10에서는 `Collectors`에 다음 세 가지 메서드가 추가되었습니다.
- `toUnmodifiableList()`
- `toUnmodifiableSet()`
- `toUnmodifiableMap()`

이 메서드들을 쓰면 `Collections.unmodifiableXxx()`를 한 번 더 감싸지 않아도, 바로 변경 불가능한 Collection을 만들 수 있습니다.
```java
// Collections.unmodifiableList
List<Integer> collectionsUnmodifiable = Collections.unmodifiableList(Stream.iterate(0, i -> i < 10, i -> i + 1).collect(Collectors.toList()));

// Collectors.toUnmodifiableList
List<Integer> collectionsUnmodifiable = Stream.iterate(0, i -> i < 10, i -> i + 1).collect(Collectors.toUnmodifiableList());
```

인자 구성도 기존 `toList()`, `toSet()`, `toMap()`과 거의 같아서 기존 코드에서 바꿔 쓰기 어렵지 않습니다. (`toMap()`은 원래처럼 key/value 매핑 지정이 필요합니다.)
## Collections

Collections API에 추가된 메서드들은 전반적으로 더 현대적인 작성 방식을 가능하게 해 줍니다. 다른 언어에서 흔히 보던 편의성을 Java식으로 받아들인 결과처럼 보이는 부분도 있습니다.
### Factory Method (9)

Java 9에서는 [팩토리 메서드](https://ko.wikipedia.org/wiki/Factory_Method_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3)에서 Collection을 작성할 수 있습니다. 사용법은 기존의 `Arrays.asList()`과 비슷한 느낌입니다.
```java
// List 생성
List<String> list = List.of("A", "B", "C");

// Set 생성
Set<Integer> set = Set.of(1, 2, 3);
```

Map의 경우 key와 value를 순서대로 넘겨 인스턴스를 만들 수도 있고, 엔트리 단위로 정의할 수도 있습니다.
```java
// Key와 Value 쌍으로 정의
Map<String, String> map = Map.of("foo", "a", "bar", "b", "baz", "c");

// 엔트리를 정의
Map<String, String> map = Map.ofEntries(
  Map.entry("foo", "a"),
  Map.entry("bar", "b"),
  Map.entry("baz", "c"));
```

이러한 팩토리 메서드로 작성한 Collection의 특징은, 처음부터 Unmodifiable인 객체가 된다는 것입니다. 따라서 응용 프로그램을 시작할 때 필드에 상수를 Collection으로 정의하는 경우 사용할 수 있습니다. 즉, 다음과 같은 기존 코드를 대체할 수 있는 것입니다.
```java
// 가장 기본적인 방식
Set<String> set = new HashSet<>();
set.add("foo");
set.add("bar");
set.add("baz");
set = Collections.unmodifiableSet(set);

// Double-brace initialization 
Set<String> set = Collections.unmodifiableSet(new HashSet<String>() {
    {
        add("foo");
        add("bar");
        add("baz");
    }
});
```

이 팩토리 메서드로 만든 Collection에는 아래 같은 특징이 있으니, 용도에 맞게 써야 합니다.
- Immutable(Unmodifiable)이 된다
- Null 요소를 지정할 수 없음
- 요소가 Serializable이면 Collection도 Serializable이 된다

#### copyOf (10)

List, Set, Map에 `copyOf()`이라는 메서드가 추가되어 있습니다. 인수에 각 Collection을 전달하면 Unmodifiable 복사할 수 있습니다.
```java
// 복사 원본 리스트
List<String> original = ...

// 복사한다
List<String> copy = List.copyOf(original);
```

## Optional

Optional을 적극적으로 쓰는 편인지 묻는다면, 저는 아직 그렇지는 않습니다. Stream 결과를 받는 정도를 제외하면 직접 Optional을 만드는 일은 많지 않습니다. 제약도 있고, 단순한 null 처리에는 오히려 과하다고 느껴질 때도 있습니다. 그래도 9와 10에서 추가된 메서드 덕분에 예전보다 활용 폭은 확실히 넓어졌습니다.
### or (9)

`or()`는 Optional이 비어 있을 때 대체 Optional을 반환하는 메서드입니다. `orElse()`나 `orElseGet()`이 실제 값을 돌려주는 것과 달리, 이쪽은 Optional 자체를 다시 돌려준다는 점이 다릅니다.
```java
String string = null;
Optional<String> optional = Optional.ofNullable(string);
System.out.println(optional.or(() -> Optional.of("default")).get()); // "default"
```

### orElseThrow (10)

Optional이 비어 있을 때 예외를 던지고 싶다면 `orElseThrow()`를 쓸 수 있습니다. 기본적으로도 비어 있는 Optional에서 값을 꺼내면 `NoSuchElementException`이 나지만, 비즈니스 로직에 맞는 예외로 바꾸고 싶을 때는 이 메서드가 더 적절합니다.
```java
String string = null;
Optional<String> optional = Optional.ofNullable(string);
String throwing = optional.orElseThrow(RuntimeException::new); // RuntimeException
```

### ifPresentOrElse (9)

`ifPresentOrElse()`는 Optional에 값이 있을 때와 없을 때의 동작을 한 번에 정의할 수 있는 메서드입니다. 첫 번째 인자는 값이 있을 때 실행할 `Consumer`, 두 번째 인자는 값이 없을 때 실행할 `Runnable`입니다.
```java
Optional<String> hasValue = Optional.of("proper value");
hasValue.ifPresentOrElse(v -> System.out.println("the value is " + v), () -> System.out.println("there is no value")); // the value is proper value

Optional<String> hasNoValue = Optional.empty();
hasNoValue.ifPresentOrElse(v -> System.out.println("the value is " + v), () -> System.out.println("there is no value")); // there is no value
```

### stream (9)

`stream()`은 Optional을 요소 1개짜리 Stream, 혹은 빈 Stream으로 바꿔 줍니다. Optional 하나를 굳이 Stream으로 바꿀 일이 많지 않아 보일 수도 있지만, 다른 Stream 파이프라인에 자연스럽게 합치고 싶을 때 꽤 편합니다.
```java
Optional<String> optional = Optional.of("value");
Stream<String> stream = optional.stream();
```

## 문자열

String API는 특히 Java 11에서 유용한 변화가 많았습니다. 요즘 애플리케이션은 문자열을 다루는 일이 워낙 많아서, 이런 개선은 체감이 큰 편입니다.
### repeat (11)

지정한 횟수만큼 문자열을 반복합니다. 예전처럼 `StringBuilder`나 `StringBuffer`를 따로 쓰지 않아도 간단히 표현할 수 있습니다.
```java
String a10 = "A".repeat(10); // "AAAAAAAAAA"
```

### strip (11)

문자열 양끝 공백을 제거할 때 예전에는 보통 `trim()`을 썼습니다. Java 11에서는 여기에 `strip()`이 추가됐고, 이제는 이쪽이 더 나은 선택인 경우가 많습니다. 가장 큰 차이는 "무엇을 공백으로 보느냐"입니다. `trim()`은 유니코드 공백을 충분히 고려하지 못하지만, `strip()`은 유니코드 기준의 whitespace 전체를 다룹니다. 그래서 전각 공백이나 개행 문자까지 더 자연스럽게 처리할 수 있습니다. 어떤 문자가 whitespace로 분류되는지는 [Character.isWhitespace() JavaDoc](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/Character.html#isWhitespace(char))을 참고하면 됩니다.
```java
String stripped = "\n  hello world  \u2005".strip(); // "hello world"
```

`strip()`은 양쪽 공백을 모두 제거하지만, 한쪽만 지우고 싶다면 `stripLeading()`과 `stripTrailing()`도 사용할 수 있습니다.
```java
String stripLeading = "\n  hello world  \u2005".stripLeading(); // "hello world   "
String stripTrailing = "\n  hello world  \u2005".stripTrailing(); // "\n  hello world"
```

여기까지 설명만으로도 `strip()`을 쓸 이유는 충분하지만, 여기에 성능 이점까지 언급되는 경우가 많습니다. [비교 사례](https://stackoverflow.com/questions/53640184/why-is-string-strip-5-times-faster-than-string-trim-for-blank-string-in-java)에서는 `trim()`보다 더 유리한 결과가 알려져 있어, 가능하면 `trim()`보다 `strip()`을 우선하는 편이 좋습니다.
### isBlank (11)

`isEmpty()`가 이미 있긴 하지만, `isBlank()`와의 관계는 `trim()`과 `strip()`의 차이와 비슷합니다. `isBlank()`는 유니코드 공백까지 고려하므로 더 많은 경우를 자연스럽게 처리할 수 있습니다.
```java
boolean isEmpty = "\n    \u2005".isEmpty(); // false
boolean isBlank = "\n    \u2005".isBlank(); // true
```

### lines (11)

문자열에 개행 코드(`\n`·`\r`·`\r\n`)를 기준으로 나눈 `Stream<String>`을 반환합니다.
```java
String multipleLine = "first\nsecond\nthird";
long lines = multipleLine.lines().filter(String::isBlank).count(); // 3
```

## Predicate.not (11)

`Predicate.not()`은 이름 그대로 Predicate 결과를 부정할 때 쓰는 메서드입니다. 기능 자체는 단순하지만, 람다나 메서드 참조와 조합했을 때 코드가 조금 더 읽기 좋아진다는 장점이 있습니다.
```java
// 부정 조건식을 사용할 때
list.stream()                       
    .filter(m -> !m.isPrepared())
    .collect(Collectors.toList());

// Predicate.not()를 사용할 때
list.stream()                          
    .filter(Predicate.not(Man::isPrepared))
    .collect(Collectors.toList());
```

## 마지막으로

다음 LTS인 Java 17이 나오면, 당시 Java 11을 쓰던 현장도 점차 그쪽으로 옮겨 갈 가능성이 크다고 봤습니다. 12부터 16 사이에도 새로운 API와 JVM 개선이 많이 들어갔지만, 막상 기존 API에 어떤 변화가 있었는지까지 따로 정리해 두기는 쉽지 않습니다. 그래서 Java 17 시점에도 같은 방식으로 12~17 구간을 다시 정리해 보고 싶었습니다. 이런 작업 자체가 공부가 되고, 실제 업무에서 쓸 수 있는 선택지도 늘어나기 때문입니다.

이 글은 그해의 마지막 포스팅이기도 했습니다. 한 해가 꽤 정신없이 지나갔지만, 그래도 꾸준히 글을 남길 수 있었다는 점은 다행이었습니다. 아직 배우는 입장에서 정리한 기록에 가깝지만, 실제로 써 볼 만한 내용을 계속 추려 나가는 데 의미가 있다고 느낍니다.
