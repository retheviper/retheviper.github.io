---
title: "JavaプログラマーがみたKotlin〜その二〜"
date: 2021-02-28
categories: 
  - kotlin
photos:
  - /assets/images/sideimage/kotlin_logo.jpg
tags:
  - kotlin
  - java
---

この度は、転職することとなり、仕事で使われる言語もJavaからKotlinに変わることになりました。個人的にKotlinで簡単なSpring WebFluxプロジェクトを作ってみたことはあり、もともとJavaプログラマがKotlinへ移行するのは難しいことではないと言われてはいるものの、やはり仕事で使われる言語が変わるというのはかなりのチャレンジではあると思います。なので、今まではJavaに関してのポストを主に載せていたのですが、これからはKotlinに関してのポストを増やしていきたいと思います。

まず、よく知られているように、KotlinはJavaと完璧な互換性を持つものです。それはJVM言語であり、コンパイルしたらJavaと同じくバイトコードになるからですね。ただ、だからと言って「Javaの感覚で」コードを書くということは、Kotlinという「違う言語」に移行する意味を無くす行為な気がします。なぜなら、Kotlinは触れば触るほどJavaとは根本的に違う考え方で設計されている言語だということが伝わってくるからです。最初はJavaの冗長さ(Verbose)を減らすことが第一の目標ではないかという印象を受けましたが、本格的に勉強を始めるとそれだけではないような気がしているのです。

