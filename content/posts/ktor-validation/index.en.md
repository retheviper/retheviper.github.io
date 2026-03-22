---
title: "Improving Request Handling in Ktor"
date: 2025-06-28
translationKey: "posts/ktor-validation"
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - ktor
  - kotlin
---

Since Kotlin is compatible with Java, I think there are many cases where you want to rewrite part of an existing system built with Java or build a new system by changing only the language. However, even though Kotlin and Java run on the same JVM, there are some areas that can be improved due to differences in frameworks and libraries.

Although Java's frameworks and libraries offer a great deal of functionality, in part due to Java's long history, you may be able to write more concise and intuitive code with Kotlin. In addition, it is possible to design using the characteristics of Kotlin, and you can expect performance improvements.

This time, I will introduce how to implement request validation in Ktor as an example of such an improvement.

## Current Approach

In practice, we use Kotlin frameworks such as [Ktor](https://ktor.io) and [Exposed](https://www.jetbrains.com/exposed). However, some parts depended on Java libraries, using Jackson for serialization and jakarta.validation for request validation.

In the case of Spring Boot, which is often used with Java and Kotlin, it is very easy to use these libraries. For example, request validation can be automatically performed using `@Valid` and `@Validated` annotations.

However, this is not the case with Ktor. Therefore, Ktor request validation had to be done manually. For example, I did something like this:

```kotlin
// Performs request validation
object AnnotationValidator {
    private val validator: Validator = Validation.buildDefaultValidatorFactory().validator

    fun getViolationMessages(model: Any?): List<String> {
        model ?: return emptyList()
        val violations = validator.validate(model)
        return violations.map { "${it.propertyPath} ${it.message}" }
    }
}

// Interface for validation
interface Validatable(
    fun validate(): List<String>
)

// Request body
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

// Controller
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

Here's what I thought was the problem:

1. The code tends to be redundant because requests must be validated manually.
2. Code duplication is likely to occur because similar validation code needs to be written for each controller and request.
3. Relying on annotations for validation and serialization results in performance degradation due to reflection and requires code modifications if annotations need to be changed.
4. Since property types are specified using annotations, it is difficult to write type-safe code.

So, to solve these problems, I thought of a way to improve Ktor's request validation.

## Improvements

The improvements I have come up with are as follows:

1. Implement validation that does not depend on annotations using Ktor official [Request Validation](https://ktor.io/docs/request-validation.html).
2. Enable common validation logic.
3. Migrate from Jackson to [Kotlinx Serialization](https://kotlinlang.org/docs/serialization.html) to further reduce annotations and improve performance.

When divided into each part, the contents are as follows.

### Use Ktor's Request Validation

First, create an interface called ValidatableController and implement it in each controller. This interface is for validating requests using Ktor's Request Validation, and has methods for registering with Ktor plugins.

```kotlin
interface ValidatableController {
    fun configureValidation(validationConfig: RequestValidationConfig)
}
```

Then modify the data class. Delete the jakarta.validation annotation and also delete the Validatable interface.

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

Next, modify the controller. Inherit ValidatableController, move the validation logic to the `configureValidation` method, and use Ktor's Request Validation to perform request validation.

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

By doing this, you can use Ktor's Request Validation to perform request validation. This allows you to avoid relying on annotations and reduce code redundancy.

### Module Configuration

Next, configure ValidationController as a Ktor module. The reason I defined `ValidatableController` as an interface is to use [Koin](https://insert-koin.io/) used in DI to retrieve all controllers and register them in Ktor's Request Validation.

First of all, Koin has several ways to bind implementation classes to interfaces. Typical methods include the following.

```kotlin
val koinModule = module {
    single<ValidatableController> { AccessLogController() }
}
```

This method registers a class that implements the `ValidatableController` interface as a single instance. However, this does not allow you to register multiple `ValidatableController`s.

In this case, it is possible to use `get()` and obtain only one `ValidatableController`. However, what we want to do here is to obtain multiple `ValidatableController` and standardize the request validation, so this method is not suitable.

Another method is to use [bind](https://insert-koin.io/docs/reference/koin-core/definitions/#additional-type-binding).

```kotlin
val koinModule = module {
    single { LogController() } bind ValidatableController::class
    single { AnotherController() } bind ValidatableController::class
}
```

Unlike the previous code, using `bind` allows Koin to retrieve all classes that implement that interface. So, you can get all controllers with code like below.

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

By doing this, you can standardize the validation of requests. From now on, when adding a new controller, just implement the `ValidatableController` interface and it will be automatically registered in Ktor's Request Validation.

### Use Kotlinx Serialization

Finally, use Kotlinx Serialization to do the serialization. This can reduce the use of annotations and improve performance.

Kotlinx Serialization uses KSP (Kotlin Symbol Processing) to generate code for serialization at compile time. This allows you to serialize without using reflection, which can improve performance, and allows you to write type-safe code during request validation, which also improves code readability.

First, to use Kotlinx Serialization, migrate from your existing Jackson to Kotlinx Serialization. Configure the serializer using the ContentNegotiation plugin as follows: You can use `JsonNamingStrategy.SnakeCase` to change the property naming convention to snake case.

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

Next, add `@Serializable` to the request body data class. Combined with the naming strategy above, this lets Kotlinx Serialization map property names to snake_case without separate field annotations. You can also rely on the property types themselves, which makes null handling explicit and type-safe.

```kotlin
@Serializable
data class CreateLogRequest(
    val userId: UUID,
    val referral: String,
    val userAgent: String
)
```

Next, fix the controller validation. The type of userId has been changed from `String` to `UUID`, so regular expression checking is no longer necessary. It felt pretty refreshing.

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

### Evaluation

In the end, I think this approach gave us the following improvements:

- Validation and serialization code is now simpler and more readable.
- We no longer use annotations for validation and serialization, avoiding performance degradation due to reflection.
- We were able to reduce checks other than business logic for validation.

There are also the following side effects:

- Regex is no longer needed or instances can be reused by making them common
- Checks such as max and min are no longer necessary.
- Common checks for frequently used parameters such as limit and offset

However, there are some concerns, such as the following.

- If you use types not supported by Kotlinx Serialization, such as LocalDateTime or LocalDate, you will need to implement your own serializer.

```kotlin
// Example serializer for LocalDateTime
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
                // Register the LocalDateTime serializer
                serializersModule = SerializersModule {
                    contextual(LocalDateTimeSerializer)
                }
            }
        )
    }
}

