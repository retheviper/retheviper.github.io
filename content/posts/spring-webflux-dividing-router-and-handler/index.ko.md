---
title: "WebFlux Functional Endpoint 구조 고민하기"
date: 2021-08-30
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - webflux
  - kotlin
  - java
  - rest api
translationKey: "posts/spring-webflux-dividing-router-and-handler"
---

이전에는 [WebFlux에서 Functional Endpoint 선택하기](../spring-webflux-router/)라는 글을 쓴 적이 있습니다. 이번에는 `Controller`/`Service`와 `Router`/`Handler`를 비교하려는 것이 아니라, `Functional Endpoint`를 쓸 때 실제로 어떤 구조로 구현하는 게 좋은지에 대해 생각한 내용을 정리해 보려 합니다.

실무에서 WebFlux를 직접 많이 쓰는 것은 아니기 때문에 다양한 패턴이 있을 수 있습니다. 다만 `Functional Endpoint`를 사용할 때 가장 먼저 떠오르는 고민은 `Router Function`(이하 `Router`)과 `Handler Function`(이하 `Handler`)을 어떻게 나눌 것인가 하는 점입니다. 개념적으로는 서로 다르지만, 구현상으로는 한 클래스에 합쳐도 애플리케이션이 잘 동작합니다. 그래서 이건 프레임워크의 문법보다는 아키텍처의 문제에 더 가깝습니다.

이번 글에서는 `Router`와 `Handler`를 분리할지 말지를 몇 가지 관점에서 살펴보겠습니다.

## `Router`와 `Handler`는 분리해야 할까

Spring MVC에서는 `Controller`와 `Service`를 나누는 것이 거의 상식처럼 받아들여집니다. 아키텍처 관점에서도 그렇고, 프레임워크의 사고방식 자체도 그런 편입니다.

그렇다 보니 Spring Framework 계열인 WebFlux에서도 `Functional Endpoint`를 쓸 때 `Router`와 `Handler`를 따로 두어야 한다고 자연스럽게 생각하기 쉽습니다. 겉으로 보면 `Controller ≒ Router`, `Service ≒ Handler`처럼 대응되는 것 같고, 인터넷의 샘플 코드도 대체로 그런 구조로 작성되어 있습니다.

하지만 실제로 `Functional Endpoint`로 앱을 작성해 보면 생각해야 할 점이 있습니다. 정말 `Router`와 `Handler`가 `Controller`와 `Service`처럼 1:1 대응해야 할까요? 꼭 MVC 패턴을 그대로 따를 필요가 있을까요? 구현 방식에는 어떤 영향이 있을까요? 이번에는 이런 관점에서 이야기를 풀어 보겠습니다.

## 대응 관계

Spring의 [공식 문서](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-fn-overview)에서는 WebFlux의 `Functional Endpoint` 예제로 다음과 같은 코드를 보여 줍니다.

```kotlin
val repository: PersonRepository = ...
val handler = PersonHandler(repository)

val route = coRouter { 
    accept(APPLICATION_JSON).nest {
        GET("/person/{id}", handler::getPerson)
        GET("/person", handler::listPeople)
    }
    POST("/person", handler::createPerson)
}

class PersonHandler(private val repository: PersonRepository) {

    // ...

    suspend fun listPeople(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun createPerson(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun getPerson(request: ServerRequest): ServerResponse {
        // ...
    }
}
```

공식 예제에서 `Handler`를 별도 클래스로 두고 있으니, `Controller ≒ Router`, `Service ≒ Handler`처럼 보이기도 합니다. 하지만 `@RestController`나 `@Service`와 달리 `@Router`나 `@Handler` 같은 어노테이션은 없습니다. 즉 Spring의 사고방식상 `Router`와 `Handler`를 반드시 분리해야 하는 것은 아니라는 뜻으로 볼 수 있습니다.

따라서 적어도 아키텍처 관점에서 `Controller ≒ Router`, `Service ≒ Handler`라는 대응을 절대적이라고 보기는 어렵습니다.

그렇다면 `Router`와 `Handler`를 DI로 분리하면 어떻게 될까요? 보통은 아래처럼 작성할 수 있습니다.

```kotlin
@Configuration
class PersonRouter(private val handler: PersonHandler) {

    @Bean
    fun route(): RouterFunction<ServerResponse> =
        coRouter {
            accept(APPLICATION_JSON).nest {
                GET("/person/{id}", handler::getPerson)
                GET("/person", handler::listPeople)
            }
            POST("/person", handler::createPerson)
        }
}    

@Component
class PersonHandler(private val repository: PersonRepository) {

    // ...

    suspend fun listPeople(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun createPerson(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun getPerson(request: ServerRequest): ServerResponse {
        // ...
    }
}
```

