---
title: "ifでの分岐を考える"
date: 2022-11-20
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
---

ほとんどの言語で、特定の条件に合致した場合にのみ実行する処理を書くには、`if`のように分岐処理のための構文を使うのが当たり前のように考えられています。言語によっては構文ではなく式として扱われたり、`switch`や三項演算子のような他の選択肢も存在しますが、基本的に「条件とそれに従う処理」を書く機能として本質は変わらないものですね。

分岐処理に限った話ではありませんが、便利な道具というのは時に危険性を伴うこともあります。`if`を使う場合、最初の実装ではわかりやすく簡単に目的を達成できますが、維持保守の観点からするとあまり良くない選択になるケースもありますね。たとえば条件が増えたり変わるなどコードに変更が必要となった場合は修正がすべてのケースを網羅しているかどうかがわからなくなったり、ユニットテストが困難になったりするなどが考えられます。

なので少なくとも`if`の処理をよりシンプルにしたり、もしくはデザインパターンなどで分岐の構文を使わず同じ処理ができるように改善する必要が出てくることもあるかなと思います。そうすることで、コードリーディングはより難しくなるとしても、維持保守の観点からはより良いコードになる可能性もあるでしょう。

もちろん、完全に`if`をなくするというのは不可能に近い話で、そこまでする必要もありません。道具自体に罪はなく、あくまで使い方が問題になるだけですので。ここではあくまで、`if`を使ったコードをどういう風にリファクタできるか、それだけに集中したいと思います。（初心者向けな感じで）

## if文の例

まずは、以下の関数をご覧ください。コードと価格を渡したら、内部ではコードに合わせて元の価格から割引の値を返すというものです。極めて単純化してはいますが、ECサイトのプロモーションなどでこのような処理が存在することもあるかなと思います。

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    return if (code == "Facebook") {
        (amount * 0.1).roundToInt()
    } else if (code == "Twitter") {
        (amount * 0.15).roundToInt()
    } else if (code == "Instagram") {
        if (amount < 1000) {
            amount
        } else {
            1000
        }
    } else {
        0
    }
}
```

すでにどのように処理をリファクタすれば良いか、一目でわかる方もいらっしゃるかなと思いますが、ここでは色々な観点でどうコードを変えられるかを述べたいので、一つ一つ項目を挙げて説明したいと思います。

## 関数のリファクタ

まずは関数内の処理をどう変えられるかを考えてみます。コードをより単純化して可読性を向上したり、処理の漏れをなくしたり、共通化できたり処理の単位が曖昧な場合は分離するなどいろいろな方法が考えられるはずです。

### 標準ライブラリ

上記の関数では`if`の中にさらに`if`が入っている構造となっているのがわかります。このようなネストは深くなれば深くなるほど良いコードとは言えませんね。なのでまずはここから直していきましょう。

`if`のネストを一つ減らす方法として、標準ライブラリを用いた方法を考えられます。標準ライブラリでなくでも関数として分離するという選択もありますが、標準ライブラリに処理を委任することでこの関数の負担がまず減るかなと思います。

Kotlinには[coerceAtLeast()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/coerce-at-least.html)という関数があり、パラメータとして渡された値をミニマムとして返すという働きをします。なので、`amount`が1000以下の場合は`amount`自身を、それ以外は1000を返すという処理はこの関数を使うことで単純化できます。以下のようにですね。

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    return if (code == "Facebook") {
        (amount * 0.1).roundToInt()
    } else if (code == "Twitter") {
        (amount * 0.15).roundToInt()
    } else if (code == "Instagram") {
        amount.coerceAtLeast(1000) // 閾値を超えない値となる
    } else {
        0
    }
}
```

単純に標準ライブラリを使用しただけですが、ネストが一つなくなりより単純なコードになりました。また、ここでの修正により閾値の修正が必要になった場合でも、1箇所だけを修正すると良いというメリットもありますね。

他にも標準ライブラリで処理ができそうな場所は積極的に利用したり、似たような処理が繰り返されるところがあったら切り出して自前のライブラリとして分離しておくことも良い選択になるでしょう。

### when

もう一つのリファクタとしては、`if`を`when`(他の言語の`switch`)に入れ替えるという方法を考えられます。`when`は結果的に同じ機能をするので、常に`if`の代替として良いというわけではありません。しかし、ifの条件が一律であれば、`when`を選ぶのは良い選択になる場合があります。

