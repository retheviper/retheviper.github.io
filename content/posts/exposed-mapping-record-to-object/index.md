---
title: "ExposedでOneToManyをどうマッピングするか"
date: 2021-07-26
categories: 
  - exposed
image: "../../images/exposed.jpg"
tags:
  - kotlin
  - exposed
  - map
---

DBのテーブルとして、1:Nのリレーションは珍しいものではありません。例えば、ECサイトで会員登録をし、複数の配送先を設定できるようにするとしたら、この会員情報のテーブルに配送先のカラムを追加するよりは、配送先のテーブルを分離して別に管理した方がデータの持ち方としては合理的で安全なはずです。そして分離した配送先のテーブルは、会員情報のテーブルとN:1の関係になるのが一般的でしょう。

ただ、データの持ち方が優先的なDBと、そのデータを処理して形にするアプリケーションでは事情が違いますね。例えば上記の通り、一つの会員情報のレコードに対して複数の配送先のレコードが存在し得る場合、SQLでデータを表現すると、以下のような形になるはずです。

```text
|-----------|-------------|-----------------|
| member.id | member.name | mailing.address |
|-----------|-------------|-----------------|
|         1 |        John |           Tokyo |
|         1 |        John |        New York |
|         1 |        John |         Beijing |
|         2 |     Simpson |           Osaka |
|         2 |     Simpson |          Nagoya |
|-----------|-------------|-----------------|
```

しかしアプリケーションではこのような形でデータを扱うことはあまりないですね。一つのレコードに対して複数のレコードが含まれるということは、Kotlinだと以下のように表現するのが一般的かと思います。

```kotlin
data class Member(
    val id: Int,
    val name: String,
    val mailingAdress: List<String>
)
```

そして普通は、このようなオブジェクトをJSONの形にしてREST APIのレスポンスとして使う場合が多いですね。なので、先程のレコードをJSONにした場合は以下のようになるはずです。

```json
{
    "members": [
        { 
            "id": 1,
            "name": "John",
            "mailingAddress": [
                "Tokyo",
                "New York",
                "Beijing"
            ]
        },
        {
            "id": 2,
            "name": "Simpson",
            "mailingAddress": [
                "Osaka",
                "Nagoya"
            ]            
        }
    ] 
}
```

