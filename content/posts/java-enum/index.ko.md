---
title: "Enum 활용하기"
date: 2019-10-27
categories:
  - java
image: "../../images/java.webp"
tags:
  - enum
  - java
translationKey: "posts/java-enum"
---

Java의 Enum은 생각보다 활용 범위가 넓습니다. 읽기 쉽고, 상수 하나에 여러 값을 붙일 수 있다는 점도 편리합니다. 여러 클래스에서 공통 코드값을 다루거나 DB와 연동할 때 특히 유용해서, 저도 업무에서 자주 쓰고 있습니다. 단순히 상수를 모아 두는 데서 끝나지 않고, 필요한 처리까지 함께 담을 수 있다는 점도 장점입니다.

이번 글에서는 제가 경험하고 조사한 내용을 바탕으로 Java Enum의 활용법과 장점을 정리해 보겠습니다.

## 읽기 쉬움

코드가 읽기 쉽다는 건 결국 유지보수에 유리하다는 뜻입니다. 개인적으로는 프로그래밍이 `일단 동작하게 만들기`에서 끝나면 안 되고, `중복 처리를 메서드나 클래스로 분리하기`, `남이 봐도 이해하기 쉬운 코드로 다듬기`까지 가야 한다고 생각합니다.

테이블에 코드값이 있다고 해 봅시다. DB 종류에 따라 `boolean`일 수도 있고 `char(1)`일 수도 있습니다. 코드값이 단순히 두 상태만 가지는 것이 아니라 세 개 이상일 수도 있습니다. Java 내부 상태를 String으로 두고 DB에는 `char(1)`로 저장하는 예를 들면, 대략 이런 코드가 필요합니다.

```java
// String으로 표현된 상태를 DB 코드값으로 바꾸는 메서드 예시
public String toCodeValue(String status) {

    if (status.equals("TRUE")) {
        return "0";
    } else {
        return "1";
    }
}
```

이 메서드를 쓰는 코드는 아래처럼 됩니다.

```java
// item 객체에 상태 지정
public void setStatusTrue(Item item) {

    item.setStatus("TRUE");
}
```

하드코딩된 `status`는 상수로 바꿀 수 있습니다. 하지만 상수를 일반 클래스에 두면 그 값이 어디에 정의돼 있는지 먼저 알아야 합니다. 어디에 있는지 모르면 수정하기도 어렵고, 같은 상수를 여러 클래스에 중복해서 만들 가능성도 있습니다.

게다가 이런 코드에서는 결과를 보기 전까지 상태가 어떻게 바뀌는지 확인하기 어렵습니다. 만약 `TRUE`도 `FALSE`도 아닌 제3의 문자열이 들어온다면 `if` 분기를 계속 늘려야 합니다.

이걸 Enum으로 바꾸면 아래처럼 됩니다.

```java
public enum StatusEnum {

    TRUE("0"),

    FALSE("1");

    private Integer codeValue;

    StatusEnum(Integer codeValue) {
        this.codeValue = codeValue;
    }

    public Integer getCodeValue() {
        return codeValue;
    }
}
```

상수명과 코드값을 붙이고 필드, 생성자, Getter만 두면 됩니다. 이런 Enum은 Lombok으로도 쉽게 만들 수 있습니다.

```java
@Getter
@AllArgsConstructor
public enum StatusEnum {

    TRUE("0"),

    FALSE("1");

    private Integer codeValue;
}
```

실제로 쓰면 이렇게 됩니다.

```java
Item item = new Item();
item.setStatus(StatusEnum.TRUE.getCodeValue()); // "0"
```

Enum을 쓰면 넣을 값이 명확해집니다. 독립된 클래스이므로 패키지로 분리해 관리하기도 쉽습니다. 나중에 코드값이 늘어나도 Enum만 수정하면 됩니다. 아예 필드 자체를 Enum으로 둘 수도 있습니다.

```java
@Data
public class Item {

    private StatusEnum status;
}
```

필드가 Enum이면 값 지정이 더 간단합니다.