先ほどの`if`で分岐する条件は、あくまで`code`という文字列がどのような値となっているか比較することだけですね。他には特に条件がないので、`when`を用いた方がブランチを文字列だけで収めるのでより単純化つ明瞭なコードになります。たとえば以下のようにです。

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    return when (code) { // codeの値を比較するだけの分岐
        "Facebook" -> (amount * 0.1).roundToInt()
        "Twitter" -> (amount * 0.15).roundToInt()
        "Instagram" -> amount.coerceAtLeast(1000)
        else -> 0
    }
}
```

`if`を`when`に変えただけで、全体的に短くなってより読みやすいコードになっているかと思います。このように、`if`の条件文がどのようなものかをみて、`when`に変えるのも場合によっては良い選択の一つになり得るかなと思います。

### 拡張関数

Kotlinのような言語には、拡張関数で既存のクラスにメソッドを追加する機能がありますね。同じような処理が2箇所以上繰り返されているなら関数として分離を考慮すべきで、その関数を拡張関数として定義することも場合によっては考えられます。

ここでは`code`が`Facebook`か`Twitter`かによる分岐がありますが、やりたいことは`amount`に特定のパーセンテージをかけて返すことですね。なので、パーセンテージを求める関数を定義しておいた方が良いでしょう。

パーセンテージを求めるのはここでしか使わないとしたら`private`な関数として定義しても良いのですが、より汎用的な使い方ができるようにしたいなら、以下のような拡張関数を定義するのもありでしょう。

```kotlin
// パーセントを求める拡張関数
infix fun Int.percentOf(amount: Int): Int = (amount * this / 100)

fun getDiscountAmount(code: String, amount: Int): Int {
    return when (code) {
        "Facebook" -> 10 percentOf amount // 10パーセントの値を返す
        "Twitter" -> 15 percentOf amount // 15パーセントの値を返す
        "Instagram" -> amount.coerceAtLeast(1000)
        else -> 0
    }
}
```

先のほどのコードと比べ、拡張関数を定義することで処理の共通化ができたことと共に、「パーセントを計算する」という意図がコードでより明確に表れているようになっているのではないかと思います。

### Map

`if`や`when`を使わない場合でも分岐ができる場合はあります。たとえば`Map`を活用する方法ですね。`code`によって違う値を掛け算したいので、`code`をKeyに、掛けたい値をValueとするMapを定義しておくことです。たとえば以下のようなものです。

```kotlin
// コードと割引率
val discountPercentages = mapOf(
    "Facebook" to 10,
    "Twitter" to 15
)

fun getDiscountAmount(code: String, amount: Int): Int {
    // 割引率が定義してあったら掛け算
    discountPercentages[code]?.let {
        return it percentOf amount
    }

    // Mapにないcodeの場合
    return when (code) {
        "Instagram" -> amount.coerceAtLeast(1000)
        else -> 0
    }
}
```

ただ、この方法では全ての分岐を網羅することはできなくなりますね。`code`の値ががMapのKeyに含まれてない場合の処理が必要となるからです。ここでクロージャを用いると`code`の値が`Instagram`の場合の処理もMapに含めることができます。たとえば以下のようにですね。

```kotlin
// Valueを(Int) -> Intに変える
val discountRules = mapOf(
    "Facebook" to { amount: Int -> 10 percentOf amount },
    "Twitter" to { amount: Int -> 15 percentOf amount },
    "Instagram" to { amount: Int -> amount.coerceAtLeast(1000) }
)

fun getDiscountAmount(code: String, amount: Int): Int {
    return discountRules[code]?.let { it(amount) } ?: 0
}
```

Mapを利用する方法が条件分岐より良い方法だとは言い切れないのですが、コード別の割引率を他の関数でも参照する必要があるなど、複数の関数やクラスを跨いで共通の値を保持しておきたい場合は考えられる方法の一つになるかなと思います。この場合は、Map一つを修正するだけで全体の処理で整合性が保証されるコードになるという効果がありますね。

## OOP的な考え方

今までは単純に関数内部の処理をどう変えていくかについて述べましたが、より高度な方法ももちろんあります。OOPの考え方として捉えると、先ほどの関数は「割引額を求める」責任がありますが、その中で「割引」の定義そのものと、その計算式まで持っていることになります。なので、責任を分離していく必要がありますね。

この修正に処理は一見より複雑なものになっていくと感じる場合もあるかと思いますが、これはOOPに原則である[SOLID](https://ja.wikipedia.org/wiki/SOLID)を考慮したものでもあります。長期的な観点からすると、このような方法をとった方がより維持保守には向いていることになるでしょう。

### interface抽出

まずは「割引ポリシー」を`insterface`として分離します。この割引ポリシーを実装するクラスで実際のポリシーに従う割引額を計算するイメージです。

```kotlin
interface DiscountPolicy {
    fun calculate(amount: Int): Int

