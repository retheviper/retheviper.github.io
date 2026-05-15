---
title: "実プロダクトで踏んだJPA / Hibernateの落とし穴"
date: 2026-05-15
translationKey: "posts/jpa-hibernate-pitfalls"
categories:
  - quarkus
image: "../../images/quarkus.webp"
tags:
  - kotlin
  - quarkus
  - jpa
  - hibernate
  - jooq
---

最近、ある業務システムの更新系APIでレイテンシ調査とエラー対応をする機会がありました。構成としてはKotlin + Quarkus + Hibernate ORMで、もともとはHibernateを中心に作られたプロジェクトです。ただ、N+1や更新時の暗黙挙動が重くなってきたため、一部ではjOOQも導入していて、今はHibernateとjOOQを混用して運用している状態です。

調査してみると、業務ロジックそのものよりも、JPA / Hibernateの振る舞いに起因するコストがかなり多いことが分かりました。もちろんHibernateが悪い、JPAが絶対に使えない、と言いたいわけではありません。ただ、実プロダクトで更新系APIのp90が秒オーダーになった時に、実際に何が起きていたかを整理しておく価値はあると思いました。

ちなみに、以前このブログでは[「MyBatisよりJPAが使いたい」](../spring-data-jpa/)という記事を書いていました。当時は、SQLをなるべく単純にしてアプリケーション側のエンティティを中心に扱えることをかなり魅力的に感じていました。今でもその発想自体が完全に間違っていたとは思っていません。ただ、実プロダクトで更新系のホットパスを追ってみると、「SQLを書かなくて済む」ことの裏側にある暗黙のSELECT、flush、refresh、lifecycleのコストを無視できなくなった、というのが今の感覚です。

今回は具体的な業務ロジックは伏せつつ、実際に踏んだ落とし穴と、その時に取った回避策を書こうと思います。

## レスポンスDTO構築でlazyプロキシが連鎖的に初期化される

最初に大きく効いていたのは、更新処理の最後でエンティティをレスポンスDTOに変換していたことでした。

PUTやPOSTの最後に、更新後のエンティティに対して`toResponseDto()`のような関数を呼びます。すると、その中で`@ManyToOne(fetch = LAZY)`や`@OneToMany`、`@ManyToMany`のgetterが順番に呼ばれます。Hibernateから見ると、それはlazyプロキシを初期化する必要がある、という意味になります。

結果として、CloudWatch Performance InsightsのTop SQLには、業務ロジック本体ではなく関連エンティティのSELECTが大量に並びました。しかも子エンティティのDTO構築がさらに自分のlazy関連を触るため、二段階でSELECTが連鎖します。

ここで取った対処はかなり単純で、更新系のレスポンスを`{ id, version }`のような最小DTOに変えました。詳細表示が必要なら、別途GETで取り直す契約にするということです。getter自体を呼ばなければ、lazy初期化も物理的に起きません。

## CascadeType.REFRESHで関連collectionが全部読まれる

次に効いていたのが`CascadeType.REFRESH`でした。

共通基底リポジトリに、永続化して、flushして、refreshする、という便利メソッドがありました。イメージとしては以下のようなものです。

```kotlin
entityManager.persist(entity)
entityManager.flush()
entityManager.refresh(entity)
```

これ自体は、楽観ロックやDB側で決まる値を取り直したい時には便利です。ただ、対象エンティティの関連に`CascadeType.REFRESH`が付いていると話が変わります。`refresh()`時にHibernateは対象エンティティを再ロードし、REFRESH cascade対象のlazy collectionを初期化し、各要素もrefreshします。

つまり「関連にもrefreshを伝搬させたい」と一箇所に書いたつもりが、更新のたびに関連collectionの読み込みコストを支払う構造になっていました。業務上は触っていないcollectionまでSELECTされるので、かなり分かりにくいコストです。

対処としては、基底リポジトリにrefreshしない兄弟メソッドを用意しました。性能クリティカルな経路ではそちらを使います。テストではHibernateの`Statistics`を使い、`getCollectionFetchCount()`などでcollection fetchが増えないことを確認できます。

## orphanRemovalとclear + addAllで全置換になる

`@OneToMany(orphanRemoval = true)`のcollectionを持つ集約でも問題がありました。更新時に以下のような書き戻しをしていたのです。

