---
title: "Kotlinで書いてみた〜その四〜"
date: 2023-09-03
categories: 
  - kotlin
image: "../../images/kotlin.jpg"
tags:
  - kotlin
---

しばらくKotlinで書いてみたシリーズを書いていなかったので、今回はその四です。

## ListをListでソート

データ構造として、ListをListでソートしたい場合がありますね。例えばとあるテーブルに対して、別のテーブルのIDを持たせている場合などです。このテーブルのIDを配列にして、並び順を表現しているとしたら、この配列を元にソートしたいことになります。

もちろんクエリでソートすることもできますが、クエリでソートすると、ソートした結果をキャッシュできないというデメリットがあります。そこで、ListをListでソートする方法を考えてみました。

まずは以下のようなデータがあるとします。イメージとしては、複数の画像を持つImageBoxというデータがあり、そのImageBoxのimageOrderにはImageのIDが配列として入っているとします。

```kotlin
data class ImageBox(
    val id: Int,
    val title: String,
    val imageOrder: List<Int> // imageのIDの配列
)

data class Image(
    val id: Int,
    val url: String
)
```

ここでDBとしてはImageBoxテーブルとImageテーブルがあり、ImageBoxテーブルのimageOrderにはImageテーブルのIDが配列として入っているとします。そしてアプリではImageBoxのimageOrderの順番にImageを表示したいとします。

```kotlin
val imageBox = ImageBox(
    id = 1,
    title = "title",
    imageOrder = listOf(3, 1, 2, 5, 4)
)

val images = listOf(
    Image(id = 1, url = "url1"),
    Image(id = 2, url = "url2"),
    Image(id = 3, url = "url3"),
    Image(id = 4, url = "url4"),
    Image(id = 5, url = "url5")
)
```

この場合、以下のようにソートすることができます。

```kotlin
// imageBoxのimageOrderのをIndexをkeyとするMapに変換
val indexMap = imageBox.imageOrder.mapIndexed { index, value -> value to index }.toMap()
// imagesをimageOrderを元にソート
val sortedImages = images.sortedBy { indexMap[it.id] }
```

原理は簡単で、imageOrderの配列をIndexとvalueのMapに変換して、その値を元にソートしているだけです。この方法であれば、クエリでソートすることなく、imageOrderの順番にソートすることができます。

また、こういったソートは以下のように拡張関数として定義すると、他の場所でも使い回すことができます。anotherには並び順を表現する配列を渡して、keySelectorにはソートしたい配列の要素から並び順の基準となる値を取り出す関数を渡します。

```kotlin
fun <T, U> List<T>.sortedBy(another: List<U>, keySelector: (T) -> U): List<T> {
    val indexMap = another.mapIndexed { index, value -> value to index }.toMap()
    return this.sortedBy { indexMap[keySelector(it)] }
}
```

## Listのページング

ページングはよくDBのクエリで行いますが、クエリでなくコードのみで処理したい場合もありますね。例えば、外部APIに対してリクエストを送る場合などです。この場合、ページングをどう実装するかが課題になります。

この場合は以下のように拡張関数を定義することで、Listのページングを実装することができます。

```kotlin
fun <T> List<T>.paginated(offset: Int, limit: Int): List<T> {
    // ページングが必要ない場合はそのまま返す
    if (offest == 0 && limit < size) {
        return this
    }
    // ページングが必要な場合はoffsetとlimitを元にsubListを取得する
    val fromIndex = (offset - 1) * limit
    return if (size <= fromIndex) {
        // ページングの範囲外の場合は空のリストを返す
        emptyList()
    } else {
        // ページングの範囲内の場合はsubListを返す
        val toIndex = size.coerceAtMost(fromIndex + limit)
        subList(fromIndex, toIndex)
    }
}
```

細かいものとしては[coerceAtMost()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/coerce-at-most.html)を使って、sizeより大きい値を渡された場合はsizeを返すようにしています。これはもちろんIndexOutOfBoundsExceptionを防ぐためです。

