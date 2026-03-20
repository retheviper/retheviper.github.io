---
title: "인스턴스를 Immutable하게 만들기 위한 고민"
date: 2019-08-25
categories: 
  - java
image: "../../images/java.webp"
tags:
  - design pattern
  - java
translationKey: "posts/java-thoughts-of-immutable"
---

Python 같은 본격적인 객체 지향 언어에서는 상대적으로 덜 의식될 수 있지만, Java에는 참조형과 별도로 primitive type이 존재합니다. Java가 처음 등장하던 시기에는 객체 지향이라는 개념 자체가 막 자리잡던 때였기 때문에, 그런 구조를 갖게 된 것이 아닐까 싶습니다. 그래서 primitive type이 있다는 점을 두고 Java는 완전한 객체 지향 언어가 아니라는 이야기도 종종 나옵니다.

물론 primitive type의 정의는 언어마다 조금씩 다를 수 있습니다. 다만 Java 기준으로 보면, primitive type은 객체가 아닌 데이터 타입을 뜻합니다. 그리고 이 차이는 메모리에 데이터를 어떻게 올리고 어떻게 참조하느냐에서 드러납니다. 객체는 메모리 어딘가에 있는 데이터의 "주소"를 참조하지만, primitive type은 값 자체가 각각의 메모리 영역에 직접 놓입니다.

이 차이는 비교 연산에서도 드러납니다. primitive type끼리는 `==`로 값 자체를 비교할 수 있습니다. 하지만 객체, 그중에서도 흔히 예로 드는 `String`은 그렇게 단순하지 않습니다. `equals()`를 사용해야 올바르게 값 비교를 할 수 있습니다. 객체가 들고 있는 것은 실제 값이 아니라 메모리 주소이기 때문에, "같은 값을 넣었다"고 생각해도 서로 다른 객체일 수 있기 때문입니다.

이런 이유로 메모리와 참조는 프로그래밍에서 매우 중요합니다. 메모리가 아무리 많아도, 데이터를 어떻게 참조하고 어떻게 다루는지 잘못되면 프로그램은 생각대로 동작하지 않을 수 있습니다. 그리고 중요한 것은 단순히 데이터를 어떻게 참조하느냐만이 아니라, 그 데이터를 어떻게 관리하느냐입니다. 처음에는 올바른 값을 넣었다고 해도 중간에 값이 바뀌면, 참조 방식이 맞더라도 프로그램이 정상적으로 동작하지 않을 수 있으니까요.

그런 의미에서 이번에는 Immutable 클래스에 대해 이야기해 보려고 합니다.

## Immutable Object란

Immutable Object는 간단히 말해 "한 번 인스턴스가 생성되면, 그 안의 데이터가 바뀌지 않는 객체"를 뜻합니다. 반대로 인스턴스가 만들어진 이후에도 데이터가 바뀔 수 있는 객체는 Mutable이라고 부릅니다.

대표적인 Mutable 클래스의 예로는 Bean을 들 수 있습니다. Setter를 통해 값을 자유롭게 바꿀 수 있기 때문입니다. 반대로 대표적인 Immutable 클래스는 `String`입니다. `String`은 값을 다시 대입해도 기존 객체 자체가 바뀌는 것이 아니라, 새로운 문자열 객체를 가리키도록 참조만 달라지기 때문입니다.

앞서 말했듯이, 프로그램 실행 중 객체가 가진 값이 중간에 바뀌기 시작하면 안정적인 동작을 보장하기 어려워집니다. 멀티스레드 환경에서는 같은 메모리를 여러 객체가 참조하는 상황이 더 큰 문제로 이어질 수도 있습니다. 그리고 이런 문제는 일단 발생하면 원인을 추적하기도 쉽지 않습니다.

그래서 Immutable 클래스를 만들 때는 크게 두 가지를 의식해야 합니다.

- 인스턴스 생성 이후에 값이 바뀌지 않도록 할 것
- 동일한 메모리 주소를 여러 객체가 공유하며 예상치 못한 변경을 일으키지 않도록 할 것

그럼 실제로 어떻게 Immutable 클래스를 만들 수 있는지 하나씩 보겠습니다.

