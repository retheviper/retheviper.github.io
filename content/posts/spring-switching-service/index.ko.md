---
title: "Service 구현 클래스를 YAML로 선택하기"
date: 2020-02-25
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - yaml
  - java
  - spring
translationKey: "posts/spring-switching-service"
---

Spring에서 비즈니스 로직을 작성할 때는 보통 `Service` 클래스를 만듭니다. `Service`는 핵심 처리를 담고 있어서 개발과 테스트 모두에서 중요한 역할을 합니다. 그런데 실제로 개발을 하다 보면, 구현은 했지만 아직 돌릴 수 없는 상황이 생깁니다. 예를 들어 환경이 아직 준비되지 않았거나, 다른 클래스에 의존하는 설계인데 그 클래스가 아직 없을 수도 있습니다. 이런 경우에는 실제 처리를 하지 않고 항상 같은 값을 돌려주는 대체 구현을 두기도 합니다.

이럴 때 미리 여러 `Service` 구현 클래스를 만들어 두고, 외부 설정 파일인 `application.yml`에서 개발 모드인지 테스트 모드인지에 따라 어떤 구현을 쓸지 선택할 수 있으면 편리합니다. 실제로 `Service`는 인터페이스와 구현 클래스(`Impl`)를 분리하는 경우가 많기 때문에, 여러 `Impl` 클래스를 만들어 상황에 따라 다른 Bean으로 등록하는 것도 가능합니다.

이번 글에서는 YAML 설정을 읽어서 어떤 상황에서 어떤 `ServiceImpl`을 Bean으로 등록할지 결정하는 방법을 정리해 보겠습니다.

## Service의 기본 구조

일반적인 `Service` 구조는 아래와 같습니다. 인터페이스를 만들고, 이를 구현한 클래스를 작성하는 방식입니다. 중간에 DB 같은 외부 시스템과 연동하는 클래스를 DI해서 쓰는 경우가 많습니다.

```java
public interface SomeService {

    public void doSomething();
}

@Service
public class SomeServiceImpl implements SomeService {

    private SomeRepository repository;

    @Autowired
    public SomeServiceImpl(SomeRepository repository) {
        this.repository = repository;
    }

    @Override
    public void doSomething() {
        // ...
    }
}
```

테스트용으로 별도 구현 클래스를 하나 더 만든다고 해 봅시다.

```java
@Service
public class SomeTestServiceImpl implements SomeService {

    private SomeTestRepository testRepository;

    @Autowired
    public SomeServiceImpl(SomeTestRepository testRepository) {
        this.testRepository = testRepository;
    }

    @Override
    public void doSomething() {
        // ...
    }
}
```

이렇게 되면 애플리케이션을 시작할 때 Spring이 "이 인터페이스의 구현체를 어느 걸 써야 하지?" 하고 헷갈리게 됩니다. 하나의 인터페이스에 두 개의 구현 클래스가 있고, 둘 다 Bean으로 등록하려고 하기 때문입니다.

## 어노테이션 제거

`Service` 클래스에는 보통 `@Service`를 붙입니다. 이 어노테이션을 붙이면 Spring이 자동으로 Bean으로 등록해 줍니다. 그런데 하나의 인터페이스에 `@Service`가 붙은 클래스가 둘 이상 있으면, Spring은 어떤 구현을 써야 할지 결정할 수 없습니다. 그래서 여기서는 `@Service`를 직접 붙이지 않는 방향으로 갑니다.

## YAML 만들기

YAML 설정은 단순합니다. 이번에는 boolean 값을 이용해 `true`일 때는 테스트 모드(`SomeTestServiceImpl` 사용), `false`일 때는 일반 모드(`SomeServiceImpl` 사용)로 동작하게 해 보겠습니다.

```yml
settings:
  TestMode: true
```

