---
title: "Kotlinはどう書いたらいいか"
date: 2022-12-30
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---

自分がJavaからKotlinに転向してからもう2年ほどが経ちます。しかし、いまだにKotlinでできることは無限にあって、新しい発見は終わることがないと感じています。Kotlinという言語自体のバージョンアップが早く、色々と機能が追加されて続けているのでまだしばらくこの発見も続きそうですね。

そんな中で思うことなのですが、Intellijを使っていると自動的にJavaのコードを変換してくれたり、Javaの書き方をそのまま流用しても問題になることは少ないものの、やはりKotlinならではのコードを書きたいという欲求も湧いてきます。つまり、「Kotlinらしき書き方」をしたいと思ってしまうのです。

## Korlinらしき書き方って？

「Kotlinらしき書き方」とは一体どういうものなのでしょうか。まずはその定義が必要ですね。いろいろな捉え方があるかと思いますが、私は基本的に「Kotlinの仕様や機能を最大限に活かすこと」なのではと思っています。つまり、スタンダードライブラリのAPI、Scope Function, Extensionなどを積極的にコードに取り入れることです。そうすることでコードを書く時間は短くなり、より効率が上がるだろうと私は思っています。

ただ、そういう概念を言葉で述べるだけでは曖昧なところがあるので、コードを持って例を挙げた方がいいでしょう。例えば、以下のような関数を実装する必要があるとします。

```kotlin
fun numbers(): List<String> {
    // TODO
}
```

この関数を通じて行いたい処理は、「0〜10の数字をStringに変換してListとして返す」ことだとしましょう。その場合、実装の一例として以下のようなコードを提示できるかなと思います。

```kotlin
fun numbers(): List<String> {
    val list = mutableListOf<String>()
    var i = 0
    while (i < 10) {
        i++
        list.add(i.toString())
    }
    return list.toList()
}
```

ここで[kotlin.collections.map()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/map.html)を使ったら、同じコードを以下のように変換することができます。

```kotlin
fun numbers(): List<String> =
    (0..10).map { it.toString() }
```

`map()`のような関数が初めて登場した際は、あまり直感的にその処理の意味を把握できないということから「可読性に欠ける」という評価もあったようです。上記の二つのコードを比べると、場合によっては`while`ループでの処理がわかりやすいと感じる方もいらっしゃるかなと思います。今は`map()`が多くのプログラミング言語で採用している関数であり、期待できる関数の実行結果も常識化していますが、古典的なコードに慣れている人からしたら「ループで何かを行う」というコードの方がわかりやすいかもです。

## 「可読性」として

しかし、果たしてより高度な処理を行う場合も「長いコードの方が可読性は良い場合もある」と断定できるのでしょうか。ここでまた、違う例を挙げてみましょう。例えば、`Listの数字を全て掛け算した結果を返す`関数を実装する必要があるとします。例えば以下のようなコードを書くことができるでしょう。

```kotlin
fun multiply(numbers: List<Int>): Int {
    if (numbers.isEmpty()) {
        throw IllegalAccessException("List is empty")
    }
    var acc = numbers[0]
    if (numbers.size == 1) return acc
    for ((index, value) in numbers.withIndex()) {
        if (index != 0) {
            acc *= value
        }
    }
    return acc
}
```

上記のコードでは当初の目的は果たしていますが、「可読性」という観点からするとどうでしょうか。関数のシグニチャーだけでなく、関数で行なっている処理全体に目を通さないと、何が起こっているかわからないのではないでしょうか？

この関数で行なっている処理を一つ一つ読んでみると、「リストが空の場合はエラーを返す」、「リストの要素が一つの場合はそれを返す」、「リストが空でなく、要素が一つでもない場合は後続の処理を行う」、「最初の要素をとり、リストの要素をループしながら最初の要素以外を全て掛け算する」という情報が込められています。

ここでExtensionと`kotlin.collections`の関数を使って同じ処理を行う関数を実装してみると、コードは大きく変わります。以下がそのサンプルコードです。

```kotlin
fun List<Int>.multiply(): Int =
    reduce { acc, e -> acc * e }
```

`map()`の場合と同じく、`reduce()`という関数が何をするかがわからない場合もあるかと思います。しかし、その関数が「要素を一つに減らす」ことであり、その中で行なっている「減らし方は掛け算」ということを理解すれば良いだけなので、こちらの方がその意図を把握しやすいのではないでしょうか。先ほどの関数は、処理全体をいくつかの単位で分けて理解する必要があったということから考えると、可読性の面ではこちらの方がより優秀だと言えるのではないかと私は思っています。

また、`List`に拡張関数として定義することで、まるで元からついていたメソッドのように使えるのも良い点でしょう（IDEから自動補完に出てくるはずなので）。

## 「工数」として

コードを書くのは工数がかかる行為です。なので、同じ機能をする関数を書くとしても、毎回全ての処理を書くより共通化できる部分は分離し、再利用するのが常識のようなものです。そのため多くのライブラリやフレームワークが存在していますね。

プログラミング言語の使用や機能をよく理解し、それらを活用するということも本質的にはライブラリやフレームワークを使うことと変わらないものです。先のように、「Listの要素を一つにまとめる」処理を毎回自前で書くとしたら、かなりの時間が必要となるでしょう。

ここで他の例をまた挙げます。例えば以下のようなデータがあるとします。

```kotlin
data class Data(val id: Int, val version: Int)

val list = listOf(
    Data(1, 1),
    Data(1, 2),
    Data(2, 1),
    Data(2, 2),
    Data(2, 3)
)
```

