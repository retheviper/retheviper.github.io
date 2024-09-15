---
title: "WebFluxのFunctional Enpointに対する小考察"
date: 2021-08-30
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - spring
  - webflux
  - kotlin
  - java
  - rest api
---

前回、[WebFluxではFunctional Endpointを使うべきか](../spring-webflux-router/)というポストを書いたことがありますが、今回は`Controller`/`Service`と`Router`/`Handler`のパターン間の比較ではなく、`Functional Endpoint`を使う場合に、どんな形で実装をしていくべきかについて少し考えたことを述べようと思います。

実際の業務でWebFluxを使っているわけではないので、さまざまなパターンがあるかなとは思いますが、この`Functional Endpoint`を使う場合に考慮すべきものが、`Router Function`(以下`Router`)と`Handler Function`(以下`Handler`)をどう分けるかについての問題かと思います。`Router`と`Handler`は概念的には別のものではありますが、実装としては一つのクラスにまとめでもアプリは問題なく動くので、フレームワークの仕様や思想というよりかは、アプリのアーキテクチャに関する内容に近いますね。

なので、今回は`Router`と`Handler`を分けた場合と分けない場合について、いくつかの観点から考えてみたいと思います。

## RouterとHandlerは分離するべきか

Spring MVCの場合、`Controller`と`Service`を明確に分けるのが常識のようになっています。アーキテクチャとしてもそうですが、フレームワークの思想（デザインの観点）としてもそうですね。

こういう前例があるので、同じくSpring Frameworkに属するWebFluxの場合でも、`Functional Endpoint`という新しい概念を導入するとしても、`Router`と`Handler`を分ける必要があると思いがちかなと思います。一見、`Controller ≒ Router, Service ≒ Handler`という対応関係が成立するようにも見えて、ネットで検索できるサンプルコードも多くがそのような構造で書かれています。

しかし、実際のアプリを`Functional Endpoint`を持って書くとしたら、いくつか考えなければならないことがあると思います。例えば、そもそも`Router`と`Handler`はそれぞれ`Controller`と`Service`に一対一の対応関係であるという前提は確かであるか？もしそうでなければ、あえてMVCのパターンに合わせる必要があるのか？実装においてはどう影響するのか？などがあるかと思います。なので、今回はこれらの観点から`Functional Endpoint`について述べていきます。

## 対応関係について

Springの[公式ドキュメント](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-fn-overview)では、WebFluxの`Functional Endpoint`の紹介において以下のようなサンプルコードを提示しています。

```kotlin
val repository: PersonRepository = ...
val handler = PersonHandler(repository)

val route = coRouter { 
    accept(APPLICATION_JSON).nest {
        GET("/person/{id}", handler::getPerson)
        GET("/person", handler::listPeople)
    }
    POST("/person", handler::createPerson)
}

class PersonHandler(private val repository: PersonRepository) {

    // ...

    suspend fun listPeople(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun createPerson(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun getPerson(request: ServerRequest): ServerResponse {
        // ...
    }
}
```

公式のサンプルとして`Handler`が別のクラスになっているのを見ると、やはり`Controller ≒ Router, Service ≒ Handler`という対応関係が成立するようにも見えます。[@RestController](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/RestController.html)や[@Service](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/stereotype/Service.html)と違って、`@Router`や`@Handler`というアノテーションは存在しないことに注目する必要があります。これはつまり、Springというフレームワークの思想としては`Router`と`Handler`を必ず分ける必要はない、ということを意味しているのではないでしょうか。

なので、少なくともアプリケーションのアーキテクチャという観点からして`Controller ≒ Router, Service ≒ Handler`という対応関係が成立する、という結論を出すのは難しいのではないかと思います。

では、実際`Router`と`Handler`をあえてアノテーションを使ってDIをするとしたら、どうなるのでしょうか。サンプルとしては、以下のような形が一般的かなと思います。

```kotlin
@Configuration
class PersonRouter(private val handler: PersonHandler) {

    @Bean
    fun route(): RouterFunction<ServerResponse> =
        coRouter {
            accept(APPLICATION_JSON).nest {
                GET("/person/{id}", handler::getPerson)
                GET("/person", handler::listPeople)
            }
            POST("/person", handler::createPerson)
        }
}    

@Component
class PersonHandler(private val repository: PersonRepository) {

    // ...

    suspend fun listPeople(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun createPerson(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun getPerson(request: ServerRequest): ServerResponse {
        // ...
    }
}
```

クラスそのものを`@Component`として登録する必要がある`Contoller`に対して、[RouterFunction](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/RouterFunction.html)は`Functional Interface`なのでそれを実装したメソッドを`@Bean`として登録する必要があります。そしてSpringで`@Bean`をアプリケーションに登録するのは一般的に`@Congifuration`が担当するので自然にRouterのアノテーションもそうなります。`Handler`は普通に`@Component`として登録することになりますね。