이 설정을 `application.yml`에 직접 넣어도 되고, 별도 파일로 분리해도 됩니다. 별도 파일을 쓸 경우에는 `application.yml`에서 해당 파일을 포함하도록 설정하는 것을 잊지 않으면 됩니다.

```yml
spring:
  profile:
    include: settings # 파일명이 application-settings.yml인 경우
```

## Configuration 설정

어노테이션을 제거하면 `ServiceImpl`은 자동으로 Bean 등록되지 않습니다. 그렇다고 원하는 구현 클래스를 직접 선택하지 않을 수는 없습니다. 결국 상황에 맞는 구현을 골라 Bean으로 등록해 주는 설정 클래스가 필요합니다. 동시에 YAML 설정도 읽어야 하므로, 이를 `@Configuration` 클래스로 묶어 처리하면 됩니다.

```java
@Configuration
public class SomeServiceConfig {

    // YAML에 설정한 값을 읽는다
    @Value("${settings.TestMode:false}")
    private boolean testMode;

    // YAML 설정에 따라 어떤 Impl을 Bean으로 등록할지 결정한다
    @Bean
    public SomeService someService(SomeRepository repository, SomeTestRepository testRepository) {
        return this.testMode ? new SomeTestSerivce(testRepository) : new SomeServiceImple(repository);
    }
}
```

Spring에서는 `@Value`나 `@ConfigurationProperties`로 YAML 값을 읽을 수 있습니다. `@ConfigurationProperties`는 클래스 전체 필드에 값을 매핑할 때 유용하지만, 여기서는 boolean 하나만 읽으면 되므로 필드 단위로 쓰기 쉬운 `@Value`를 사용합니다. YAML 파일이 없을 때를 대비해 기본값으로 `false`도 지정해 두었습니다.

`@Bean`으로 등록할 때는 단순히 새 인스턴스를 만들어 반환하면 됩니다. 여기서는 구현 클래스가 의존하는 Repository도 함께 필요하므로, 메서드 인자로 받아 Bean으로 주입받습니다. 메서드 인자에 Repository 타입을 적어 두면 Spring이 자동으로 해당 Bean을 넣어 줍니다. 이후에는 테스트 모드인지 일반 모드인지에 따라 적절한 구현체를 반환하면 됩니다.

## `@Profile`을 쓰는 경우

Bean을 상황별로 바꾸는 데는 `@Profile`을 쓰는 방법도 있습니다. 이 방식도 어렵지 않습니다. 먼저 YAML을 다음처럼 정의했다고 해 봅시다.

```yml
spring:
  profile:
    active: dev
```

이후에는 어떤 프로필에서 어떤 Bean을 사용할지 어노테이션으로 지정하면 됩니다.

```java
@Configuration
public class SomeServiceConfig {

    // dev나 debug일 때 등록할 Bean
    @Bean
    @Profile({"dev", "debug"})
    public SomeService someService(SomeTestRepository testRepository) {
        return new SomeTestSerivce(testRepository);
    }

    // prod일 때 등록할 Bean
    @Bean
    @Profile("prod")
    public SomeService someService(SomeRepository repository) {
        return new SomeServiceImple(repository);
    }
}
```

`@Profile`은 배열로 여러 프로필을 지정할 수 있어서, YAML 설정에 따라 어떤 Bean을 등록할지 비교적 간단하게 바꿀 수 있습니다. 두 방식 모두 쓸 수 있으니, 상황에 맞는 쪽을 고르면 됩니다.

## 마지막으로

요즘은 Django나 Express처럼 다른 언어의 웹 프레임워크도 더 접해 보고 싶지만, 아직은 Spring에서 배울 것이 훨씬 더 많습니다. 기능을 하나씩 알아 갈수록, 다른 프레임워크가 많아도 Spring이 엔터프라이즈 시장에서 오래 살아남은 이유는 결국 "할 수 있는 일이 많다"는 데 있다는 생각이 듭니다. 당분간은 Spring만으로도 충분히 더 쓸 이야기가 나올 것 같습니다.
