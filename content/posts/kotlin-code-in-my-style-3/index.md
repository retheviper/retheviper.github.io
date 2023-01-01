---
title: "Kotlinで書いてみた〜その三〜"
date: 2021-09-18
categories: 
  - kotlin
image: "../../images/kotlin.jpg"
tags:
  - kotlin
  - java
---

JavaからKotlinに移行したものの立場から言うと、Kotlinはスタンダードライブラリだけでも色々な関数を提供しているので、Javaに比べてかなり生産性が上がるといえますが、逆にいまいち関数の有効な使い方がわからなかったり、どうやって処理を書いたら「Kotlinらしい」かわからない場合もあるかと思います。なのでもう3回目のポストになりますが、今回もKotlinで色々とコードを書いてみて、そのうち良さそうなものをいくつか共有します。

## Listの要素をスワップ

Listの要素の順番を変える方法はソートなどを含め色々とありますが、二つの要素をスワップ（インデックスを交換）したい場合もあるかと思います。こういう時に活用できる拡張関数を考えてみました。

### インデックスがわかる場合

スワップしたい要素のインデックスがわかる場合は、そのインデックスを交換すればいいだけですね。ここでインデックスの交換は、二つの変数の値をスワップすることと変わらないです。変数の値を交換するのは伝統的には以下のような方法がありますね。

```kotlin
var a = 10
var b = 20
var c = a

a = b
b = c
```

もう少しKotlinらしい方法では、`also`を用いたものがあります。その方法だと、必要な処理は以下のようにもっとシンプルになります。

```kotlin
var a = 10
var b = 20

a = b.also { b = a }
```

これと同じく、Listの要素をスワップする処理を拡張関数で書くとしたらと以下のようになります。

```kotlin
fun <T> List<T>.swapByIndex(indexFrom: Int, indexTo: Int): List<T> =
    toMutableList().apply {
        this[indexFrom] = this[indexTo].also { this[indexTo] = this[indexFrom] }
    }.toList()
```

### インデックスがわからない場合

スワップしたい要素のインデックスがわからない場合もありますが、これも結局インデックスを持って値をスワップすることになるので、まずインデックスを抽出する処理だけを足せば良いかなと思います。

インデックスを取得する方法は、要素を渡して取得する[indexOf](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/index-of.html)とPredicateを渡して取得する[indexOfFirst](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/index-of-first.html)があるので、これらを活用することにします。あとはこれらの方法で取得したインデックスを、先に実装しておいた拡張関数に渡すだけで良いです。例えば以下のような実装ができます。

```kotlin
// indexOf(element)を使うケース
fun <T> List<T>.swapByElement(from: T, to: T): List<T> =
    swapByIndex(indexOf(from), indexOf(to))

// indexOfFirst(predicate)を使うケース
fun <T> List<T>.swapByCondition(from: (T) -> Boolean, to: (T) -> Boolean): List<T> =
    swapByIndex(indexOfFirst { from(it) }, indexOfFirst { to(it) })
```

## 時間を数字に

`java.time`パッケージの`LocalDate`や`LocalDateTime`のようなオブジェクトは、コード上で時間を扱うには便利ですが、ファイルに書き込むなどでフォーマットを変更する必要がある時もあります。つまり、`yyyy-MM-dd`ではなく`yyyyMMddhhmmss`のような形にしたい場合があるということです。こういうときは、簡単にInt型に変更できる拡張関数を書いておくと便利でしょう。例えば以下のようなものを考えられます。

```kotlin
fun LocalDate.toInt(): Int = "$year$monthValue$dayOfMonth".toInt()

val date = LocalDate.of(2021, 12, 31) // 2021-12-31
println(date.toInt()) // 20211231
```

ただ、こうする場合、以下のように月や日付が一桁のものになってしまうケースもあります。

```kotlin
val date = LocalDate.of(2021, 9, 1) // 2021-09-01
println(date.toInt()) // 202191
```

この問題を解決するには、まず月や日付を二桁の文字列に変える必要がありますね。例えば以下のようなことができます。

```kotlin
fun LocalDate.toInt(): Int = 
    "$year${monthValue.toString().padStart(2, '0')}${dayOfMonth.toString().padStart(2, '0')}".toInt()

val date = LocalDate.of(2021, 9, 1) // 2021-09-01
println(date.toInt()) // 20210901
```

しかし、これでも完璧とはいえません。`LocalDate`のみでなく`LocalDate`や`LocalDateTime`, `YearMonth`など、`java.time`パッケージに属する他のオブジェクトも使いたい場合には、全てのオブジェクトに対して同じような拡張関数を書く必要があるからです。

