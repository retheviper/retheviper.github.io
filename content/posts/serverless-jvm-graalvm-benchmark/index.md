---
title: "サーバーレス環境で JVM と GraalVM Native Image を比べてみた"
date: 2026-04-29
translationKey: "posts/serverless-jvm-graalvm-benchmark"
categories:
  - java
image: "../../images/java.webp"
tags:
  - java
  - jvm
  - graalvm
  - serverless
  - spring
  - ktor
  - quarkus
---

サーバーレス環境では、最初のリクエストにどれだけ早く応答できるかがかなり重要になります。常にプロセスが温まっているサーバーと違って、コールドスタートの遅さはそのままユーザー体験に出ますし、メモリ使用量もコストに直結しやすいです。

以前、[JVM はまだ進化する](../jvm-evolution/)という記事で Leyden や Loom などについて書きました。JVM は古いだけのランタイムではなく、起動時間や warm-up、メモリ効率を改善する方向で今も進化しています。特に Java 25 では Leyden に関連する改善も入り始めているので、今回は「GraalVM Native Image にすれば本当にどれくらい違うのか」「Java 25 の JVM でもどこまで戦えるのか」を、サーバーレス風の条件で比べてみました。

比較に使ったリポジトリは [Serverless](https://github.com/retheviper/Serverless) です。測定結果の HTML サマリは [serverless-summary.html](https://github.com/retheviper/Serverless/blob/main/docs/reports/serverless-summary.html) に置いてあります。

## 何を比較したか

今回は、同じ HTTP API と同じ PostgreSQL アクセスを持つアプリケーションを、以下の構成で比較しました。

| 対象 | ランタイム |
|---|---|
| Quarkus | JVM Java 25 |
| Quarkus | GraalVM Native Image Java 25 |
| Spring Boot | JVM Java 25 |
| Spring Boot | GraalVM Native Image Java 25 |
| Ktor | JVM Java 25 |
| Ktor | GraalVM Native Image Java 25 |

Ktor Kotlin/Native も別枠で入れていますが、これは DB 付き比較の対象にはしていません。理由は単純で、JDBC は JVM API なので、JVM / GraalVM Native Image と同じ PostgreSQL 経路を共有できないからです。現状では実験的なサーバー起動測定として見るのが妥当だと思います。

API の形は以下のように揃えました。

| Endpoint | 処理 |
|---|---|
| `GET /health` | JSON health response |
| `GET /api/items/{id}` | PostgreSQL point read と JSON object serialization |
| `GET /api/items?limit=50` | PostgreSQL ordered list read と JSON array serialization |
| `GET /api/items/report` | PostgreSQL aggregate query と JSON array serialization |

単純に「プロセスが起動した」だけではなく、DB 接続、JSON シリアライズ、DI/component graph 初期化、HTTP server startup まで含めたかったので、主要指標は `apiReadyMillis` にしました。これは `/api/items/1` が実際に応答するまでの時間です。

## 測定結果

HTML サマリの中央値を見ると、結果はかなりはっきりしていました。

| Target | Median API ready ms | Peak RSS MiB | Median steady RSS MiB |
|---|---:|---:|---:|
| Quarkus GraalVM Native | 81 | 41.2 | 27.3 |
| Spring Boot GraalVM Native | 118 | 98.9 | 60.4 |
| Ktor GraalVM Native | 243 | 34.9 | 28.9 |
| Ktor JVM | 641 | 160.4 | 134.1 |
| Quarkus JVM | 710 | 151.1 | 129.1 |
| Spring Boot JVM | 1716 | 332.3 | 270.7 |

一番目立つのは、やはり GraalVM Native Image の起動の速さです。Quarkus Native は中央値 81ms、Spring Native は 118ms、Ktor Native は 243ms でした。JVM 版も Java 25 を使っているので、昔の JVM と比べればかなり改善されているはずですが、それでもコールドスタートでは Native Image の差が大きく出ます。

メモリも同じ傾向です。Ktor Native と Quarkus Native は steady RSS が 30MiB 前後に収まりました。一方で JVM 版は Ktor と Quarkus が 130MiB 前後、Spring Boot は 270MiB 前後でした。サーバーレスのようにメモリ割り当てが課金や同時実行数に影響する環境では、この差は無視しにくいと思います。

## フレームワークごとの差

Quarkus はこの種の用途にかなり強い印象でした。Native Image では起動時間もメモリも小さく、JVM 版でも Spring Boot よりだいぶ軽く出ています。Quarkus はもともとビルド時処理を強く使う設計なので、GraalVM Native Image との相性がよいのは想定通りですが、実際に DB 付きの API ready でもかなり良い数字でした。

Spring Boot は JVM 版だと一番重く出ました。これは Spring が悪いというより、DI や auto configuration、アプリケーションコンテキスト初期化の厚みがそのままコールドスタートに出たと見る方が自然です。ただし Native Image にすると 118ms まで短縮されていて、Spring AOT と GraalVM の効果はかなり大きいです。

Ktor は JVM 版では Quarkus JVM に近い位置でしたが、Native Image では Quarkus や Spring より遅く出ました。今回の Ktor Native では CIO engine を使い、Ktor の runtime serializer lookup に頼らず `kotlinx.serialization` の serializer を明示的に呼ぶようにしています。また Hikari の native-image metadata 問題を避けるため、DB 付きターゲットでは小さな eager JDBC pool をリポジトリ内に実装しています。つまり Ktor そのものだけでなく、Native Image 対応のための実装上の制約も数字に含まれています。

## 実装上の制約

GraalVM Native Image は、単に JVM アプリをそのままコンパイルすれば終わりというものではありませんでした。

今回もいくつか制約がありました。

- Quarkus native build では通常の JAR package と native executable を同じビルドで同時に出せない
- Spring native build では Spring AOT と GraalVM Native Build Tools plugin を使う必要がある
- Kotlin Spring application では Spring AOT が Kotlin metadata を読むために `kotlin-reflect` が必要になる
- Ktor GraalVM native は CIO engine を使う必要がある
- Ktor native では serializer lookup や Hikari まわりをそのまま使いづらい部分がある
- Kotlin/Native は JDBC を使えないので、DB 付きの JVM/GraalVM 比較とは別枠になる

このあたりは、ベンチマークの数字を見る時に重要だと思います。Native Image は確かに速くて軽いですが、その代わりに build time、reflection、dynamic proxy、serializer、JDBC driver、framework integration などを意識する必要があります。

もちろん Quarkus や Spring Boot はこのあたりをかなり吸収してくれます。だからこそフレームワークの Native Image 対応度が、そのまま実用性の差として出やすいです。Ktor のように薄いフレームワークでは、自分で意識する部分が増える反面、どこで何をしているかは見えやすいという面もあります。

## 以前の 5 スタック比較との違い

以前、[SpringとKtor、5つの構成を比較してみた](../spring-ktor-five-stack-benchmark/)では、同じ PostgreSQL を使って Spring MVC JDBC、Spring MVC virtual thread、Spring WebFlux R2DBC、Ktor JDBC、Ktor R2DBC を比較しました。

あの時は throughput を中心に見たので、「リクエストを受け続けている状態でどれが強いか」が主な関心でした。結果としては、Spring MVC + JDBC のような伝統的な構成がかなり強く、非同期や R2DBC が必ず勝つわけではないという印象でした。

今回はそこから視点を変えて、サーバーレス的なコールドスタートとメモリを見ています。つまり、定常時の req/s ではなく、プロセスを起動して最初の DB-backed JSON response を返すまでの時間です。

この二つは同じ性能比較でも、見ているものが違います。常駐サーバーなら throughput や latency distribution が重要ですが、サーバーレスや scale-to-zero に近い環境では cold start と RSS がかなり大きな意味を持ちます。なので、前回の結果と今回の結果は矛盾するというより、別の軸を見ていると考えるのがよさそうです。

## 環境による差はありえる

今回の測定環境は arm macOS です。Native binary は host-specific なので、x64 Linux 上で同じ差になるとは限りません。

実際のサーバーレス環境は多くの場合 Linux ですし、CPU アーキテクチャ、container runtime、filesystem、network、PostgreSQL への距離、メモリ制限、プロセス起動方法によって結果は変わります。特に Native Image はビルドされた OS / architecture の影響を受けるので、最終的にはデプロイ先に近い環境で測るべきです。

また、今回は `SERVERLESS_DEPENDENCY_COUNT` のデフォルトを 180 にして、DI が多いアプリケーションをある程度模擬しています。ただ、実際の Spring アプリケーションはもっと多くの bean、auto configuration、configuration properties、external clients、security filter、metrics などを持つことがあります。DI 対象が増えるほど Spring Boot JVM の起動時間やメモリにはより強く影響する可能性がありますし、Native Image 側でも reachability metadata や AOT 処理の複雑さが増えるかもしれません。

つまり今回の数字は「この条件ではこうだった」という実測値であって、すべての Spring / Ktor / Quarkus アプリケーションにそのまま一般化できるものではありません。

## 最後に

今回の結果を見る限り、サーバーレス環境で即応性とメモリ効率を重視するなら、GraalVM Native Image はかなり有力です。特に Quarkus Native と Spring Native は、DB 接続と JSON 応答まで含めてもかなり短い時間で ready になっていました。

一方で、JVM もまだ終わった選択肢ではないと思っています。Java 25 を使った JVM 版でも、Ktor や Quarkus は 1 秒未満で API ready になっていますし、Leyden のような改善が進めば、今後さらに差が縮まる可能性もあります。JVM の柔軟性、デバッグのしやすさ、既存ライブラリとの相性を考えると、単純に Native Image だけを選べばよいという話でもありません。

ただ、コールドスタートとメモリの数字だけを見ると、今の時点では Native Image の強みはかなり明確でした。常駐サーバーなら前回の 5 スタック比較のように throughput を見るべきですが、サーバーレスではまた別の判断軸が必要になります。

結局は、どの環境で、どの頻度で cold start が発生し、どれくらいメモリコストを気にするか次第ですね。今回の [Serverless](https://github.com/retheviper/Serverless) はその判断材料の一つとして、今後ももう少し育てていきたいと思います。

では、また！
