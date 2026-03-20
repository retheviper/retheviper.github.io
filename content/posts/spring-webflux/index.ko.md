---
title: "Spring WebFlux 이해하기"
date: 2020-09-06
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - webflux
  - nonblocking
translationKey: "posts/spring-webflux"
---

Spring이 처음 공개된 지 벌써 20년 가까운 시간이 흘렀습니다. 지금은 Java라고 하면 자연스럽게 Spring을 떠올릴 정도로 익숙한 조합이 됐고, Spring MVC나 Maven, `properties`처럼 예전에는 번거로웠던 환경 구성도 Spring Boot 덕분에 많이 편해졌습니다. 저 역시 Spring Boot, Gradle, YAML을 조합해 XML 없는 애플리케이션을 자주 만들고 있는데, 확실히 예전보다 훨씬 편합니다.

이렇게 계속 발전해 온 Spring에도 몇 년 전부터 Spring MVC의 한계를 개선하고 싶다는 흐름이 있었습니다. 그리고 Spring 5.0부터 MVC와는 전혀 다른 새로운 프레임워크가 등장했는데, 그게 바로 이번에 소개할 Spring WebFlux입니다.

## Spring WebFlux는 MVC와 무엇이 다른가

처음부터 다시 만든 프레임워크인 만큼 근본부터 다른 부분이 많습니다. 이론적으로 보면 핵심 키워드는 다음과 같습니다.

- 비동기
- 논블로킹
- 반응형

제가 느낀 실제 코드 레벨의 차이는 아래 정도입니다.

- `Mono`와 `Flux`
- `Controller`/`Service` 대신 `Function`
- Tomcat 대신 Netty
- JPA/JDBC 대신 R2DBC
- 새로운 추상화 클래스

각 항목을 하나씩 살펴보겠습니다.

## 이론적인 이야기

### 비동기, 논블로킹, 반응형

