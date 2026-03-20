---
title: "디자인 패턴: 빌더"
date: 2019-07-14
categories:
  - java
image: "../../images/java.webp"
tags:
  - design pattern
  - java
translationKey: "posts/java-design-pattern-builder"
---

예전에, 저보다 먼저 일본에서 개발자로 일하던 대학 후배에게 어떤 언어나 프레임워크를 공부하면 좋겠냐고 물은 적이 있습니다. 주변에서는 C#을 하라는 사람도 있었고, React나 Node.js처럼 유행하는 라이브러리를 익히는 편이 낫다는 말도 있었습니다. 하지만 저는 현업에서 실제로 쓰는 사람의 의견을 듣고 싶었습니다. 그 후배는 언어나 프레임워크는 메인 언어를 충분히 익히면 언제든 바꿀 수 있지만, 실무에서 진짜 필요한 건 디자인 패턴이라고 말했습니다.

이후 책을 사서 몇 가지 디자인 패턴을 살펴봤지만, 그 패턴을 실제 코드에 어떻게 적용해야 하는지는 혼자 생각하기 쉽지 않았습니다. 혼자 코드를 짤 때는 내가 이해할 수 있는 형태로만 작성하면 되니, 패턴을 따질 여유가 없었습니다. 무엇보다 사용자만 생각하고 만든 코드가 많아서, 패턴을 의식한 설계를 할 필요도 별로 없었습니다.

그런데 지금은 Java 기반 프레임워크 개발에 참여하게 되면서, 내가 만든 코드를 다른 사람이 쓸 수 있도록 작성하라는 요구를 받았습니다. 처음에는 늘 하던 대로 DTO를 바탕으로 Wrapper 클래스를 만들면 되지 않을까 생각했지만, 직접 구현해 보니 이 방식은 맞지 않았습니다. 설정해야 할 값이 수십 개에서 수백 개까지 늘어날 수 있었고, 인수로 넘기는 값도 단순한 숫자나 문자열만은 아니었기 때문입니다. 내가 만든 코드를 사용하는 입장에서 보면, 이런 형태는 분명 불편할 수밖에 없습니다.

어떻게 개선하면 좋을지 고민하던 중, 지시를 내린 분에게 Builder 패턴을 써 보면 좋겠다는 조언을 들었습니다. 인수는 최소화하면서도 직관적으로 사용할 수 있다는 설명이었습니다. 실제로 써 보니 전통적인 DTO 방식과는 확실히 다르다는 걸 느꼈습니다. 여기서는 DTO와 Builder 패턴이 어떻게 다른지 차례로 정리해 보겠습니다.

## Telescoping Constructor Pattern

Telescoping은 망원경이라는 뜻입니다. 생성자 인수 목록이 점점 길어지는 모습이 망원경처럼 늘어나는 것 같아서 붙은 이름이라고 보면 됩니다. JavaBean에서도 종종 보이는 방식입니다. 객체를 만들 때 인수 개수에 따라 값을 넣을 필드 수를 조절할 수 있고, Method Overloading[^1]으로 여러 생성자를 준비하면 됩니다.

예를 들어 카페에서 커피를 주문하는 과정을 Java 클래스로 표현해 보겠습니다. 컵 크기, 핫/아이스 여부, 시럽 추가 여부, 크림 추가 여부 같은 옵션이 여러 개 있습니다. 이를 Telescoping Constructor Pattern으로 작성하면 다음과 같습니다.

```java
public class CoffeeOrder {

    private final String size;
    private final boolean hot;
    private final boolean addCream;
    private final boolean addSugar;
    private final boolean takeout;

    public CoffeeOrder(String size, boolean hot) {
        this(size, hot, false, false, false);
    }

    public CoffeeOrder(String size, boolean hot, boolean addCream) {
        this(size, hot, addCream, false, false);
    }

    public CoffeeOrder(String size, boolean hot, boolean addCream, boolean addSugar) {
        this(size, hot, addCream, addSugar, false);
    }

    public CoffeeOrder(String size, boolean hot, boolean addCream, boolean addSugar, boolean takeout) {
        this.size = size;
        this.hot = hot;
        this.addCream = addCream;
        this.addSugar = addSugar;
        this.takeout = takeout;
    }
}
```

이제 주문을 표현하는 클래스는 하나 만들었습니다. 실제로 사용하면 아래와 같습니다.

