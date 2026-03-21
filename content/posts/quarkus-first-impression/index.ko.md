---
title: "Quarkus를 써 본 첫인상"
date: 2021-10-24
categories:
  - quarkus
image: "../../images/quarkus.webp"
tags:
  - java
  - kotlin
  - ktor
translationKey: "posts/quarkus-first-impression"
---

Spring MVC는 좋은 프레임워크이지만, 최근 자주 이야기되는 [마이크로서비스](https://ko.wikipedia.org/wiki/%EB%A7%88%EC%9D%B4%ED%81%AC%EB%A1%9C%EC%84%9C%EB%B9%84%EC%8A%A4) 환경에는 잘 맞지 않는다는 비판도 있습니다. 대표적으로 앱 기동 속도가 느리고, 패키지 크기가 크며, 메모리 사용량이 많다는 점이 자주 언급됩니다. 시작이 느리면 변경 사항을 빠르게 반영하기 어렵고, 앱 크기와 메모리 사용량이 크면 인스턴스가 늘어날수록 비용 부담도 커집니다. 이런 이야기는 마이크로서비스에만 한정되지 않습니다. 서버리스 환경에서 굳이 JavaScript나 Python 같은 인터프리터 언어를 선택하는 이유도 결국 비슷한 맥락에 있다고 볼 수 있습니다.

그렇다면 이런 문제를 어떻게 해결할 수 있을까요? 사실 이 문제를 전부 Spring 하나의 문제로만 보기는 어렵습니다. 다른 프레임워크와 비교했을 때 Spring이 아주 빠른 편은 아니고 메모리 사용량도 적지 않다는 점은 분명하지만, JVM 기반 언어를 쓰는 한 어느 정도는 피하기 어려운 한계도 있습니다. JVM 언어에서는 애플리케이션의 기동 시간, 크기, 메모리 사용량을 따질 때 결국 JVM 자체가 차지하는 비중까지 함께 봐야 하기 때문입니다.

물론 그렇다고 해결책이 전혀 없는 것은 아닙니다. 이번에 소개할 [Quarkus](https://quarkus.io)는 바로 이런 문제의식에서 나온 프레임워크라고 볼 수 있습니다.

## Quarkus란

Quarkus는 [RHEL](https://www.redhat.com/ko/technologies/linux-platforms/enterprise-linux)로 잘 알려진 Red Hat이 만든 Java용 웹 프레임워크입니다. 공식 홈페이지 설명이 가장 정확하니, 우선 이 문장을 그대로 보는 편이 좋습니다.

> A Kubernetes Native Java stack tailored for OpenJDK HotSpot and GraalVM, crafted from the best of breed Java libraries and standards.

즉 Quarkus의 핵심 정체성은 Java 애플리케이션을 `Kubernetes Native`하게 만들 수 있다는 데 있습니다. 이름은 Java 프레임워크지만, Kotlin 같은 다른 JVM 언어도 충분히 사용할 수 있으므로 Kotlin 기반 서비스에도 자연스럽게 고려할 수 있습니다.

여기서 눈에 띄는 표현은 역시 `Kubernetes Native`입니다. 이 말은 단순히 "컨테이너 만들기 쉽다" 정도로 이해하면 부족하다고 생각합니다. Spring Boot 2.3부터도 [Docker 이미지 생성 기능](https://spring.io/blog/2020/08/14/creating-efficient-docker-images-with-spring-boot-2-3)이 들어갔고, [Jib](https://github.com/GoogleContainerTools/jib) 같은 도구를 쓰면 Java 애플리케이션을 컨테이너화하는 것 자체는 얼마든지 가능합니다. 굳이 `Kubernetes Native`라고 강조하는 이유는, 단순히 컨테이너를 만들 수 있다는 수준이 아니라 Kubernetes 환경에 맞춰 설계되고 최적화되었다는 뜻으로 보는 편이 자연스럽습니다.

그렇다면 애플리케이션 관점에서 `Kubernetes Native`하다는 것은 무엇일까요? Quarkus는 공식적으로 다음과 같은 특징을 내세웁니다.

- Native 컴파일 가능
- 빠른 기동 속도
- 낮은 메모리 사용량

Native 컴파일이 가능하다는 것은 결국 JVM 없이 애플리케이션을 실행할 수 있다는 뜻이므로, 앞서 말한 세 가지 문제를 크게 줄일 수 있습니다. 그렇다면 마이크로서비스나 서버리스뿐 아니라, 컨테이너 단위로 배포하는 환경에서도 상당한 장점이 됩니다. 게다가 JVM 모드로 실행할 때도 다른 프레임워크보다 기동 속도와 메모리 사용량에서 유리하다고 알려져 있으니, Native 컴파일을 하지 않더라도 충분히 매력적인 선택지가 될 수 있습니다.

## 실제로 만져 보면

표면적인 특징만 보면 꽤 매력적이지만, 프레임워크는 직접 써 보기 전까지는 잘 드러나지 않는 부분이 많습니다. 그래서 간단한 샘플을 만들어 직접 만져 봤고, 그 과정에서 느낀 점을 정리해 보겠습니다.

### 기동 속도

#### Spring Boot의 경우

Spring의 기동 속도는 주입되는 빈과 구성에 따라 크게 달라집니다. 여기서는 최대한 단순한 기준점을 보기 위해 [Spring Initializr](https://start.spring.io)에서 아래 조건만 설정한 애플리케이션을 사용했습니다.

- Project: Gradle
- Language: Kotlin
- Spring Boot: 2.5.5
- Packaging: War
- Java: 11
- Dependencies: 없음

실행은 로컬에서 Oracle JDK 17로 했습니다. 체감상 Java 11 때보다 조금 빠른 느낌도 있었습니다. 실제 결과는 다음과 같았습니다. 로컬 머신 정보는 일부 지워 두었습니다.

```bash
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.5.5)

2021-10-17 19:10:21.472  INFO 48844 --- [           main] com.example.demo.DemoApplicationKt       : Starting DemoApplicationKt using Java 17 on Local.
2021-10-17 19:10:21.475  INFO 48844 --- [           main] com.example.demo.DemoApplicationKt       : No active profile set, falling back to default profiles: default
2021-10-17 19:10:22.058  INFO 48844 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2021-10-17 19:10:22.067  INFO 48844 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2021-10-17 19:10:22.068  INFO 48844 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.53]
2021-10-17 19:10:22.117  INFO 48844 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2021-10-17 19:10:22.118  INFO 48844 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 607 ms
2021-10-17 19:10:22.335  INFO 48844 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2021-10-17 19:10:22.342  INFO 48844 --- [           main] com.example.demo.DemoApplicationKt       : Started DemoApplicationKt in 1.129 seconds (JVM running for 1.396)
```

JVM이 뜨는 데 1.396초, 애플리케이션 자체가 시작되는 데 1.129초가 걸렸습니다. 의존성이 거의 없는 상태였기 때문에 제 환경에서는 사실상 가장 빠른 축에 속하는 결과라고 볼 수 있습니다. 하지만 실제 업무용 애플리케이션에서는 시작만 10초 이상 걸리는 경우도 흔합니다. 한 번만 뜨는 서버라면 큰 문제가 아닐 수 있지만, 로컬 테스트나 반복적인 실행 환경에서는 이 차이가 꽤 크게 느껴집니다.

#### Quarkus Native의 경우

이번에는 Quarkus를 보겠습니다. Quarkus는 Native 컴파일을 지원하므로 [GraalVM](https://www.graalvm.org/)으로 빌드해 봤습니다. Gradle 태스크로 실행할 수 있어서 시도 자체는 어렵지 않았습니다. 실행 결과는 아래와 같습니다.

```bash
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2021-10-23 19:24:18,395 INFO  [io.quarkus] (main) quarkus-sample 1.0.0-SNAPSHOT native (powered by Quarkus 2.3.0.Final) started in 0.018s. Listening on: http://0.0.0.0:8080
2021-10-23 19:24:18,397 INFO  [io.quarkus] (main) Profile prod activated. 
2021-10-23 19:24:18,397 INFO  [io.quarkus] (main) Installed features: [cdi, config-yaml, kotlin, resteasy-reactive, resteasy-reactive-jackson, smallrye-context-propagation, vertx]
```

0.018초입니다. 샘플 프로젝트가 단순하다는 점은 감안해야 하지만, 그래도 이 기동 속도는 확실히 인상적입니다. 이 정도면 마이크로서비스는 물론이고, 요청량이 많은 서버리스 환경에서도 충분히 고려할 수 있겠다는 생각이 들었습니다.

#### Quarkus JVM의 경우

이번에는 Spring과 동일하게 Oracle JDK 17에서 JVM 모드로 실행해 봤습니다. Quarkus에는 개발 모드가 있어서 서버를 띄운 채로 수정하는 경험도 제공하지만, 여기서는 일부러 Jar를 만들어 실행했습니다. 참고로 Spring에서는 의존성을 통째로 묶은 결과물을 보통 `war`로 이야기하는 경우가 많은데, Quarkus가 `uber-jar`라는 표현을 쓰는 점은 조금 재미있었습니다.

```bash
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2021-10-23 19:20:59,897 INFO  [io.quarkus] (main) quarkus-sample 1.0.0-SNAPSHOT on JVM (powered by Quarkus 2.3.0.Final) started in 0.761s. Listening on: http://0.0.0.0:8080
2021-10-23 19:20:59,905 INFO  [io.quarkus] (main) Profile prod activated. 
2021-10-23 19:20:59,906 INFO  [io.quarkus] (main) Installed features: [cdi, config-yaml, kotlin, resteasy-reactive, resteasy-reactive-jackson, smallrye-context-propagation, vertx]
```

이번에는 0.761초가 걸렸습니다. Native에 비하면 수십 배 느리지만, 그래도 Spring과 비교하면 꽤 빠른 편입니다.

이렇게 기동 속도가 빨라지면 로컬 개발에서도 이점이 큽니다. 특히 테스트에서 애플리케이션을 여러 번 띄우는 구조라면 차이가 더 크게 납니다. Spring에서 [WebTestClient](https://spring.pleiades.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/reactive/server/WebTestClient.html) 같은 도구를 활용한 테스트를 많이 작성하면, 테스트 수가 늘어날수록 앱 기동 시간이 그대로 부담이 됩니다. 그런 관점에서 Quarkus의 빠른 스타트업은 개발 생산성에도 의미 있는 차이를 만들 수 있습니다.

### Spring에서 옮기기 쉽다

처음에는 크게 의식하지 않았지만, Quarkus의 장점 중 하나는 Spring에서 넘어오기 꽤 쉽다는 점이었습니다. 새 프로젝트를 시작하거나 기존 프로젝트의 프레임워크를 바꾸는 상황에서는 성능만이 아니라, 얼마나 공수를 줄일 수 있는지, 얼마나 쉽게 인력을 확보할 수 있는지도 중요한 판단 기준이 됩니다. 현재 팀에게 완전히 새로운 기술이거나 업계에서 잘 쓰이지 않는 기술이면 회사와 엔지니어 모두에게 부담이 커집니다.

그런 면에서, 새로운 기술이면서도 이미 널리 쓰이는 스타일과 비슷하다는 점은 꽤 큰 장점입니다. 실제 코드로 보면 이 부분이 더 잘 드러납니다.

#### Spring의 경우

예를 들어 쿼리 파라미터로 ID를 받아 `Person` 데이터를 조회하는 API가 있다고 해 봅시다. Spring에서는 보통 이런 식으로 작성할 수 있습니다.

```kotlin
@RestController
@RequestMapping("/api/v1/person")
class PersonController(private val service: PersonService) {
    @GetMapping
    fun getPerson(id: Int): PersonResponse {
        return service.getPerson(id).let { PresonResponse.from(it) }
    }
}

@Service
class PersonService(private val repository: PersonRepository) {
    fun getPerson(id: Int): PersonDto {
        return repository.findById(id).let { PersonDto.from(it) }
    }
}
```

#### Ktor의 경우

비교를 위해 같은 코드를 [Ktor](https://ktor.io/)로 작성하면 어떤 느낌인지도 보겠습니다. Ktor도 좋은 프레임워크지만, 설계 철학 자체가 Spring과는 꽤 다릅니다. 그래서 기존 Spring 애플리케이션을 옮긴다고 생각하면 고려해야 할 지점이 늘어납니다. 예를 들어 기본적으로 DI를 내장하고 있지 않으니 별도 라이브러리를 도입해야 합니다.

아래는 같은 API를 [Koin](https://insert-koin.io/)으로 DI를 붙여 구현한 예시입니다.

```kotlin
fun main() {
    // DI 설정
    val personModule = module {
        single { PersonService(get()) }
        single { PersonRepository() }
    }

    // Koin 설치
    install(Koin) {
        modules(personModule)
    }

    // 라우팅 설정
    configureRouting()
}

// 라우터에 Controller 등록
fun Application.configureRouting() {
    routing {
        personController()
    }
}

fun Route.personController() {
    // Service 주입
    val service: PersonService by inject()
    get("/api/v1/person") {
        return service.getPerson(id).let { PresonResponse.from(it) }
    }
}

// Service
class PersonService(private val repository: PersonRepository) {
    fun getPerson(id: Int): PersonDto {
        return repository.findById(id).let { PersonDto.from(it) }
    }
}
```

Spring과는 구조 자체가 꽤 다르다는 점이 바로 보입니다.

#### Quarkus의 경우

이제 Quarkus 쪽을 보겠습니다. Quarkus에서는 [RESTEasy](https://resteasy.github.io)와 [Reactive Routes](https://quarkus.io/guides/reactive-routes) 같은 방식이 있지만, 여기서는 RESTEasy 기반 구현을 예로 들겠습니다.

```kotlin
@Path("/api/v1/person")
class PersonController(private val service: PersonService) {
    @GET
    fun getPerson(id: Int): PersonResponse {
        return service.getPerson(id).let { PresonResponse.from(it) }
    }
}

@ApplicationScoped
class PersonService(private val repository: PersonRepository) {
    fun getPerson(id: Int): PersonDto {
        return repository.findById(id).let { PersonDto.from(it) }
    }
}
```

Spring 코드와 비교해 보면, 쓰는 어노테이션만 다를 뿐 구조와 감각은 거의 비슷합니다. 그래서 Ktor처럼 아키텍처를 다시 고민해야 하는 부담이 상대적으로 적고, 마이그레이션도 훨씬 수월해 보입니다.

또 RESTEasy 기반으로 가면 Reactive API를 만들기 쉬운 점도 장점입니다. 이 경우 [Mutiny](https://smallrye.io/smallrye-mutiny/)를 쓰게 되는데, `Uni`/`Multi`는 `Mono`/`Flux`와 거의 일대일 대응으로 이해해도 될 정도라서, Spring WebFlux를 써 본 사람이라면 금방 적응할 수 있습니다.

```kotlin
@Path("/api/v1/person")
class PersonController(private val service: PersonService) {
    @GET
    fun getPerson(id: Int): Uni<PersonResponse> {
        return Uni.createFrom().item(service.getPerson(id).let { PresonResponse.from(it) })
    }
}
```

물론 Ktor나 Spring WebFlux의 Router Function 스타일도 나름의 장점이 있습니다. 다만 많은 엔지니어가 여전히 Spring MVC 스타일에 익숙하고, 그 방식이 특별히 문제가 되는 것도 아니기 때문에, Quarkus처럼 기존 감각을 살린 채 새로운 프레임워크를 제공하는 접근은 분명한 강점이라고 생각합니다. JavaScript 진영의 [NestJS](https://nestjs.com/)가 Spring과 비슷한 방식으로 코드를 쓰게 해 주는 것도 같은 맥락일 것입니다.

이런 면에서 보면 Spring 경험이 있는 엔지니어라면 Quarkus에 비교적 쉽게 적응할 수 있고, 기존 Spring 프로젝트도 꽤 현실적으로 옮길 수 있을 것 같습니다.

## 우려되는 점

직접 만져 보니 좋은 점은 분명했지만, Native 빌드까지 시도하면서 몇 가지 우려도 느꼈습니다.

### Native 빌드는 느리다

Native 모드에서 기동 속도가 매우 빠르다는 점은 분명 장점입니다. 하지만 그만큼 빌드 자체는 느립니다. Native 빌드는 결국 모든 코드를 머신 코드로 미리 컴파일하는 작업이기 때문에 시간이 많이 걸릴 수밖에 없습니다. JVM용 바이트코드는 어느 환경에서나 동일하지만, 머신 코드는 플랫폼별로 달라지므로 그에 맞는 결과물을 만드는 데 비용이 드는 것은 당연합니다.

제가 테스트용으로 만든 프로젝트를 Native 이미지로 빌드했을 때는 다음 정도의 시간이 걸렸습니다.

```bash
./gradlew build -Dquarkus.package.type=native

> Task :quarkusBuild
building quarkus jar

[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]    classlist:   2,311.58 ms,  1.19 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]        (cap):   3,597.91 ms,  1.19 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]        setup:   5,450.32 ms,  1.19 GB
19:22:21,827 INFO  [org.jbo.threads] JBoss Threads version 3.4.2.Final
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]     (clinit):     779.71 ms,  4.33 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]   (typeflow):  14,308.32 ms,  4.33 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]    (objects):  16,140.38 ms,  4.33 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]   (features):   1,145.40 ms,  4.33 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]     analysis:  33,857.15 ms,  4.33 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]     universe:   1,718.32 ms,  5.14 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]      (parse):   2,635.36 ms,  5.14 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]     (inline):   7,363.76 ms,  5.99 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]    (compile):  26,314.40 ms,  6.15 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]      compile:  40,954.87 ms,  6.15 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]        image:  10,493.47 ms,  6.15 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]        write:   2,111.59 ms,  6.15 GB
[quarkus-sample-1.0.0-SNAPSHOT-runner:49336]      [total]:  97,207.01 ms,  6.15 GB

BUILD SUCCESSFUL in 1m 43s
```

CI에서 자주 빌드해야 하거나, 수정과 배포를 빠르게 반복해야 하는 환경에서는 이 부분이 병목이 될 수 있습니다. 머신 파워가 충분하거나 배포 시간 자체가 크게 중요하지 않다면 괜찮겠지만, 기동 속도만을 위해 빌드 시간이 크게 늘어난다면 결국 또 다른 형태의 트레이드오프가 됩니다. 이런 경우에는 Jar 빌드 시간, 전체 배포 파이프라인, 다른 프레임워크 대비 빌드부터 실행까지의 총 시간을 함께 보고 판단하는 편이 맞겠습니다.

### 피크 성능은 별도 이야기다

일반적으로는 C나 C++ 같은 네이티브 언어보다 Java 같은 JVM 언어가 느리다는 인식이 강합니다. 하지만 모든 상황에서 꼭 그렇지는 않습니다. 알고리즘이나 애플리케이션 설계 같은 언어 외적 요소도 크고, 언어 특성만 놓고 보더라도 상황에 따라 결과는 달라질 수 있습니다. 이유는 네이티브 언어와 JVM 언어의 컴파일 전략이 서로 다르기 때문입니다.

처음부터 머신 코드를 생성하는 언어가 유리한 것은 분명합니다. Java가 처음 등장했을 당시 성능 비판을 많이 받았던 것도 그런 이유였습니다. 지금은 Java가 성능이 중요한 서버 애플리케이션에도 자주 쓰이지만, 여기에 하드웨어 발전이 큰 역할을 했다는 점도 부정하기 어렵습니다.

다만 JVM 언어가 항상 Native보다 느리다고 단정할 수는 없습니다. 컴파일에는 [AOT](https://ja.wikipedia.org/wiki/%E4%BA%8B%E5%89%8D%E3%82%B3%E3%83%B3%E3%83%91%E3%82%A4%E3%83%A9)처럼 미리 전부 컴파일하는 방식만 있는 것이 아니라, [JIT](https://ja.wikipedia.org/wiki/%E5%AE%9F%E8%A1%8C%E6%99%82%E3%82%B3%E3%83%B3%E3%83%91%E3%82%A4%E3%83%A9)처럼 실행 중 필요한 부분을 최적화하는 방식도 있기 때문입니다.

JVM은 JIT를 통해 바이트코드를 분석하고, 자주 쓰이는 코드 경로를 더 효율적으로 최적화해 머신 코드로 바꿉니다. 이런 최적화가 잘 작동하면, 오히려 런타임 성능이 상당히 좋아질 수 있습니다. 물론 모든 코드가 다 JIT 대상이 되는 것은 아닙니다. 한 번만 실행되는 코드를 전부 머신 코드로 바꾸는 것은 오히려 낭비이기 때문에, 실제로는 실행 빈도에 따라 최적화 대상이 결정됩니다. Java 마이크로벤치마크에서 [JMH](https://github.com/openjdk/jmh)로 워밍업을 충분히 거치는 것도 바로 이런 JIT 특성을 반영한 방식입니다.

#### 직접 확인해 보면

그래서 Native와 JVM의 런타임 성능 차이가 실제로 어느 정도일지 궁금해서, 10만 건의 데이터를 루프로 생성해 반환하는 단순한 서비스를 만들어 처리 시간을 측정해 봤습니다. Controller의 반환값으로 [Multi](https://smallrye.io/smallrye-mutiny/getting-started/creating-multis)를 사용한 탓인지 요청마다 응답 시간 편차가 좀 있었기 때문에, 여기서는 엄밀한 API 응답 시간보다는 "for 루프로 데이터 생성에 걸린 시간"을 봤다고 이해하는 편이 맞습니다.

Native 빌드와 Jar 실행 모두 GraalVM CE 21.3.0(OpenJDK 11)을 사용했고, 처리 시간은 Kotlin의 [measureTimeMillis](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.system/measure-time-millis.html)로 측정한 값을 로그로 남겼습니다.

먼저 Native 실행 결과입니다.

```bash
2021-10-23 17:12:14,061 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-6) measured time was 89 ms
2021-10-23 17:12:15,630 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-1) measured time was 52 ms
2021-10-23 17:12:17,079 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-15) measured time was 106 ms
2021-10-23 17:12:18,174 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-5) measured time was 49 ms
2021-10-23 17:12:19,523 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-11) measured time was 50 ms
2021-10-23 17:12:20,468 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-4) measured time was 50 ms
2021-10-23 17:12:21,739 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-7) measured time was 124 ms
2021-10-23 17:12:23,113 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-12) measured time was 53 ms
2021-10-23 17:12:24,073 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-13) measured time was 49 ms
2021-10-23 17:12:25,308 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-2) measured time was 53 ms
```

다음은 JVM 실행 결과입니다.

```bash
2021-10-23 17:10:32,240 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-8) measured time was 163 ms
2021-10-23 17:10:35,057 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-6) measured time was 33 ms
2021-10-23 17:10:39,418 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-11) measured time was 40 ms
2021-10-23 17:10:42,211 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-3) measured time was 25 ms
2021-10-23 17:10:44,149 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-10) measured time was 38 ms
2021-10-23 17:10:46,283 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-2) measured time was 24 ms
2021-10-23 17:10:48,262 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-20) measured time was 22 ms
2021-10-23 17:10:49,854 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-12) measured time was 26 ms
2021-10-23 17:10:51,552 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-23) measured time was 23 ms
2021-10-23 17:10:52,967 INFO  [com.ret.dom.ser.MemberService] (vert.x-eventloop-thread-7) measured time was 51 ms
```

역시 JIT 영향 때문인지 JVM에서는 첫 실행이 느리고, 그 이후부터는 처리 속도가 크게 좋아지는 모습을 볼 수 있었습니다. GraalVM 쪽 Native 성능도 앞으로 좋아질 수 있겠지만, JVM 역시 같이 발전할 수 있기 때문에, 런타임의 피크 성능이 정말 중요하다면 JVM 모드도 여전히 충분히 검토할 만하다고 생각합니다.

## 마지막으로

사실 메모리 사용량도 더 엄밀하게 재 봐야겠지만, 이 부분은 이미 참고할 만한 [비교 글](https://medium.com/swlh/springboot-vs-quarkus-a-real-life-experiment-be70c021634e)이 있어서 여기서는 길게 다루지 않겠습니다. 요약하면 메모리 사용량은 힙을 포함해 Quarkus 쪽이 더 적은 편이지만, CPU 최대 사용량과 지연 시간 측면에서는 Spring Boot가 더 나은 결과를 보인 사례도 있었습니다. 다만 이 부분은 Quarkus가 상대적으로 역사가 짧다는 점도 감안해야 할 것 같습니다.

직접 만져 본 인상으로는, Quarkus는 확실히 `Kubernetes Native`라는 표현을 내세울 만한 프레임워크였습니다. Native 빌드를 하면 Jar보다 애플리케이션 크기 자체는 오히려 커질 수 있지만, 대신 JDK가 필요 없어지는 점이 꽤 큽니다. JDK 크기만 해도 AdoptOpenJDK 기준으로 대략 300MB 수준이기 때문에, 인스턴스 수가 늘어날수록 이 차이가 의미 있게 다가올 수 있습니다.

여기에 다양한 라이브러리와 조합하기 쉽고, Spring Security 같은 익숙한 자산을 활용할 수 있다는 점도 장점입니다. Spring 경험이 있는 개발자라면 금방 적응할 가능성이 높아서, 회사 입장에서도 다른 완전히 새로운 프레임워크를 도입하는 것보다 인력 수급 부담이 덜할 수 있겠다는 생각이 들었습니다.

Spring WebFlux나 Ktor도 충분히 매력적이지만, 또 다른 강한 선택지가 하나 더 생긴 셈이라 어떤 것을 고를지 더 고민되는 시대가 된 것 같습니다. 개인적으로는 [Rocket](https://rocket.rs/)도 한 번 만져 보고 싶은데, 과연 그때까지 시간이 날지는 모르겠습니다.