## Exposed

厳密にいって[Exposed](https://github.com/JetBrains/Exposed)はORMなのですが、Kotlinの書き方によってさらに便利になる部分もあるので今回はこちらも紹介したいと思います。

### InsertとUpdateの共通化

ExposedでInsertやUpdateを書くのはかなり直感的なのですが、多くの場合にInsertとUpdateで同じ処理を書くことになります。そこで、InsertとUpdateの共通化を考えてみました。例えば以下のようなコードがあるとします。

```kotlin
fun insert(image: Image): Image {
    ImageTable.insert {
        this[id] = image.id
        this[url] = image.url
        this[title] = image.title
        this[description] = image.description
    }
    return image
}

fun update(image: Image): Image {
    ImageTable.update({ ImageTable.id eq image.id }) {
        this[url] = image.url
        this[title] = image.title
        this[description] = image.description
    }
    return image
}
```

この場合、InsertとUpdateではID以外だとurlやtitleのような値をセットするという面では同じ処理を書いているだけですね。テーブルが増えれば増えるほど、このようなコードも増えていきます。幸いExposedではUpdate時の[UpdateBuilder](https://github.com/JetBrains/Exposed/blob/main/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/statements/UpdateBuilder.kt)をInsert時の[InsertStatement](https://github.com/JetBrains/Exposed/blob/main/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/statements/InsertStatement.kt)が継承している形の実装となっているので、以下のように共通化することができます。

```kotlin
private fun UpdateBuilder<Int>.setParametersFrom(image: Image) {
    this[url] = image.url
    this[title] = image.title
    this[description] = image.description
}
```

このような関数を定義すると、InsertとUpdateの処理は以下のように書くことができます。

```kotlin
fun insert(image: Image): Image {
    ImageTable.insert {
        this[id] = image.id
        it.setParametersFrom(image)
    }
    return image
}

fun update(image: Image): Image {
    ImageTable.update({ ImageTable.id eq image.id }) {
        it.setParametersFrom(image)
    }
    return image
}
```

これはbatchInsertの場合でも同じです。[BatchInsertStatement](https://github.com/JetBrains/Exposed/blob/main/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/statements/BatchInsertStatement.kt)もまたInsertStatementを継承しているので、以下のように共通化することができます。これでだいぶすっきりしたコードが書けるようになります。

```kotlin
fun batchInsert(images: List<Image>): List<Image> {
    ImageTable.batchInsert(images) {
        this[id] = it.id
        this.setParametersFrom(it)
    }
    return images
}
```

### BatchUpsertの実装

ExposedにbatchInsertはありますが、batchUpsertはありません。ただ、クエリの書き方とアプリ側の実装によってbatchUpsertを実現することはできます。もちろん[ExposedPowerUtils](https://github.com/PerfectDreams/ExposedPowerUtils)のようなライブラリもあるので、それを導入するのも良いでしょう。

ただ、あのライブラリを使う場合は少し問題がありました。まずこのライブラリの実装では特定のカラムを指定して、それを基準にデータがすでに存在する場合はUpdate、存在しない場合はInsertという処理を行っています。最終的に作られるコードは、Insertをまず実行して、指定したカラムが重複している場合は`ON CONFLICT`でUpdateを実行するというものです。

ここでテーブルのIDはアプリ側で生成するUUIDであり、ON CONFLICTの基準となるカラムがFKの場合にはIDまでUPDATEされるという問題がありました。なので、先ほどのライブラリの実装を元に、PKを除いてUpdateするように実装を変更しました。それが以下です。（あと一部使ってない引数は省略してます）

```kotlin
fun <T : Table, E> T.batchUpsert(
    data: Collection<E>,
    vararg keys: Column<*> = (primaryKey ?: throw IllegalArgumentException("primary key is missing")).columns,
    body: T.(BatchInsertStatement, E) -> Unit
): BatchInsertOrUpdate {
    return BatchInsertOrUpdate(table = this, keys = keys).apply {
        data.forEach {
            addBatch()
            body(this, it)
        }
        execute(TransactionManager.current())
    }
}

class BatchInsertOrUpdate(
    table: Table,
    private vararg val keys: Column<*>,
) : BatchInsertStatement(table, false, false) {
    override fun prepareSQL(transaction: Transaction): String {
        val transactionManager = TransactionManager.current()
        val primaryKey = table.primaryKey?.columns?.toSet() // ここでprimaryKeyを取得
        val updateSetter = ((table.columns - keys.toSet())
            .let { columns -> primaryKey?.let { columns - primaryKey } ?: columns }) // ここでprimaryKeyを除外
            .joinToString { "${transactionManager.identity(it)} = EXCLUDED.${transactionManager.identity(it)}" }
        val onConflict =
            "ON CONFLICT (${keys.joinToString { transactionManager.identity(it) }}) DO UPDATE SET $updateSetter"
        return "${super.prepareSQL(transaction)} $onConflict"
    }
}
```

### トランザクションを分ける

この度はリードレプリカを導入することになり、GET系のAPIにはとりあえずリードレプリカを使うことになりました。ただ、リードレプリカは更新系のクエリを受け付けないので、更新系のクエリはプライマリに送る必要があります。

そして、APIの実装としては例外を投げる前に明示的にロールバックを行うため別途の関数を定義していて、これと同じような使い方としてリードレプリカに接続するものを作りたいと思いました。まず、既存で使っていた関数は以下のようなものです。

```kotlin
fun <T> transactionWrapper(statement: Transaction.() -> T): T {
    return transaction {
        try {
            statement()
        } catch (ex: Exception) {
            TransactionManager.current().rollback()
            throw ex
        }
    }
}
```

Exposedでは[transaction](https://github.com/JetBrains/Exposed/blob/7c8d8a20e2efef5d0282ca6b703a433ecbf4a607/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/transactions/ThreadLocalTransactionManager.kt#L156)の引数に[Database](https://github.com/JetBrains/Exposed/blob/7c8d8a20e2efef5d0282ca6b703a433ecbf4a607/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/Database.kt)を渡すことで、簡単にどのDBに接続するかを指定することができます。なので、これを使ってリードレプリカに接続する関数を以下のように定義することができます。

```kotlin
fun <T> readReplicaTransactionWrapper(statement: Transaction.() -> T): T {
    val readReplicaDatabase: Database = getKoinModule(named("readReplicaDatabase")))
    return transaction(readReplicaDatabase) { statement() }
}
```

ここでの`Database`を[Koin](https://github.com/InsertKoinIO/koin)を使用してinjectしているので、追加で以下のような設定を加えてます。

```kotlin
// デフォルトのDBはプライマリに固定
val defaultDatabase: Database = getKoinModule(named("primaryDatabase"))
TransactionManager.defaultDatabase = defaultDatabase

// Top level functionでKoinのinjectを使うための設定
fun <T> getKoinModule(qualifier: Qualifier? = null): T {
    return object : KoinComponent {
        val value: T by inject(qualifier)
    }.value
}
```

これでtransactionを使うたびに関数名だけを変えるだけで、プライマリとリードレプリカに接続することができます。

## 最後に

いかがでしたでしょうか。今回もKotlinの小技的なものを紹介してみました。最近はあまり複雑なロジックを書いてない上、プライベートでの開発もあまりしてないので、なかなかブログのネタが思いつかないですね。たまにこちらのブログの記事を参考にしていただいている方がいるようなので、もう少し頻繁に更新できるように頑張ります。

しばらくブログを更新してませんでしたが、今回はCompose Multiplatformも1.5.0になり、Java 21のリリースもまもなくなので、近いうちにまたそれらについての記事も更新していきたいと思います。

では、また！
