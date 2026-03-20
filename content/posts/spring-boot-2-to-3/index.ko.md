---
title: "Spring Boot 3으로 올리기"
date: 2023-01-29
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - kotlin
  - java
  - spring
translationKey: "posts/spring-boot-2-to-3"
---

Spring Boot 2(Spring Framework 5)에서 Spring Boot 3(Spring Framework 6)으로 업그레이드해 봤습니다.

## Java 17

예전에 Java를 업무에서 쓰던 시절에는 Java 17을 꽤 오래 기다렸습니다. 이전에 이 블로그에서도 정리한 적이 있지만, 텍스트 블록이나 `switch`를 식처럼 쓸 수 있는 등 유용한 기능이 많이 추가됐기 때문입니다. 하지만 지금은 Kotlin을 쓰고 있어서, 기능이나 언어 자체의 관점에서 Java 버전을 굳이 신경 쓸 필요는 거의 없습니다. 그래서 기존 애플리케이션이 Java 11로 잘 돌아가고 있다면, 굳이 17로 올릴 동기가 크지 않아 보일 수도 있습니다.

그럼에도 버전을 올린 이유는 몇 가지가 있습니다. 우선 Java 11의 지원 기간이 올해 9월에 끝나기 때문입니다. 정확히는 Java 11의 Premier Support가 2023년 9월에 종료되고, Extended Support는 2026년 9월까지입니다. 다만 11은 이제 꽤 오래된 버전이고, 17도 출시된 지 1년 넘게 검증된 상태라서 올릴 만하다고 판단했습니다.

지원 기간만 보면 아직 종료까지 시간이 남아 있습니다. 하지만 회사에서 "리팩터 기간"을 따로 두고 있어서 이 타이밍에 진행했습니다. 이 기간에는 주로 기술 부채를 정리하고 의존성 버전을 올리기 때문에, 새 기능 개발과 버전 업이 겹쳐서 문제가 생기는 상황은 피하고 싶었습니다.