```kotlin
children.clear()
children.addAll(newChildren)
```

この書き方をすると、Hibernateは既存要素を孤児としてDELETEし、新規要素をINSERTします。差分が1行だけでも、既存全行DELETE、新規全行INSERTになります。業務的には一部だけ変わったつもりでも、DB上は全置換です。

この経路はjOOQでの差分適用に置き換えました。`INSERT ... ON CONFLICT`、`UPDATE`、`DELETE WHERE NOT IN (...)`のように、必要な差分だけをSQLで明示的に書く形です。

ここで大事なのは、GET系で関連グラフを返すためのJPA mappingをただちに全部捨てる必要はない、ということです。読み取りでは便利なmappingを残しつつ、更新経路だけはSQLビルダに逃がす、という分け方も現実的でした。

## addAllだけでもlazy collectionが初期化される

似ていますが、もっと地味にハマったのが`addAll`単体です。

履歴系の子テーブルに追記しているだけのつもりなのに、更新経路で毎回既存の履歴全件がSELECTされていました。中にはJSONカラムも含まれていたため、読み込みコストは無視できません。

原因は、親エンティティのヘルパーが内部でcollectionに`addAll`していたことでした。Hibernateの`PersistentBag`は`addAll`前に既存要素との等値比較を行う必要があるため、collectionを初期化します。つまり、`clear() + addAll()`の全置換だけでなく、追記用途の`addAll`単体でも既存collectionを読みに行くことがあります。

対処は、追記系の子テーブルへの書き込みではcollectionを一切触らず、jOOQで直接INSERTすることでした。エンティティのヘルパーを呼ぶと便利に見えますが、その裏でlazy collectionが初期化されるなら、更新経路ではかなり高い便利さです。

## flush + refreshとPreUpdate再発火のリスク

レスポンスDTOを軽くしても、まだ更新APIのレイテンシが残る箇所がありました。ここでも共通基底の`flush + refresh`が効いていました。

単純なラウンドトリップのコストもありますが、より怖いのはlifecycle listenerとの組み合わせです。たとえば`@PreUpdate`で`updatedAt`を上書きする設計になっていると、ハンドラ内で`updatedAt`を読んで別カラムに同期するような素朴な実装はタイミングの問題で壊れます。

また、jOOQで副カラムを更新したあと、その値をmanaged entityに書き戻すと、エンティティがdirtyになります。そのままtransaction commitに進むと再flushされ、`@PreUpdate`が再発火し、`updated_at`だけ余計に進む、という事故が起きます。

ここでは、refreshしない経路を使うことに加えて、jOOQで更新した値をmanaged entityへ書き戻さない方針にしました。副カラムの同期はSQLビルダで直接書き、JPA lifecycleを通さない。in-memoryのエンティティを「きれいに同期したくなる」気持ちはありますが、そこを触ることで別の更新が発火するなら、同期しない方が安全です。

## Hibernate.initializeと同一集約の二重ロード

レスポンスを軽量化したあとに、`Hibernate.initialize(entity.someAssociation)`のような呼び出しも棚卸ししました。

以前はレスポンスDTOのために必要だったのかもしれませんが、最小DTO化したあとは単に消費されないIOです。refresh直後にManyToManyをもう一度SELECTしているような箇所もありました。

さらに、履歴生成のためにエンティティをロードし、その後に共通基底の更新メソッドでもう一度同じエンティティをロードする、といった重複もありました。「ロード済みのエンティティを渡す」メソッドと「IDを渡して内部で取り直す」メソッドが混ざっていると、このあたりのコストは見えにくくなります。

この手の問題は、個別の`initialize`を消すだけではなく、基底メソッドの契約を整理する必要がありました。

## 悲観ロックのラウンドトリップ自体が重い

lazy初期化やrefreshを外しても、更新APIのp90がまだ数百msから秒オーダーで残る経路がありました。

見ると、共通基底リポジトリの「バージョン付き更新」メソッドが、デフォルトで以下のような流れになっていました。

1. `findById(id, LockModeType.PESSIMISTIC_WRITE)`で`SELECT ... FOR UPDATE`
2. `persistAndFlush`
3. `refresh`

cascadeもlazyも触らないケースでも、この三段ラウンドトリップ自体が支配的になります。特に競合がほとんど起きない更新経路では、「念のため」の悲観ロックが毎回1往復分のコストを足しているだけでした。