こうなった場合、クラスやその実装を見て`Router`と`Handler`を分離しているのはわかりますが、アノテーションだけだと違和感を感じられますね。実装は簡単なのでそれぞれに対応するアノテーションを作るのが難しいわけでもないようですが、なぜこのような構造になっているのでしょうか。[公式のドキュメント](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-programming-models)では、以下のような説明があります。

> The big difference with annotated controllers is that the application is in charge of request handling from start to finish versus declaring intent through annotations and being called back.

つまり、「アノテーションをつけたContoller」と「Functional Endpoint」の違いは、前者が「アノテーションでコールバックと意図を表す」に対して、後者は「リクエストのハンドリングを開始から終了まで担当する」ということです。プログラミングモデルとしてこのような観点の差があるので、アノテーションがないのは当たり前なのかもしれません。そして結果的に、`Controller ≒ Router, Service ≒ Handler`という対応関係は、少なくともプログラミングモデルという観点では当てはならないと考えられます。

## 責任の分散という側面で

アノテーションの実装を見ると、`@Controller`と`@Service`を分けているのがフレームワークのアーキテクチャや思想によるものであることがより明確になります。それぞれのアノテーションの実装は、以下の通りです。

```java
@Target(value=TYPE)
@Retention(value=RUNTIME)
@Documented
@Component
public @interface Controller

@Target(value=TYPE)
@Retention(value=RUNTIME)
@Documented
@Component
public @interface Service
```

両方とも実装が同じであるので、極端的にいうと`Controller`に`@Service`をつけても機能的には同一ということになります。そして`@Service`では、以下のようなコメントでこのアノテーションが存在する理由をあくまで「デザインパターンに基盤を置いている」ことを明示しています。

> Indicates that an annotated class is a "Service", originally defined by Domain-Driven Design (Evans, 2003) as "an operation offered as an interface that stands alone in the model, with no encapsulated state."
May also indicate that a class is a "Business Service Facade" (in the Core J2EE patterns sense), or something similar. This annotation is a general-purpose stereotype and individual teams may narrow their semantics and use as appropriate.

なので、アプリケーションデザインの観点からすると`Controller`はリクエストを受信、レスポンスを返す、エンドポイントを`Service`につなぐという義務だけを持ち、`Service`はビジネスロジックを処理する義務を持つと考えられます。同じ観点から考えると、アノテーションはないものの、`Router`と`Handler`もまた同じ義務を持つように書くこともできるでしょう。

ただ、問題は「リクエストのハンドリングを開始から終了まで担当する」という定義です。先程のサンプルコードをよく見ると、Handlerのメソッドはどれも[ServerRequest](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerRequest.html)を引数として、戻り値は[ServerResponse](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/ServerResponse.html)になっています。これはつまり、`Router`と`Handler`をあえて別のクラスとして分割するとしても、リクエストとレスポンスまでを`Handler`で処理することを意味します。

ここで「`Controller`/`Service`の場合と同じく、`Handler`の引数と戻り値だけを変えて良いのでは？」と考えられます。しかし、それこそフレームワークの思想に反することです。`ServerRequest`と`ServerResponse`のJavaDocでは、以下の通り「`ServerRequest`と`ServerResponse`は[HandlerFunction](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/server/HandlerFunction.html)でハンドリングする」ことを明示しています。

```kotlin
/**
 * Represents a server-side HTTP request, as handled by a {@code HandlerFunction}.
 *
 * <p>Access to headers and body is offered by {@link Headers} and
 * {@link #body(BodyExtractor)}, respectively.
 *
 * @author Arjen Poutsma
 * @author Sebastien Deleuze
 * @since 5.0
 */
public interface ServerRequest {

/**
 * Represents a typed server-side HTTP response, as returned
 * by a {@linkplain HandlerFunction handler function} or
 * {@linkplain HandlerFilterFunction filter function}.
 *
 * @author Arjen Poutsma
 * @author Juergen Hoeller
 * @author Sebastien Deleuze
 * @since 5.0
 */
public interface ServerResponse {
```

以上のことでわかるように、WebFluxでは`ServerRequest`と`ServerResponse`は`HandlerFunction`で扱うようにデザインされています。なので、既存の`Service`のように、`Handler`がビジネスロジック「のみ」を扱うというのはそれが実装として可能かどうか以前の問題になるのです。

ただ、「責任の分散」という観点からして、責任によってクラスを分けるという発想は間違っているわけではないですね。なのでビジネスロジックを担当するクラスを`Handler`と分離して運用するケースは考えられますが、必ずしもクラスを分ける基準が`Router`と`Handler`である必要はないのではないかと思われます。

