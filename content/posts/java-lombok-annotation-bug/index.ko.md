---
title: "Lombok 어노테이션 버그 기록"
date: 2019-08-12
categories:
  - java
image: "../../images/java.webp"
tags:
  - lombok
  - java
translationKey: "posts/java-lombok-annotation-bug"
---

[디자인 패턴: 빌더](../java-design-pattern-builder)에서 Builder 패턴과 함께 `Lombok`을 소개했습니다. Bean은 물론 immutable 클래스[^1]나 Builder도 쉽게 만들 수 있고, 어노테이션만으로 다양한 옵션을 붙일 수 있어서 꽤 편리합니다. 그런데 이번에는 Lombok을 쓰다가 버그처럼 보이는 현상을 발견해서 기록해 둡니다.

## 버그가 생긴 곳

어디서 문제가 생겼는지 설명하기 전에, 먼저 어떤 Lombok 어노테이션을 조합해서 썼는지 정리하겠습니다. 저와 비슷한 조합으로 쓰는 분이라면 같은 현상을 겪을 수도 있기 때문입니다.

Lombok의 `@Builder`에는 `toBuilder=true` 옵션이 있습니다. 이 옵션을 쓰면 새 인스턴스를 만들 때 static 메서드를 사용하고, 이미 존재하는 인스턴스에서 일부 값만 바꿔 다시 만들 수 있습니다.

```java
// @Builder만 사용해 새 인스턴스 생성
House house2 = House.builder().type("wooden").build();

// toBuilder로 기존 인스턴스에서 일부 값만 바꿔 재생성
House house3 = house2.toBuilder().type("block").build();
```

또 Builder를 쓰면서도 기본값으로 유지하고 싶은 필드가 있습니다. 제 경우는 List였습니다. 인스턴스를 만들 때 List를 초기화해 두고, Builder로 통째로 넣거나 `addCard()` 같은 메서드로 하나씩 추가하고 싶었습니다.

이를 위해 `@Builder.Default`로 필드를 초기화하고, Builder 클래스에 커스텀 메서드를 붙였습니다.

```java
@Builder(toBuilder = true)
public class Wallet {

    // Builder로 생성될 때도 null이 되지 않게 한다
    @Builder.Default
    List<String> cards = new ArrayList<>();

    // Builder에 커스텀 메서드를 추가
    public class WalletBuilder {
        public WalletBuilder addCard(String card) {
            this.cards.add(card);
            return this;
        }
    }
}
```

이렇게 하면 아래처럼 동작하길 기대했습니다.

```java
// 인스턴스를 만들면서 List에 추가
Wallet myWallet = Wallet.builder().addCard("Apple Card").build();

// 기존 인스턴스에 추가
Wallet newWallet = myWallet.toBuilder().addCard("American Express Card").build();
```

그런데 실제로 테스트해 보니 이 조합에서 문제가 생겼습니다.

## 어떤 버그였나

기존 인스턴스에서는 `add`가 잘 됐지만, 인스턴스를 생성하면서 바로 `add`하면 NPE[^2]가 발생했습니다. 확인해 보니 `this.cards.add(card);`에서 예외가 났고, 생성되지 않은 객체에 요소를 추가하려 한 것이었습니다. 즉, List가 제대로 초기화되지 않았던 것입니다.

조금 찾아보니 [비슷한 이슈](https://github.com/rzwitserloot/lombok/issues/1347)가 있었습니다. 2017년에 올라온 글이라 꽤 오래된 문제처럼 보였지만, 지금 겪는 현상과 거의 같았습니다. 게다가 Lombok `1.18.2`에서 해결됐다는 이야기도 있었는데, 제가 쓰던 버전은 `1.18.8`이었습니다. 해결된 줄 알았는데 다시 생긴 것인지, 버전업 과정에서 재발한 것인지 모르겠지만 같은 현상이 반복됐습니다.

## 해결 방법

제가 찾은 해결책은 `toBuilder=true`와 `@Builder.Default`를 둘 다 쓰지 않는 것이었습니다. Builder에 필드가 제대로 전달되지 않는다면 `@Builder.Default`는 의미가 없고, `toBuilder`도 직접 메서드를 작성해 버리면 됩니다.

```java
// toBuilder 옵션을 쓰지 않는다
@Builder
public class Wallet {

    // @Builder.Default도 쓰지 않는다
    List<String> cards = new ArrayList<>();

    // toBuilder도 직접 작성한다
    public WalletBuilder toBuilder() {
        return new WalletBuilder().cards(this.cards);
    }
}
```

이제는 인스턴스를 만들 때도 List가 제대로 초기화된 상태로 전달됐습니다. 결국 어노테이션을 과하게 믿지 말아야 한다는 뜻이기도 합니다.

## 배운 점

라이브러리를 써서 코드 양을 줄이고 자동화하는 것은 생산성을 높이는 좋은 방법입니다. 다만 사람이 쓴 코드든 라이브러리가 생성한 코드든, 어디에서 문제가 생길지는 항상 열어 두어야 합니다. 때로는 조금 번거롭더라도 명시적인 코드가 더 안전합니다.

이번 일은 어노테이션만 믿고 끝까지 우회하려 했다면 원인을 더 늦게 찾았을 수도 있는 사례였습니다. 그런 의미에서, 자동 생성 코드를 어디까지 신뢰할지 다시 생각해 보게 된 계기였습니다.

같은 방식으로 구현할 생각이 있다면 이런 문제도 생길 수 있다는 정도로 참고하면 좋겠습니다. 편리한 라이브러리일수록 내부 동작을 너무 당연하게 믿지 않는 편이 낫다는 걸 다시 느꼈습니다.

[^1]: 한 번 생성하면 중간에 값을 바꿀 수 없는 클래스입니다. `String`이 대표적입니다.
[^2]: `NullPointerException`입니다. 참조하려는 객체가 메모리에 없을 때 발생하는, Java에서 가장 자주 마주치는 예외 중 하나입니다.
