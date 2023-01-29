---
title: "Spring Boot 3を導入してみた"
date: 2023-01-29
categories: 
  - spring
image: "../../images/spring.jpg"
tags:
  - kotlin
  - java
  - spring
---

Spring Boot 2 (Spring Framework 5) から Spring Boot 3 (Spring Framework 6) にアップデートしてみました。

## Java 17

仕事でJavaを使っていたごろは、Java 17をずっと待っていました。以前こちらのブログの記事として整理したこともありましたがテキストブロックやSwitchを式で使えるなど便利な機能がたくさん追加されたためです。しかし、Kotlinを使っている今は機能や言語仕の観点からJavaのバージョンを機にする必要はありません。なので、既存のアプリケーションがJava 11で動いているのであれば、Java 17に上げるモチベーションはあまりないように見えるかもしれません。

それでもあえてバージョンを上げようとした理由は、まずサポート期間が今年の9月で終わるからということになります。正確にはJava 11のPremier Supportが2023年9月で終わり、それに準するExtended Supportは2026年9月までということになります。ただ、「準する」という表現が曖昧で、11は今の時点ではもうだいぶ古く、17もリリースから1年以上の期間の間に十分検証されていると判断しました。

サポート期間の話だけだと、まだ終了まで半年以上の時間が残っているのですが、このタイミングで行うのは会社で「リファクタ期間」というものを設けているためです。この期間中は主に技術負債の解消や依存関係のバージョンアップなどを重点的に行うので、新しい機能の開発とバージョンアップが重なり問題が起こるような事項は避けたいと思いました。