Spring WebFlux는 [Project Reactor](https://github.com/reactor/reactor-core)를 기반으로 만들어졌고, 그 근간은 [Reactive Streams](https://www.reactive-streams.org)입니다. Reactive Streams는 표준 API로, Java 9에서 `java.util.concurrent.Flow`로 들어왔습니다.

Reactive Streams는 디자인 패턴 관점에서는 Observer와 비슷합니다. 쉽게 말하면 어떤 이벤트가 발생했을 때 "알려 달라"고 등록해 두고, 그에 맞춰 데이터를 받는 방식입니다. 이 "알려 달라"는 요청을 `Subscription`이라고 하고, 데이터를 내보내는 쪽을 `Publisher`, 받는 쪽을 `Subscriber`라고 부릅니다. 이런 이벤트 기반 프로그램을 만드는 것이 반응형 프로그래밍입니다.

Reactive Streams에서는 이 `Subscription`의 주고받음이 비동기와 논블로킹으로 이루어집니다. 즉, 코드가 실행된 뒤 완료될 때까지 멈춰 기다릴 필요가 없고, 그 사이에 다른 작업을 할 수 있습니다. 같은 수의 같은 작업을 실행할 때는 동기·블로킹과 성능 차이가 크지 않지만[^1], 스레드 수가 병목이 되는 상황이라면 비동기·논블로킹 쪽이 유리합니다.

이론은 끝없이 깊어질 수 있으니, 이제 실제 코드에서는 무엇이 달라지는지 보겠습니다.

## 코드 레벨의 이야기

### `Mono`와 `Flux`

Spring WebFlux에서는 컨트롤러 메서드의 반환값으로 `Mono`와 `Flux`를 사용합니다. Spring MVC라면 문자열로 JSP를 지정하거나 REST API에서 JSON 객체를 반환했겠지만, WebFlux에서는 반환 방식이 조금 다릅니다.

앞서 말했듯 WebFlux의 기반이 Reactive Streams이고, WebFlux에서는 이를 `Mono`와 `Flux`로 구현합니다. `Mono`는 `0 또는 1`, `Flux`는 `0 또는 N` 정도로 이해하면 됩니다. 꼭 Collection이면 `Flux`, 아니면 `Mono`로 고정할 필요는 없고, 상황에 따라 `Mono`로도 충분합니다.

이름만 보면 Java 8 Stream API와도 연결이 있어 보입니다. 실제로 데이터 생성과 소비 시점은 다르지만[^2], 메서드 이름이나 람다로 조합해 가는 방식은 비슷한 느낌이 있습니다. 이미 [RxJava](https://github.com/ReactiveX/RxJava)나 [JOOL](https://github.com/jOOQ/jOOL)에 익숙하다면 적응이 어렵지 않겠지만, 그렇지 않다면 처음에는 낯설 수 있습니다.

예를 들어 GET 요청을 받으면 1초 후 응답을 돌려주는 간단한 메서드를 작성해 보겠습니다. Spring MVC의 REST API는 보통 이렇게 씁니다.

```java
@GetMapping
@ResponseStatus(HttpStatus.OK)
public String getTest() {
    try {
        Thread.sleep(1000);
        return "task completed by mvc";
    } catch (InterruptedException e) {
        return "task failed by mvc";
    }
}
```

WebFlux에서는 `Mono`를 만들어 반환합니다.

```java
@GetMapping
@ResponseStatus(HttpStatus.OK)
public Mono<String> getTest() {
    return Mono.delay(Duration.ofMillis(1000)).then(Mono.just("task completed by Mono"));
}
```

요즘은 Flutter나 SwiftUI처럼 선언적인 방식이 많이 늘어나고 있어서 이런 스타일이 아주 낯설지는 않습니다. 하지만 전통적인 명령형 프로그래밍에 익숙한 사람에게는 꽤 다르게 느껴질 수 있습니다. 저 역시 `Stream`과 `Lambda`는 좋아하지만, 명령형을 중첩하는 방식과 메서드 체인으로 길어지는 선언형 중 어느 쪽이 더 좋은지 아직은 확신이 없습니다.

### `Controller`/`Service` 대신 `Function`

WebFlux의 또 다른 특징은 `Controller`와 `Service`를 대체하는 구조를 쓸 수 있다는 점입니다. 물론 기존 방식 그대로 `Controller`와 `Service`를 써도 됩니다. 하지만 새 프레임워크라면 새로운 방식도 한 번 써 보고 싶어집니다.

Spring에서 `Controller`/`Service`를 만든다는 것은 결국 어노테이션 기반의 메타프로그래밍에 의존한다는 뜻이기도 합니다. 어노테이션은 편리하고 Spring에서는 거의 모든 걸 어노테이션으로 처리하는 느낌도 있지만, 다음 같은 문제도 있습니다.

- 컴파일러가 검증하기 어렵다
- 코드의 동작을 직접 드러내지 않는다
- 상속과 확장 규칙이 일관되지 않다
- 읽기 어렵고 오해하기 쉬운 코드를 만들 수 있다
- 테스트가 어렵다
- 커스터마이징이 어렵다

이유는 결국 어노테이션이 Reflection에 의존하기 때문입니다. Reflection을 쓰면 런타임에 바이트코드를 다루게 되므로, 컴파일 타임에 확인할 수 있는 것이 많지 않습니다. Reflection은 강력하지만 성능 저하와 디버깅 난이도라는 문제도 있습니다. 그래서 Java 코드를 네이티브로 컴파일하려는 [GraalVM](https://www.graalvm.org)에서는 Reflection 대응이 중요한 이슈가 됩니다.

이런 문제를 줄이기 위해 WebFlux에서 도입된 것이 `Function`입니다. 말 그대로 함수형 모델입니다. 기존 `Controller`에 대응하는 `Router`와 `Service`에 대응하는 `Handler`를 작성해, 함수형 스타일로 코드를 구성할 수 있습니다. 물론 함수형이라고 해서 어노테이션이 완전히 사라지는 것은 아닙니다. 예를 들어 아래처럼 쓸 수 있습니다.

```java
@Configuration
public class Router {

    @Bean
    public RouterFunction<ServerResponse> route(final Handler handler) {
        return nest(
                path("/api/v1/web/members"),
                RouterFunctions.route(GET("/"), handler::listMember)
                        .andRoute(POST("/"), handler::createMember)
        );
    }
}

@Component
public class Handler {

    private final MemberRepository repository;

    @Autowired
    public Handler(final MemberRepository repository) {
        this.repository = repository;
    }

    public Mono<ServerResponse> listMember(final ServerRequest request) {
        return ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(Flux.fromIterable(repository.findAll()), Member.class);
    }

    public Mono<ServerResponse> createMember(final ServerRequest request) {
        return ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(request.bodyToMono(Member.class)
                        .flatMap(member -> Mono.fromCallable(() -> repository.save(member))
                                .then(Mono.just(member))), Member.class);
    }
}
```

함수형이 되어 읽기 쉬워졌는지, 아니면 오히려 더 복잡해졌는지는 사람마다 다를 것 같습니다.

### Tomcat 대신 Netty

Spring WebFlux의 기본 서버는 Netty입니다. 이유는 단순합니다. [Netty](https://netty.io)는 처음부터 논블로킹을 전제로 설계되었기 때문입니다. 반면 Tomcat은 동기·블로킹 방식이므로, 둘은 구조가 꽤 다릅니다.

- Tomcat: 요청과 스레드가 1:1
- Netty: 요청과 스레드가 N:1

물론 Spring MVC처럼 Netty 대신 Tomcat을 쓸 수도 있습니다. Gradle에서는 대략 이렇게 설정합니다.

```groovy
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-webflux:2.3.3.RELEASE') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-reactor-netty'
    }
    implementation('org.springframework.boot:spring-boot-starter-tomcat:2.3.3.RELEASE')
}
```

### JPA/JDBC 대신 R2DBC

이것도 애플리케이션 서버와 비슷한 이야기입니다. JPA/JDBC 같은 기존 ORM은 블로킹이므로, 논블로킹에 맞는 [R2DBC](https://r2dbc.io)를 써 보자는 것입니다. NIO가 그랬듯이, 블로킹 환경에서도 R2DBC를 쓰면 경우에 따라 성능 개선을 기대할 수 있습니다.

### 새로운 추상화 클래스

WebFlux에서는 Spring MVC에서 쓰던 추상화 클래스도 바뀌었습니다. 이것 역시 함수형 스타일과 논블로킹에 맞추기 위한 변화입니다.

| 종류 | Spring MVC | Spring WebFlux |
|---|---|---|
| 요청 | HttpServletRequest | ServerHttpRequest |
| 응답 | HttpServletResponse | ServerHttpResponse |
| 다른 API 호출 | RestTemplate | WebClient |

`WebClient`가 나오면서 `RestTemplate`은 `deprecated`가 되었기 때문에, Spring MVC를 쓰는 경우에도 `WebClient` 도입을 고려할 필요가 있습니다. 실제로 Spring MVC에서 쓰더라도 `RestTemplate`보다 성능이 좋아지는 경우도 있다고 합니다.

## 어디에 써야 하는가

여기까지 WebFlux의 특징을 대략 살펴봤습니다. 어떠신가요. 작성 방식도 꽤 다르고, Servlet 기반 MVC와는 전혀 다른 Reactor 기반이기 때문에 WebFlux 도입은 쉬운 선택은 아닙니다. 기존 Spring MVC와 동시에 쓰기도 어렵고, Spring Security 같은 다른 라이브러리도 WebFlux용으로 다시 맞춰야 하는 경우가 많습니다. 이미 기존 시스템과 Spring MVC 기반 라이브러리가 많다면 영향 범위도 꽤 큽니다.

성능 측면에서도 논블로킹이 강한 상황은 "지정된 스레드 수를 넘어서는 요청이 들어올 때"입니다. 논블로킹으로 바꾼다고 해서 단일 스레드 성능이 자동으로 올라가는 것은 아닙니다[^3].

다만 다음과 같은 경우에는 WebFlux 도입을 검토해 볼 수 있습니다.

1. 완전히 새로운 서비스를 처음부터 만드는 경우
1. 여러 서비스가 있고 서비스 간 호출이 많은 경우, 즉 마이크로서비스 환경
1. BFF[^4] 구조를 쓰는 경우

## 마지막으로

간단히 정리하려던 내용이 예상보다 길어졌지만, 그만큼 WebFlux가 기존 Spring MVC와는 결이 꽤 다른 프레임워크라는 점은 분명해졌습니다. `RestTemplate`이 `deprecated`가 된 흐름까지 보면, 적어도 반응형과 논블로킹 모델을 이해해 둘 필요는 점점 커지고 있다고 생각합니다.

다만 그렇다고 해서 모든 시스템이 곧바로 WebFlux로 옮겨가야 하는 것은 아닙니다. 어떤 패러다임이 절대적으로 더 낫다기보다, 현재 시스템 구조와 병목 지점에 맞게 선택하는 것이 더 중요합니다. WebFlux도 그런 선택지 가운데 하나로 이해하는 편이 자연스럽습니다.

[^1]: 실제로는 스레드 수가 병목이 아닌 상황에서는 함수형이 조금 느린 편이라고 합니다. 함수형 API 구현이 더 복잡하기 때문입니다. 다만 읽기 쉬움과 구현 편의성을 생각하면 충분히 트레이드오프로 볼 만한 차이입니다.
[^2]: Stream은 동기라서 데이터 생산과 소비가 함께 일어납니다. 하지만 Reactive Stream은 데이터 생산이 곧바로 소비로 이어진다고 보기는 어렵습니다. 비동기이기 때문입니다.
[^3]: 오히려 단일 스레드 처리에서는 WebFlux가 MVC보다 조금 느리다는 이야기도 있습니다. for 루프보다 Stream이 느린 이유와 비슷하다고 볼 수 있습니다.
[^4]: Back-end For Front-end의 약자입니다. 마이크로서비스의 한 형태로, 여러 엔드포인트를 묶어 프런트엔드용 응답을 따로 만들어 주는 백엔드입니다. 프런트엔드는 하나의 엔드포인트만 호출하면 됩니다.
