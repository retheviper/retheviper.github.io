---
title: "Ktor를 써 본 첫인상"
date: 2021-07-18
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - kotlin
  - ktor
  - exposed
translationKey: "posts/ktor-first-impression"
---

서버 측 언어로 Kotlin은 점점 더 쓰이고 있지만, 웹 프레임워크로는 여전히 Spring이 가장 흔한 선택이라고 생각합니다. 회사 사정도 있겠지만, Java 엔지니어에게 익숙한 선택지라는 점과 검증된 생태계가 이미 충분히 쌓여 있다는 점이 크다고 봅니다. 아직도 Struts를 쓰다가 Spring으로 옮기려는 곳도 꽤 있습니다.

Kotlin은 Java와 호환되기 때문에 Java로 만든 애플리케이션을 Kotlin으로 옮기는 데 큰 장애는 없습니다. Java보다 생산성이 높고, Spring뿐 아니라 Jackson, Apache POI, JUnit, JaCoCo 같은 라이브러리도 그대로 쓸 수 있다는 점은 분명한 장점입니다. 그래서 기업 입장에서도 Kotlin 도입을 검토할 이유가 충분합니다. Java 개발자가 많다는 점도 도입을 쉽게 만드는 요소입니다.

다만 Kotlin을 제대로 활용하려면, 장기적으로는 Kotlin으로 작성된 라이브러리와 프레임워크를 함께 쓰는 편이 좋습니다. 컴파일 결과물은 Java와 호환되더라도, 소스 언어가 다르면 클라이언트 코드에서 불편한 지점이 생기거나 Kotlin에 맞지 않는 부분이 남을 수 있기 때문입니다. 또 Kotlin은 JVM뿐 아니라 Native로도 컴파일할 수 있으니, Native 앱을 고려한다면 Java 독립적인 API를 고르는 쪽이 낫습니다.

그래서 이번에는 JetBrains의 웹 프레임워크인 Ktor와, 함께 사용할 수 있는 ORM Exposed를 조금 써 보면서 Spring과 비교해 보려 합니다.

## Ktor

Ktor는 JetBrains가 만든 마이크로서비스용 경량 웹 프레임워크입니다. 공식 사이트 소개를 보면 장점이 여러 가지 있지만, 그중에서도 Spring과 비교했을 때 눈에 띄는 특징은 다음과 같습니다.

### 경량

Spring도 경량이라고 하지만, 실제로 써 보면 기동 속도는 꽤 무겁게 느껴질 수 있습니다. 초기화 시점에 여러 설정과 DI, 어노테이션 처리를 한꺼번에 하기 때문이라고 생각합니다. 그래서 시작 시간을 줄이려고 `late init`을 쓰는 식의 테크닉도 자주 보입니다.

반면 Ktor는 시작이 꽤 빠릅니다. 같은 규모의 앱을 직접 같은 조건에서 엄밀히 비교한 것은 아니지만, 체감 속도는 확실히 빠릅니다. 예를 들어 H2와 기본 CRUD만 넣은 Spring WebFlux 애플리케이션은 로컬에서 2.7초 정도 걸렸습니다.

```bash
2021-07-18 15:08:25.150  INFO 29047 --- [main] c.r.s.SpringWebfluxSampleApplicationKt   : Started SpringWebfluxSampleApplicationKt in 2.754 seconds (JVM running for 3.088)
```

같은 구성의 Ktor 애플리케이션은 1초도 걸리지 않았습니다.

```bash
2021-07-18 15:09:29.683 [main] INFO  ktor.application - Application started in 0.747 seconds.
```

이 차이는 기본적으로 DI에 덜 의존하고, 어노테이션이나 Reflection을 많이 쓰지 않으며, REST API에 필요한 최소한의 기능에 집중한 구조 덕분이라고 생각합니다.

시작 시간이 빠르다는 것은 테스트 시간을 줄인다는 점에서도 유리하고, 서버리스 애플리케이션에도 잘 맞습니다. 저는 AWS Lambda나 Azure Functions를 써 본 적이 있는데, 이런 환경에서는 요청이 올 때마다 앱을 띄워야 하므로 기동 속도가 중요합니다. Spring은 이 점에서 부담이 컸지만, Ktor는 훨씬 현실적인 선택지에 가깝다고 느꼈습니다.

### 확장 가능

