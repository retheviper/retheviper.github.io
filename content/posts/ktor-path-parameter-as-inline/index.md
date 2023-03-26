---
title: "Path ParameterをInline classで受け取る"
date: 2023-03-26
categories: 
  - ktor
image: "../../images/ktor.jpg"
tags:
  - kotlin
  - ktor
  - exposed
---

最近はサイドプロジェクトとしてシンプルなWebアプリケーションを作っていて、サーバサイドのフレームワークではKtorを採用しています。今まではずっとSpringを触ってきたので、たまにはこうやって違うフレームワークで何か作ってみるのも楽しいですね。

さて、そこで今回の記事のテーマとなるのが、KtorでPath Parameterを受け取りInline Classとして扱う方法についてです。高度な技術を要するものではないのですが、より型安全で便利に使えるコードが書ける方法ではないかと思い試したことの紹介となります。

## Inline Classとは

Kotlin 1.6から[Inline Class](https://kotlinlang.org/docs/inline-classes.html)という機能が追加されました。Inline Classは、型のラッパーとして使うことができる機能です。例えば、以下のようなコードがあるとします。プレイヤーがいて、そのプレイヤーごとの成績を表現したものです。

```kotlin
// プレイヤー
data class Player(val id: Int, val playerRecordId: Int)
// プレイヤーの成績
data class PlayerRecord(val id: Int, val score: Int)
```

そして以下のようなメソッドがあるとします。

```kotlin
// プレイヤーの成績を記録する
fun createPlayerRecord(val playerId: Int, val score: Int) {
    // ...
}
```

このメソッドを呼び出すときに、プレイヤーのIDとスコアを渡す必要があります。しかし、このコードでは、プレイヤーのIDとスコアの両方ともがInt型になっているため、プレイヤーのIDをスコアとして渡してしまうというようなミスが起こりうるのです。そこで、以下のように、プレイヤーのIDとスコアをラップした型を定義して、それを使うようにすると、より安全にコードを書くことができます。

```kotlin
// プレイヤーのID
@JvmInline
value class PlayerId(val value: Int)

// スコア
@JvmInline
value class Score(val value: Int)
```

二つのInline Classを定義することで、先ほどの関数は以下のように修正できます。こうなると、パラメータを間違えて指定したらコンパイラーがエラーを吐いてくれるので、より安全にコードを書くことができますね。

```kotlin
// プレイヤーの成績を記録する
fun createPlayerRecord(val playerId: PlayerId, val score: Score) {
    // ...
}
```

ここだけみると、Inline ClassがJavaのラッパークラスや普通のdata classと何が違うんだ？という疑問が湧いてくるかもしれません。また、typealiasのような既存の機能もありますね。ただ、Inline Classはコンパイル時にprimitive型の扱いでありながら、実行時にはラッパークラスのように振る舞うという特徴があります。そのため、Inline Classだと型安全を担保できつつパフォーマンスへの影響も少ないというメリットがあります。

## KtorでPath Parameterを扱う

Ktorでは、Path Parameterを受け取るためには、以下のように書く必要があります。

```kotlin
routing {
    get("/{id}") {
        val id = call.parameters["id"]?.toInt()
    }
}
```

Path Parameterを取得するために、[ApplicationCall](https://api.ktor.io/ktor-server/ktor-server-core/io.ktor.server.application/-application-call/index.html)から[Parameters](https://api.ktor.io/ktor-http/io.ktor.http/-parameters/index.html)で`{id}`に指定された値をまずStringで読み込むようになります。そして、さらに`toInt()`でIntに変換しています。これでPath ParameterをIntで受け取り、処理の中で使うことができるようになります。

## Path Parameterを受け取る処理を改善する

`call.parameters`を利用したサンプルは一瞬見てシンプルなコードなのであまり改善の余地はないかなと思うかもしれませんが、実はこのコードにはいくつか問題があります。例えば、Int変換時のエラーを考慮する必要がありますね。`toInt()`でIntに変換するときに、`null`や`"abc"`といった文字列が渡された場合には、`NumberFormatException`が発生してしまいます。また、`toInt()`でIntに変換するときに、`Int.MAX_VALUE`を超える値が渡された場合には、`NumberFormatException`が発生してしまいます。このように、Path Parameterを受け取るときには、必ず`null`チェックや`NumberFormatException`のチェックを行う必要があります。

また、`ApplicationCall`は`get()`や`post()`のような関数の中でしか呼び出せないです。ということは、エンドポイントごとに同じような処理（エラーハンドリングなど）を書く必要があるということです。ルータ内で何回も同じような処理があるのは望ましくないので、このコードを共通化したいですね。なので、以下のように`ApplicationCall`に拡張関数を定義して、Path Parameterを取得する処理を共通化すると良いはずです。

```kotlin
fun ApplicationCall.getIdFromPathParameter(name: String): Int {
    val parameter = parameters[name] ?: throw IllegalArgumentException("id is required")
    val id = parameter.toIntOrNull() ?: throw IllegalArgumentException("id must be integer")
    return idInt
}
```

このようにすると、以下のようにエラーハンドリングを共通化できます。try-catchで例外を処理していますが、ここは必要に応じて[Status Pages](https://ktor.io/docs/status-pages.html)によるエラーハンドリングを追加すると良いでしょう。

```kotlin
routing {
   get("/{id}") {
       try {
           val id = call.getIdFromPathParameter("id")
       } catch (e: IllegalArgumentException) {
           call.respond(HttpStatusCode.BadRequest, e.message)
       }
   }
}
```

## Path ParameterをInline Classでラップする

さて、Path ParameterからInt型で取得する処理を共通化できたので、次はPath ParameterをInline Classでラップすることで、より安全にコードを書くことを考えてみましょう。まず、もっとも簡単な方法は以下のように拡張関数が返す値をInline Classにすることです。先ほどの関数だと、以下のようになります。

```kotlin
routing {
   get("/{playerId}") {
        val id = PlayerId(call.getIdFromPathParameter("playerId"))
   }
}
```

ただ、Inline Classもとりあえずコード上ではクラスの扱いなので、ジェネリックを使うこともできます。なので、先ほどの関数をジェネリックを使ったものにする方法も考えられます。イメージ的には、以下のようになります。

```kotlin
routing {
   get("/{playerId}") {
        val id = call.getIdFromPathParameter<PlayerId>("playerId")
   }
}
```

このようにすると、`getIdFromPathParameter()`の戻り値を`PlayerId`に変換する処理を`getIdFromPathParameter()`の中で行うことができます。また、ジェネリックであるため、使える型を特定のInterfaceに制限し、IDに関するInline Classがそれを実装するという形にしたらより安全なコードになるでしょう。なので、まずは以下のようにID系の共通のInterfaceを定義します。

```kotlin
// ID系の共通のInterface
interface Id(val value: Int)

// プレイヤーのID
@JvmInline
value class PlayerId(val value: Int) : Id

// 監督のID
@JvmInline
value class DirectorId(val value: Int) : Id
```

そして、以下のように`getIdFromPathParameter()`をジェネリックにして、`Id`を実装したクラスのみを受け取れるようにします。

```kotlin
// IDを取得する拡張関数
inline fun <reified T: Id> ApplicationCall.getIdFromPathParameter(name: String): T {
    val parameter = parameters[name] ?: throw IllegalArgumentException("id is required")
    val id = id.toIntOrNull() ?: throw IllegalArgumentException("id must be integer")
    return T::class.java.getDeclaredConstructor(Int::class.java).apply { isAccessible = true }.newInstance(id)
}
```

修正は簡単で、指定された`Id`タイプのインスタンスを作成して、Path Parameterから取得したInt値をラップして返すだけですね。これで`Id`を実装するInline Classのみ対応するという制限もかけながら、型の安全性も確保できるようになります。

ただ一つ、Path Parameterとしての変数名をInline Classの方にcompanion objectとして持たせて共通化できるといいのですが、残念ながらそれは難しいようです。interfaceのcompanion objectはoverrideできなく、Inline Classはabstract classを実装することができないからです。なので、他の方法でPath Parameter名の指定ができるようにすれば拡張関数の引数を減らしよりシンプルなものになるという改善の余地がまだありそうです。

## 最後に

久々にKtorを触り、ブログの記事にしてみましたが、どうだったでしょうか。ずっとRest APIの実装ばかりしていたので、プライベートで何か新しいチャレンジをしてみないと、なかなか発見がない状態となっているのではないかという気がしています。ブログを始めてかれこれ5年目になり、まだ勉強不足だと感じるところが多いと感じつつもなかなかそれを言語化することは簡単ではないとも感じています。

ブログの更新の頻度が減ることになっても、次からはより良い記事になるように頑張りたいと思います。では、また！
