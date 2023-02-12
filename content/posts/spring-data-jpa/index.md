---
title: "MyBatisよりJPAが使いたい"
date: 2020-07-10
categories: 
  - spring
image: "../../images/spring.jpg"
tags:
  - database
  - jdbc
  - jpa
  - spring
---

個人的に、クエリーやDBのプラグインなど、DBそのものによるデータの整形はあまりよくないと思っていて、なんからの「処理」が入る場合にSQL文はなるべく最低限のCRUDに制限し、必要な作業は全てアプリにて処理するような実装をしたほうが設計として望ましい形だと思っています。なぜかというと、システム全体の性能、アプリのDBに対しての独立性、処理の容易さという観点からそちらの方が優れていると思っているからです。

まず性能ですが、DBサーバとAPサーバが物理的に同じ性能のマシンを使っているとしたら、実際は最適化されたクエリーでパフォーマンスをあげることは十分できますね。しかし、処理すべきデータやリクエストが増えると必ずしもそうとは言えません。Webアプリケーションでユーザ数が増えるとスケールアウト[^1]やスケールアップ[^2]を考慮することになりますが、スケールアウトする場合の場合にDBへの依存が強いアプリは問題を起こす可能性があります。APサーバは複数を利用してもアプリの動作には変わりがありませんが、DBサーバの場合は台数が増えるとデータの整合性という新しい問題が出てきますね。どのDBにも同じデータが保存されていることを保証したいなら、DBサーバをスケールアウトするよりはAPサーバをスケールアウトした方が良いと思います。なのでこの場合はアプリにデータの「処理」を任せた方が望ましいですね。