```java
public class Cafe {

    public static void main(String[] args) {
        // 주문마다 객체를 만든다
        CoffeeOrder order_1 = new CoffeeOrder("tall", false);
        CoffeeOrder order_2 = new CoffeeOrder("grande", false, true);
        CoffeeOrder order_3 = new CoffeeOrder("venti", true, true, false);
        CoffeeOrder order_4 = new CoffeeOrder("tall", false, false, true, false);
    }
}
```

이 방식의 문제는 객체를 만들 때 각 인수가 무슨 의미인지 알기 어렵다는 점입니다. 실제 클래스를 열어 보지 않으면 연달아 나오는 `false`와 `true`가 무엇을 뜻하는지 감이 잘 오지 않습니다. 그리고 예를 들어 크기와 시럽만 지정하고 싶다면 그에 맞는 생성자를 또 하나 만들어야 합니다. 필드가 많아질수록 생성자도 계속 늘어나고, 나중에 옵션이 추가되거나 바뀌면 대응하기도 까다롭습니다.

## JavaBean, DTO(VO)

Java에서 OOP[^2]를 배울 때 초반에 자주 접하는 것이 `JavaBean`입니다. 이것도 하나의 전형적인 패턴이라고 볼 수 있습니다. 익숙한 `getter`와 `setter`로 값을 주고받는 방식이죠. 같은 주문 클래스를 이번에는 이런 형태로 만들어 보겠습니다.

```java
public class CoffeeOrder {

    private final String size;
    private final boolean hot;
    private final boolean addCream;
    private final boolean addSugar;
    private final boolean takeout;

    public CoffeeOrder() {}

    public void setSize(String size) {
        this.size = size;
    }

    public String getSize() {
        return this.size;
    }

    public void setHot(boolean hot) {
        this.hot = hot;
    }

    public boolean getHot() {
        return this.hot;
    }

    public void setAddCream(boolean addCream) {
        this.addCream = addCream;
    }

    public boolean getAddCream() {
        return this.addCream;
    }

    public void setAddSugar(boolean addSugar) {
        this.addSugar = addSugar;
    }

    public boolean getAddSugar() {
        return this.addSugar;
    }

    public void setTakeout(boolean takeout) {
        this.takeout = takeout;
    }

    public boolean getTakeout() {
        return this.takeout;
    }
}
```

생성자로 일부 값을 함께 받는 형태도 가능하지만, JavaBean의 핵심은 결국 `getter`와 `setter`에 있습니다. 여기서는 그 부분만 보겠습니다. 같은 주문을 이 방식으로 만들면 아래와 같습니다.

```java
public class Cafe {

    public static void main(String[] args) {
        // 주문 내용은 setter로 설정한다
        CoffeeOrder order_1 = new CoffeeOrder();
        order_1.setSize("tall");
        order_1.setHot(false);

        CoffeeOrder order_2 = new CoffeeOrder();
        order_2.setSize("grande");
        order_2.setHot(false);
        order_2.setAddCream(false);
    }
}
```

이전보다 항목별로 값을 넣을 수 있어서, 각 `setter`만 봐도 어떤 주문인지 더 분명해집니다. 필드가 늘어나더라도 그에 맞는 `getter`와 `setter`만 추가하면 됩니다.

다만 주문 하나를 완성하는 데 필요한 옵션이 많아지면 코드가 지나치게 길어진다는 문제가 있습니다. 지금은 필드가 5개뿐이지만, 20개나 30개가 되면 이야기가 달라집니다. 그걸 하나씩 써 넣는 건 꽤 번거롭습니다. 제가 막혔던 지점도 바로 이 부분이었습니다. 그래서 Builder를 써서 이 문제를 풀어 보겠습니다.

## Builder Pattern

```java
public class CoffeeOrder {

    private final String size;
    private final boolean hot;
    private final boolean addCream;
    private final boolean addSugar;
    private final boolean takeout;

    public CoffeeOrder(String size, boolean hot, boolean addCream, boolean addSugar, boolean takeout) {
        this.size = size;
        this.hot = hot;
        this.addCream = addCream;
        this.addSugar = addSugar;
        this.takeout = takeout;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {

        private String size;
        private boolean hot;
        private boolean addCream;
        private boolean addSugar;
        private boolean takeout;

        public Builder() {}

        public Builder size(String size) {
            this.size = size;
            return this;
        }

        public Builder hot(boolean hot) {
            this.hot = hot;
            return this;
        }

        public Builder addCream(boolean addCream) {
            this.addCream = addCream;
            return this;
        }

        public Builder addSugar(boolean addSugar) {
            this.addSugar = addSugar;
            return this;
        }

        public Builder takeout(boolean takeout) {
            this.takeout = takeout;
            return this;
        }

        public CoffeeOrder build() {
            return new CoffeeOrder(size, hot, addCream, addSugar, takeout);
        }
    }
}
```

