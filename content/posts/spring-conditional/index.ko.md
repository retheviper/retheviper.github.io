---
title: "조건에 따라 동작하는 어노테이션 활용하기"
date: 2020-04-26
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - annotation
  - yaml
  - java
  - spring
translationKey: "posts/spring-conditional"
---

어노테이션은 일반 Java에서도 쓸 수 있고, 여러 라이브러리와 프레임워크에서 적극적으로 활용합니다. 그중에서도 어노테이션을 가장 잘 쓰는 곳은 Spring이라고 생각합니다. DI를 하든 클래스의 역할을 구분하든, 대부분을 어노테이션으로 간단하게 정의할 수 있게 해 주기 때문입니다. 그래서 Spring으로 Web 애플리케이션을 만들 때는 어떤 어노테이션이 있는지 익혀 두는 것이 중요합니다.

이 이야기를 꺼낸 이유는, [Service 구현 클래스를 YAML로 선택하기](/ko/posts/spring-switching-service/)에서 Service를 바꾸는 방법을 설명한 뒤 다른 방법이 있는지 찾아보다가, Spring에서는 조건에 따라 Bean이나 Configuration을 등록하는 어노테이션도 있다는 걸 알게 됐기 때문입니다.

또 조건에 따라 쓸 수 있는 어노테이션이 여러 가지라서, 이번에는 그중 자주 보이는 것들을 정리해 보려고 합니다. 이런 어노테이션들은 Spring Boot의 auto-configuration 계열과 맞닿아 있습니다. 이미 많은 설정이 자동화되어 있지만, 그 자동 설정을 더 세밀하게 다룰 수 있도록 도와주는 도구라고 보면 됩니다.

참고로, 여기서 소개하는 것들을 전부 직접 실험해 본 것은 아니고, 일단 정리 차원에서 소개하는 글입니다.

## 조건에 따라 DI하기

이전 글에서 소개한 코드를 다시 보겠습니다. `application.yml`에 적힌 값에 따라 어떤 Repository를 DI할지 결정하는 방식이었습니다.

```java
@Configuration
public class SomeServiceConfig {

    // YAML에 설정한 값을 읽는다
    @Value("${settings.TestMode:false}")
    private boolean testMode;

    // YAML 설정에 따라 어떤 Impl을 쓸지 결정해 Bean 등록
    @Bean
    public SomeService someService(SomeRepository repository, SomeTestRepository testRepository) {
        return this.testMode ? new SomeTestSerivce(testRepository) : new SomeServiceImple(repository);
    }
}
```

이와 비슷한 동작을 Spring의 조건부 어노테이션으로 구현하면 이렇게 바뀝니다.

```java
@Configuration
public class SomeServiceConfig {

    // 운영용 서비스 클래스
    @Bean
    @ConditionalOnProperty(value = "settings.Testmode", havingValue = "false")
    public SomeService someService(SomeRepository repository) {
        return new SomeServiceImple(repository);
    }

    // 테스트용 서비스 클래스
    @Bean
    @ConditionalOnProperty(value = "settings.Testmode", havingValue = "true")
    public SomeService someTestService(SomeTestRepository testRepository) {
        return new SomeTestService();
    }
}
```

`@ConditionalOnProperty`를 쓰면 특정 조건에 따라 메서드가 실행되도록 만들 수 있습니다. 굳이 분기문을 직접 쓰거나 커스텀 `@Value` 처리를 만들지 않아도 되니 훨씬 깔끔합니다.

이 어노테이션은 `@Bean`뿐 아니라 다른 Spring 클래스 어노테이션인 `@Configuration`, `@Component`, `@Service`, `@Repository`, `@Controller`에도 쓸 수 있어서 활용 범위가 넓습니다.

Spring Boot의 `org.springframework.boot.autoconfigure.condition` 패키지에는 이런 식의 `@Conditional...` 어노테이션이 더 있습니다. 하나씩 간단히 보겠습니다.

## 미리 정의된 Conditional 어노테이션

### `@ConditionalOnProperty`

`application.yml`이나 시스템 프로퍼티에 지정한 값이 있을 때 실행되는 어노테이션입니다. Spring Boot에서 가장 자주 쓰이는 조건부 어노테이션 중 하나입니다. 괄호 안에서 프로퍼티 이름과 기대값을 지정하고, 값이 없을 때도 동작하게 할지 정할 수 있습니다.

아래 코드는 `use.testmode`가 존재하고 값이 `true`일 때 실행되는 Configuration 예시입니다. `matchIfMissing`를 `true`로 두면 해당 프로퍼티가 없어도 실행됩니다.

```java
@Configuration
@ConditionalOnProperty(value = "use.testmode", havingValue = "true", matchIfMissing = true)
public class TestConfig {
  // 테스트 모드에서 사용할 Configuration
}
```

### `@ConditionalOnExpression`

프로퍼티를 표현식으로 판단해 실행하는 어노테이션입니다. 괄호 안에 조건식을 넣으면 됩니다. 이때 조건식은 `@Value` 등에서도 쓰는 [SpEL](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions)을 사용합니다. 아래 예시는 `use.testmode`와 `use.submode`가 모두 `true`일 때 실행됩니다.

```java
@Configuration
@ConditionalOnExpression("${use.testmode:true} and ${use.submode:true}")
public class TestSubConfig {
  // 테스트 모드와 서브 모드가 모두 true일 때 사용할 Configuration
}
```

### `@ConditionalOnBean`

지정한 Bean이 존재할 때 실행되는 어노테이션입니다. 괄호 안에 Bean으로 등록됐는지 확인할 클래스를 넣으면 됩니다. 예를 들어 어떤 Bean이 등록됐을 때 그에 맞는 보조 설정도 함께 넣고 싶을 때 사용할 수 있습니다.

