---
title: "JavaプログラマーがみたKotlin"
date: 2020-10-25
categories: 
  - kotlin
photos:
  - /assets/images/sideimage/kotlin_logo.jpg
tags:
  - kotlin
  - java
---

KotlinがAndroidの公式言語になってからもだいぶ長い時間が経ちましたが、まだまだWebアプリケーションの業界ではサーバサイド言語としてはJavaを使う企業も多く(自分の場合がそうです)、モバイル業界でもJavaを使うところが多いようです。Javaも9以降はアジャイル開発でバージョンアップにスピードを出していて、いわゆるモダンな言語の特徴を吸収していってますが、そもそもの言語のデザインが古いし、互換性のために昔ながらの名残を捨てられてないところもあるので根本から違う哲学でデザインされた言語とはかなり違うはずです。また、JVMを利用しない[Kotlin Native](https://kotlinlang.org/docs/reference/native-overview.html)も発表されているので、今後Javaよりも活躍できる場面が多いかも知れないなーという気がしたりもします。([GraalVM](https://www.graalvm.org/)は使われることあるのかな…)

取りまとめ、まだ正式の研修とかを受けたわけでなく、あくまでSpring WebFluxを使って簡単なアプリを作ってみるついでに使ってみただけなので、よくわかってない部分も多いかと思いますが、今まで触れてみた感想をJavaプログラマーの観点から簡単に紹介したいと思います。

## これがよかった

まずは使ってみてよかったところから。結論からとなりますが、良いと思ったところはおおよそイメージ通り(期待してた通り)という感覚です。

### やはりモダンな感じ

Kotlinで書いたコードを見ると、モダンな言語だとやはりこんなものかなという感覚ではあります。モダンな言語が何か、という定義から必要になるのではという気もしますが、例えばSwift、Kotlin、Goみたいなものですね。あまり他の言語に詳しいわけではないのですが、これらの言語はなんとなくPythonに似ているような気がします。例えば`var`、`fun`のように基本文法で略語をよく使っていたり、型の指定はコロンの後につけたり、セミコロンがなかったり、`in`や`Range`、`is`があるなどの共通点があったりしますので。

ただ、そんなモダンな感覚でありながらも、やはりKotlinはJavaよりな感覚ではあります。厳格なJavaをよりゆるくしただけの感覚といえばいいでしょうか。例えばPythonだと`elif`なのですが、Kotlinでは`else if`だったりしますので。JVM言語という理由だけでなく、基本文法からしてもJavaプログラマーならすぐに適応できる言語でもあります。例えばforループにラベルをつけることができたりします。

あえてJavaとの比較をするとしたら、やはり冗長さを省けただけでなく、Javaという言語のデザインを根本的に変えようとしている気もしていました。例えばNullや、Mutableを扱う方式がそうです。Kotlinでは基本的に変数はNullになれなくて、Nullになれるオブジェクトは最初からそうであると宣言する必要があり、Nullになれるオブジェクトを扱う時もSafe Callを強制することでNullに対してはコンパイラレベルではできるだけサポートしている感覚です(おそらくこれはモダンな言語だと全部がそうですが)。そしてCollectionなどを宣言する時も、あえてMutableという宣言をしない限りは基本的にImmutableなオブジェクトが生成されます。これだけでもJavaでもっともよく見つかるNPEをよく避けられる気がしてしまいます。いちいち宣言して、コールもクエスチョンマークをつける必要があるのは面倒臭いことな気もしますが、コンパイルエラーの方がランタイムエラーよりはずっとマシだというのは我々みんなが知っていることではないでしょうか。

あとは、個人的にPythonで使ってみてよかったなと思った機能がKotlinにもあってよかったです。例えばMultiple Return(複数の戻り値)だったり、Named argument(名前付き引数)があります。前者は特に、Pair/Tripleという型で明確な戻り値を提示できるのが素晴らしいと思います。こういうところはモダンながらも、Javaの持つ安定性もしくは丈夫さを捨ててなかったなという印象を与えてくれました。

ただ、これらのメリットは最近のJavaもかなり近づいている状態ではあります。まだ少し遅い感はありますが…

### クラス=ファイルではない

Javaの場合は、一つのファイルには一つのクラスというのが常識のようになっています。もちろんInner Classを書く場合もありますが、それだと名前とおりクラスの中に含まれたものになるので、インスタンスを生成するときに複雑だったりしますね。でもKotlinだと、純粋なクラスを一つのファイルに複数書くことができます。

なので、似たようなクラスを一つのファイルの中に集めておくことができますね。例えばDTO、DAO、Entityなど似たようなクラスが複数損際するパターンでは、一つのファイルの中にそれらを集めておいた方がパッケージの中が複雑にならないような気がします。実際、Kotlinを試しながら好みの領域の話かも知れませんが。

どちらかを選択できる自由があるというのが、必ずしも良いこととは言い切れませんが、ファイル内にクラスを複数書くかどうかはキメの問題であって実装時のコーディングスタイルに影響を与えるものではないので(今時importを直接書く人もいないだろうし…)、良い点として挙げられるのではないか、と思います。

### 拡張関数で自由に関数を追加できる

Javaのデメリットとしてよく挙げられているのが、冗長すぎる(verbose)ということです。いわゆるBoilerplateなコードを毎回書かなくてはならないというのは、生産性の面からもよくないです。Javaにこういう面があるので、さまざまなデザインパターンが発達したり、IDEでコードを自動生成してくれたり、Lombokのようにコードの量を減らしてくれるライブラリが人気だったりしますね。自分が開発に参加したフレームワークの開発の案件も、結局は冗長化するコードを減らしたいという目的によるものでした。

とにかく、Kotlinはこういう問題に対する反発ないしは反省から言語がデザインされているようにも見えます。最近のモダンな言語の特徴をコピーしただけでなく、Javaを改善させたいという強い意志が言語のデザインから感じ取れているような感覚でした。

### スタンダードライブラリがとにかく便利

拡張関数が便利な理由ともつながるようなことですが、Kotlinのスタンダードライブラリに存在する関数たちもまた同じ観点から便利といえます。例えば、すでに有名なのがScopping Functionsという[let](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/let.html)、[with](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/with.html)、[apply](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html)、[run](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/run.html)、[also](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/also.html)のような関数です。

これらはJavaだと別途ユーティリティクラスを作るか、プライペートなメソッドを定義するか、特定のクラスを継承してからオーバライドで関数を新しく定義するなどの方法で対応はできるものの、やはり手間がかかるのでやりたくはないものです。これをKotlinでは、より関数型的な方法で解決してくれます。例えばletの例を見ていきましょう。以下のようなdata classがあるとします。

```kotlin
data class Member(val region: String, val name: String)
```

このdata classのインスタンスを一つ作成します。そうすると以下のようになるでしょう。

```kotlin
var john = Member("Tokyo", "John")
```

あとで、同じくMemberのインスタンスとしてjakeという変数を追加するとします。jakeは常にjohnと同じregionである必要があります。

```kotlin
var jake = Member("Tokyo", "Jake")
```

これをJavaの考え方で、コードを整理するとしたら以下のようになります。regionを同じインスタンスを使うようにすることですね。

```kotlin
var tokyo = "Tokyo"
var john = Member(tokyo, "John")
var jake = Member(tokyo, "Jake")
```

これをletを使う場合のコードとして書くと、以下のようになります。

```kotlin
var jake = john.let {
    Member(it.region, "jake")
}
```

共通のregionを別途変数として宣言したくても、jakeのregionはjohnに指定したregionと同じ値となります。そしてある意味、こちらの方が「johnとjakeは同じregionを共有する」という意図がコードの中によく表れているのではないか、という気もします。今は簡単なフィールドを共有しているだけですが、変数の数が増えたり処理すべき項目が多くなった場合はいちいち定数を宣言するよりも、このような書き方の方がより優雅になるのではないか、と思います。そういう意味では、かなり洗練された方法を提供していますね。同じことをJavaでまねるとしたら…あまりやりたくなくなりそうです。

## これはいまいち

KotlinがJavaのさまざまな問題や不便さに注目し、それらの多くを解消してくれたのは事実ですが、果たして`全て`Javaより発展しているか、というとそうでもないような気もします。ただし、ここであげているKotlinの問題点ないしデメリットは、メリットと同様、個人的な見解なので参考までに。

### varと型

モダンな言語から接した人なら、変数の宣言が`var`だけで集結するのはメリットと言いたくなるかも知れません。実際Kotlinだけでなく、JavaScriptやC#など現代に使われる大体の言語は`var`に対応していて、あのJavaすらも10から`var`による変数の表記を導入しています。また、Pythonのようにそもそも`var`の宣言すらいらない言語があったりもしますね。`var`をつけることで変数であることが明確だという考え方から来てるのか、Javaとは違って関数も[First Class Object](https://ja.wikipedia.org/wiki/%E7%AC%AC%E4%B8%80%E7%B4%9A%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88)として扱う言語としては関数と同様に表記したいからそうしてるのか、よくわかってないのですが、どっちかというと流行りのもののようです。

こういう傾向だけを見ると、「変数は変数であることが分かればいい」というだけの話になっているようにも思えます。しかし、私はこの`var`が果たして良いものであるかという疑問を持っています。Javaに慣れすぎていて、新しいのを受け入れられてない、もしくは`var`の良さをわかってないだけかも知れませんが、とにかく「型指定で変数であることも、型もわかるからこっちの方が良くないか」と思ってしまいます。

こう思うまたの理由は、モダンなプログラミング言語の中でもTypeScriptのように、あえて型指定ができるように既存の言語を変えようとする動きもあるからです。Pythonの場合も3.6から型を宣言できるようになっています。これ自体が、「変数は変数であることが分かればいい」から「変数の型もわかった方がいい」に変わっているように見えます。ただ、問題は最初から型指定で変数を指定する方法がなく、`var`しかない言語に型指定(型注釈)が付く場合です。`var`のメリットである短くかけるというところが、型指定をすることで台無しになります。

例えば、Kotlinでの`var`だけの宣言だと以下のようになりますね。

```kotlin
var a = "this is string"
```

そして`var`に型を指定すると以下のようになります。

```kotlin
var b: string = "this is string"
```

Javaの伝統的な書き方だと以下です。こちらの方が、むしろコードは短くなるし、変数であることも明確ではないでしょうか。

```java
String a = "this is string";
```

また、厳密にいうと変数ではないところでは`var`をつけないのは当たり前なのかも知れませんが、Javaだと変数でも戻り値でも引数でも型をつけてしまうのに対して、Kotlinではこれらに`var`をつけるか型をつけるか方が省略できるかという場面がそれぞれ区別されてしまうので、これだけはJavaよりも厳格じゃないか？という気になったりもします。例えば以下のような例です。

関数の引数の場合は、型の指定が必要です。

```kotlin
fun getMember(request: ServerRequest): Mono<ServerResponse> =
        ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(repository.findById(request.pathVariable(id).toLong()).map { MemberDto(it.username, it.name) })
            .switchIfEmpty(notFound().build())
```

そして関数の戻り値は、型推論により省略可能です。

```kotlin
fun getMember(request: ServerRequest) =
        ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(repository.findById(request.pathVariable(id).toLong()).map { MemberDto(it.username, it.name) })
            .switchIfEmpty(notFound().build())
```

しかし、これはあくまで関数をSingle Expressionで書いた時の話です。明示的にreturnを書く場合は戻り値を省略するとコンパイルエラーになります。例えば以下のような場合がそうです。

```kotlin
// これはコンパイルエラー(戻り値はUnitとなってしまう)
fun getMember(request: ServerRequest) {
      return ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(repository.findById(request.pathVariable(id).toLong()).map { MemberDto(it.username, it.name) })
            .switchIfEmpty(notFound().build())
}
```

また、data classの場合はフィールドに`val`か`var`をつける必要があります。しかし、一般的なクラスを宣言する場合は要りません。

```kotlin
// data classではvalかvarが必要
data class MemberDto(val username: String, val name: String)

// classなら必要ない
class MemberEntity(username: String, name: String)
```

通常はコンパイルエラーになるので、慣れるまではKotlinのルールに従ってコードを見直すしかないのですが、Javaからの入門だとかなり混乱する部分です。(自分だけかも知れませんが…)

### 依存関係

プロジェクトにKotlinを使う場合はスタンダードライブラリを追加して使う必要があります。ただ、ここで`kotlin-stdlib`だけを追加すると、Java 1.7以降に追加された一部の機能(AutoCloseableなど)を使えなくなります。なので、Java 1.7以降の機能を使いたい場合は`kotlin-stdlib-jdk7`か`kotlin-stdlib-jdk8`を依存関係に追加する必要があります。

個人的にはOracleとGoogleの訴訟沙汰のようなことがあって、わざと著作権を避けるための独自のパッケージを作ったりしらからではないかなと思いましたが、実際はJava 9から導入されたModuleシステムに対応するための理由だそうです。なので`kotlin-stdlib-jre7`が`kotlin-stdlib-jdk7`に、`kotlin-stdlib-jre8`が`kotlin-stdlib-jdk8`に代替されたらしいですね。

とにかく、これらのスタンダードライブラリを使うには、MavenやGradleのような依存関係を管理するパッケージマネージャを使って一回だけ登録しておけばよく、そこまでめんどくさくはないことなのかも知れませんが、例えば`kotlin-stdlib-jre8`があったりするので、初めはどれを選べば良いか、どれが必要であるかを把握するのにも時間を使ってしまうことになるのでデメリットになるのではないかと思います。例えば`kotlin-stdlib-jdk7`がなくても、AutoCloseable以外のJDK 1.7の機能は使えたりするのですが、今から作るプロジェクトや既存のプロジェクトにAutoCloseableが使われるかどうかで依存関係をまた追加するかどうかを調べるのもかなり面倒くさそうです。

そしてJDK7やJDK8対応のスタンダードライブラリが別途存在するということは、今後JavaがバージョンアップしたらまたJDK 17などの新しいスタンダードライブラリが追加される可能性があるということでもあるでしょう。7(1.7)と17はよく勘違いしそうだし…あと、JDK以外の依存関係のパッケージが色々あるので(`kotlin-reflect`など)、プロジェクトの構成によってはKotlinの導入にはかなり慎重になる必要がありそうです。ある意味、KotlinがPost Javaとしてのポテンシャルは十分でありながらも、Androidアプリの作成以外にではあまり導入されてないのはこのような理由もあるのではないかという気もしています。

## 最後に

簡単にPros・Consに分けて自分が感じたKotlinに対して書いてみました。実はまだ本格的な案件で触れてみたわけでもないので、[Type aliases](https://kotlinlang.org/docs/reference/type-aliases.html)や[inline class](https://kotlinlang.org/docs/reference/inline-classes.html)のような良さそうな機能に触れてもないです。でもやはり、使えば使うほど魅力的な言語であるなと感じているところです。なので個人的な意見としては、すでにJavaを使っているところなら本格的にKotlinへの移行を考慮しても良いのでは、と思っています。Javaプログラマーなら慣れやすく、より生産性も高いながら、Javaとの100%の互換性も担保されているので…言語の完成度やJVM対応でありながらNative、JavaScriptとの連動も可能なのを見るとまさにPost Javaとして相応しい言語なのではないかと思うくらいです。(他のJVM言語には悪いですが…)

そういう意味で皆さん、今からでもKotlinやりませんか！