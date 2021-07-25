---
title: "Ktorを触ってみた"
date: 2021-07-18
categories: 
  - kotlin
photos:
  - /assets/images/sideimage/kotlin_logo.jpg
tags:
  - kotlin
  - ktor
  - exposed
---

サーバサイド言語としてのKotlinは普及しつつありますが、Kotlinを使う場合でもウェブフレームワークとして使われるのはやはりSpringが多いかと思います。理由としては会社ごとの事情というものもあるはずですが、一般的な理由としてはJavaエンジニアにとってKotlinは馴染みやすい物であっても、フレームワークの場合はそうでなく、Springほど検証されたフレームワークはないからということからでしょう。いまだにStrutsを使っていて、Springに移行しようとするところもありますしね。

KotlinはJavaと完璧（に近い）互換性があるので、Javaで書かれてあるアプリをそのままKotlinに移行しても大した問題はありません。Javaより生産性が高い上にSpringだけでなくJackson、Apache POI、JUnit、Jacocoなどの数多くのライブラリをそのまま使えるのは確かにメリットであって、企業側としてKotlinの導入を検討する理由は確かにそこにあると思います。Javaエンジニアはその数が多いので、エンジニアを募集し安くなるというところもメリットの一つと言えるでしょう。

ただ、Kotlinを使う場合に長期的にはKotlinで書かれたライブラリやフレームワークを導入することを検討した方が良いかもしれません。コンパイルした結果として生成されるByte codeがJavaと全く一緒だとしても、そもそものソースコードが違う言語なので、使う側のコード（クライアントコード）としては不便なところがあったり、Kotlinに最適化されてない場合もある可能性があるからです。また、KotlinはJVMだけでなく、ネイティブにコンパイルすることもできるので、ネイティブなアプリを作りたい場合はJavaに依存しないAPIを選ぶ必要があるでしょう。

ということで、今回はJetBrains制のウェブフレームワーク、Ktorと、Ktorと一緒に使えるORMのExposedを少し触ってみて、Springと比べながら紹介したいと思います。

## Ktor