```java
@Configuration
@ConditionalOnBean(TestRepository.class)
public class TestBeanConfig {
  // 테스트용 Bean이 등록됐을 때 사용할 Configuration
}
```

### `@ConditionalOnMissingBean`

`@ConditionalOnBean`의 반대입니다. 지정한 Bean이 없을 때 실행됩니다.

```java
@Configuration
public class AlternativeConfigutration {

    @Bean
    @ConditionalOnMissingBean
    public Repository testRepository() {
        return new TestRepository();
    }
}
```

### `@ConditionalOnResource`

리소스 파일이 존재할 때 실행되는 어노테이션입니다. 예를 들어 Logback을 쓸 때 `xml` 파일이 있으면 그 설정도 함께 활성화하고 싶을 때 사용할 수 있습니다.

```java
@Configuration
@ConditionalOnResource(resources = "/logback.xml")
public class LogbackConfig {
  // resources 폴더에 logback.xml이 있을 때 사용하는 Configuration
}
```

### `@ConditionalOnClass`

지정한 클래스가 존재할 때 실행됩니다. `@ConditionalOnBean`과 비슷하지만, Bean이어야 할 필요는 없습니다. 외부 라이브러리가 클래스패스에 들어왔는지 확인할 때 유용합니다.

```java
@Configuration
@ConditionalOnClass(name = "com.custom.library.module.SomeClass")
public class CustomLibraryConfig {
  // 커스텀 라이브러리의 SomeClass가 있을 때 사용하는 Configuration
}
```

### `@ConditionalOnMissingClass`

`@ConditionalOnClass`의 반대입니다. 지정한 클래스가 없을 때 실행됩니다.

```java
@Configuration
@ConditionalOnMissingClass(name = "com.custom.library.module.SomeClass")
public class NoCustomLibraryConfig {
  // 커스텀 라이브러리의 SomeClass가 없을 때 사용하는 Configuration
}
```

### `@ConditionalOnJava`

애플리케이션이 어떤 Java 버전에서 실행되는지에 따라 동작을 바꾸는 어노테이션입니다. 버전별로 API가 달라질 수 있으니, 여러 환경을 지원해야 할 때 유용합니다.

```java
@Configuration
@ConditionalOnJava(JavaVersion.EIGHT)
public class JavaEightConfig {
  // Java 1.8에서 사용하는 Configuration
}
```

## 커스텀 Condition

`Condition` 인터페이스를 직접 구현하면 커스텀 조건도 만들 수 있습니다. 예를 들어 애플리케이션이 Linux에서 실행될 때만 동작하도록 만들 수 있습니다.

핵심은 `matches()` 메서드가 `boolean`을 반환하도록 구현하는 것입니다.

```java
class OnUnixCondition implements Condition {

    // OS가 Linux인지 판별하는 Condition
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return SystemUtils.IS_OS_LINUX;
    }
}
```

이렇게 만든 Condition은 `@Conditional`에 붙여서 사용합니다.

```java
@Configuration
public class OsConfig {

    // Linux일 때 Bean 등록
    @Bean
    @Conditional(OnUnixCondition.class)
    public UnixBean unixBean() {
        return new UnixBean();
    }
}
```

`AnyNestedCondition`을 상속하면 더 복잡한 조건도 만들 수 있습니다. 앞의 `OnUnixCondition` 외에 Windows 여부를 판별하는 `OnWindowsCondition`도 있다고 해 봅시다. 이런 경우는 아래처럼 만들 수 있습니다.

```java
public class OnWindowsOrUnixCondition extends AnyNestedCondition {

    OnWindowsOrUnixCondition() {
        super(ConfigurationPhase.REGISTER_BEAN);
    }

    @Conditional(OnWindowsCondition.class)
    public static class OnWindows {
        // Windows일 때
    }

    @Conditional(OnUnixCondition.class)
    public static class OnUnix {
        // Linux일 때
    }
}
```

이렇게 만든 Condition도 같은 방식으로 `@Conditional`에 넣어 사용합니다.

```java
@Configuration
public class OsConfig {

    // Windows 또는 Linux일 때 등록되는 Bean
    @Bean
    @Conditional(OnWindowsOrUnixCondition.class)
    public WindowsOrUnixBean windowsOrUnixBean() {
        return new WindowsOrUnixBean();
    }
}
```

## 마지막으로

여기서 소개한 것 말고도 `org.springframework.boot.autoconfigure.condition` 패키지 아래에는 여러 조건부 클래스가 더 있습니다. 예를 들어 Web 애플리케이션인지, 클라우드 플랫폼인지에 따라 동작을 바꾸는 어노테이션도 있고, 앞으로도 여러 조건이 더 추가될 수 있습니다.

이런 `Conditional` 계열 어노테이션은 Spring Boot의 Auto Configuration에서도 쓰이는 것들입니다. 직접 프로퍼티를 읽고 `if`를 쓰는 것보다 훨씬 안정적이고, 조건별로 어노테이션이 따로 존재하니 재사용성도 좋습니다. 커스텀 Condition까지 만들 수 있으니 꽤 유용합니다.

직접 프로퍼티를 읽고 `if`로 분기하는 방식도 물론 가능합니다. 하지만 Spring이 이미 제공하는 조건부 어노테이션을 활용하면 코드가 더 분명해지고, 설정 의도도 훨씬 잘 드러납니다. 이런 도구를 알고 있느냐가 결국 Spring 코드를 얼마나 깔끔하게 유지할 수 있는지와도 연결된다고 느꼈습니다.