そのほかでも、新しいJavaのバージョンを採用することでパフォーマンスの向上を図ることができる点があります。例えば、[Java 11よりGCの性能が改善されていたり](https://www.optaplanner.org/blog/2021/09/15/HowMuchFasterIsJava17.html)、状況によって[ZGCの導入を考慮できる](https://malloc.se/blog/zgc-jdk17)オプションができたりです。Javaのバージョンが上がるということは、JVMの改善を含むということになるので、Kotlinでも十分その恩義を受けられることになるでしょう。

なので、Java 17を採用することで決め、以降時の経験に関して共有したいと思います。

### Record

社内ではORMとして[jOOQ](https://www.jooq.org/)を採用していて、これから自動生成されたテーブル定義のコードをKotlinのコード(Repository)で呼び出す構造となっています。ただ、場合によってはこの自動生成のコードによりJava 17でのコンパイルが失敗することがあります。今回の場合は、以下のようなエラーメッセージと共にコンパイルが失敗するのを確認できました。

```text
エラー: Recordの参照はあいまいです
    public <O extends Record> SomeTable(Table<O> child, ForeignKey<O, SomeRecord> key) {
                      ^
  org.jooqのインタフェース org.jooq.Recordとjava.langのクラス java.lang.Recordの両方が一致します
```

これはjOOQに[Record](https://www.jooq.org/javadoc/latest/org.jooq/org/jooq/Record.html)というクラスが存在していて、自動生成のコードでそれを利用しているためです。Java 14以降Lombokの[@Value](https://projectlombok.org/features/Value)と似た機能を持つ[Record Class](https://docs.oracle.com/en/java/javase/15/language/records.html)が登場していて、`record`キーワードを使って定義したクラスは全て[java.lang.Record](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Record.html)を実装する形になっています。というわけで、「jOOQのRecordを意味するのか、JavaのRecordを意味するのかコンパイラが判断できない」とエラーが発生してしまうのです。

これは、既存の自動生成コードのインポートを修正するだけで回避できます。既存のコードだと、自動生成のコードでは以下のようにjOOQのクラスをインポートしています。

```java
import org.jooq.*;
```

これを、明確にjOOQのRecordを指定するように修正します。

```java
import org.jooq.Record;
import org.jooq.*;
```

以上で問題なくコンパイルが通るようになりました。もし自作のライブラリなどで`Record`というクラスを定義している場合はここと同じエラーになる可能性があるので、なるべくクラスの名前を変えた方が良いかもです。

### Docker Image

現在開発中のサービスは[Jib](https://github.com/GoogleContainerTools/jib)を使ってコンテナ化しています。ここでベースとなるイメージの指定が必要なのですが、既存で使っていたイメージはOpenJDKの[11-jre](https://hub.docker.com/layers/library/openjdk/11-jre/images/sha256-762d8d035c3b1c98d30c5385f394f4d762302ba9ee8e0da8c93344c688d160b2?context=explore)でした。これが17からはJREのみのイメージはなく、[17-jdk](https://hub.docker.com/layers/library/openjdk/17-jdk/images/sha256-98f0304b3a3b7c12ce641177a99d1f3be56f532473a528fda38d53d519cafb13?context=explore)のみとなったので、バージョンをあげる際は注意する必要があります。

ただ、OpenJDK以外のイメージ意外を使っている場合は状況が違うかも知れませんので確認が必要です。例えば、[Temurin](https://hub.docker.com/layers/library/eclipse-temurin/17-jre/images/sha256-402c656f078bc116a6db1e2e23b08c6f4a78920a2c804ea4c2d3e197f1d6b47c?context=explore)や[Zulu](https://hub.docker.com/layers/azul/zulu-openjdk/17-jre/images/sha256-09163c13aefbe5e0aa3dc7910d9997e416031e777ca0c50bd9b66a63b46be481?context=explore)は`17-jre`を提供していて、Libericaの場合は[17](https://hub.docker.com/layers/bellsoft/liberica-openjdk-alpine/17/images/sha256-50feb980f142b152e19c9dc25c835e0e451321eb3b837a3a924f457b11ce8c59?context=explore)とバージョンだけになっているなど使っているJDKの種類によってタグ名が違うので、JDKのバージョンアップの際には使っているイメージのタグはチェックしておく方が良いでしょう。

### 依存関係

自分の担当しているアプリで発生していた問題ではないのですが(マイクロサービスとして、Kotlinのサービスは複数あります)、一部でJavaのバージョンを11から17に上げた際に[Jasperreports](https://github.com/TIBCOSoftware/jasperreports)を使った帳票の出力で報告された問題がありました。このライブラリはPDFの出力のために利用しているのですが、レイアウトに問題はなかったものの、表の中の表示文字数が少し減ったという問題がありました。幸い、これは大きい問題ではなかったのでまずは対応なしとなりそうですが場合によっては致命的かも知れません。

おそらくこのような問題が発生したら、依存しているライブラリのバージョンをJava 17に対応したものにあげれば解消できるのではないかと思いますが、まだ17に対応していないライブラリがある可能性もあるので、事前に依存関係の方をチェックしておいた方が胃良いでしょう。

## Spring Boot 3

Java 17は今年に11のサポートが終了するということで必須としていましたが、Spring Boot 3(Spring Framework 6)の場合は去年の12月にリリースされたばかりなので今回はあえて移行を試す必要はありませんでした。ただ、Spring Boot 3からちょうどJava 17が最低バージョンになり、以前からサーバレスの[GAE](https://ja.wikipedia.org/wiki/Google_App_Engine)上で起動しているプロジェクトもあったので、起動時間を減らすために[Spring Native](https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html)を試してみたいと思っていたためでもあります。Spring NativeはSpring Boot 3から正式にサポートされることになったので。

また、Spring Boot 3で依存関係が変わったり、プロパティの記述方法が変わったりするケースがあるとしても、自分が扱っているアプリケーションはマイクロサービスなので比較的に影響が少ないと予測できたからです。おそらくモノリシックで、Springの様々な機能に依存している場合は移行が難しい場合もあるかなと思います。なので、もし移行を考えている場合はなるべくリリースノートなどを確認しておいた方が良いでしょう。

Spring Boot 2系から3系の移行は、基本的に[公式のマイグレーションガイド](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)を参照すると良いのですが、マイグレーションガイドだけではわからないことや、Spring Frameworkの依存関係の変化により他のライブラリに影響が出ることがあります。なので自分の場合はどのように対応したかを少し紹介したいと思います。

### @ConstructorBinding

[@ConstructorBinding](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/context/properties/ConstructorBinding.html)は、[@ConfigurationProperties](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/context/properties/ConfigurationProperties.html)を付けたクラスにアプリケーションプロパティファイルから読み込んだ値をコンストラクタインジェクションするためのアノテーションです。現在のプロジェクトではAWSのS3のバケット名やメールのテンプレート、他のAPIを呼び出すためのエンドポイントなどを`application.yaml`に記載して読み込むようにしています。

ここで、Spring Boot 3だと`@ConstructorBinding`なしでもプロパティを読み込むようになって、そもそもこのアノテーション自体がDeprecatedになっています。なので、最初はコンパイルエラーとなりますが、`@ConstructorBinding`を削除するだけで問題なく動作するようになりました。最も簡単な対応でした。

### Tracingの変更

現在のアプリケーションでは、他のマイクロサービスのAPIを呼び出す際に[Brave](https://github.com/openzipkin/brave)を使ってヘッダに`Trace Context`を載せています。この場合、同じTrace IDが連携されるのでログの確認がやりやすくなりますね。今までは[Spring Cloud Sleuth](https://spring.io/projects/spring-cloud-sleuth)を通じてBraveをDIしていましたが、これがSpring Boot 3になくなり(Spring Bootに統合）、[Micrometer](https://micrometer.io/)の新しいAPIを利用することになったらしいです。

つまり、既存のSpring Cloud SleuthによるDIは使えなくなり、依存関係も変化があるということです。まずは、DIを担当していた[BraveAutoConfiguration](https://docs.spring.io/spring-boot/docs/3.0.1/api/org/springframework/boot/actuate/autoconfigure/tracing/BraveAutoConfiguration.html)が[Spring Boot Actuator](https://github.com/spring-projects/spring-boot/tree/v3.0.2/spring-boot-project/spring-boot-actuator)に移りました。なので、もうSpring Cloud Sleuthはいらなくなります。

ただ、Spring Boot Actuatorだけだと`BraveAutoConfiguration`によるDIはできません。なので以下のようにブリッジの依存関係を追加する必要があります。(バージョンはSpring Bootのものが使われる)

```kotlin
implementation("io.micrometer:micrometer-tracing-bridge-brave")
```

依存関係が変わるので、`application.yaml`の設定も変更が必要になります。既存では、以下のように記載していました。

```yml
spring:
  sleuth:
    enabled: true
    propagation:
      type:
        - w3c
```

Micrometerに移行したので、このように変更します。実際は、以下の設定はデフォルトなので記載しなくても問題はありませんが、メンテナンス時に現在の設定がわかりやすいように記載しています。

```yml
management:
  tracing:
    enabled: true
    propagation:
      type: w3c
```

ただ、最もつまづいた部分はユニットテストでした。テストを実行してみると、なぜか以下のようなエラーメッセージが出ていました。

```shell
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'brave.Tracing' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {}
```

Spring Cloud Sleuthを使っていた頃はユニットテスト時もDIができていたのですが、なぜか`BraveAutoConfiguration`によるDIが聞かない状態です。ローカルで起動した場合は同じエラーにならないので、おそらくテスト時にだけAutoConfigurationが呼ばれてないように思われます。

[@TestConfiguration](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/context/TestConfiguration.html)などを使って、直接Bean登録をする方法もあるかと思いますが、調査してみると[Micrometerにはテスト用のライブラリが用意されている](https://www.baeldung.com/spring-boot-3-observability#testing-observations)ので、それを使うことにしました。このライブラリがリリースされたのは2022年の12月なので、なかなか情報を見つけるのが難しいです。とにかく、以下のようにライブラリを追加することでDIの問題は解消できます。

```kotlin
testImplementation("io.micrometer:micrometer-tracing-test")
```

今回はライブラリの追加だけで解消できたのですが、新しいライブラリの情報はこちらで調べないとなかなかわからないというのが問題でしたね。他のライブラリの場合も同じ問題があるかもしれないので、もしAutoConfigurationによりDIされるはずのBeanが見つからないとのエラーが出る場合は、テスト用のライブラリが追加されてないか確認するのが良いかも知れません。

追加で、[Datadog](https://www.datadoghq.com/ja/)の場合もMicrometerの設定の記載方法が全体的に変わったためか、一部プロパティの記載方法が変わっています。以前の場合は以下のように記載していました。

```yml
management:
  metrics:
    export:
      datadog:
        enabled: true
        apiKey: ${API_KEY}
        step: 1m
```

これが以下のように変わります。

```yml
management:
  datadog:
    metrics:
      export:
        enabled: true
        api-key: ${API_KEY}
        step: 1m
```

他にもMicrometerの機能を使っている場合、既存のSpring Boot 2と比べプロパティの設定方法が変わってないか確認した方がいいでしょう。

### Jakarta EE

Spring Framework 6はOracleの`Java EE`からEclipseの`Jakarta EE`に移行しているので、コンパイルが通らない場合があります。主に[HttpServletRequest](https://docs.oracle.com/javaee/7/api/javax/servlet/http/HttpServletRequest.html)などsevlet系や[@Valid](https://docs.oracle.com/javaee/7/api/javax/validation/Valid.html)などvalidation系パッケージがよく使われているかと思います。これらはimport時のパッケージ名を変更することで対応できます。例えば、以下のようなパッケージを使っている場合だとします。

- javax.servlet
- javax.validation

これらは以下のように変更します。

- jakarta.servlet
- jakarta.validation

パッケージ名が変わっただけで、中身は変わっていないので、簡単な対応となります。

### springdoc

[springdoc-openapi](https://springdoc.org/)を使ってAPIドキュメントを生成している場合、既存のバージョンだとSpring Boot 3と互換性がないようです。CIでは`/v3/api-docs`から取得したJSONを解析してS3にドキュメントをアップロードするようになっていますが、Spring Boot 3だと404エラーとなっていました。

幸い、これも依存関係を変更することで対応ができます。以前は以下のように依存関係を追加していました。

```kotlin
implementation("org.springdoc:springdoc-openapi-ui:1.6.14")
implementation("org.springdoc:springdoc-openapi-kotlin:1.6.14")
```

これを、[springdoc-openapiのv2](https://springdoc.org/v2/)に変更するだけで良いです。Kotlinのサポートも含まれるので、以下のみで問題なく動きます。

```kotlin
implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.0.2")
```

これで404エラーは発生することなく、S3へのドキュメントのアップロードもできます。

### Liquibase

Spring Frameworkとの直接的な関係がある訳ではないのですが、Spring Bootのバージョンを上げたことにより影響を受けたのでこちらのケースも紹介します。DBのマイグレーションのためにGradleのプラグインとして[Liquibase](https://www.liquibase.org/)を使っている場合、バージョンによっては以下のようなエラーが出る場合があります。

```text
SLF4J: No SLF4J providers were found.
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See https://www.slf4j.org/codes.html#noProviders for further details.
SLF4J: Class path contains SLF4J bindings targeting slf4j-api versions 1.7.x or earlier.
SLF4J: Ignoring binding found at [jar:file:/root/.gradle/caches/modules-2/files-2.1/ch.qos.logback/logback-classic/1.2.11/4741689214e9d1e8408b206506cbe76d1c6a7d60/logback-classic-1.2.11.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See https://www.slf4j.org/codes.html#ignoredBindings for an explanation.
Exception in thread "main" java.lang.ClassCastException: class org.slf4j.helpers.NOPLogger cannot be cast to class ch.qos.logback.classic.Logger (org.slf4j.helpers.NOPLogger and ch.qos.logback.classic.Logger are in unnamed module of loader 'app')
	at liquibase.integration.commandline.Main.setupLogging(Main.java:233)
	at liquibase.integration.commandline.Main.run(Main.java:145)
	at liquibase.integration.commandline.Main.main(Main.java:129)
```

これはおそらく、Spring Boot 3から[SLF4J](https://www.slf4j.org/)のバージョンが2系になったためですね。Liquibaseのプラグインでは今まで[Logback](https://logback.qos.ch/index.html)を使ってマイグレーションの状況を出力していたのですが、このLogbackのバージョンが1.2だったのでSLF4Jの2系と互換性がなかったのです。Logbackの1.3から互換性があるらしいので、それ以前のバージョンを使っている場合はこちらも合わせてバージョンを上げておく必要があるでしょう。

## 最後に

何とかJavaとSpring Bootのバージョンアップには成功していますが、前述した通り、これはあくまで自分が開発しているアプリケーションがマイクロサービスで、依存関係が比較的少なかったからできたことなのではないかと思います。モノリシックなアプリケーションだったり、より複雑な依存関係を持っているアプリケーションならここに記載したこと以外の部分でも問題が発生する可能性は高いでしょう。

ただ、Springの場合はいずれSpring Boot 2のサポートが切れ、3系に移行するしかない状況が来るかも知れませんが、その時は十分マイグレーションの実例も出てくるのではないかと思います。Javaの場合は11から17に上げる場合なら大きく問題はないかと思いますが、8から移行する場合はJava 9で導入された[Module](https://www.oracle.com/webfolder/technetwork/jp/javamagazine/Java-JA18-LibrariesToModules.pdf)で問題が起こる可能性もあるかと思いますので、慎重に行うべきかなと思います。

嬉しいことに、まだNative化までは試してないのですが、JVM上でもアプリケーションの起動時間が減少していることを確認できました。ローカルで起動した場合、Java 11 & Spring Boot 2だと31秒がかかり、Java 17 & Spring Boot 3だと23秒がかかっていたので、断定はできないものの全体的な性能の向上もある程度は期待できるのではないかという気がします。正確なデータはまだ取れていないので、今後の課題として残しておきますが、ありがたいことですね。

では、また！