    companion object {
        val NONE: DiscountPolicy = object : DiscountPolicy {
            override fun calculate(amount: Int): Int = 0
        }
    }
}
```

あとはこの`interface`を実装するクラスを、コード別に定義しておきます。

```kotlin
class FacebookDiscountPolicy : DiscountPolicy {
    override fun calculate(amount: Int): Int = 10 percentOf amount
}

class TwitterDiscountPolicy : DiscountPolicy {
    override fun calculate(amount: Int): Int = 15 percentOf amount
}

class InstagramDiscountPolicy : DiscountPolicy {
    override fun calculate(amount: Int): Int = amount.coerceAtLeast(1000)
}
```

こうやって割引ポリシーを定義しておくと、`getDiscountAmount()`は以下のように変えられるでしょう。

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    val discountPolicy = when (code) {
        "Facebook" -> FacebookDiscountPolicy()
        "Twitter" -> TwitterDiscountPolicy()
        "Instagram" -> InstagramDiscountPolicy()
        else -> DiscountPolicy.NONE
    }
    return discountPolicy.calculate(amount)
}
```

### Factory

先ほどの`interface`抽出で割引ポリシー自体は分離できたものの、`getDiscountAmount()`ではまだ「割引ポリシーを生成する」という責任を持っています。これもまた別の役割として分離ができるでしょう。ここは以下のように割引ポリシーを生成する`Factory`を定義しておくと良いでしょう。

```kotlin
object DiscountPolicyFactory {
    fun getDiscountPolicy(code: String): DiscountPolicy {
        return when (code) {
            "Facebook" -> FacebookDiscountPolicy()
            "Twitter" -> TwitterDiscountPolicy()
            "Instagram" -> InstagramDiscountPolicy()
            else -> DiscountPolicy.NONE
        }
    }
}
```

最終的に、`getDiscountAmount()`は以下のように修正できます。`interface`の抽出や`Factory`の作成でコードの量は増えましたが、この関数の責任はより軽くなり、割引ポリシーの追加や修正が必要な場合でも柔軟な対応ができるようになりました。

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    val discountPolicy = DiscountFactory.getDiscountPolicy(code)
    return discountPolicy.calculate(amount)
}
```

### Enum

割引ポリシーを生成するために`Factory`を使う代わりに、`Enum`を使うこともできます。先ほどの`DiscountPolicy`を継承して、クラスではなく列挙定数として扱う方法です。たとえば以下のようなものが定義できます。

```kotlin
enum class DiscountPolicies(private val code: String) : DiscountPolicy {
    FACEBOOK("Facebook") {
        override fun calculate(amount: Int): Int = 10 percentOf amount
    },
    TWITTER("Twitter") {
        override fun calculate(amount: Int): Int = 15 percentOf amount
    },
    INSTAGRAM("Instagram") {
        override fun calculate(amount: Int): Int = amount.coerceAtLeast(1000)
    };

    companion object {
        fun fromCode(code: String): DiscountPolicy {
            return values().find { it.code == code } ?: DiscountPolicy.NONE
        }
    }
}
```

上記の`Enum`を利用する場合、`getDiscountAmount()`は以下のようになるでしょう。

```kotlin
fun getDiscountAmount(code: String, amount: Int): Int {
    val discountPolicy = DiscountPolicies.fromCode(code)
    return discountPolicy.calculate(amount)
}
```

`Enum`の場合、`fromCode()`で`code`による分岐そのものが必要なくなり、割引ポリシを追加したい場合でも列挙定数を追加することで拡張も容易にできるので`Factory`よりも良い方法ではないかと思います。

## 最後に

最初に断っておいたように、全ての`if`をなくすのは不可能に近い話で、そうする必要もありません。しかしその`if`が行っている処理の本質、責任、可読性のような要素は注意深く観察する必要があり、最初は`if`でとりあえず動くコードを作ったあとは他の方法で改善できるかどうか振り返ってみる必要はあるかなと思います。

こういう自分も常に綺麗なコードを書いているわけではないのですが、たまにはこうやって初心者の気持ちで自分の書いたコードを振り返ってみるという姿勢は常に必要なのではという気持ちにもなります。良いコードを書くのは常に難しいものですね。でも、難しいことを最初にやっておいた方が後に後悔しないことにもつながるだろうと思います。

では、また！
