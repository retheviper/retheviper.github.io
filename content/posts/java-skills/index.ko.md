---
title: "Java의 여러 코딩 기술"
date: 2019-11-17
categories:
  - java
image: "../../images/java.webp"
tags:
  - java
translationKey: "posts/java-skills"
---

이번 글에서는 대단한 비기라기보다는, 코드를 쓰면서 "이건 편하다" 혹은 "그냥 멋있다"라고 느꼈던 Java 코딩 팁 몇 가지를 정리해 보겠습니다.

## Stream으로 List 변환

다음과 같은 두 클래스가 있다고 가정해 봅시다. 코드 양을 줄이기 위해 Lombok을 사용한다고 생각하겠습니다.

```java
@Getter
public class Item {

    private String value;
}

@Setter
public class Product {

    private String value;
}
```

업무적으로 이 두 클래스의 `value` 필드가 같다고 가정하겠습니다. 그러면 `Item`의 값을 `Product`로 옮겨야 할 때가 있습니다. 객체가 하나뿐이라면 크게 복잡하지 않습니다.

```java
public Product setProductValueFromItemValue(Item item) {

    Product product = new Product();
    product.setValue(item.getValue());
    return product;
}
```

여러 객체를 매핑해야 한다면 [ModelMapper](http://modelmapper.org/) 같은 라이브러리를 쓸 수도 있습니다. 이름이 같은 Getter/Setter를 자동으로 연결해 주기 때문에 편리합니다.

```java
public Product setProductValueFromItemValue(Item item) {

    ModelMapper mapper = new ModelMapper();
    Product product = mapper.map(item, Product.class);
    return product;
}
```

그런데 이 대상이 `List`나 `Map`이라면 어떨까요. 보통은 `for` 문 안에서 하나씩 매핑하게 됩니다.

```java
public List<Product> itemListToProductList(List<Item> itemList) {

    List<Product> productList = new ArrayList<>();
    for (Item item : itemList) {
        productList.add(mapper.map(item, Product.class));
    }
    return productList;
}
```

이걸 `Stream`과 `Lambda`로 더 간단하게 쓸 수 있습니다.

```java
public List<Product> itemListToProductList(List<Item> itemList) {

    List<Product> productList = itemList.stream().map(item -> mapper.map(item, Product.class)).collect(Collectors.toList());
    return productList;
}
```

하는 일 자체는 `for` 문과 크게 다르지 않습니다. 원본 리스트에서 요소를 하나씩 꺼내 매핑하고, 새 객체를 만들어 다시 모으는 방식입니다. 다만 `map()`의 인자로 `Lambda`를 쓰기 때문에 단순 매핑뿐 아니라 더 복잡한 처리도 넣을 수 있습니다. 같은 일을 하더라도 코드가 짧고 읽기 쉬워집니다.

## Collection을 Immutable하게 쓰기

Immutable, 즉 불변 클래스에 대해서는 [인스턴스를 Immutable하게 만들기 위한 고민](../java-thoughts-of-immutable)에서도 다뤘습니다. 이번에는 `Collection`을 이용해 클래스의 `List`와 `Map`도 불변처럼 다루는 방법을 보겠습니다. 아래 코드는 `List` 예시입니다.

```java
public List<Item> returnAsUnmodifiableList(List<Item> list) {

    return Collections.unmodifiableList(list);
}
```

같은 방식으로 `Collections.unmodifiableMap()`을 사용하면 `Map`도 불변처럼 만들 수 있습니다. 이렇게 만든 `List`나 `Map`은 수정할 수 없기 때문에 설정값 같은 데이터를 담는 데 유용합니다. 다만 `null`이 들어가면 `NullPointerException`이 발생할 수 있으니 주의해야 합니다. 감싸려는 `List`가 `null`일 가능성이 있다면 `Collections.emptyList()`를 대신 사용할 수 있습니다.

반대로 불변으로 만든 `List`나 `Map`을 수정하고 싶다면 새 객체로 복사하면 됩니다.

```java
public List<Item> returnAsModifiableList(List<Item> list) {

    return new ArrayList<>(list);
}
```

다만 이렇게 복사한 뒤 데이터를 수정하면 원본과의 관계를 잘 확인해야 합니다. 참조를 어떻게 넘기고 있는지에 따라 예상과 다른 동작이 생길 수 있기 때문입니다.

## 커스텀 클래스를 Iterable로 만들기

어떤 클래스 안에 자식 요소가 `List`로 들어 있다고 가정해 봅시다. 예를 들면 이런 형태입니다.

```java
public class Container {

    private List<Baggage> baggages = new ArrayList<>();
}
```

경우에 따라서는 이 클래스의 자식 요소를 전부 꺼내 `for` 문으로 순회하고 싶을 수 있습니다. 보통은 이런 식으로 작성합니다.

```java
public void printBaggageNames(Container container) {

    List<Baggage> baggages = container.getBaggages();
    for (Baggage baggage : baggages) {
        System.out.println(baggage.getName());
    }
}
```

그런데 이 클래스 자체를 향상된 `for` 문에서 바로 사용할 수 있다면 더 편리합니다.

```java
public void printBaggageNames(Container container) {

    for (Baggage baggage : container) {
        System.out.println(baggage.getName());
    }
}
```

이렇게 되면 Getter도 필요 없어지고, 코드도 한결 간결해집니다. 또 `List` 자체를 노출하지 않기 때문에 내부 구조를 숨기기에도 좋습니다.

이것은 `Iterable`을 구현하면 됩니다.

```java
public class Container implements Iterable<Baggage> {

    private final List<Baggage> baggages = new ArrayList<>();

    @Override
    public Iterator<Baggage> iterator() {
        return baggages.iterator();
    }
}
```

이렇게 하면 상위 클래스의 자식 요소를 향상된 `for` 문으로 쉽게 순회할 수 있습니다. 생각보다 단순합니다.

## 마지막으로

아주 고급 기술만 모은 것은 아니지만, 기억해 두면 어디선가 도움이 될 만한 팁들을 정리해 봤습니다. 실제 업무에서도 바로 써 먹을 수 있는 내용들이고, 단순히 "동작만 하면 된다"를 넘어서고 싶을 때 특히 유용합니다. 이런 작은 차이가 결국 코드 품질과 작업 속도의 차이로 이어진다고 생각합니다.

앞으로도 비슷하게 실무에서 자주 쓰이지만 지나치기 쉬운 팁이 생기면 계속 정리해 볼 생각입니다.