## Setter를 사용하지 않는다

요즘은 Lombok을 많이 사용하고, 정형적인 코드를 줄여 주기 때문에 Lombok이 생성하는 코드를 기준으로 설명하는 편이 이해하기 쉽습니다. Lombok 자체에 대한 설명은 [이전 글](../java-design-pattern-builder)을 참고해 주세요.

Lombok에서 `@Data`를 붙이면 Getter와 Setter가 자동으로 만들어집니다. 손으로 전부 작성하는 것보다 코드 양을 줄일 수 있어서 꽤 편리합니다.

하지만 Setter가 생긴다는 것은, 곧 Immutable 객체가 아니라는 뜻이기도 합니다. 예를 들어 아래처럼 `@Data`를 사용한 클래스가 있다고 해 보겠습니다.

```java
// @Data를 사용한 경우
public class Car {

    // 필드만 정의
    private String name;
    private String color;

    // 아래 메서드들이 자동 생성됨
    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }

    public void setColor(String color) {
        this.color = color;
    }

    public String getColor() {
        return this.color;
    }
}
```

이 구조에서는 필드가 `final`로 보호되지 않는 한, 언제든 Setter를 통해 값이 바뀔 수 있습니다. 심지어 일부 Setter가 호출되지 않으면 필드가 `null`인 상태로 남을 수도 있습니다. 이런 구조는 Immutable의 정의와는 맞지 않습니다.

다행히 Lombok에는 Immutable 객체 생성에 더 적합한 `@Value`가 있습니다. `@Value`를 사용하면 값 변경을 막는 방향으로 코드를 생성할 수 있습니다.

```java
// @Value를 사용한 경우
public class Car {

    // 필드만 정의
    private String name;
    private String color;

    // 아래 메서드들이 자동 생성됨
    public Car(String name, String color) {
        this.name = name;
        this.color = color;
    }

    public String getName() {
        return this.name;
    }

    public String getColor() {
        return this.color;
    }
}
```

이 경우 인스턴스를 만들 때 생성자를 통해 모든 필드를 전달해야 합니다. 필드가 많아지면 인자가 헷갈릴 수는 있지만, 적어도 "값이 빠진 채로 생성되는 문제"는 줄일 수 있습니다. 그리고 Setter가 없으므로, 한 번 생성한 인스턴스는 이후에 상태를 바꿀 수 없습니다.

또 `@Value`의 장점 중 하나는 Builder 패턴과도 함께 쓸 수 있다는 점입니다. `@Builder`를 붙이면 어떤 필드에 어떤 값을 넣는지 조금 더 분명하게 볼 수 있습니다. 다만 Builder는 기본적으로 "모든 필드를 반드시 채워야 한다"는 보장을 하지 않습니다. 그래서 필요하다면 `build()`를 직접 작성해서 검증을 넣을 수 있습니다.

```java
@Value
@Builder
public class Car {

    private String name;
    private String color;

    // 아래 코드만 직접 작성
    public static CarBuilder {

        // null 필드가 있으면 NPE 발생
        public Car build() {
            if (this.name == null) {
                throw new NullPointException();
            }
            if (this.color == null) {
                throw new NullPointException();
            }
            return new Car(this.name, this.color);
        }
    }
}
```

물론 이 방식은 다시 코드가 복잡해지고, 필드가 많아질수록 검증도 늘어납니다. 필드가 `List` 같은 컬렉션이면 내부 요소까지 검사해야 할 수도 있습니다. 결국 편의성과 안전성 사이에서 어느 쪽을 더 우선할지 판단해야 합니다.

## final 선언

경우에 따라서는 Bean 형태를 써야 할 수도 있습니다. 예를 들어 어떤 필드는 `null`이어도 괜찮고, 이후 다른 메서드에서 검증하거나 보완하는 방식이 더 자연스러운 경우도 있습니다.

Bean을 쓸 때는 보통 필드를 `private`로 두고 외부에서 직접 접근하지 못하게 만드는 것이 기본입니다. 필드를 `public`으로 열어 두면 어디서나 값을 바꿀 수 있어서, 언제 상태가 바뀌었는지 파악하기 어려워지기 때문입니다. 그래서 Setter로 값을 넣고 Getter로 값을 읽는 방식이 일반적인 규칙처럼 자리잡았습니다.

