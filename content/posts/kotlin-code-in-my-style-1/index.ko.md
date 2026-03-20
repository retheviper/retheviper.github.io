---
title: "Kotlin으로 써 본 코드 1"
date: 2021-03-28
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
translationKey: "posts/kotlin-code-in-my-style-1"
---

지난번에는 Go 이야기를 했지만, 본업은 여전히 Kotlin이라서 Kotlin으로도 업무 코드를 어떻게 정리했는지 적어보려 합니다. 서버 쪽 일을 하다 보면 요구사항의 형태가 꽤 비슷하게 반복되는 경우가 많습니다. 그래서 기능 자체보다도, 같은 문제를 더 읽기 좋은 방식으로 푸는 법을 고민하게 됩니다.

저는 알고리즘에 아주 강한 편은 아니기 때문에, 처음부터 완벽하게 효율적인 코드를 짜는 데에는 한계가 있습니다. 보통은 먼저 동작하는 코드를 만들고, 그다음 리팩터링하면서 조금씩 다듬는 쪽에 가깝습니다.

그래도 좋은 코드를 위해 할 수 있는 일이 아예 없는 것은 아닙니다. Java든 Kotlin이든 이미 잘 만들어진 API가 많고, 버전업과 함께 더 나은 기능도 계속 추가됩니다. 이런 API를 적절히 활용하는 것만으로도 코드의 가독성과 성능을 함께 챙길 수 있습니다.

이 글에서는 Kotlin의 API를 사용해 업무 코드를 어떻게 정리할 수 있었는지, 실제 예시 몇 가지를 중심으로 소개하겠습니다.

## 목록 그룹화

DB에 상품 정보 테이블이 있고, 그 밖에 상품 속성 테이블이나 생산지, 판매점 테이블이 따로 있다고 가정해 보겠습니다. 업무에 따라서는 "판매점별로 어떤 상품이 판매되고 있는지 보고 싶다"거나 "특정 상품 속성에 해당하는 상품만 보고 싶다"는 요구가 생깁니다.

이럴 때 API는 테이블에서 가져온 데이터를 특정 컬럼 기준으로 묶어서 반환해야 합니다. 코드로 보면 `List`로 가져온 데이터를 하나의 속성을 키로 하는 `Map`으로 바꿔서 돌려주는 형태입니다. Java라면 대략 이런 모습이 됩니다.

```java
// DB 데이터 예시
List<User> list = List.of(
        new User("John", 20, "USA", "Programmer"),
        new User("James", 30, "Canada", "Sales"),
        new User("Jack", 35, "UK", "Programmer")
);
// User의 Job을 기준으로 묶는다
Map<String, List<Pair>> map = list.stream()
        .collect(Collectors.groupingBy(User::getJob,
                Collectors.mapping(user -> new Pair(user.getAge(), user.getName()), Collectors.toList())));
// {James=[Pair(first=30, second=Sales), John=[Pair(first=20, second=Programmer), Pair(first=35, second=Writer)}

@Data
@AllArgsConstructor
static class User {
    private String name;
    private int age;
    private String address;
    private String job;
}

@Data
@AllArgsConstructor
static class Pair {
    private Object first;
    private Object second;
}
```

Kotlin에서도 Java API를 그대로 사용할 수 있지만, 가능하면 Kotlin이 제공하는 API로 같은 일을 표현하고 싶습니다. Kotlin의 Collection API는 `Stream`과 `Collector`를 조합한 것에 가까운 처리를 꽤 쉽게 만들 수 있어서, Java용 기능을 하나씩 찾기보다 Kotlin 방식으로 푸는 편이 더 자연스러운 경우가 많습니다. 특히 `Collectors.groupingBy()`와 `Collectors.mapping()`에 해당하는 처리를 `groupBy()` 하나로 정리할 수 있다는 점이 좋습니다. 그래서 코드를 Kotlin으로 바꾸면 다음과 같습니다.

```kotlin
// DB 데이터 예시
val list = listOf(
    User("John", 20, "USA", "Programmer"),
    User("James", 30, "Canada", "Sales"),
    User("Jack", 35, "UK", "Programmer")
  )
// Job을 기준으로 Map<String, List<Pair<Int, String>>>로 묶는다
val map = list.groupBy({ it.job }, { it.age to it.name })
// {Programmer=[(20, John), (35, Jack), Sales=[(30, James)}

data class User(
    val name: String,
    val age: Int,
    val address: String,
    val job: String
)
```

## Map의 값만 변경

여기에 한 단계 더 조건이 붙는 경우도 있습니다. 예를 들어 금액 계산을 생각해 보면, 직원이 여러 안건의 대금을 받는 상황에서 같은 안건의 금액은 하나로 합쳐서 보고 싶을 수 있습니다. 이럴 때는 먼저 데이터를 묶은 뒤, 중복된 항목의 값만 다시 합산하면 됩니다.

그런데 이런 처리를 그룹핑 단계에서 한 번에 넣으려면 코드가 꽤 복잡해질 수 있습니다. 그래서 여기서는 일단 `List`를 `Map`으로 만든 뒤, 그 결과에 추가 처리를 더하는 방식을 사용합니다.