ここでは更新セマンティクスを見直し、`@Version`による楽観ロックで足りる経路では悲観ロックを外しました。基底メソッドも、「ロックする / しない」「refreshする / しない」を分ける必要があります。便利な共通基底が、暗黙にコストを積み上げていないかはかなり疑った方がいいと思います。

## JSONカラムを別経路で書くと表現がずれる

jOOQへ逃がした時に、別の種類の問題も出ました。`@JdbcTypeCode(SqlTypes.JSON)`のJSONカラムをHibernate cascadeではなくSQLビルダで直接書くようにしたところ、DB上のJSON表現が微妙に変わったのです。

たとえばプロパティ順、`null`フィールドの有無、命名規則などです。後段で比較やハッシュ、テストをしていると、この差分で壊れます。

原因は、Hibernate経由のシリアライズではQuarkusでDIされたJackson `ObjectMapper`が使われていたのに、jOOQ側で自前の`ObjectMapper`を作っていたことでした。`ObjectMapperCustomizer`が効いていないので、同じオブジェクトでもJSON文字列が一致しません。

対処としては、DI済みの`ObjectMapper`を明示的にinjectして、Hibernate側と同じ設定でシリアライズするようにしました。そして両経路の結果がbit-for-bitで一致することをテストで確認しました。

ORMのカラム型変換をSQLビルダ側に移植する時は、SQLだけ見ていると同じに見えます。しかし実際には、シリアライゼーション層の暗黙依存も一緒に移植しないといけません。これは`AttributeConverter`系の列でも同じ問題が起きるはずです。

## detachしたつもりでも永続コンテキストに関連エンティティが残る

一括編集APIでは、別の種類の事故もありました。処理としては、まず対象エンティティをfetch join付きでロードし、更新前の値を保持したうえでjOOQで一括UPDATEし、更新後の値を再ロードして履歴を登録する、という流れです。

この時、更新前の値を保持するために各エンティティへ`entityManager.detach(it)`を呼んでいました。個別entityを永続コンテキストから外せば、後続のjOOQ更新と干渉しないだろう、という考えです。

しかし、`detach()`は指定したエンティティしか外しません。fetch joinで一緒にロードされた関連エンティティや子collectionの要素は、managedのまま永続コンテキストに残ります。その状態でjOOQがDBを直接UPDATEすると、DB上の値とmanaged entityの値が乖離します。次のクエリ発行時やtransaction commit時にHibernateのauto-flushが走ると、残っていたmanaged entityのdirty checkingがDBの最新状態と衝突します。

さらに怖いのは、更新前の値だと思って参照していたものが、実はHibernate管理下の同じインスタンスだった場合です。auto-flushや後続処理の影響で、履歴に入れるbefore値がすでに更新後の値になっている、というメモリ上の不整合も起きえます。

対処としては、更新前の値をplainなdata classのsnapshotにコピーして保持する形に変えました。そのうえで、個別`detach()`ではなく`entityManager.clear()`で永続コンテキスト全体をクリアします。fetch joinで引き連れてきた関連エンティティもまとめて外し、以後の差分判定や履歴before値はsnapshotだけを見るようにしました。

ORMで読んだものをSQLビルダで更新するなら、「読んだentityをそのまま業務ロジックの値として信じない」ことが大事です。必要な値はimmutableなsnapshotに逃がし、永続コンテキストは早めに切る。`detach()`はcascadeしない、という前提を忘れるとかなり危ないです。

## Hibernateを入れている限りKotlinのビルド設定にも影響する

最後はランタイムではなく、ビルドや言語バージョンの話です。

KotlinでJPAエンティティを書く場合、基本的に二つの制約があります。

- 引数なしコンストラクタが必要なので`kotlin-noarg`、または`kotlin-jpa`が必要
- プロキシ生成のためにクラスを`open`にする必要があるので`kotlin-allopen`が必要

これらのプラグインはKotlin本体やGradle plugin、serializationなどの関連プラグインと強く結びついています。結果として、Kotlinのバージョン更新、K2コンパイラ移行、kapt除去のような話題が、JPAプラグイン群の互換性確認に引きずられます。

