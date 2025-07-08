---
title: "JVM はまだ進化する"
date: 2025-07-08
categories: 
  - java
image: "../../images/java.webp"
tags:
  - jvm
  - java
  - kotlin
  - spring
---

現代のプログラミング環境は、Go や Rust などの軽量ランタイムが受け入れられる中、「JVM の方向性」に疑問を持つ声も聞かれるようになってきました。私自身もそうで、特にクラウド環境では、JVM の起動時間やメモリ使用量が問題となるケースも少なくないです。なのでサーバレス環境の場合、純粋に起動時間のためPythonやNode.jsを選ぶこともあります。

しかし JVM は、現在も新技術によって性能改善の余地を持ち続けています。この記事では、Leyden や Loom を始めとした現在進行中の主要プロジェクトを概覧し、JVM の「性能」は未来にわたってどれだけ改善可能かを検討します。

## JIT から AOT + PGO へ

[JIT (Just-In-Time Compiler)](https://en.wikipedia.org/wiki/Just-in-time_compilation)は、バイトコードを実行時にネイティブコードへコンパイルする技術です。
最適化コードをランタイムで生成できる依存性が高い一方、warm-up期間[^1]や"jitter[^2]による性能不定は無視できません。

これらを解決するために考案されたのが[Graal](https://www.graalvm.org) で、 C2 の代替として設計された高性能 JIT コンパイラです。inlining, escape analysis, vectorization などで事前に最適化を行い、JIT の不安定性を削減します。

例えば[Twitterの事例](https://www.oracle.com/a/ocom/docs/graalvm-twitter-casestudy-constellation.pdf) では Graal JIT を通じて CPU 利用率を8〜11%削減し、Kotlinのマイクロベンチマークでは [+18%の性能向上](https://martijndwars.nl/2020/02/24/graal-vs-c2.html) を報告しています。

## Leyden で起動の早さと安定性を両立

[Project Leyden](https://openjdk.org/projects/leyden/) は、JVM アプリケーションの起動時間と warm-up 時間を根本的に短縮することを目標にしています。

従来の JIT ベースの最適化は、ランタイムの初期段階でプロファイル情報を集めながら段階的に性能を引き上げていく「適応的」最適化方式ですが、Leyden はそれを「事前確定的」（static）な最適化に置き換える方向性をとります。

このために Leyden は、以下の要素をアーカイブとして事前に保存する「condensers」という概念を導入しています。この condensers は、以下の情報を含むアーカイブです。

* 使用クラスのリストと linking 情報
* ヒープの初期状態（AppCDS + heap snapshot）
* プロファイル情報（PGO）
* コンパイル済みコード

これにより、アプリケーションの起動直後から JIT 後のような性能を発揮し、jitter を最小限に抑えることが期待されます。

たとえば [Quarkus](https://quarkus.io/) チームは Leyden の初期実装に基づき、以下のような改善を報告しています。

* 起動時間を 50% 以上短縮
* Native Image に近い応答速度を実現しつつ JVM の柔軟性は保持
* メモリ使用量を最大 30% 削減

Leyden の関連 JEP には、[JEP 483](https://openjdk.org/jeps/483)（class loading/linking の事前保存）、[JEP 515](https://openjdk.org/jeps/515)（PGO support）などがあり、JDK 25 以降で順次取り込まれる予定です。

## ZGC, Shenandoah, そして LXR

GCは JVM の性能に大きく影響する要素です。近年導入した[ZGC](https://wiki.openjdk.org/display/zgc/Main) や [Shenandoah GC](https://wiki.openjdk.org/display/shenandoah/Main) は「pause-less GC」を目指し、ZGC は広大なヒープを持つ JVM でも少なくとも <1ms の GC ポーズを実現しています。

最新の [LXR GC](https://www.steveblackburn.org/pubs/papers/lxr-pldi-2022.pdf)（研究段階）は、ZGC より九分すぐれた性能を見せ、tail latency を30倍減少、throughput を6倍向上という報告も出ています。

[Cassandra ベンチマーク](https://www.datastax.com/blog/apache-cassandra-benchmarking-40-brings-heat-new-garbage-collectors-zgc-and-shenandoah)では Shenandoah が p99 レイテンシを77%減らすなど、リアルなサービスへの影響も証明されています。

これらは、小型のサービスから広大な JVM グラフまで、性能の jitter を削減しながら throughput を保てるっという為に大きな意味を持ちます。

## Valhalla の支援するメモリー効率

[Project Valhalla](https://openjdk.org/projects/valhalla/) は「value class」を JVM に実装することで、ヒープのオブジェクト依存を解消し、キャッシュローカリティを向上させます。

通常のオブジェクトはヒープ内の分散やヘッダーを持ち、GC やキャッシュミスに負担をかけますが、value class はプリミティブな構造体のように inlined されるため、メモリーバンドを向上させます。

特に Kotlin の data class は Valhalla から大きな影響を受け、変数配列やベクタ計算のような性能パスでは [数倍の向上](https://www.reddit.com/r/java/comments/1dnhgut/i_actually_tested_jdk_20_valhalla_here_are_my/) も期待できます。

## Loom でスケーラブルな I/O 並行性の実現

[Project Loom](https://openjdk.org/projects/loom/) は仮想スレッドによって、従来のブロッキングな I/O コードをそのまま保ちながら、大量の同時接続を処理できるようにする試みです。

ここで重要なのが、既存の並行処理との関係です。

従来の Java（および Spring MVC）は、**リクエストごとに OS スレッドを1本消費する**アーキテクチャでした。
しかしこれは、同時接続数が多くなるとすぐにスレッド枯渇やコンテキストスイッチによる性能低下を招きます。

この問題を解決するために導入されたのが [Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html) です。
WebFlux は [Reactor](https://projectreactor.io/) に基づくノンブロッキング非同期モデルで、少数のスレッドで大量のリクエストを捌くことを可能にしました。

ただしその代償として、開発者は `Mono` や `Flux` を理解し、非同期パイプラインの構築を強いられ、デバッグも難しくなるという課題がありました。幸 Kotlin では `suspend` 関数を使うことで非同期処理を簡潔に書けますが、Java では依然として複雑なコードが必要でしたね。

ここで登場する Loom の仮想スレッドは、**WebFlux 並のスケーラビリティを持ちながらも、従来通りの「同期的なコード」で記述できる**という点で画期的です。コード自体も、既存のスレッドベースのコードとほぼ同じままです。

Java 21 で仮想スレッドが正式に導入され、[Spring MVC の VirtualThreadTaskExecutor](https://docs.spring.io/spring-framework/reference/integration/virtual-threads.html) などがその代表例です。従来の Servlet ベースの構成に最小限の変更で仮想スレッドを導入可能であり、[WebFlux よりも低レイテンシかつ高スループットを実現](https://github.com/chrisgleissner/loom-webflux-benchmarks)できます。

Netty や Tomcat など、一部のアプリケーションサーバも Loom 対応を進めており、今後は仮想スレッドが標準的な選択肢となっていくのではと思います。

## まとめ

以下に、この記事で紹介した JVM の主要な改善点をまとめます。

* [Graal](https://www.graalvm.org/) や [Leyden](https://openjdk.org/projects/leyden/) により JIT 不安定性や warm-up 時間を削減
* [ZGC](https://wiki.openjdk.org/display/zgc/Main), [Shenandoah](https://wiki.openjdk.org/display/shenandoah/Main), [LXR GC](https://arxiv.org/abs/2210.17175) により低レイテンシなヒープでの GC を実現
* [Valhalla](https://openjdk.org/projects/valhalla/) によるメモリ実行効率向上
* [Loom](https://openjdk.org/projects/loom/) による I/O スケーラビリティの高効化

## 最後に

JVM は長い歴史を持つ一方で、近年はその限界も語られがちでした。

でも、ここで紹介した Leyden や Loom のようなプロジェクトは、Java の価値を単なる互換性だけでなく「今のニーズに応える性能基盤」として再構築しようとしています。

特に Kotlin や Scala など、Java 以外の言語でもこのような JVM の改善を活用できるので、これからの進化にも期待が持てますね。

まだまだアプリケーション開発のパラダイムや性能基盤は変わり続けていくもので、これからの先はまたどうなるかわかりませんが、このような技術革新が続くといいですね。

では、また！

[^1]: JITが最適化を行うために必要な初期段階のこと
[^2]: 実行時間のばらつきのこと