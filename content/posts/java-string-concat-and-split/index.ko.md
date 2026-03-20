---
title: "다시 보는 문자열 연결과 분할"
date: 2021-02-08
categories: 
  - java
image: "../../images/java.webp"
tags:
  - java
  - string
translationKey: "posts/java-string-concat-and-split"
---

이번 글은 세 번째 "더 많은 시리즈"이고, 이번에도 벤치마크를 곁들였습니다. 주제는 Java의 문자열 처리, 그중에서도 `연결(join)`과 `분할(split)`입니다. 처음 궁금했던 건 단순했습니다. 문자열 분할은 보통 `String.split()` 하나로 끝나는데, 연결은 `String.join()`이나 `Collectors.joining()`처럼 방법이 여러 가지라는 점이었습니다. 같은 일을 여러 API로 할 수 있다고 해서 전부 단순한 문법 설탕이라고 볼 수는 없습니다. 실제 구현이 꽤 다른 경우도 있기 때문입니다. 특히 Java처럼 역사가 긴 언어에서는 이런 경우가 더 자주 보입니다.

게다가 겉보기에는 비슷한 API라도, 앞뒤 코드의 맥락이나 가독성까지 생각하면 선택 기준이 달라질 때가 있습니다. 예전에 다뤘던 InputStream의 `transferTo()`도 그런 예였습니다. 그래서 가능하면 API 사용법만 보지 말고, 내부 구현도 한 번쯤 확인해 보는 편이 결국 더 좋은 코드를 쓰는 데 도움이 됩니다.

이제 본론으로 들어가 보겠습니다. 먼저 문자열 연결부터 보겠습니다.

## Concatenating

문자열 연결이라고 해도 경우의 수는 꽤 많습니다. `String.concat()`이나 `String.format()` 같은 방법까지 전부 한 번에 비교하기는 어렵기 때문에, 이번 글에서는 범위를 하나로 좁히겠습니다. 즉, `문자열 배열이나 컬렉션을 구분자로 이어 붙여 하나의 문자열로 만든다`는 경우만 다룹니다.

Java에서 구분자를 넣어 문자열을 이어 붙이는 방법으로는 보통 다음과 같은 것들을 생각할 수 있습니다. 각각의 API를 사용법과 특징 기준으로 먼저 비교하고, 그다음 평소처럼 벤치마크로 성능도 확인해 보겠습니다. 참고로 `+` 연산으로 문자열을 잇는 방식은 이번 비교 대상에서 제외했습니다.

- `String.join()`
- `StringJoiner`
- `StringBuffer`
- `StringBuilder`
- `Collectors.joining()`

### StringBuffer ||StringBuilder

단순히 여러 문자열을 이어 붙이는 경우도 있지만, 실제로는 "특정 구분자를 끼워 넣는다"는 조건이 붙는 경우가 훨씬 많습니다. 예를 들어 대시(`-`), 언더스코어(`_`), 쉼표(`,`) 같은 문자를 사이사이에 넣는 식입니다. 이런 경우에는 `StringBuffer`나 `StringBuilder`가 오히려 조금 불편합니다. 마지막 요소 뒤에는 구분자가 붙지 않도록 따로 신경 써야 하기 때문입니다. 아래 예제가 그런 경우입니다.

```java
// StringBuffer를 사용하는 예
List<String> list = List.of("A", "B", "C");
String delimiter = ", ";

StringBuffer buffer = new StringBuffer();
// List 요소와 구분 문자를 더한다
for (String string : list) {
    buffer.append(string);
    buffer.append(delimiter);
}

String result = buffer.toString(); // A, B, C,
```

문자열 끝에 구분자가 붙지 않게 하려면 보통 이렇게 작성해야 합니다.

```java
List<String> list = List.of("A", "B", "C");
String delimiter = ", ";
int limit = list.size() - 1;

StringBuffer buffer = new StringBuffer();
// List 요소와 구분 문자를 더한다(마지막 인덱스 전까지)
for (int i = 0; i < limit; i++) {
    buffer.append(list.get(i));
    buffer.append(delimiter);
}
// 마지막 요소를 더한다
buffer.append(list.get(limit));

String result = buffer.toString(); // A, B, C
```

반면 다른 방법(`String.join()`, `StringJoiner`, `Collectors.joining()`)은 마지막 요소 뒤에 구분자가 붙지 않도록 따로 신경 쓸 필요가 없습니다. 그래서 코드가 더 단순해집니다. 결국 `StringBuffer`나 `StringBuilder`는 "특정 규칙에 따라 여러 문자열을 이어 붙인다"는 상황에서는 가독성 면에서 그다지 좋은 선택이 아닙니다.

