---
title: "Ktor의 요청 처리 개선"
date: 2025-06-28
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - ktor
  - kotlin
translationKey: "posts/ktor-validation"
---

Kotlin은 Java와 호환되기 때문에, 기존 Java 시스템의 일부를 Kotlin으로 옮기거나 같은 JVM 위에서 새 시스템을 Kotlin으로 구축하는 경우가 많습니다. 다만 같은 JVM 생태계 안에서도 프레임워크와 라이브러리 선택에 따라 더 단순하게 만들 수 있는 부분이 있고, 경우에 따라서는 성능 개선도 기대할 수 있습니다.

Java 쪽 프레임워크와 라이브러리는 오랜 시간 동안 매우 다양한 기능을 제공해 왔지만, Kotlin에서는 더 간결하고 직관적인 코드를 작성할 수 있는 경우가 많습니다. 그리고 그 과정에서 Kotlin의 특성에 맞는 설계를 택하거나, 불필요한 비용을 줄이는 방향으로 개선할 여지도 있습니다.

이번 글에서는 그런 개선 사례 중 하나로, Ktor에서 요청 유효성 검사를 구현하는 방식을 어떻게 정리할 수 있었는지 소개해 보겠습니다.

## 기존 방식

실무에서는 [Ktor](https://ktor.io), [Exposed](https://www.jetbrains.com/ko-kr/exposed) 같은 Kotlin용 프레임워크를 사용하고 있습니다. 다만 일부 영역은 여전히 Java 라이브러리에 의존하고 있었고, 직렬화에는 Jackson, 요청 유효성 검사에는 `jakarta.validation`을 사용하고 있었습니다.

Spring Boot처럼 Java/Kotlin 양쪽에서 널리 쓰이는 프레임워크에서는 이런 라이브러리를 붙이기가 꽤 쉽습니다. 예를 들어 요청 검증은 `@Valid`, `@Validated` 같은 어노테이션만으로 자동 처리할 수 있습니다.

하지만 Ktor는 사정이 다릅니다. 기본적으로는 요청 유효성 검사를 직접 호출해야 했고, 실제 코드도 대략 다음과 같은 형태였습니다.

```kotlin
// 요청 유효성 검사 수행
object AnnotationValidator {
    private val validator: Validator = Validation.buildDefaultValidatorFactory().validator

    fun getViolationMessages(model: Any?): List<String> {
        model ?: return emptyList()
        val violations = validator.validate(model)
        return violations.map { "${it.propertyPath} ${it.message}" }
    }
}

// 유효성 검사용 인터페이스
interface Validatable(
    fun validate(): List<String>
)

// 요청 바디
data class CreateLogRequest(
    @field:NotNull
    @field:NotBlank
    @field:Size(max = 36)
    @field:Pattern(regexp = "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[1-5][0-9a-fA-F]{3}-[89abAB][0-9a-fA-F]{3}-[0-9a-fA-F]{12}$")
    @JsonProperty("user_id")
    val userId: String?,

    @field:NotNull
    @field:NotBlank
    @field:Size(max = 2048)
    @JsonProperty("referral")
    val referral: String?,

    @field:NotNull
    @field:NotBlank
    @field:Size(max = 2048)
    @JsonProperty("user_agent")
    val userAgent: String?
) : Validatable {
    override fun validate(): List<String> {
        return AnnotationValidator.getViolationMessages(this)
    }
}

// 컨트롤러
class LogController : KoinComponent {
    private val logUseCase: sLogUseCase by inject()

    fun createLog(request: CreateLogRequest): CreateLogResponse {
        val validationErrors = request.validate()
        if (validationErrors.isNotEmpty()) {
            throw BadRequestException("Validation failed: ${validationErrors.joinToString(", ")}")
        }

        return logUseCase.createLog(request)
    }
}
```

이 방식에서 불편하다고 느낀 점은 다음과 같았습니다.

1. 요청 검증을 수동으로 호출해야 해서 코드가 쉽게 장황해진다.
2. 컨트롤러와 요청마다 비슷한 검증 코드를 반복해서 작성하게 된다.
3. 검증과 직렬화가 어노테이션 중심이라 리플렉션 비용이 생기고, 요구 사항이 바뀌면 수정 지점도 많아진다.
4. 타입보다 어노테이션에 많이 의존하게 되어 Kotlin다운 타입 안전성을 살리기 어렵다.

그래서 이번에는 이 문제를 줄이기 위해 Ktor 쪽 요청 처리 방식을 조금 정리해 보기로 했습니다.

## 개선 방향

이번에 적용한 방향은 크게 세 가지였습니다.

1. Ktor 공식 [Request Validation](https://ktor.io/docs/request-validation.html)을 사용해 어노테이션 의존도를 줄인다.
2. 검증 로직을 공통화할 수 있게 구조를 정리한다.
3. Jackson에서 [Kotlinx Serialization](https://kotlinlang.org/docs/serialization.html)로 옮겨 어노테이션을 더 줄이고 성능도 개선해 본다.

아래부터는 각각을 어떻게 적용했는지 순서대로 보겠습니다.

### Ktor Request Validation 사용

먼저 `ValidatableController`라는 인터페이스를 만들고, 각 컨트롤러가 자신이 담당하는 요청 검증 로직을 등록하도록 했습니다. 이 인터페이스는 Ktor의 Request Validation 플러그인에 검증 규칙을 연결하기 위한 용도입니다.

```kotlin
interface ValidatableController {
    fun configureValidation(validationConfig: RequestValidationConfig)
}
```

그다음 요청 데이터 클래스를 정리합니다. `jakarta.validation` 어노테이션을 제거하고, 기존 `Validatable` 인터페이스도 더 이상 사용하지 않도록 바꿨습니다.

```kotlin
data class CreateLogRequest(
    @JsonProperty("user_id")
    val userId: String?,

    @JsonProperty("referral")
    val referral: String?,

    @JsonProperty("user_agent")
    val userAgent: String?
)
```

이제 컨트롤러에서 `ValidatableController`를 구현하고, 검증 로직을 `configureValidation` 안으로 옮깁니다.

```kotlin
class LogController : KoinComponent, ValidatableController {
    private val logUseCase: LogUseCase by inject()

    override fun configureValidation(validationConfig: RequestValidationConfig) {
        validationConfig.validate<CreateLogRequest> { request ->
            when {
                request.userId == null -> ValidationResult.Invalid("User ID must not be null")
                request.userId.isBlank() -> ValidationResult.Invalid("User ID must not be blank")
                request.userId.length > 36 -> ValidationResult.Invalid("User ID must not exceed 36 characters")
                !request.userId.matches(Regex("^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[1-5][0-9a-fA-F]{3}-[89abAB][0-9a-fA-F]{3}-[0-9a-fA-F]{12}$")) ->
                    ValidationResult.Invalid("User ID must be a valid UUID format")

                request.referral == null -> ValidationResult.Invalid("Referral must not be null")
                request.referral.isBlank() -> ValidationResult.Invalid("Referral must not be blank")
                request.referral.length > 2048 -> ValidationResult.Invalid("Referral must not exceed 2048 characters")

                request.userAgent == null -> ValidationResult.Invalid("User Agent must not be null")
                request.userAgent.isBlank() -> ValidationResult.Invalid("User Agent must not be blank")
                request.userAgent.length > 2048 -> ValidationResult.Invalid("User Agent must not exceed 2048 characters")

                else -> ValidationResult.Valid
            }
        }
    }

    fun createLog(request: CreateLogRequest): CreateLogResponse {
        return logUseCase.createLog(request)
    }
}
```

이렇게 바꾸면 Ktor의 Request Validation을 통해 요청 검증을 처리할 수 있고, 개별 컨트롤러에서 수동 검증 코드를 직접 호출할 필요도 줄어듭니다.

### 모듈 설정

다음 단계는 `ValidatableController` 구현체들을 한 번에 모아서 Ktor 모듈에 등록하는 것입니다. 이 구조를 인터페이스 중심으로 만든 이유는, DI에 사용 중인 [Koin](https://insert-koin.io/)으로 모든 컨트롤러를 가져와 Request Validation에 연결하기 위해서입니다.

Koin에서 인터페이스와 구현 클래스를 연결하는 방법은 몇 가지가 있습니다. 예를 들어 아래처럼 하나만 등록할 수 있습니다.

```kotlin
val koinModule = module {
    single<ValidatableController> { AccessLogController() }
}
```

하지만 이렇게 하면 `ValidatableController`를 하나만 다루게 되므로, 여러 컨트롤러의 검증 로직을 한꺼번에 모으기에는 적합하지 않습니다.

여기서는 [bind](https://insert-koin.io/docs/reference/koin-core/definitions/#additional-type-binding)를 사용하는 방식이 더 잘 맞았습니다.

```kotlin
val koinModule = module {
    single { LogController() } bind ValidatableController::class
    single { AnotherController() } bind ValidatableController::class
}
```

이렇게 등록해 두면 Koin이 `ValidatableController` 구현체 전체를 가져올 수 있습니다. 그리고 아래처럼 Request Validation 플러그인에 모두 연결할 수 있습니다.

```kotlin
fun Application.configureRequestValidation() {
    val validatableControllers: List<ValidatableController> = getKoin().getAll()

    install(RequestValidation) {
        validatableControllers.forEach { controller ->
            controller.configureValidation(this)
        }
    }
}
```

이 구조의 장점은 새 컨트롤러를 추가할 때도 `ValidatableController`만 구현하면 자동으로 검증 등록 흐름에 편입된다는 점입니다.

### Kotlinx Serialization 사용

마지막으로 직렬화는 Kotlinx Serialization으로 옮겼습니다. 목적은 어노테이션 사용을 더 줄이고, 가능하면 성능도 개선해 보려는 것이었습니다.

Kotlinx Serialization은 KSP(Kotlin Symbol Processing)를 활용해 컴파일 시점에 직렬화 코드를 생성합니다. 그 덕분에 리플렉션 없이 동작할 수 있고, 코드 구조도 조금 더 타입 중심으로 정리하기 쉬워집니다.

우선 기존 Jackson 설정을 Kotlinx Serialization 기반으로 바꿉니다. `ContentNegotiation` 플러그인에 JSON 설정을 추가하고, 필요하면 `JsonNamingStrategy.SnakeCase`로 스네이크 케이스 규칙도 적용할 수 있습니다.

```kotlin
fun Application.configureContentNegotiation() {
    install(ContentNegotiation) {
        json(
            Json {
                namingStrategy = JsonNamingStrategy.SnakeCase
            }
        )
    }
}
```

그다음 요청 데이터 클래스에 `@Serializable`을 붙이고, 가능한 한 타입 자체로 제약을 표현합니다. 예를 들어 `userId`를 `String`이 아니라 `UUID`로 바꾸면, 별도 정규식 검증 없이도 타입 수준에서 제약을 줄 수 있습니다.

```kotlin
@Serializable
data class CreateLogRequest(
    val userId: UUID,
    val referral: String,
    val userAgent: String
)
```

이제 컨트롤러의 검증 로직도 조금 더 단순해집니다. `userId`가 이미 `UUID` 타입이므로 정규식 검사는 필요 없습니다.

```kotlin
class LogController : KoinComponent, ValidatableController {
    private val logUseCase: LogUseCase by inject()

    override fun configureValidation(validationConfig: RequestValidationConfig) {
        validationConfig.validate<CreateLogRequest> { request ->
            when {
                request.referral.isBlank() -> ValidationResult.Invalid("Referral must not be blank")
                request.referral.length > 2048 -> ValidationResult.Invalid("Referral must not exceed 2048 characters")
                request.userAgent.isBlank() -> ValidationResult.Invalid("User Agent must not be blank")
                request.userAgent.length > 2048 -> ValidationResult.Invalid("User Agent must not exceed 2048 characters")
                else -> ValidationResult.Valid
            }
        }
    }
}
```

### 평가

정리해 보면 이번 변경으로 아래와 같은 개선을 얻을 수 있었습니다.

- 검증과 직렬화 코드가 더 간결해져 읽기 쉬워졌다.
- 검증과 직렬화에서 어노테이션 의존도를 줄여 리플렉션 비용을 피할 수 있었다.
- 검증 코드에서 비즈니스 로직과 무관한 반복 체크를 줄일 수 있었다.

부수적인 장점도 몇 가지 있습니다.

- Regex를 아예 제거하거나 공통화해서 재사용할 수 있다.
- `max`, `min` 같은 검증을 타입과 구조로 일부 대체할 수 있다.
- `limit`, `offset`처럼 자주 쓰는 파라미터 검사를 공통 처리하기 쉬워진다.

물론 단점이나 주의점도 있습니다.

- `LocalDateTime`, `LocalDate`처럼 Kotlinx Serialization이 기본 지원하지 않는 타입은 직접 시리얼라이저를 만들어야 한다.

```kotlin
// LocalDateTime 시리얼라이저 예시
object LocalDateTimeSerializer : KSerializer<LocalDateTime> {
    override val descriptor: SerialDescriptor = PrimitiveSerialDescriptor("java.time.LocalDateTime", PrimitiveKind.STRING)

    override fun serialize(encoder: Encoder, value: LocalDateTime) {
        encoder.encodeString(value.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME))
    }

    override fun deserialize(decoder: Decoder): LocalDateTime {
        return LocalDateTime.parse(decoder.decodeString(), DateTimeFormatter.ISO_LOCAL_DATE_TIME)
    }
}

fun Application.configureSerialization() {
    install(ContentNegotiation) {
        json(
            Json {
                // LocalDateTime 시리얼라이저 등록
                serializersModule = SerializersModule {
                    contextual(LocalDateTimeSerializer)
                }
            }
        )
    }
}

@Serializable
data class ExampleRequest(
    // LocalDateTime 필드에는 직접 만든 시리얼라이저를 사용
    @Contextual
    val timestamp: LocalDateTime
)
```

- Enum처럼 값이 정해진 타입도 경우에 따라 별도 시리얼라이저가 필요할 수 있다.
- Kotlinx Serialization은 아직 Jackson만큼 기능 폭이 넓지 않아, 필요한 기능에 따라 한계가 있을 수 있다.
- Ktor Request Validation은 요청 본문 검증에 초점이 맞춰져 있어, 쿼리 파라미터와 헤더 검증은 별도로 처리해야 한다.

그래서 이번 방향은 의도했던 목적에는 잘 맞았지만, 모든 환경에서 정답이라고 보기는 어렵습니다. 요구 사항과 사용하는 데이터 타입에 따라 적합성이 달라질 수 있습니다.

## 개선 내용 요약

아래 표에 변경 전후 차이를 간단히 정리했습니다.

| 항목 | 개선 전 | 개선 후 |
|---|---|---|
| 유효성 검사 방식 | `jakarta.validation` + 수동 호출 | Ktor 공식 `RequestValidation` 플러그인 |
| 어노테이션 사용 | `@NotNull`, `@Pattern`, `@JsonProperty` 등 다수 | 기본적으로 `@Serializable` 정도만 사용 |
| UUID 검증 | 정규식으로 검사 | `UUID` 타입으로 처리 |
| 직렬화 라이브러리 | Jackson (리플렉션 기반) | `kotlinx.serialization` (KSP 기반) |
| 검증 로직 공통화 | 컨트롤러마다 개별 작성 | `ValidatableController`로 공통 등록 가능 |
| Koin 연동 | 등록/조회가 번거롭다 | `bind` + `getAll()`로 일괄 처리 가능 |

## 마지막으로

이번에는 Ktor의 요청 처리를 조금 더 Kotlin답게 정리하는 방법을 살펴봤습니다. Ktor의 Request Validation을 활용하면 요청 검증 로직을 더 명확하게 구성할 수 있고, Kotlinx Serialization을 함께 쓰면 직렬화 구조도 한층 단순하게 정리할 수 있었습니다.

물론 Java 쪽 프레임워크와 라이브러리가 여전히 강력한 선택지인 것은 분명합니다. 다만 Kotlin과 Ktor를 사용할 때는, 언어 특성을 더 적극적으로 살리는 설계를 고민해 볼 여지가 꽤 많다고 느꼈습니다. 앞으로도 이런 식으로 구조와 성능을 함께 개선할 수 있는 방법이 있으면 계속 정리해 보고 싶습니다.