Ktor의 경량성은 곧 필요한 기능은 직접 붙이거나 플러그인 형태로 추가해야 한다는 뜻이기도 합니다. 예를 들면 이런 식입니다.

```kotlin
fun main() {
    embeddedServer(Netty, port = 8080, host = "127.0.0.1") {
        install(CORS)
        install(Compression)
        install(Sessions) {
            cookie<MyCookie>("MY_COOKIE")
        }
        install(ContentNegotiation) { // kotlinx.serialization
            json()
        }
    }.start(wait = true)
}
```

이런 구조는 도입 초반에는 다소 번거롭게 느껴질 수 있습니다. 특히 Spring Security에서 기본 제공하던 기능들, 예를 들어 이전 글에서 다뤘던 Role-Based Authorization 같은 것들이 공식 플러그인으로 바로 제공되지 않는 경우에는 직접 구현해야 합니다. 처음에는 손이 더 가지만, 익숙해지면 오히려 구조를 잘 통제할 수 있다는 장점도 있습니다.

Ktor는 기본적으로 DI를 제공하지 않으므로, 필요하다면 [Injekt](https://github.com/IVIanuu/injekt), [Kodein](https://kodein.org/Kodein-DI/?6.3/ktor), [Koin](https://insert-koin.io) 같은 라이브러리를 추가해야 합니다. 하지만 애플리케이션 구조에 따라서는 DI가 꼭 필요하지 않을 수도 있고, `object`로 충분한 경우도 있습니다. 결국 어떤 구조로 갈지는 프로젝트 성격에 맞게 판단해야 합니다.

### Coroutine 대응

Spring WebFlux도 그렇지만, 요즘 웹 프레임워크는 대부분 비동기와 논블로킹을 적극적으로 지원합니다. 인프라가 좋아졌다고 해도 소프트웨어 차원에서 성능을 더 끌어올릴 수 있다면 충분히 의미가 있습니다.

Ktor는 라우팅에서 [Route](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/-route/index.html)의 [route](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/route.html), [get](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/get.html), [post](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/post.html), [put](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/put.html), [delete](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/delete.html) 같은 함수를 사용합니다. 느낌은 Spring WebFlux의 Router/Handler Function과 꽤 비슷합니다.

```kotlin
routing {
    get("/hello") {
        call.respondText("Hello")
    }
}
```

또한 각 HTTP 메서드의 body는 기본적으로 `suspend`로 작성하게 됩니다. 구현하는 쪽에서 비동기를 따로 강하게 의식하지 않아도 된다는 점은 꽤 편합니다. Spring WebFlux도 Coroutine으로 쉽게 만들 수 있지만, `suspend`까지 자연스럽게 녹아 있다는 점은 Ktor만의 장점이라고 생각합니다.

### 테스트

`ktor-server-test-host`, `kotlin-test`, JUnit 등을 이용해 테스트할 수 있습니다. Spring도 테스트 방법이 다양하지만, 기본적으로 테스트의 형태 자체가 크게 달라지는 것은 아닙니다. 예를 들어 `GET` 응답을 검증하려면 다음처럼 쓸 수 있습니다.

```kotlin
@Test
fun getMember() {
    withTestApplication(Application::module) {
        handleRequest(HttpMethod.Get, "api/v1/web/members/$id").apply {
            assertEquals(
                actual = response.status(),
                expected = HttpStatusCode.OK
            )
            assertEquals(
                actual = response.content,
                expected = Json.encodeToString(
                    MemberResponse(
                        id = id,
                        userId = userId,
                        name = name
                    )
                ),
            )
        }
    }
}
```

## Exposed

Ktor와 함께 쓸 수 있는 Kotlin 기반 ORM으로는 대표적으로 [Exposed](https://github.com/JetBrains/Exposed)가 있습니다. [jOOQ](https://www.jooq.org)처럼 SQL DSL을 써서 쿼리를 코드처럼 작성할 수 있다는 점이 좋습니다. 실제로는 DSL을 해석해 SQL을 만들어 주지만, 작성 감각은 훨씬 직접적입니다.

예를 들어 `User` 테이블에서 레코드를 찾는 코드는 다음과 같습니다.

```kotlin
val userInUsa: List<User> = transaction {
    UserTable.select {
        UserTable.deleted eq false
    }.map {
        User(
            id = it[UserTable.id],
            name = it[UserTable.name],
            country = it[UserTable.country]
        )
    }.filter {
        it.country = Country.USA
    }
}
```

Exposed는 DAO 패턴도 지원하므로, 이런 식으로 더 단순하게 쓸 수도 있습니다. JPA나 R2DBC와 비슷한 감각입니다.

```kotlin
val userInGermany: List<User>  = transaction {
    User.find { (UserTable.country eq Country.GERMANY) and (UserTable.deleted eq false)}
}
```

또 하나의 특징은 테이블 정의를 코드로 작성해서 DB 스키마에 반영할 수 있다는 점입니다. 지금까지는 Liquibase나 Flyway처럼 별도 도구로 형상 관리를 하는 경우가 많았지만, 애플리케이션 코드 안에서 테이블 정의를 함께 관리하면 실제 DB와 정의가 어긋날 가능성을 줄일 수 있습니다. 특히 테이블 변경이 잦거나 마이크로서비스가 많은 환경에서는 꽤 편리합니다.

Exposed의 테이블 정의는 다음처럼 작성할 수 있습니다.

```kotlin
object Member : IntIdTable() {
    val userId: Column<String> = varchar(name = "user_id", length = 16)
    val name: Column<String> = varchar(name = "name", length = 16)
    val password: Column<String> = varchar(name = "password", length = 255)
    val deleted: Column<Boolean> = bool("deleted")
    val createdBy: Column<String> = varchar("created_by", 16)
    val createdDate: Column<LocalDateTime> = datetime("created_date")
    val lastModifiedBy: Column<String> = varchar("last_modified_by", 16)
    val lastModifiedDate: Column<LocalDateTime> = datetime("last_modified_date")
}
```

실제로 생성되는 SQL은 다음과 같습니다.

```sql
CREATE TABLE IF NOT EXISTS "MEMBER" (ID INT AUTO_INCREMENT PRIMARY KEY, DELETED BOOLEAN NOT NULL, CREATED_BY VARCHAR(16) NOT NULL, CREATED_DATE DATETIME(9) NOT NULL, LAST_MODIFIED_BY VARCHAR(16) NOT NULL, LAST_MODIFIED_DATE DATETIME(9) NOT NULL, USER_ID VARCHAR(16) NOT NULL, "NAME" VARCHAR(16) NOT NULL, PASSWORD VARCHAR(255) NOT NULL)
```

JPA나 R2DBC에서 Auditable 클래스를 두고 상속하는 방식처럼, Exposed에서도 비슷한 구조를 만들 수 있습니다.

```kotlin
abstract class Audit : IntIdTable() {
    val deleted: Column<Boolean> = bool("deleted")
    val createdBy: Column<String> = varchar("created_by", 16)
    val createdDate: Column<LocalDateTime> = datetime("created_date")
    val lastModifiedBy: Column<String> = varchar("last_modified_by", 16)
    val lastModifiedDate: Column<LocalDateTime> = datetime("last_modified_date")
}

object Member : Audit() { // Audit 컬럼도 포함해 테이블이 생성된다
    val userId: Column<String> = varchar(name = "user_id", length = 16)
    val name: Column<String> = varchar(name = "name", length = 16)
    val password: Column<String> = varchar(name = "password", length = 255)
}
```

MyBatis에 익숙한 사람이라면 처음엔 조금 낯설 수 있지만, 결국 SQL을 Kotlin 코드로 쓰는 감각에 가깝습니다. 익숙해지면 꽤 편하게 사용할 수 있습니다.

## 마지막으로

간단한 CRUD 앱을 Ktor와 Exposed로 만들어 본 뒤의 인상을 정리해 봤습니다. 전체적으로는 코드가 꽤 간결하고, 성능도 좋아서 마이크로서비스와 잘 맞는 구성이란 느낌이 강했습니다. 순수 Kotlin 기반 프레임워크라는 점도 분명한 장점입니다.

아직 Spring이나 다른 Java 계열 라이브러리만큼 생태계가 넓게 검증됐다고 보기는 어렵지만, Exposed 외에도 [Ktorm](https://www.ktorm.org) 같은 선택지가 있고 IntelliJ의 Ktor 지원도 좋아서 앞으로 더 발전할 여지는 충분해 보입니다. 업무에 바로 쓰기엔 검토할 부분이 남아 있어도, 개인 프로젝트에서는 적극적으로 시도해 볼 만합니다.

Kotlin으로 할 수 있는 일이 점점 늘어나고 있어서 반갑습니다.