幸い、`LocalDate`、`LocalDateTime`、`YearMonth`は共通的に[Temporal](https://docs.oracle.com/javase/jp/8/docs/api/java/time/temporal/Temporal.html)というインタフェースを継承しているので、`Temporal`に拡張関数を追加することで問題は解決できます。

そしてこれらの実装クラスで扱っている時間の範囲はオブジェクトごとに違うので、実装も変える必要がありますね。これらのオブジェクトはどれも時間を数字として表現しているので、まず`toString`で文字列に変換した後、数字だけを抽出することです。`String`は`CharSequence`を継承しているので、`filter`で数字だけを抽出すると良いでしょう。そうすると、以下のような方法が使えます。

```kotlin
fun Temporal.toDigit(): Long = toString().filter { it.isDigit() }.toLong()

val yearMonth = YearMonth.of(2021, 8) // 2021-08
println(yearMonth.toDigit()) // 202108
val dateTime = LocalDateTime.of(2021, 10, 2, 10, 10, 10) // 2021-10-02T10:10:10
println(dateTime.toDigit()) // 20211002101010
```

Stringのフォーマットで数字に変換する場合は[toInt](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/to-int.html)や[toLong](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/to-long.html)でループが一回発生するだけですが、CharSequenceとして扱う場合はループが2回発生するという違いがあるので性能的には前者が良いはずですが、時間を扱うくらいではそこまでループは長くないので気にするほどではないかと思います。

## 要素の一部を合算

Listの値を一つに集約したい（合算値を出したい）場合があります。`sum`を使っても良いですが、これはそもそも要素が数字ではないと難しいですね。例えば要素が以下のようなクラスとなっているケースはどうしたら良いでしょうか。

```kotlin
data class Data(
    val name: String,
    val amount: Int,
    val price: Int
)
```

### 合算したい値が一つの場合

合算したい値が一つだけの場合は、`sumOf`で合算したい値だけを指定すれば良いです。以下は、`Data`クラスの`amount`だけを合算したい場合に使える方法です。

```kotlin
val list = listOf(Data("data1", 10, 100), Data("data2", 20, 200))
val totalAmount = list.sumOf { it.amount }
```

### 合算したい値が複数の場合

ここで`amount`のみでなく、`price`も合算したい場合はどうすれば良いでしょう。同じく`sumOf`を`price`にも使うことで実装はできますが、同じListに対してループが2回も発生するのあまり効率的ではありません。こういうときは、素直にそれぞれの合算値を変数として宣言しておいて`forEach`ループの中で値を足していく方が効率が良いでしょう。例えば以下のようにです。

```kotlin
var totalAmount = 0
var totalPrice = 0

list.forEach {
    totalAmount += it.amount
    totalPrice += it.price
}
```

もう一つの方法は、[fold](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html)を使う方法です。`fold`は`reduce`と似たようなもので、初期値(initial)を指定できるという違いがありますが、[reduceTo](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/reduce-to.html)のようにこの初期値の型はListの要素とは違うものに指定できます。そして関数を実行した結果はinitialと同じ型になるので、これを応用すると`Data`のリストを二つの値(`Pair`)に`reduce`することもできます。例えば上記の処理は`fold`を使うと以下のようにワンライナで実装ができます。

```kotlin
val (totalAmount, totalPrice) = list.fold(0 to 0) { acc, value ->
    (acc.first + value.amount) to (acc.second + value.price)
}
```

`fold`を使う場合、合算したい値が三つある場合は[Triple](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-triple/)を使うこともできますし、さらに値が多い場合は専用のクラスを作ることで対応もできるかと思います。ただ、こうする場合、合算した値を`val`として宣言できるというメリットはありますが、ループごとにインスタンスが作成されるので合算したい項目が増えれば増えるほど性能的にはあまり良くない可能性が高いので場合によって適切なものを選ぶ必要がありそうですね。

## 最後に

いかがだったでしょうか。私はずっとJavaでコードを書いていたので、完全にKotlinに転向した今でもついJavaらしいコードを書いてしまうのではないか、と思う時があります。元を辿ると、「Javaらしいコード」や「Kotlinらしいコード」がそもそも何であるかを考えなければならないとは思いますが、それでも、確かに言語が違うとその言語に合わせて自分のコーディングスタイルも変化する必要はあるのではないかと思います。そうすることで、より良いコードが書けるようになりそうな気がしていますので。

というわけで、これからもKotlinならではの、Kotlinに特化したコードを書くための工夫はこれからも続きます。特に今月はJava 17もリリースされたので、新しいAPIの一覧を眺めてKotlinではどう活用できるか考えてみたいですね。

では、また！