### StringJoiner

`StringBuffer`와 `StringBuilder`는 루프 안에서 직접 문자열을 조립하므로, 조건 분기 같은 다른 처리가 필요할 때 더 유연해 보일 수 있습니다. 그래도 구분자를 붙이는 목적이라면 `StringJoiner` 쪽이 대체로 더 낫습니다. 사용법도 크게 다르지 않고, 무엇보다 마지막 구분자를 따로 처리할 필요가 없기 때문입니다. 아래는 가장 기본적인 예입니다.

```java
List<String> list = List.of("A", "B", "C");
// 구분 문자를 지정해 인스턴스를 만든다
StringJoiner joiner = new StringJoiner(", ");
// 이후에는 요소를 계속 더한다
for (String string : list) {
    joiner.add(string);
}
String result = joiner.toString(); // A, B, C
```

또한 `StringJoiner`을 사용하면 Prefix와 Suffix를 지정할 수 있습니다. 이것을 지정했을 경우, 문자열의 선두와 말미에 지정한 Prefix와 Suffix가 붙게 됩니다.

```java
List<String> list = List.of("A", "B", "C");
// 구분 문자와 Prefix, Suffix까지 지정한다
StringJoiner joiner = new StringJoiner(", ", "[", "]");
for (String string : list) {
    joiner.add(string);
}
String result = joiner.toString(); // [A, B, C]
```

사용법만 봐도, 구분자를 포함해 문자열을 연결할 때는 `StringBuffer`나 `StringBuilder`보다 `StringJoiner`가 더 간단합니다.

#### 번외: StringJoiner 구현

그다음에는 `StringJoiner` 구현도 간단히 볼 만합니다. 먼저 `add()`를 보면, 느낌상 `ArrayList` 구현과 비슷한 면이 있습니다. 내부에 `String[]`을 들고 있고, `add()`가 호출될 때마다 필요하면 더 큰 배열로 복사해 가는 구조입니다.

```java
public StringJoiner add(CharSequence newElement) {
    final String elt = String.valueOf(newElement);
    if (elts == null) {
        elts = new String[8];
    } else {
        if (size == elts.length)
            elts = Arrays.copyOf(elts, 2 * size);
        len += delimiter.length();
    }
    len += elt.length();
    elts[size++] = elt;
    return this;
}
```

`toString()`에서는 내부 `String[]`을 순회하면서 구분자를 끼워 넣어 최종 문자열을 만듭니다. 흥미로운 점은 중간에 `char[]`를 이용해 새 `String` 인스턴스를 만든다는 것입니다. 구현도 나름 성능을 신경 쓴 흔적이 보입니다.

```java
public String toString() {
    final String[] elts = this.elts;
    if (elts == null && emptyValue != null) {
        return emptyValue;
    }
    final int size = this.size;
    final int addLen = prefix.length() + suffix.length();
    if (addLen == 0) {
        compactElts();
        return size == 0 ? "" : elts[0];
    }
    final String delimiter = this.delimiter;
    final char[] chars = new char[len + addLen];
    int k = getChars(prefix, chars, 0);
    if (size > 0) {
        k += getChars(elts[0], chars, k);
        for (int i = 1; i < size; i++) {
            k += getChars(delimiter, chars, k);
            k += getChars(elts[i], chars, k);
        }
    }
    k += getChars(suffix, chars, k);
    return new String(chars);
}
```

### String.join()

`String.join()`은 `InputStream.transferTo()`처럼, 더 짧고 읽기 쉽게 쓰기 위한 문법적 편의에 가깝다고 볼 수 있습니다. 실제 코드는 아래와 같습니다.

```java
public static String join(CharSequence delimiter,
        Iterable<? extends CharSequence> elements) {
    Objects.requireNonNull(delimiter);
    Objects.requireNonNull(elements);
    StringJoiner joiner = new StringJoiner(delimiter);
    for (CharSequence cs: elements) {
        joiner.add(cs);
    }
    return joiner.toString();
}
```

인수에 대한 null 체크를 빼면, 사실상 prefix와 suffix가 없는 `StringJoiner`를 감싼 형태입니다. 그래서 더 짧은 코드를 원한다면 `StringJoiner`보다 이쪽이 더 편합니다.

### Collectors.joining()

