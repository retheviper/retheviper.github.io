---
title: "Kotlinだけでファイルサーバを作ってみた"
date: 2022-10-10
categories: 
  - kotlin
image: "../../images/kotlin.jpg"
tags:
  - kotlin
  - compose
  - gui
---

世の中にはさまざまなプログラミング言語があり、それぞれの特徴も明確で、言語ごとにできる・できないことも違うケースが多いですね。企業ならエンジニアの採用や費用など現実的な観点から技術選定をするので、プロジェクトにおいてどの言語を使うかは明確かつ一般的な基準があるかと思います。しかし、個人のレベルだとチームでの作業を考慮すべきでもなく、その人の好みや慣れというものから言語を選ぶ傾向があるのではないかと思います。なので、割とマイナーな言語やフレームワークを使うケースもあるでしょう。

自分がまさにそうであって、個人的に使うために実装するアプリや自動化のスクリプトなどは、なるべくKotlinやPythonで作成しています。特にKotlinの場合、仕事でも使っているので最も慣れているからでもありますが、さまざまなフレームワークや言語自体の特徴によりサーバサイドというジャンルやJVMという環境に限らずいろいろなことにチャレンジできるのが魅力的で好きです。

というわけで、今回もプライベートでちょっと変わった形でウェブアプリを一つ作ってみた、という話です。どこが変わっているかというと、表題にも書いてある通りですが、「Kotlin」だけでファイルサーバのアプリを実装した話となります。

## 背景

まず、どんなアプリをなぜ作ったかから述べないとですね。私の実家には、以前から使っていたWindowsのパソコンがあります。組み立てたのはおよそ8年ほど前のことで、最近は自分が実家に帰ることも少ないのであまり使われてないです。ただ、今のPC(Mac)からファイルを送ったりもらったりして使うことがあります。