또 다른 이유는 새 Java 버전이 성능 개선을 가져다줄 수 있다는 점입니다. 예를 들어 [Java 11보다 GC 성능이 좋아졌다](https://www.optaplanner.org/blog/2021/09/15/HowMuchFasterIsJava17.html)거나, 상황에 따라 [ZGC를 고려할 수 있다](https://malloc.se/blog/zgc-jdk17)는 식의 이점이 있습니다. Java 버전이 올라간다는 것은 JVM 자체가 개선된다는 뜻이기도 하므로 Kotlin도 그 이점을 함께 받을 수 있습니다.

그래서 Java 17을 도입하기로 했고, 그 과정에서 겪은 경험을 정리해 보겠습니다.

### `Record`

사내에서는 ORM으로 [jOOQ](https://www.jooq.org/)를 쓰고 있고, 자동 생성된 테이블 정의 코드를 Kotlin 코드(Repository)에서 호출하는 구조입니다. 그런데 경우에 따라 이 자동 생성 코드 때문에 Java 17 컴파일이 실패할 수 있습니다. 이번에는 다음과 같은 오류가 났습니다.

```text
에러: Record 참조가 모호합니다
    public <O extends Record> SomeTable(Table<O> child, ForeignKey<O, SomeRecord> key) {
                      ^
  org.jooq 인터페이스 org.jooq.Record와 java.lang 클래스 java.lang.Record가 모두 일치합니다
```

이건 jOOQ에 `Record`라는 클래스가 있고, 자동 생성 코드가 그것을 사용하기 때문입니다. Java 14 이후에는 Lombok의 `@Value`와 비슷한 기능을 가진 [Record Class](https://docs.oracle.com/en/java/javase/15/language/records.html)가 등장했고, `record` 키워드로 정의한 클래스는 모두 [java.lang.Record](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Record.html)를 구현합니다. 그래서 컴파일러가 "jOOQ의 `Record`인지, Java의 `Record`인지" 구분하지 못해 에러가 납니다.

이 문제는 자동 생성 코드의 import를 조금 바꾸는 것으로 해결할 수 있습니다. 기존 코드는 보통 이렇게 되어 있습니다.

```java
import org.jooq.*;
```

이것을 jOOQ의 `Record`를 명시적으로 쓰도록 바꿉니다.

```java
import org.jooq.Record;
import org.jooq.*;
```

이렇게 수정하면 컴파일이 정상적으로 통과합니다. 만약 직접 만든 라이브러리에서 `Record`라는 클래스를 쓰고 있다면, 여기와 같은 충돌이 생길 수 있으니 클래스 이름을 바꾸는 편이 안전합니다.

### Docker Image

현재 개발 중인 서비스는 [Jib](https://github.com/GoogleContainerTools/jib)으로 컨테이너화하고 있습니다. 여기서 베이스 이미지가 필요한데, 기존에는 OpenJDK의 [11-jre](https://hub.docker.com/layers/library/openjdk/11-jre/images/sha256-762d8d035c3b1c98d30c5385f394f4d762302ba9ee8e0da8c93344c688d160b2?context=explore)를 쓰고 있었습니다. 그런데 17부터는 JRE 전용 이미지가 없고 [17-jdk](https://hub.docker.com/layers/library/openjdk/17-jdk/images/sha256-98f0304b3a3b7c12ce641177a99d1f3be56f532473a528fda38d53d519cafb13?context=explore)만 있으니 버전 업 시 주의해야 합니다.

다만 OpenJDK가 아닌 이미지를 쓰고 있다면 상황이 다를 수 있습니다. 예를 들어 [Temurin](https://hub.docker.com/layers/library/eclipse-temurin/17-jre/images/sha256-402c656f078bc116a6db1e2e23b08c6f4a78920a2c804ea4c2d3e197f1d6b47c?context=explore)이나 [Zulu](https://hub.docker.com/layers/azul/zulu-openjdk/17-jre/images/sha256-09163c13aefbe5e0aa3dc7910d9997e416031e777ca0c50bd9b66a63b46be481?context=explore)는 `17-jre`를 제공합니다. Liberica는 [17](https://hub.docker.com/layers/bellsoft/liberica-openjdk-alpine/17/images/sha256-50feb980f142b152e19c9dc25c835e0e451321eb3b837a3a924f457b11ce8c59?context=explore)처럼 버전만 붙은 경우도 있습니다. 즉, JDK 종류에 따라 태그명이 다르므로 업그레이드할 때는 쓰는 이미지 태그를 꼭 확인해야 합니다.

### 의존성

제가 직접 맡은 앱에서 생긴 문제는 아니지만, 일부 Kotlin 서비스에서 Java 버전을 11에서 17로 올렸을 때 [JasperReports](https://github.com/TIBCOSoftware/jasperreports)를 사용한 보고서 출력 문제를 본 적이 있습니다. PDF 출력에는 문제 없었지만, 표 안에서 보이는 글자 수가 조금 줄어드는 현상이 있었습니다. 다행히 큰 문제는 아니라 우선 대응하지 않았지만, 경우에 따라서는 치명적일 수도 있습니다.

이런 문제가 생기면 의존 라이브러리 버전을 Java 17 대응 버전으로 올리면 해결되는 경우가 많습니다. 하지만 아직 17을 완전히 지원하지 않는 라이브러리도 있을 수 있으니, 미리 의존성을 확인해 두는 편이 좋습니다.

## Spring Boot 3

Java 17은 11의 지원 종료 때문에 사실상 필요했지만, Spring Boot 3(Spring Framework 6)은 작년 12월에 막 릴리스된 상태였기 때문에 꼭 지금 옮길 이유는 없었습니다. 다만 Spring Boot 3부터 Java 17이 최소 버전이 되었고, 이전부터 서버리스의 [GAE](https://ja.wikipedia.org/wiki/Google_App_Engine)에서 돌고 있던 프로젝트도 있어서, 시작 시간을 줄이기 위해 [Spring Native](https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html)를 시험해 보고 싶었습니다. 게다가 Spring Native는 Spring Boot 3부터 공식 지원 대상이 되었습니다.

또 하나는 Spring Boot 3에서 의존성이나 프로퍼티 표현 방식이 바뀌더라도, 제가 다루는 애플리케이션이 마이크로서비스라 영향이 비교적 작을 것이라고 봤기 때문입니다. 모놀리식이거나 Spring 기능에 크게 의존하는 앱이라면 마이그레이션이 더 어려웠을 겁니다. 그래서 실제로 옮길 계획이 있다면 릴리스 노트를 꼭 확인하는 편이 좋습니다.

Spring Boot 2에서 3으로의 이동은 기본적으로 [공식 마이그레이션 가이드](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)를 참고하면 됩니다. 다만 가이드만으로는 알기 어려운 부분이 있고, Spring Framework 변화 때문에 다른 라이브러리까지 영향을 받을 수 있습니다. 그래서 실제로 제가 어떻게 대응했는지 몇 가지 소개해 보겠습니다.

### `@ConstructorBinding`

`@ConstructorBinding`은 `@ConfigurationProperties`가 붙은 클래스에 애플리케이션 프로퍼티 파일의 값을 생성자 주입으로 넣어 주는 어노테이션입니다. 현재 프로젝트에서는 AWS S3 버킷 이름, 메일 템플릿, 다른 API 호출용 엔드포인트 등을 `application.yaml`에 적어 읽어 오고 있습니다.

Spring Boot 3에서는 `@ConstructorBinding`이 없어도 프로퍼티를 읽어 오도록 바뀌었고, 이 어노테이션 자체도 `Deprecated`가 되었습니다. 그래서 처음에는 컴파일 에러가 나지만, `@ConstructorBinding`을 지우는 것만으로 정상 동작했습니다. 가장 간단한 대응이었습니다.

### Tracing 변경

현재 애플리케이션에서는 다른 마이크로서비스 API를 호출할 때 [Brave](https://github.com/openzipkin/brave)를 써서 헤더에 `Trace Context`를 싣고 있습니다. 같은 Trace ID를 따라갈 수 있어서 로그를 보기 편합니다. 예전에는 [Spring Cloud Sleuth](https://spring.io/projects/spring-cloud-sleuth)를 통해 Brave를 DI했는데, Spring Boot 3에서는 이 구조가 바뀌고 [Micrometer](https://micrometer.io/)의 새 API를 쓰게 됩니다.

즉, 기존 Spring Cloud Sleuth 기반 DI는 더 이상 쓸 수 없고 의존성도 바뀝니다. 우선 DI를 맡던 [BraveAutoConfiguration](https://docs.spring.io/spring-boot/docs/3.0.1/api/org/springframework/boot/actuate/autoconfigure/tracing/BraveAutoConfiguration.html)이 [Spring Boot Actuator](https://github.com/spring-projects/spring-boot/tree/v3.0.2/spring-boot-project/spring-boot-actuator)로 옮겨졌습니다. 그래서 이제 Spring Cloud Sleuth는 필요 없게 됩니다.

다만 Spring Boot Actuator만으로는 `BraveAutoConfiguration`이 제대로 주입되지 않습니다. 그래서 아래처럼 브리지 의존성을 추가해야 합니다. 버전은 Spring Boot 쪽에 맞춰집니다.

```kotlin
implementation("io.micrometer:micrometer-tracing-bridge-brave")
```

의존성이 바뀌므로 `application.yaml` 설정도 바꿔야 합니다. 예전에는 이렇게 썼습니다.

```yml
spring:
  sleuth:
    enabled: true
    propagation:
      type:
        - w3c
```

Micrometer로 옮기면 아래처럼 바꿉니다. 사실 이 설정은 기본값이라 없어도 되지만, 나중에 설정을 봤을 때 현재 상태를 바로 알 수 있게 적어 두고 있습니다.

```yml
management:
  tracing:
    enabled: true
    propagation:
      type: w3c
```

가장 막혔던 부분은 유닛 테스트였습니다. 테스트를 돌리면 다음과 같은 에러가 났습니다.

```shell
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'brave.Tracing' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {}
```

Spring Cloud Sleuth를 쓸 때는 테스트에서도 DI가 잘 됐는데, 이상하게 `BraveAutoConfiguration` 기반 DI가 테스트에서는 안 잡혔습니다. 로컬에서 직접 실행하면 같은 문제가 나지 않아서, 테스트 시점에만 AutoConfiguration이 적용되지 않는 것처럼 보였습니다.

`@TestConfiguration` 등으로 직접 Bean을 등록할 수도 있겠지만, 찾아보니 [Micrometer에 테스트용 라이브러리](https://www.baeldung.com/spring-boot-3-observability#testing-observations)가 따로 있었습니다. 이 라이브러리가 나온 지 얼마 안 돼서 정보를 찾기 어려웠지만, 어쨌든 아래 의존성을 추가하면 문제가 해결됩니다.

```kotlin
testImplementation("io.micrometer:micrometer-tracing-test")
```

이번에는 라이브러리 추가만으로 해결됐지만, 이런 정보는 직접 찾아보지 않으면 잘 안 보인다는 점이 문제였습니다. 다른 라이브러리도 비슷할 수 있으니, AutoConfiguration으로 주입돼야 할 Bean이 없다는 에러가 나면 테스트용 라이브러리가 빠졌는지 확인해 보는 게 좋습니다.

추가로 [Datadog](https://www.datadoghq.com/ja/)도 Micrometer 설정 방식이 바뀌면서 일부 프로퍼티의 적는 방법이 달라졌습니다. 예전에는 아래처럼 썼습니다.

```yml
management:
  metrics:
    export:
      datadog:
        enabled: true
        apiKey: ${API_KEY}
        step: 1m
```

이제는 이렇게 바뀝니다.

```yml
management:
  datadog:
    metrics:
      export:
        enabled: true
        api-key: ${API_KEY}
        step: 1m
```

Micrometer 관련 기능을 쓰고 있다면, 기존 Spring Boot 2와 비교해 프로퍼티 설정 방식이 달라졌는지 확인해 보는 편이 좋습니다.

### Jakarta EE

Spring Framework 6은 Oracle의 `Java EE`에서 Eclipse의 `Jakarta EE`로 옮겨갔기 때문에 컴파일 에러가 날 수 있습니다. 주로 `HttpServletRequest` 같은 servlet 계열이나 `@Valid` 같은 validation 계열 패키지에서 영향을 받습니다. 이 경우 import 패키지명을 바꾸면 됩니다.

예를 들어 아래 패키지를 쓰고 있다면:

- `javax.servlet`
- `javax.validation`

다음처럼 바꿉니다.

- `jakarta.servlet`
- `jakarta.validation`

패키지 이름만 바뀐 것이므로 대응 자체는 어렵지 않습니다.

### springdoc

[springdoc-openapi](https://springdoc.org/)로 API 문서를 생성하고 있다면, 기존 버전은 Spring Boot 3과 호환되지 않는 것 같습니다. CI에서는 `/v3/api-docs`에서 가져온 JSON을 파싱해 S3에 문서를 올리고 있는데, Spring Boot 3에서는 404가 발생했습니다.

다행히 의존성을 바꾸면 해결됩니다. 예전에는 아래처럼 넣었습니다.

```kotlin
implementation("org.springdoc:springdoc-openapi-ui:1.6.14")
implementation("org.springdoc:springdoc-openapi-kotlin:1.6.14")
```

이걸 [springdoc-openapi v2](https://springdoc.org/v2/)로 바꾸면 됩니다. Kotlin 지원도 포함되니 아래 하나면 충분합니다.

```kotlin
implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.0.2")
```

이제 404 없이 S3에 문서를 업로드할 수 있습니다.

### Liquibase

Spring Framework와 직접 관련된 것은 아니지만, Spring Boot 버전 업에 영향을 받았기 때문에 같이 소개합니다. DB 마이그레이션을 위해 Gradle 플러그인으로 [Liquibase](https://www.liquibase.org/)를 쓰는 경우 버전에 따라 아래와 같은 에러가 날 수 있습니다.

```text
SLF4J: No SLF4J providers were found.
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See https://www.slf4j.org/codes.html#noProviders for further details.
SLF4J: Class path contains SLF4J bindings targeting slf4j-api versions 1.7.x or earlier.
SLF4J: Ignoring binding found at [jar:file:/root/.gradle/caches/modules-2/files-2.1/ch.qos.logback/logback-classic/1.2.11/4741689214e9d1e8408b206506cbe76d1c6a7d60/logback-classic-1.2.11.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See https://www.slf4j.org/codes.html#ignoredBindings for an explanation.
Exception in thread "main" java.lang.ClassCastException: class org.slf4j.helpers.NOPLogger cannot be cast to class ch.qos.logback.classic.Logger (org.slf4j.helpers.NOPLogger and ch.qos.logback.classic.Logger are in unnamed module of loader 'app')
  at liquibase.integration.commandline.Main.setupLogging(Main.java:233)
  at liquibase.integration.commandline.Main.run(Main.java:145)
  at liquibase.integration.commandline.Main.main(Main.java:129)
```

아마 Spring Boot 3에서 [SLF4J](https://www.slf4j.org/) 버전이 2 계열로 올라갔기 때문입니다. Liquibase 플러그인은 예전에는 [Logback](https://logback.qos.ch/index.html)으로 마이그레이션 상황을 출력했는데, 해당 Logback 버전이 1.2라서 SLF4J 2와 호환되지 않았습니다. Logback 1.3부터 호환되므로, 그 이전 버전을 쓰고 있다면 같이 올려 두는 편이 좋습니다.

## 마지막으로

결과적으로 Java와 Spring Boot 버전 업은 성공했습니다. 다만 위에서 본 것처럼, 이건 제가 다루는 애플리케이션이 마이크로서비스였고 의존성이 비교적 적었기 때문에 가능했던 일이라고 생각합니다. 모놀리식 애플리케이션이거나 의존성이 더 복잡했다면 여기 적은 것 말고도 추가 문제가 있었을 가능성이 큽니다.

그래도 Spring Boot 2의 지원이 끝나면 결국 3계열로 옮겨야 하는 시점이 오게 됩니다. Java도 11에서 17로 올리는 정도라면 큰 문제는 없겠지만, 8에서 바로 옮기는 경우에는 Java 9에서 도입된 [Module](https://www.oracle.com/webfolder/technetwork/jp/javamagazine/Java-JA18-LibrariesToModules.pdf) 때문에 더 신중한 검토가 필요할 수 있습니다.

아직 Native화까지는 시도하지 않았지만, JVM 위에서도 애플리케이션 시작 시간이 줄어든 것은 확인했습니다. 로컬에서 실행했을 때 Java 11 + Spring Boot 2는 31초, Java 17 + Spring Boot 3는 23초가 걸렸습니다. 정확한 데이터는 더 모아야 하지만, 전체적인 성능 향상도 어느 정도 기대해 볼 만하다고 생각합니다.
