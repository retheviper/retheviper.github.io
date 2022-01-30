---
title: "Kotlinで書いてみた〜その一〜"
date: 2021-03-28
categories: 
  - kotlin
image: "../../images/kotlin.jpg"
tags:
  - kotlin
  - java
---

こないだはGoに関するポストを作成しましたが、やはり本業はKotlinなので、Kotlinに関しても何かわかったことや閃いたことなどあれば、順次に書いていこうと思っています。今回はKotlinでAPIを作りながら、業務での要件をどんなコードで満たしたかを簡単に説明させていただきたいと思います。

サーバサイドエンジニアをやっていると、要求される機能を以下に実現している方法がどんなものあれ(GraphQL、REST API、マイクロサービスみたいな技術やアーキテクチャの観点の以前の話として)、業務としてはある程度パターン化しているように感じることがあります。こういう場合には、コードよりもロジックが大事であるかのように見える場合もありますね。でも逆に、むしろ似たようなロジックが多いので、より良いコードを書くために工夫できる余地もまた多いのではないか、とも思います。

正直自分はアルゴリズムに強いわけでもないので、効率的なコードを書くとしたら限界はあるだろうなという気はしています。とりあえず動くコードを書いて、それをリファクタリングしながら少しづつ整える感じのことしかできないのかも知れません。