## テストの観点で

JavaでJUnitなどを用いてユニットテストを作る場合、テスト自体はユースケース単位で作成しますが、それらのテストはクラス単位でまとめるというケースが多いかなと思います。なので同じ観点でユニットテストを書く場合、`Router`と`Handler`が分けられているとしたら当然ユニットテストもその単位で分けられるでしょう。

ただ、こうする場合の問題は、テスト自体があまり意味を持たなくなる可能性があるということです。まず`Router`は単純にエンドポイントと`Handler`をつなぐ役割しか持たなくなるので、そのテストも「想定通りの`HadlerFunction`を呼び出しているのか」に限るものになります。そして`Handler`の場合、`ServerRequest`を受信して`ServerResponse`を発するので、テストが非常に難しくなるという問題があります。

なぜ`ServerRequest`を受信して`ServerResponse`を発するのが問題になるかというと、`ServerRequest`のインスタンスを生成するのが難しく、`ServerResponse`の場合でもレスポンスボディーを抽出するのが難しいからです。なので、[WebTestClient](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/reactive/server/WebTestClient.html)で行うことになるかと思いますが、`WebTestClient`を使う場合はエンドポイントとHTTPメソッドなどを利用して実際のAPIを呼び出すことになるので、結果的に`Handler`のテストをするつもりが`Router`のテストまでふくむしかないということになります。こうするとクラス単位でテストケースをまとめることが難しいだけでなく、`Router`のみのテストも実質的には意味をなくすということになります。

## ではどうすればいいか

今まで論じた3つの観点からすると、`Router`と`Handler`は別のクラスにする理由もあまりなく、むしろ別クラスに色々と問題が生じるように見えます。しかし、これが必ずしもエンドポイントに対するルーティングとビジネスロジックを分離する必要はない、ということではないかと思います。先に述べた通り、クラスを分ける基準を`Router`と`Handler`にしないだけで良いかなと思います。例えば、以下のようなコードがあるとします。

```kotlin
@Configuration
class PersonRouter(private val repository: PersonRepository) {

    @Bean
    fun route(): RouterFunction<ServerResponse> =
        coRouter {
            GET("/person/{id}") {
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(
                        repository.findById(it.pathVariable("id").toLong())
                            .map { record ->
                                PersonDto(record.id, record.name)
                            }
                    ).awaitSingle()
                }
        }
}
```

`Handler`で、`Body`を作る箇所以外はビジネスロジックと言えるものがあまりありません。なので、ここでは`Body`だけを分離して別のクラス（`Service`）に一任しても良さそうです。例えば以下のようにです。

```kotlin
@Configuration
class PersonRouter(private val service: PersonService) {

    @Bean
    fun route(): RouterFunction<ServerResponse> =
        coRouter {
            GET("/person/{id}") {
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(service.getPersonById(it.pathVariable("id").toLong()))
                    .awaitSingle()
                }
        }
}

@Service
class PersonService(private val repository: PersonRepository) {

    suspend fun getPersonById(id: Long): Mono<PersonDto> =
        repository.findById(id)
                  .map { PersonDto(it.id, it.name) }
}
```

こうすると、`Router`から直接`Repository`にアクセスこともなくなり、今まで挙げていたさまざまな問題も解消できるようになりますね。

## 最後に

ここで提示した方法でビジネスロジックを分けるのは可能だとして、その方法をとった場合に残る疑問はあります。これは果たして`Functional`なのか？`Functional Endpoint`は`Lambda-based`と説明されてあるが、`Lambda`が使われないので設計意図とは違う形になってないか？そもそもSpring MVCとは違うコンセプトのフレームワークなので既存とは違うアプローチが必要なのでは？などなど。

これらの問題を判断するのはなかなか難しいですが、個人的には新しい技術だからといって常に新しい方法論を適用するということは難しく、既存の良い体系があるのならそれに従うのもそう間違っていることとは思いません。Springの公式ドキュメントでは「すでに問題なく動いているSpring MVCアプリケーションにあえてWebFluxを導入する必要はない(If you have a Spring MVC application that works fine, there is no need to change)」と述べていますが、これと同じく、既存の検証されてあるアーキテクチャがあるのならばそれをそのまま適用するもの悪くないのではと思います。まぁ、そもそもWebFluxを導入するところでMVCパターンを使うとしたらこういうことを気にする理由すら無くなるのですが…むしろこのようなプログラミングモデルが増えていくと今後は新しいアーキテクチャが生まれそうな気もしますね。今回のポストを書きながらはそういういうものを感じました。

では、また！
