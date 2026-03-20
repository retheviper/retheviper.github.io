---
title: "new로 만든 인스턴스에서 Bean 쓰기"
date: 2019-12-22
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - dependency injection
  - java
  - spring
translationKey: "posts/spring-bean-with-yaml"
---

일반적인 Java 프로젝트라면 외부 설정 파일(YAML)을 작성해서 그 값을 읽어들이는 것은 [이전 글](../java-yaml-for-configuration)처럼 할 수 있습니다. 그런데 이번에는 Spring 프로젝트에서 비슷한 일을 하게 되었습니다. Spring은 YAML을 읽을 때만의 방식과 규칙이 있고, 읽어들인 값은 Bean으로 만들 수 있어서 애플리케이션 안 어디서든 `@Autowired`로 꺼내 쓸 수 있다는 장점이 있습니다.

하지만 편리한 DI에도 불편한 점은 있습니다. 예를 들어 `new`로 직접 만든 인스턴스 안에서는 `@Autowired`를 쓸 수 없습니다. 이번에도 그 부분에서 꽤 오래 막혔습니다. Builder로 객체를 만들되, 사용자가 따로 지정하지 않은 값은 YAML에서 읽어 온 Bean을 기본값으로 쓰고 싶었습니다. 그런데 Builder는 새 인스턴스를 만들어 버리니 Bean을 주입받지 못해서 막혀 있었습니다.

결국 다른 방법을 쓰면 `@Autowired` 없이도 Bean을 가져올 수 있다는 걸 알게 됐습니다. 이번 글에서는 그 과정을 코드와 함께 정리해 보겠습니다. YAML을 만들고, `new`한 인스턴스 안에서 Bean을 꺼내 쓰는 방법까지 살펴보겠습니다.

## YAML에서 Bean 만들기

Spring에서는 `application.yml`에 다음처럼 적어 특정 YAML 파일을 읽도록 설정할 수 있습니다.

```yaml
spring:
  profiles:
    active: buildingdefault
```

`active`에 적은 값에 맞춰 `application-` 접두사가 붙은 커스텀 YAML 파일을 준비합니다. 따라서 이번 파일 이름은 `application-buildingdefault.yml`이 됩니다.

파일을 만들고 아래처럼 항목과 값을 적습니다.

```yaml
settings:
  material: "cement"
```

작성한 YAML 파일은 `src/main/resource`에 넣습니다. 이제 Spring에서 이 YAML을 읽어들이는 클래스를 만들 차례입니다.

Spring에서 YAML을 읽어 Bean으로 만드는 방법은 두 가지가 있습니다. 첫 번째는 필드에 어노테이션을 붙여 YAML의 항목과 직접 연결하는 방식입니다.

```java
@Getter
@Component
public class DefaultSettings {

    @Value("${settings.material}")
    private String material;
}
```

필드에 `@Value`를 붙이고, 어노테이션 값으로 YAML 항목명을 지정합니다. 그러면 YAML에서 읽어 온 값이 `String` 형태로 Bean에 들어갑니다. 필드가 꼭 `String`일 필요는 없고, `int`나 `double` 같은 기본형도, `ENUM`도 사용할 수 있습니다. `Locale`도 `ja_JP`처럼 적어 두면 잘 읽힙니다.

YAML 값을 Bean으로 만드는 두 번째 방법은 필드가 아니라 클래스에 어노테이션을 붙이는 것입니다. 아래처럼 `@ConfigurationProperties`에 prefix를 지정하면, 해당 경로 아래의 값이 모두 필드 매핑 대상이 됩니다.

```java
@Data
@Configuration
@ConfigurationProperties(prefix = "settings")
public class DefaultSettings {

    private String material;
}
```

## 여러 설정을 읽고 싶을 때

YAML에 설정을 여러 개 넣고 상황에 따라 골라 쓰고 싶을 수도 있습니다. YAML은 배열을 쓸 수 있고, Spring도 이를 `List`로 받을 수 있습니다. 그래서 여러 설정을 Bean으로 만드는 방법을 살펴보겠습니다.

YAML은 다음처럼 작성할 수 있습니다.

```yaml
settings:
  - preset-name: "default"
    material: "cement"
  - preset-name: "cabin"
    material: "wood"
```

여기서 `preset-name`은 Java에서 여러 설정 세트를 구분하기 위한 키 역할을 합니다. 없어도 읽기는 되지만, 이름을 붙여 두면 나중에 어떤 값인지 알아보기 쉽습니다.

YAML 작성이 끝나면 각 설정 세트에 맞는 Bean 클래스를 만들어 둡니다.

```java
@Data
public class Material {

    private String presetName;

    private String material;
}
```

마지막으로 YAML을 읽는 클래스를 만듭니다. 이 클래스에 Bean 목록을 필드로 두면 Spring 애플리케이션이 시작될 때 함께 읽힙니다.

```java
@Data
@Configuration
@ConfigurationProperties(prefix = "settings")
public class MultiSettings {

    private List<Material> presets;
}
```

