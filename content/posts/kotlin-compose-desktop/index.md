---
title: "Kotlinでデスクトップアプリを作ってみた"
date: 2022-09-09
categories: 
  - kotlin
image: "../../images/kotlin.jpg"
tags:
  - kotlin
  - compose
  - gui
---

バックエンドの開発をしていると、テストの自動化では対応できない場合もありますね。理想的なシナリオとしては、ユニットテストから全てのシナリオを想定したインテグレーションテストまでを全部作成でき、開発・企画に属するものがそれらを理解しきっていることだろうとは思いますが、現実ではなかなか難しいものです。特に、サービスが成長していきながら技術的負債を解消しようとしたり、新しい機能を足したり、昔は対応できなかった改修が必要となったり、運用上のイレギュラー対応が必要となったりなどで最初の仕様は変わり続けていき、改修を行うエンジニアや運用する側でも今の状態がどうで、どう変わるべきかを判断するのは難しくなりがちなのだから、と自分は理解しています。

なので、少なくとも今のアプリがちゃんと想定通りに動くかどうかを人の手で検証する必要が出てくるケースも十分にあり得るものです。そしてそうなった場合はどうやってテストを行うかを考える必要もありますね。テストの方法も色々あり、小さい機能単位でユニットテストを行い、最終的にはインテグレーションテストやシナリオテストまで上がっていくと良いはずですが、その全部を自動化するのが難しいケースもあるかと思います。例えばテストするためのデータのパターンを色々と用意する必要があったり、エンジニアが完全に仕様を把握してなかったりなどの場合もありますね。なので、人の手によるテスト（モンキーテスト的な）が必要となる場面も存在すると思います。

今回はその「人の手によるテスト」を手伝うために、テストツールを作った話です。自分の扱っているシステムはマイクロサービスの一つであり、業務仕様が複雑でさまざまなパターンで機能をテストする必要がありました。なのでエンジニアとしては実装を進めながら、同時に業務仕様に詳しい人にさまざまなパターンのデータを使ってテストができるツールを作ることになったわけです。

## 目標と設計、技術選定

実は以前から、リポジトリにはすでにテストツールが存在していました。しかし調べたところ、作られて2年以上放置されていて、Ruby on railsという自分が全く触れたことも（興味を持ったことも）ないフレームワークで作られているという問題がありました。これだと、自分がRubyを勉強して既存のツールを改修するという手もあったかも知れませんが、以下の理由から一から作り直そうと思いました。

1. KotlinエンジニアがRubyアプリをメンテするのは良くない
2. ドキュメント化が進んでなく、使い方が不便

そう決めてからは、テストを行いたい側（企画）からの要請を受け、ツールに要求される仕様としての機能をまとめることに。テストが行えるツールという確実な目標があったので、仕様は極めて単純です。要求事項としてツールに揃うべき機能は以下のようなものでした。

- テストデータのファイルを読み込ませる
- バックエンドのアプリのAPIを呼び出す
- APIの実行結果をファイルに書き込む

テストツールとしては上記に挙げている要求事項を満たしているならテストツールとしては合格というわけです。しかし、実際のテストを行いたい側がまずエンジニアではなく、今後もエンジニアではない人がツールを触る可能性があります。そこまでを考慮して、以下の追加的な目標を立てました。

- 環境構築をしなくても使えるようにする
- 手順書がなくても使えるくらい簡単なものに仕上げる

ここまで決めたら、次に要求されている機能の細部を掘り下げていきます。設計書を書くほどでもないですが、土台となる設計のようなものです。

- データの読み込みと書き込み
  - ツールを使える人はSQLが使える=テーブル（表）が読める
  - テーブルの形でデータの入出力ができた方がわかりやすい
  - テストデータはCSVで読み込む
  - API実行結果もCSVに書き込む
- APIコールができる
  - HTTPクライアントでGET・POSTする
  - APIコールにはトークンが必要
    - トークンはセキュリティ問題でソースコードに埋め込むのはNG
    - しかし毎回入力するのはめんどくさい
    - アプリを実行して最初はトークンを入力し、次回からはそのトークンを使い回すようにする
- 本番以外の環境が対象
  - 複数の環境があるのでどれかを選択できるようにする
  - これも毎回入力はめんどくさい
  - 最初に一回だけ選択できるようにしたい

そして自分の立てた目標を、上記の要求事項を達成できるかどうかを考えながら振り返ってみます。環境構築をしなくても使えるなら、実行可能な一つのバイナリとして提供した方が良いでしょう。また、使い方が簡単な方だと、やはりGUIを含めた方が良いですね。特にGUIを採用したら要求事項に対してもかなり良い感じで機能を完成できると思いました。

