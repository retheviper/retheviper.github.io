---
title: "Kotlinで書いてみた〜その二〜"
date: 2021-06-28
categories: 
  - kotlin
photos:
  - /assets/images/sideimage/kotlin_logo.jpg
tags:
  - kotlin
---

[前回](../../../../2021/03/28/kotlin-code-in-my-style-one)に続いて、今回も簡単にKotlinで色々書いてみましたのでその紹介となります。Kotlinではスタンダードライブラリや言語仕様として提供している機能がかなり多いので、これらを使いこなすだけでも生産性やコードのクォリティが大幅に上がるのではないかと思います。なので、今回もJava的な書き方を、Kotlinではどんな方法で効率よく実現できるかを中心に紹介したいと思います。

もちろんKotlinでは基本的にJavaの書き方でも全く問題なく動くコードを書けますが、Kotlinならではのコードに変えた方がより簡単で短いコードを書ける場合が多く、色々と手間を省けることができるので（そして大抵の場合、スタンダードライブラリの実装の方が自分の書いたコードよりクォリティ高いような…）こういう工夫はする価値が十分にあるのではないかと思います。

なので、今回は自分が調べたKotlinの小技を少し紹介したいと思います。

## Sequentialなデータを作成する

よくユニットテストなどでテスト用データを作成して使う場合がありますね。こういう時に必要となるデータの種類は色々とあるかと思いますが、複数のレコードを番号をつけて順番に揃えた感じのものを作りたい場合もあると思います。例えばData01、Data02、Data03…といったデータを作りたい場合ですね。

この場合は、ループでデータを作り、Listにまとめるというのが一般的ではないかと思います。例えば以下のような例があるとしましょう。

```kotlin
// テスト用データを作成する
fun createTestDatas(): List<String> {
    // テスト用データのリスト
    val testDatas = mutableListOf<String>()
    
    // 10件のデータを追加
    for (i in 0 until 10) {
        testDatas.add("テスト$i")
    }

    // read-onlyに変換して返却
    return testDatas.toList()
}
```

ただ、どちらかというとこれはJavaのやり方に近いので、まずはこれをベースに、Kotlinらしきコードではどうやって同じことができるかを考えてみたいと思います。

### repeat

まず考えられる方法は、ループの単純化ですね。サイズが10のリストを作りたいということは、ループが10回であることなので、それに相応しい関数を使います。例えば[repeat](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/repeat.html)がありますね。`repeat`を使うと、スコープ内のパラメータとしてインデックスが渡されるので、簡単に

```kotlin
fun createTestDatas(): List<String> {
    val testDatas = mutableListOf<String>()

    // 10回繰り返す
    repeat(10) { 
        testDatas.add("テスト$i")
    }

    return testDatas.toList()
}
```

次に考えたいのは、`MutableList`を`Immutable`に変えることです。テストで使うデータとしては問題ない場合はありますが、変更する必要のないデータをそのまま`Mutable`にしておくのはあまり良い選択ではありませんね。なので、データの作成を最初から`List`にできる方法を取りたいものです。

