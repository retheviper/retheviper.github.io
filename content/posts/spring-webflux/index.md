---
title: "Spring WebFluxって何？"
date: 2020-09-06
categories: 
  - spring
image: "../../images/spring.jpg"
tags:
  - spring
  - webflux
  - nonblocking
---

Springが初めて発表されたのが2002年なので、およそ20年に近い時間が経ちました。今はJavaと言えば当たり前のようにSpringを使っていて、Spring MVCやMaven、propertiesのような煩雑な環境構築と初期設定の問題も、Spring Bootが登場したおかげでだいぶマシになりましたね。特に自分の場合がそうですが、Spring Boot、Gradle、yamlを使ってXMLは一つもないアプリをよく書いていて、なんでも楽と思います。

こうも発展を成し遂げているSpringですが、実は数年前から、そもそものSpring MVCの問題を改善したいという声があり、Spring 5.0からはMVCとは全く違う、新しいフレームワークが登場していました。それが今回紹介します、Spring WebFluxです。

## Spring WebFluxはMVCと何が違うか

最初から作り直したフレームワークなので、根本的な部分から違うところが多いので、理論的なだと以下のキーワードをあげられますでしょう。

- 非同期、ノンブロッキング、反応型

私の(コードモンキーの)レベルからしたら、実際のコードのレベルで体感するMVCとの違いは以下です。

- MonoとFlux
- Controller/Serviceの代わりにFunction
- TomcatよりNetty
- JPA/JDBCよりR2DBC
- 新しい抽象化クラス

では、これらの違いについて、一つ一つみていきたいと思います。

## 理論的な話

### 非同期、ノンブロッキング、反応型