`RouterFunction`은 `Functional Interface`라서 이를 반환하는 메서드를 `@Bean`으로 등록해야 합니다. `@Bean`은 보통 `@Configuration`이 담당하니, `Router` 쪽은 자연스럽게 설정 클래스가 됩니다. `Handler`는 일반적인 컴포넌트로 두면 됩니다.

이렇게 보면 구조는 분리되어 있지만, 어노테이션만 봐서는 딱히 드러나지 않습니다. 구현은 어렵지 않지만, 왜 굳이 이런 구조여야 하는지까지 생각해 보게 됩니다. 공식 문서의 다른 설명을 보면, WebFlux는 결국 `HandlerFunction`을 중심에 둔 모델임을 알 수 있습니다.

즉 `Handler`는 단순히 비즈니스 로직만 담당하는 것이 아니라, 요청과 응답을 직접 다루는 역할까지 함께 맡습니다. 그래서 기존 `Service`처럼 `Handler`가 비즈니스 로직만 담당한다고 생각하면 WebFlux의 철학과 어긋납니다.

물론 책임을 분리한다는 관점에서 클래스 자체를 나누는 발상은 틀린 것이 아닙니다. 다만 그 기준이 꼭 `Router`와 `Handler`일 필요는 없다는 뜻입니다.

## 책임 분산이라는 측면에서

어노테이션의 구현을 살펴보면, `@Controller`와 `@Service`를 나누는 것이 프레임워크의 아키텍처와 철학에 기반하고 있음을 더욱 명확히 알 수 있습니다. 각 어노테이션의 구현은 다음과 같습니다.

```java
@Target(value=TYPE)
@Retention(value=RUNTIME)
@Documented
@Component
public @interface Controller

@Target(value=TYPE)
@Retention(value=RUNTIME)
@Documented
@Component
public @interface Service
```

둘 다 구현이 동일하므로, 극단적으로 말하면 `Controller`에 `@Service`를 붙여도 기능적으로는 동일합니다. `@Service`의 Javadoc에는 다음과 같이 이 어노테이션이 존재하는 이유가 디자인 패턴에 기반하고 있음을 명시하고 있습니다.

> Indicates that an annotated class is a "Service", originally defined by Domain-Driven Design (Evans, 2003) as "an operation offered as an interface that stands alone in the model, with no encapsulated state."
May also indicate that a class is a "Business Service Facade" (in the Core J2EE patterns sense), or something similar. This annotation is a general-purpose stereotype and individual teams may narrow their semantics and use as appropriate.

따라서 애플리케이션 설계 관점에서 보면 `Controller`는 요청을 받고, 응답을 반환하고, 엔드포인트를 `Service`에 연결하는 책임만 가지며, `Service`는 비즈니스 로직을 처리하는 책임을 가진다고 할 수 있습니다. 같은 관점에서 생각하면, 어노테이션은 없지만 `Router`와 `Handler`도 동일한 책임을 가지도록 작성할 수 있을 것입니다.

다만 문제는 "요청 핸들링을 시작부터 끝까지 담당한다"는 정의입니다. 앞서의 샘플 코드를 자세히 보면, Handler의 메서드들은 모두 [ServerRequest](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerRequest.html)를 인자로 받고, 반환 값은 [ServerResponse](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerResponse.html)입니다. 이는 곧 `Router`와 `Handler`를 별도 클래스로 분리하더라도, 요청과 응답까지를 `Handler`에서 처리해야 한다는 뜻입니다.

여기서 "기존 `Controller`/`Service`처럼 `Handler`의 인자와 반환값만 달리 하면 안 될까?"라고 생각할 수 있습니다. 하지만 그것은 바로 프레임워크의 철학에 어긋나는 것입니다. `ServerRequest`와 `ServerResponse`의 Javadoc을 보면, 다음과 같이 "이들은 [HandlerFunction](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/HandlerFunction.html)에서 핸들링된다"고 명시하고 있습니다.

```kotlin
/**
 * Represents a server-side HTTP request, as handled by a {@code HandlerFunction}.
 *
 * <p>Access to headers and body is offered by {@link Headers} and
 * {@link #body(BodyExtractor)}, respectively.
 *
 * @author Arjen Poutsma
 * @author Sebastien Deleuze
 * @since 5.0
 */
public interface ServerRequest {

/**
 * Represents a typed server-side HTTP response, as returned
 * by a {@linkplain HandlerFunction handler function} or
 * {@linkplain HandlerFilterFunction filter function}.
 *
 * @author Arjen Poutsma
 * @author Juergen Hoeller
 * @author Sebastien Deleuze
 * @since 5.0
 */
public interface ServerResponse {
```

