---
title: SpringとKtor、5つの構成を比較してみた
date: 2026-03-20
categories:
  - kotlin
image: ../../images/kotlin.webp
tags:
  - spring
  - ktor
  - benchmark
  - r2dbc
  - jdbc
  - virtual-thread
  - kotlin
---

最近、[Exposed](https://www.jetbrains.com/exposed/)が1.0からR2DBCを正式に扱えるようになったので、KtorでもDBアクセスまで含めて全体をノンブロッキングで構成しやすくなりました。もともとKtorはCoroutineベースで書けるので、ここにR2DBCまで揃えば「Kotlinらしい構成のまま、かなり素直に高い同時処理性能が出るのでは」と少し期待したくなりますね。

一方でSpringも、Spring Boot 3系から仮想スレッドをかなり扱いやすくなりました。そうなると、従来のSpring MVC + JDBCのようなオーソドックスな構成と、WebFlux + R2DBCのようなリアクティブ構成をどう見比べるべきかも気になります。さらにそこへKtor + JDBC、Ktor + R2DBCまで加えると、フレームワーク、シリアライザ、ORM、ドライバの違いがだいぶ見えやすくなりそうです。

なので今回は、同じPostgreSQLと同じHTTPシナリオを使って、以下の5つの構成を同じ条件で比較してみました。

## 比較対象

今回比較したのは、以下の5構成です。

| 構成 | 主なスタック |
|---|---|
| Spring MVC JDBC (platform) | Tomcat, Jackson, jOOQ, JDBC, HikariCP, platform threads |
| Spring MVC JDBC (virtual) | Tomcat, Jackson, jOOQ, JDBC, HikariCP, virtual threads |
| Spring WebFlux R2DBC | Netty, Jackson, jOOQ SQL builder, R2DBC PostgreSQL, R2DBC pool |
| Ktor JDBC | Netty, kotlinx.serialization, Koin, Exposed JDBC, HikariCP |
| Ktor R2DBC | Netty, kotlinx.serialization, Koin, Exposed R2DBC, R2DBC PostgreSQL |

ここでSpring MVCはアプリ自体は同じで、仮想スレッドの有効・無効だけを切り替えています。Ktor側はJDBC版とR2DBC版を分けることで、「Ktorそのものの差」なのか、「R2DBC + Exposed R2DBCの差」なのかを見やすくしました。

個人的には、Ktor + kotlinx.serialization + ExposedのようなJetBrains公式ライブラリで揃った構成はかなり好きです。コード量も比較的抑えやすく、Kotlinで書いている感覚も強いですね。ただ、好き嫌いと性能はまた別の話なので、そこを一度ちゃんと数字で見ておきたいというのが今回の出発点です。

## テストシナリオと測定方法

今回は、単に「どれが速いか」を見るというより、どのレイヤで差が出ているのかを切り分けやすいように、シナリオをいくつかの種類に分けて用意しました。DBを使わない軽いレスポンス、DBの単純な読み取り、大きいペイロード、検索や集計っぽい処理、そして書き込みと競合です。

シナリオの意図はざっくり以下の通りです。

| 分類 | シナリオ | 見たいこと |
|---|---|---|
| non-DB | `GET /health` | もっとも軽いHTTP経路。ルーティングや最小限のレスポンス処理の差が出やすい |
| small payload | `GET /api/books/bench/small-json` | DBなしの小さいJSON。フレームワークとシリアライザの素の軽さを見やすい |
| large payload | `GET /api/books/bench/large-json?limit=200` | 大きいJSONの組み立てとシリアライズの負荷を見る |
| point read | `GET /api/books/{id}` | 主キーで1件読む典型的なCRUD read |
| list read | `GET /api/books?limit=50` | 軽めの一覧取得 |
| large list read | `GET /api/books?limit=500` | 取得件数が多い時のマテリアライズとレスポンス生成 |
| search read | `GET /api/books/search?q=Book&limit=20` | 条件付き読み取り |
| aggregate-like read | `GET /api/books/bench/aggregate-report?limit=50` | 集計風の読み取り経路 |
| write | `POST /api/checkouts` | 通常の在庫減算 + checkout作成 |
| write contention | `POST /api/checkouts` (`bookId=1` 固定) | 同じ行を何度も更新した時の競合 |
| distributed write | `POST /api/checkouts` (ランダム bookId) | 競合をある程度分散した書き込み |

特に`checkout`系は見方に少し注意が必要です。内部では「本を読む -> 在庫を確認する -> 在庫を減らす -> checkoutをinsertする」という書き込みトランザクションを走らせています。なので、単純なJSONレスポンスとは違って、コネクションプール、トランザクション、ロック競合の影響がかなり強く出ます。

測定条件は以下の通りです。

- PostgreSQL 17 を毎回コンテナで起動し、各実行ごとに初期化
- すべて同じベンチマークランナーから同じHTTPシナリオを実行
- ウォームアップ1秒、計測3秒
- 同時実行数は128と256の2パターン
- Spring MVCはHikariCP最大32、Spring WebFluxとKtor R2DBCはR2DBC pool最大32、Ktor JDBCはHikariCP最大32

今回はスタック全体を比較したかったので、フレームワークだけではなく、シリアライザ、DI、ORM、ドライバも含めてそのまま測っています。つまりこれは「純粋なKtor vs Spring」の比較というより、「実際にこの組み合わせで組んだ時の全体差」の比較です。

## 測定結果

細かい数字は最後に紹介するHTMLサマリを見てもらうのが早いですが、傾向として面白かったところを先にまとめると以下のようになります。

### 1. 小さいJSONではWebFluxが強いが、大きいJSONではSpring MVCが強い

`smallJson`ではSpring WebFlux R2DBCが最も高く、同時実行256で約29,220 req/sでした。ここはDBアクセスがなく、フレームワークとシリアライズ経路の軽さが出やすいところなので、Nettyベースの構成が素直に強く出たのだと思います。

しかし`largeJson`になると話が変わります。同時実行256で見ると、Spring MVC platformが約4,011 req/s、Spring MVC virtualが約3,980 req/sに対して、Ktor JDBCは約2,741 req/s、Ktor R2DBCは約1,389 req/sでした。ここは以前から言われていた通り、`kotlinx.serialization`が大きいJSONではまだJacksonより不利な面を残しているように見えます。小さいペイロードではかなり健闘していますが、大きくなると差が出ますね。

### 2. DB読み取りでは、思った以上にSpring MVCが強い

本命だった読み取り系を見ると、`bookById`、`bookList`、`bookList500`、`bookSearch`の多くでSpring MVCの二構成が上位でした。例えば同時実行256では以下のような結果です。

| シナリオ | 1位 | 2位 | Ktor JDBC | Ktor R2DBC |
|---|---|---|---|---|
| `bookById` | Spring MVC virtual 14,903 | Spring MVC platform 14,901 | 8,343 | 7,448 |
| `bookList` | Spring MVC virtual 9,574 | Spring MVC platform 9,553 | 5,773 | 4,300 |
| `bookList500` | Spring MVC platform 2,526 | Spring MVC virtual 2,493 | 1,427 | 820 |
| `bookSearch` | Spring MVC platform 9,677 | Spring MVC virtual 9,418 | 6,020 | 5,359 |

最初の期待としては、CoroutineベースでR2DBCまで揃えたKtor R2DBCが、少なくとも読み取りでかなり強く出るかと思っていました。しかし結果はむしろ逆で、かなりオーソドックスなSpring MVC + JDBCが基準点として非常に強かったです。

仮想スレッドも面白くて、「常にplatformより速い」というわけではありませんでした。`bookById`や`bookList`のようなところではvirtualが少し勝つ一方、`health`や`bookSearch`、`checkoutCreate`ではplatformが勝つケースもあります。仮想スレッドは確かに有力ですが、入れた瞬間に何でも速くなる魔法ではない、というのはかなり実感しやすい結果でした。

### 3. Ktor JDBCは悪くないが、Ktor R2DBCはかなり苦しい

Ktor側だけを見ると、JDBC版とR2DBC版の差がかなりはっきり出ました。例えば同時実行256で、`bookList500`はKtor JDBCが1,427 req/s、Ktor R2DBCが820 req/s、`largeJson`はKtor JDBCが2,741 req/s、Ktor R2DBCが1,389 req/sです。`checkoutCreate`でもKtor JDBC 3,540 req/sに対し、Ktor R2DBCは4,044 req/sで少し戻すものの、少なくとも「R2DBCにすると全体的に有利になる」という結果ではありませんでした。

この差を見ると、Ktorにおけるボトルネックは単純に「Ktorそのもの」ではなく、かなりの部分がExposed R2DBCを含む非同期DBアクセス経路にあるように見えます。Spring WebFluxも非DBの軽いパスでは非常に強い一方、DB readではJDBCのSpring MVCを明確に上回る場面は多くありませんでした。少なくとも今回のようなCRUD中心のワークロードでは、Reactive StreamsベースのDBアクセスが期待したほど素直な優位にはつながっていない、という見方が自然だと思います。

### 4. hotspot系はreq/sだけ見てもあまり意味がない

`checkoutHotspot`だけは少し特殊で、同じ本の在庫をひたすら減らすので、途中から失敗が大量に出ます。そのためここは純粋なreq/sより、どれだけ「有効な成功」を出せているかを慎重に見る必要があります。

数字だけ見ると同時実行256でKtor JDBCが約8,522 req/sと最も高いですが、失敗数も25,617と非常に多いです。Ktor R2DBCも6,417 req/sで失敗19,295、Spring WebFluxは2,883 req/sで失敗8,705でした。つまり、ここは「速い」の中に「速く失敗している」が大量に含まれているので、通常のシナリオとは少し別物として扱うべきですね。

### 5. 例外的にaggregateReportではKtor JDBCが目立った

`aggregateReport`ではKtor JDBCが同時実行128で1,430 req/s、256でも1,439 req/sで最も高い結果でした。ただしこのシナリオについては、Ktor側がまだSpring側と完全に同じDBの`GROUP BY`経路ではないという注意書きがあるので、ここはあくまで参考値として見た方がよさそうです。

## そこから得たこと

今回の結果で一番印象的だったのは、「ノンブロッキングだから高いRPSが出るはず」という期待が、そのまま素直には当たらなかったことです。もちろんノンブロッキングは重要ですし、特定のワークロードでは非常に有効でしょう。ただ、今回のように比較的オーソドックスなCRUD中心のベンチマークでは、Spring MVC + JDBCのような伝統的な構成がまだかなり強いです。

また、`kotlinx.serialization`は小さいJSONではかなり健闘する一方、大きいJSONではJacksonとの差が出やすいように見えました。以前から言われていた傾向ですが、今回の計測でもその差はある程度確認できました。

R2DBCについても、少なくともこの条件では「JDBCを明確に置き換えるほど速い」とは言いづらい結果でした。思想としては綺麗でも、DBアクセス込みの実アプリでは別のオーバーヘッドをきちんと払っている、という印象です。

## 最後に

個人的な好みで言えば、やはりKtorと`kotlinx.serialization`、ExposedのようなKotlinネイティブな構成はかなり魅力的です。コードスタイルも揃いますし、書いていて気持ちがよいところがあります。ただ、AIの助けでコードそのものを書くコストが以前より下がってきた今は、「どれだけ少ないコードで書けるか」だけでなく、「最終的にどれだけ速く動くか」を以前より重視した方がよさそうだ、とも思いました。

その意味では、性能重視であれば今でもSpring MVCのようなオーソドックスな構成はかなり有力です。一方で、コード量や読みやすさ、Kotlinらしさを優先するならKtorを選ぶのも十分ありだと思います。実際、Ktor JDBCは全体として悲観するほど悪い結果ではありませんでしたし、アプリケーションの性質次第では十分実用的でしょう。

測定結果の詳細については、Benchmarkリポジトリの[README](https://github.com/retheviper/Benchmark/blob/main/README.md)と、そこで公開している[five-stack-summary.html](https://github.com/retheviper/Benchmark/blob/main/docs/reports/five-stack-summary.html)を見てもらうのが早いと思います。概要、目的、比較対象のスタック、測定方法はREADMEにまとまっているので、個別の数字やシナリオごとの傾向を追いたい場合はそちらを参照してください。