Ktorは、JetBrainsで開発しているマイクロサービス向けの軽量ウェブフレームワークです。[公式ホームページ](https://ktor.io)の紹介にも色々書いてありますが、特にSpringと比べて以下の特徴があるかと思います。

### 軽量

Springも軽量とは言われているものの、起動が遅いので、実装する側としてはあまり軽量だという感覚はないです。Springで書かれたアプリケーションの起動が遅いのは、起動時にさまざまなアノテーションを読み込み、DIや設定などを完璧に終わらせているというフレームワークそのもののアーキテクチャに起因しているのではないかと思います。なのでDIされるオブジェクトをLate initにするなどで起動速度を短縮させるテクニックなどが知られていますね。

しかし、Ktorは起動がかなり早いです。同一規模のアプリをSpringとKtorの両方で作成してベンチマークした訳ではないので正確な数値に関しては割愛しますが、体験だと数倍は早いですね。例えば、In memoryタイプのH2と基本的なCRUDを実装したSpring WebFluxアプリケーションの場合、自分のPCで起動に2.7秒ほどかかりました。

```bash
2021-07-18 15:08:25.150  INFO 29047 --- [main] c.r.s.SpringWebfluxSampleApplicationKt   : Started SpringWebfluxSampleApplicationKt in 2.754 seconds (JVM running for 3.088)
```

同じ構成でKtorのアプリを実装した場合、起動には1秒もかからなかったです。

```bash
2021-07-18 15:09:29.683 [main] INFO  ktor.application - Application started in 0.747 seconds.
```

これはおそらく基本的にDIをしなく、アノテーションをあまり使わない（Reflectionを使わない）構造やKtorそのものはREST APIを作成するための必要最低限の機能だけを揃っているのが理由かと思われます。

アプリの起動が早いというのは、テストにかかる時間を短縮させられるという面でもメリットといえますが、サーバレスなアプリにも適しているということにもなるでしょう。私もAWSのLambdaやAzureのFunctionsなどを触った経験がありますが、この場合にJavaやKotlinの使用を考慮したことはありません。サーバレスの場合、アプリが常に稼働中ではないので、リクエストが発生したたびにアプリを起動しなければならないです。なので起動の遅いSpringはそもそもの考慮対象にならなかったですね。Ktorを使う場合は起動速度が大幅に短縮できるので、JVMの起動速度が許されるというならば、サーバレスアーキテクチャで導入を検討できるレベルになっていると思います。

### 拡張可能

Ktorが軽量であることとも繋がる話ですが、必要な機能があればプラグイン（モジュール）を追加したり、自分で実装する必要はあります。コードとしては、以下のようになります。

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

なので、Ktorの導入直後はモジュールの管理や開発のスピード感という側面ではマイナスになる部分もあるかなと思います。特にまだSpring Securityでは基本的に提供している[Role-Based Authorization](https://ja.wikipedia.org/wiki/%E3%83%AD%E3%83%BC%E3%83%AB%E3%83%99%E3%83%BC%E3%82%B9%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E5%88%B6%E5%BE%A1)などの機能が公式プラグインとして提供されてないので自前の処理を書くしかないという部分もあります。個人的には、モジュール化そのものは慣れたらメリットになる可能性の方が高いと思いますが、導入初期としてはSpringに比べ不利なところなのではないかと思います。

特にKtorはDIに対応していなく、JetBrains公式のモジュールもないので、DIをするためには[Injekt](https://github.com/IVIanuu/injekt), [Kodein](https://kodein.org/Kodein-DI/?6.3/ktor), [Koin](https://insert-koin.io)などをディペンデンシーとして追加する必要があります。ただ、アーキテクチャによってはDIが必要なく、`object`で代替することもできると思いますので、どんなアーキテクチャにするかはよく考えて決める必要があるかなと思います。

### Coroutine対応

Spring WebFluxもそうでしたが、最近は多くのウェブフレームワークに非同期・ノンブロッキング対応が行われていますね。PaaSが普及され簡単にインフラの構築ができ、ハードウェアが安くなった今でもソフトウェアで性能を改善できる箇所があるならそれは十分価値があると思っています。だとすると、非同期・ノンブロッキング対応のフレームワークを導入するということも良い選択ではないかと思います。

Ktorではルーティングの実装として、[Route](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/-route/index.html)の[route](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/route.html)、もしくは[get](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/get.html), [post](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/post.html), [put](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/put.html), [delete](https://api.ktor.io/ktor-server/ktor-server-core/ktor-server-core/io.ktor.routing/delete.html)などのfunctionを呼び出すことになります。これはSpring WebFluxのRouter/Hanlder Functionとよく似ていますね。コードで表すと、以下のようなものです。

```kotlin
routing {
    get("/hello") {
        call.respondText("Hello")
    }
}
```

そしてこのHttpメソッドごとの関数のbodyを実装することになりますが、これが基本的に`suspend`となっています。これはつまり、実装する側で特に意識しなくてもコードは非同期になるということですね。Spring WebFluxの場合も、Coroutineを使うと簡単に実装ができましたが、`suspend`すら意識しなくて良いというところはKtorならではのメリットなのではという気がします。

### テスト

`ktor-server-test-host`や`kotlin-test`、JUnitなどを使ってテストが可能です。Springでもユニットテストは色々な書き方があるかと思いますが、よりKotlinらしき書き方になっているだけで、基本的にテストの仕方が大きく変わったりはしません。例えば、`Get`をのレスポンスをテストするためには以下のようなコードを書くことができます。

```kotlin
@Test
fun getMember() {
    withTestApplication((Application::module) {
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

Ktorで使える、Kotlinで書かれたORMは代表的に[Exposed](https://github.com/JetBrains/Exposed)があります。Javaの[jOOQ](https://www.jooq.org)がそうであったように、SQL DSLを使うことでクエリをコードで書くような感覚で（実施はDSLを解釈してSQLは自動生成されますが）使えるというところが良いです。例えば、Userというテーブルからレコードを取得する場合のコードは、以下のようになります。

```kotlin
List<User> = userInUsa = transaction {
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

また、ExposedでははDAOパターンも使えるので、DAOパターンでクエリを書くとしたら以下のようなことができます。JPAやR2DBCと似たような感覚で使えそうですね。(デメリットもおそらく同じかと思いますが)

```kotlin
List<User> userInGermany = transaction {
    User.find { (UserTable.country eq Country.GERMANY) and (UserTable.deleted eq false)}
}
```

また、Exposedの特徴は、テーブルをコードとして定義することでDBに反映させることができるということです。今まで[Liquibase](https://www.liquibase.org)や[Flyway](https://flywaydb.org)でDBの形状管理をやっていたことが多かったのですが、個人的に実際のDBとアプリケーションのテーブル定義に乖離があるケースを考えるとこうやってコードの中に定義した方が、データのオーナーという観点からもかなり良いのではないかと思います。特に、頻繁なテーブル定義の修正があったり、マイクロサービスが多いケースではかなり開発が便利になるのではないかと思います。

Exposedのテーブル定義は、以下のようにできます。

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

そして実際発行されるSQLは以下のようになります。

```sql
CREATE TABLE IF NOT EXISTS "MEMBER" (ID INT AUTO_INCREMENT PRIMARY KEY, DELETED BOOLEAN NOT NULL, CREATED_BY VARCHAR(16) NOT NULL, CREATED_DATE DATETIME(9) NOT NULL, LAST_MODIFIED_BY VARCHAR(16) NOT NULL, LAST_MODIFIED_DATE DATETIME(9) NOT NULL, USER_ID VARCHAR(16) NOT NULL, "NAME" VARCHAR(16) NOT NULL, PASSWORD VARCHAR(255) NOT NULL)
```

ここで、JPAやR2DBCの場合、Auditableクラスを定義して、エンティティがそれを継承することでカラムを共有したり、Spring Securityに連携することができましたが、Exposedでも似たようなことができました。

```kotlin
abstract class Audit : IntIdTable() {
    val deleted: Column<Boolean> = bool("deleted")
    val createdBy: Column<String> = varchar("created_by", 16)
    val createdDate: Column<LocalDateTime> = datetime("created_date")
    val lastModifiedBy: Column<String> = varchar("last_modified_by", 16)
    val lastModifiedDate: Column<LocalDateTime> = datetime("last_modified_date")
}

object Member : Audit() { // Auditのカラムも含めてテーブルが作成される
    val userId: Column<String> = varchar(name = "user_id", length = 16)
    val name: Column<String> = varchar(name = "name", length = 16)
    val password: Column<String> = varchar(name = "password", length = 255)
}
```

MyBatisなどに慣れている場合は少し適応に時間が必要かもしれませんが、基本的にはテーブルの定義を除くとほぼSQLの発行をKotlinのコードで書くことになるという感覚なので、便利になるかと思います。

## 最後に

以上で、簡単なCRUDアプリをKtor + Exposedで実装してみた後の感想と紹介を少し書いてみました。まとめると、かなりサクサクコードを書けて性能も良いので、マイクロサービスに特化している構成ではないかと思いました。また、冒頭に述べた通り、ピュアなKotlin制のフレームワークであることも良いですね。Ktorの紹介でもKotlin Multiplatformに基づいていてどのプラットフォームにもアプリをデプロイできると強調していますので、色々なところで活用ができるかと思います。

まだSpringと他のJavaライブラリに比べ足りないモジュールや機能もありますが、Exposed以外でも[Ktorm](https://www.ktorm.org)のようなORMがあるなどKotlin制のライブラリの開発も進めていて、IntellijでもKtorのサポートは強力なので今後も発展を期待できそうであります。個人的にまだ仕事で使うことには無理があっても、自作アプリなどを作りたい時は導入をぜひ検討したいと思いました。Kotlinでできることがだんだん増えてきていて、嬉しいですね。

では、また！