문자열 연결을 `Stream`으로 처리하면 `filter()`, `map()`, `peek()` 같은 중간 연산을 자연스럽게 이어 쓸 수 있다는 장점이 있습니다. 개인적으로도 처리 단계와 의도가 드러나기 쉬워서 `Stream` 스타일을 꽤 선호합니다. 다만 많은 경우 `Stream`은 전통적인 루프보다 성능상 불리할 수 있으니, 상황에 맞게 고르는 편이 좋습니다.

그렇다면 `Collectors.joining()`은 내부적으로 어떻게 동작할까요? 구현은 아래와 같습니다. 결국 내부에서는 `StringJoiner`를 사용하므로, `String.join()`이나 `StringJoiner`와 비교할 때 차이는 주로 `Stream` 파이프라인 자체에서 생긴다고 볼 수 있습니다.

```java
public static Collector<CharSequence, ?, String> joining(CharSequence delimiter,
                                                            CharSequence prefix,
                                                            CharSequence suffix) {
    return new CollectorImpl<>(
            () -> new StringJoiner(delimiter, prefix, suffix),
            StringJoiner::add, StringJoiner::merge,
            StringJoiner::toString, CH_NOID);
}
```

### toString()

사실 Collection(`List<String>`)이라면 더 단순한 방법도 있습니다. `toString()`을 호출하면 쉼표로 구분된 문자열 비슷한 결과가 바로 나옵니다. 다만 앞뒤에 `[]`가 붙기 때문에, 실제로 쓰려면 `substring()` 같은 추가 처리가 필요합니다. 아래는 그 예입니다.

```java
List<String> list = List.of("A", "B", "C");
String toString = list.toString(); // [A, B, C]
String result = toString.substring(1, toString.length() - 1); // A, B, C
```

구분자가 콤마가 아니라면, `toString()` 결과에 다시 `replace()`를 적용하는 방법도 떠올릴 수 있습니다. 하지만 이 방식은 꽤 비효율적입니다. `replace()` 구현을 보면 결국 내부에서 `StringBuilder`로 새 문자열을 다시 만들기 때문입니다.

```java
public String replace(CharSequence target, CharSequence replacement) {
    String tgtStr = target.toString();
    String replStr = replacement.toString();
    int j = indexOf(tgtStr);
    if (j < 0) {
        return this;
    }
    int tgtLen = tgtStr.length();
    int tgtLen1 = Math.max(tgtLen, 1);
    int thisLen = length();

    int newLenHint = thisLen - tgtLen + replStr.length();
    if (newLenHint < 0) {
        throw new OutOfMemoryError();
    }
    StringBuilder sb = new StringBuilder(newLenHint);
    int i = 0;
    do {
        sb.append(this, i, j).append(replStr);
        i = j + tgtLen;
    } while (j < thisLen && (j = indexOf(tgtStr, j + tgtLen1)) > 0);
    return sb.append(this, i, thisLen).toString();
}
```

적어도 구분 기호가 쉼표가 아닌 경우에는 `toString()`과 `replace()`에서 문자열을 생성하는 것보다 다른 방법을 취하는 것이 성능면에서는 유리하지 않은가 하는 추측이 가능합니다. 물론, 요소수라고 하는 변수가 있으므로, 실제의 성능은 측정해 보지 않으면 모르는 것입니다만…

### 벤치 마크 해보기 (1)

이제 각 API의 특징을 대략 정리했으니, 다음은 성능을 볼 차례입니다. 특히 궁금한 점은 `String.join()`과 `Collectors.joining()`도 결국 내부에서 `StringJoiner`를 쓰는데, 그렇다면 `StringBuffer`나 `StringBuilder`보다 실제로 더 나을지가 아닐까 싶었습니다.

실제 애플리케이션 코드를 쓰는 입장에서는, 당연히 더 간단한 코드를 쓰고 싶습니다. 다만 이런 편의 API는 내부 구현이 더 복잡한 대신 성능을 챙겼을 수도 있기 때문에, 감으로만 판단하기는 어렵습니다. 게다가 문자열 처리에서는 보통 `StringBuilder`가 빠르다고 알려져 있으니 더 궁금해졌습니다. 그래서 이번에도 벤치마크를 돌려 봤습니다.

벤치마크는 쉼표로 구분된 문자열을 연결하는 예제로 작성되었습니다. 다음은 그 코드입니다.