例えばファイルの読み込みや書き込みにはパスの指定が必要で、トークンと環境の選択も入力が必要な項目で、CLIだとやはり不便です。エンジニアではない人が触るとしたら尚更ですね。Windowsユーザなら、コマンドラインも考慮しなければならないかも知れません。その反面、GUIだとファイルパスならダイアログ、トークンの入力もテキストポックス、環境の選択ならプールダウンメニューで対応できます。なので、「バイナリの実行で起動できるGUIのアプリを作る」という結論を出しています。

## Compose for Desktop

テストツールの仕様と技術的な要件が決まったら次は技術選定になりますがね。まずどの言語を使うかについてですが、自分以外でも同じチーム、つまりKotlinエンジニアがこれからもメンテを行うことになる可能性が高いのでKotlinにしました。Kotlinを使うことで、機能の実現で必要なライブラリの選定も楽になりますね。すでにテスト対象のバックエンドアプリで使っているHTTPクライアントがあるので、一部のコードはそのまま移植しても良いはずです。また、同じライブラリを使うことでメンテもより簡単になるでしょう。

あとはGUIですが、今回は[Compose for Desktop](https://www.jetbrains.com/ja-jp/lp/compose-desktop/)を使うことにしました。KotlinはJavaと互換性があるので、当然[Swing](https://ja.wikipedia.org/wiki/Swing)や[JavaFX](https://openjfx.io/)などJavaのGUIツールキットをそのまま使うという選択肢もあります。他にも[TornadoFX](https://github.com/edvin/tornadofx)という選択肢があったりもしますが、今回あえてComposeを選んだのはいくつかの理由があります。

まずは個人的にモバイルに興味があって以前から興味を持っていたので、今回本格的にこれでアプリを作ってみたいという願望もありましたが、今後もKotlinエンジニアの手でメンテが行われるとすると、やはりモバイルの経験があるか、少なくとも興味を持つ方が多いだろうという点です。Composeはまだ正式リリースされて1年ほどしか経ってない新しいものですが、最近流行っているいわゆる「宣言型」のフレームワークなので、少なくともAndroidアプリの開発ではメインストリームになる可能性が高いだろうという判断からでした。

また、Composeはモバイルのみでなく、そもそもマルチプラットフォーム向けに開発されたものなので、Windows/Mac/Linuxの環境を問わず実行可能なバイナリをビルドできるという点でも魅力的だったです。これならテスターがどんなOSを使っていても同じ感覚でツールを使えて、

ただ、やはり今まであまり接したことのない技術なので勉強はもちろん試行錯誤などもあったので、テストツールを作りながらこれは覚えておいた方が良いなと思ったところをいくつか挙げてみようと思います。

### 状態管理

[SwiftUIのポスト](../swift-ui-first-impression-2/)の時も触れた状態管理ですが、Composeでも同じくGUIを扱うことになるので、状態管理が大事となります。今回はアプリとしての画面がひとつしかないので、複数の画面にまたがって状態を管理する必要はないかなと思いましたが、それでもやはり処理を行うためにはアプリ全体で共有する状態として管理が必要なものがいくつかありました。

ただ、上記ポストでも述べた通り、SwiftUIとComposeとは状態管理の方式が少し違います。SwiftUIでは状態がどこで使われるかによって明確に使われるアノテーションやクラスなどが変わっていたなら、Composeでは大体[remember()](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary?hl=ja#remember(kotlin.Function0))と[MutableState<T>](https://developer.android.com/reference/kotlin/androidx/compose/runtime/MutableState?hl=ja)の組み合わせで事足りることになります。画面の構成要素の最小単位をComposeではWidgetでも使い方が同じだということは、SwiftUIと比べると定義するのは簡単ですが、使い方には少し注意が必要だなという感覚でした。

まず、Composeでの状態は、以下のような三つの方法で定義することができます。

```kotlin
// Delegateで定義する
var isOn: Boolean by remember { mutableStateOf(false) }
// 直接値を書き換えできる
isOn = false

// 分解宣言で定義する
val (isOff: Boolean, setIsOff: (Boolean) -> Unit) = remember { mutableStateOf(true) }
// 参照と更新が分離される
if (isOff) {
    setIsOff(!isOff) // toggle
}

// MubtableState<T>として扱う
val isNotOff: MutableState<Boolean> = remember { mutableStateOf(false) }
// ラッパーになっているので、値を更新するためにはvalueにアクセスする必要がある
isNotOff.value = !isNotOff.value
```

ここでDelegateで`var`として定義した場合は最も使いやすくなりますが、Intellij上ではコンパイルエラーになりがちです。なぜかというと、Delegateを使うためには[androidx.compose.runtime.setValue](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#(androidx.compose.runtime.MutableState).setValue(kotlin.Any,kotlin.reflect.KProperty,kotlin.Any))と[androidx.compose.runtime.getValue](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#(androidx.compose.runtime.State).getValue(kotlin.Any,kotlin.reflect.KProperty))をimportする必要がありますが、これが自動で行われないからです。最初このエラーの理由がわからなかったり、忙しい場合にいちいちimport文を書いていくが面倒だったりでかなり使わなくなるケースも多いかなという気がします。ただこれは、まだIntellijでのCompose対応が完璧ではないのが原因なので、これはいずれ解消されると期待できるでしょう。

分解宣言で値の参照と更新を別々で使うのは、どこで使うか悩ましいケースもあるかなと思いますが、Composeの一部Widgetに状態を渡す場合に使われる場面があります。代表的なものが[TextField](https://developer.android.com/reference/kotlin/androidx/compose/material/package-summary#TextField(kotlin.String,kotlin.Function1,androidx.compose.ui.Modifier,kotlin.Boolean,kotlin.Boolean,androidx.compose.ui.text.TextStyle,kotlin.Function0,kotlin.Function0,kotlin.Function0,kotlin.Function0,kotlin.Boolean,androidx.compose.ui.text.input.VisualTransformation,androidx.compose.foundation.text.KeyboardOptions,androidx.compose.foundation.text.KeyboardActions,kotlin.Boolean,kotlin.Int,androidx.compose.foundation.interaction.MutableInteractionSource,androidx.compose.ui.graphics.Shape,androidx.compose.material.TextFieldColors))で、これはコードを見るとすぐにその目的がわかります。実際のコードで、以下のように使われます。

```kotlin
val (text: String, updateText: (String) -> Unit) = remember { mutableStateOf("") }
TextField(
    onValueChange = setValue, // TextFieldに文字を入力するとその値でtextを更新する
    value = text // textの値をTextFieldに表示する
)
```

最後に`MutableState<T>`として定義するケースですが、直接的な値の更新ができないので最も使い方としては不便なのですが、実際は最も多く使われるのではないかと思います。なぜかというと、アプリ全体で状態を共有するなど複数のWidgetをまたがって使う場合は、以下のようにclassの中にフィールドとして`MutableState<T>`を定義することになるからです。

```kotlin
// アプリ全体で共有するためにクラスに状態を定義する
class AppState(
    val isOn: MutableState<Boolean>
)
```

これはもちろん別途[getter/setter](https://kotlinlang.org/docs/properties.html#getters-and-setters)をclassに定義しておくと、中のvalueにアクセスしなくても直接プロパティにアクセスする感覚で使えます。イメージ的には以下のようなものですね。これだと状態として管理したい項目が増えれば増えるほどコードの量が増えてしまう面倒さがあるというのが短所かなと思います。

```kotlin
class AppState(
    private val _isOn: MutableState<Boolean>
) {
    var isOn: Boolean
        get() = _isOn.value
        set(value) { _isOn.value = value }
}
```

このように、Composeでの状態には定義する方法が色々あり、それぞれの特徴があるわけなのでどの場面で使うかによって適切な定義の方法を考えるのが何よりも大事だという印象です。

### Swing/AWT

Compose for Desktopの特徴の一つは、Swingや[AWT](https://en.wikipedia.org/wiki/Abstract_Window_Toolkit)に対する互換性があるという点です。最初は[Macのトレイ、メニューバー、通知](https://github.com/JetBrains/compose-jb/blob/master/tutorials/Tray_Notifications_MenuBar_new/README.md)にも対応していたので基本的な機能は全て揃っているのではないかと思いましたが、実はそうでもなく、一部の機能はSwingやAWTの機能を借りて実装することになるケースもありました。実際、私の作ったテストツールでも一部SwingとAWTの機能を使っているところがあります。

例えばファイル選択機能がそうです。CSVを読み込むためにファイル選択のダイアログを表示したかったのですが、ComposeのWigdetではまだ対応できてないので、やむを得なくAWTの[FileDialog](https://docs.oracle.com/javase/jp/8/docs/api/java/awt/FileDialog.html)を使う必要がありました。以下がその実装の例です。

```kotlin
 // 選択したファイル名を状態として保持する
var fileName by remember { mutableStateOf("") }
// AWTのファイル選択ダイアログを使う
FileDialog(ComposeWindow()).apply {
        // 選択できるのはCSVのみにしたい
        setFilenameFilter { _, name -> name.endsWith(".csv", ignoreCase = true) }
        isVisible = true
        // ファイルが選択された場合は状態を更新する
        if (!file.isNullOrBlank()) {
            fileName = file
        }
    }
```

しかし、これでも十分ではない場合もありました。フォルダのみを選択できるようにしたい場合に`FileDialog`はあまり良い選択ではなかったのです。名前からそうですが、あくまでファイルの選択を想定したものであるため、フォルダのみを選択できるようにはできなかったのです。なので、フォルダのみを選択できるようにするためには、Swingの力も借りる必要があります。その場合は、以下のように実装ができます。

```kotlin
// 選択したフォルダのパスを状態として保持する
var selectedPath by remember { mutableStateOf("") }
// Swingのファイル選択ダイアログをディレクトリのみ選択できるように設定して定義する
val fileChooser = JFileChooser().apply {
        dialogTitle = "Choose Directory"
        fileSelectionMode = JFileChooser.DIRECTORIES_ONLY
    }
// ダイアログを表示する
if (fileChooser.showOpenDialog(ComposeWindow()) == JFileChooser.APPROVE_OPTION) {
    // ダイアログで選択したパスが状態として保持しているパスと違う場合、選択したディレクトリの絶対パスを状態として更新する
    val path = fileChooser.selectedFile.absolutePath
    if (selectedPath != path) {
        selectedPath = path
    }
}
```

今回はこの二つユースケースのみSwingやAWTが登場していませんでしたが、どんなアプリを実装するかによって他のAPIも色々と使う必要性が出てくるかも知れないという良い一例になっている気がします。まだComposeはリリースされて1年ほどしか経っていないので、今後のバージョンアップでより多彩なWidgetが追加されることに期待ですね。

### ビルド

Composeを選んだ理由の一つのバイナリのビルドができるという点ですが、これはかなり満足度が高かったです。`gradle`を使って、コマンドひとつで実行可能なバイナリが生成されます。Macでビルドして見ると、他のアプリと同じくパッケージが生成されます。中を見ると、実行に必要なJREと依存関係のJarが含まれていて、ネイティブではなくJVM上で起動される構造になっていました。

バイナリをビルドするときのオプションには色々なものがあり、OSの種類(Windows, Mac, Linux)によって違うアイコンを使ったり、基本的には含まれないモジュールを含むように指定したりすることができました。以下が実際のビルド時のオプションのサンプルです。

```kotlin
compose.desktop {
    application {
        mainClass = "com.testtool.MainKt" // 実行時のメインクラスを指定
        nativeDistributions {
            packageName = "Test Tool"
            packageVersion = "1.0.0"
            modules("java.sql", "java.naming") // デフォルトでは含まれないパッケージを追加
            macOS {
                iconFile.set(project.file("misc/appicon/icon-mac.icns"))
            }
            windows {
                iconFile.set(project.file("misc/appicon/icon-win.ico"))
            }
        }
    }
}
```

ただ、ビルド時は注意が必要です。ビルドするとき、Composeでは内部的に[jpackage](https://docs.oracle.com/javase/jp/14/docs/specs/man/jpackage.html)を使うので、まずJava 15以上が必要となります。また、CPUのアーキテクチャによって違うJDKをインストールするようになっているため、ビルドするマシンと違うのアーキテクチャのCPUを使っているマシンをターゲットにすることはできません。

つまり、自分の使用のMacだとApple Silicon用のバイナリが生成され、IntelチップのMacだとx64用のバイナリが生成されるということです。実際ComposeでApp Storeにアプリを提出した人もいるらしいのですが、Rosettaで起動できるということでIntelチップのMacを使ってビルドしているとのことでした。Universal Binaryを作りたい場合は、JDKそのものがまずUniversal Binaryとして提供されることを待つしかなさそうです。

## 最後に

今回はデスクトップアプリの中でもかなり制限された機能と単純なロジックしかないシンプルなものを作ったので、もしこれからまたComposeを使ってさまざまな機能を持つように実装するとしたら(マルチウィンドウやダークモード対応、ナビゲーションなど)また色々と発見があるかも知れない気がしています。個人的にはかなりためになる経験で、思ったより実装もそこまで難しくなかったので、ツールの機能を拡張するか新しいツールを作ってみるチャンスがあるとしたら再度Composeを使ってみたいなと思いました。

まだリリースされてからそう長くもなく、足りない機能や情報も多かったり競合のフレームワークが色々とあるので未来はどうなるかわからないものですが、自分のようにKotlinをメインとしているエンジニアで、GUIに興味がある方なら一度はComposeを使って見ることをお勧めしたいですね。

では、また！
