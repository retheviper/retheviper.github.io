---
title: "data classのListをCSVにする"
date: 2022-08-27
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - csv
---

アプリを書いていると、DBに保存した形とは違う形でデータを読み込んだり出力するケースが多いです。代表的に、多くのバックエンドアプリで採用しているREST APIがそうですね。APIの入力値と戻り値は、DBに保存されている形とは一致しないケースが多く、必要(リクエストを送信してきた側の都合)に合わせて適切にパラメータを受け取り、レスポンスを返すようになっています。そして時と場合によっては、人間が楽に読める形としてデータをまとめる必要もあります。そういった場合はExcelファイルやCSV、PDFといったいろいろなものを想定できますね。

今回のポストも場合も人が読める形のファイルとして、特にCSVを扱う場合にKotlin側の実装をどうやっていくかに関するものです。

## KotlinでCSVを扱う

KotlinのCSV読み込み/書き込みのライブラリとして[kotlin-csv](https://github.com/doyaaaaaken/kotlin-csv/)があり、JVMだけでなくKoltin/JSの場合でもこのライブラリを使って簡単にCSVを扱えます。更に[kotlin-grass](https://github.com/blackmo18/kotlin-grass)というライブラリもあり、`kotlin-csv`との組み合わせででCSVのデータを簡単にdata classのListとしてまとめることもできますね。読み込みの際に指定できるデータのフォーマットやカスタムマッピングオプションなどの機能も豊富にあり、かなり使いやすく良いライブラリとなっています。

しかし、実は`kotlin-csv`を使うときに問題が一つあります。先に述べた通りCSVにデータの出力そのものは可能なものとなっているのですが、読み込みの時にdata classへのマッピングには別のライブラリが必要であったように、data classのリストを書き込むには追加の処理が必要となります。これは、kotlin-csvの書き込み用のメソッドが以下のようになっているからです。

```kotlin
fun writeAll(rows: List<List<Any?>>, targetFile: File, append: Boolean = false) {
    open(targetFile, append) { writeRows(rows) }
}
```

ここで`rows`が書き込みで使うデータとなりますが、型が`List<List<Any?>>`になっているので、列のデータを一つの行としてListに定義し、それをさらにListに格納することでCSVのデータ全体を定義する必要があります。これはつまり、data classのリストを書き込むためには、フィールド一つ一つを列として定義し、それらをListとしてまとめる必要があるということです。また、CSVには一般的にヘッダが含まれますが、`List<List<Any?>>`の形だと最初の行にヘッダのみを定義した行は必要となることでもありますね。

一見複雑に見えますが、[reflection](https://kotlinlang.org/docs/reflection.html)を利用すると、data classのフィールド名とその値を得ることができますので、それを利用してdata classのListをこのメソッドに適した形に変えられます。これをヘッダを作る方法と、data classの値を行に変更する二つの段階で分けて説明していきます。

### data classからヘッダを作る

まずはヘッダを作ります。ヘッダを作るには、data classからフィールドを取得し、そのフィールドの名前のみを取得するといいでしょう。`id`というフィールドがあるとしたら、ヘッダもそのまま`id`になるということです。フィールド名とは別の名前をつけたい場合はアノテーションを活用する方法を考えられますが、まずはフィールド名をそのまま使う方法から述べたいと思います。

Kotlinのdata classから、フィールドを取得する方法が3つがあります。まずは、[KClass.members](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-class/members.html)があります。ただ、これだとメソッドを含め、全てのメンバーを持ってくることになります。以下のようにですね。

```kotlin
data class Data(val id: Int, val name: String)

val members = Data::class.members
println(members)
// [val Line_2.Data.id: kotlin.Int, val Line_2.Data.name: kotlin.String, fun Line_2.Data.component1(): kotlin.Int, fun Line_2.Data.component2(): kotlin.String, fun Line_2.Data.copy(kotlin.Int, kotlin.String): Line_2.Data, fun Line_2.Data.equals(kotlin.Any?): kotlin.Boolean, fun Line_2.Data.hashCode(): kotlin.Int, fun Line_2.Data.toString(): kotlin.String]
```

フィールド名が現れているのでこれも使える方法の一つではありますが、やはりdata classだと基本的に`equals()`, `hashCode()`, `copy()`, `toString()`, `componentN()`のようなメソッドが
できてしまうので、これらをフィルタする必要があります。例えば、以下のようにですね。

```kotlin
val memberProperties = Data::class.members.filterNot { it.name.contains("component") || it.name == "copy" || it.name == "equals" || it.name == "hashCode" || it.name == "toString" }

println(memberProperties)
// [val Line_2.Data.id: kotlin.Int, val Line_2.Data.name: kotlin.String]
```

しかし、フィルタをしなくてももっと簡単にフィールドのみを抽出できる方法もあります。[memberProperties](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect.full/member-properties.html)を使うことです。

```kotlin
val memberProperties = Data::class.memberProperties

println(memberProperties)
// [val Line_2.Data.id: kotlin.Int, val Line_2.Data.name: kotlin.String]
```

ただ、この方法にも問題はあります。取得したフィールドの順番が、data classに定義した通りにならなく、アルファベット順になるということです。以下の例を見ると、nameとageの順で定義したフィールドが、ageとnameの順になっているのがわかります。

```kotlin
data class Person(val name: String, val age: Int)

val memberProperties = Person::class.memberProperties
println(memberProperties) // name, age順にならない
// [val Line_11.Person.age: kotlin.Int, val Line_11.Person.name: kotlin.String]
```

ここでどうしてもフィールドを定義した順に取得したい場合、data classのコンストラクタを使う方法があります。まずはコンストラクタを使った方法ですが、これはdata classに、自動的にコンストラクタがフィールドを定義した順番通りのパラメータを持つように生成されるということを利用した方法です。以下のようになります。

```kotlin
val parameters = Person::class.primaryConstructor!!.parameters.mapNotNull { it.name }

println(parameters) // [name, age]
```

多少強引な方法ではありますが、これでフィールドとして使うフィールド名は取得できました。では、次にアノテーションを使う場合を見ていきましょう。

#### アノテーションを使う場合

フィールド名をそのままヘッダとして利用したくない場合は、アノテーションを活用できます。フィールドとしてStringを持つアノテーションを定義して、ヘッダを読み取るときにそのフィールドにつけたアノテーションを取得するという方法です。例えば、以下のようなアノテーションを定義したとしましょう。

```kotlin
@Target(AnnotationTarget.PROPERTY)
annotation class CsvHeaderName(val value: String)
```

アノテーションは、以下のようにdata classで使います。

```kotlin
data class Person(
    @CsvHeaderName("名前")
    val name: String,
    @CsvHeaderName("年齢")
    val age: Int
)
```

そしてこのdata classのアノテーションを取得していきます。`memberProperties`でフィールドを取得した場合、そのフィールドからアノテーションの一覧(アノテーションは複数存在できるので)を取得し、そこから先に定義した`CsvHeaderName`のみをフィルタします。あとはアノテーションがあるかどうかをみて、どちらの値を使うかを決めればいいですね。以下のコードが、そのサンプルです。

```kotlin
val datas = listOf(Person("John", 20))

val headers = datas.first()!!::class.memberProperties.map { property ->
            val name = property.annotations.filterIsInstance<CsvHeaderName>().firstOrNull() // アノテーションはないケースもある
            name?.value ?: property.name // アノテーションがnullの場合は、フィールド名を使う
        }

println(headers) // [年齢, 名前]
```

`primaryConstructor`のパラメータで取得した場合でも、やり方は大きく変わりません。この場合は、コンストラクタのパラメータを基準にループしながら一致するフィールドを探すという処理が追加されるだけです。例えば以下のようにです。

```kotlin
val fieldNames = seeds.first()::class.primaryConstructor!!.parameters.mapNotNull { it.name }

val headers = fieldNames.mapNotNull { name ->
    // パラメータと一致するフィールドを対象に処理を行う
    datas.first()::class.memberProperties.find { it.name == name }?.let { property ->
        val headerName = property.annotations.filterIsInstance<CsvHeaderName>().firstOrNull()
        headerName?.value ?: property.name
    }
}
```

これで、ヘッダのデータはできました。次は、このヘッダに合わせて下に出力する行としてdata classをListに変換するのみですね。

### data classをListに変える

ヘッダの処理で既にやっていたように、data classをListに変換する場合でも処理は大きく変わりません。違う点は、フィールドから実際のデータを取得するだけですね。ここでは、コンストラクタからパラメータを取得した場合を想定してコードを書きましょう。

Kotlinのreflectionでフィールドの値を取得する場合はJavaと変わらなくて、取得したフィールドにdata classのインスタンスを渡すだけとなります。ただ、フィールドがnullの場合は考慮する必要があります。nullになってしまうと、その列自体がスキップされ、最終的に出力されたCSVのデータで列がずれる場合があるからです。なので、空白のStringを指定するなどで、行ごとの長さ(Listのサイズ)を合わせる必要があります。以下のようにですね。

```kotlin
datas.map { d ->
    fieldNames.mapNotNull { name ->
        d::class.memberProperties.find { it.name == name }?.let { field ->
            field.call(d) ?: "" // フィールドの値を取得し、nullのばいは空白にする
            } 
        }
    }
```

ただ、ここで時間や日付を扱う場合、フォーマッタを利用したいケースがあるかと思います。例えば、アプリの中では`LocalTime`として扱っているが、CSVとしては`HH:mm`のような形で出力したい場合や、`LocalDate`を`yy/MM/dd`にしたい場合などですね。ここでフォーマット自体は、[DateTimeFormatter](https://docs.oracle.com/javase/jp/8/docs/api/java/time/format/DateTimeFormatter.html)を使うだけですが、問題は取得したフィールドがどの型であるかの判定です。

Kotlinのreflectionで取得したフィールドは、[KProperty1](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-property1/)という型になっています。ここでどうやって元の型を取得するかが問題ですね。このクラスは[KCallable](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-callable/)というインタフェースを実装していて、ここには[returnType](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-callable/return-type.html)というプロパティがあります。これで[KType](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-type/)というインタフェースが取得できるようになるので、これを持って判定をおこ泣くことになります。

しかし、名前から分かるように、`KType`はKotlinの型に関するインタフェースとなっています。比較したい`LocalDate`や`LocalTime`などのクラスはJavaのものなので、直接的な比較ができないですね。幸い、JavaのクラスでもKotlinで参照できる`Ktype`として変換することはできます。以下のようにです。

```kotlin
val localDateKType: KType = LocalDate::class.createType()
```

なので、あとは分岐によって適切なフォーマッタを使うだけですね。以下のようにです。

```kotlin
datas.map { d ->
        fieldNames.mapNotNull { name ->
            d::class.memberProperties.find { it.name == name }?.let { field ->
                field.call(d)?.let {
                    when (field.returnType) { // タイプによる分岐
                        LocalDate::class.createType() -> dateFormatter.format(it as LocalDate)
                        LocalTime::class.createType() -> timeFormatter.format(it as LocalTime)
                        LocalDateTime::class.createType() -> dateTimeFormatter.format(it as LocalDateTime)
                        else -> it
                    }
                } ?: ""
            }
        }
    }
```

ただ、ここでもう一つ注意しなければならないのは、nullableなフィールドのKTypeは別のものになるということです。つまり、上記のコードでは、以下のようなdata classのフィールドは分岐処理でフォーマッタが働かなくなるということです。

```kotlin
// どのフィールドもフォーマットされない
data class DateTimes(
    val date: LocalDate?,
    val time: LocalTime?,
    val dateTime: LocalDateTime?
)
```

この場合は、`KType`を作るときにnullableであることを指定することで簡単に解決できます。あとは分岐で、両方チェックするようにするだけです。以下のようにですね。

```kotlin
datas.map { d ->
        fieldNames.mapNotNull { name ->
            d::class.memberProperties.find { it.name == name }?.let { field ->
                field.call(d)?.let {
                    when (field.returnType) { // nullableでもnullableではない場合でも分岐でフォーマットする
                        LocalDate::class.createType(), LocalDate::class.createType(nullable = true) -> dateFormatter.format(it as LocalDate)
                        LocalTime::class.createType(), LocalTime::class.createType(nullable = true) -> timeFormatter.format(it as LocalTime)
                        LocalDateTime::class.createType(), LocalDateTime::class.createType(nullable = true) -> dateTimeFormatter.format(it as LocalDateTime)
                        else -> it
                    }
                } ?: ""
            }
        }
    }
```

あとは、ヘッダと、data classから取得した値のリストを結合して`kotlin-csv`の`writeAll()`に渡すだけですね。一つ、値は上記のコードで既に`List<List<Any>>`の形となっていますが、ヘッダは`List<String>`なので、ヘッダは更にListに入れる必要があります。

```kotlin
// ヘッダ
val header = // 省略
// 実際のデータ
val rows = datas.map { /** 省略 */ }

csvWriter().writeAll(
            rows = listOf(headers) + rows,
            targetFile = targetFile
        )
```

これでヘッダが先に行に書き込まれ、次の行からはdata classのフィールドに格納した実際の値が出力されることになりました。

## 最後に

この度軽く「KotlinなんだからKotlin制のライブラリを使おう」と、軽い気持ちで採用したライブラリが想定していたものと違ったので困っていましたが、幸いJavaを使っていた時に[Apache POI](https://poi.apache.org/)を使って似たような機能をするライブラリを作ってみた経験があったのでその知識を活かせたと言えます。当時はまだ駆け出しのエンジニア(今もそうと思っていますが)だったので大変苦労した思い出でもありますが、今はその経験があってこそ対処できたようなものなので大変ありがたい経験だったなと思いました。

上記のコードに対してはちょっとしたライブラリを作ってみたので、またどこかで活用してみたいものですね。色々と改善して、のちにMaven Repositoryのようなところでも公開できるようになったらなと思います。

では、また！