하지만 Setter를 거친다고 해서 충분히 안전한 것은 아닙니다. 여전히 값은 몇 번이든 바뀔 수 있기 때문입니다. 한 번 값을 넣었다가 나중에 다시 Setter를 호출하면 기존 값은 덮어써집니다.

이 문제를 줄이기 위해 사용할 수 있는 것이 `final`입니다. `final`로 선언된 필드는 초기화 이후 다시 값을 대입할 수 없으므로 안정성이 훨씬 높아집니다. 그리고 잘못 값을 다시 넣으려 하면 컴파일 타임에 걸리기 때문에 실수를 빨리 발견할 수 있다는 장점도 있습니다.

모든 필드를 `final`로 선언한 경우, `@Data`가 생성하는 코드는 사실상 `@Value`와 비슷해집니다. 물론 일부 필드만 `final`로 두는 식의 절충도 가능합니다.

```java
// @Data를 사용한 경우
public class Car {

    // color만 final
    private String name;
    private final String color;

    // 아래 메서드들이 자동 생성됨
    public Car(String color) {
        this.color = color;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }

    public String getColor() {
        return this.color;
    }
}
```

다만 여기서 주의할 점은, `final`이 보장하는 것은 어디까지나 "이 클래스의 필드 참조가 바뀌지 않는다"는 점입니다. 필드가 참조하는 객체 내부까지 항상 불변이라는 뜻은 아닙니다.

## 얕은 복사와 깊은 복사

Setter 메서드의 또 다른 문제는, 객체를 복사했다고 생각했는데 실제로는 같은 객체를 함께 바라보는 상황이 생길 수 있다는 점입니다.

예를 들어 아래 코드를 보겠습니다.

```java
// @Data 기반 Car 클래스
Car car1 = new Car();
car1.setName("My car");
car1.setColor("red");

// 친구가 같은 차를 샀다고 가정
Car car2 = car1;
car2.setName("Your car");

// 출력해 본다
System.out.println(car1);
System.out.println(car2);
```

이 코드를 실행해 보면 `car1`과 `car2` 모두 `name`이 `Your car`가 되어 있을 것입니다.

이유는 단순합니다. 이 코드는 `car1`의 값을 복사한 것이 아니라, `car1`과 `car2`가 같은 객체를 가리키도록 만든 것이기 때문입니다. 같은 메모리 주소를 함께 참조하니, 한쪽에서 값을 바꾸면 다른 쪽도 영향을 받습니다.

이런 식으로 객체의 "참조만 복사되는" 상황을 얕은 복사(shallow copy)라고 합니다. primitive type이 값 자체를 복사하는 것과는 다릅니다.

반대로, 같은 값을 가진 새 객체를 별도로 만들어서 서로 다른 참조를 갖게 하면 한쪽 변경이 다른 쪽에 영향을 주지 않습니다. 이것이 깊은 복사(deep copy)입니다.

가장 단순한 방법은 필드 객체를 새로 만들어 주는 것입니다.

```java
// 인스턴스를 만들고 값 넣기
Car car2 = new Car();
car2.setName(new String(car1.getName()));
car2.setName(new String(car1.getColor()));

// 값 변경해 보기
car2.setName(new String("Your car"));
```

이렇게 하면 `car2`만 바뀌는 것을 확인할 수 있습니다. 다만 모든 필드에 대해 일일이 이런 작업을 하는 것은 번거롭습니다.

그래서 자주 쓰는 방법 중 하나가 `Cloneable`을 이용하는 것입니다.

```java
@Data
public class Car implements Cloneable {

    private String name;
    private String color;

    // clone 메서드 작성
    public Car clone() throws CloneNotSupportedException {
        return (Car) super.clone();
    }
}
```

그러면 아래처럼 복사할 수 있습니다.

```java
try {
    car2 = car1.clone();
    car2.setName("Your car");
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
```

