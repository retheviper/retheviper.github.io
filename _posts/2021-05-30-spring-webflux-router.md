---
title: "WebFluxではFunctional Enpointを使うべきか"
date: 2021-05-30
categories: 
  - spring
photos:
  - /assets/images/sideimage/spring_logo.jpg
tags:
  - spring
  - webflux
  - kotlin
  - rest api
---

以前、[Spring WebFluxに関するポストを書いたこと](../../../../2020/09/06/spring-webflux/)があって、そこで少しだけMVCパターン(`Controller`/`Service`)と`Functional Endpoint`(`Router`/`Handler`)に関して触れました。結論だけ先に述べますと、Functional Endpointの導入はMVCパターンは長く使われている良いパターンでありますが、性能や関数型プログラミングには適してないのという問題があるので、それを改善するためのものだといえます。

さて、その説明だけだと、Spring WebFluxを使う際にはなるべくFunctional Endpointを使うべきな気もします。しかし、実際はどうでしょうか？例えば、従来のSpring MVCと同じくController/Serviceを使う場合は本当にRouter/Handlerを使う時と比べ性能が劣るのか？また、Functional Endpointを使う際に考慮すべき、「MVCパターンにはなかった問題」はないか？といったことを考えられます。

なので、今回はその二つのパターンを用いて、Spring WebFluxによるサーバサイドアプリケーションを実装するときに考えたいことを少しまとめてみました。

## プログラミングのパラダイムとして

