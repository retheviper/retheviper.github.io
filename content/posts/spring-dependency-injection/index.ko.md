---
title: "Spring의 DI는 생성자로 하자"
date: 2019-12-09
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - dependency injection
  - java
  - spring
translationKey: "posts/spring-dependency-injection"
---

Spring의 대표적인 특징을 하나 꼽자면 역시 `@Autowired`를 통한 DI[^1]라고 생각합니다. 처음 Spring을 접했을 때는 `new` 없이 객체가 돌아가는 모습이 꽤 신기했습니다. 나중에 이것이 디자인 패턴의 한 형태라는 걸 알고 나서는 더 놀랐습니다. 좋은 코드를 쓰려면 여러 방향의 고민이 필요하다는 걸 다시 느꼈습니다.

그런데 Spring Boot를 쓰면서 한 가지 궁금한 점이 생겼습니다. 예전에는 `@Autowired`를 필드에 붙이는 것이 익숙했는데, 최근 프로젝트에서는 생성자에 붙이는 경우가 더 많았습니다. 왜 어떤 곳은 필드에 붙이고, 어떤 곳은 생성자에 붙이는지 궁금했고, 결국 모든 주입을 생성자로 바꾸기 전에 차이를 다시 정리해 보기로 했습니다.

결론부터 말하면, 대부분의 경우 `@Autowired`는 필드보다 생성자에 붙이는 편이 낫다고 합니다. 그리고 `@Autowired`를 필드나 생성자에 붙이는 방식은 각각 `Field Injection`과 `Constructor Injection`이라고 부릅니다. 이제 코드로 보겠습니다.

## Field Injection

먼저 주입을 위해 다음과 같은 설정 클래스를 정의했다고 가정해 봅시다.

```java
@Configuration
public class MapperConfig {

    @Bean
    public ModelMapper mapper() {
        return new ModelMapper();
    }
}
```

여기서는 [ModelMapper](http://modelmapper.org/)를 사용하겠습니다. ModelMapper는 이름이 같은 Getter/Setter를 기준으로 Bean 사이를 자동 매핑해 주는 라이브러리입니다.

이렇게 Bean을 등록한 뒤 `@Autowired`를 필드에 붙이는 서비스 클래스는 다음과 같습니다. 이것이 `Field Injection`입니다.

```java
@Service
public class ItemServiceImpl implements ItemService {

    @Autowired
    private ModelMapper mapper;
}
```

저도 Spring을 처음 배울 때는 이런 필드 주입이 일반적이었습니다. 하지만 필드 주입에는 치명적인 문제가 있습니다. 필드가 `null`이어도 프로그램이 일단 실행된다는 점입니다. 클래스가 정상 동작하는 데 필요한 조건이 충족되지 않았는데도 실행되어 버리면 버그를 만들기 쉽습니다. 그래서 필드 주입은 좋지 않습니다.

## Setter Injection

사실 주입은 Setter를 통해서도 가능합니다. 아주 일반적이진 않지만, 이를 `Setter Injection`이라고 부릅니다.

```java
@Service
public class ItemServiceImpl implements ItemService {

    private ModelMapper mapper;

    @Autowired
    public void setMapper(ModelMapper mapper) {
        this.mapper = mapper;
    }
}
```

Setter 주입의 문제는 필드 주입과 비슷합니다. 필요한 객체가 실제로 주입되었는지와 무관하게 프로그램이 실행될 수 있습니다. 이 문제를 막아 주는 방식이 다음에 나오는 생성자 주입입니다.

## Constructor Injection

생성자를 통한 주입은 `Constructor Injection`이라고 하며, 코드는 다음과 같습니다.

```java
@Service
public class ItemServiceImpl implements ItemService {

    private final ModelMapper mapper;

    @Autowired
    public ItemServiceImpl(ModelMapper mapper) {
        this.mapper = mapper;
    }
}
```

생성자 주입의 장점은 앞서 말한 문제를 비교적 일찍 드러낸다는 점입니다. 이건 Spring보다는 Java 언어 차원의 이야기이기도 한데, 생성자 인자를 만족하지 못하면 애초에 인스턴스를 만들 수 없습니다. 따라서 필요한 의존성이 빠진 상태로 객체가 만들어질 가능성을 줄일 수 있습니다.

또한 생성자 주입은 순환 참조[^3]를 미리 막는 데도 도움이 됩니다. 필드 주입이나 Setter 주입은 실제 코드가 호출되기 전까지 문제를 파악하기 어렵지만, 생성자 주입은 순환 참조가 있으면 애플리케이션 시작 시점에 경고가 뜹니다.

필드 주입은 단위 테스트하기도 불편합니다. `@Autowired`가 붙은 필드에 직접 객체를 넣을 방법이 마땅치 않기 때문입니다. Setter를 쓰면 주입 자체는 가능하지만, 굳이 Setter를 쓸 이유도 크지 않습니다.

생성자 주입이 좋은 또 다른 이유는 필드를 `final`로 둘 수 있다는 점입니다. 필드가 클래스 안에서 바뀌지 않도록 보장할 수 있으니 더 안전합니다.

## 마지막으로

저도 예전에는 필드 주입을 별 생각 없이 썼습니다. 하지만 단위 테스트나 의존성 명시성 같은 문제를 알고 나서는 가능한 한 생성자 주입으로 정리하고 있습니다. IntelliJ도 생성자 주입을 권장하고, Spring 공식 문서도 같은 방향을 이야기하니 지금은 이유가 꽤 분명하다고 느낍니다.

Java 코딩 관점에서도 생성자는 중요합니다. Java에서는 따로 적지 않으면 인자 없는 생성자가 암묵적으로 생기기 때문에, 생성 시점에 필요한 값을 강제하고 싶다면 생성자를 명시적으로 작성해야 합니다. 이런 점까지 생각하면 생성자 주입이 단순히 Spring 스타일의 차이가 아니라, 객체를 더 분명하게 설계하는 방법이라는 점도 이해할 수 있습니다.

[^1]: Dependency Injection(의존성 주입). 자세한 설명은 많으니 여기서는 생략하겠습니다. 간단히 말하면 객체를 외부에서 만들어 코드에 주입해, 코드와 의존성을 분리하는 방식입니다.
[^2]: `@Configuration`을 붙이면 Spring이 설정 클래스로 인식합니다. 여기서 객체를 Bean으로 정의하면 DI가 가능합니다. 예전에는 XML에 적기도 했습니다.
[^3]: 순환 참조란 여러 객체가 서로를 참조하는 상황을 말합니다. 예를 들어 A를 만들 때 B가 필요하고, B를 만들 때 다시 A가 필요하면 서로를 끝없이 찾게 되어 무한 루프에 빠질 수 있습니다.