내부 클래스까지 들어가서 처음 보면 조금 복잡해 보입니다. 하지만 실제로 써 보면 생각보다 단순합니다. 이 Builder 클래스를 사용하면 어떻게 되는지 확인해 보겠습니다.

```java
public class Cafe {

    public static void main(String[] args) {
        // 주문을 만들면서 Builder를 사용한다
        CoffeeOrder order_1 = CoffeeOrder.builder()
                .size("tall")
                .hot(false)
                .build();

        CoffeeOrder order_2 = CoffeeOrder.builder()
                .size("grande")
                .hot(false)
                .addCream(true)
                .takeout(true)
                .build();
    }
}
```

`setter`와 비슷하게 여러 옵션을 순서대로 정리할 수 있고, 생성과 동시에 주문을 완성할 수도 있습니다. Builder가 자기 자신을 반환하므로 메서드를 연달아 호출할 수 있기 때문입니다. 이 방식이라면 필드가 더 늘어나도 훨씬 유연하게 대응할 수 있습니다.

## Lombok을 사용하면

이런 패턴들은 [Lombok](https://projectlombok.org)을 쓰면 어노테이션만으로도 어느 정도 정리할 수 있습니다. 생성자는 `@NoArgsConstructor`, `@RequiredArgsConstructor`, `@AllArgsConstructor`로 만들 수 있고, JavaBean은 `@Data`만 붙여도 `getter`와 `setter`를 생성할 수 있습니다. Builder 역시 `@Builder`로 처리할 수 있습니다. 아래는 Lombok 예시입니다.

```java
import lombok.Builder;

@Builder
public class CoffeeOrder {

    private final String size;
    private final boolean hot;
    private final boolean addCream;
    private final boolean addSugar;
    private final boolean takeout;
}
```

참고로 `@Builder(toBuilder = true)`를 쓰면 새 인스턴스를 만들 때 `CoffeeOrder.builder()`로 바로 Builder에 접근할 수 있고, 기존 인스턴스의 값을 이어받을 때는 `order_1.toBuilder()`를 사용할 수 있습니다. 실제로는 대략 아래와 같은 코드가 생성된다고 보면 됩니다.

```java
import lombok.Builder;

@Builder(toBuilder = true)
public class CoffeeOrder {

    private final String size;
    private final boolean hot;
    private final boolean addCream;
    private final boolean addSugar;
    private final boolean takeout;

    // 기본적인 Builder 구현

    public CoffeeOrderBuilder toBuilder() {
        return new CoffeeOrderBuilder()
                .size(this.size)
                .hot(this.hot)
                .addCream(this.addCream)
                .addSugar(this.addSugar)
                .takeout(this.takeout);
    }
}
```

그리고 Builder를 사용할 때 부모 클래스의 필드를 그대로 이어받고 싶다면, 필드에 `@Builder.Default`를 붙이면 됩니다. 그 밖에도 접근 범위를 조절하는 기능 등 편리한 점이 많아서, 실제 프로젝트에서도 꽤 유용합니다.

## 마지막으로

디자인 패턴의 종류를 알고 구조를 이해하는 것도 중요하지만, 더 중요한 건 결국 적절한 상황에서 꺼내 쓸 수 있느냐라고 생각했습니다. 처음부터 Builder 패턴을 알고 있었다고 해도, 실제 문제를 만나기 전에는 스스로 떠올리지 못했을 가능성이 큽니다. 그래서 앞으로는 디자인 패턴 자체를 공부하는 것과 함께, 어떤 상황에서 쓰면 좋은지도 같이 고민해 보려고 합니다.

[^1]: 인수의 수나 종류를 바꿔 같은 이름의 메서드를 여러 개 만드는 방식입니다.
[^2]: Object Oriented Programming. 객체 지향 프로그래밍이라고도 합니다. 코드를 위에서 아래로 순서대로만 흘려보내는 것이 아니라, 서로 분리된 객체 간의 데이터 교환으로 프로그램을 구성하는 패러다임입니다.
