---
title: "KotlinのString実装を覗く〜whitespace編〜"
date: 2021-05-08
categories: 
  - kotlin
photos:
  - /assets/images/sideimage/kotlin_logo.jpg
tags:
  - kotlin
  - java
  - string
---

Kotlin(JVM)は、コンパイルした結果がJVMのバイトコードになります。Javaで書かれたライブラリをそのままKotlinで利用できるのはそのためですね。これはKotliのライブラリに対しても同じなので、スタンダードライブラリを覗くとJavaの機能に依存しているところも少なくないです。

ただ、KotlinがコンパイルしたらJVMのバイトコードになるということは、単純にKotlinが「書き方が違うJava」であるという意味ではないです。これはKotlinがJavaと言語スペックが違うという点もありますが、JVMだけでなく、JavaScriptやネイティブコードにコンパイルされることも想定してデザインされているので、スタンダードライブラリスタンダードライブラリはプラットフォームに合わせて違う実装になっています。そしてJVMだとしてもJavaのAPIをそのまま利用しているわけではありません。

Kotlinのこういう構造は、内部のソースコードを見るとはっきりします。スタンダードライブラリの一部メソッドやクラスには`expect`と`actual`というキーワードが使われていますが、これらはJavaのinheritanceと似ているようなものです。Javaでは`interface`で定義したメソッドを、それを継承したクラスで`override`で実装して使うことになりますね。同じく、Kotlinでは`expect`として定義された機能をプラットフォームに合わせて`actual`で実装しているわけです。

また、KotlinのスタンダードライブラリはJavaと一見同じようなものに見えるとしても、実際は違うケースもあります。`actual`によって実装されたコードがKotlinに合わせて、書かれているからですね。なので、Kotlinのスタンダードライブラリに対しては「Javaと同じだろう」という認識をするのは危険な可能性もあります。

今回はそういうことで、文字列のwhitespaceに関しての機能を、スタンダードライブラリのソースコードを中心に見ていきたいと思います。

## whitespaceの判定

とある文字列が意味のある(有効な)データであるかどうかを判定する方法の一つは、その文字列がただの空白であるかどうかを判定することです。つまり、そもそもなんのデータもなかったり、whitespaceだけでないかというチェックをするということですね。

こういう場合の判定はKotlinのスタンダードライブラリで簡単に行うことができます。KotlinではStringのメソッドとして基本的に以下の二つを提供しています。

- `isEmpty()`
- `isBlank()`

Java 11以降でもこれらと同名のメソッドが存在しているので、一見そのままの感覚で良さそうにも見えます。しかし、Kotlinではこれらのメソッドがまず`kotlin.text.Strings`から呼ばれるものとされています。JavaのAPIをそのまま使っているわけではないので、処理も違う可能性があるという推測ができますね。

ここで前者の場合、文字列が単純になんのデータも持ってないかどうかに対する判定をおこないます。実際のソースコードを見ると、文字列の長さだけをチェックしているのを確認できます。

ちなみにJavaでは`String`は`CharSequence`を継承していますが、Kotlinとしてもライブラリは違えどそういう継承関係は一緒です。なので、Kotlinでは`String`のメンバーでありながらも`CharSequence`の関数として書かれています。

```kotlin
public inline fun CharSequence.isEmpty(): Boolean = length == 0
```

後者の場合は、文字列にwhitespaceまで含めているのかを判定します。以下のコードを見ると、何をやっているかが明確でしょう。

```kotlin
public actual fun CharSequence.isBlank(): Boolean = length == 0 || indices.all { this[it].isWhitespace() }
```

`isBlank()`で呼び出している`isWhitespace()`は、以下のような実装となっています。

```kotlin
public actual fun Char.isWhitespace(): Boolean = Character.isWhitespace(this) || Character.isSpaceChar(this)
```

Kotlinの`Char.isWhitespace()`は最終的に`Character.isWhitespace()`と`Character.isSpaceChar()`を使って判定することになります。前者の場合はUnicodeのwhitespaceに当てはまるか、後者の場合はUnicodeのspace(改行コードなど)に当てはまるかを判定するJavaのAPIです。ここでわかるように、特集なケースでなければなるべく`isEmpty()`を使った方が文字列をチェックする時に良いでしょう。

## whitespaceの削除

文字列が単純に意味のあるデータを持っているかどうかを判定するには、前述通り`isEmpty()`を使うと良いですが、文字列にwhitespaceだけでなく、意味のあるデータも混在する場合もありますね。こういう時は前後のwhitespaceを取り除きたくなります。

Javaでは、文字列の前後のwhitespaceを消去する方法として`trim()`と`strip()`がありました。前者は昔ながらのもので、全角のwhitespaceを検知できなく、性能の問題もあるのでJava 11以降は後者を使うことが推奨されています。

ただ、Kotlinの場合は少し都合が違います。Kotlinでは基本的に`trim()`だけを使うことになります。まずは`trim()`の実装をみていきましょう。

```kotlin
public inline fun String.trim(): String = (this as CharSequence).trim().toString()
```

