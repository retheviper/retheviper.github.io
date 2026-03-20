---
title: "다시 보는 컬렉션 루프"
date: 2020-11-30
categories: 
  - java
image: "../../images/java.webp"
tags:
  - stream
  - java
  - collection
  - foreach
translationKey: "posts/java-collection-loop"
---

Java는 원래 절차형 언어에 가깝지만, 함수형 언어의 특징도 꽤 자연스럽게 받아들여 두 성격이 함께 공존하는 언어가 됐습니다. 저도 함수형 프로그래밍에 관심이 있어서 Java에서도 Stream이나 Lambda를 자주 쓰고, Kotlin이나 Spring WebFlux로도 이것저것 실험해 보고 있습니다.
다만 Java 8 이후로 계속 따라다닌 질문이 하나 있습니다. `Stream`이 과연 모든 `for` 루프를 대체할 수 있느냐는 것입니다. 보통은 성능, 가독성, 디버깅 난이도 때문에 전통적인 루프를 완전히 버리지는 못한다고 말합니다. 내부 처리가 조금 더 복잡하고, 예외가 났을 때 원인을 좁히기 어렵고, [method chaining](https://en.wikipedia.org/wiki/Method_chaining)이나 lambda에 익숙하지 않은 사람도 많기 때문입니다.
그래서 저도 평소에는 `List`는 `ArrayList`, 루프는 확장 `for`문, 요소를 가공하거나 새 인스턴스를 만들어야 할 때만 `Stream`을 쓰는 식으로 지내 왔습니다. 그런데 문득 이 기준이 정말 맞는지 다시 확인해 보고 싶었습니다. Java도 이미 16까지 올라왔고, 함수형 스타일을 더 적극적으로 써도 되는 시점인지, 아니면 예전 상식이 아직도 유효한지 직접 보고 싶었습니다.
그래서 이번에는 가벼운 벤치마크를 곁들여 여러 루프 방식을 비교해 보았습니다. 사실은 벤치마크를 돌려 보고 싶었던 마음이 더 컸습니다.
## 루프 방법

이번 글에서는 Collection과 배열을 다루는 네 가지 루프 방식을 중심으로 이야기해 보겠습니다. 많은 경우 아래 같은 기준으로 루프를 선택하지 않을까 생각합니다.

| 종류 | 사용하는 상황 |
|---|---|
| For | 인덱스가 필요할 때 |
| 확장 For | 다른 방법을 쓸 필요가 없을 때 |
| Iterator | 기본적으로 잘 쓰지 않음 |
| forEach() | 기본적으로 잘 쓰지 않음 |

이 기준의 중심에는 결국 성능이 있습니다. 가독성 같은 다른 요소도 물론 중요하지만, 성능이 우선되는 경우가 가장 많습니다. 다른 요소는 튜닝이 어렵거나 아예 바꾸기 힘든 경우도 많고, 성능은 리팩토링으로 눈에 보이게 개선할 수 있는 대표적인 항목이기 때문입니다. 같은 처리를 한다면 더 빠른 쪽이 좋은 건 당연합니다.

제가 루프를 처음 배울 때는 전통적인 `For`문과 `While`문이 기본이었고, 나중에는 Collection이나 배열을 돌릴 때는 확장 `For`문을 쓰는 게 좋다는 식으로 배웠습니다. 그때의 근거도 "For와 확장 For는 성능 차이가 크지 않고, 확장 For는 요소 수만큼 자연스럽게 돈다"는 설명이었습니다. 꽤 납득이 가는 이야기였고, 저도 그 말을 믿고 계속 확장 `For`문을 써 왔습니다.

하지만 실제로 어떤 차이가 있는지는 별도로 검증해 본 적이 많지 않았습니다. 검색해 보면 확장 `For`문은 기존 루프를 더 쓰기 쉽게 만든 형태라거나, Iterator의 문법적 설탕이라는 설명이 많지만, 정작 어떤 방식이 가장 빠른지는 직접 봐야 한다고 생각했습니다. `Stream`과 `forEach()`도 마찬가지입니다. Java에 도입된 뒤에도 성능이 느리다는 평이 많고, 굳이 써야 할 이유를 모르겠다는 의견이나 어렵다는 의견도 있습니다. Java 8 시절부터 성능 비교가 많이 이루어졌고, 적어도 성능 면에서는 기존 방식이 유리하다는 결론이 자주 나왔습니다. Java 버전이 16까지 올라온 지금도 그 인식은 크게 바뀌지 않았습니다.

하지만 "누군가 그렇게 말했으니 그렇다"고 믿는 건 좋은 태도는 아닙니다. Java는 계속 발전하고 있고, JVM이나 컴파일러 차원에서 보이지 않는 개선이 있었을 수도 있습니다. 함수형 스타일에 익숙한지 아닌지는 개인 차이지만, 성능까지 좋아졌다면 굳이 새 방식을 피할 이유도 없습니다. 게다가 정말로 확장 `For`문이 모든 경우에 최선인지도 직접 확인해 볼 필요가 있습니다.

그래서 우선 비교 대상으로 삼을 네 가지 루프 방식을 소개하고, 이어서 벤치마크 결과를 살펴보겠습니다.
### For 문

우선은 전통적인 형태의 `For`문입니다. 일부에서는 `c-style`이라고도 부릅니다. 가장 기본적인 방식이라 익숙하지만, 한편으로는 다소 낡은 느낌도 있습니다. 최근의 이른바 모던한 언어에서는 이런 형태의 루프를 아예 쓸 수 없는 경우도 있습니다. 기본 형태는 다음과 같습니다.
```java
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}
```

마이크로 최적화 차원에서 루프 대상 Collection이나 배열의 길이를 미리 변수에 담아 두는 경우도 있습니다. 이렇게 하면 매 반복마다 크기를 다시 확인하지 않아도 되기 때문에 조금이나마 유리하다고들 합니다. 물론 이 정도는 컴파일러가 최적화해 줄 수도 있습니다.
```java
int size = list.size();
for (int i = 0; i < size; i++) {
    list.get(i);
}
```

전통적인 `for`문의 장점은 인덱스를 기준으로 제어할 수 있다는 점입니다. 그래서 인덱스가 꼭 필요한 상황에서는 여전히 가장 직접적인 선택지입니다. 예를 들면 아래와 같습니다.
```java
// 짝수 인덱스만 처리하고 싶을 때
for (int i = 0; i < list.size(); i += 2) {
    System.out.println(list.get(i));
}

// 조건과 일치하는 요소의 인덱스를 알고 싶을 때
for (int i = 0; i < list.size(); i++) {
    if (list.get(i).length() == 10) {
        System.out.println(i);
    }
}

// 앞뒤 요소와 비교하고 싶을 때
for (int i = 1; i < list.size(); i += 2) {
    System.out.println("인덱스 " + (i - 1) + "의 길이: " + list.get(i - 1).length() + ", 인덱스 " + i + "의 길이: " + list.get(i).length());
}
```

다만 인덱스를 직접 다루는 만큼 범위를 잘못 잡기 쉽다는 문제가 있습니다. `i - 1` 같은 접근을 잘못 쓰면 바로 예외로 이어질 수 있고, 숫자 상수가 많아지면 [매직 넘버](https://ko.wikipedia.org/wiki/%EB%A7%A4%EC%A7%81_%EB%84%98%EB%B2%84) 때문에 가독성도 떨어집니다. 그래서 인덱스 기반 처리가 꼭 필요할 때만 조심해서 쓰는 편이 낫습니다.
### 확장 For 문

이른바 `for-each`문입니다. Collection이나 배열의 모든 요소를 순서대로 처리할 때는 가장 읽기 쉽고 안전한 방식에 가깝습니다.
```java
for (String element : list) {
    System.out.println(element);
}
```

최근에는 Java뿐 아니라 다른 언어에서도 이런 방식이 기본에 가깝습니다. 인덱스를 직접 움직이는 것보다, "이 컬렉션의 각 요소를 순회한다"는 의도가 더 분명하게 드러나기 때문입니다.
물론 단점도 있습니다. 확장 `for`문 안에서는 인덱스를 바로 쓸 수 없습니다. 꼭 인덱스가 필요하다면 루프 밖에서 별도 정수를 증가시키거나, `indexOf()`, `Collections.binarySearch()` 같은 방법을 고려할 수 있습니다.
```java
// 상수를 이용하는 방법
int i = 0;
for (String element : list) {
    System.out.println(element + "의 인덱스: " + i);
    i++;
}

// indexOf()를 사용하는 경우
for (String element : list) {
    System.out.println(element + "의 인덱스: " + list.indexOf(element));
}

// Collections.binarySearch()를 사용하는 경우
for (String element : list) {
    System.out.println(element + "의 인덱스: " + Collections.binarySearch(values, value));
}
```

다만 루프 안에서 `indexOf()`를 매번 호출하는 것은 좋은 선택이 아닙니다. 결국 컬렉션 내부를 다시 순회하게 되므로 사실상 이중 루프가 됩니다. 인덱스가 정말 필요하다면 차라리 카운터 변수를 두거나 전통적인 `for`문을 쓰는 편이 낫습니다.
```java
// ArrayList.indexOf()
public int indexOf(Object o) {
    return indexOfRange(o, 0, size);
}

int indexOfRange(Object o, int start, int end) {
    Object[] es = elementData;
    if (o == null) {
        for (int i = start; i < end; i++) {
            if (es[i] == null) {
                return i;
            }
        }
    } else {
        for (int i = start; i < end; i++) {
            if (o.equals(es[i])) {
                return i;
            }
        }
    }
    return -1;
}
```

`Collections`의 `binarySearch()`을 이용하는 경우도, 결국 루프하면서 인덱스를 찾는 것은 변하지 않으므로 주의를. 다음은 그 구현입니다.
```java
// Collections.binarySearch()
public static <T> int binarySearch(List<? extends Comparable<? super T>> list, T key) {
    if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
        return Collections.indexedBinarySearch(list, key);
    else
        return Collections.iteratorBinarySearch(list, key);
}

private static <T> int indexedBinarySearch(List<? extends Comparable<? super T>> list, T key) {
    int low = 0;
    int high = list.size()-1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        Comparable<? super T> midVal = list.get(mid);
        int cmp = midVal.compareTo(key);

        if (cmp < 0)
            low = mid + 1;
        else if (cmp > 0)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found
}

private static <T> int iteratorBinarySearch(List<? extends Comparable<? super T>> list, T key) {
    int low = 0;
    int high = list.size()-1;
    ListIterator<? extends Comparable<? super T>> i = list.listIterator();

    while (low <= high) {
        int mid = (low + high) >>> 1;
        Comparable<? super T> midVal = get(i, mid);
        int cmp = midVal.compareTo(key);

        if (cmp < 0)
            low = mid + 1;
        else if (cmp > 0)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found
}
```

### Iterator

Iterator는 개인적으로도 그다지 손이 가지 않는 방식입니다. 컬렉션 자체보다 Iterator가 전면에 나오기 때문에, 코드에서 "무엇을 순회하는가"보다 "어떻게 순회하는가"가 더 두드러져 보이는 느낌이 있습니다. 확장 `for`문보다 의도가 덜 직접적으로 보인다는 뜻입니다.
어쨌든 Iterator는 `for`와 `while` 둘 다로 사용할 수 있습니다.
```java
// For를 사용하는 경우
for (Iterator iterator = values.iterator(); iterator.hasNext(); ) {
    System.out.println(iterator.next());
}

// While를 사용하는 경우
Iterator iterator = values.iterator();
while (iterator.hasNext()) {
  System.out.println(iterator.next());
}
```

Iterator의 문제는 사용법이 직관적이지 않을 때가 있다는 점입니다. 예를 들어 아래 코드는 `getFoo()`와 `getBar()`가 같은 객체에 대해 호출되는 것처럼 보이기 쉽지만, 실제로는 `next()`가 두 번 호출됩니다.
```java
for (Iterator iterator = list.iterator(); iterator.hasNext(); ) {
    System.out.println(iterator.next().getFoo());
    System.out.println(iterator.next().getBar()); // 주의!
}
```

흥미롭게도 확장 `for`문의 바이트코드는 결국 Iterator를 사용하는 형태로 바뀝니다. 그런 의미에서 확장 `for`문은 Iterator를 더 안전하고 읽기 쉽게 감싼 문법이라고 볼 수 있습니다.
### forEach()

`forEach()`는 비교적 현대적인 스타일의 반복 방식입니다. 확장 `for`문과 크게 다르지는 않지만, 람다나 메서드 참조를 자연스럽게 쓸 수 있다는 장점이 있습니다. 처리 범위가 블록 안에 깔끔하게 묶인다는 점도 나쁘지 않습니다.
```java
list.forEach(System.out::println)
```

구현도 기본적으로는 확장 `for`문 안에서 lambda를 실행하는 단순한 형태입니다. 그래서 직관적으로는 확장 `for`문보다 조금 불리할 수 있어 보입니다. 아래는 `Iterable` 쪽 구현입니다.
```java
// Iterable의 forEach()
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```

다만 `ArrayList`는 구현이 꽤 다릅니다. 그래서 성능도 꽤 달라질 가능성이 있습니다. 아래는 그 구현입니다.
```java
// ArrayList의 forEach()
public void forEach(Consumer<? super E> action) {
    Objects.requireNonNull(action);
    final int expectedModCount = modCount;
    final Object[] es = elementData;
    final int size = this.size;
    for (int i = 0; modCount == expectedModCount && i < size; i++)
        action.accept(elementAt(es, i));
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}

static <E> E elementAt(Object[] es, int index) {
    return (E) es[index];
}
```

## 벤치 마크로 확인해 보면

이번에도 JMH로 간단한 벤치마크를 만들어 봤습니다. 처음에는 `static final` 필드로 두면 모든 벤치마크가 같은 객체를 공유할 거라고 생각했는데, 실제로는 그렇지 않았습니다. 그래서 이번에는 `@Setup`을 써서 필드를 명시적으로 초기화했습니다.
```java
@State(Scope.Thread)
public class LoopTest {

    private List<String> values;

    @Setup
    public void init() {
        final DecimalFormat format = new DecimalFormat("0000000");
        values = IntStream.rangeClosed(0, 9999999).mapToObj(format::format).collect(Collectors.toList());
    }

    @Benchmark
    public void indexLoop(Blackhole bh) {
        final int length = values.size();
        for (int i = 0; i < length; i++) {
            bh.consume(values.get(i));
        }
    }

    @Benchmark
    public void iteratorLoopFor(Blackhole bh) {
        for (Iterator iterator = values.iterator(); iterator.hasNext(); ) {
            bh.consume(iterator.next());
        }
    }

    @Benchmark
    public void iteratorLoopWhile(Blackhole bh) {
        final Iterator iterator = values.iterator();
        while (iterator.hasNext()) {
            bh.consume(iterator.next());
        }
    }

    @Benchmark
    public void extendedLoop(Blackhole bh) {
        for (String value : values) {
            bh.consume(value);
        }
    }

    @Benchmark
    public void forEachLoop(Blackhole bh) {
        values.forEach(bh::consume);
    }
}
```

그리고 벤치마크 결과는 다음과 같습니다.
```dos
Benchmark                    Mode  Cnt   Score   Error  Units
LoopTest.indexLoop          thrpt   25  27.737 ± 0.475  ops/s
LoopTest.iteratorLoopFor    thrpt   25  26.968 ± 0.556  ops/s
LoopTest.iteratorLoopWhile  thrpt   25  27.250 ± 0.557  ops/s
LoopTest.extendedLoop       thrpt   25  13.186 ± 0.152  ops/s
LoopTest.forEachLoop        thrpt   25  12.479 ± 0.104  ops/s
```

역시 네 가지 루프는 서로 다른 결과를 보여 줍니다. 적어도 여기서는 전통적인 `For`문이 성능 면에서 가장 유리해 보입니다. "확장 `For`문을 써도 성능 차이는 크지 않다"는 말이 다시 보일 정도였습니다.

그렇다고 해서 바로 "성능이 좋은 쪽만 고르면 된다"는 결론을 내려도 될까요?
## 생각하고 싶은 것

결과가 같다면 더 빠른 쪽을 고르고 싶어지는 건 자연스럽습니다. 특히 업무 시스템에서는 성능이 곧 비용과 연결되기도 합니다. 하지만 실제 애플리케이션 개발에서 고려할 요소는 성능만이 아닙니다. 극단적으로 성능만 따진다면 웹 애플리케이션도 C나 C++로 작성해야겠지만, 그러면 생산성과 유지보수성이 크게 떨어질 수 있습니다. 읽기 어려운 고성능 코드가 결국 더 큰 비용을 부르는 경우도 흔합니다.
그래서 루프를 고를 때도 성능 외에 가독성, 오류 발생 가능성, 처리 대상의 특성까지 함께 봐야 합니다. 이제는 그런 관점에서 네 가지 루프를 다시 비교해 보겠습니다.
### 가독성과 오류 발생 가능성 측면에서 생각

확장 `for`문이나 `forEach()`는 기본적으로 요소 자체에 집중하게 만듭니다. 반대로 전통적인 `for`문이나 Iterator는 인덱스나 현재 위치를 더 직접적으로 제어할 수 있습니다. 겉보기에는 조건에 맞는 요소만 골라 처리할 때 후자가 더 강력해 보일 수 있습니다.
하지만 관점을 바꾸면, 원본 객체를 직접 수정할 수 있다는 점은 오히려 단점이 되기도 합니다. 사이드 이펙트를 줄이려면 원본을 바꾸는 대신 조건에 맞는 요소만 담은 새 컬렉션을 만드는 편이 더 안전할 때가 많습니다. 이런 의도를 코드로 드러내기에는 확장 `for`문이나 `Stream` 쪽이 더 읽기 쉽습니다. 예를 들면 다음과 같습니다.
```java
// 원본 리스트가 변해 버리는 방식
public void filterFor(List<String> list) {
    for (int i = 0; i < list.size(); i++) {
        if (list.get(i).length() > 10) {
            list.remove(i);
        }
    }
}

// 원본 리스트에 영향이 없는 방식 - For문
public List<String> filterFor(List<String> list) {
    List<String> result = new ArrayList<>();
    for (int i = 0; i < list.size(); i++) {
        String element = list.get(i);
        if (element.length() <= 10) {
            result.add(element);
        }
    }
    return result;
}

// 원본 리스트에 영향이 없는 방식 - 확장 For문
public List<String> filterForEach(List<String> list) {
    List<String> result = new ArrayList<>();
    for (String element : list) {
        if (element.length() <= 10) {
            result.add(element);
        }
    }
    return result;
}

// 원본 리스트에 영향이 없는 방식 - Stream.forEach()
public List<String> filterStream(List<String> list) {
    return list.stream().filter(element -> element.length() <= 10).collect(Collectors.toList());
}
```

좋은 코드는 결국 짧기만 한 코드가 아니라, 읽는 사람이 쉽게 이해하고 실수하기 어려운 코드라고 생각합니다. 그런 기준에서 보면 전통적인 `for`문과 Iterator는 꼭 필요한 경우에만 쓰고, 기본값으로는 더 안전한 방식을 선택하는 편이 낫습니다.
### 처리 능력 측면에서 생각

처리 능력이라는 관점은 결국 성능과도 이어집니다. 다만 여기서 핵심은 "어떤 컬렉션 구현이 들어와도 크게 흔들리지 않는 방식이 무엇인가"입니다.
예를 들어 `List`를 인자로 받아 루프를 도는 메소드를 만든다고 해 봅시다. 이때 어떤 루프를 고를지는, 그 `List`가 실제로 어떤 구현 클래스인지 모른다는 사실까지 함께 고려해야 합니다. `List`는 인터페이스이기 때문에, 실제로는 [ArrayList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/ArrayList.html)일 수도 있고 [LinkedList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/LinkedList.html)일 수도 있습니다. 더 나아가 다른 구현이 들어올 가능성도 있습니다.
인터페이스를 기준으로 코드를 작성한다는 것은, 결과 자체는 구현 클래스가 달라도 유지된다는 뜻입니다. 하지만 성능까지 같다는 뜻은 아닙니다. `List` 구현이 여러 개 존재하는 이유 자체가 사용 목적에 따라 서로 다른 특성을 가지기 때문입니다. 가장 흔한 구현은 ArrayList지만, 경우에 따라서는 LinkedList 같은 다른 구현이 들어올 수도 있습니다. 그렇다면 ArrayList에서 빠른 방식이 LinkedList에서도 그대로 빠를 것이라고 가정하는 건 위험합니다.
위에서 실시한 벤치마크만 보고 성능은 절대 이것이 유리하다고 말할 수 없는 이유가 여기에 있습니다. 왜냐하면 테스트와 같은 데이터를 `Collectors.toList()`을 사용하여 List로 작성하고 있습니다.
```java
public static <T> Collector<T, ?, List<T>> toList() {
    return new CollectorImpl<>((Supplier<List<T>>) ArrayList::new, List::add,
                                (left, right) -> { left.addAll(right); return left; },
                                CH_ID);
}
```

그래서 다른 구현 클래스에서도 벤치마크를 해 보기로 했습니다. 물론 모든 `List` 구현을 다 테스트하는 것은 현실적이지 않습니다. 여기서는 ArrayList와 성격이 확실히 다른 반례 하나면 충분하다고 보고 `LinkedList`를 골랐습니다.
다행히 `Collectors.toCollection()`을 사용하면 컬렉션 구현을 쉽게 바꿀 수 있습니다. 그래서 기존 벤치마크 코드에서 아래 한 줄만 바꾸면 됩니다.
```java
// LinkedList의 경우
values = IntStream.rangeClosed(0, 9999999).mapToObj(format::format).collect(Collectors.toCollection(LinkedList::new));
```

LinkedList의 경우 요소 수가 증가하면 성능이 급격히 떨어지는 경향이 있습니다. 그래서 ArrayList 때보다 요소수는 2자리 정도 줄여 벤치마크를 실시했습니다. 결과는 다음과 같습니다.
```bash
Benchmark                    Mode  Cnt    Score    Error  Units
LoopTest.indexLoop          thrpt   25    0.084 ±  0.005  ops/s
LoopTest.iteratorLoopFor    thrpt   25  854.459 ± 36.771  ops/s
LoopTest.iteratorLoopWhile  thrpt   25  839.233 ± 18.142  ops/s
LoopTest.extendedLoop       thrpt   25  659.999 ± 47.702  ops/s
LoopTest.forEachLoop        thrpt   25  780.463 ± 78.591  ops/s
```

결과는 ArrayList 때와 거의 정반대였습니다. 특히 인덱스 기반 루프는 사실상 쓰기 어려울 정도로 느렸고, `forEach()`가 확장 `for`문보다 좋은 결과를 보인 것도 꽤 의외였습니다. 이 수치를 절대 기준으로 삼을 수는 없지만, 적어도 "ArrayList에서 빠르니까 모든 List에서도 빠를 것"이라고 단정하는 건 위험하다는 점은 분명합니다. 결국 중요한 것은 특정 구현 하나에서 가장 빠른 방식이 아니라, 다양한 구현에서 평균적으로 무난한 방식을 고르는 일입니다. 그런 기준에서는 확장 `for`문이 꽤 안정적인 선택처럼 보입니다.
## 마지막으로

모든 상황에서 완벽한 코드를 처음부터 쓰기는 어렵습니다. 그래서 예전에 작성한 코드를 다시 읽어 보면, 그때는 몰랐던 중복이나 불필요한 객체 생성이 눈에 들어오는 경우가 많습니다. 저도 가끔 옛 코드를 보면 꽤 놀랍니다.
그래서 이렇게 기본적인 주제라도 한 번 다시 검토해 보는 일은 의미가 있습니다. 루프는 너무 익숙해서 별 고민 없이 쓰기 쉽지만, 막상 비교해 보면 선택 기준이 생각보다 단순하지 않습니다. 그런 점을 직접 확인해 보는 데 벤치마크는 꽤 재미있고 유용한 도구였습니다.
루프처럼 익숙한 주제일수록, 한 번쯤 이렇게 다시 확인해 보는 과정이 꽤 도움이 됩니다.