ここでファイルのやりとりには、今までMicrosoft社の[OneDrive](https://www.microsoft.com/ja-jp/microsoft-365/onedrive/online-cloud-storage)を使っていました。片方で必要なファイルをOneDriveのフォルダにコピーしておくと、そのファイルがクラウドにアップロードされ、自動的に同期される方式ですね。これでも問題は全然なく安定的に使ってはいましたが、ふと思うとクラウドを経由するというステップが無駄だという気がしました。また、同期の前後でファイルをコピーしたり移動したりすることもめんどい作業になっています。

ここで、自分でインターネット越しでファイルのやり取りができるアプリを作ってみたらどうかと思ったわけです。すでに自分が思っているような機能を提供している何らかのサービスはあるかもしれませんが、そこまで複雑なものでもないので、数日で作れるような気がしましたのでとりあえずチャレンジしてみることにしました。（SFTPというオプションもありましたが、GUIで楽にしたかったので却下です）

## 要件

さて、作りたいものがあったらやることは決まっています。いつものことですが、アプリを作る前に簡単に要件を決めておきます。まず、機能的には以下のようなことができれば良いかなと思いました。

- サーバアプリを起動すると、クライアントからサーバのストレージにアクセスできる
- サーバのパスを指定したらその中身（ファイルとフォルダ）が見える
- フォルダをクリックすると表示中のパスが変わる
- ファイルをクリックするとダウンロードできる
- パスにファイルをアップロードできる

機能が決まったらそれを実現するための技術の番ですね。ここでは、何よりもKotlinで全て解決したい！という考えで、技術選定は全てKotlinを中心にしています。

まずFrontendでは、ちょうどこないだ[Compose for Desktop](https://www.jetbrains.com/lp/compose-desktop/)で簡単なアプリを作ったとこのブログに書いたことがありましたが、[Compose for Web](https://compose-web.ui.pages.jetbrains.team/)というものもあったので、今回はこれを使ってみるとどうかなと思いました。これに関しては言語を統一したいという理由が最も大きいのですが、他にはFrontendの経験や知識があんまりないので少しでも触れてみた技術を使いたかったという理由もあります。

Backendのフレームワークは[Ktor](https://ktor.io/)にすることとしました。普段はSpringをやっているのでこちらの場合もあまり本格的な経験があるわけではありませんが、以前触れてみた感覚だとアプリの起動がはやく、実装も簡単だったので採用。また同じく、最も大きい理由はKotlin用ということです。

大きくはこの二つで、他にも当然色々とライブラリなどが必要となるわけですが、ここは実装を進めながら必要なものがKotlin製かJetBrainsのものかを基準に選んで実装することにしました。もしくは実装において参考となるだろう公式のドキュメントに出てくるものを採用するという方針です。

## Frontend

Frontendでは、先に述べた通りCompose for Webを使いました。やはり初めてということもあったのですが、まだ新しい技術だったり、そもそも自分がFrontendに対してあまりわかってないということもあったので最も工数がかかった部分です。ここについては、肌で感じたことを良かった点・思ったことと違った点・問題だった点という三つの軸で分けて述べていきたいと思います。

### 良かった点

良かった点としては、やはりComposeでデスクトップアプリを作ってみた経験を活かした実装ができたというところです。Composeでは[`remember`と`MutableState<T>`](https://developer.android.com/jetpack/compose/state#state-in-composables)を組み合わせて状態を管理したり、[@Composable](https://developer.android.com/reference/kotlin/androidx/compose/runtime/Composable)をつけた関数の単位で画面の構成要素を分けて実装することができますが、ここでもそれは同じでした。

なので、「指定したパスをブラウズ」する機能を実装した時、「一つ前のパスに戻る機能を追加したいな」と思ったときはそのパスを保持するために状態にパスを持たせたり、サーバから取得したパスの中身のオブジェクト（ファイルやフォルダなど）を画面に描画するためのコンポーネントを一つの`@Composable`関数として定義して使ったりなどが思ったよりも簡単にできたわけです。

他にもKotlinなのでCoroutineが簡単に使えたり、サーバサイドと同じリポジトリにソースコードを作成できるというところも良いところでした。特に後者の場合、GradleでKotlinのプラグインを`multiplatform`にすることでFrontendではJavaScriptにコンパイルされ、サーバサイドではいつも通りJVMのバイトコードにコンパイルされるようにできるという点がお気に入りです。

### 思ったことと違った点

自分の考えが甘かったのですが、Desktopとはかなり違うところがありました。何かというと、言語としてはKotlinを使うとしても、HTMLやCSSを排除することはできないという点です。ここでもやはり`div`や`form`のようなタグを使ったり、タグにマウスオーバ時のカーソルを変えるためにタグの`attr`を変更する必要がありました。例えば、以下はファイルアップロードのコンポーネントですが、Kotlinで書いているだけで実際はHTMLをそのまま書いているような感覚です。

```kotlin
@Composable
private fun FileUploadForm(currentPath: String) {
    Div {
        Form(
            action = "$API_URL$ENPOINT_UPLOAD",
            attrs = {
                method(FormMethod.Post)
                encType(FormEncType.MultipartFormData)
                target(FormTarget.Blank)
            }
        ) {
            HiddenInput {
                name("target")
                value(currentPath)
            }
            Input(InputType.File) { name("file") }
            Input(InputType.Submit)
        }
    }
}
```

ここは完全に他のプラットフォームでのComposeを使うというよりは、Kotlinようにラップされたクラスを提供するだけという印象が強く、やはりある程度Frontendの知識が必要となる部分ではないかと思っています。なので、[React](https://reactjs.org/)や[Vue.js](https://vuejs.org/)などメジャーなFrontendのフレームワークの知識がある場合にはあまりComposeを選ぶ理由はなさそうな気がしています。

他には、いつもとは違ってKotlin/JSとKotlin/JVMが共存するプロジェクトとなっているためか、intellij上の自動補完やビルド時の挙動が少し違う感覚があります。例えば、Gradleで依存関係を変更してもすぐに反映されなかったり…

### 問題だった点

意外と問題になったのは、プロジェクトのビルドでした。Compose for Webでは`index.html`ファイルとWebpackなどを使ってビルドされた`js`ファイルを使うことになり、ビルド自体はGradleのコマンドひとつで簡単にできるものですが、どうやら内部的に[yarn](https://yarnpkg.com/)などを使っているようですが、intellijで生成したプロジェクトのデフォルト設定ではビルド時にエラーが出ることが多かったです。

調べてみると自分のようなエラーが出る場合、[ビルドできない場合はKotlinのバージョンが`v1.6.20`以降だと解消される](https://github.com/Kotlin/kotlinx-datetime/issues/193)らしいのですが、問題はComposeのバージョンでした。このアプリを実装した時点の最新は[v1.1.1](https://github.com/JetBrains/compose-jb/releases/tag/v1.1.1)なのですが、これだと対応しているKotlinのバージョンが`v1.6.10`までです。なので、自分の場合は`v1.2.0`のベータ版を使ってKotlinのバージョンを`v1.7.10`にしてから解消できました。これはマイナーなプロジェクトのハマりどころと言えるものかもしれないですね。

また、HTTPクライアントとしては[Ktor Client](https://ktor.io/docs/getting-started-ktor-client.html)を使っていますが、大容量のファイルをアップロードする場合を想定して`form`タグでMultipartのデータを直接送るよりHTTPクライアントを使う方法を取ろうとするとうまくいかなかったです。Ktor ClientはMultiplatform対応のものなので、クライアントの宣言時に[どのEngineを使うかを選択できる](https://ktor.io/docs/http-client-engines.html)のですが、Kotlin/JSで使えるEngineだと[公式で紹介している内容](https://ktor.io/docs/request.html#binary)通りに実装しても`File`オブジェクトを直接扱えないので送信ができませんでした。ここは今後の改善に期待するか、Websocketなどを使うなど他の方法を取る必要がありそうです。

## Backend

次にBackendですが、こちらは自分の慣れている分野で、Ktor自体については他のポストでも述べたことがあり、技術的な面の話よりはロジック面で試行錯誤をしたことを中心に述べていきたいと思います。

### ファイルツリーのブラウズ

このアプリにはまずファイルをブラウズする機能があるので、クライアントで指定したパスを探索して、その中にあるコンテンツ（ファイルとフォルダ）を返す必要があります。問題は、JSONの構造をどうするかですね。ここではまず一つの方法を試してみてから判断することにしました。

#### 全取得する

最初は、以下のような形で実装をしようと思いました。パスを指定したら、その配下にある全てのフォルダをたどり、親子関係をネストで表現する形です。

```json
{
  {
    "name": "Documents",
    "type": "directory",
    "children": [
      {
        "name": "SomeDocument.pdf",
        "type": "file",
        "size": "1391482",
        "mimeType": "application/pdf"
      }
    ]
  },
  {
    "name": "Images",
    "type": "directory",
    "children": []
  }
}
```

このようなファイルツリー返すために、サーバ側のコードは以下のようなものを使いました。

```kotlin
// ルートとなるパスを指定すると、子要素（ファイルとフォルダ）を全て取得する
val files =  Files.list(root)
        .filter { !it.isHidden() }
        .map { it.toFileTree() }
        .toList()

// PathをJSONオブジェクトとして加工する
fun Path.toFileTree(): FileTree {
    return FileTree(
        name = this.fileName.toString(),
        size = if (this.isDirectory()) null else this.fileSize(),
        type = if (this.isDirectory()) FileType.DIRECTORY else FileType.FILE,
        children = if (this.isDirectory()) {
            Files.list(this)
                .filter { !it.isHidden() }
                .map { it.toFileTree() }
                .toList()
        } else {
            null
        }
    )
}
```

[Files.walk()](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#walk-java.nio.file.Path-java.nio.file.FileVisitOption...-)を使うと、指定したパスを基準にネストされているファイルツリーを全て`Stream<Path>`として取得してくれますが、それだと上記のJSONの形として加工するのが簡単ではないです。一度取得した結果をもとに、親子関係を追跡しながらJSONオブジェクトとしてまとめるにはかなり複雑な処理になるっでしょう。

なので、ここでは[Files.list()](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#list-java.nio.file.Path-)を使って指定したパスに含まれた要素を取得し、その要素がディレクトリの場合はさらに子要素として取得するように再帰を使って子要素を再度取得するという形としてまとめています。単純な処理ですが、効率的になりましたね。

ただ、この方法で思った通りのファイルツリーをJSONとして返すことはできましたが、問題がありました。まず、指定したパスがルートに近くなればなるほど探索にかかる時間が長くなり、レスポンスも遅く慣ればJSONのサイズも大きくなるという問題がありました。また、JSONを受け取ったところで、Frontendで描画をするにも難点がありそうな気がしました。というわけで、この案はまず廃棄して、二つ目の方法を取ることにしました。

#### ネストさせない

次に試した方法は、指定したパスのみに処理を制限することでした。何かというと、JSONオブジェクトのネストを無くして、指定したパスにどんなファイルとフォルダが含まれているかだけをリストとして返すということです。つまり、以下のような形になります。

```json
[
  {
    "name": "Documents",
    "type": "directory"
  },
  {
    "name": "Images",
    "type": "directory"
  }
]
```

こうなると、中のフォルダを辿る必要がなくなるのでレルポンスも早く、軽くなるわけです。最初からちゃんと考えるべきだったのですが、こちらの方がFrontendとしても実装が楽であって、さらにサブパスのフォルダにアクセスしたい場合はそのパスを再度Backendに送ればいいだけですね。コードとしては再帰を使わなくなったくらいです。

```kotlin
val files =  Files.list(root)
        .filter { !it.isHidden() }
        .map { it.toFileTree() }
        .toList()

// PathをJSONオブジェクトとして加工する
fun Path.toFileTree(): FileTree {
    return FileTree(
        name = this.fileName.toString(),
        size = if (this.isDirectory()) null else this.fileSize(),
        type = if (this.isDirectory()) FileType.DIRECTORY else FileType.FILE
    )
}
```

### ファイルアップロード

ファイルアップロードについては、Mutlipartとして送られているデータをどう扱うかですが、これはKtorらしく簡単な処理で対応できました。以下のコードが実際の実装となっています。

```kotlin
// router
post(ENPOINT_UPLOAD) {
    // Multipartデータを受信
    val multipart = call.receiveMultipart()
    // ファイル保存先のパス
    var path = Path.of(ROOT_DIRECTORY)
    multipart.forEachPart { part ->
        when (part) {
            // ルートでないパスを指定した場合は保存先を更新
            is PartData.FormItem -> {
                if (part.name == "target") {
                    path = path.resolve(part.value)
                }
            }
            // ファイルデータを保存
            is PartData.FileItem -> {
                withContext(Dispatchers.IO) {
                    val file = Files.createFile(path.resolve(part.originalFileName!!))
                    part.streamProvider().use { input ->
                        Files.newOutputStream(file).use { output ->
                            input.copyTo(output)
                        }
                    }
                }
            }
            // どちらでもない場合は一旦出力
            else -> {
                println("Unknown part: $part")
            }
        }
        // 処理の終わったデータはdispose
        part.dispose()
    }
}
```

ただ、個人的にはストレージアクセスのある処理に対してはNIOを使いたいので、はじめは[Files.copy()](https://docs.oracle.com/javase/jp/8/docs/api/java/nio/file/Files.html#copy-java.io.InputStream-java.nio.file.Path-java.nio.file.CopyOption...-)を使おうと思ったのですが、なぜかファイルの保存処理を以下のような作成するとうまくいかなったです。Coroutineとの相性に何か問題があるのかもしれないですので、ここは注意ですね。

```kotlin
val file = Files.createFile(path.resolve(part.originalFileName!!))
Files.copy(part.streamProvider(), file) // ファイルが保存されない
```

### ファイルダウンロード

ファイルダウンロードの場合も、ロジックは特にないので、ほとんどKtorのみのコードとなっています。自分の好みでPathを使っているくらいですのでコードだけを紹介します。ひとつ注意すべきところは、アップロードする時もそうですが、ファイル名を返すときにURLパスとしてエンコードすることですね。

```kotlin
get(ENDPOINT_DOWNLOAD) {try {
    val filepath = call.request.queryParameters["filepath"] ?: ""
    val path = FileService.getFullPath(filepath) // ルートディレクトリからのフルパスを取得
    if (Files.notExists(path)) {
        call.respond(HttpStatusCode.BadRequest, "File not found")
    } else {
        call.response.header(
            name = HttpHeaders.ContentDisposition,
            value = ContentDisposition.Attachment.withParameter(
                key = ContentDisposition.Parameters.FileName,
                value = path.fileName.toString().encodeURLPath()
            ).toString()
        )
        call.respondFile(path.toFile())
    }
}
```

### 注意すべきところ

まず、一つのプロジェクトにKotlin/JSとKotlin/JVMを両立する場合、`dependencies`として記述するものに対しては`build.gradle.kts`ファイルで以下のように指定することができます。

```kotlin
kotlin {
    sourceSets {
        // Kotlin/JSの依存関係
        val jsMain by getting {
            dependencies {
                implementation(compose.web.core)
                implementation(compose.runtime)
                // ...省略
            }
        }
        // Kotlin/JVMの依存関係
        val jvmMain by getting {
            dependencies {
                implementation("io.ktor:ktor-server-core-jvm:$ktor_version")
                implementation("io.ktor:ktor-server-auth-jvm:$ktor_version")
                // ...省略
            }
        }
    }
}
```

しかし、Composeを使うためには`plugin`として指定する必要があり、これがプロジェクト全体の依存関係に追加されることになっていました。なので、アプリの作りとしてはまずFrontendのComposeをビルドし、サーバを起動したらビルドしたファイルをstaticとして提供する構造になっていますが、Backendの起動にもComposeのランタイムが必要になります。このランタイムを追加してくれないと、エラーが吐き出され、Ktorが起動できなくなっています。何かKotlin/JSのみの依存関係にpluginを追加する他の方法があるかもしれませんが、とりあえずはJVMの依存関係に以下のようにランタイムを追加することで問題は解消できました。

```kotlin
val jvmMain by getting {
    dependencies {
        implementation("io.ktor:ktor-server-core-jvm:$ktor_version")
        implementation("io.ktor:ktor-server-auth-jvm:$ktor_version")
        // ...省略
        implementation(compose.runtime) // Composeランタイム
    }
}
```

## その他

Kotlin/JSとKotlin/JVMを一つのプロジェクトとして扱う場合に、`common`というパッケージを設けることで、コードの共有ができるのが何より嬉しかったところです。例えば、JSONオブジェクトをdata classとして定義してcommonパッケージに置くことで、FrontendとBackendの両方で同じオブジェクトを使うことができます。他にももちろんEnumやconstを共有できたりするので、実装がかなり楽でした。

また、今回は採用しなかったのですが、Ktor Serverの場合[Type-safe Routing](https://ktor.io/docs/type-safe-routing.html#resource_classes)というものに対応しているので、うまく活用できたらかなり良さそうな気がしました。これはKtor Clientでも[Type-safe Request](https://ktor.io/docs/type-safe-request.html#define_route)として対応しているので、FrontendとBackend両方で使える機能です。またKtorを使う機会があったら、ぜひ使ってみたいと思っています。

## 最後に

ファイルアップロードが思った通り改善できなかったので、アプリの完成はまだ少し先のことになりそうですが、かなり面白い経験となりました。Kotlinでできることは色々とあるので、また何か作ってみたいものがあればチャレンジしてみたくなります。ただ、やはりまだ成熟してない技術なので、思ってもなかったところで問題が発生したりリファレンスがあまりないという点ではまだプロダクションレベルでは使えないものかなという気がします。

アプリ全体のコードはGitHubにて公開していますので、[こちら](https://github.com/retheviper/FileTransporter)から参照できます。

では、また！