今回のポストは、[Coursera](https://www.coursera.org)の[Kotlin for Java Developers](https://www.coursera.org/learn/kotlin-for-java-developers)の講義の内容に基づいて作成されました。

## 冗長さを減らすということ

Javaは今でも良い言語であり、多くの言語が発表されエンタープライズレベルで使われるようになった今でも、幅広い分野で使われていますね。Javaが依然としてよく使われる言語であることは、[TIOBE index](https://www.tiobe.com/tiobe-index/)やJetBrainsの[The State of Developer Ecosystem](https://www.jetbrains.com/lp/devecosystem-2020)、Stackoverflowの[Developer Survey](https://insights.stackoverflow.com/survey/2020)からも読み取れることです。

ただ、Javaが依然として人気の言語だとしても、それは圧倒的にJavaが他の言語に比べ優秀だとか、使いやすい言語であるという意味ではないでしょう。どの言語でもそうであると思いますが、Javaでよく指摘されている問題の一つは、「冗長すぎる」というところです。数多くのライブラリがあり、MavenやGradleのような優秀なビルドツールを使えながらも、やはり言語の仕様は変わってないですからね。Java 9からはこの問題を解消するため、他の言語から影響を受けたような機能を多く導入していますが(例えば、[instanceofのパターンマッチング](https://blogs.oracle.com/javamagazine/pattern-matching-for-instanceof-in-java-14)や[record](https://blogs.oracle.com/javamagazine/records-come-to-java)など)、言語そのものの設計思想が変わるというよりは「違う言語の特徴をJavaに合わせた仕様で導入する」ことに近いので、根本的な変化とは言えないものです。なので今まで書かれている冗長なコードは残るもので、またこれからも使われることになるはずです。

### コードが短くなる

冗長さを減らすということは、簡単にいうと「より短いコードで、同じ結果を得る」と定義できるでしょう。そういう観点からすると、KotlinはJavaの冗長さを減らすために工夫した痕跡が言語の仕様から感じ取れるようなものです。例えば以下のようなコードがあるとしましょう。

```java
public void updateWeather(int degrees) {
    String description;
    Color color;
    if (degrees < 10) {
        description = "cold";
        color = BLUE;
    } else if (degrees < 25) {
        description = "mild";
        color = ORANGE;
    } else {
        description = "hot";
        color = RED;
    }
}
```

これをKotlinで書き換えると、以下のようになります。

```kotlin
fun updateWeather(degrees: Int) {
    val (description, color) =
        if (degrees < 10) {
            Pair("cold", BLUE)
        } else if (degrees < 25) {
            Pair("mild", ORANGE)
        } else {
            Pair("hot", RED)
        }
}
```

まず二つの変数を、戻り値が`Pair`のオブジェクトの表現式でより短くできることがわかります。そしてこのコードは、`when`句を使ってより短い形で変えることもできます。その結果が以下です。

```kotlin
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 10 -> Pair("cold", BLUE)
        degrees < 25 -> Pair("mild", ORANGE)
        else -> Pair("hot", RED)
    }
}
```

さらに`Pair`は、`to`を使うことでもっと簡単に表現することもできます。そうすると、以下のようなコードになります。

```kotlin
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 10 -> "cold" to BLUE
        degrees < 25 -> "mild" to ORANGE
        else -> "hot" to RED
    }
}
```

最初のJavaのコードと比べ、かなり簡単かつ明瞭なコードになっているのがわかります。他の言語を使っていた人だとしても、一瞬ですぐに何をしているのかがわかって、より短く効率なコードになっているのがわかりますね。こういうところこそ、KotlinがJavaの冗長さ−無駄を減らすことに力を入れている部分ではないかと思います。

### コードを簡単に書ける

自分は最初、Kotlinの文法を簡単にみながら`switch`がここでは`when`に変わって、`case`を書く必要がないんだな、くらいの印象しか受けてなかったです。しかし、よくよく見ると、他にもJavaと違うところが良く見えます。ここで読み取れるものは、例えばさっきのコードだと以下がありますね。

- `when`句が表現式として使える
- `when`句の条件の対象は条件式の中のみで良い
- 表現式の中で複数の値を戻り値として定義し、それを
- `to`で二つのオブジェクトを`Pair`にまとめることができる

他にも、Javaの`switch`に比べKotlinの`when`句は以下のよう活用ができるというところもあります。オブジェクトの比較がより簡単ですね。例えば以下のようなコードで、簡単に二つのオブジェクトに対しても比較が可能になります。
    
```kotlin
fun mix(c1: Color, c2: Color) =
    when (setOf(c1, c2)) {
        setOf(RED, YELLOW) -> ORANGE
        setOf(YELLOW, BLUE) -> GREEN
        setOf(BLUE, VIOLET) -> INDIGO
        else -> throw Exception("Dirty Color")
    }
```

これをあえてJavaのコードで書くとしたら、おそらく以下のようになるでしょう。個人的に、たくさんの`else if`はあまり読みやすいコードではなく、書く立場としても綺麗ではないと思います。

```java
private Color mix(Color c1, Color c2) {
    if (c1 == Color.RED && c2 == Color.YELLOW) {
        return Color.ORANGE;
    } else if (c1 == Color.YELLOW && c2 == Color.BLUE) {
        return Color.GREEN;
    } else if (c1 == Color.BLUE && c2 == Color.VIOLET) {
        return Color.INDIGO;
    } else {
        throw new RuntimeException("Dirty Color");
    }
}
```

ここでわかるのは、KotlinではJavaと同じことをするとしても、短いだけでなく、より簡単にコードをかけるということですね。もちろん、別のメソッドを作ったり、`Comparable`なオブジェクトを作ったり、`Comparator`クラスを実装することでJavaでも似たようなことはできるかも知れません。しかし、そこまでしたいかというと微妙ですね。

もちろん、Java 12からはKotlinの`when`に近い感覚でコードを書くこともできるようになっています。表現式としても使えて、複数の条件を指定することができ、`Lambda`の感覚で書けるということも良いですね。

```java
var result = switch (month) {
    case JANUARY, JUNE, JULY -> 3;
    case FEBRUARY, SEPTEMBER, OCTOBER, NOVEMBER, DECEMBER -> 1;
    case MARCH, MAY, APRIL, AUGUST -> {
        int monthLength = month.toString().length();
        yield monthLength * 4;
    }
    default -> 0;
};
```

このような変化を見ると、この「冗長さを減らす」という面では、Javaもまたバージョンアップとともに新機能を次々と導入してきているので、Kotlinの魅力が半減しているように見えるかも知れません。しかし、Kotlinではもっと重要なポイントがまた一つあります。言語自体の拡張性です。

## 拡張ができるということ

言語自体の拡張性と言いましたが、簡単にいうと、以前にも紹介したことのある拡張関数、つまり`extension`のことです。Kotlinの仕様としてもこれは大きい部分として紹介されているものですね。これをよく使うと、ただ「継承しなくてもそのクラスにメソッドを追加できる」だけでなく、`infix`と組み合わせることでまるで予約後であるように使うことができます。

実際、[Kotlin for Java Developers](https://www.coursera.org/learn/kotlin-for-java-developers)のコーディング問題では、`infix`で書かれた以下の拡張関数を持って結果の確認を行っていると言われています。

```kotlin
infix fun <T> T.eq(other: T) {
    if (this == other) println("OK")
    else println("Error: $this != $other")
}
```

この`infix`を使うと、以下のようなコードが書けるようになります。

```kotlin
"ABC" eq "ABC"
```

このような特徴があるということは、使う側からしても便利ですが、これから言語そのもののバージョンアップにしたがってより便利な機能が追加され安いことにもなっていると思います。例えば先ほどの`Pair`オブジェクトを作る`to`が、このように`infix`関数として作られているものです。これからもこういった便利な機能が追加され、追加しやすくなるのは確かに開発のコストの削減をという面でも良いことですね。

## もう十分便利であること

冗長さを減らし、拡張性がある言語だという特徴は、おそらくKotlinを作っているJetBrainsにとっても十分有効な特徴であるかと思います。Kotlinのスタンダードライブラリを見ると、すでに便利な関数が多く存在しています。例えば、簡単なループでは以下のようなことができます。

```kotlin
val list = listOf("A", "B", "C")

// 一般的なfor文
for (element in list) {
    println(element)
}

// インデックスを含むfor文
for ((index, element) in list.withIndex()) {
    println("$index: $element")
}

// インデックスのみのfor文
for (index in list.indices) {
    println(index)
}
```

以前もJavaのfor文の性能についてのポストで簡単に述べたことがありますが、そこではJavaならインデックスが必要な場合は伝統的なfor文を使い、そうではない場合は一般的に拡張for文を使った方がいいという結論をJMHでのベンチマークで出していました。しかし、こうやってすでに言語から便利な方法を提供していると、性能を気にすることなく便利な方法を取れるという面でも魅力的です。

そして、forEachでもインデックスが必要であるなら、`forEachIndexed`を使えるという良い点もあります。例えば、以下のような書き方ができます。

```kotlin
// 一般的なforEach文
list.forEach(::println)

// インデックスを含むforEach文
list.forEachIndexed { index, element -> println("$index: $element") }
```

インデックスを簡単に取得できるということは、ループ対象のオブジェクトが持つ全インデックスを取得したい場合に、あえて`0`のような、マジックナンバーにありえる数値を指定する必要がないというところでも良いですね。Javaだと毎回、static finalなフィールドとして宣言したり、別の定数として管理したりするケースが多いので…

他にも、正規表現なしでも文字に関して簡単にチェックできる関数が事前に提供されているとか(`Char.isLetter()`や`Char.isDigit()`など)、`Map`には`Pair`で要素を入れることができるとか、iterableなオブジェクトからStream APIのような操作がすぐできるなど、確かにJavaに比べ「悩む必要がない」のが魅力的と思います。まぁ、人によってはこれはデメリットと認識する可能性もあるのでは、といは思いますが…

## 最後に

色々とKotlinの特徴・メリットについて述べましたが、こういう自分もまだ実際に業務でKotlinを使っているわけではないので、まだまだ表面的な知識のみに止まっていると思います。しかし、ここで紹介したことだけでも、Kotlinの魅力は感じ取れるのではないかと思います。

言語自体も魅力的なのですが、他にもKotlinを扱うことで得られるメリットは多いです。例えば、JetBrainsが開発しているので、Intellijとの相性が良いこと。JVM言語でありJavaとの互換性があるので、Javaの発展をそのまま吸収できるということ。NativeやJavaScriptへのコンパイルもできるということ。他の言語も十分魅力的なポイントはありますが、Javaプログラマーであるなら、一度Kotlinに触れてみる価値はあると信じています。皆さんもまだKotlinに触れたことがないのであれば、この度ぜひ軽い気持ちで挑戦してみてください。

では、また！