이로부터 알 수 있듯이, WebFlux에서는 `ServerRequest`와 `ServerResponse`를 `HandlerFunction`에서 다루도록 설계되어 있습니다. 따라서 기존의 `Service`처럼 `Handler`가 비즈니스 로직 "만" 다룬다는 것은, 구현이 가능한지의 문제보다 먼저 프레임워크 설계의 관점에서 문제가 되는 것입니다.

물론 "책임 분산"이라는 관점에서 책임에 따라 클래스를 나누는 생각 자체는 잘못된 것이 아닙니다. 그렇기 때문에 비즈니스 로직을 담당하는 클래스를 `Handler`와 분리하여 운영하는 경우도 생각해볼 수 있습니다. 다만 클래스를 나누는 기준이 반드시 `Router`와 `Handler`일 필요는 없다고 생각됩니다.

## 테스트 관점

Java에서 JUnit으로 단위 테스트를 작성할 때는 보통 유스케이스 단위로 테스트를 만들되, 클래스 단위로 묶는 경우가 많습니다. 그렇다면 `Router`와 `Handler`를 분리했을 때도 자연스럽게 그 단위로 테스트를 나누고 싶어집니다.

하지만 여기에는 문제가 있습니다. `Router`는 결국 엔드포인트와 `Handler`를 연결하는 역할뿐이라 테스트가 "정말 의도한 `HandlerFunction`을 호출하는가" 정도로만 줄어듭니다. `Handler` 쪽은 `ServerRequest`를 받아 `ServerResponse`를 반환하므로 테스트가 꽤 까다로워집니다.

왜 까다로운가 하면, `ServerRequest`를 직접 만들기도 어렵고 `ServerResponse`에서 본문을 꺼내기도 번거롭기 때문입니다. 결국 [WebTestClient](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/reactive/server/WebTestClient.html)를 쓰게 되는데, 그렇게 되면 사실상 실제 API를 호출하는 형태가 되어 `Handler`만 테스트하려던 의도가 `Router`까지 포함하게 됩니다. 이렇게 되면 클래스 단위로 테스트를 나누는 의미가 줄어들고, `Router`만 따로 테스트하는 것도 실익이 적어집니다.

## 그렇다면 어떻게 할까

지금까지의 세 관점만 보면 `Router`와 `Handler`를 굳이 나눌 이유가 크지 않아 보입니다. 오히려 분리하면 문제가 생길 가능성이 더 커 보입니다. 그렇다고 엔드포인트 라우팅과 비즈니스 로직을 분리하지 말자는 뜻은 아닙니다. 단지 그 기준을 `Router`와 `Handler`로 둘 필요는 없다는 이야기입니다.

예를 들어 다음 코드가 있다고 해 봅시다.

```kotlin
@Configuration
class PersonRouter(private val repository: PersonRepository) {

    @Bean
    fun route(): RouterFunction<ServerResponse> =
        coRouter {
            GET("/person/{id}") {
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(
                        repository.findById(it.pathVariable("id").toLong())
                            .map { record ->
                                PersonDto(record.id, record.name)
                            }
                    ).awaitSingle()
                }
        }
}
```

이 코드에서 `Body`를 만드는 부분을 제외하면 비즈니스 로직이라고 할 만한 것이 많지 않습니다. 그렇다면 `Body` 생성만 따로 분리해서 다른 클래스, 예를 들면 `Service`에 맡기는 편이 더 자연스럽습니다.

```kotlin
@Configuration
class PersonRouter(private val service: PersonService) {

    @Bean
    fun route(): RouterFunction<ServerResponse> =
        coRouter {
            GET("/person/{id}") {
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(service.getPersonById(it.pathVariable("id").toLong()))
                    .awaitSingle()
                }
        }
}
```

이런 식이면 `Router`는 경로와 응답 형태를 책임지고, `Service`는 실제 데이터를 만드는 책임을 맡게 됩니다. 책임 분리가 필요하다는 점은 유지하면서도 굳이 `Router`와 `Handler`로 나눌 필요는 없습니다.

## 마지막으로

개인적으로는 WebFlux의 `Functional Endpoint`에서 `Router`와 `Handler`를 반드시 분리해야 한다는 확신은 들지 않았습니다. 오히려 무조건 분리하면 테스트나 구조 면에서 애매해지는 부분도 있었습니다. 대신 엔드포인트와 비즈니스 로직을 분리해야 한다는 원칙은 유지하되, 그 기준을 꼭 `Router`와 `Handler` 조합으로 고정할 필요는 없다고 생각합니다.

결국 중요한 건 어떤 이름의 클래스를 쓰느냐가 아니라, 어떤 책임을 어디에 둘 것이냐입니다. WebFlux는 그 책임을 Spring MVC와는 조금 다르게 배치하게 만드는 프레임워크라는 점이 핵심입니다.
