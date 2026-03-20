---
title: "함수형 인터페이스 활용하기"
date: 2019-08-06
categories:
  - java
image: "../../images/java.webp"
tags:
  - functional interface
  - lambda
  - java
translationKey: "posts/java-functional-interface"
---

이번 글도 업무에서 부딪힌 내용을 정리한 것입니다. `Iterable` 클래스를 만들고 `for` 루프로 순회해야 했는데, 단순히 `Iterator`를 구현하는 것 자체는 어렵지 않았습니다. 문제는 "루프를 돌다가 사용자가 지정한 규칙에 따라 중간에 종료하는 기능"을 넣는 부분이었습니다. 순회 중인 요소를 받아 판단하는 메서드가 필요했고, 그 판단 기준은 인스턴스마다 달라져야 했습니다.

## 메서드를 필드처럼 쓰기

처음에는 "메서드처럼 동작하는 것을 필드로 들고 있는다"는 발상이 조금 낯설었습니다. 그런데 `function`을 쓰면 된다는 이야기를 듣고 찾아보니, 필드로 선언하면서도 메서드처럼 동작하게 만들 수 있었습니다. 직접 적용했던 예를 기준으로 설명해 보겠습니다.

먼저 `Iterable` 클래스를 준비합니다.

```java
// Line을 리스트로 가지고 반복 가능한 클래스
public class Factory implements Iterable<Line> {

    private List<Line> lines = new ArrayList<>();

    public Iterator<Line> iterator() {
        return this.lines.iterator();
    }
}
```

이제 이 `Factory`를 순회하는 예를 보겠습니다. 루프 도중 `Rule` 클래스의 `isEnd(Line)` 판정이 들어갑니다. 조건에 맞으면 `break`로 빠져나갑니다.

```java
Factory factory = new Factory();
Rule rule = new Rule();

for (Line line : factory) {
    if (rule.isEnd(line)) {
        break;
    }
    // ... 처리
}
```

판정을 담당하는 `Rule` 클래스는 아래처럼 만들 수 있습니다.

```java
public class Rule {

    // 판정 규칙을 필드로 둔다
    private Predicate<Line> endRule = line -> line.isBroken();

    // Predicate 조건에 맞는지 판정
    public boolean isEnd(Line line) {
        return endRule.test(line);
    }

    public class RuleBuilder {

        private Predicate<Line> endRule;

        public RuleBuilder endRule(Predicate<Line> endRule) {
            this.endRule = endRule;
            return this;
        }
    }
}
```

`Predicate`가 뭔지, Lambda만으로 이런 게 되나 싶을 수 있지만 실제로 잘 동작합니다. Java 8에서 추가된 `java.util.function` 덕분입니다. 필드는 데이터를 담는다고만 생각했는데, 그 데이터가 메서드 자체일 수도 있다는 점이 새로웠습니다.

## Functional Interface

`java.util.function`에 들어 있는 인터페이스들을 보통 함수형 인터페이스라고 부릅니다. Java 8에서 추가된 Lambda는 "추상 메서드가 하나뿐인 인터페이스를 구현하는 문법"이라고 볼 수 있는데, 바로 그런 인터페이스가 함수형 인터페이스입니다.

말로 들으면 조금 어렵지만, 핵심은 하나입니다. Lambda로 채워서 완성하는 인터페이스입니다. 종류는 여러 가지지만, 하고 싶은 일에 맞춰 고르면 됩니다.

### Function

Function은 가장 기본적인 함수입니다. 인자와 반환값을 지정합니다. 실행은 `apply`입니다.

```java
Function<Integer, String> function = number -> String.valueOf(number);

String result = function.apply(12);
```

#### BiFunction

`Bi`가 붙는 인터페이스는 인자가 두 개입니다.

```java
BiFunction<String, String, Integer> biFunction = (string1, string2) -> Integer.parseInt(string1) + Integer.parseInt(string2);

int result = biFunction.apply("1", "2");
```

### Predicate

`Predicate`는 "참/거짓을 판정하는 함수"라고 보면 됩니다. 인자는 하나, 반환값은 Boolean입니다. 실행은 `test`입니다.

```java
Predicate<String> predicate = string -> string.isEmpty();

boolean result = predicate.test("비어있지 않다");
```

#### BiPredicate

인자가 두 개인 Predicate입니다.

```java
BiPredicate<String, Integer> biPredicate = (string, number) -> string.equals(Integer.toString(number));

boolean result = biPredicate.test("1", 1);
```

### Consumer

Consumer는 인자를 받고 반환값이 없는 타입입니다. 실행은 `accept`입니다.

```java
Consumer<String> consumer = string -> System.out.println(string);

consumer.accept("출력");
```

#### BiConsumer

인자가 두 개인 Consumer입니다.

```java
BiConsumer<String, Integer> biConsumer = (string, number) -> System.out.println(string + "：" + number);

biConsumer.accept("금액", 0);
```

### UnaryOperator

Unary는 단항, Operator는 연산자라는 뜻입니다. 인자와 반환값이 같은 함수입니다.

```java
UnaryOperator<String> uOperator = string -> string + "완성";

String result = uOperator.apply("이 문자를 넣으면");
```

#### BinaryOperator

인자가 두 개인 Operator입니다.

```java
BinaryOperator<String> biOperator = (string1, string2) -> string1 + string2 + "입니다";

String result = biOperator.apply("나는", "괜찮다");
```

### Supplier

Supplier는 인자가 없고 반환값만 있는 타입입니다. 실행은 `get`입니다.

```java
Supplier<String> supplier = () -> "예를 들면 인자 없이 문자열을 돌려준다";

String result = supplier.get();
```

## 마지막으로

Java 8이 나온 지는 꽤 됐지만, 아직도 많은 현장에서 Java 8 기반 코드를 다루게 됩니다. 그래서 Lambda나 `java.util.function` 같은 기능도 그냥 문법으로만 알고 넘기기보다, 실제 문제를 풀 때 어떻게 쓰는지 익혀 두는 편이 좋습니다. 처음에는 낯설어도 한 번 손에 익으면 코드 구조를 꽤 유연하게 만들 수 있습니다.