DBの中に履歴を残したい場合はversionや枝番など名称の列を持たせ、同じIDのデータをいくつか挿入する場合があるかと思います。そしてその場合、最新のデータのみを処理したいケースもありますね。上記の例だと、`Data(1, 2)`と`Data(2, 3)`のみを取得したいということです。

最初からクエリで最新のデータのみを取得できるといいのですが、外部APIのレスポンスの場合はそのようにフィルタされたデータでない場合もあります。なのでこちらでversionがmaxのデータをフィルタする処理を書くとします。例えば以下のようなコードを考えられます。

```kotlin
fun maxVersions(list: List<Data>) {
    val result = mutableListOf<Data>()
    for ((index, value) in list.withIndex()) {
        if (index == 0) {
            result.add(value)
        } else {
            val last = result[result.size - 1]
            if (value.id == last.id) {
                if (value.version > last.version) {
                    result.remove(last)
                    result.add(value)
                }
            } else {
                result.add(value)
            }
        }
    }
    return result.toList()
}
```

可読性の問題は一旦置いといて、このようなコードを書くときの工数はどうでしょうか。慣れてしまえば簡単なのかもしれませんが、初めてこの処理を書く人の立場からしたらかなりの工数がかかり、関数が期待通りに動作するかの検証を含めるとさらに工数が必要となりそうだと思います。慣れている場合でも、常にこのようなコードを的確に書けるかどうかが疑問です。

それに対して、スタンダードライブラリを使った例を考えてみます。以下のように、メソッドチェーンによって簡単に同じことができます。

```kotlin
val maxVersions = list.groupBy { it.id }.map { it.value.maxBy { it.version } }
```

ここで使われている`groupBy()`、`map()`、`maxBy()`はそれぞれ「ValueがListのMapを作る」、「要素を違う形にマッピングする」、「Listの要素のうちmaxの値を探す」という関数なので、ここだけでなく色々な場面で活用できる関数となっています。このように便利な関数を使いこなし、さらに組み合わせることでより複雑な処理でも簡単に書くことができるという点をみると、スタンダードライブラリの機能を理解するのは工数の面でもかなり効率を上げられることになるのではないかと思います。

## 注意点

ここまではKotlinの仕様や機能を活かすと可読性と工数という二つの観点から、メリットがあるという話をしてきました。しかし、どんなことでもメリットがあればデメリットもあるものですね。

当たり前ながら、どんな機能でも単純に「それができる」という理由で乱用するとむしろ逆効果が出るケースがあります。例えば以下のような例を考えられます。二つのデータクラスがあって、`Data`から`Request`にマッピングする必要があるとしましょう。

```kotlin
data class Data(
    val id: Int?,
    val version: Int?
)

data class Request(
    val id: Int,
    val version: Int
)
```

`Data`のフィールドはnullableとなっていますが、`Request`の場合はそうではないです。このようにビジネスロジック上はnullになることはなくても、実装上の都合によってnullableにするケースもありますね。その場合、どうやってマッピングしたらいいでしょうか。例えば以下のような例があるとします。

```kotlin
data class Request(
    val id: Int,
    val version: Int
) {
    companion object {
        fun from(data: Data): Request {
            return Request(
                id = requireNotNull(data.id) {
                    "${Data::id.name} should not be null."
                },
                version = requireNotNull(data.version) {
                    "${Data::version.name} should not be null."
                }
            )
        }
    }
}
```

ここではcompanion objectを利用して`Request`のインスタンスを生成しながら、データのマッピングを行うようにしています。そこで、[requireNotNull()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/require-not-null.html)を利用したバリデーションを行い、nullだった場合はエラーメッセージを出力するようにしていますね。

ここまでは良い実装だとして、問題はエラーメッセージの方です。どのパラメータがnullだったかわかるようにしていますが、そこで`Data`のフィールドをRefelctionで取得し、そのフィールド名をStringに埋め込むようにしています。エラーが発生した時には意図的に動作するとしても、果たして性能の劣るRefelctionを使うまでのことだったでしょうか。

このように、言語が提供する機能を活用する場合は適切な場面を判断する必要があります。ここで紹介した`map()`や`reduce()`、`groupBy()`なども便利で簡単な実装ができるようにしてくれる優秀な関数ですが、これらの関数の実装をみるとそれぞれが一つのループを必ず行い新しいCollectionを生成するという処理を行なっているということを理解すれば、メソッドチェーンで複数の関数を利用する場合は性能に影響を及ぼす可能性もあるということがわかるでしょう。他にも可読性の面でもわかりにくくなったり、別の関数として分離した方が良いコードが一つの関数内でいくつも繰り広げられることになる可能性もあるかと思います。

なので、「曖昧な理解」や「慣性」としてこれらの機能を利用するという行為にはリスクがあるということを理解し、どのような実装をするかは常に悩むべきではないかと思います。

## 最後に

すでに何回か、「Kotlinで書いてみた」というタイトルで、Kotlinならこういう書き方ができるというポストを載せていて、今回の記事もまたそれらと大きく離れた題ではないです。また、ある意味ここで挙げたことはプログラマなら基本的に熟知しておくべき常識のようなもので、今更な感があるのかもしれません。

それでもあえて記事として書いている理由は、基本こそ大事だという自分の考えゆえでもあります。経験を積んでいくと発展する部分もありますが、逆に変な習慣がついてしまいなかなか直せないところも出てくるものですので。自分はそうでないかという警戒を兼ねて、一度は基本に戻り今の自分と照らし合わせてみるのもまた一つの勉強になるのではないかと思います。ちょうど年末ということもありますが。

少し遅れてしまいましたが、2022年のポストはこれにて終わりとなります。来年は自分も、ここに訪れるみなさんにも成長のある一年になることをお祈りします。

では、また！
