---
title: "Ktor에서 Role-based Authorization 구현"
date: 2021-08-09
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - kotlin
  - ktor
  - authorization
translationKey: "posts/ktor-role-based-authorization"
---

지난 글에서 Ktor를 소개하면서, Ktor에는 아직 `Role-based Authorization`을 바로 제공하는 기능이 없어서 직접 구현해야 한다고 이야기했습니다. Ktor는 아직 역사가 길지 않아서 Spring이나 Django, Rails처럼 기능이 풍부한 프레임워크와 비교하면 부족한 부분이 있을 수 있습니다. 필요한 기능이 없다면 직접 만들어야 한다는 뜻이기도 합니다.

다행히 Ktor는 기능을 [Plugin](https://ktor.io/docs/plugins.html)이라는 모듈 형태로 추가할 수 있습니다. 그래서 필요한 기능을 구현해 두면, 그것을 하나의 Plugin으로 붙여서 사용할 수 있습니다. 다만 모듈 방식은 유연한 만큼, 내부 구조를 이해하고 그에 맞는 작성법을 알아야 합니다.

이번 글에서는 참고한 [글](https://medium.com/@shrikantjagtap99/role-based-authorization-feature-in-ktor-web-framework-in-kotlin-dda88262a86a)을 바탕으로, Ktor에서 Role-based Authorization을 직접 구현하는 흐름을 정리해 보겠습니다.

## Role-based Authorization이란

먼저 Role-based Authorization이 무엇인지부터 정리하겠습니다. 이는 웹 애플리케이션에서 흔히 쓰는 권한 제어 방식 중 하나로, 사용자의 `Role`을 기준으로 API 접근을 제한하는 방식입니다. 예를 들어 EC 사이트에서는 상품 문의는 인증된 사용자라면 누구나 할 수 있지만, 공지를 작성하거나 상품 재고를 변경하는 기능은 관리자에게만 허용해야 합니다.

즉, "일반 사용자"와 "관리자"처럼 역할을 나누고, 요청이 들어올 때 그 역할을 먼저 확인한 뒤 허용된 사용자만 API를 실행하게 만드는 구조입니다. 이를 위해 필요한 것은 크게 두 가지입니다. 하나는 역할 자체를 관리하는 개념이고, 다른 하나는 요청을 가로채서 그 역할을 검사하는 기구입니다.

전자의 경우는 사용자에게 어떤 역할이 붙는지 정하면 되므로 비교적 단순합니다. 테이블이나 컬럼을 추가해 기존 사용자 정보와 연결하면 됩니다. 반면 후자는 프레임워크가 요청을 어떻게 처리하는지부터 이해해야 하므로, 먼저 Ktor의 요청 처리 구조를 살펴보는 편이 좋습니다.

## Pipeline과 Feature

Ktor에서 가장 중요한 개념 중 하나는 [Pipeline](https://ktor.io/docs/pipelines.html)입니다. 공식 설명은 다소 길지만, 쉽게 말하면 요청이 들어와 응답이 나갈 때까지의 처리 흐름을 단계별로 구성한 구조입니다. Ktor의 `Router`도 이런 Pipeline 위에서 동작합니다.

```kotlin
fun Application.main() {
    install(ContentNegotiation) {
        json()
    }
}
```

위에서 호출한 `install`의 시그니처를 보면 다음과 같습니다. `feature`와 설정용 `configure`가 인수로 들어갑니다.

```kotlin
public fun <P : Pipeline<*, ApplicationCall>, B : Any, F : Any> P.install(
    feature: ApplicationFeature<P, B, F>,
    configure: B.() -> Unit = {}
): F
```

이를 보면 `ContentNegotiation` 같은 기능이 `ApplicationFeature`로 구현되어 있고, `install`을 통해 애플리케이션에 붙는다는 점을 알 수 있습니다. 실제 `ContentNegotiation` 구현도 `Configuration` 클래스와 `ApplicationFeature`를 상속한 `companion object`를 포함하고 있습니다.

```kotlin
public class ContentNegotiation internal constructor(
    public val registrations: List<ConverterRegistration>,
    private val acceptContributors: List<AcceptHeaderContributor>,
    private val checkAcceptHeaderCompliance: Boolean = false
) {

    // ...

    /**
     * Configuration type for [ContentNegotiation] feature
     */
    public class Configuration {
        // ...
    }

    /**
     * Implementation of an [ApplicationFeature] for the [ContentNegotiation]
     */
    public companion object Feature : ApplicationFeature<ApplicationCallPipeline, Configuration, ContentNegotiation> {
        // ...
    }
```

정리하면, Ktor에서 새로운 기능을 만들려면 설정을 담는 `Configuration`과, 실제 동작을 담당하는 `ApplicationFeature` 구조를 맞춰야 합니다. 이 구조를 이해하면 자작 모듈을 애플리케이션에 자연스럽게 붙일 수 있습니다.

## Plugin 구현

이제 실제로 권한 검사 기능을 만든다고 해보겠습니다. 먼저 역할부터 정의하겠습니다. 예시는 단순하게 관리자와 일반 사용자 두 가지로 두겠습니다.

```kotlin
enum class Role(val value: String) {
    ADMIN("ADMIN"),
    USER("USER")
}
```

이 역할 정보는 테이블에서 관리할 수도 있지만, 실제 권한 판단에는 로그인한 사용자 정보가 필요합니다. 그래서 `io.ktor.auth.Principal`을 상속한 사용자 정보를 별도의 클래스로 두고, 로그인에 성공했을 때 여기에 역할을 담아 두는 식으로 구성할 수 있습니다.

```kotlin
data class UserPrincipal(
    val username: String,
    val roles: Set<Role> = emptySet()
) : Principal
```

다음으로는 역할을 기준으로 접근을 제한하는 함수를 만듭니다. 라우팅 블록 안에서 특정 역할만 접근할 수 있는 엔드포인트를 감싸는 구조를 생각하면 됩니다.

```kotlin
fun Application.main() {
    routing {
        // 관리자만 접근할 수 있다
        withRole(Role.ADMIN) {
            get("/admin") {
                call.respond("This is admin page")
            }
        }

        // 일반 사용자가 접근할 수 있다
        withRole(Role.USER) {
            get("/user") {
                call.respond("This is user page")
            }
        }
    }
}
```

Ktor의 코드 블록은 중첩이 가능하므로 이런 구조를 만들 수 있습니다. `withRole` 안에서 역할을 검사하고, 조건에 맞을 때만 내부 라우트를 실행하면 됩니다.

### AuthorizedRouteSelector 구현

먼저 `RouteSelector`를 구현합니다. 이 선택자는 `routing` 안에서 나중에 만들 함수가 자연스럽게 중첩되도록 하기 위한 장치입니다.

```kotlin
class AuthorizedRouteSelector() : RouteSelector() {
    override fun evaluate(context: RoutingResolveContext, segmentIndex: Int) = RouteSelectorEvaluation.Constant
}
```

### child route 구현

이제 `AuthorizedRouteSelector`를 사용해 실제 `child route`처럼 동작하는 함수를 만들겠습니다. `routing` 아래에 중첩되므로 `Route`의 확장 함수로 정의하면 됩니다. 인수로는 판정을 위한 역할과 내부에 둘 라우트 블록을 받도록 하면 됩니다.

```kotlin
fun Route.withRole(val role: Role, build: Route.() -> Unit): Route {
    val authorizedRoute = createChild(AuthorizedRouteSelector())
    application.feature(RoleBaseAuthorizer).interceptPipeline(this, role)
    build()
    return authorizedRoute
}
```

이 함수는 먼저 `AuthorizedRouteSelector`로 child route를 만들고, 이어서 Pipeline을 가로채 사용자 역할을 검사합니다. 문제가 없으면 `build()`를 실행하고, 마지막에 생성한 child route를 반환합니다.

여기서 사용하는 `RoleBaseAuthorizer`는 별도 Feature로 만들어 두는 편이 좋습니다.

### 권한 처리 모듈 구현

이제 본격적으로 권한 검사 모듈을 구현하겠습니다. 앞에서 말한 것처럼 `Configuration`과 `Feature`를 함께 갖는 클래스를 만들면 됩니다. 여기서 `Configuration`은 로그인한 사용자로부터 어떻게 역할 정보를 얻을지 정하는 곳입니다.

```kotlin
fun Application.main() {
    // RoleBaseAuthorizer를 Feature로 설치한다
    install(RoleBaseAuthorizer) {
        // 로그인한 사용자의 정보에서 롤을 가져오는 방법을 Configuration으로 지정한다
        getRoles { (it as UserPrincipal).roles }
    }
}
```

`Configuration`에서는 사용자 정보를 받아 역할 집합을 돌려주는 함수를 넘깁니다. 그러면 `RoleBaseAuthorizer`는 `withRole`에서 지정한 역할과 현재 사용자의 역할을 비교할 수 있습니다.

이제 실제 구현을 보겠습니다.

```kotlin
class RoleBaseAuthorizer(config: Configuration) {

    class Configuration {
        var userRoles: (Principal) -> Set<Role> = { emptySet() }

        // 로그인한 사용자의 정보에서 롤을 가져오는 방법을 설정한다
        fun getRoles(roles: (Principal) -> Set<Role>) {
            userRoles = roles
        }
    }

    private val getRoles = config.userRoles

    fun interceptPipeline(
        pipeline: ApplicationCallPipeline,
        role: Role
    ) {
        // Pipeline 위치를 지정한다
        pipeline.insertPhaseAfter(ApplicationCallPipeline.Features, Authentication.ChallengePhase)
        pipeline.insertPhaseAfter(Authentication.ChallengePhase, AuthorizationPhase)

        // 인터셉트 시 처리
        pipeline.intercept(AuthorizationPhase) {
            // 로그인한 사용자의 정보를 가져온다
            val principal = call.authentication.principal<UserPrincipal>()
                ?: throw AuthorizationException("Missing principal")

            // 사용자 정보에서 롤을 가져온다
            val roles = getRoles(principal)

            if (roles.none { role }) {
                // 로그인한 사용자의 롤에 접근 가능한 롤이 없을 때의 처리
            }
        }
    }

    companion object Feature : ApplicationFeature<ApplicationCallPipeline, Configuration, RoleBaseAuthorizer> {

        override val key = AttributeKey<RoleBaseAuthorizer>("RoleBaseAuthorizer")

        val AuthorizationPhase = PipelinePhase("Authorization")

        override fun install(
            pipeline: ApplicationCallPipeline,
            configure: Configuration.() -> Unit
        ): RoleBasedAuthorization {
            val configuration = Configuration().apply(configure)
            return RoleBaseAuthorizer(configuration)
        }
    }
}
```

위 구현에서 `Configuration`은 사용자 역할을 얻는 함수를 저장하고, `interceptPipeline`은 그 함수를 이용해 실제 권한을 검사합니다. 또 `interceptPipeline`에서는 Pipeline의 위치도 정해야 하는데, 위 코드에서는 인증 이후에 권한 검사가 들어가도록 배치했습니다.

`Feature`는 이 모듈을 Ktor의 독립적인 기능으로 설치할 수 있게 해 줍니다. 다시 말해, 설정을 받아 인스턴스를 돌려주는 설치용 껍데기라고 보면 됩니다.

이제 권한 처리 모듈 자체는 거의 끝났습니다. 다만 `interceptPipeline`에서 사용자의 역할이 요구 역할과 맞지 않을 때의 처리 방식은 몇 가지 선택지가 있습니다.

#### 응답을 반환하고 종료

가장 단순한 방법은 적절한 응답을 반환한 뒤 Pipeline을 종료하는 것입니다.

```kotlin
if (roles.none { role }) {
    // 응답을 반환한다
    call.respond(
        status = HttpStatusCode.Forbidden,
        message = "permission not granted: $role"
    )

    // Pipeline 종료
    finish()
}
```

여기서 중요한 점은 응답을 보냈다고 해서 Pipeline이 자동으로 멈추지는 않는다는 것입니다. 처리를 완전히 중단하려면 `finish()`를 호출해야 합니다.

#### Exception 던지기

다른 방법은 예외를 던져서 처리 흐름을 끊는 방식입니다.

```kotlin
// 인가되지 않은 경우의 예외
class AuthorizationException(override val message: String) : Exception(message)

if (roles.none { role }) {
    throw AuthorizationException("permission not granted: $role")
}
```

예외를 던지면 Pipeline은 바로 멈추지만, 로그에는 예외로 남습니다. 그래서 Ktor의 `StatusPages` 같은 기능을 함께 써서 적절히 처리하는 편이 좋습니다.

```kotlin
// 인가되지 않은 경우의 응답
data class AuthFailedResponse(val reason: String)

// 예외 처리
install(StatusPages) {
    exception<Throwable> { cause ->
        when (cause) {
            // 인가된 경우의 처리
            is AuthorizationException -> {
                call.respond(
                    status = HttpStatusCode.Forbidden,
                    message = AuthFailedResponse(
                        reason = cause.message
                    )
                )
            }
        }
    }
}
```

이렇게 하면 애플리케이션 로그도 깔끔하게 유지할 수 있고, 다른 예외 처리도 `when` 분기를 추가하는 식으로 쉽게 확장할 수 있습니다.

## 마지막으로

생각보다 방대한 내용을 다루게 되었지만, Pipeline을 이용해 자작 모듈을 붙이는 흐름은 어느 정도 정리할 수 있었던 것 같습니다. 이 구조를 이해해 두면 다른 모듈을 추가하는 것도 생각보다 어렵지 않습니다. 더 깊게 파고들면 또 다른 흥미로운 이야기들이 나오겠지만, 그건 다음 기회로 넘기겠습니다.

개인적으로는 Role-based Authorization을 직접 만들면서, 요청 흐름 전체를 하나의 Pipeline으로 다룬다는 개념이 꽤 인상적이었습니다. Spring에서도 인터셉트는 할 수 있지만, 흐름 자체를 단위로 바라보는 느낌은 Ktor 쪽이 더 명확합니다. 아직 Ktor를 막 써 보기 시작한 단계라 더 공부할 부분은 많지만, 적어도 프레임워크 자체는 상당히 매력적이라고 느꼈습니다.

처음에는 Spring 같은 익숙한 프레임워크에 비해 기능이 부족해 보였지만, 이렇게 직접 모듈을 만들 수 있다면 생각보다 충분히 대응할 수 있을지도 모릅니다. 물론 프로덕션에 적용하려면 더 많은 검증이 필요하겠지만요.