Kotlin의 `Map`에는 `mapKeys()`와 `mapValues()` 같은 함수가 있어서 필요한 부분만 바꿀 수 있습니다. 이번에는 값만 바꾸면 되므로 `mapValues()`를 쓰는 편이 의도도 분명하고 낭비도 적습니다. 코드는 다음과 같습니다.

```kotlin
data class User(val name: String, val id: Int, val amount: Int)

// DB 데이터 예시
val list = listOf(
    User("A", 1, 1000),
    User("A", 1, 2000),
    User("A", 2, 4000),
    User("B", 3, 5000)
)
// name으로 묶은 뒤 중복되는 id를 하나로 합친다(amount를 합산한다)
val map = list.groupBy({ it.name }, { it.id to it.amount })
    .mapValues {
        // id로 그룹핑
        it.value.groupBy { pair -> pair.first }
        // key는 그대로 두고 value만 합산한다
            .map { map -> map.key to map.value.sumBy { pair -> pair.second } }
    }
// {A=[(1, 3000), (2, 4000), B=[(3, 5000)}
```

`List`를 `Map`으로 묶는 또 다른 방법은 `groupingBy()`입니다. 이 함수를 쓰면 Collection이 [Grouping](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-grouping) 객체로 바뀌고, 여기에 `aggregate()`, `reduce()`, `fold()`, `eachCount()` 같은 함수를 이어서 사용할 수 있습니다. 위의 코드를 `Grouping` 버전으로 바꾸면 다음과 같습니다.

```kotlin
// Grouping의 aggregate를 이용해 Map으로 바꾼 뒤 value를 처리한다
val map = list.groupingBy { it.name }
    .aggregate { _, accumulator: MutableList<Pair<Int, Int>>?, element, first ->
        // 새로운 키면 MutableList를 만든다
        if (first)
            mutableListOf(element.id to element.amount)
        // 그렇지 않으면 기존 List에 요소를 추가한다
        else
            accumulator?.apply { add(element.id to element.amount) }
    }.mapValues {
        it.value?.groupBy { pair -> pair.first }
            ?.map { pair -> pair.key to pair.value.sumBy { pair -> pair.second } }
    }
```

언뜻 보면 `groupingBy()`가 더 복잡해 보이지만, `accumulator`를 써서 값을 누적할 수 있다는 장점이 있습니다. 상황에 따라서는 충분히 검토할 만합니다.

## Map을 사용한 캐시

DB 조회가 자주 일어나지만 데이터 자체는 자주 바뀌지 않는 경우에는, 애플리케이션 내부에 캐시를 두는 편이 좋습니다. 이때는 파라미터를 키로 하는 `Map`을 준비해 두고, 키가 없을 때만 DB를 조회한 뒤 `Map`에 넣으면 됩니다. Java 8부터는 `computeIfAbsent()`가 있어서 이런 패턴을 쉽게 구현할 수 있습니다. 예를 들면 다음과 같습니다.

```java
// DB 데이터 예시
List<String> list = List.of("A", "B", "C");
// 캐시용 Map
Map<String, Boolean> map = new ConcurrentHashMap<>();
// 파라미터
String element = "A";

// 캐시에 파라미터가 없으면 DB 데이터를 참조해 추가한 뒤 반환한다
Boolean exists = map.computeIfAbsent(element, key -> list.contains(element));
// Method Reference를 사용하는 예
exists = map.computeIfAbsent(element, list::contains);
```

Java에서 제공하는 기능이니 Kotlin에서도 같은 방식으로 쓸 수 있습니다. 다만 Kotlin에서는 [Lambda와 Method Reference의 표기 방식이 조금 다르기](https://kotlinlang.org/docs/lambdas.html#instantiating-a-function-type) 때문에 그 부분만 주의하면 됩니다. Java식 표기에 익숙한 사람이라면 처음엔 잠깐 헷갈릴 수도 있습니다.

```kotlin
// DB 데이터 예시
val list = listOf("A", "B", "C")
// 캐시용 Map
val map = ConcurrentHashMap<String, Boolean>()
// 파라미터
val element = "A"

// Lambda인 경우
var exists = map.computeIfAbsent(element) { list.contains(element) } // false
// Method Reference인 경우
exists = map.computeIfAbsent(element, list::contains)
```

비슷한 메서드로 `putIfAbsent()`도 있지만, `computeIfAbsent()`는 키가 없을 때만 이후 처리가 실행된다는 점이 다릅니다. 캐시 용도라면 보통 `computeIfAbsent()`가 더 적합합니다.

## 마지막으로

이번 편에서는 제가 실제로 업무에서 써 본 패턴 몇 가지를 정리해 봤습니다. Kotlin으로 옮겨온 지 오래되지 않았을 때라 지금 다시 보면 더 나은 방법도 있을 수 있지만, 다른 언어와 비교하면서 표준 API를 어떻게 활용할지 고민하는 과정 자체가 꽤 재미있었습니다.

이 시리즈는 앞으로도 같은 방식으로 이어 가 볼 생각입니다. Kotlin다운 코드를 찾는 과정은 생각보다 끝이 없고, 그만큼 배울 거리도 계속 생깁니다.