```java
Item item = new Item();
item.setStatus(StatusEnum.TRUE); // TRUE로 저장

String itemCodeValue = item.getStatus().getCodeValue();
```

## 여러 값 보관

Enum의 장점은 상수 하나에 여러 값을 붙일 수 있다는 점입니다. 예를 들어 두 개 이상의 DB를 쓰고 있고, 같은 항목을 한쪽은 `char(1)`, 다른 쪽은 `boolean`으로 관리한다면 둘 다 하나의 Enum으로 묶을 수 있습니다.

```java
@Getter
@AllArgsConstructor
public enum MultiValueEnum {

    Y("0", true),

    N("1", false);

    private Integer charValue;

    private boolean booleanValue;
}
```

어떤 Getter를 쓰느냐에 따라 같은 상수라도 다른 타입의 값을 돌려줍니다.

```java
Item item = new Item();
item.setStatus(MultiValueEnum.Y.getCharValue()); // "0"

item.setStatus(MultiValueEnum.Y.getBooleanValue()); // true
```

배열이나 List로도 처리할 수 있지 않느냐고 할 수 있습니다. 물론 가능합니다. `Stream`을 쓰면 List를 값으로 들고 있다가 그 안의 값과 비교하는 방식도 만들 수 있습니다.

```java
@Getter
@AllArgsConstructor
public enum ListValueEnum {

    Y(Arrays.asList("Good", "Excellent")),

    N(Arrays.asList("Bad", "Unavailable")),

    NULL(null);

    private List<String> codeValueList;

    public boolean hasCodeValue(String codeValue) {
        return codeValueList.stream().anyMatch(code -> code.equals(codeValue));
    }

    public ListValueEnum findByValue(String codeValue) {
        return Arrays.stream(ListValueEnum.values())
                .filter(listValueEnum -> listValueEnum.hasCodeValue(codeValue))
                .findAny()
                .orElse(NULL);
    }
}
```

저도 처음 봤을 때는 조금 낯설었지만, 상황에 따라 충분히 써 볼 만한 방식입니다.

필드로 쓰는 Enum에 별도 값이 없다면 아래처럼 어노테이션만 붙여도 됩니다.

```java
@Data
public class Item {

    @Enumerated(EnumType.STRING)
    private StringEnum codeValue1;
}
```

## 클래스처럼 쓰기

Enum은 클래스이기 때문에 처리 로직도 함께 넣을 수 있습니다. 메서드를 값처럼 들고 있는 형태라고 보면 됩니다. 이 부분은 이전에 Lambda를 다룬 [함수형 인터페이스 활용하기](/ko/posts/java-functional-interface/)를 함께 보면 이해가 더 쉽습니다.

```java
@AllArgsConstructor
public enum CalculateEnum {

    TYPE_1(num -> num),

    TYPE_2(num -> num + 10);

    private Function<Integer, Integer> calculate;

    public Integer calculate(Integer number) {
        return calculate.apply(number);
    }
}
```

사용하면 이런 식입니다.

```java
Integer type_1 = CalculateEnum.TYPE_1.calculate(10); // 10

Integer type_2 = CalculateEnum.TYPE_2.calculate(10); // 20
```

## 마지막으로

공통 부품을 중복해서 만들면 관리도 어렵고, 코드도 불필요하게 늘어납니다. Enum은 이런 문제를 줄이는 데 꽤 효과적입니다.

다만 모든 정수를 꼭 Enum으로 바꿔야 하는 것은 아닙니다. 경우에 따라 테이블로 관리하는 편이 더 나을 수도 있고, 특정 클래스 안에서만 쓰는 값이라면 굳이 Enum을 만들 필요도 없습니다.

그래도 이런 활용법을 알고 있으면 나중에 분명 쓸 일이 생깁니다. 저도 처음에는 Lambda를 그저 읽기 어려운 문법 정도로만 생각했지만, `Function`과 함께 쓰는 방식을 이해하고 나서는 훨씬 적극적으로 활용하게 됐습니다. 결국 중요한 건 도구를 많이 아는 것보다, 어떤 상황에서 무엇을 쓰면 좋은지 판단할 수 있는 경험이라고 생각합니다.