Spring WebFluxは[Project Reactor](https://github.com/reactor/reactor-core)に基づいて作られていて、その根幹となる考え方は[Reactive Stream](https://www.reactive-streams.org)だそうです。Reactive Streamは標準APIであり、Java 9から`java.util.concurrent.Flow`として導入されています。

Reactive Streamはデザインパターン的にはObserverと似ています。簡単にいうと、何かのイベントが発生した時に「通知してくれ」と頼んで、データをもらうということです。この「通知してくれ」と頼む行為のことをSubscription(購読)といい、データを発行するPublisherと購読するSubscriberの間でSubscriptionのやり取りで行われます。こういうイベント基盤のプログラムを作ることが、いわゆる「反応型」だそうです。

そしてReactive StreamではこのSubscriptionのやり取りが、非同期・ノンブロッキングで行われるらしいです。ということは、コードが実行された時点か終わるまでまつ必要がなく、その間に他のことができるということです。なので同じ数の同じタスクを実行するときは同期・ブロッキングと比べあまり性能面での違いはないのですが[^1]、スレッド数がボトルネックとなる場合だと、非同期・ノンブロキングの方が早くなります。

理論的な話は、深くまで踏み入るとキリが無くなるので、実際のコードではどんな違いがあるのか？をみていきましょう。

## コードのレベルでの話

### MonoとFlux

Spring WebFluxだと、コントローラのメソッドに戻り値(レスポンス)としてMonoとFluxを使うということです。Spirng MVCなら、文字列でJSPファイルを指定したり、REST APIならJSONとして返すオブジェクトを指定していましたね。もちろんMonoとFluxもJSONオブジェクトとして出力されるのですが、作り方が少し独特です。

Spring WebFluxの根幹となる考え方がReactive Streamであると先に述べましたね。そしてReactive StreamをWebFlux側で実装したものが、MonoとFluxになります。Monoは`0か1か`、Fluxは`0かNか`の違いだそうですが、必ずしもCollection=Fluxにする必要はなくて、Monoとして扱うことßもできます。

Reactive Streamは、名前からしてJava 1.8のStream APIとなんらかの関係があるようにも見えます。実際、データの作成と消費の時点が違うのですが[^2]、似たような名のメソッドや、Lambdaで完成していくところは似ています。すでに[RxJava](https://github.com/ReactiveX/RxJava)や[JOOL](https://github.com/jOOQ/jOOL)などに慣れているなら、書き方的にはまあり問題なく適応できそうですが、そうでない方には適応が難しいかもしれません。

例えば、GETで、リクエストを受けたら1秒後にレスポンスを返す簡単なメソッドを実装するとしましょう。Spring MVCによるREST APIだと、以下のようになるでしょう。

```java
@GetMapping
@ResponseStatus(HttpStatus.OK)
public String getTest() {
    try {
        Thread.sleep(1000);
        return "task completed by mvc";
    } catch (InterruptedException e) {
        return "task failed by mvc";
    }
}
```

WebFluxでは、以下のようにMonoを作成して返します。

```java
@GetMapping
@ResponseStatus(HttpStatus.OK)
public Mono<String> getTest() {
    return Mono.delay(Duration.ofMillis(1000)).then(Mono.just("task completed by Mono"));
}
```

最近はなんでも宣言的な言語やフレームワークなどが増えているので(例えばFlutterやSwiftUIがそんな感じですね)、こういう書き方は珍しくもないですが、伝統的な命令型プログラミングに慣れている方には少し辛い書き方になるかもしれません。私自身も、StreamやLambdaは好きなものの、ネストしていく命令型とメソッドチェインで長くなる宣言型のどちらがいいか確信がないです…

### Controller/Serviceの代わりにFunction

WebFluxのコード上でのもう一つの特徴は、ControllerとServiceの代替となるクラスを作ることができるということです。もちろん従来通りControllerとServiceクラスを利用することもできますが、どうせなら新しいものが使ってみたくなりますね。

SpringでController/Serviceを作るということは、つまりアノテーションによる「メタプログラミング」に依存して開発するというです。確かにアノテーションは便利であって、Springではアノテーションでなんでもできるような感じもしますね。しかし、アノテーションによる開発では以下のような問題があります。

- コンパイラで検証できない
- コードの行為を規定しない
- 継承、拡張のルールに標準がない
- 理解できない、誤解しやすいコードをを生み出す可能性がある
- テストが極めて難しい
- カスタマイズが難しい

なぜかというと、アノテーションを使うということは結局Reflectionに依存するということになるからです。Reflectionを使っているとランタイムでバイトコードが生成されてしまうので、コンパイルタイムにできることがあまりないですね。Reflectionは確かに強力な道具ですが、他にも問題はあります。例えばパフォーマンスは低下し、デバッグも難しいです。こういう問題があるのでJavaのコードをネイティブにコンパイルしてくれるという[GraalVM](https://www.graalvm.org)ではReflectionに対応していないのかもしれないですね。

とにかく、このような問題を解決するためにWebFluxで新しく導入されたのは、`Function`です。はい、言葉通り、関数です。既存のControllerに対応する`Router`とServiceに対応する`Handler`を作り、関数型モデルとして(Functinalに)コードを書くことができます。もちろんFunctionalに書くとしても、アノテーションは使えます(むしろアノテーションなしではだめです…)。例えば以下のような書き方になります。

```java
@Configuration
public class Router {

    @Bean
    public RouterFunction<ServerResponse> route(final Handler handler) {
        return nest(
                path("/api/v1/web/members"),
                RouterFunctions.route(GET("/"), handler::listMember)
                        .andRoute(POST("/"), handler::createMember)
        );
    }
}

@Component
public class Handler {

    private final MemberRepository repository;

    @Autowired
    public Handler(final MemberRepository repository) {
        this.repository = repository;
    }

    public Mono<ServerResponse> listMember(final ServerRequest request) {
        return ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(Flux.fromIterable(repository.findAll()), Member.class);
    }

    public Mono<ServerResponse> createMember(final ServerRequest request) {
        return ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(request.bodyToMono(Member.class)
                        .flatMap(member -> Mono.fromCallable(() -> repository.save(member))
                                .then(Mono.just(member))), Member.class);
    }
}
```

関数型になってわかりやすくなったような、難しくなったような…

### TomcatよりNetty

Spring WebFluxのデフォルトのアプリケーションサーバはNettyです。簡単に推論できる理由ですが、[Netty](https://netty.io)の方が最初からノンブロッキングという考え方に基づいて作られているからでしょう。Tomcatはもちろん同期、ブロッキングなので、Nettyと比較すると以下のような違いがあるらしいです。

- Tomcat：リクエストとスレッドは1:1
- Netty：リクエストとスレッドはN:1

もちろん、Spring MVCみたいにNettyの代わりにTomcatを使うこともできます。例えばGradleでは以下のような書きます。

```groovy
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-webflux:2.3.3.RELEASE') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-reactor-netty'
    }
    implementation('org.springframework.boot:spring-boot-starter-tomcat:2.3.3.RELEASE')
}
```

### JPA/JDBCよりR2DBC

これもアプリケーションサーバと同じような話です。JPA/JDBCなど、従来のORMはブロッキングなのでノンブロッキングに対応する[R2DBC](https://r2dbc.io)に変えましょう、ということです。NIOがそうであったように、ブロッキングでもR2DBCを使うと場合によってはパフォーマンスの向上を図ることもできるらしいです。

### 新しい抽象化クラス

WebFluxでは、Spring MVCで使っていた抽象化クラスも変わっています。これも同じく、関数型としての書き方とノンブロッキングに対応するためですね。

| 種別 | Spring MVC | Spring WebFlux |
|---|---|---|
| リクエスト | HttpServletRequest | ServerHttpRequest |
| レスポンス | HttpServletResponse | ServerHttpResponse |
| 他のAPIをコール | RestTemplate | WebClient |

WebClientの場合は、RestTemplateが`deprecated`に変更されるので、Spring MVCを使う場合でも導入を考える必要はあります。実際Spring MVCで使う場合でもResttemplateに比べてパフォーマンスが向上される場合もあると言われています。

## どこに使えばいい？

さて、さっくりWebFluxの特徴をみてみましたが、どうでしょうか。書き方もかなり違っていて、Servlet基盤のMVCとは全く違うReactorに基盤しているので、Spring WebFluxの導入はかなり悩ましいことです。実際Spring MVCと同時に使うこともできず(無理矢理Dependencyを追加しても、MVCが優先され、WebFluxの機能は動作しなかったりもします)、Spring Securityなど他のフレームワークもSpring WebFluxのために作りなおったものに変えなくては行けないので、既存のシステムやSpring MVCに基づいて整備したライブラリなどがたくさんある場合はその影響範囲が測定できません。

そして性能面でも、ノンブロッキングが強いのは、「指定されたスレッド数を超えるリクエストがある場合」という条件下の話です。ノンブロッキングに変えたからって、単一スレッドでの性能が上がるわけでもないということですね。[^3]

ただし、以下のような場合はWebFluxの導入を考えられます。

1. 完全新規サービスをはなから作る
1. 複数のサービスがあり、サービス同士での呼び出しが多い場合(マイクロサービス)
1. BFF[^4]の場合

## 最後に

簡単に整理しようとしていた内容が、いつの間にかかなり長くなりましたが…おかげで色々と勉強はできたと思います。WebFluxが登場してからもすでに数年が経っていて、RestTemplateがdeprecatedになる予定であるように、究極的には全体をWebFlux基盤に移行する必要が生じる日がいつか来るかもしれません。なんでも最近は非同期、関数型、反応型というキーワードがすごく流行っていますし。

静的タイプの言語が最初に生まれて、動的タイプの言語も生まれ、TypeScriptのように静的な世界にまた戻るような現象が起きているのを見ると、また関数型から命令型に移行する日もいつか来る可能性があるのかな、と思ったりもします。ただ、こういうパラダイムはどれが絶対というわけではないので、優秀なプログラマならどれも適時適切に使いこなせるようにならないと、という気もします。エンジニアとしての勉強の道は本当に終わりがないですね。

[^1]: 実際は、スレッド数によるボトルネックのない状態だと、関数型の方が少し遅いらしいです。実際は関数型のAPIの実装の方が複雑だからですね。ただ、この違いはコードの読みやすさや実装のしやすさなどを考慮した時は、十分トレードオフとして考慮できるくらいの差のようです。
[^2]: Streamは同期なので、データを生産と消費が一緒に行われます。しかし、Reactive Streamではデータの生産がすぐ消費までつながるとは言い切れません。非同期だからです。
[^3]: むしろ、単一スレッドでの処理は、WebFluxの方がMVCに比べ少し遅いという話もあります。forループに比べStreamが遅い理由と似ているような気がしますが…
[^4]: Back-end For Front-endの略。マイクロサービスの一種で、複数のエンドポイントをまとめて固有のオブジェクトを生み出すバックエンド。フロントエンドは一つのエンドポイントを呼び出すだけでことが済みます。
