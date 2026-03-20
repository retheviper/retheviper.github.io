---
title: "Java 모듈 충돌 기록"
date: 2019-09-08
categories:
  - java
image: "../../images/java.webp"
tags:
  - jigsaw
  - module
  - java
translationKey: "posts/java-conflict-of-module"
---

최근 Java는 버전 업이 꽤 빠릅니다. 제가 처음 배운 것은 1.8이었는데 금방 9가 나왔고, 지금은 13도 릴리스를 앞두고 있었습니다. 버전 업에는 버그 수정과 성능 개선 같은 장점이 많아서 가능한 한 최신 버전을 쓰고 싶지만, 언어 버전이 올라갈 때마다 바뀐 점을 확인하고 기존 코드를 다시 점검하는 일은 쉽지 않습니다.

Java는 역사가 길기 때문에 지금의 트렌드와 비교하면 불편한 점도 많습니다. 1.8이 오래 유지된 탓에 유행을 따라가는 속도도 느렸습니다. 10에서 타입 추론이 도입되는 등 변화가 있긴 했지만, Kotlin처럼 처음부터 다른 철학으로 설계된 언어와 비교하면 여전히 거리가 있습니다.

물론 변화 자체는 나쁜 것이 아닙니다. 기존의 특징은 유지하면서도 새로운 문법을 받아들일 수 있게 되면 사용자층이 넓어집니다. 다만 모든 요소에서 옛것과 새것을 함께 끌고 갈 수는 없습니다. 이럴 때는 무엇을 선택할지 정해야 합니다.

이번 글에서 이야기할 모듈이 바로 그런 사례입니다. 오래된 문제를 해결하기 위해 도입됐지만, 결국 기존 코드에 영향을 주어서 대응이 필요한 기능이기도 합니다. 처음에는 제 코드에서는 신경 쓰지 않아도 될 줄 알았는데, 그렇지 않았습니다. 그래서 Java의 모듈이 무엇인지, 그리고 어떤 문제를 겪었는지 정리해 보려 합니다.

## Project Jigsaw

모듈은 Project Jigsaw라는 이름으로 1.7부터 도입이 검토된 기능입니다. 이름 그대로 애플리케이션을 실행할 때 읽을 라이브러리(Java 내장 포함)를 선택할 수 있는 시스템입니다. 1.8까지는 커맨드라인 애플리케이션을 만들어도 Swing 같은 기본 시스템 라이브러리가 포함됐지만, 이제는 그 구성을 조정할 수 있게 됐습니다. 쓰지 않는 라이브러리를 빼면 애플리케이션 크기가 줄고 메모리도 아낄 수 있습니다. JVM이 완전히 올라오기까지 시간이 걸린다는 문제도 어느 정도 줄일 수 있습니다.

패키지가 너무 많이 공개되는 문제도 모듈로 해결할 수 있습니다. Java 클래스는 `protected`만으로는 같은 패키지에서만 접근이 가능하고, 패키지가 잘게 나뉘면 같은 라이브러리 안에서도 접근이 어려워집니다. 그래서 결국 `public`으로 열어 버리는 경우가 많았는데, 그러면 라이브러리 내부용 클래스까지 외부에 노출될 수 있습니다. 모듈을 쓰면 외부에 공개할 것과 내부에서만 쓸 것을 나눌 수 있습니다.

## 모듈 예시

모듈이 `public` 문제를 어떻게 풀어 주는지 코드로 보겠습니다. 아직 제가 모듈을 적극적으로 쓰는 편은 아니라 기본적인 부분만 적겠습니다. 핵심은 아래 세 가지입니다.

- 이름
- exports
- requires

`name`은 모듈 자체의 이름입니다. 패키지명과 비슷한 규칙으로 정합니다. `exports`는 외부에 공개할 패키지를 뜻합니다. 모듈 내부에서 `public`이어도 `exports`되지 않으면 외부에서 접근할 수 없습니다. `requires`는 다른 모듈에 대한 의존성을 뜻합니다.

