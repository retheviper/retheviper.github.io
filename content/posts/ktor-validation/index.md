---
title: "KtorのRequest処理を改善する"
date: 2025-06-28
categories: 
  - ktor
image: "../../images/ktor.webp"
tags:
  - ktor
  - kotlin
---

KotlinはJavaとの互換性があるので、Javaで構築した既存のシステムの一部を書き換えたり、言語だけを変えて新しいシステムを構築したりするケースが多いかなと思います。しかし、同じJVM上で動作するKotlinとJavaでも、フレームワークやライブラリの違いにより改善できる箇所もあります。

Javaの歴史が長いこともあり、Javaのフレームワークやライブラリは非常に多くの機能を提供していますが、Kotlinではより簡潔で直感的なコードを書くことができる場合があります。また、その中ではKotlinの特性を活かした設計が可能だったり、性能の向上が期待できたりします。

今回はそのような改善の一例として、Ktorでのリクエストバリデーションの実装方法について紹介します。

## 既存

実務では[Ktor](https://ktor.io)、[Exposed](https://www.jetbrains.com/ja-jp/exposed)といったKotlin用のフレームワークを採用しています。ただ一部ではJavaのライブラリに依存していて、直列化ではJackson、リクエストのバリデーションではjakarta.validationを使用していました。

JavaやKotlinでよく使われるSpring Bootの場合、これらのライブラリを使うのがとても簡単ですね。例えば、リクエストのバリデーションは`@Valid`や`@Validated`アノテーションで自動的に行うことができます。

しかし、Ktorではそうでもいきません。なので、Ktorのリクエストバリデーションは、手動で行う必要がありました。例えば、以下のようにしていました。

```kotlin
// リクエストのバリデーションを行う
object AnnotationValidator {
    private val validator: Validator = Validation.buildDefaultValidatorFactory().validator

    fun getViolationMessages(model: Any?): List<String> {
        model ?: return emptyList()
        val violations = validator.validate(model)
        return violations.map { "${it.propertyPath} ${it.message}" }
    }
}

// バリデーションを行うインターフェース
interface Validatable(
    fun validate(): List<String>
)

// リクエストボディ
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

// コントローラー
class LogController : KoinComponent {
    private val logUseCase: sLogUseCase by inject()

    fun createLog(request: CreateLogRequest): CreateLogResponse {
        val validationErrors = request.validate()
        if (validationErrors.isNotEmpty()) {
            ValidationResult.Invalid("Validation failed: ${validationErrors.joinToString(", ")}")
        }

        return logUseCase.createLog(request)
    }
}
```

ここで自分が問題と思ったのは、以下のようなものがあります。

1. リクエストのバリデーションを手動で行う必要があるため、コードが冗長になりやすい。
2. 各コントローラーやリクエストに対して同じようなバリデーションコードを書く必要があるため、コードの重複が発生しやすい。
3. バリデーションと直列化でアノテーションに依存しているため、リフレクションによるパフォーマンスの低下や、アノテーションの変更が必要な場合にコードの修正が必要になる。
4. プロパティの型をアノテーションで指定するため、型安全なコードが書きにくい。

なので、これらの問題を解決するために、Ktorのリクエストバリデーションを改善する方法を考えました。

## 改善

自分で考えた改善策は以下のようなものです。

1. Ktor公式の[Request Validation](https://ktor.io/docs/request-validation.html)を使用して、アノテーションに依存しないバリデーションを実装する。
2. バリデーションのロジックを共通化できるようにする。
3. Jacksonから[Kotlinx Serialization](https://kotlinlang.org/docs/serialization.html)に移行して、アノテーションをさらに減らしつつとパフォーマンスを向上させる。

それぞれ分けると以下のような内容になります。

### KtorのRequest Validationを使用する

まずはValidatableControllerというインターフェースを作成して、それぞれのコントローラーで実装します。このインターフェースは、KtorのRequest Validationを使用してリクエストのバリデーションを行うためのもので、Ktorのプラグインに登録するためのメソッドを持っています。

```kotlin
interface ValidatableController {
    fun configureValidation(validationConfig: RequestValidationConfig)
}
```

そして、data classを修正します。jakarta.validationのアノテーションは削除し、Validatableというインターフェースも削除します。

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

次に、コントローラーを修正します。ValidatableControllerを継承させ、バリデーションのロジックを`configureValidation`メソッドに移動し、KtorのRequest Validationを使用してリクエストのバリデーションを行います。

```kotlin
class LogController : KoinComponent, ValidatableController {
    private val logUseCase: LogUseCase by inject()

    override fun configureValidation(validationConfig: RequestValidationConfig) {
        validationConfig.validate<CreateLogRequest> { request ->
            when {
                request.userId == null -> ValidatValidationResult.InvalidtionResult.Invalid("User ID must not be null")
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

このようにすることで、KtorのRequest Validationを使用してリクエストのバリデーションを行うことができます。これにより、アノテーションに依存せず、コードの冗長性を減らすことができます。

### モジュールの設定

次は、ValidationControllerをKtorのモジュールとして設定します。インターフェースとして`ValidatableController`を定義した理由は、DIで使用している[Koin](https://insert-koin.io/)を使用して、すべてのコントローラーを取得し、KtorのRequest Validationに登録するためです。

まずKoinではinterfaceに対して実装クラスをバインディングするいくつかの方法があります。代表的には、以下のような方法があるでしょう。

```kotlin
val koinModule = module {
    single<ValidatableController> { AccessLogController() }
}
```

この方法では、`ValidatableController`インターフェースを実装したクラスを単一のインスタンスとして登録します。しかし、これでは複数の`ValidatableController`を登録することができません。

この場合、`get()`を使用して、1つの`ValidatableController`のみを取得することはできます。しかしここでやりたいことは、複数の`ValidatableController`を取得して、リクエストのバリデーションを共通化することなのでこの方法は適していません。

またの方法が、[bind](https://insert-koin.io/docs/reference/koin-core/definitions/#additional-type-binding)を使用する方法です。

```kotlin
val koinModule = module {
    single { LogController() } bind ValidatableController::class
    single { AnotherController() } bind ValidatableController::class
}
```

先ほどのコードとは違って、`bind`を使用することで、Koinはそのインターフェースを実装したすべてのクラスを取得できるようになります。なので、以下のようなコードで全てのコントローラーを取得することができます。

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

こうすることでリクエストのバリデーションを共通化することができます。これからは新しいコントローラーを追加する際も、`ValidatableController`インターフェースを実装するだけで、KtorのRequest Validationに自動的に登録されるようになります。

### Kotlinx Serializationの使用

最後に、Kotlinx Serializationを使用して直列化を行います。これにより、アノテーションの使用を減らし、パフォーマンスを向上させることができます。

Kotlinx Serializationは、KSP（Kotlin Symbol Processing）を使用して、コンパイル時に直列化のコードを生成します。これにより、リフレクションを使用せずに直列化を行うことができ、パフォーマンスの向上が期待できまし、リクエストのバリデーションでも型安全なコードを書くことができるため、コードの可読性も向上します。

まず、Kotlinx Serializationを使用するために、既存のJacksonからKotlinx Serializationに移行します。以下のように、ContentNegotiationプラグインを使用してシリアライザを設定します。プロパティの命名規則をスネークケースに変更するために、`JsonNamingStrategy.SnakeCase`を使用することができます。

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

次に、リクエストボディのデータクラスに`@Serializable`アノテーションを追加します。これにより、Kotlinx Serializationがこのクラスを直列化できるようになります。シリアライザの方でスネークケースに変換されるので、アノテーションは要らず、プロパティ名をそのまま使用できます。また、プロパティはそのまま型を指定するだけです。型を指定することでNullチェックも自動になります。

```kotlin
@Serializable
data class CreateLogRequest(
    val userId: UUID,
    val referral: String,
    val userAgent: String
)
```

次に、コントローラのバリデーションを修正します。userIdの型が`String`から`UUID`に変更されたので、正規表現のチェックは不要になりました。かなりスッキリしましたね。

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

### 評価

結論的に以下のような改善ができたと思います。

- バリデーションと直列化のコードが簡潔になり、可読性が向上した
- バリデーションと直列かでアノテーションを使用しなくなり、リフレクションによるパフォーマンスの低下を回避できた
- バリデーションに対してビジネスロジック以外のチェックを減らすことができた

また、サイドエフェクトとして以下のような点もあります。

- Regexが不要になっているか、共通化してインスタンスの使い回しができる
- max, minなどのチェックが不要になっている
- limit, offsetなど頻繁に使うパラメータのチェックを共通化できる

ただ、以下のような懸念もあります。

- LocalDateTimeやLocalDateなどのKotlinx Serializationでサポートされていない型を使用する場合、独自のシリアライザを実装する必要がある
- Kotlinx SerializationはまだJacksonほどの機能がないため、特定の機能が必要な場合はKotlinx Serializationでは実装できないことがある
- Ktor Request Validationはリクエストボディのみのバリデーションをサポートしているため、クエリパラメータやヘッダのバリデーションは別途実装する必要がある

なので、今回の対応は意図としては成功したと思いますが、まだまだ改善の余地はあるかなと思います。また、環境や要件によっては、このような改善が適さない場合もあるかもしれません。

## 最後に

今回はKtorのRequest処理を改善する方法について紹介しました。Kotlinの特性を活かし、KtorのRequest Validationを使用することで、リクエストのバリデーションを簡潔に行うことができました。また、Kotlinx Serializationを使用することで、直列化のパフォーマンスも向上させることができました。

このように、KotlinとKtorを使用することで、Javaのフレームワークやライブラリでは実現できないような改善が可能になることがあります。特に、Kotlinの特性を活かした設計や、パフォーマンスの向上が期待できる点は大きなメリットです。

今後もKotlinやKtorを使った開発において、より良い設計やパフォーマンスの向上を目指していきたいと思います。もし、他にもKtorやKotlinに関する改善点やアイデアがあれば、また共有していきたいと思います。

では、また！