```java
@State(Scope.Benchmark)
public class StringConcatTest {

    private static final String DELIMITER = ", ";

    private List<String> target;

    @Setup
    public void init() {
        final DecimalFormat format = new DecimalFormat("0000000");
        this.target = IntStream.rangeClosed(0, 1000000).mapToObj(i -> format.format(i)).collect(Collectors.toList());
    }

    @Benchmark
    public void toString(final Blackhole bh) {
        final String toString = target.toString();
        bh.consume(toString.substring(1, toString.length() - 1));
    }

    @Benchmark
    public void stringJoin(final Blackhole bh) {
        bh.consume(String.join(DELIMITER, target));
    }

    @Benchmark
    public void collectorsJoining(final Blackhole bh) {
        bh.consume(target.stream().collect(Collectors.joining(DELIMITER)));
    }

    @Benchmark
    public void stringBuffer(final Blackhole bh) {
        final StringBuffer buffer = new StringBuffer();
        final int limit = this.target.size() - 1;
        for (int i = 0; i < limit; i++) {
            buffer.append(this.target.get(i));
            buffer.append(DELIMITER);
        }
        buffer.append(this.target.get(limit));
        bh.consume(buffer.toString());
    }

    @Benchmark
    public void stringBuilder(final Blackhole bh) {
        final StringBuilder builder = new StringBuilder();
        final int limit = this.target.size() - 1;
        for (int i = 0; i < limit; i++) {
            builder.append(this.target.get(i));
            builder.append(DELIMITER);
        }
        builder.append(this.target.get(limit));
        bh.consume(builder.toString());
    }
}
```

그리고 결과는 다음과 같습니다.

```dos
Benchmark                            Mode  Cnt   Score   Error  Units
StringConcatTest.toString           thrpt   25  41.445 ± 0.461  ops/s
StringConcatTest.stringJoin         thrpt   25  28.396 ± 0.447  ops/s
StringConcatTest.collectorsJoining  thrpt   25  31.024 ± 1.313  ops/s
StringConcatTest.stringBuffer       thrpt   25  30.570 ± 1.205  ops/s
StringConcatTest.stringBuilder      thrpt   25  45.965 ± 1.736  ops/s
```

이 결과를 보면, 역시 `StringBuilder` 성능이 가장 좋습니다. 다만 잘 알려져 있듯이 `StringBuilder`는 멀티스레드를 고려한 API가 아니므로, 스레드 안전성이 중요한 환경이라면 다른 선택지를 봐야 합니다. 그 점을 감안하면 `String.join()`과 `Collectors.joining()`이 생각보다 큰 차이를 보이지 않는 것도 흥미롭습니다.

또 `toString()` 자체는 빠르지만, 여기에 `replace()`까지 끼우는 순간 성능이 크게 떨어집니다. 그래서 굳이 문자열 연결 용도로 `toString()`을 우회해서 쓸 이유는 거의 없어 보입니다. 코드 의도도 바로 드러나지 않습니다.

그리고 한 가지는 꽤 분명합니다. 이제 새 코드에서 `StringBuffer`를 적극적으로 선택할 이유는 거의 없어 보입니다. 특별한 사정이 없다면 다른 API를 우선 고려하는 편이 낫겠습니다.

## Split

다음은 문자열 분할입니다. 앞서 말했듯이 문자열 분할은 사실상 `String.split()`이 기본 선택지라고 봐도 됩니다. `substring()`으로 어떻게든 나눌 수도 있겠지만, 그러려면 루프와 조건 분기가 따라붙으니 여기서는 비교 대상에서 빼겠습니다.

여기서 보고 싶은 건 분할한 뒤의 처리입니다. `String.split()` 결과는 `String[]`이므로, 상황에 따라 `Collection`으로 바꾸고 싶어집니다. 결국 이 구간은 "배열을 List로 어떻게 바꿀 것인가"에 대한 비교라고 보면 됩니다.

### Arrays.asList()

배열을 List로 바꾸는 가장 쉬운 방법은 `Arrays.asList()`이라고 생각합니다. 코드도 간단하네요.

```java
String string = "A, B, C";
// 먼저 분할한다
String[] array = string.split(", ");
// List로 바꾼다
List<String> list = Arrays.asList(array);
```

다만 이렇게 만든 List는 고정 길이이기 때문에, 요소 추가나 삭제 같은 변경은 할 수 없습니다.

물론 이것은 새 List 인스턴스에 요소를 복사하여 해결할 수 있습니다. 가장 간단한 것은 생성자의 인수로 List를 전달하는 방법입니다. 그래서 "배열을 Mutable List로 만듭니다"가장 쉬운 방법은 아마 다음과 같습니다.