SpringのMVCパターンは、アノテーションによるメタプログラミングとオブジェクト指向といった昔ながらの考え方に基づいたパラダイムに近いといえます。もちろん、AOPやDIといったSpring Framework特有の特徴はありますが、[Reactive Streams](https://www.reactive-streams.org)の実現体である`Mono`/`Flux`で書かれたWebFluxのコードと比べたら、まだ伝統的な書き方に近いという感覚はありますね。

ここでオブジェクト指向と関数型のうち、どれが良いかという議論はしません。また、Javaは元々オブジェクト指向の言語としてデザインされましたが、1.8以降はFunctional Interfaceの導入である程度関数型プログラミングができるようになりましたし、Kotlinでもそれは大きく変わらないことです。なので、Spring MVCとSpring WebFluxのうちどれかを選ぶということがコードをオブジェクト指向として書くか、関数型として書くかという結論にもなりません。

しかし、Spring WebFluxでは、MVCパターンとFunctional Endpointのどれかを選べるという点からして、どちらかのパラダイムに寄せた書き方はできるというのも事実です。ここでどれを取るかを判断するには、コードを書く人同士の合意がもっとも重要なのではないかと思います。なぜかというと、結局プログラミングのパラダイムというものは何よりもプログラミングの「効率」のために発展してきたからです。

なので、ここでの判断基準は「如何に読みやすいか」「如何に早く成果を出せるか」など、実利的なものとなるべきでしょう。例えば、すでにサービスとして機能しているアプリケーションの同時実行性能を向上させたい場合は、MVCパターンとして書いた方がすぐにサービスを立ち上げられるので良いと思ったら、それで理由は十分かと思います。もしくは、すでにFunctional Endpointに慣れているプログラマが多い場合は積極的にそれを導入するとかですね。つまり、私の観点からすると、プログラミングのパラダイムは実務者の立場からすると効率により選択するべきものではないかと思います。

では、Contorller/ServiceのパターンとRouter/Handlerのパターンの実際はどう違うのかを、コードを通じて見ていきたいと思います。

### MVCパターンで書く場合

Spring WebFluxのMVCパターン、つまりContoller/Serviceのパターンは、その名の通り既存のSpring MVCと比べあまり変わらない感覚で書くことができます。なので、例えば以下のようなコードを書くとしたら、これだけではSpring MVCとの違いがあまりわからないくらいですね。

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

ただ、Spring WebFluxでは、DB接続を含め完全なノンブロッキングを実現するためには[R2DBC]のようなノンブロッキング対応のAPIを使う必要があります。これはつまり、`Reactive Stream`を使う必要があるとのことであって、必然的にその実現体であるMono/Fluxを使う必要があるということです。

なので、とりあえずRepositoryからMono/Fluxを取得し、Reactive Stream固有の書き方に合わせてコードを書いていくしかないということになります。問題は、Reactive Streamはその名前から普通にJavaのStreamの感覚で扱えば良い印象ですが、実際の処理はそう簡単じゃないということです。例えば、JPAやMyBatisのような既存のブロッキングベースのAPIを使う場合は、Serviceのメソッドでは以下のようなコードを書くことになりますね。

```kotlin
// ユーザIDでユーザ情報とメール送信履歴を取得する
fun getMemberWithMailRecord(memberId: Int): MemberWithMailRecord {
    // ユーザ情報を取得する
    val member = memberRepository.getMember(id = memberId) ?: throw RuntimeException("Can't find member")
    // ユーザが作成したメール送信履歴を取得する
    val mailRecord = mailRecordRepository.getMailRecord(memberId = memberId) ?: throw RuntimeException("Can't find mailRecord")
    // ユーザ情報とメール送信履歴を合わせて返却
    return MemberWithMailRecord(
            member = member,
            mailRecord = mailRecord  
        )
}
```

しかし、Mono/Fluxを返すAPIを使う場合は、以下のようなコードになります。

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

やっていることは同じでも、果たしてこれが書きやすく、読みやすいコードであるかどうかは悩ましいですね。他にもFluxで取得したデータをListに変えたい場合、取得したMonoのデータでさらにMonoを取得したい場合など、より複雑な処理が必要な場面ではますます書き方は複雑になります。

幸い、Kotlinには[Coroutines](https://kotlinlang.org/docs/coroutines-overview.html)があるので、このような複雑な書き方をより簡単に書くことはできます。Corutinesを適用したら、Mono/Fluxを使う場合でも上記のコードは以下のようになりますね。

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

Coroutinesを使う場合はスコープの指定が必要となるのが一般的ですが、実際はControllerのメソッドまでを`suspend`として定義しておくと良いみたいです。ただ、既存のプロジェクトをSpring MVCからWebFluxに移行する場合にこうやって多くの処理をsuspendメソッドにすると、ユニットテストの方を直すのが大変になる可能性もあるのでそこは要注意です。

### Functional Endpointで書く場合

続いて、Functional Endpontを使う場合のコードです。MVCパターンの問題としてアノテーションがあげられていますが、Router/Handlerでもアノテーションを使うことはできますし、アプリケーションのアーキテクチャによっては必然的にクラスを分けてアノテーションで管理することになるのが一般的かなと思います。なので、そのようなケースではRouterを`@Bean`として登録したり、Handlerも`@Component`として定義する場合もあります。そういう場合は、以下のようなコードになります。

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

Functional Endpointを使う場合の特徴は、RouterはあくまでエンドポイントとHandlerをつなぐ役割をするだけなので、Handlerで[ServerRequest](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerRequest.html)を受け取り[ServerResponse](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerResponse.html)を返す処理までを書くことになるということです。MVCパターンではRestControllerの戻り値としてResponseEntityや自分で定義したクラスを自由に指定でるし、Serviceではビジネスロジックだけを担当するパターンが多いのを考えるとかなり独特であるといえます。

このようにServerRequestとServerResponseを使うため、HandlerはServiceと比べビジネスロジック部分が一回層が深くなった形になります。ServerResponseのbodyでロジックを書いて、それを返す形ですね。例えば以下のようなコードになります。

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

この場合でもでもCoroutinesを使うことはもちろんできます。Corutinesを使う場合は、以下のような書き方ができるでしょう。

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

## 性能

MVCパターンの問題としてあげられるものの一つとして、アノテーションがあります。アノテーションを使うということは、必然的にリフレクションを使うことになるので、自然に性能の低下にもつながるという話ですね。これだけみると、WebFluxではMVCパターンよりもFunctional Endpointを使ったほうが性能でも有利であるように見えます。しかし実際はどうでしょうか？

[Springの公式ドキュメント](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-programming-models)では、Functional Enddpointのことを`lightweight(軽量)`とは表現しているものの、それ以外に性能がどうという話は一切述べてないです。多くの場合、性能の比較はSpring MVCとSpring WebFluxを対象としていて、WebFluxでのMVCパターンとFunctional Endpointのケースはあまり探せませんでした。なので、ここでは簡単にベンチマークを行うことで二つのパターンでの性能の違いを検証してみました。

ベンチマークツールとしては、Jmeterを使うこともできましたが、短いコマンドで測定ができるのもあり、今回は[Apache HTTP server benchmarking tool](https://httpd.apache.org/docs/2.4/en/programs/ab.html)を使ってテストを実施しています。

### 使ったコード

性能測定として知りたいのは「実装パターンで性能が変わるか」ということなので、あえてDB接続は排除しました。比較のために作ったサンプルアプリケーションでは、単純にデータを生成する共通のロジックと、それを返すだけのContoller/Service, Router/Hanlderのセットで構成されています。

#### 共通

データを生成するロジックそのものは共通で、単純にループでListを生成するようにしています。

```kotlin
// 固定値のデータを生成し返すクラス
object DataCreationUtil {

    // 1970年1月1日から2021年12月31日まで
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

// 生成されるデータ
data class Person(
    val id: Long,
    val name: String,
    val birth: LocalDate,
    val joined: LocalDate
)
```

#### Controller/Serviceの実装

MVCパターンについてはコードだけでも十分わかると思いますので、説明は割愛します。

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

#### Router/Handlerの実装

Functional Endpointでは、MVCパターンと比べ処理と言えるものは全部Handlerの方に書かれてある、という違いがあります。

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

### テスト結果

テストは以下のような条件で実施しました。

- ユーザ数は5000, ユーザごとのリクエストは50に設定
- ワームアップ時間を考慮して、パターンごとにテストを分ける
  - サーバの再起動後にテストを実施
  - テストは10回ループ

実際に使ったスクリプトは以下のようなものです。サーバの起動後にこれを実行し、10回のループが終わったら再起動後にFunctional Endpointのテストを実施しています。

```bash
#!/bin/bash

for i in {1..10}
do
 ab -n 5000 -c 50 http://localhost:8080/performance-controller
done
```

ただ、こうやってもやはりテスト結果では周回ごとに偏差があったので、ここでは中間値に当てはまる結果を紹介します。その結果は、以下の通りになりますが、あらかじめ結論だけ先に言いますとMVCパターンでもFunctional Endpointでもその性能の違いというものは「誤差範囲以内」と表現しても良いかなと思います。

#### Controller/Serviceの結果

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

#### Router/Handlerの結果

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

## ドキュメンテーション

次にドキュメンテーションの観点からすると、Functional Endpointはまだ導入するには早い気がします。ここでいうドキュメンテーションは、JavaDocやKdocのようなコメントのことではなく、最近よく使われる[Swagger](https://swagger.io)のことを指します。

最近は[SpringFox](http://springfox.github.io/springfox)などを使うと簡単にAPIのドキュメンテーションが可能ですが、Functional Endpointだとそう簡単にはできません。すでに理由がわかる方もいらっしゃるかと思いますが、Routerには引数としてエンドポイントとHandlerの処理を渡しているだけで、Handlerは引数がServerRequest、戻り値はServerResponseに固定されてあるのが理由です。

もちろん、ServerRequestとServerResponseを使う場合でもそれを自分の欲しいデータとして扱うことはできます。例えば、リクエストからパラメータを取る方法は以下のようになります。

```kotlin
// Path Variableで渡されたIDを持ってユーザ情報を取得する
suspend fun getMember(request: ServerRequest): ServerResponse {
    // Path Variableを取得する
    val id = request.pathVariable("id")
    // ...
}

// Request Bodyで渡されたデータを元に新しいユーザを作成する
suspend fun createMember(request: ServerRequest): ServerResponse {
    // Request Bodyをクラスにマッピングする
    val form = request.bodyToMono(MemberCreateForm::class.java).awaitFirst()
    // ...
} 
```

ただ、SpringFoxのように自動でAPIのドキュメンテーションを行ってくれるような便利なAPIは、おそらくリフレクションを使っています。なので、Handlerで実際はどんな処理が行われているかを判断するのは難しいでしょう。

幸い、この問題はSpringの開発者も認識しているようで、[springdoc-openapi](https://springdoc.org)を使うとFunctional EndpointでもAPIのドキュメンテーションは可能になります。ただ、この場合でも現時点ではやはり問題があります。なぜなら、これはAPIのドキュメンテーションを自動化するものではなく、「ドキュメンテーションのための手段を提供する」だけだからです。なので、以下のように、RouterやHandlerに関して一つ一つアノテーションを指定する必要があります。

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

ご覧の通り、ドキュメンテーションのためのアノテーションが実際のコードよりも長くなっています。Functional EndpointでもSwaggerを利用できる手段ができたのは良いことですが、MVCパターンと比べやはり不便ではありますね。なので、ドキュメンテーションが大事であるなら、まだFunctional Pointを使うべきではないかも知れません。

## 最後に

今回は、Spring WebFluxのMVCパターン及びFunctional Endpointをコードの書き方、性能、ドキュメンテーションという観点から比較してみました。Spring WebFluxも発表されたのが2017年なので、もう今年で5年目になりますが、まだまだMVCパターンに比べては色々と補完すべき点が多い印象です。自分の場合はWebFluxのメインコンセプトであるノンブロッキングや関数型プログラミングを活かすためにはやはりFunctional Endpointを選んだ方が良さそうな気はしていますが、まだあえてそうする必要はないのではないか、という感覚です。特にエンタプライズのアプリケーションを書くことになるとますますそうでしょう。もちろん、そもそもWebFluxそのものを導入すべきかということから考える必要がありますが。

それでも、やはりFunctional Endpointという実装方式には色々と可能性があると思います。Spring WebFluxでなくても、最近のウェブフレームワークでは多く採用されているものですからね。例えばTechEmpowerの[ベンチマーク](https://www.techempower.com/benchmarks)でJavaのフレームワークのうちではもっとも性能がよかった[jooby](https://jooby.io)でもMVCパターンとFunctional Endpointとよく似たScript Routeパターンに対応していますし、JetBrainsで開発しているKotlin用のウェブフレームワークである[Ktor](https://ktor.io)ではMVCパターンなしで、同じくFunctional Endpointとよく似たRoutingにのみ対応しています。なので、他にも[Express](https://expressjs.com/ja)や[Gin](https://github.com/gin-gonic/gin)のようなフレームワークでも似たようなAPIの実装方法を提供しているので、余裕があったら個人的に試してみて慣れるのも良い勉強になるかも知れません。また、関数型プログラミングはこれからも幅広く使われそうなので、これを持って練習してみるのも良いかも知れませんね。

では、また！
