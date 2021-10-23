---
title: "KtorでRole-based Authorizationを実装する"
date: 2021-08-09
categories: 
  - ktor
photos:
  - /assets/images/sideimage/ktor_logo.jpg
tags:
  - kotlin
  - ktor
  - authorization
---

前回、Ktorを紹介しながら、Ktorにはまだ`Role-based Authorization`に対応してないので、自前でそのような機能を実装する必要がある、と述べました。Ktorはまだ歴史が短く、SpringやDjango、Railsのように幅広く使われているフレームワークでもないので、おそらく他に比べ実のアプリケーションを作るにあたっては必要な機能が十分でない可能性がありますね。なので、こうやって必要な機能がない場合は直接その機能を実装するしかないです。

幸い、Ktorでは機能を[Plugin](https://ktor.io/docs/plugins.html)といい、モジュール単位で追加できるため、必要な機能を実装するのもそのPluginを作ることでできるようになります。ただ、モジュールを利用するということは、機能単位の管理がやりやすくなるものの、そのモジュールはどうやって機能するか、また、どういうお作法が必要となるかを知る必要がありますね。

今回はネット上に公開されてある[記事](https://medium.com/@shrikantjagtap99/role-based-authorization-feature-in-ktor-web-framework-in-kotlin-dda88262a86a)を参考にしながら、KtorのRole-based Authorizationを`Plugin`として実装してみました。そこで、今回のポストではこういう自作の`Plugin`がどうやってKtorの機能として動作するか、どうやって実装するのかについて述べたいと思います。

## Role-based Authorizationとは

まずは、そもそも`Role-based Authorization`とは何か、からですね。これは、ウェブアプリケーションでよく言われている「認可」の方式のうち、ユーザの`Role`（役割）に基づいて、APIの実行を制御するものです。例えばECサイトの場合、商品に対して問い合わせをするのは認証されたユーザなら誰でもできるべきですが、「お知らせを書く」や「商品の在庫数を変更する」などの機能はその権限を持つユーザ（Admin）に限定すべきですね。なので、ここで「一般ユーザ」と「管理者」などの`Role`を設け、APIに対してのリクエストが発生した際にその`Role`をまず確認し、その権限のあるユーザのみがAPIを実行できるようにする、というのが`Role-based Authorization`の基本的な概念です。

これを実現するために既存のアプリに導入する必要のあるものは、大きく分けて`Role`の概念と、それを元にリクエストをフィルタリングする機構の二つです。前者の場合はどんなロールがあり、どういう形でユーザに紐付けるかを考えればいいだけなので、テーブルやカラムを追加して既存のユーザの情報と紐づけるだけですみます。しかし、後者はまずフレームワークでどうやってリクエストをフィルタするか、まずその構造から理解する必要がありますね。なんので、まずはKtorでリクエストを扱う方法に対して紹介したいと思います。

## PipelineとFeature

Ktorの特徴のうち、最も重要と言えるものは、[Pipeline](https://ktor.io/docs/pipelines.html)の概念です。この`Pipeline`に対して、公式では以下のように説明しています。

> The pipeline is a structure containing a sequence of functions (blocks/lambdas) that are called one after another, distributed in phases topologically ordered, with the ability to mutate the sequence and to call the remaining functions in the pipeline and then return to current block.

この説明だけでは理解が難しいものですが、要するに、Ktorにおいての処理の単位のことを指していると言ってもよいものです。`Pipeline`ではAPIのコールからレスポンスまで一連の流れとしての処理を定義することができます。なので`Pipeline`として実現されている代表的な機能は`Router`、リクエストに対してのハンドリングを定義する機能（Springの`Controller`に対応するもの）となります。

また、`Pipeline`は拡張できるものなので、その形式に合わせて新しい`Pipeline`を実装することでモジュール(公式の表現では`Plugin`)を実現するのもできます。これらのモジュールを実装し、アプリケーションにインストールすることで、そのモジュールの機能を利用できるようになるのがKtorの特徴です。例えば、kotlin公式のJSON Mapperである[kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization)をアプリケーションに追加するためには以下のようなコードを書きます。

```kotlin
fun Application.main() {
    install(ContentNegotiation) {
        json()
    }
}
```

ここで呼び出している`install`関数の実装を見ると、以下のようになっています。`feature`(モジュール)と、そのモジュールの設定となる`configure`が引数になっているのがわかります。

```kotlin
public fun <P : Pipeline<*, ApplicationCall>, B : Any, F : Any> P.install(
    feature: ApplicationFeature<P, B, F>,
    configure: B.() -> Unit = {}
): F
```

先ほどの`kotlinx.serialization`をインストールするために使っていたコードでは、`feature`として`ContentNegotiation`を渡し、その設定として`json`を使うという設定をしているのだなという推測ができますね。実際、`ContentNegotiation`の実装は、以下のような形となっています。一部のコードは省略していますが、クラスの中には`Configuration`というクラスと、`ApplicationFeature`を継承した`companion object`を中に持っているのがわかります。

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

上記の実装でわかるように、`Pipeline`として機能するためにはモジュールの設定のための`Configuration`というクラスと、モジュールとして機能するための`ApplicationFeature`を継承した`companion object`が必要であることがわかります。なので、この構造を持ったクラスを定義できれば、自作のモジュールをアプリケーションに実装できるということがわかりますね。

## Pluginの実装

では、実際に`Pipeline`として、リクエストに対する認可を判定する機能を作るとしましょう。まずはロールを定義します。`enum`が良さそうですね。ここではシンプルに管理者と一般ユーザの2種を作ってみます。

```kotlin
enum class Role(val value: String) {
    ADMIN("ADMIN"),
    USER("USER")
}
```

これらのロールは、テーブルなどで管理する必要もありますが、ログイン中のユーザ情報から取得する必要もありますね。認可のためには、ログイン中のユーザにとあるロールが与えられているかどうかの確認が必要となるからです。なので、`io.ktor.auth.Principal`を継承したユーザの情報もクラスとして作り、ログインに成功した時はこのクラスにユーザのロールを格納することにします（方法は認可とは関係ないのでここでは割愛させてください）。以下はユーザの情報を格納するための簡単な例です。

```kotlin
data class UserPrincipal(
    val username: String,
    val roles: Set<Role> = emptySet()
) : Principal
```

次に、ロールでアクセスを制限する関数を作ります。`Router`のエンドポイントに、どのロールの場合にアクセスできるかを指定するようなイメージです。例えば以下のような形で使えたらいいかと思います。

```kotlin
fun Application.main() {
    routing {
        // 管理者のみアクセスできる
        withRole(Role.ADMIN) {
            get("/admin") {
                call.respond("This is admin page")
            }
        }

        // 一般ユーザがアクセスできる
        withRole(Role.USER) {
            get("/user") {
                call.respond("This is user page")
            }
        }
    }
}
```

`Router`の使い方でわかるように、`Pipeline`でのコードブロック（関数）はネストが可能なのでこのように一つのレイヤーを挟むのも可能です。ここで追加した`withRole`という関数でロールを確認し、APIにアクセスできるかどうかを判定するようにしたら良いでしょう。

### AuthorizedRotueSelectorの実装

まずは`RouteSelector`を実装します。これは、`routing`の中にこれから作る認可の関数がネストできるようにするためのものです。もっともシンプルな実装は以下のようになります。

```kotlin
class AuthorizedRouteSelector() : RouteSelector() {
    override fun evaluate(context: RoutingResolveContext, segmentIndex: Int) = RouteSelectorEvaluation.Constant
}
```

### child routeの実装

先に実装した`AuthorizedRouteSelector`を利用して、実際に`child route`として機能する関数を作ります。この`child route`は`Router`の下にネストすることになるので、`Route`の拡張関数を作ることにします。引数としては判定のためのロールと、その下にネストするエンドポイントの関数を設定できるようにすれば良いでしょう。実装は以下のようにします。

```kotlin
fun Route.withRole(val role: Role, build: Route.() -> Unit): Route {
    val authorizedRoute = createChild(AuthorizedRouteSelector())
    application.feature(RoleBaseAuthorizer).interceptPipeline(this, role)
    build()
    return authorizedRoute
}
```

ここで実装しているものは、まず`AuthorizedRouteSelector`で`child route`を作り、その後`Pipeline`をインターセプトして、ユーザが指定したロールに該当するかどうかを判定します。問題なければ`build`を実行させますが、これがネストしている`child route`になります。最後に、エンドポイントをネストできるように先ほど作成した`child route`のインスタンスを返します。

`Pipeline`をインターセプトする時に呼び出している`RoleBaseAuthorizer`は、別途クラスとして作ることにします。これを`Feature`として作ることになります。

### 認可処理のモジュールの実装

では、本格的に認可の処理を担当するモジュール（`Feature`）を実装することにします。先に述べた通り、`Configuration`と`Feature`を内部に持ったクラスを作ります。ここで`Configuration`は、ログイン中のユーザからどうやってロールの情報を取得するかの設定ができるクラスにします。こうすることで、以下のようなことが可能になるでしょう。

```kotlin
fun Application.main() {
    // RoleBaseAuthorizerをFeatureとしてインストール
    install(RoleBaseAuthorizer) {
        // ログイン中のユーザの情報からロールを取得する方法をConfigurationとして指定
        getRoles { (it as UserPrincipal).roles }
    }
}
```

`Configuration`では、ログイン中のユーザ情報となる`UserPrincipal`から`roles`を取得する、という関数を渡します。これを持って、`RoleBaseAuthorizer`では`withRole`関数で指定したロールとユーザのロールを比較するようにします。

認可のモジュールの設定方法のイメージができたので、次に`RoleBaseAuthorizer`を実装します。例えば以下のようになります。

```kotlin
class RoleBaseAuthorizer(config: Configuration) {

    class Configuration {
        var userRoles: (Principal) -> Set<Role> = { emptySet() }

        // ログイン中のユーザの情報からロールの取得方法をセット
        fun getRoles(roles: (Principal) -> Set<Role>) {
            userRoles = roles
        }
    }

    private val getRoles = config.userRoles

    fun interceptPipeline(
        pipeline: ApplicationCallPipeline,
        role: Role
    ) {
        // Pipelineの位置付け
        pipeline.insertPhaseAfter(ApplicationCallPipeline.Features, Authentication.ChallengePhase)
        pipeline.insertPhaseAfter(Authentication.ChallengePhase, AuthorizationPhase)

        // インターセプト時の処理
        pipeline.intercept(AuthorizationPhase) {
            // ログイン中のユーザの情報を取得
            val principal = call.authentication.principal<UserPrincipal>()
                ?: throw AuthorizationException("Missing principal")

            // ユーザ情報からロールを取得
            val roles = getRoles(principal)

            if (roles.none { role }) {
                // ログイン中のユーザのロールに、アクセス可能なロールが含まれてない場合の処理
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

先に説明した通り、`Configuration`ではユーザのロール情報を取得する関数を設定し、保存します。そして`interceptPipeline`では、その関数を持って`Pipeline`をインターセプトし、ロールの検証を行うようにします。

また、`interceptPipeline`では、引数として渡された`Pipeline`の位置付けを設定する必要があります。上記のコードでは、「認証の後」に位置付けしています。その後のロジックは、色々な方法があると思いますので、ここでは割愛させていただきます。

他に、`Feature`の場合は、`RoleBaseAuthorizer`が独立したモジュールとして使える設定を行います。単純に名前をつけてインスタンスを返すような、お作法的なものですね。

ここまでの実装が終わったら、一通り認可に関するモジュールの作成は終わります。ただ、`interceptPipeline`の処理としてユーザのロールが、APIにアクセスできない場合の処理として考えられることは二つほどあります。

#### レスポンスを返して終了

まず考えられる方法は、適当なレスポンスを返し、そこで処理を終了させることです。この場合、以下のように実装ができます。

```kotlin
if (roles.none { role }) {
    // レスポンスを返す
    call.respond(
        status = HttpStatusCode.Forbidden,
        message = "permission not granted: $role"
    )

    // Pipelineの終了
    finish()
}
```

ここで注意すべきことは、レスポンスを返すだけで`Pipeline`は終わらないということです。レスポンスを返し処理を止めたい場合は必ず`finish()`を呼び出して、`Pipeline`を終了させましょう。

#### Exceptionを投げる

もう一つの方法は、例外を投げる方法ですね。例えば以下のようにします。

```kotlin
// 認可されてない場合の例外
class AuthorizationException(override val message: String) : Exception(message)

if (roles.none { role }) {
    throw AuthorizationException("permission not granted: $role")
}
```

例外を投げる場合は、当然`Pipeline`の処理が止まることになりますが、アプリケーションのログでも例外になるのであまり良くないですね。幸い、KtorにもSpringの`ExceptionHandler`のような機能があるので、それを活用したら適切な例外のハンドリングが可能になります。例えば以下のようなことができますね。

```kotlin
// 認可されてない場合のレスポンス
data class AuthFailedResponse(val reason: String)

// 例外のハンドリング
install(StatusPages) {
    exception<Throwable> { cause ->
        when (cause) {
            // 認可の場合の処理
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

これでアプリケーションのログも綺麗になりますし、他の例外処理に対しても`when`の分岐を増やすだけで対応ができるようになります。

## 最後に

最初に思っていたことよりも膨大な内容を扱うことになったので、いつもより説明が大雑把な気もしますが、これで`Pipeline`とそれを応用した自作モジュールの実装についての説明は一通りできたかなと思います。なので、これを応用すれば、他のモジュールを追加するのもそう難しくなさそうな気がしますね。深堀すると色々また出そうな気がしますが、それについては機会があればまた今度のポストのネタにしましょう。（正直あまり詳しくありませんので…）

個人的には、このようにRole-based Authorizationの機能を作りながら知った。一連の処理を`Pipeline`という単位で扱うという概念ががかなり新鮮で、良いと思いました。処理に対してのインターセプトはSpringでもできるのですが、処理の流れ自体を一つの単位として扱えるならより色々なことができそうな気もしますね。まだKtorに触れたばかりなので、詳しいことはもっと時間をかけてゆっくり調べる必要がありそうですが。

確かなのは、Ktorはかなり魅力的なフレームワークであるということです。最初はSpringなど、既存の有名なフレームワークと比べ色々と機能が足りない認証だったのですが、こうやって簡単にモジュールを作れるとしたら意外と問題ないかもしれない、という気がします。もちろんそれでも、プロダクションレベルのものを作るにはまだ色々と検証が必要そうな認証はありますが。

では、また！
