---
title: "WebFlux에서 Functional Endpoint 선택하기"
date: 2021-05-30
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - webflux
  - kotlin
  - rest api
translationKey: "posts/spring-webflux-router"
---

이전에 [Spring WebFlux 이해하기](/ko/posts/spring-webflux/)에서 MVC 패턴(`Controller`/`Service`)과 `Functional Endpoint`(`Router`/`Handler`)를 잠깐 비교한 적이 있습니다. Functional Endpoint는 MVC를 완전히 대체한다기보다, 함수형 스타일에 더 잘 맞는 선택지를 제공하는 쪽에 가깝습니다.

그렇다면 WebFlux에서는 가능한 한 Functional Endpoint를 쓰는 편이 좋을까요? 전통적인 Spring MVC처럼 `Controller`/`Service`를 쓰는 방식이 정말 `Router`/`Handler`보다 불리한지, 그리고 Functional Endpoint만의 불편함은 없는지 한 번쯤 궁금해집니다.
이번 글에서는 WebFlux 서버 애플리케이션을 만들 때 이 두 패턴을 어떤 기준으로 고르면 좋을지 정리해 보겠습니다.
## 프로그래밍의 패러다임으로

Spring MVC는 어노테이션과 객체 지향을 바탕으로 한 익숙한 구성입니다. AOP나 DI 같은 Spring의 장점은 여전히 크지만, [Reactive Streams](https://www.reactive-streams.org) 기반의 `Mono`/`Flux`를 다루는 WebFlux 코드와 비교하면 확실히 전통적인 방식에 가깝습니다.
물론 여기서 객체 지향과 함수형 중 무엇이 더 낫다고 단정할 생각은 없습니다. Java도 1.8 이후 `Functional Interface` 덕분에 함수형 스타일을 어느 정도 쓸 수 있고, Kotlin도 크게 다르지 않습니다. 그래서 Spring MVC와 Spring WebFlux를 고른다고 해서 곧바로 "객체 지향 vs 함수형"의 문제로 이어지지는 않습니다.
다만 WebFlux에서는 `Controller`/`Service`와 `Router`/`Handler` 중 하나를 택해야 하므로, 구현 스타일을 어디에 둘지는 정해야 합니다. 이때는 팀이 익숙한 방식, 그리고 코드가 더 빨리 읽히고 유지보수하기 쉬운 쪽을 고르는 편이 현실적입니다. 결국 프로그래밍 패러다임도 효율을 높이기 위해 발전해 왔기 때문입니다.
그럼 두 방식이 실제로 어떻게 다른지 코드로 보겠습니다.
### MVC 패턴으로 작성하는 경우

Spring WebFlux의 MVC 패턴, 즉 `Controller`/`Service` 방식은 기존 Spring MVC와 크게 다르지 않게 사용할 수 있습니다. 아래처럼만 작성하면, 겉으로는 Spring MVC와 거의 구분이 가지 않습니다.
```kotlin
@RestController
class HelloController(private val service: HelloService) {

    @GetMapping("/hello")
    fun hello(): ResponseEntity = 
        ResponseEntity
            .ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(service.getHello())
}

@Service
class HelloService {

    fun getHello(): String = "Hello Spring WebFlux"
}
```

다만 DB 연결까지 완전히 논블로킹으로 처리하려면 [R2DBC](https://r2dbc.io/) 같은 비동기 API를 써야 합니다. 즉 `Reactive Stream`을 따라가야 하고, 그 결과 코드도 `Mono`/`Flux` 중심으로 바뀝니다.
Repository가 `Mono`/`Flux`를 반환하기 시작하면, 그에 맞춰 서비스 코드도 다시 짜야 합니다. 이름만 보면 Java Stream처럼 가볍게 다룰 수 있을 것 같지만, 실제로는 생각보다 손이 많이 갑니다. 예를 들어 JPA나 MyBatis 같은 블로킹 기반 API를 쓰면 Service 메서드는 보통 아래처럼 끝납니다.
```kotlin
    // 사용자 ID로 사용자 정보와 메일 발송 이력을 조회한다
fun getMemberWithMailRecord(memberId: Int): MemberWithMailRecord {
    // 사용자 정보를 조회한다
    val member = memberRepository.getMember(id = memberId) ?: throw RuntimeException("Can't find member")
    // 사용자가 생성한 메일 발송 이력을 조회한다
    val mailRecord = mailRecordRepository.getMailRecord(memberId = memberId) ?: throw RuntimeException("Can't find mailRecord")
    // 사용자 정보와 메일 발송 이력을 합쳐 반환한다
    return MemberWithMailRecord(
            member = member,
            mailRecord = mailRecord  
        )
}
```

반대로 `Mono`/`Flux`를 반환하는 API를 쓰면 코드는 이렇게 바뀝니다.
```kotlin
fun getMemberWithMailRecord(memberId: Int): Mono<MemberWithMailRecord> =
    memberRepository.getMember(id = memberId)
        .switchIfEmpty(Mono.error(RuntimeException("Can't find member")))
        .zipWith(mailRecordRepository.getMailRecord(memberId = memberId).switchIfEmpty(Mono.error(RuntimeException("Can't find mailRecord")))
        .map { MemberWithMailRecord(
              member = it.t1,
              mailRecord = it.t2
          )
        }
```

하려는 일은 같아도 코드가 과연 쓰기 쉽고 읽기 쉬운지는 별개의 문제입니다. 특히 `Flux`를 `List`로 바꾸거나, 한 `Mono`의 결과를 바탕으로 또 다른 `Mono`를 이어 붙여야 하는 상황에서는 점점 더 복잡해집니다.
다행히 Kotlin에는 [Coroutines](https://kotlinlang.org/docs/coroutines-overview.html)이 있어서 이런 코드를 훨씬 단순하게 쓸 수 있습니다. Coroutines를 적용하면 `Mono`/`Flux`를 써도 코드는 아래처럼 정리됩니다.
```kotlin
suspend fun getMemberWithMailRecord(memberId: Int): MemberWithMailRecord {
    val member = memberRepository.getMember(id = memberId).awaitFirstOrNull() ?: throw RuntimeException("Can't find member")
    val mailRecord = mailRecordRepository.getMailRecord(memberId = memberId).awaitFirstOrNull() ?: throw RuntimeException("Can't find mailRecord")
    return MemberWithMailRecord(
            member = member,
            mailRecord = mailRecord  
        )
}
```

Coroutines를 쓸 때는 스코프를 신경 써야 하지만, 실제로는 Controller 메서드까지 `suspend`로 선언해 두는 편이 더 깔끔합니다. 다만 기존 Spring MVC 프로젝트를 WebFlux로 옮길 때 이런 식으로 많은 처리를 `suspend` 메서드로 바꾸면 유닛 테스트 수정이 번거로울 수 있습니다.
### Functional Endpoint로 작성하는 경우

이어서 Functional Endpoint를 사용하는 경우를 보겠습니다. MVC 패턴처럼 애플리케이션 구조에 따라 Router와 Handler도 클래스를 나눠 관리하게 됩니다. 이때 Router는 `@Bean`으로 등록하거나 Handler는 `@Component`로 정의할 수 있습니다. 예시는 다음과 같습니다.
```kotlin
@Configuration
class HelloRouter(private val handler: HelloHanlder) {

    @Bean
    fun hello(): router {
        GET("/hello", handler::getHello)
    }
}

@Component
class HelloHandler {

    fun getHello(request: ServerRequest): Mono<ServerResponse> =
        ServerResponse
            .ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(Mono.Just("Hello Spring WebFlux"))
}
```

Functional Endpoint의 핵심은 Router가 엔드포인트와 Handler를 연결하는 역할만 맡고, Handler가 [ServerRequest](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerRequest.html)를 받아 [ServerResponse](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerResponse.html)를 직접 반환한다는 점입니다. MVC 패턴에서는 `RestController`의 반환값으로 `ResponseEntity`나 별도 DTO를 비교적 자유롭게 쓸 수 있고, Service는 비즈니스 로직에만 집중하는 경우가 많아서 구조가 꽤 다르게 느껴집니다.
`ServerRequest`와 `ServerResponse`를 직접 다루다 보면 Handler가 Service보다 한 단계 더 로우레벨에 가까워집니다. 예를 들면 다음과 같습니다.
```kotlin
fun getMember(request: ServerRequest): Mono<ServerResponse> =
    ServerResponse
        .ok()
        .contentType(MediaType.APPLICATION_JSON)
        .body(memberRepository.getMember(id = request.PathVariable("id"))
            .switchIfEmpty(Mono.error(RuntimeException("Can't find member")))
            .zipWith(mailRecordRepository.getMailRecord(memberId = request.PathVariable("id")).switchIfEmpty(Mono.error(RuntimeException("Can't find mailRecord")))
            .map { MemberWithMailRecord(
                member = it.t1,
                mailRecord = it.t2
                )
            }
        )

```

이 경우에도 Coroutines를 사용할 수 있습니다. Coroutines를 쓰면 다음과 같이 정리할 수 있습니다.
```kotlin
suspend fun getMember(request: ServerRequest): Mono<ServerResponse> {
    val member = memberRepository.getMember(id = memberId).awaitFirstOrNull() ?: throw RuntimeException("Can't find member")
    val mailRecord = mailRecordRepository.getMailRecord(memberId = memberId).awaitFirstOrNull() ?: throw RuntimeException("Can't find mailRecord")
    return ServerResponse
                    .ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .bodyValueAndAwait(
                        Mono.just(
                            MemberWithMailRecord(
                                member = it.t1,
                                mailRecord = it.t2
                            )
                        )
                    }
}
```

## 성능

MVC 패턴의 약점으로 어노테이션이 자주 거론됩니다. 어노테이션을 많이 쓰면 리플렉션이 따라오고, 그만큼 성능 손실로 이어질 수 있다는 이야기입니다. 이 관점만 보면 WebFlux에서는 MVC 패턴보다 Functional Endpoint가 유리해 보입니다. 실제로는 어떨까요?
[Spring 공식 문서](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-programming-models)에서도 Functional Endpoint를 `lightweight(경량)`라고 설명하지만, 그 밖의 성능 차이에 대해서는 자세히 언급하지 않습니다. 일반적인 성능 비교는 Spring MVC와 Spring WebFlux 전체를 대상으로 한 경우가 많아서, WebFlux 내부에서 두 패턴을 직접 비교한 자료는 많지 않았습니다. 그래서 여기서는 간단한 벤치마크로 차이를 확인해 보았습니다.
벤치마크 도구는 JMeter도 고려했지만, 명령 한 줄로 쉽게 측정할 수 있는 점을 생각해 이번에는 [Apache HTTP server benchmarking tool](https://httpd.apache.org/docs/2.4/en/programs/ab.html)을 사용했습니다.
### 사용한 코드

성능 측정에서 알고 싶은 것은 "구현 패턴에 따라 성능이 달라지는가"였습니다. 그래서 DB 접속은 일부러 제외하고, 데이터를 생성하는 공통 로직과 그 값을 반환하는 Controller/Service, Router/Handler 조합만 준비했습니다.
#### 공통

데이터 생성 로직은 공통으로 두고, 단순히 루프로 `List`를 만드는 방식으로 구성했습니다.
```kotlin
// 고정 데이터를 생성해 반환하는 클래스
object DataCreationUtil {

    // 1970년 1월 1일부터 2021년 12월 31일까지
    var data: List<Person> = (0..18992)
        .map {
            Person(
                id = it.toLong(),
                name = "Person${it}",
                birth = LocalDate.ofEpochDay(it.toLong()),
                joined = LocalDate.ofEpochDay(it.toLong() + 10000)
            )
        }.toList()
}

// 생성되는 데이터
data class Person(
    val id: Long,
    val name: String,
    val birth: LocalDate,
    val joined: LocalDate
)
```

#### Controller/Service 구현

MVC 패턴은 코드만 봐도 구조를 쉽게 파악할 수 있으므로 설명은 짧게 보겠습니다.
```kotlin
@RestController
class PerformanceTestController(
    private val service: PerformanceTestService
) {

    @GetMapping("/performance-controller")
    fun getData(): ResponseEntity<List<Person>> =
        ResponseEntity
            .ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(service.getData())
}

@Service
class PerformanceTestService {

    fun getData(): List<Person> = service.getData()
}
```

#### Router/Handler 구현

Functional Endpoint에서는 MVC 패턴에서 Service가 맡던 처리까지 Handler가 직접 담당합니다.
```kotlin
@Configuration
class PerformanceTestRouter(private val handler: PerformanceTestHandler) {

    @Bean
    fun route() = router {
        GET("/performance-router", handler::getData)
    }
}

@Component
class PerformanceTestHandler {

    fun getData(request: ServerRequest): Mono<ServerResponse> =
        ServerResponse
            .ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(Flux.fromIterable(DataCreationUtil.data))
}
```

### 테스트 결과

테스트는 다음 조건으로 진행했습니다.
- 사용자 수는 5000, 사용자당 요청 수는 50으로 설정
- 웜업 시간을 고려해 패턴별로 테스트를 분리
  - 서버를 재부팅한 뒤 테스트 수행
  - 총 10회 반복

실제로 사용한 스크립트는 다음과 같습니다. 서버를 시작한 뒤 이 스크립트를 실행하고, 10회 루프가 끝나면 재부팅한 뒤 Functional Endpoint 테스트를 진행했습니다.
```bash
#!/bin/bash

for i in {1..10}
do
 ab -n 5000 -c 50 http://localhost:8080/performance-controller
done
```

다만 이렇게 해도 결과는 회차마다 차이가 있었습니다. 그래서 여기서는 중앙값에 해당하는 결과만 소개하겠습니다. 결론부터 말하면 MVC 패턴과 Functional Endpoint의 성능 차이는 사실상 오차 범위 안으로 봐도 무방했습니다.
#### Controller/Service 결과

```bash
Server Software:        
Server Hostname:        localhost
Server Port:            8080

Document Path:          /performance-controller
Document Length:        1440242 bytes

Concurrency Level:      50
Time taken for tests:   24.989 seconds
Complete requests:      5000
Failed requests:        0
Total transferred:      7201590000 bytes
HTML transferred:       7201210000 bytes
Requests per second:    200.09 [#/sec] (mean)
Time per request:       249.892 [ms] (mean)
Time per request:       4.998 [ms] (mean, across all concurrent requests)
Transfer rate:          281433.26 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   1.4      0      11
Processing:    29  248  79.8    242     516
Waiting:       24  192  60.8    185     442
Total:         29  249  79.4    243     516

Percentage of the requests served within a certain time (ms)
  50%    243
  66%    275
  75%    295
  80%    309
  90%    354
  95%    394
  98%    430
  99%    449
 100%    516 (longest request)
```

#### Router/Handler 결과

```bash
Server Software:        
Server Hostname:        localhost
Server Port:            8080

Document Path:          /performance-router
Document Length:        1440257 bytes

Concurrency Level:      50
Time taken for tests:   25.541 seconds
Complete requests:      5000
Failed requests:        0
Total transferred:      7201775000 bytes
HTML transferred:       7201285000 bytes
Requests per second:    195.76 [#/sec] (mean)
Time per request:       255.410 [ms] (mean)
Time per request:       5.108 [ms] (mean, across all concurrent requests)
Transfer rate:          275360.22 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   3.2      0     151
Processing:    33  253  80.4    246     612
Waiting:       28  194  59.8    184     475
Total:         39  254  80.0    247     613

Percentage of the requests served within a certain time (ms)
  50%    247
  66%    286
  75%    302
  80%    312
  90%    361
  95%    398
  98%    441
  99%    459
 100%    613 (longest request)
```

## 다큐멘테이션

다음은 문서화 관점입니다. 여기서 말하는 문서는 JavaDoc나 KDoc 같은 주석이 아니라, 흔히 쓰는 [Swagger](https://swagger.io)입니다.
[SpringFox](http://springfox.github.io/springfox) 등을 쓰면 API 문서를 쉽게 만들 수 있지만, Functional Endpoint에서는 그렇게 단순하지 않습니다. Router는 엔드포인트와 Handler를 연결할 뿐이고, Handler는 입력이 `ServerRequest`, 반환값이 `ServerResponse`로 고정되기 때문입니다.
물론 `ServerRequest`와 `ServerResponse`를 써도 원하는 형태로 데이터를 다룰 수 있습니다. 예를 들어 요청에서 매개변수를 꺼내는 방법은 다음과 같습니다.
```kotlin
// Path Variable로 전달된 ID의 사용자 정보를 조회한다
suspend fun getMember(request: ServerRequest): ServerResponse {
    // Path Variable를 가져온다
    val id = request.pathVariable("id")
    // ...
}

// Request Body로 전달된 데이터를 바탕으로 새 사용자를 만든다
suspend fun createMember(request: ServerRequest): ServerResponse {
    // Request Body를 클래스로 매핑한다
    val form = request.bodyToMono(MemberCreateForm::class.java).awaitFirst()
    // ...
} 
```

다만 SpringFox처럼 API 문서를 자동 생성하는 도구는 대체로 리플렉션에 의존합니다. 그래서 Handler 내부에서 실제로 어떤 처리가 이루어지는지 추적하기가 어렵습니다.
다행히 이 문제를 Spring 쪽에서도 인식하고 있어서, [springdoc-openapi](https://springdoc.org)를 사용하면 Functional Endpoint에서도 API 문서를 만들 수 있습니다. 하지만 이 경우에도 완전 자동은 아니고, 문서를 위한 메타데이터를 따로 지정해야 합니다. 그래서 Router나 Handler에 대해 다음처럼 어노테이션을 일일이 붙여야 합니다.
```kotlin
@Bean
@RouterOperations(
    value =
        [
            RouterOperation(
                path = "/api/v1/members",
                beanClass = MemberHandler::class,
                beanMethod = "listMember",
                method = [RequestMethod.GET],
                produces = [MediaType.APPLICATION_JSON_VALUE]
            ),
            RouterOperation(
                path = "/api/v1/members/{id}",
                beanClass = MemberHandler::class,
                beanMethod = "getMember",
                method = [RequestMethod.GET],
                produces = [MediaType.APPLICATION_JSON_VALUE]
            ),
            RouterOperation(
                path = "/api/v1/members",
                beanClass = MemberHandler::class,
                beanMethod = "createMember",
                method = [RequestMethod.POST],
                produces = [MediaType.APPLICATION_JSON_VALUE]
            ),
            RouterOperation(
                path = "/api/v1/members/{id}",
                beanClass = MemberHandler::class,
                beanMethod = "updateMember",
                method = [RequestMethod.PUT],
                produces = [MediaType.APPLICATION_JSON_VALUE]
            ),
            RouterOperation(
                path = "/api/v1/members/{id}",
                beanClass = MemberHandler::class,
                beanMethod = "deleteMember",
                method = [RequestMethod.DELETE],
                produces = [MediaType.APPLICATION_JSON_VALUE]
            ),
        ]
    )
fun routeMember() = coRouter {
    GET("/api/v1/members") { handler.listMember() }
    GET("/api/v1/members/{id}", handler::getMember)
    POST("/api/v1/members", handler::createMember)
    PUT("/api/v1/members/{id}", handler::updateMember)
    DELETE("/api/v1/members/{id}", handler::deleteMember)
}
```

보시다시피 문서용 설정이 실제 코드보다 더 길어집니다. Functional Endpoint에서도 Swagger를 쓸 수는 있지만, MVC 패턴보다 불편한 건 사실입니다. 그래서 문서화가 중요하다면 아직은 Functional Endpoint를 신중하게 선택하는 편이 좋습니다.
## 마지막으로

여기서는 Spring WebFlux의 MVC 패턴과 Functional Endpoint를 코드 작성 방식, 성능, 문서화 관점에서 비교해 보았습니다. WebFlux가 나온 지는 꽤 됐지만, 아직은 MVC 패턴보다 손이 더 가는 부분이 분명히 있습니다. 제 경우에는 논블로킹과 함수형 스타일을 적극적으로 살리고 싶다면 Functional Endpoint가 더 어울린다고 보지만, 실제 프로젝트에서는 굳이 그 정도까지 갈 필요가 없는 경우도 많습니다. 특히 엔터프라이즈 애플리케이션이라면 더 그렇습니다. 결국 먼저 따져 봐야 할 것은 "Functional Endpoint를 쓸까"보다 "애초에 WebFlux가 정말 필요한가"일 수도 있습니다.

그렇다고 Functional Endpoint의 가능성이 작은 것은 아닙니다. Spring WebFlux 바깥에서도 이런 라우팅 방식은 이미 널리 쓰이고 있습니다. [Express](https://expressjs.com/ko)나 [Gin](https://github.com/gin-gonic/gin) 같은 프레임워크도 비슷한 모델을 제공하므로, 함수형 스타일에 익숙해지는 연습으로도 충분히 의미가 있습니다.