하지만 여기에도 주의점이 있습니다. `clone()`을 만들었다고 해서 모든 필드가 자동으로 깊은 복사가 되는 것은 아닙니다. 예를 들어 `Car` 내부에 `Engine` 같은 별도 객체가 필드로 들어 있다면, 그 필드는 여전히 얕은 복사가 될 수 있습니다.

그 경우에는 해당 객체도 함께 복제하도록 만들어야 합니다.

```java
@Data
public class Car implements Cloneable {

    private String name;
    private String color;
    // 추가한 필드
    private Engine engine;

    // 필드도 깊은 복사가 되게 함
    public Car clone() throws CloneNotSupportedException {
        Car car = (Car) super.clone();
        car.engine = this.engine.clone();
        return car;
    }
}

@Data
public class Engine implements Cloneable {

    private String name;
    private String cylinders;

    // clone 메서드 작성
    public Engine clone() throws CloneNotSupportedException {
        return (Engine) super.clone();
    }
}
```

`List`나 `Map`도 결국 안에 들어 있는 객체를 하나씩 복제해 새 컨테이너에 넣는 방식으로 처리할 수 있습니다.

```java
// List 복사
List<Car> carList1 = new ArrayList<>();
List<Car> carList2 = new ArrayList<>();
for (Car car : catList1) {
    carList2.add(car.clone());
}

// Map 복사
Map<String, Car> carMap1 = new HashMap<>();
Map<String, Car> carMap2 = new HashMap<>();
for (Entry<String, Car> entry : carMap1.entrySet()) {
    carMap2.put(entry.getKey(), entry.getValue().clone());
}
```

이렇게 하면 더 안전한 복사 구조를 만들 수 있습니다.

## Immutable 클래스에서 주의할 점

[Reflection 글](../java-reflection)에서도 소개했듯이, Reflection을 사용하면 `private` 필드에도 직접 접근할 수 있습니다. 즉, 아무리 Immutable하게 설계한 클래스라고 해도 Reflection을 쓰면 값을 바꾸는 것이 완전히 불가능한 것은 아닙니다.

또 Immutable 클래스를 많이 사용한다는 것은, 결국 새로운 객체를 더 자주 만들어 낸다는 뜻이기도 합니다. 즉 메모리 사용량이 늘어날 수 있습니다. 물론 요즘 환경에서는 이 정도를 크게 걱정할 필요가 없는 경우가 많고, GC도 충분히 잘 동작합니다. 그래도 "불변 객체는 공짜가 아니다"라는 점 정도는 알고 있는 편이 좋겠습니다.

## Singleton 클래스의 필드도 조심해야 한다

이전 글에서 다룬 [Singleton 클래스](../java-design-pattern-singleton)와도 연결되는 이야기입니다. Singleton은 애플리케이션 종료 시점까지 하나의 인스턴스를 계속 사용하기 때문에, 내부 필드가 바뀔 수 있다면 굉장히 위험해질 수 있습니다. 요청이나 실행 흐름에 따라 결과가 달라질 수 있기 때문입니다.

그래서 Singleton 클래스에는 가능하면 mutable한 필드를 두지 않거나, 최소한 `final`로 막아 두는 것이 좋습니다. 값이 바뀔 가능성을 구조적으로 줄여 두는 편이 훨씬 안전합니다.

## 마지막으로

개인적으로 프로그래밍의 시작이 "어떻게 구현할까"라면, 프로그래밍의 완성은 "어떻게 안정적으로 유지할까"라고 생각합니다. 성능 개선이나 유지보수성도 물론 중요하지만, 결국 가장 어려운 것은 안정적으로 동작하는 프로그램을 만드는 일이라고 느낍니다. 요구 사항이 많아질수록 코드는 복잡해지고, 예외가 발생할 가능성도 커지기 때문입니다.

Immutable 클래스를 만든다는 것은 그런 예외 상황을 줄이기 위한 한 걸음이라고 생각합니다. 즉, 더 신뢰할 수 있는 프로그램을 만들기 위해 꼭 알아 둘 만한 중요한 개념이라는 뜻입니다.

앞으로도 이런 기본기를 계속 익혀 두고 싶습니다.
