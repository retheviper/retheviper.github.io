---
title: "Effective Kotlinを読む"
date: 2022-02-20
categories: 
  - kotlin
image: "../../images/kotlin.jpg"
tags:
  - kotlin
  - java
  - design pattern
---

今回は久々に本を読んだのでそれに関する感想を少し書こうと思います。転職前は主にJavaを扱っていたため、[Effective Java](https://www.amazon.co.jp/Effective-Java-%E7%AC%AC3%E7%89%88-%E3%82%B8%E3%83%A7%E3%82%B7%E3%83%A5%E3%82%A2%E3%83%BB%E3%83%96%E3%83%AD%E3%83%83%E3%82%AF-ebook/dp/B099922DML/ref=sr_1_1?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=1OT3QRYCGG9BB&keywords=Effective+Java&qid=1645265607&sprefix=effective+kotlin%2Caps%2C753&sr=8-1)を読み自分の書いたコードを振り返って見たことがありました。転職後はKotlinという違う言語を触るようになったものの、やはりJVMで動く言語であり、現在使っているフレームワークもSpringから変わってないので基本的には同じ観点でコードを作成すると良いのかなと思っていました。しかし、Kotlinに触れてから1年が経った今、やはり言語が違うとコードを作成するときの週間も一度は見直す必要があるのではないかと思っています。

そこで、ちょうど[Effective Kotlin](https://www.amazon.co.jp/Effective-Kotlin-Best-practices-English-ebook/dp/B08WXCRVD2/ref=sr_1_1?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=2HVT6TZYJL65A&keywords=effective+kotlin&qid=1645264802&sprefix=effective+kotli%2Caps%2C230&sr=8-1)という本を発見したので早速読んでみました。そして今回のポストではその内容について色々と書こうと思います。

ちなみに、この本自体は出版されて少し経つので、ネット上でもたまにその内容やPDFの資料などを発見することあができました。例えば、この本で「可読性」のチャプタに関しての話は[こちらのブログ]の方によく整理されてあるので、参考にしてください。

## 全体的な印象

個人的に`Effective Java`は上級者向けの本で、ある程度Javaでアプリを書いたこと経験のある人でないと理解が難しいところが多かったかなと思います。例えば、「`try-finally`を
`try-with-resource`に代替した方がいいとか、`Stream`で副作用のない関数を書く方法などが紹介されていますが、これらはやはりある程度Javaという言語の設計と仕様に対する基盤知識を必要とするものですね。

それに比べ、`Effective Kotlin`には初心者向けの内容も結構あります。例えば、そもそものオブジェクト志向が何かのような内容がありました。ただそれだけではどう思っても`Effective Java`を意識したような題名が意味をなくすと判断したからか（前書きでも`Effective Java`を言及しています）他には「ベストプラクティス」として書かれているものも多いです。

そして当たり前ながら、Kotlinにおいても有効なことは`Effective Java`と重なっているような部分もあります。例えば、オブジェクトのインスタンスを作るときはfactory methodを書いた方が良いとかがそうですね。

ただ、Kotlinのバージョンアップの速さに追いついてないと感じるところがあったり（これは出版物の限界でもありますが）、上級者向けの内容は多少十分ではない感覚でしたので、どちらかというとジュニア向けの感覚ではあります。

## 興味深い

ジュニア向けといいつつ、自分もまだジュニア（の気持ち）であるため、興味深いと思ったところもありました。ここでその一部を紹介したいと思います。

### Single responsibility principle

いわゆる[SOLID](https://ja.wikipedia.org/wiki/SOLID)に触れるパートです。Kotlinでは拡張関数を用いることで、単一責任の原則を守れるという主張をしていました。まず以下のようなケースがあるとしましょう。

```kotlin
class Student {
    // ...

    fun isPassing(): Boolean = 
        calculatePointsFromPassedCourses() > 15

    fun qualifiesForScholarship(): Boolean = 
        calculatePointsFromPassedCourses() > 30

    private fun calculatePointsFromPassedCourses(): Int {
        //...
    }
}
```

ここで`isPassing()`は`accreditations`というモジュールで、`qualifiesForScholarship()`は`scholarship`というモジュールで使われるとします。そしたら、`Student`というクラスがこれらの関数を持つのは単一責任としてどうか、ということですね。

なので、モジュール別にこれらの関数を拡張関数として定義することが良いとのことでした。

```kotlin
// scholarship module
fun Student.qualifiesForScholarship(): Boolean{
    /*...*/
}

// accreditations module
fun Student.calculatePointsFromPassedCourses(): Boolean{
    /*...*/
}
```

もしくは`calculatePointsFromPassedCourses()`を外に出す方法を考えられるでしょう。しかし、この場合はこれらの二つのメソッド専用のprivateメソッドとしてつかえません。なので、

1. どのモジュールでも使える共通関数を作っておく
2. department別にhelper関数を作っておく

とかの方法も考えられます。

確かに、よく考えると拡張関数の良いところは「interfaceの実装ややスーパークラスの継承なし」でも簡単に処理を追加できるということなので、このような使い方をするのがユースケース別に処理を分けられて良さげな気がします。特に拡張関数を使うと、関数を配置するパッケージと可視性の制御が効くというところが個人的には新しい発見でした。

### Consider defining a DSL for complex object creation

オブジェクトの作成時の複雑な処理はDSLを使いましょう、というパートです。Kotlinですでに提供している例としたら、HTMLがありますね。以下のような形で定義することになります。

```kotlin
body {
    div {
        a("https://kotlilang.org") {
            target = ATarget.blank
            +"google"
        }
    }
    +"Some content"
}
```

確かにKtorのようなフレームワークでもよく使われている物なので、ある程度需要はあるのかなという気がしました。Kotlinだと高階関数を作るのが難しくはないので、十分挑戦できるところでもありますね。

ただ、DSL特有の書き方を確立し、その書き方をエンジニアに共有することや最初の設計と維持管理が難しそうな気がするので、アプリの縮小が求められる今のご時世に果たして合うかとうかは少し疑問ののころところでした。

個人的に何かのライブラリやフレームワークを作るとしたら、挑戦してみたいなと思いました。

## まあそうだよねって思ったところ

なんとなくそうではないかと思っていたところを（もしくはどこかで聞いて理論的な部分は忘れたけど、無意識的のうちに習慣化されていた部分を）文として親切に整理してくれているようなパートもありました。なのでもう一度自分の考えを再確認できたといえるところでしょうか。

### Do not repeat common algorithms

「スタンダードライブラリで解決できる一般的なアルゴリズムを自前のコードで書くな」というパートです。理由は以下の通りです。

1. 呼び出しの方がコードを書くより時間が短くかかる
2. わかりやすい名前になっている
3. コードがわかりやすくなる
4. 最適化が効く

私自身もなるべくスタンダートライブラリを活用した方が良いと思っていたので、ここはすぐに納得できました。自分で書いた処理が果たして最適化されたものかどうかもわからないし、業務使用以外のロジックを触るのは避けたいという理由でした。

この本では、以下のようなコードを上げています。自前のロジックを書いた場合です。

```kotlin
val percent = when {
    number > 100 -> 100
    number < 0 -> 0
    else -> number
}
```

上記のコードは、[coerceIn()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/coerce-in.html)を使うことでシンプルにできます。以下がその例です。

```kotlin
val percent = number.coerceIn(0, 100)
```

Kotlinには特にスタンダードラリブラリに良い関数が多いので、自前のロジックを書くよりは一度どんなAPIがあるのかを確認した方が良いケースが個人的には多かった気がします。そしてそれが納得できる理由が書いてあってよかったと思います。

### Implementing your own utils

スタンダードライブラリで解消できる問題以外で、プロジェクトに必要な共通の処理はユーティリティ関数として作っておきましょうってパートです。ユーティリティはクラスでなく、拡張関数として作ったら以下のようなメリットがあるらしいです。

- 関数は状態を持たないので副作用がない
- トップレベル関数と比べると型が決まっているので使い勝手がいい
- 引数よりはクラスについた形が直観的
- オブジェクトに関数をまとめるより必要な機能を探しやすい
- 特定のクラスに従属されるので親クラスのものか、子クラスのものか悩まない

確かにJavaを使っていた時は、私もいわゆるSingleton Patternでユーティリティクラスを作ったり、DIして使えるクラスを定義しておいて、staticメソッドを書いていました。Kotlinだと、ユーティリティクラスなしでも特定のクラスに関数を追加することができるのでより使い勝手がよくなります。

例えば、同じことをするとしても、拡張関数で書く場合とユーティリティクラスを作る場合のコードは以下のような違いがあります。

```kotlin
// 拡張関数を使う場合
val isEmptyByExtension = "Text".isEmpty()

// ユーティリティクラスを使う場合
val isEmptyByUtilClass = TextUtils.isEmpty("Text")
```

ユーティリティクラスを使う場合はまず、「どのユーティリティクラスの関数を使うか」を考えなければならないです。それに比べ、拡張関数はIDEでも自動補完ですぐ欲しい関数を見つけることができるので、より直観的なものになっていますね。

他にも具体的なクラスにのみ追加ができたりするので、より安全な使い方ができるというのも良いところですね。色々と拡張関数は使い道が多いなという、再発見ができたと言えるところでしょうか。

### Builder pattern

Kotlinでは[named arguments](https://kotlinlang.org/docs/functions.html#named-arguments)が使えて、Builderパターンがいらないというパートです。KotlinでもBuilderパターンを使うことが技術的に不可能ではないが、named parameterを使った方が良い理由としては以下が挙げられています。

- より短い
- より綺麗
- 使い方がシンプル
- スレッドセーフ

私自身も、Javaで使っていたのでKotlinでもBuilderパターンが必要かなと思ったことがありますが、いらないという結論を出しています。上記で挙げた理由ももちろん妥当ですが、Builderパターンだとインスタンスを生成するときに必須パラメータが揃っているかどうかを判断するのが難しくなるという理由からでした。

例えば本に出てくるBuilderパターンの例があるとしましょう。

```kotlin
class Pizza private constructor(
    val size: String,
    val cheese: Int,
    val olives: Int,
    val bacon: Int
) {
    class Builder(private val size: String) {
        private var cheese: Int = 0
        private var olives: Int = 0
        private var bacon: Int = 0

        fun setCheese(value: Int): Builder = apply { cheese = value }

        fun setOlives(value: Int): Builder = apply { olives = value }

        fun setBacon(value: Int): Builder = apply { bacon = value }

        fun build() = Pizza(size, cheese, olives, bacon)
    }
}
```

このBuilderは以下のような使い方ができると思います。

```kotlin
val villagePizza = Pizza.Builder("L")
        .setCheese(1)
        .setOlives(2)
        .setBacon(3)
        .build()
```

しかし以下の場合でもビルドはできますね。

```kotlin
val villagePizza = Pizza.Builder("L").build()
```

もし`cheese`、`olives`、`bacon`が`0`を許容しない作りになっていると、これを修正するのは大変なことになるでしょう。もしくは、パラメータが複雑な作りのオブジェクトだったらデフォルト値を設定するか、強制null check(`!!`)などを入れるか…より複雑になるだけですね。

しかし、named parameterを使うと簡単に解決できる問題です。デフォルト値を指定しない`val`だったら、それが必須項目であるということもわかりやすいですね。

```kotlin
val myFavorite = Pizza(
                    size = "L",
                    cheese = 5,
                    olives = 5,
                    bacon = 5
                )
```

### Consider factory functions instead of constructors

Javaでも最近は色々とfactory functionを導入していて、簡単にimmutableなオブジェクトを作りやすくなりました。Kotlinでもコンストラクタの作成や、named parameterによるインスタンスの生成が色々と便利ではあるものの、それでもfactory functionが良いケースがあるというパートです。理由は以下の通りです。

- 関数には名前があるので、どうやってオブジェクトが生成されるかわかる
  - `ArrayList(3)`よりは`ArrayList.withSize(3)`がわかりやすい
- 戻り値としてサブタイプのオブジェクトを指定できる
  - 具体的な実装を時と場合によって違う形にすることができる
- 呼び出されるたび新しいオブジェクトを作るわけではない
  - `Connections.createOrNull()`のようにnullを返すこともできる
- まだ存在しないオブジェクトを提供できる
  - プロキシなしで動くようなオブジェクトを作るなどで応用できる
- オブジェクトの外に作ることで可視性を制御できる
- `inline`にできるので、[reified](https://kotlinlang.org/docs/inline-functions.html#reified-type-parameters)にもできる
- インスタンスを作るのが複雑なオブジェクトの手間を省く
- スーパークラスやプライマリコンストラクタを呼び出さずにインスタンスを生成できる

こちらも読みながらなるほどと納得しました。特に私の場合でも、Service層のDTOとController層のResponseなどのオブジェクト間のマッピングではfactory functionを導入してコードを再使用性を高められたと思っていたので、良い判断だったなと今は思っています。

他に、factory functionを作る方法としても以下のようなものが提示されてありました。一般的にはcompanion object内に定義しておくことが多いかと思いますが、他の方法も必要であれば考慮したいものですね。

#### companion object

Javaのstaticメソッドのようなパターン。最もわかりやすいですね。以下のような形です。

```kotlin
class MyLinkedList<T>(
    val head: T,
    val tail: MyLinkedList<T>?
) {
    companion object {
        fun <T> of(vararg elements: T): MyLinkedList<T>? {
            /*...*/
        }
    }
}

// Usage
val list = MyLinkedList.of(1, 2)
```

factory functionは大体以下の規則を持って命名されるという説明もありました。

##### from

一つのパラメータを渡し、タイプを変える時

```kotlin
val date: Date = Date.from(instant)
```

#### of

複数のパタメータを渡し、それを束ねタイプにするとき

```kotlin
val faceCards: Set<Rank> = EnumSet.of(JACK, QUEEN, KING)
```

##### valueOf

`of`の違う形

```kotlin
val prime: BigInteger = BigInteger.valueOf(Integer.MAX_- VALUE)
```

##### instance / getInstance

Singletonのインスタンス取得（パラメータが同じだと常に同じインスタンスが帰ってくる）

```kotlin
val luke: StackWalker = StackWalker.getInstance(options)
```

##### createInstance / newInstance

instance / getInstanceは似ているが、常に新しいインスタンスを返す

```kotlin
val newArray = Array.newInstance(classObject, arrayLen)
```

##### getType

instance / getInstanceと似ているが、違うタイプのインスタンスを返すとき

```kotlin
val fs: FileStore = Files.getFileStore(path)
```

##### newType

createInstance / newInstanceに似てるが、違うタイプのインスタンスを返す時

```kotlin
val br: BufferedReader = Files.newBufferedReader(path)
```

#### extension

クラスにからのcompanion objectを定義しておいて、外部から拡張関数でfactory functionを付ける形です。元のクラスをいじらなくても良くなるし、パッケージと可視性の制御など拡張関数の持つ特徴を活用できますね。

```kotlin
interface Tool {
    companion object { /*...*/ }
}

fun Tool.Companion.createBigTool( /*...*/ ): BigTool{
    //...
}
```

#### top-level

[listOf()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/list-of.html)、[setOf()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/set-of.html)、[mapOf()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/map-of.html)のようなものですね。

よく使うタイプに関しては使い勝手がいいので便利なものの、IDEの自動補完などに現れたら混乱するケースもあるので命名は慎重にする必要がある、とのことでした。

#### fake constructor

大文字を使って、関数をコンストラクタに見せかけるものです。Kotlinのスタンダードライブラリとしては、以下のようなものがあります。

```kotlin
List(4) { "User$it" } // [User0,User1,User2,User3]
```

これは実際は以下のような関数ですね。

```kotlin
public inline fun <T> List(
    size: Int,
    init: (index: Int) -> T
): List<T> = MutableList(size, init)

public inline fun <T> MutableList(
    size: Int,
    init: (index: Int) -> T
): MutableList<T> {
    val list = ArrayList<T>(size)
    repeat(size) { index -> list.add(init(index)) }
    return list
}
```

これはinterfaceに対してコンストラクタを作る必要があったり、`reified`タイプの引数が必要な時に考慮できるものらしいです。

他にもfake constructorを作る方法があります。

```kotlin
class Tree<T> {

    companion object {
        operator fun <T> invoke(size: Int, generator: (Int) -> T): Tree<T> {
            // ...
        }
    }
}

// Usage
Tree(10) { "$it" }
```

ただ、この場合constructor referenceではコードが複雑になる問題があるらしいですね。

```kotlin
// Constructor
val f: () -> Tree = ::Tree

// Fake Constructor
val d: () -> Tree = ::Tree

// Invoke in companion object
val g: () -> Tree = Tree.Companion::invoke
```

なのでfake constructorを使うとしたら、関数として定義したほうがよさそうです。

#### factory class

別途Factoryというクラスを置いてインスタンスを返すようにする方法ですね。Javaではinterfaceでそのようなことをするケースがありますが（[List.of()](https://docs.oracle.com/javase/jp/11/docs/api/java.base/java/util/List.html#of(E))みたいな）、Kotlinでも良いのか？という疑問が湧きました。結論から言いますと、「factoryクラスは状態を持つことが可能」なため、場合によっては考慮しても良いとのことです。これは思ったより活用できそうな可能性がありますね。

```kotlin
data class Student(
    val id: Int,
    val name: String,
    val surname: String
)

class StudentsFactory{
    var nextId = 0
    fun next(name: String, surname: String) =
        Student(nextId++, name, surname)
}

val factory = StudentsFactory()
val s1 = factory.next("Marcin", "Moskala")
println(s1) // Student(id=0, name=Marcin, Surname=Moskala)
val s2 = factory.next("Igor","Wojda")
println(s2) // Student(id=1, name=Igor, Surname=Wojda)
```

## 最後に

ざっくりなまとめとなりますが、以上が私のこの本で得られた知識への感想となります。新しい発見もあり、自分の習慣が間違ってなかったということを人の説明で補ってもらったような気にもなり、かなり興味深かったです。

ただやはり、Kotlinがまだ新しい言語であり、いろいろなパラダイムを吸収しているためか、`Effective Java`のようなレベルの高い作法に対する議論は少し足りてないような気がしていて、そこは多少残念に思います。まあ、こう思うようになったということ自体が、少しは自分が成長した証拠でもあるかなという生意気な想像もしてみるのですが。

では、また！