しかし、そんな自分にも良いコードを書くためにできることが全くないわけでもないと思います。例えば、Javaでコードを書くときは、参照の問題などからなるべく`final`をつけてオブジェクトを[immutable](https://ja.wikipedia.org/wiki/%E3%82%A4%E3%83%9F%E3%83%A5%E3%83%BC%E3%82%BF%E3%83%96%E3%83%AB)にするようにと教わりましたが、実際は[ベンチマークで比較した結果](https://www.baeldung.com/java-final-performance?__s=m4suw1p9x2sbizbhxrew)でもわかるように、性能の改善にも繋がっています。また、JavaでもKotlinでも色々と便利なAPIを提供していて、バージョンアップの度にまた新しいAPIが追加されるので、それらの用途と使い方をよく理解した上で、積極的に使用するだけでも読みやすく、性能も良いコードを書くことができます。

ということで、今回はKotlinのAPIを使って書いていたコードを一部紹介したいと思います。

## リストのグループ化

DBに商品情報テーブルがあって、さらに商品属性テーブル、生産地や販売店テーブルなどがある場合に、業務によっては「販売店ごとにどんな商品が販売されているかを確認したい」とか、「特定の商品属性に当てはまる商品だけみたい」とかのケースがあるはずですね。

そういった場合、APIとしてはテーブルから取得したデータを、特定のカラムを基準にまとめたもの返す必要があります。これをコードに書くとしたら`List`で取得したデータを、中の一つの属性をキーに`Map`にまとめて返すということになりますね。Javaだと、以下のような形になるかと思います。

```java
// DBのデータの例
List<User> list = List.of(
        new User("John", 20, "USA", "Programmer"),
        new User("James", 30, "Canada", "Sales"),
        new User("Jack", 35, "UK", "Programmer")
);
// UserのJobを基準にまとめる
Map<String, List<Pair>> map = list.stream()
        .collect(Collectors.groupingBy(User::getJob,
                Collectors.mapping(user -> new Pair(user.getAge(), user.getName()), Collectors.toList())));
// {James=[Pair(first=30, second=Sales)], John=[Pair(first=20, second=Programmer), Pair(first=35, second=Writer)]}

@Data
@AllArgsConstructor
static class User {
    private String name;
    private int age;
    private String address;
    private String job;
}

@Data
@AllArgsConstructor
static class Pair {
    private Object first;
    private Object second;
}
```

KotlinでもJavaのAPIをそのまま使うことができるので、上記の`Stream`と`Collector`を使って同じことはできます。ただ、せっかく違う言語と使っているわけなので、できればKotlinが提供するAPIを活用して同じことをしたいものです。

KotlinはCollectionで提供する機能だけでも`Stream`と`Collector`を組み合わせたものと似たような処理ができる場合が多いので、JavaのAPIに対応した機能があるかどうかを探すだけで事足りるケースが多いです。ということは、上記の処理でキモになっている`Collectors.groupingBy()`と`Collectors.mapping()`と似たようなものがあればいいというわけですが、`groupBy()`でそれらの処理をまとめることができます。なので、上記のコードをKotlinで変えると、以下のようになります。色々とスッキリしますね。

```kotlin
// DBデータの例
val list = listOf(
    User("John", 20, "USA", "Programmer"),
    User("James", 30, "Canada", "Sales"),
    User("Jack", 35, "UK", "Programmer")
  )
// Jobを基準にMap<String, List<Pair<Int, String>>>にまとめる
val map = list.groupBy({ it.job }, { it.age to it.name })
// {Programmer=[(20, John), (35, Jack)], Sales=[(30, James)]}

data class User(
    val name: String,
    val age: Int,
    val address: String,
    val job: String
)
```

## Mapのvalueだけを変える

上記の処理に加えて、もっと条件がつく場合もあるかと思います。例えば、金額計算とかの例があるとします。従業員が案件ごとに賃金をもらうということになっていて、案件はコードで管理されている場合、賃金を払う側としては同じ案件に対しては合算した金額のみが知りたいとかのケースもあるでしょう。こういう場合には、従業員ごとにデータをまとめた上で、さらにその人が担当した案件のリスト野中で重複するものがあれば、金額だけを合算するようにする必要がありますね。

こういう場合は、グルーピングの段階からそういう処理を入れるのがもっとも効率的ではあるとは思いますが、スレッドの問題もあるので(生成中のMapの中を巡回するという)、実際のコードに書くとするとかなり複雑になる可能性もあります。なのでここではまず、`List`を`Map`にまとめた結果を持ってさらに処理を加えるという形を取ります。

Kotlinの`Map`には、`map()`以外にも`mapKeys()`や`mapValues()`のような関数があって、必要な部分だけをマッピングできます。今回は`value`だけを変えたいので、`mapValues()`を使った方が無駄がなく、コードを読む側としても意図が明確になって良いと思います。`mapValues()`を使ってさらにマッピングを行うコードは、以下のようになります。

```kotlin
data class User(val name: String, val id: Int, val amount: Int)

// DBデータの例
val list = listOf(
    User("A", 1, 1000),
    User("A", 1, 2000),
    User("A", 2, 4000),
    User("B", 3, 5000)
)
// nameでまとめた後、重複するidを一つにまとめる(amountを合算)
val map = list.groupBy({ it.name }, { it.id to it.amount })
    .mapValues {
        // idでグルーピング
        it.value.groupBy { pair -> pair.first }
        // keyはそのまま、valueだけを合算する
            .map { map -> map.key to map.value.sumBy { pair -> pair.second } }
    }
// {A=[(1, 3000), (2, 4000)], B=[(3, 5000)]}
```

`List`を`Map`にまとめるもう一つの方法は、`groupingBy()`があります。この関数を使うと、Collectionが[Grouping](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-grouping)というオブジェクトに変わって、`aggregate()`・`reduce()`・`fold()`・`eachCount()`のような関数を使うことで後続の処理ができます。上記のコードを`Grouping`を使ったものに変えるとしたら、以下のようになります。

```kotlin
// Groupingのaggregateを利用してMapに変えた後から、valueの処理を行う
val map = list.groupingBy { it.name }
    .aggregate { _, accumulator: MutableList<Pair<Int, Int>>?, element, first ->
        // 新しいキーなら、MutableListを作る
        if (first)
            mutableListOf(element.id to element.amount)
        // そうではない場合は、存在するListに要素を追加する
        else
            accumulator?.apply { add(element.id to element.amount) }
    }.mapValues {
        it.value?.groupBy { pair -> pair.first }
            ?.map { pair -> pair.key to pair.value.sumBy { pair -> pair.second } }
    }
```

一見、`groupingBy()`の方が複雑にも見えますが、`accumulator`を使ってマッピングした値を積み重ねることができるので、場合によっては考慮する価値があるかもですね。

## Mapを使ったキャッシュ

DBの参照が頻繁であり、なお参照されるデータそのものは更新される頻度が高くない場合は、アプリケーション内にキャッシュして置くのが良いケースもたまにありますね。こういう場合には、パラメータをキーとして持つ`Map`を宣言しておいて、そのキーがない場合だけDBにアクセスする(そして`Map`に追加する)という形にすれば良いでしょう。Javaでは1.8から`computeIfAbsent()`というメソッドを提供しているので、簡単に実装ができます。例えば以下のようになります。

```java
// DBデータの例
List<String> list = List.of("A", "B", "C");
// キャッシュのMap
Map<String, Boolean> map = new ConcurrentHashMap<>();
// パラメータ
String element = "A";

// キャッシュにパラメータがない場合はDBデータを参照して、追加した後に返す
Boolean exists = map.computeIfAbsent(element, key -> list.contains(element));
// Method Referenceを使った例
exists = map.computeIfAbsent(element, list::contains);
```

Javaで提供する機能なので、もちろんKotlinでも全く同じ形で実装できます。ただ、Kotlinの仕様上`compute`のコードが[LambdaかMethod Referenceかによって書き方が違う](https://kotlinlang.org/docs/lambdas.html#instantiating-a-function-type)ので、そこだけ注意する必要があります。これはKotlin自体の仕様によるものですが、Javaの書き方に慣れていると最初はなかなかわかりにくいところかも知れません。

```kotlin
// DBデータの例
val list = listOf("A", "B", "C")
// キャッシュのMap
val map = ConcurrentHashMap<String, Boolean>()
// パラメータ
val element = "A"

// Lambdaの場合
var exists = map.computeIfAbsent(element) { list.contains(element) } // false
// Method Referenceの場合
exists = map.computeIfAbsent(element, list::contains)
```

ちなみに、似たような機能をするメソッドとして`putIfAbsent()`がありますが、`computIfAbsent()`の場合`Map`にキーがなかった場合にだけ後続の処理が行われるに対して、`putIfAbsent()`はキーがあるかないかに関係なく処理が走ってしまうという違いがあるらしいです。なのでキャッシュとして使う場合は、`computeIfAbsent()`を使った方が良いでしょう。

## 最後に

自分が書いたコードをいくつか紹介しましたが、いかがだったでしょうか。まだKotlinに移行したばかりなので色々とわからないことが多く、本当はもっとスマートな方法があるのかも知れませんが、自分的には、こうやって実際の業務の要件に合わせて違う言語とコードを比べながら、APIのソースをみたりで自分なりにどうやって書くかを考えてみるのは意味のあることで、楽しいとも思います。

というわけで、これからもKotlinでの書き方に対する研究はこれからも続きます。そろそろGoでも簡単なAPIでも作ってみたりで勉強をしないとやばそうな気もしていますが…まぁ、なんとかなるでしょう。

では、また！