次にアプリのDBに対しての独立性ですが、最近は費用の高い[Oralce](https://www.oracle.com/database/technologies)から無料の[PostgreSQL](https://www.postgresql.org)や[MySQL](https://www.mysql.com)に移行するシステムも増えてきていますね。しかし、ここでDBのクエリーやプラグインに依存した部分が多ければ多いほど移行は難しくなる可能性があります。微妙に違うカラムのデータ型から、互換性のないクエリーや機能などを全面的に検討する必要がありますね。なのでSQL文はなるべく単純にした方が、移植生が上がります。

最後に、処理の容易さなのですが、現在DBの主流であるRDBSの場合は、徹底的にデータをいかに効率的に保存するかが最も重要な課題となっているものです。正規化を終えたテーブル構造は、データを加工するには適合していません。なのでアプリでは必要に応じてエンティティを作って使いますが、そのエンティティに合わせてSQLを書く際はまた複雑なJoinやWhereなどを描かなければならなくなりますね。エンティティの中に他のエンティティがま他入っていたりする場合はJoinをするか、しないかのケースまで考えてクエリーを描かなければなりません。

このような理由からして、自分はなるべく複雑なクエリーを排除し、なるべくアプリ中心的に開発できる方法はないかと思っていました。初めはJDBCに触れ、そのあとはMyBatisを使うようになっていましたが、MyBatisを使ってもケース別にクエリーを書く必要があるのはまた同じでした。(プラスで、xmlが必要となることもありますね)最近はテレワークにより通勤時間がなくなり、仕事が終わった後は勉強をかねて自作アプリを実装していますが、ここでより簡単に、またよりアプリに優しいDBのAPIがないかと思っていましたが、そこで発見したのがJPAでした。

## JPAとは

JPAは[Java Persistence API](https://www.oracle.com/java/technologies/persistence-jsp.htm)の略で、Javaが提供する標準APIです。ただ、標準APIだとしても、Interfaceとして定義しているだけなので、これを実装したものが必要となります。JPAを実装したものとして有名なのが[Hibernate](https://hibernate.org)で、これをさらに使いやすく整理したモジュールとして[Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#reference)が存在しています。

だいぶ序論が長くなりましたが、結論的に今回言いたいことが何かというと、Spring Data JPAを使うと他のライブラリやフレームワークを使うよりも楽に開発ができるということです。おそらくSQL中心的な開発に関しては自分と同じようなことを考えていた人が他にもいたらしく、JPAは単純なクエリーを自動で作ってくれたり、自動でテーブルのJoinをしてくれるなど全くSQLに触れなくてもDBを扱うことを可能にしてくれています。同じくORMとしてよく使われているMyBatisに比べても、JPAを使った方がより効率的なのではないかな、と個人的には考えています。その理由からか、海外では他のORMよりもJPAを使っている割合の方が圧倒的に高いようでした。なのでもし今もMyBatisを使っている方がいらっしゃるなら、ぜひJPAを一度は使ってみて欲しいところです。

では、実際にJPA(Spring Data JPA)の特徴と使用法を、一つづつ紹介しましょう。

### JPAの特徴

まずはJPAの特徴です。正確には特徴というよりは、JPAを使う場合のメリットに近いのではないかと思います。

#### 自動グラフ探索

まず、このようなクラスがあるとします。

```java
class Team {
    private long id;
    private String name;
}
```

そしてチームには所属された選手がいます。

```java
class Player {
    private long id;
    private String name;
    private Team team;
}
```

既存のSQLでこの選手オブジェクトをDBから照会する場合は、以下の手順が必要となります。

1. TeamテーブルをSelectする
1. TeamオブジェクトにSelectしたデータを埋め込む
1. PlayerテーブルをSelectする
1. PlayerオブジェクトにSelectしたデータを埋め込む
1. PlayerオブジェクトにTeamオブジェクトをセットする

この手順は、テーブルのJoinが多くなればなるほど増えます。また、SQLもさらに長くなりますね。例えばTeamクラスにRegionというクラスがフィールドとして入り、RegionにはさらにCountryというクラスが入るとしたら？それぞれのテーブルを照会してJoinするようなクエリーを作る必要があって、またオブジェクトも全部マッピングする必要がありますね。もしこの手順に何らかの抜け漏れなどがあったら、コードは思い通りに作動しないはずです。

JPAでは、このような参照関係を自動で解決してくれます。例えば、何もしなくても以下のようなことが可能です。

```java
// RepositoryからPlayerオブジェクトを取得する
Player player = repository.findById(1);
// Playerが所属したTeamを取得する
Team team = player.getTeam();
```

### 性能の最適化

自動的にグラフ探索をしてくれるところをみると、まさかPlayerテーブルを照会するたびにTeamテーブルも照会しているような無駄なことをしているわけではないか、と思われそうですが、実際はLazy Loadingにより、Teamは`getTeam()`を実行した時点でまたのSelect文を発行して照会していることになります。なので、Playerだけの情報が必要な場合はその情報しか照会しないことになります。

では、場合によっては数回もトランザクションが発生するのではないか、という疑問もあり得ると思いますが、これをJPAではキャッシュとバッチ、フェッチタイプの指定で解決しているようです。これらは以下のように作動します。

- キャッシュ：同じエンティティに対して(同一トランザクションならば)2回目以降のSelectはキャッシュから照会する
- バッチ：バッチメソッドを提供し、一回のコミットで複数のクエリを転送する機能を実装している
- フェッチタイプ：アノテーションによりLazy Loadingをするか、最初からJoinして照会するかを選択可能

#### 自動クエリ生成

先にRepositoryからオブジェクトを取得する例を照会しましたが、これが可能になるのはJPAが自動的に生成してくれるクエリーがあるからです。JPAが提供するアノテーションやインタフェースを使うとこで、自動生成されるクエリーを利用することができます。

つまり、エンティティとして利用するクラスとJPAをアノテーションとインタフェースに紐づけておくといちいちクエリーを作成してオブジェクトにマッピングする必要がなくなるということです。もちろん自作クエリーを使うこともできるので、自動生成されるクエリーを使いたい場合も問題ないです。

このクエリーの生成は、エンティティのCRUDだけに限ることではありません。エンティティに変更が生じた場合(フィールドを追加する必要がある場合)は、DDLも自動的に作成してくれます。ここで自動生成されるDDLをDrop-Createにするか、AlterにするかはアプリケーションプロパティのHibernate設定でできます。

### JPAの使用法

では、実際のJPAをコードとしてはどう使えるかをみていきましょう。

#### Entity

JPAでは、エンティティとそのフィールド(テーブルのカラムや参照関係)をアノテーションによって簡単に定義することができます。例えば以下のようになります。

```java
// Entityとして指定する
@Entity
class Car {

    // 自動的に増加するPK
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    // UniqueかつNonNull設定
    @Column(nullable = false, unique = true)
    private String name;

    // Companyとは多対一の関係
    @ManyToOne
    @JoinColumn(name = "company_id")
    private Company company;

    // Insuranceとは一対多の関係で、Lazy Loadingをする
    @OneToMany(fetch = FetchType.LAZY)
    @JoinColumn(name = "insurances_id")
    private List<Insurance> insurances;
}
```

こうやってEntityにはGetter・Setterをつけておくことで、Selectした時点でこのクラスに紐づくCompanyはInsuranceのようなテーブルの情報も参照できるようになります。

#### Repository

JPAでは、いくつかのRepositoryというインタフェースを提供しています。よく使われているものは、[JpaRepository](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/JpaRepository.html)や[CrudRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html)、[PagingAndSortingRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/PagingAndSortingRepository.html)などがあって、それぞれ基本的に提供しているメソッドが異なります。例えばCrudRepositoryは普通にCRUDのためのメソッド(Selectに該当するfind()、Upsertに該当するsave()、Deleteに該当するdelete()など)を基本的に提供していて、PagingAndSortingRepositoryの場合はページング(ページネーション)に対応したメソッドを提供しているなど、基本的によく使われるメソッドが揃っています。例えば以下のようなものです。

```java
// レコードを数える
long count = repository.count();

// 全レコードを取得
Iterable<Car> cars = repository.findAll();

// レコードを一件取得する
Optional<Car> car1 = repository.findById(1);

// レコードを一件削除する
repository.delete(car2);

// レコードを保存する(Upsert)
Car saved = repository.save(car3);
```

最初エンティティを定義して、それぞれのエンティティに合わせてJPAが提供するRepositoryを作成すると(extend)、そのRepositoryを使ってクエリを発行することができるようになります。また、基本的に提供しているメソッドが十分でない場合は、命名規則に従ってRepositoryに直接[クエリーメソッド](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods)を書くと自動的にクエリーを作成してくれたり、[@Query](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.at-query)アノテーションで必要なクエリーを直接書くこともできます。

```java
public interface CarRepository extends CrudRepository<Car, Long> {

    // nameで照会するカスタムクエリーメソッド
    boolean existsByName(String name);

    // アノテーションにクエリーを指定する場合
    @Query("select c from Car c where c.name = ?1")
    Optional<Car> findByName(String name);
}
```

## 他に

2018年には、[Spring Data JDBC](https://spring.io/projects/spring-data-jdbc)というものも発表されています。Spring Data JPAとJDBCのAPIの間にはJPA + Hibernateという階層を経由することに比べ、Spring Data JDBCは直接JDBCのAPIを呼び出すことで性能面での利点があるらしいですね。他にも実装をより単純化することでJPAの持つ問題を解消しているらしいですね。例えばキャッシュやLazy Loadingをなくしたとか(JPAのメリットとして紹介しましたが…)

ただ、Spring Data JDBCは2018年にリリースされたばかりのものなので、クエリーメソッドだけでクエリーを自動生成してくれるなどの機能には対応してなかったり、テーブルやカラムの命名規則がJPAと違うなど、JPAからマイグレートするにはまだいろいろと考慮しなければならない点が多いように思われます。こういうところは、まるでNode.jsとDenoの関係みたいですね。

Spring Data JDBCは自分でもあまり詳しくなく、サンプルコードをみてテストしてみただけなのですが、すでにSpring Data JPAの経験がある人ならすぐにでも適応できそうな構成となっていました。パッケージは変わってもアノテーションやインタフェースそのものの使い方が大きく変わっている訳でもないので、感覚的には大差ありません。`@Entity`のようなアノテーションを使う必要がなくなったというところが目立つくらいでした。なので、個人的なプロジェクトでは一度試してみるのも良いかもしれません。(エンタープライズレベルでは、まだ検証すべきところがあると思うので)

また、Spring 5からReactiveなプログラミングのできる[Webflux](https://docs.spring.io/spring-framework/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/web-reactive.htm)が紹介されたことと共に、同じくNon-blockingに対応する[Spring R2DBC](https://spring.io/projects/spring-data-r2dbc)というものも注目されているようです。既存のSpringプロジェクトにWebfluxを今すぐ導入する必要はないらしいのでSpring R2DBCも今すぐ導入する必要はないのかもしれませんが、Non-blockingによって性能を改善できる場合[^3]は確実に存在するので、場合によっては導入を考慮する価値があるのかもしれませんね。

## 最後に

自分の場合、Springでアプリケーションを作りながらなるべく`XML`を排除しJavaのコードだけで全てを管理したかったため、クエリーを自動生成してくれるというところで惹かれていましたが、実際に使ってみると使い方も簡単で(癖はあると思いますが)、テーブルが途中で変わっても簡単に直せるというところがかなり魅力的なものですね。占有率としては標準ということもあり、JPAが他のORMと比べよく使われているようですが、自分みたいにMyBatisしか扱ったことのない方には一度触れてみて損はないかと思います。

JPAの他にも紹介したようにSpring Dataだけでもいろいろなライブラリが存在していてそれぞれの思想が違うので、やっと一つを少しわかったくらいではまだまだ先が遠いような気もしていますね。でもこうやって、一つのライブラリが作られた経緯や思想を調べるだけでも得られる知識は多く、どれも無駄にはならないと思われるくらい新鮮なものなので、意味のあることとなっていると思います。またこうやって情報を集めつつ、それが誰かにとっては貴重な情報になるといいですね。今後もいろいろ調べてブログに載せて行きます。では、また！

[^1]: サーバの台数を増やすこと。
[^2]: サーバのマシンパワーをあげること。
[^3]: スレッドプールが小さすぎたり、一つのリクエストに対して処理時間が長すぎてボトルネックになるケースなど