ここでは二つの道があって、最初からサイズを指定した`List`を宣言するか、ループの範囲、つまり[Range](https://kotlinlang.org/docs/ranges.html#range)を指定する方法があります。

### List

まずはサイズを指定した`List`を作る方法からみていきましょう。インスタンスの作成時に、サイズと要素に対してのイニシャライザを引数として渡すことで簡単に指定したサイズ分の要素を作ることができます。例えば、上で紹介したコードは`List`を使うことで以下のように変えることができます。

```kotlin
fun createTestDatasByList(): List<String> =
    List(10) { "テスト$it" }
```

この方法は、実は先に紹介した方法と根本的に違うものではありません。実装としては、以下のようになっているので、Syntax sugarとして使えるということがわかります。

```kotlin
@SinceKotlin("1.1")
@kotlin.internal.InlineOnly
public inline fun <T> List(size: Int, init: (index: Int) -> T): List<T> = MutableList(size, init)

@SinceKotlin("1.1")
@kotlin.internal.InlineOnly
public inline fun <T> MutableList(size: Int, init: (index: Int) -> T): MutableList<T> {
    val list = ArrayList<T>(size)
    repeat(size) { index -> list.add(init(index)) }
    return list
}
```

他にも`List`を使う場合は、`itnit`としてどんな関数を渡すかによって、`step`の設定などができるのも便利ですね。例えば以下のようなことができます。

```kotlin
List(5) { "Test${ it * 2 }" }
// [Test0, Test2, Test4, Test6, Test8]

List(5) { (it * 2).let { index -> "$index は偶数" } }
// [0 は偶数, 2 は偶数, 4 は偶数, 6 は偶数, 8 は偶数]
```

ただ、結果的に作られる`List`のインスタンスは`MutableList`なので、生成したデータをread-onlyにしたい場合はまたこれを`toList()`などで変換する必要があるという問題があります。

### Range

では、もう一つの方法をまた試してみましょう。Kotlinでは数字の範囲を指定することだけで簡単に`Range`オブジェクトを作成することができます。`Range`を使う場合、上記のコードは以下のように変えられます。

```kotlin
// Rangeを使ってテストデータを作る
fun createTestDatasByRange(): List<String> =
    (0..10).map { "テスト%it" }
```

`List`の時とは違って、`Range`には[IntRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-int-range)や[LongRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-long-range)、[CharRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-char-range)などがあり、引数の数字や文字を調整することで簡単にアレンジができるということも良いです。

また、一般的に性能は`List`より`Range`の方が良いようです。以下のようなコードでベンチマークした際、大抵`Range`の方が`List`の倍ぐらい早いのを確認できました。

```kotlin
import kotlin.system.measureTimeMillis

data class Person(val name: String, val Num: Int)

fun main() {
    benchmark { list() }
    benchmark { range() }
}

fun benchmark(function: () -> Unit) =
    println(measureTimeMillis { function() })
    
fun list() =
    List(200000) { Person("person$it", it) }
    
fun range(): List<Person> =
    (0..200000).map { Person("person$it", it) }
```

一つ気にしなくてはならないのは、`Range`の場合は基本的に値が1づつ増加することになっているので、`for`や`List`のような`step`の条件が使えません。なので場合によってどちらを使うかは考える必要があります。

## Check

Validationなどで、パラメータの値を確認しなければならない場合があります。Kotlinでは`Nullable`オブジェクトとそうでないオブジェクトが分けられているので、Javaと違って引数に`null`が渡される場合はコンパイルエラーとなりますが、ビジネスロジックによってはそれ以外のことをチェックする必要もあり、自前のチェックをコードで書くしかないです。

まず、お馴染みのJavaのやり方を踏襲してみると、以下のようなコードを書くことができるでしょう。関数の引数と、その戻り値のチェックが含まれている例です。

```kotlin
fun doSomething(parameter: String): String {
    if (parameter.isBlank()) {
        throw IllegalArgumentException("文字列が空です")
    }

    val result = someRepository.find(parameter)

    if (result == null) {
        throw IllegalStateException("結果がnullです")
    }
    return result
}
```

ここで少し違う言語の例をみていきたいと思います。Kotlinとよく似ていると言われているSwiftの場合、ここで[Guard Statement](https://docs.swift.org/swift-book/ReferenceManual/Statements.html#grammar_if-statement)を使うのが一般的のようです。チェックのための表現が存在することで、ビジネスロジックとチェックが分離されるのが良いですね。Swiftをあまり触ったことがないので良い例にはなっていないかも知れませんが、イメージ的には以下のようなコードになります。

```swift
func doSomething(parameter: String) throws -> String {
    guard !parameter.isEmpty else {
        throw ValidationError.invalidArgument
    }

    guard let result = someRepository.find(parameter) else {
        throw ValidationError.notFound
    }
    return result
}
```

同じく、Kotlinでもチェックのための表現とビジネスロジックが分離できれば、コードの意味がより明確になるはずです。Kotlinではどうやってそれを実現できるのでしょうか。例えば以下のようなことを考えられます。

```kotlin
fun doSomething(parameter: String?): String {
    val checkedParameter = requireNotNull(parameter) {
        "文字列がnullです"
    }

    val result = someRepository.find(checkedParameter)

    return checkNotNull(result) {
        "結果がnullです"
    }
}
```

[requireNotNull](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/require-not-null.html)は、渡された引数が`null`である場合は`IllegalArgumentException`を投げ、そうでない場合は引数を`non-null`タイプとして返します、明確に`null`チェックをしていることが解るだけでなく、以降チェックがいらないので便利です。また、`lazy message`として`IllegalArgumentException`が発生した時のメッセージを指定できるのも良いですね。

[checkNotNull](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/check-not-null.html)の場合も機能的には`requireNotNull`と変わらないですが、`null`の場合に投げる例外が`IllegalStateException`となります。なので、用途に合わせてこの二つを分けて使えますね。

他に使えるものとしては[require](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/require.html)があります。こちらは条件式を渡すことで、`null`チェック以外のこともできます。なので、以下のコードのように、`Int`型のデータに対して範囲をチェックするということもできるようになります。

```kotlin
fun doSomething(parameter: Int) {
    require(parameter > 100) {
        "$parameterは大きすぎます"
    }

    // ...
}
```

他にも、[Elvis operator](https://kotlinlang.org/docs/null-safety.html#elvis-operator)を使う方法もありますね。この場合は、`null`の場合にただ例外を投げるだけでなく、代替となる処理を書くことができますので色々と活用できる余地があります。例えば以下のようなことができますね。

```kotlin
fun doSomething(parameter: String?): String {
    val checkedParameter = parameter ?: "default"

    val result = someRepository.find(checkedParameter)

    return result ?: throw CustomException("結果がnullです")
}
```

## Listの分割

とある条件と一致するデータをListから抽出したい場合は、`filter`のようなoperationを使うことでできます。しかし、条件が二つだとどうすればいいでしょうか。正確には、一つのリストに対して、指定した条件に一致する要素とそうでない要素の二つのリストに分離したい場合です。

こういう場合はとりあえず下記のように2回ループさせる方法があると思いますが、これはあまり効率がよくないです。

```kotlin
val origin = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// 奇数を抽出
val odd = origin.filter { it % 2 != 0 }
// 偶数を抽出
val even = origin.filter { it % 2 == 0 }
```

ループを減らすためには、あらかじめ宣言したリストに対してループの中で分岐処理を行うという方法があるでしょう。例えば以下のようにです。

```kotlin
val origin = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// 奇数と偶数のリストを宣言しておく
val odd = mutableListOf<Int>()
val even = mutableListOf<Int>()

// ループ処理
origin.forEach {
    if (it % 2 != 0) {
        odd.add(it) // 奇数のリストに追加
    } else {
        even.add(it) // 偶数のリストに追加
    }
}
```

幸い、この状況にぴったりな方法をKotlinのスタンダードライブラリが提供しています。[partition](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/partition.html)というoperationです。このopreationを使うと、元のリストの要素を条件に一致するものとそうでないもので分割してくれます。

このopretionの戻り値`Pair<List<T>, List<T>>`なので、[destructuring-declaration](https://kotlinlang.org/docs/destructuring-declarations.html)と組み合わせることでかなり短いコードになります。実際のコードは以下のようになるりますが、はかなりスマートですね。

```kotlin
val origin = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
val (odd, even) = origin.partition { it % 2 != 0 } // 条件に一致するものと一致しないものでリストを分離
```

## 最後に

Kotlinは便利ではありますが、言語自体が提供する便利さ（機能）が多いゆえに、APIの使い方を正しく活用できるかどうかでコードのクォリティが左右される部分が他の言語と比べ多いような気がしています。さらにバージョンアップも早く、次々と機能が追加されるのでキャッチアップも大事ですね。

でも確かに一つづつKotlinでできることを工夫するうちに、色々とできることが増えていく気もしていますね。研究すればするほど力になる言語を使うということは嬉しいことです。ということで、これからもKotlinで書いてみたシリーズは続きます。

では、また！
