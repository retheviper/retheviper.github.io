---
title: "Gradle로 멀티 프로젝트 만들기"
date: 2019-10-20
categories:
  - gradle
image: "../../images/gradle.webp"
tags:
  - groovy
  - java
  - gradle
translationKey: "posts/gradle-multi-project"
---

처음에는 Maven으로 Spring MVC 프로젝트를 만드는 방법을 주로 배우고 있었습니다. 그런데 최근에는 Gradle로 관리되는 Spring Boot 프로젝트를 다루게 되면서 생각이 조금 바뀌었습니다. Gradle을 단순히 Maven 비슷한 의존성 관리 도구 정도로만 봤는데, 직접 task를 만들고 Jar를 빌드하면서 보니 훨씬 유연한 도구였습니다.

특히 마음에 들었던 부분은 서로 다른 역할을 가진 프로젝트를 하나의 멀티 프로젝트로 묶어 관리할 수 있다는 점이었습니다. 클래스나 패키지 단위로만 나누는 것보다 프로젝트 단위로 책임을 분리하면, 더 유지보수하기 쉬운 구조를 만들 수 있겠다고 느꼈습니다.

여기서는 Gradle 멀티 프로젝트가 무엇인지, 그리고 기본적인 구성 방법을 간단히 정리해 보겠습니다.

## Gradle Multi Project

Gradle의 멀티 프로젝트는 이름 그대로 여러 프로젝트를 하나로 묶은 구조입니다. 보통 루트 프로젝트가 기준이 되고, 그 아래에 여러 하위 프로젝트가 붙습니다. 이 하위 프로젝트를 서브 프로젝트라고 부릅니다.

물론 멀티 프로젝트를 만드는 방식은 여러 가지가 있을 수 있습니다. 폴더 구조도 조금씩 다를 수 있고, IDE 설정 방식도 차이가 납니다. 다만 목적은 결국 여러 프로젝트를 한 리포지토리 안에서 편하게 관리하는 데 있다고 생각합니다. 그래서 여기서는 루트 프로젝트 아래에 서브 프로젝트를 두는 형태를 기준으로 설명하겠습니다.

이 글은 Eclipse 기준으로 정리한 내용입니다.

## 루트 프로젝트

먼저 전체를 관리할 루트 프로젝트를 만듭니다. 일반적인 Gradle 프로젝트를 생성한 뒤, 서브 프로젝트로 옮겨갈 `build.gradle`은 따로 보관해 두고 나머지 파일은 정리합니다. 루트 프로젝트에 남길 파일은 보통 다음 정도입니다.

- `gradle`
- `build.gradle`
- `gradlew`
- `gradlew.bat`

루트 프로젝트의 `build.gradle`은 가능한 한 단순하게 두는 편이 좋습니다. 루트 프로젝트는 주로 서브 프로젝트를 묶어 주는 역할만 하고, 실제 플러그인이나 의존성 설정은 각 서브 프로젝트에서 처리하는 편이 더 깔끔합니다.

```groovy
subprojects {
    repositories {
        mavenCentral()
    }
}
```

이 예시에서는 서브 프로젝트들이 공통으로 사용할 저장소만 지정했습니다. 필요하다면 공통 플러그인이나 공통 설정도 함께 넣을 수 있습니다.

다음으로 `settings.gradle`을 작성합니다. 이 파일은 어떤 프로젝트를 포함할지, 그리고 각 프로젝트의 이름을 어떻게 붙일지 정합니다.

```groovy
include 'core'
include 'web'

rootProject.name = 'TestProject'
rootProject.children.each { it.name = "${rootProject.name}-${it.name}" }
```

`include`로 서브 프로젝트를 등록하고, `rootProject.name`으로 루트 프로젝트 이름을 정합니다. 아래 줄은 각 서브 프로젝트 이름 앞에 루트 프로젝트 이름을 붙여서 `TestProject-core`, `TestProject-web`처럼 보이게 만듭니다.

Eclipse에서 보면 프로젝트가 다음처럼 표시됩니다.

- `TestProject`
- `TestProject-core`
- `TestProject-web`

서브 프로젝트를 추가할 때도 `include`에 이름만 더해 주면 같은 규칙으로 이름이 붙기 때문에 관리가 수월합니다.

## Sub Project

루트 프로젝트를 만든 뒤에는 실제 기능을 담는 서브 프로젝트를 만듭니다. 별도의 Gradle 프로젝트로 만들어도 되고, 빈 폴더를 만든 뒤 루트 프로젝트의 설정을 가져와도 됩니다. 여기서는 빈 폴더를 만들어 구성하는 방식을 기준으로 설명하겠습니다.

먼저 루트 프로젝트 아래에 서브 프로젝트용 폴더를 만듭니다. 폴더 이름은 `settings.gradle`의 `include` 이름과 같아야 합니다. 위 예시라면 `core`와 `web` 폴더를 만들면 됩니다.

그 다음 각 폴더 안에 `build.gradle`을 두고, Eclipse용 플러그인을 설정합니다.

```groovy
plugins {
    id 'eclipse'
}
```

설정한 뒤 루트 프로젝트에서 `gradlew tasks`를 실행하면 IDE 관련 작업에 `eclipse`가 보입니다. 그 작업을 실행하거나, 커맨드라인에서 `gradlew eclipse`를 실행하면 Eclipse 프로젝트로 인식될 준비가 됩니다.

그 다음에는 일반적인 Java 프로젝트처럼 `src/main/java`를 만들고 패키지와 클래스를 채워 넣으면 됩니다.

만약 Eclipse가 멀티 프로젝트를 제대로 인식하지 못한다면 프로젝트를 새로 고치거나, Gradle 메뉴에서 프로젝트를 리프레시하면 됩니다.

## Sub Project Dependency

이 방식으로 구성한 멀티 프로젝트는 서브 프로젝트끼리 의존할 수도 있습니다. 예를 들어 `web` 프로젝트에서 `core` 프로젝트의 클래스를 쓰고 싶다면 `web`의 `build.gradle`에 다음처럼 적으면 됩니다.

```groovy
dependencies {
    implementation project(':TestProject-core')
}
```

이렇게 하면 `web`을 빌드할 때 `core`도 함께 컴파일됩니다. 다만 `core` 변경사항이 즉시 반영되지 않는 상황도 있으니, 테스트할 때는 의존 관계를 의식해 두는 것이 좋습니다.

## 마지막으로

Eclipse에서 멀티 프로젝트를 구성하는 과정은 조금 번거롭습니다. 그래도 구조를 한 번 잡아 두면 프로젝트가 커질수록 관리가 훨씬 편해집니다.

멀티 프로젝트가 어떤 구조인지 이해해 두면 나중에 다른 프로젝트에도 그대로 응용할 수 있습니다. 규모가 커질수록 이런 분리가 더 중요해지니, 한 번쯤 직접 구성해 보는 것도 좋은 경험이 됩니다.