아래처럼 `module-info.java`에 선언합니다.

```java
// module-info.java 작성 예시
module com.module.mylibrary {
    exports com.module.mylibrary.api;
    requires com.module.exlibrary;
}
```

`exports`는 공개 대상을 더 좁힐 수도 있습니다.

```java
// exlibrary에만 공개
module com.module.mylibrary {
    exports com.module.mylibrary.api
    to com.module.exlibrary;
}
```

외부 라이브러리에도 모듈을 적용할 수 있습니다. `module-info.java`가 있는 경우도 있지만, Java 9 이전 라이브러리는 대부분 없을 것입니다. 이럴 때는 `Automatic Module`이나 `Unnamed Module`로 다루게 됩니다. 둘 다 자동으로 모듈처럼 취급된다는 점은 같지만, `Automatic Module`은 `modulepath`에 올라가 이름을 가지는 반면, `Unnamed Module`은 `classpath`에 속해서 이름이 없고 `requires`로 직접 지정할 수도 없습니다.

## 문제 발생

제가 겪은 문제는 같은 패키지를 가진 두 라이브러리의 충돌이었습니다. 기존 프로젝트에 Gradle 작업을 추가하려고 하던 중이었는데, 공식 문서를 따라 `java-gradle-plugin`을 붙이는 방식으로 작업했습니다. 그러면 Java 라이브러리가 자동으로 추가되어 플러그인을 Java로 작성할 수 있지만, 여기 포함된 라이브러리가 시스템 라이브러리와 충돌했습니다.

원래 프로젝트에서는 `javax.xml`을 쓰고 있었고, 이 패키지는 Java 9부터 deprecated 되었으며 Java 11에서 제거된 것으로 알고 있습니다. 그런데 Eclipse에서는 이것이 `Unnamed Module`로 읽혀 있었고, `java-gradle-plugin` 쪽에도 같은 이름의 패키지가 있어서 충돌이 났습니다. 에러 메시지는 `The package javax.xml.transform is accessible from more than one module: <unnamed>, javax.xml` 형태였습니다.

[비슷한 사례](https://stackoverflow.com/questions/51094274/eclipse-cant-find-xml-related-classes-after-switching-build-path-to-jdk-10)를 보면 해결책이 두 가지 제시되지만, 제 경우에는 둘 다 쓸 수 없었습니다. `module-info.java`를 만들면 멀티 프로젝트 구조까지 고려해야 해서 복잡했고, Eclipse 설정에서 `javax.xml`을 빼 버리면 다른 코드가 의존하는 `java.sql`까지 영향을 받았습니다.

링크를 따라가 보면 Java 13 시점까지도 이 문제는 완전히 풀리지 않았습니다. `java-gradle-plugin`은 Gradle이 관리하는 라이브러리라 제가 손댈 수 있는 부분도 아니었습니다.

## 결론

지금 시점에서는 외부 라이브러리를 유지하면서 충돌만 피하는 뾰족한 방법은 없어 보입니다. 제 모듈 이해가 부족한 탓도 있겠지만, 결국 이런 문제가 나면 충돌 원인이 되는 라이브러리를 빼는 수밖에 없는 것 같습니다. 편의를 위해 들어온 기능이 예상 못 한 곳에서 문제를 일으키는 일은 드물지 않지만, 막상 겪으면 꽤 번거롭습니다.

물론 조건이 허락한다면 Java를 1.8로 낮추는 방법도 있습니다. 다만 1.8도 결국 지원이 끝날 것이고, Java 버전은 앞으로도 계속 올라갈 테니 언젠가는 다시 마주칠 문제일 수 있습니다. 결국 모듈 시스템을 완전히 외면하기보다는, 이런 충돌이 왜 생기는지 정도는 이해해 두는 편이 낫다고 느꼈습니다.