ここで問題は、オブジェクトをJSONに変えることはそう難しくないのですが（[Jackson](https://github.com/FasterXML/jackson), [Gson](https://github.com/google/gson), [Kotlin Serialization](https://github.com/Kotlin/kotlinx.serialization)のようなライブラリがすでにありますし）

では、このような場合、DBから取得したレコードをどうやってオブジェクトにマッピングしたら良いのでしょう。JPAのようなORMを使う場合、クラスにテーブル間の関係を表すフィールドとアノテーションを適切に使うことでレコードへのマッピングは自動に行われますが、[jOOQ](https://www.jooq.org)や[Querydsl](https://querydsl.com), [Exposed](https://github.com/JetBrains/Exposed), [Ktorm](https://www.ktorm.org)のようなORMで、DSLを使ってSQLを書く場合はデータのマッピングを手動で行う必要があります。そして取得したデータは行の配列という形になるので、どうマッピングしたら（効率が）いいかは少し悩ましいところです。

なので、今回はExposedのDSLを使って取得したOne to Manyのレコードを、コード上でどうやってオブジェクトにマッピングするかについて考えたことを述べたいとお思います。

## テーブルごとにSelectする

もっとも簡単な方法は、そもそもレコードの取得時に個別のテーブルに対してSelectしてからマッピングすることですね。個別のテーブルに対してクエリを発行するので、書き方としては明瞭になるというメリットもあります。例えば、以下のようなことができます。

```kotlin
transaction {
    // まずはMemberテーブルをSelectし、オブジェクトにマッピングする
    val member = Member.select { Member.id eq id }
        .first()
        .let {
            MemberDto(
                id = row[Member.id].value,
                name = row[Member.name],
                role = listOf(row[Mailing.role])
            )
        }

    // MailingテーブルをSelectし、リストにする
    val mailingAddress = Mailing
            .select { Mailing.memberId eq member.id }
            .map { it[Mailing.address] }

    // オブジェクトのコピーを作り配送先のデータをマッピング
    member.copy(mailingAddress = mailingAddress)
}
```

もっとも簡単で、コードとしてもわかりやすい方法ではありますが、トランザクションの観点からするとあまりよくない方法ですね。Exposedでは、`transaction`ブロックに包むことでトランザクションの単位を制御できるものの、一回で済ませるクエリの発行が複数になるという問題があります。ここではMemberテーブルを照会するたびにMailingテーブルも照会することになるので1回のクエリが追加されているだけですが、もしMemberテーブルに1:Nの関係となっているテーブルが増えれば増えるほどクエリの発行数も増えることになるでしょう。そして今回は一つのレコードに対してのコードとなっていますが、照会対象のMemberテーブルのレコードが増えれば増えるほど発行されるクエリの数も多くなります。

また、オブジェクトのインスタンスを作っておいて、わざわあコピーするというのもあまり効率が良いとは言えません。これもまたクエリの数が増える問題と同じく、照会対象のレコードが増えれば増えるほど生成されるオブジェクトのインスタンスも増えることになるという訳ですね。なので、全く性能や効率を考えてないコードと言えます。

## joinしてマッピングする

関係のあるデータを複数のテーブルを跨いで取得するには、やはり`join`が効率的でしょう。これならまず発行されるクエリの数は個別のテーブルに対してSelectする時に比べ、劇的に減ります。アルゴリズムでよく使われる表現の[Big O記法](https://vmm.dev/ja/cci/cci-0.md)で表現すると、前者は`O(N^2)`であり、これは`O(1)`と表現できるはずです。

ならばデータを取得する際に、クエリとしては`join`を使うのが理想的なのはわかりますが、問題はそうやって取得したデータをどう加工するかです。先に述べましたが、取得したデータのうち重複するものがあるからですね。そしてこれが重複しているかどうかはクエリを実行した結果を取得して、確認するまではわかりません。

ここで考えられる方法は三つほどありますので、一つづつ紹介していきます。

### reduce

まずはクエリの結果として取得した行を、それぞれオブジェクトにマッピングした後、`reduce`でまとめる方法です。例えば以下のようになります。

```kotlin
transaction {
    Member.leftJoin(Mailing)
        .select { (Member.id eq id) and (Mailing.memberId eq Member.id) }
        .map {
            // とりあえずオブジェクトにマッピングする
            MemberDto(
                id = it[Member.id].value,
                name = it[Member.name],
                mailingAddress = listOf(it[Mailing.address])
            )
        }.reduce { acc, memberDto ->
            // オブジェクトを一つに集約させる（mailingAddressは累計）
            acc.copy(
                mailingAddress = acc.mailingAddress + memberDto.mailingAddress
            )
        }
}
```

このやり方で考えられる問題は、まず行数分のオブジェクトのインスタンスが作られるということです。このクエリとして取得しようとしているMemberのレコードは一つのみですが、そのレコードに紐づくMailingのレコードが多ければ多いほど件数は増え、当然生成されるオブジェクトの数も多くなります。また、マッピングだけでなく、`reduce`でもオブジェクトをコピーしているので、やはり行数分のオブジェクトが生成されていると考えられます。オブジェクトのインスタンス数は行数x2になる訳ですね。

そしてもう一つの問題は、Memberのレコードを複数取得する場合、全部一つのオブジェクトにまとまってしまうという問題がありますね。なので、このやり方だと一つのレコードを取得する場合のみしか適応できなくなります。

### groupBy

取得したレコードを、一度Mapに変換するとどうでしょうか。KotlinのCollectionには`groupBy`というメソッドがあり、keyとvalueのマッピング方法を指定すると、一つのkeyに`List`形式のvalueになります。Mapなので、keyでMemberのオブジェクトをマッピングしておいて、valueとしてはMailingのレコードをまとめておくと良いでしょう。keyは同じものだと上書きされるので、取得したいMemberのレコードが複数の場合でも問題ないはずです。コードでは、以下のようになります。

```kotlin
transaction {
    Member.leftJoin(Mailing)
        .select { (Member.id eq id) and (Mailing.memberId eq Member.id) }
        // keyはMemberオブジェクトのマッピング、valueではMailingのレコードを集約
        .groupBy({
            MemberDto(
                id = it[Member.id].value,
                name = it[Member.name],
            )
        }, { it[Mailing.address] })
        // keyのオブジェクトにMailingのレコードを設定
        .map { (key, value) ->
            key.copy(mailingAddress = value)
        }
}
```

この方法だと、今まで照会した他の方法で考えられる問題をだいぶ解消できそうな気がしますね。ただ、気になるのは、`groupBy`の引数がLambdaであることです。引数として関数を渡すということは、ループしながらその関数を実行することになるという意味なので、`reduce`の時と同じ量のインスタンスが作られる可能性がありそうですね。なので、`groupBy`の実装を見ていきたいと思います。中のコードは、以下の通りです。

```kotlin
public inline fun <T, K, V> Iterable<T>.groupBy(keySelector: (T) -> K, valueTransform: (T) -> V): Map<K, List<V>> {
    return groupByTo(LinkedHashMap<K, MutableList<V>>(), keySelector, valueTransform)
}
```

`groupBy`の実装では、`groupByTo`という関数に自分の引数と、作られるMapのインスタンスを渡しているだけですね。では、さらに`groupByTo`の中身を見ていきましょう。

```kotlin
public inline fun <T, K, V, M : MutableMap<in K, MutableList<V>>> Iterable<T>.groupByTo(destination: M, keySelector: (T) -> K, valueTransform: (T) -> V): M {
    for (element in this) {
        val key = keySelector(element)
        val list = destination.getOrPut(key) { ArrayList<V>() }
        list.add(valueTransform(element))
    }
    return destination
}
```

ここで確かになっていることは、やはり最初のCollectionの要素数分、`keySelector`と`valueTransform`を実行しているということです。Mapに変えることになるので、`reduce`の時とは違ってMemberレコードがいくつあっても一つに集約されるような事態は起こらないと考えられますが、依然としてオブジェクトのインスタンスが複数できてしまうという問題はあります。なので、また他の方法を探してみましょう。

### Map

最後に考えられるのは、Selectした行をMapにまとめるのではなく、外部にMapを宣言し、それを利用することです。Mapには、`compute`という関数があり、引数として渡したkeyに対してどんな処理をするか（どんなvalueを作って入れるか）を指定できます。例えば、指定したkeyに対してvalueが存在しない場合は要素として追加し、存在する場合はそのvalueを変えるなどの処理ができるようになります。なので、これをうまく使うとインスタンスの生成問題を解決できる気がしますね。

トランザクションとは関係のないMapをまず宣言し、Selectしたデータに対して`compute`を実行することにします。`compute`では指定したkey（Memberのidなど）がMapの中にない場合にMemberのインスタンスを作成するようにして、すでにある場合はそのオブジェクトにMailingのデータを追加するようにすれば良いでしょう。そしてループが終わったらMapのvalueのみを取得すると良いですね。

以上のことを、コードで表すと以下のようになります。

```kotlin
// オブジェクトをまとめるためのMap（keyはMember.id）
val helperMap = mutableMapOf<Int, MemberDto>()

transaction {
    Member.leftJoin(Mailing)
        .select {
            (Member.id eq id) and (Mailing.memberId eq Mailing.id)
        }
        .forEach {
            helperMap.compute(it[Member.id].value) { key, value ->
                // valueがnullではない場合、コピーしてmailingAddressを累計
                value?.copy(
                    mailingAddress = value.mailingAddress + it[Mailing.address]
                // valueがnullの場合はインスタンスを作る
                ) ?: MemberDto(
                    id = key,
                    name = it[Member.name],
                    mailingAddress = listOf(it[Mailing.address])
                )
            }
        }.let {
            // valueをListに変換
            helperMap.map { it.value }
        }
}
```

いかがでしょうか。これで重複するデータなく、インスタンスの作成も最低限に抑えることができたかと思います。もちろん、mailingAddressを追加するたびにコピーが発生するという問題はありますが、ここは専用のsetterなどを作っておくことで回避できると思います。

一つ注意しなくてはならないのは、ここで使っているMapをフィールドとして宣言したりするとデータの整合性やアプリケーションのメモリ使用量に響くということです。なので必ずメソッドの中でのみMapのインスタンスが作成されるようにする必要があります。

## 最後に

DSLを使ってクエリを直接作成する場合、JPAのようなORMの問題とされているN+1(必ず連関しているテーブルもjoinしてくる)のような問題を回避できますが、直接オブジェクトへのマッピングも書かなくてはならないという短所がありますね。個人的にクエリを書くのは楽しくないですが、クエリをコードとして管理でき、必要なクエリだけを書けるというところでDSLの方が良い点もあると思います。テーブルの構造や処理によってはORMが勝手にクエリもマッピングもしてくれるのが楽ではありますが。

ただ、ORMでどうやってデータを取得するかの問題だけでなく、ここで扱った「重複するデータをどう違う形のデータに加工するか」の問題は、必ずしもDBからレコードを取得する場合のみのことに限らないので（例えば他のAPIを読んだ結果としてもそんなデータはあり得ますね）、色々方法を考えておく必要はありそうです。今の時点ではMapを利用した方法がもっとも良さそうな気がしますが、他にもっと効率的な方法があるかもしれませんしね。

では、また！