@Serializable
data class ExampleRequest(
    // Use the custom serializer for LocalDateTime fields
    @Contextual
    val timestamp: LocalDateTime
)
```

- When using a type whose value is determined at compile time, such as Enum, you also need to implement your own serializer.
- Kotlinx Serialization does not yet have the same functionality as Jackson, so if you need a specific feature, it may not be possible to implement it with Kotlinx Serialization.
- Ktor Request Validation supports validation of only the request body, so validation of query parameters and headers must be implemented separately.

So I think this change was successful for its intended purpose, but there is still room for improvement. Depending on your environment and requirements, this approach may not always be the best fit.

## Summary of improvements

The table below summarizes the differences before and after the improvements.

| Item | Before improvement | After improvement |
|-----------------------|----------------------------------------|---------------------------------------|
| Validation method | `jakarta.validation` + manual call | Ktor official RequestValidation plugin |
| Use of annotations | `@NotNull`, `@Pattern`, `@JsonProperty` and many others | Only `@Serializable`, basically unnecessary |
| UUID validation | Check with regular expressions | Type safety with `UUID` type specification |
| Serialization library | Jackson (uses reflection) | kotlinx.serialization (fast using KSP) |
| Common validation | Described separately for each controller | Can be commonly defined with `ValidatableController` |
| Cooperation with Koin | Acquisition and registration are complicated or not possible | Batch acquisition with `bind` + `getAll()` |

## Finally

This time, I introduced how to improve Ktor's request processing. By taking advantage of Kotlin's characteristics and using Ktor's Request Validation, we were able to easily validate requests. We were also able to improve serialization performance by using Kotlinx Serialization.

In this way, Kotlin and Ktor can sometimes enable improvements that cannot be achieved with Java frameworks and libraries. In particular, it has great advantages in that it can be designed to take advantage of Kotlin's characteristics and can be expected to improve performance.

I would like to continue to aim for better design and performance improvements in development using Kotlin and Ktor. If you have any other improvements or ideas for Ktor or Kotlin, please share them with us.

See you soon!