まず`String`としては、`CharSequence`にアップキャストしてその`trim()`を呼び出すことにしています。そのあとは単純に`toString()`で返すだけですね。

続いて、`String`で呼ばれている`CharSequence`側の`trim()`をみていきましょう。

```kotlin
public fun CharSequence.trim(): CharSequence = trim(Char::isWhitespace)
```

ここでは、オーバロードした他の`trim()`に`isWhitespace()`をメソッドレファレンスとして渡しているのがわかります。`Boolean`が戻り値なので、引数は`Predicate`であると推測できますね。続けて、こちらで呼び出している`trin(predicate)`の方を確認します。こちらのコードは以下の通りです。

```kotlin
public inline fun CharSequence.trim(predicate: (Char) -> Boolean): CharSequence {
    var startIndex = 0
    var endIndex = length - 1
    var startFound = false

    while (startIndex <= endIndex) {
        val index = if (!startFound) startIndex else endIndex
        val match = predicate(this[index])

        if (!startFound) {
            if (!match)
                startFound = true
            else
                startIndex += 1
        } else {
            if (!match)
                break
            else
                endIndex -= 1
        }
    }

    return subSequence(startIndex, endIndex + 1)
}
```

ここまできてやっと実際の処理がでました。`CharSequence`をループしながら左(start)から右の方にwhitespaceを探し、初めてwhitespaceでない文字を見つけたら右(end)から左の方にループしながら繰り返すという処理ですね。意外と単純ですが、効率的な処理です。

そしてその処理での判断基準が`isWhitespace()`になっているわけですが、先に確認している通りこちらは最終的にJavaのAPIを呼ぶことになっているので、`trim()`でも十分Unicodeに定義されてあるwhitespaceやspaceまでを削除してくれると推論できます。なので、Javaとは違ってあえて`strip()`を使う必要はなさそうです。

また、`trim()`は文字列の前後のwhitespaceを削除しますが、場合によっては前方のみ、後方のみで分けて使いたい場合もあるかも知れません。その時は、以下のようなことができます。

```kotlin
val string = "  string  "

// 左のみtrim
println(string.trimStart()) // "string  "

// 右のみtrim
println(string.trimEnd()) // "  string"
```

これらのメソッドは引数として`Predicate`を渡すこともできるので、他の条件を自前で書く必要がある場合にはそちらを使えますね。

他にも、whitespaceではない、前後の特定の文字(prefix、suffix)を削除してたい場合は以下のメソッドが提供されています。

```kotlin
val string = "--hello--"

// prefixのみ削除
println(string.removePrefix("--")) // "hello--"

// suffixのみ削除
println(string.removeSuffix("--")) // "--hello"

// 前後を削除
println(string.removeSurrounding("--")) // "hello"
```

### 改行を削除

改行が文字列の前後に入っていれば`trim()`で十分ですが、文字列の中に改行が含まれていて、それを変えたい場合もありますね。例えばJSONをログに一行で出力したいだったり、以下のようなMulitiline Stringを一行にまとめたい場合です。

```kotlin
val string = """
    Hello
    World
"""
```

Intellijだと自動的に`trimIndent()`をつけてくれますが、これはあくまでインデントに関与するものであって、中の改行まではtrimしてくれないです。こういう場合は、KotlinでもJavaでも対応するメソッドは特にないので、自分で処理を書くしかないですね。例えば、以下のようなコードが使えるでしょう。

```kotlin
fun String.stripLine() = replace(System.lineSeparator(), " ")
```

ただ、Javaでも13から[Text Block](https://openjdk.java.net/jeps/355)が導入されているので、今後はJavaのAPIの方で上記のようなメソッドが追加されることを期待できるかも知れません。

## 最後に

最初に`expect`と`actual`の話をしましたが、これらのキーワードは[Kotlin Multiplatform](https://kotlinlang.org/docs/multiplatform.html)でもっとも重要な概念です。Kotlinで書いたコードをさまざまなプラットフォームで共有できるようにすることを目的としているので、こういう構造になっているのは自然ですね。なので、Kotlin/JVMだけでなく、他のことを試したい方にはとりあえず理解しておく必要があるキーワードだと思います。ちょっと独特なだけで、実体は単純なので、理解は簡単でしょう。

また、KotlinのStringに関しては、[JetBrains公式YouTubeチャンネルの動画](https://youtu.be/n4WBip822A8)で簡単に説明しているので、Kotlinで開発をしている方なら一度は参考にした方が良いかも知れません。

他に、`strip()`をあえて使う必要はないと言いましたが、実際Kotlinの最新バージョンである1.5.0でも`strip()`は`deprecated`になっていて、以下のようなコメントがついているので、次のバージョンで正式対応するまでは使わない方が良いですね。

> 'strip(): String!' is deprecated. This member is not fully supported by Kotlin compiler, so it may be absent or have different signature in next major version

こういうケースでもわかるように、KotlinがJavaと100%互換性があると言い切れない側面もあるのではと思います。なので、JavaからKotlinに移行した場合(実際のコードであれ、開発者自身のスキルであれ)には、一度注意深くスタンダードライブラリの説明を読む必要があるかも知れません。

では、また！