Hibernate自体を直接触っていない更新作業まで、ORMの存在に律速されるわけです。個人的には、これはORM選定時に見落とされがちなコストだと思います。

## 傾向をまとめる

今回の問題を並べると、かなり一貫した傾向があります。

| カテゴリ | 中核機序 | 回避策 |
|---|---|---|
| lazyプロキシ初期化連鎖 | DTO構築のgetterがSELECTを連鎖発火する | 更新系レスポンスを最小DTOにする |
| cascade REFRESH全伝搬 | `refresh()`が関連collectionを初期化する | refreshしない経路を用意する |
| orphanRemoval全置換 | `clear() + addAll()`で全行DELETE + INSERTになる | jOOQで差分適用する |
| collection暗黙初期化 | `addAll`だけで`PersistentBag`が初期化される | collectionを触らず直書きする |
| flush + refresh | 往復コストと`@PreUpdate`再発火が起きる | refreshを外し、lifecycleを迂回する |
| 悲観ロック | `SELECT ... FOR UPDATE`の1往復が常に乗る | 楽観ロックで足りる経路から外す |
| 重複ロード | 同じ集約を複数層でロードする | メソッド契約を整理する |
| JSON直列化 | ORM経路とSQL経路で表現がずれる | DI済み`ObjectMapper`を再利用する |
| 永続コンテキストとauto-flush | `detach()`が関連entityを外さず、managed entityが後続更新と衝突する | snapshotに逃がして`entityManager.clear()`する |
| ビルド層への波及 | JPA用Kotlinプラグインが更新を律速する | 新規ではORMごと外す選択肢を持つ |

共通しているのは、コード上は小さく見える操作が、Hibernateの暗黙挙動を通じて大きなIOやflushに変換されることです。getter、`refresh()`、`addAll()`、`clear()`、`initialize()`、`PESSIMISTIC_WRITE`、`detach()`。どれも単体では便利ですが、更新系APIのホットパスに入るとかなり重くなります。

## 新規プロジェクトではどうするか

今回の経験を踏まえると、自分なら新規プロジェクトではHibernate ORMを最初から入れず、SQLビルダを中心に作ると思います。このプロジェクトではjOOQを使っています。

これは、以前の自分から見ると、かなり大きな意見の変化です。変わった理由は、JPAの概念を学んだからではなく、実際の運用で更新系APIの遅さを追い、何がいつDBへアクセスするのかを一つずつ剥がしたからです。

もちろん、既存プロジェクトからHibernateを一気に消すのは現実的ではありません。GET系で関連グラフを返すためのmappingが便利な場面もあります。そのため、今のプロジェクトではHibernateとjOOQを混用し、特に更新系のホットパスからHibernateの暗黙挙動を外していく方針にしています。

ただ、新規であれば最初から`@OneToMany`、`CascadeType.*`、`orphanRemoval`、`Hibernate.initialize`、`persistAndFlushRefresh`、`entityManager.detach`のような道具を閉じておく方が安いと感じています。便利そうに見えるものほど、実際に何回SELECTするのか、いつflushされるのか、どのlistenerが走るのか、どのインスタンスがmanagedなのかを読み解く必要が出ます。

一点だけ補足すると、Hibernate ValidatorはORMではありません。Bean Validationのリファレンス実装なので、これは導入しても問題ありません。避けたいのは`quarkus-hibernate-orm`のようなORM本体です。CIで設定キーをgrepするなら、`quarkus.hibernate-orm.*`だけをreject対象にする、という分け方が必要です。

## 最後に

今回踏んだ問題は、個別のバグというより、エンティティ、集約境界、lifecycle listener、共通基底リポジトリがHibernateの暗黙挙動と絡み合った構造的な負債でした。1箇所を直しても、別の場所で同じパターンが再発します。

なので、対処もかなり地味です。更新系レスポンスではgetterを呼ばない。refreshしない。collectionを触らない。悲観ロックをデフォルトにしない。SQLビルダへ逃がす時はシリアライゼーション設定まで揃える。ORMで読んだ値はsnapshotに切り離し、必要なら`entityManager.clear()`で永続コンテキストごと捨てる。便利な共通メソッドほど、中で何をしているかを疑う。

JPAに潜在する地雷の総量は、踏んでみるまで分かりにくいです。今回の調査で、そのコストをかなり実感しました。

では、また！