```java
String string = "A, B, C";
// 먼저 분할한다
String[] array = string.split(", ");
// List로 바꾼다
List<String> list = Arrays.asList(array);
// Mutable한 List 인스턴스를 만든다
List<String> mutableList = new ArrayList<>(list);
```

### Arrays.stream()

배열을 List로 만드는 또 다른 방법은 `Stream`을 쓰는 것입니다. 이 방식은 `map()`이나 `filter()` 같은 중간 연산을 자연스럽게 붙일 수 있다는 장점이 있습니다. 또 어떤 `Collectors`를 쓰느냐에 따라 결과 List를 변경 가능하게 만들지, 불변으로 만들지도 결정할 수 있습니다. 대신 코드는 `Arrays.asList()`보다 조금 더 길어집니다.

```java
String string = "A, B, C";
// 먼저 분할한다
String[] array = string.split(", ");
// List로 바꾼다(Mutable)
List<String> mutableList = Arrays.stream(array).collect(Collectors.toList());
// List로 바꾼다(Immutable)
List<String> mutableList = Stream.of(array).collect(Collectors.toUnmodifiableList());
```

### 벤치 마크 해보기 (2)

이제 다시 벤치마크입니다. 코드만 보면 `Arrays.asList()` 쪽이 더 단순하지만, 변경 가능한 List가 필요하면 한 번 더 감싸야 하므로 성능상 손해가 있을 수도 있습니다. 그래서 `Arrays.asList()`와 `Stream` 기반 변환을, 변경 가능/불변 두 경우로 나눠 비교해 봤습니다.

```java
@State(Scope.Benchmark)
public class StringSplitTest {

    private static final String DELIMITER = ", ";

    private String target;

    @Setup
    public void init() {
        final DecimalFormat format = new DecimalFormat("0000000");
        this.target = IntStream.rangeClosed(0, 1000000).mapToObj(i -> format.format(i)).collect(Collectors.joining(DELIMITER));
    }

    @Benchmark
    public void arraysAsListImmutable(final Blackhole bh) {
        bh.consume(Arrays.asList(target.split(DELIMITER)));
    }

    @Benchmark
    public void arraysAsListMutable(final Blackhole bh) {
        bh.consume(new ArrayList<>(Arrays.asList(target.split(DELIMITER))));
    }

    @Benchmark
    public void streamCollectImmutable(final Blackhole bh) {
        bh.consume(Arrays.stream(target.split(DELIMITER)).collect(Collectors.toUnmodifiableList()));
    }

    @Benchmark
    public void streamCollectMutable(final Blackhole bh) {
        bh.consume(Arrays.stream(target.split(DELIMITER)).collect(Collectors.toList()));
    }
}
```

그리고 결과는 다음과 같습니다.

```dos
Benchmark                                Mode  Cnt  Score   Error  Units
StringSplitTest.arraysAsListImmutable   thrpt   25  8.316 ± 1.085  ops/s
StringSplitTest.arraysAsListMutable     thrpt   25  8.133 ± 0.435  ops/s
StringSplitTest.streamCollectImmutable  thrpt   25  6.086 ± 0.312  ops/s
StringSplitTest.streamCollectMutable    thrpt   25  7.247 ± 0.262  ops/s
```

여기서는 `Arrays.asList()` 쪽이 더 좋은 성능을 보였습니다. 중간에 추가 가공이 필요하다면 `Stream` 쪽이 더 자연스러울 수 있지만, 단순히 배열을 List로 바꾸는 목적이라면 `Arrays.asList()`가 코드도 더 짧고 성능도 조금 앞서는 편이었습니다. 결국 이번에도 결론은 같습니다. 무엇을 하고 싶은지에 따라 코드를 고르는 편이 낫습니다.

## 마지막으로

문자열 성능 쪽은 [Baeldung의 글](https://www.baeldung.com/java-string-performance)도 꽤 잘 정리돼 있어서 함께 보면 좋습니다. 요즘 애플리케이션에서 가장 자주 다루는 데이터 중 하나가 문자열인 만큼, 성능과 가독성 사이의 균형은 계속 신경 쓰게 됩니다. 저도 개인적으로는 `Stream`을 좋아해서 웬만하면 그쪽으로 풀고 싶지만, 이런 비교를 해 보면 언제나 같은 답이 나오지는 않습니다.