이후에는 이 클래스에서 `List`를 꺼내 `presetName`으로 원하는 설정을 찾으면 됩니다.

## Builder에서 Bean을 쓰려다 실패한 예

이제 평범한 Spring 애플리케이션 안에서는 Bean을 DI해서 쓸 수 있습니다. 하지만 이번 글은 DI 없이 Bean을 가져오는 방법을 보여 주는 것이므로, 제가 먼저 시도했다가 실패한 방법부터 보겠습니다.

제가 하고 싶었던 것은 Builder 안에서 YAML 값을 읽어 둔 Bean을 쓰는 것이었습니다. YAML에 적은 값은 기본값으로 쓰고, 필요하면 일부 항목만 `build()` 시점에 덮어쓰고 싶었습니다. 하지만 처음 시도한 코드는 아래처럼 잘 되지 않았습니다.

```java
public class Building {

    public static BuildingBuilder() {
        return new BuildingBuilder();
    }

    public static class BuildingBuilder {

        // DI가 안 된다
        @Autowired
        private DefaultSettings settings;

        private String material;

        public BuildingBuilder() {
            this.material = this.settings.getMaterial();
        }

        ...
    }
}
```

Builder는 결국 새 인스턴스를 만들어야 합니다. 그런데 이렇게 `new`한 인스턴스 안에서는 `@Autowired`를 붙여도 DI가 제대로 되지 않습니다. 실제로 위처럼 작성하면 Bean 필드가 `null`인 것을 확인할 수 있습니다.

그래서 DI에 기대지 말고, `new`한 인스턴스 안에서 직접 Bean을 꺼내 쓰는 방법을 택해야 했습니다.

## `ApplicationContextProvider` 만들기

`ApplicationContext`는 Spring에서 Bean 생성과 객체 간 관계 설정을 담당하는 인터페이스입니다. 중요한 것은 Spring 애플리케이션이 시작될 때 미리 등록된 Bean을 생성하고 관리한다는 점입니다. 즉 이 인터페이스에 접근할 수 있으면 Bean을 가져올 수 있습니다.

다만 `ApplicationContext` 자체는 인터페이스이므로, 인스턴스를 얻으려면 그 역할을 해 줄 클래스를 하나 만들어야 합니다. 아래처럼 만들면 됩니다.

```java
@Component
public class ApplicationContextProvider implements ApplicationContextAware {
     
    private static ApplicationContext context = null;
 
    public static ApplicationContext getApplicationContext() {
        return this.context;
    }
 
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        this.context = context;
    }
}
```

구조는 단순합니다. `ApplicationContext`를 담는 필드와 Getter, Setter만 있으면 됩니다. Spring 애플리케이션이 동작하면 이 클래스의 Setter를 통해 `ApplicationContext`가 자동으로 들어옵니다. 다만 이 클래스는 직접 `new`해서 쓰는 구조가 아니므로 필드와 Getter는 `static`으로 두어야 합니다.

## Builder에서 Bean을 쓰는 성공 예

이제 `ApplicationContext`를 얻을 수 있으니 Builder를 수정해 봅시다.

```java
public class Building {

    public static BuildingBuilder() {
        return new BuildingBuilder();
    }

    public static class BuildingBuilder {

        private DefaultSettings settings;

        public BuildingBuilder() {
            this.settings = ApplicationContextProvider.getApplicationContext().getBean(DefaultSettings.class);
        }

        ...
    }
}
```

앞서 만든 `ApplicationContextProvider`에서 `ApplicationContext`를 가져오고, 거기서 `getBean()`을 호출합니다. 이때 가져오려는 Bean 클래스만 넘기면 인스턴스를 얻을 수 있습니다. 물론 생성자 안이 아니라 필드 초기화 시점에 바로 써도 됩니다.

```java
public class Building {

    public static BuildingBuilder() {
        return new BuildingBuilder();
    }

    public static class BuildingBuilder {

        private DefaultSettings settings = ApplicationContextProvider.getApplicationContext().getBean(DefaultSettings.class);

        public BuildingBuilder() {
        }

        ...
    }
}
```

이렇게 고친 뒤 실행해 보면, Bean 필드가 `null`이 아니라 YAML에서 읽은 값으로 들어오는 것을 확인할 수 있습니다.

## 마지막으로

Spring을 쓰면서도 실제 애플리케이션 내부에서 어떤 일이 일어나는지 제대로 모르고 있었다는 점이 이번엔 꽤 부끄럽게 느껴졌습니다. 단순히 "동작한다"는 것만 확인할 게 아니라, 내가 쓰는 언어나 프레임워크가 왜 그렇게 동작하는지 이해하지 않으면 이런 식으로 막히게 됩니다. 이번에는 새 지식을 얻은 동시에 제 부족함도 다시 보게 되었습니다. 앞으로는 제가 쓰는 것들이 어떻게, 왜 동작하는지 제대로 이해하고 사용해야겠습니다.
