---
title: "Quarkusを触ってみた"
date: 2021-10-24
categories: 
  - quarkus
photos:
  - /assets/images/sideimage/quarkus_logo.jpg
tags:
  - java
  - kotlin
  - ktor
---

Spring MVCは良いフレームワークではありますが、最近流行りの[マイクロサービス](https://ja.wikipedia.org/wiki/%E3%83%9E%E3%82%A4%E3%82%AF%E3%83%AD%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9)には向いてないという批判もあります。理由としては、アプリの起動時間が遅い、サイズが大きい、メモリの使用量が多いなどの問題が挙げられていますね。アプリの起動速度が遅い場合は、変更があった場合の素早い反映が期待できません。アプリのサイズが大きいのとメモリ使用量が多いとインスタンスが増えれば増えるほどコストが高くなるということになりますね。また、これはマイクロサービスだけの話でもないです。サーバレスアプリで、あえてJavaScirptやPythonのようなインタプリタ言語を採用しているのも同じ理由があってこそですね。

では、これらの問題はどうやって回避できるのでしょうか。そもそもの問題から考えると、全てが全てSpringに局限する問題でもないはずです。他のフレームワークに比べてSpringの起動時間が決して早いとは言えなかったり、メモリの使用量が多いという問題があるのは確かですが、JVMをベースにしている言語を使う限り、ある程度は仕方ない問題にも見えます。JVM言語においてはそのアプリの起動時間、サイズ、メモリ使用量のどれにもJVMが占める割合を含めて考える必要があるからですね。

ただ、これらの問題を解決できる方法が全くないわけでもありません。今回紹介するのが、その答えとして開発されている[Quarkus](https://quarkus.io)です。

## Quarkusとは

Quarkusは、[RHEL](https://www.redhat.com/ja/technologies/linux-platforms/enterprise-linux)で有名なRed Hatが作ったJava用のウェブフレームワークです。公式ホームページの説明が何よりも正確だと思いますので、以下の文を確認してください。

> A Kubernetes Native Java stack tailored for OpenJDK HotSpot and GraalVM, crafted from the best of breed Java libraries and standards.

Javaのアプリを`Kubernetes Native`として作成できる、というのがこのフレームワークの正体性です。Javaと説明していますが、もちろんKotlinのような他のJVM言語も使えるので、そのような言語を使っている場合でも導入を考えられます。

ここで`Kubernetes Native`という言葉が気になりますが、これは単純にコンテナを作ることに特化されている、という表現ではないと思います。Spring Boot 2.3から導入された[Docker Image作成機能](https://spring.io/blog/2020/08/14/creating-efficient-docker-images-with-spring-boot-2-3)があり、Googleが提供している[Jib](https://github.com/GoogleContainerTools/jib)のようなライブラリでいくらでもJavaアプリケーションをコンテナ化することができますし、そのほかにもコンテナを作る方法はいくらでもあります。なので、ここで`Kubernetes Native`という表現をあえて使っているのは、Kubernetesに特化したものとして設計されているということを意味すると思った方が自然でしょう。

では、一体何を持って`Kubernetes Native`と言えるのでしょう。インフラストラクチャの観点でいう`Kubernetes Native`は、Kubernetesだけで完結するアーキテクチャを指しているようです。Kubernetesで完結するということは、それに合わせて最適化しているということと同じ意味合いでしょう。アプリケーションの観点からしてもそれは大きく違わないはずです。まず、Quarkusでは以下のような特徴があると紹介されています。

- Nativeコンパイルができる
- 起動速度が速い
- メモリ使用量が少ない

Nativeコンパイルができるということは、JVMを使用する必要がなくなるということなので、先に挙げた三つの問題を全部解消できます。だとすると、マイクロサービスやサーバレスのみでなく、コンテナ単位でのデプロイでもかなり有利になりますね。そして、JVMを使った場合でも他のフレームワークに比べて起動速度とメモリ使用量で優位にあると言われているので、これが本当だとNativeコンパイルしない場合でも十分メリットがあると思われます。

## 実際触ってみると

特徴として挙げられているもの全てが魅力的ではありますが、実際そのフレームワークを使ってみないとわからないこともあります。なので、ちょっとしたサンプルを作り触ってみました感想について少し述べたいと思います。

### 起動速度

#### Spring Bootの場合

Springの起動速度は、DIしているクラスによって大きく異なるので、ここでは[Spring initializr](https://start.spring.io)から以下の項目のみ設定したアプリケーションを使って起動してみました。

- Project: Gradle
- Language: Kotlin
- Spring Boot: 2.5.5
- Packaging: War
- Java: 11
- Dependencies: なし

そしてローカルでは、Oracle JDK 17を使って起動しています。気のせいかも知れませんが、Java 11を使っていた時より起動が早いような気がしますね。とりわけ、上記通りの設定を済ましたアプリを起動してみると、以下のような結果となりました。（ローカルマシンの情報は消しています）

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

JVMの起動に1.396秒、アプリの起動に1.129秒がかかっていますね。何も依存関係がないので、おそらくこれが自分のマシンでは最速の起動時間と言えるのではないかと思います。これが実際の業務用のアプリとなると、アプリの起動だけで10秒以上かかることもあリますね。一回の起動では10秒でもあまり問題になることはありませんが、ローカルでのテストではテストごとにアプリが起動するような

#### Quarkus Nativeの場合

では、Quarkusの場合を見ていきたいと思います。まずNativeコンパイルができるというので、[GraalVM](https://www.graalvm.org/)を利用してビルドしてみました。実際のビルドはGradleのタスクとして実行できて（固有のパラメータは必要ですが）、簡単です。そしてそれを実行してみた結果が以下です。

```bash
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2021-10-23 19:24:18,395 INFO  [io.quarkus] (main) quarkus-sample 1.0.0-SNAPSHOT native (powered by Quarkus 2.3.0.Final) started in 0.018s. Listening on: http://0.0.0.0:8080
2021-10-23 19:24:18,397 INFO  [io.quarkus] (main) Profile prod activated. 
2021-10-23 19:24:18,397 INFO  [io.quarkus] (main) Installed features: [cdi, config-yaml, kotlin, resteasy-reactive, resteasy-reactive-jackson, smallrye-context-propagation, vertx]
```

0.018秒がかかっています。ビルドしたプロジェクトの構造が単純であるのもありますが、それでもこの起動速度は確かに速いですね。これなら確かにマイクロサービスだけでなく、リクエストの多いサーバレスアプリケーションでも十分使えると思います。

#### Quarkus JVMの場合

Springの場合と同じく、Oracle JDK 17を利用して起動してみました。Quarkusには開発モードというものがあり、サーバを起動したまま修正ができるのですが、ここではあえてJarを作って起動しています。余談ですが、Springでは依存関係を全部含む場合はwarになりますが、Quarkusではuber-jarと言っているのが面白いです。

```bash
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2021-10-23 19:20:59,897 INFO  [io.quarkus] (main) quarkus-sample 1.0.0-SNAPSHOT on JVM (powered by Quarkus 2.3.0.Final) started in 0.761s. Listening on: http://0.0.0.0:8080
2021-10-23 19:20:59,905 INFO  [io.quarkus] (main) Profile prod activated. 
2021-10-23 19:20:59,906 INFO  [io.quarkus] (main) Installed features: [cdi, config-yaml, kotlin, resteasy-reactive, resteasy-reactive-jackson, smallrye-context-propagation, vertx]
```

今回は0.761秒がかかりました。Nativeと比べると確かに数十倍も遅くなっていますが、それでもSpringと比べ速い方ですね。

こうやってアプリの起動が速くなると、ローカルで開発するときもユニットテストが早くなるので即座で確認ができるというメリットもあるかと思います。特にSpringで[RestTemplate]や[WebTestClient](https://spring.pleiades.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/reactive/server/WebTestClient.html)などを使ったテストケースを書くとテストごとにアプリを起動することになるので、テストケースが増えれば増えるほどかかる時間が多いのが辛いものですね。なので、起動が速いと同じようなテストをQuarkusで書いてもかなり時間が節約できそうです。

### Springからの移行が簡単

最初あまり意識してなかった部分ですが、Quarkusの良い点の一つは、Springからの移行が簡単ということです。アプリを新規に開発するときや既存のアプリのフレームワークを変更する場合には、技術選定において色々と考慮すべきものや観点があると思いますが、その中でいくつかを取り上げると「いかに工数を減らせるか」、「エンジニアを募集しやすいか」などがあるのはないかと思います。こういう観点からすると、現在のエンジニアにとって全く新しい技術だったり、業界であまり使われてない技術だったりすると会社としてもエンジニアとしても大変でしょう。こういう問題があるので、企業にとって新しい技術の導入は難しくなっていると思います。

なので、新しい技術でありながらも業界でよく使われているものと似ているという点は、エンジニアの学習コストを減らせるのでかなりのメリットと言えるでしょう。では、実際のコードを観ながら、SpringのコードをQuarkusに移行するとした場合はどうなるかを見ていきたいと思います。

#### Springの場合

まず、クエリパラメータにIDを渡し、Personというレコードのデータを取得するAPIがあるとしましょう。Springなら、以下のようなコードになるかと思います。

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

#### Ktorの場合

Quarkusのコードを見る前に、まず同じコードをこのブログでも紹介したことのある[Ktor](https://ktor.io/)で書くとどうなるかをまず見ていきたいと思います。これでSpringと全く違うフレームワークを選ぶという場合の比較ができるでしょう。

Ktorもよいフレームワークではありますが、フレームワークそのものの設計思想はSpringと異なるので、既存のアプリを移行するとしたら色々と考慮すべきものが多いです。例えば、基本的にDIに対応していないのでライブラリを別途導入する必要がありますね。

以下は、上記のSpringと同じAPIを、DIライブラリとして[Koin](https://insert-koin.io/)を導入して実装したKtorの例です。かなり違う構造になっているのがわかります。

```kotlin
fun main() {
    // DIの設定
    val personModule = module {
        single { PersonService(get()) }
        single { PersonRepository() }
    }

    // Koinをアプリにインストール
    install(Koin) {
        modules(personModule)
    }

    // ルーティングをモジュール化して設定
    configureRouting()
}

// ルータにControllerを登録
fun Application.configureRouting() {
    routing {
        personController()
    }
}

fun Route.personController() {
    // Serviceのインジェクション
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

#### Quarkusの場合

では、続いてQuarkusでAPIを作成した場合のコードを見ていきましょう。QuarkusでAPIを作成する方法は[RESTEasy](https://resteasy.github.io)と[Reactive Routes](https://quarkus.io/guides/reactive-routes)の二つのパターンがありますが、どちらを使った場合でもアプリの作成そのものに大きい違いはないので、ここではRESTEasyを使った実装を紹介したいと思います。まずは以下のコードをご覧ください。

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

Springのコードど比較して、使っているアノテーションの種類が違うだけで、ほぼ同じ感覚で実装ができるのがわかります。なので、Ktorの場合のようにアーキテクチャを考え直す必要もなく、移行も簡単になるわけですね。また、RESTEasyを使う場合、ReactiveのAPIを作りやすいというメリットもあります。Reactiveだと[Mutiny](https://smallrye.io/smallrye-mutiny/)を使うことになりますが、Uni/Multiの概念がMono/Fluxと1:1対応していると思って良いので、Spring WebFluxや他のReactive Streamを触ったことのある方ならすぐに適応できそうです。

```kotlin
@Path("/api/v1/person")
class PersonController(private val service: PersonService) {
    @GET
    fun getPerson(id: Int): Uni<PersonResponse> {
        return Uni.createFrom().item(service.getPerson(id).let { PresonResponse.from(it) })
    }
}
```

KtorやSpring WebFluxのRouter Functionのような書き方もそれなりの良い点はあるかと思いますが、やはりSpring MVCのような書き方に慣れている人も多いだろうし、そのような書き方で特に問題になることもないので、新しいフレームワークだとしてもこのように既存のものと同じような感覚でコードを書けるというのもそのフレームワークを選択しやすくする一つのセールズポイントになるのではないかと思います。例えば[NestJS](https://nestjs.com/)のように、JavaScript用のフレームワークでもSpring MVCに似たようなコードを書けるのですが、おそらくこれもまたSpringを触った経験のあるエンジニアにアピールするためでしょう。

こういう面からすると、すでにSpringの経験があるエンジニアならすぐにQuarkusに移行できて、既存のSpringプロジェクトも簡単に移行できそうなので良さそうです。

## 懸念

Quarkusを実際触ってみて、最も良いと思われたのは上記の通りですが、Nativeでアプリをビルドしながら、いくつかの懸念もあると感じました。例えば以下のようなものがあります。

### Nativeのビルドは遅い

Nativeで起動速度が早くなるのは確かに良いところですが、問題はビルド自体は遅いということです。当然ながら、Nativeとしてビルドするということは、最初から全てのコードをマシンコードとしてコンパイルするということを意味します。JVM用のバイトコードはどの環境でも同じですが、マシンコードはそうではないので、そのプラットフォームに合わせたコードを生成するのに時間がかかるのは当然のことですね。例えば、ローカルでテストに使ったプロジェクトをNativeイメージとしてビルドした場合は以下のような時間がかかりました。

```bash
$ ./gradlew build -Dquarkus.package.type=native

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

CIでビルドを行っていたり、頻繁にアプリの修正とデプロイが必要な場合にこれでかなりボトルネックになる可能性もあるかと思います。マシンパワーが十分か、デプロイまでの時間があまり気にならない場合は問題にならないと思いますが、起動速度が大事であるなら、その分ビルドに時間がかかると結局は同等のトレードオフになるだけですね。こういう場合はJarとしてビルドする時間や、他のフレームワークを使ってビルド〜起動までにかかる時間を測定してから判断した方が良いかなと思います。

### ピークパフォーマンス

一般的にCやC++のような言語と比べ、Java(JVM言語)は性能で劣るという話は常識のように受け入れられています。しかし、全ての状況においてそういうわけでもありません。適切なアルゴリズム、アプリケーションのデザインなど言語そのものとは無関係と言えることが理由な場合もありますが、言語の特性を考えてもそういうケースがあるということです。なぜなら、CやC++のようなネイティブコードを生成する言語と、JVM言語のコンパイラの特性が違うからです。

仮想マシンを挟み、バイトコードをマシンコードにもう一度変換する必要があるJVM言語と比べ、最初からマシンコードを生成する言語の方が性能が優秀であることは当然です。実際それは数値としても表れていて、Javaが登場した当時には性能問題で色々と批判を受けていたらしいですね。今はJavaが比較的性能が大事であるサーバサイドアプリケーションを作成する場合によく採用されていますが、これも「ハードウェアの発展がある」からと言われるケースも多いです。

ただ、全ての場合においてJVMを挟むアプリケーションがNativeより遅いわけでもありません。なぜなら、コンパイルには「最初から全てコンパイルしておく」[AOT](https://ja.wikipedia.org/wiki/%E4%BA%8B%E5%89%8D%E3%82%B3%E3%83%B3%E3%83%91%E3%82%A4%E3%83%A9)だけでなく、「必要に応じてコンパイルする」[JIT](https://ja.wikipedia.org/wiki/%E5%AE%9F%E8%A1%8C%E6%99%82%E3%82%B3%E3%83%B3%E3%83%91%E3%82%A4%E3%83%A9)の方式もあるからです。

JVMではJITによりバイトコードの分析と最適化を行い、マシンコードを生成することでより良い性能のコーどを作り出すと言われています。ここで最適化とは、利用頻度の高いメソッドや定数などを含めてオーバヘッドを減らすことを含みます。このような最適化が行われたコードをマシンコードに変換するとしたら、当然性能がより良くなることを期待できますね。ただ、ITは全ての場合に動作してくれるわけでもありません。コンパイルには時間とマシンパワーが必要なので、一度しか利用されないコードをいちいちマシンコードに変換するのは無駄なことですね。なので、JITでコンパイルされるコードは、そのコードの利用頻度により決定されます。よくJavaのマイクロベンチマークで使われている[JMH](https://github.com/openjdk/jmh)でテストを行うとき、事前にウォーミングアップをおこなっているのも、JITによる最適化でベンチマークの精度を上げるための工夫です。

#### 検証してみると

実際NativeかJVMかによってどれくらいランタイム性能が違うのか気になったので、ループで10万件のデータを作って返すだけのServiceを作成して処理時間を計測してみました。ここでControllerの戻り値に[Multi](https://smallrye.io/smallrye-mutiny/getting-started/creating-multis)を使ったせいか、APIが呼び出されるたびにレスポンスまでの時間が大きく変化していたので、測定しているのはリクエストからレスポンスまでの時間より「forループによるデータの生成にかかった時間」を計測していると理解してください。

NativeビルドとJarの実行に使ったのはどれもGraalVM CE 21.3.0(OpenJDK 11)で、処理時間の測定はKotlinの[measureTimeMillis](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.system/measure-time-millis.html)で取得した値をログに吐くという方法を使っています。

まずNativeで起動した場合の結果です。

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

また、以下がJVMで起動した場合の結果です。

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

やはりJITが関与しているせいか、JVMでは最初の実行で時間がかかっていて、その次から大幅に処理速度が早くなっているのがわかります。GraalVMのコンパイラのバージョンアップでさらにパフォーマンスが向上する可能性はあると思われますが、それはJVMの場合でも同じなので、どうしてもランタイムのピークパフォーマンスが大事な場合はJVMの利用を考慮しても良いかなと思います。

## 最後に

本当はメモリ使用量などをより正確に測る必要があると思いますが、それに関してはすでに[記事があったので](https://medium.com/swlh/springboot-vs-quarkus-a-real-life-experiment-be70c021634e)、ここでは割愛します。結論から言いますと、確かにメモリ使用量はヒープを含めQuarkusの方が少ないですが、CPU使用量の最大値とLatencyにおいてはSpring Bootの方が優れているのを確認しました。ただ、ここはQuarkusの方が歴史が短いためであるということもありそうですね。

とりあえず触ってみた感覚では、確かにKubernetes nativeと言えるだけのものではあると思われます。Nativeビルドしてみると、Jarと比べアプリのサイズ自体は大きくなるものの（倍ほど）、JDKがいらなくなるというのも良いですね。JDKのサイズはAdoptOpenJDKを基準におよそ300MBくらいです。インスタンスが一つの場合だとしたらあまり問題になりそうではないですが、もしインスタンスが増えるとしたらJDKだけでも必要なストレージのサイズが乗数で増えることになるので、Nativeにしたくもなるかなと思います。

そのほかにも、さまざまなライブラリやフレームワークの組み合わせができるし、Spring Securityなどをそのまま用いることができるのも魅力的です。Springの経験のあるエンジニアなら誰でもすぐに慣れそうなので、会社の立場からも他のフレームワークを使う場合に比べ比較的エンジニアの募集に負担がなくなるのでは、と思ったりもします。

Spring WebFluxやKtorもよかったのですが、また新しい強者が現れてどれを使うか悩ましい時代になりましたね。本当は[Rocket](https://rocket.rs/)も触ってみたいんですが、果たして今年内にできるかどうか…

では、また！
