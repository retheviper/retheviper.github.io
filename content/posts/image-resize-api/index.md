---
title: "動的画像リサイズAPIを作る"
date: 2024-03-30
categories: 
  - ktor
image: "../../images/ktor.jpg"
tags:
  - ktor
  - kotlin
  - scrimage
  - rust
---

この度は、動的画像リサイズのAPIを作りましたのでその紹介です。ここでおいう動的画像リサイズAPIとは、元の画像のURLとリサイズしたいサイズを指定すると、そのサイズにリサイズした画像を返すAPIです。

## 目的

そもそも動的画像リサイズAPIを作る理由はなにか。それから説明しないとですね。今までは画像の配信において、エンドユーザが画像をアップロードする場合、あらかじめサムネイルの画像も作成してアップロードするようにしていました。

ただ、その方法だと全ての画面比と解像度に対応したサムネイルを作れないという問題があります。そのため代案として、フロントエンドから画面に最適化したサイズを指定して、APIからリサイズした画像を取得するようにしたいというのが目的です。

## 設計・技術選定

APIはマイクロサービスとして、なるべくシンプルに作ることにしました。Getのエンドポイントを一つ持っていて、そこにクエリパラメータで元の画像のURLとリサイズしたいサイズを指定すると、リサイズした画像を返すというものです。フロントエンドではそのままimgタグに使えるようにしたいので、返すのは画像のデータそのまま（ヘッダーには`Content-type`を指定）にします。また、リサイズだけでなく、画像の形式を変換することもできるようにします。

また、APIは[Cloud Run](https://cloud.google.com/run?hl=ja)上で動かすことにしました。既存のAPIもそうなので使い方を合わせるためでもあり、ローカルでの開発でもdocekr composeを使って楽に開発できるためです。他にもコンテンツの配信には[Cloud Storage](https://cloud.google.com/storage?hl=ja)を使っているので、それとの連携もしやすいためです。そして最終的に処理された画像は、Cloud CDNに保存され、2回目以降の呼び出しではキャッシュを返すようにします。

会社ではすでにウェブフレームワークとしてKtorを採用しているので、それと揃えるためにこちらもKtorを採用。Ktorの以前ににNode.jsとRustによる実装を試みたことがありますが、前者は性能の問題から、後者はメンテが難しくなる問題から（社内にRustができる人が少ないので）採用しない方になっています。当時作成していたRustバージョンに近いサンプルコードはGitHubにて公開していますので、[こちら](https://github.com/retheviper/resize-api)から確認できます。

画像の変換とリサイズのためには[Scrimage](https://github.com/sksamuel/scrimage)を採用することにしました。他の候補としてはJavaの[ImageIO](https://docs.oracle.com/javase/jp/8/docs/api/javax/imageio/ImageIO.html)なども検討しました。ただ、リサイズ対象の画像のフォーマットと、返すデータのフォーマットとしてWebPを処理する必要があったのですがそれに対応していないものが多かったです。

他に考えたものとしては、[GraphicsMagick](http://www.graphicsmagick.org/)のように画像の変換やリサイズを行うツールを使う方法もあります。こちらの場合はJavaから取得したデータを一度ファイルに書き出して、それをコマンドラインで実行するという方法になりますので、その分のI/Oコストがかかるため今回はScrimageを使うことにしました。

## リサイズ処理

では、実際のAPIを書いていきます。Ktorは使い慣れているのもあり、今回はKtorでのAPI構築というよりはScrimageを使った画像のリサイズ処理が重要なので、その部分に焦点を当てていきます。

API全体で処理のフローは大まかに以下の通りです。

1. クエリパラメータからurlとリサイズ後の大きさを取得
2. 画像の取得
3. 画像のリサイズ
4. 画像の形式変換

ここでScrimageを使った画像の処理は3〜4の部分ですが、実際のリサイズを行う前に取得した画像の形式をまず判定したり、画像のサイズを確認する必要もあります。理由としては処理の効率化のためですね。

今回はPNG, JPEG, WEBP, GIFの4つの形式に対して、リサイズ後にWEBPに変換するという処理を行うことにしています。ここで元の画像がWEBPだった場合、あえてWEBPに変換する必要はないです。また、リサイズが必要ない場合もあります。そのため、まずは画像の形式を判定して、リサイズが必要な場合のみリサイズ処理を行うようにします。

### 画像の形式判定

URLもしくはローカルストレージ（今回はCloud RunにCloud Storageをマウントする形で使っている）から画像を取得する際、その画像の形式を判定する必要があります。Scrimageでは画像の形式を判定するための[FormatDetector](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/format/FormatDetector.java)というクラスを提供しています。

使い方は簡単で、以下のように読み込んだ画像のデータをByteの配列で渡すだけです。API上では想定してないフォーマットが来た場合はエラーを返すようにしていて、ここで返す[Format](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/format/Format.java)はPNG, GIF, JPEG, WEBPでありScrimageのものをそのまま使っています。

```kotlin
fun detectImageFormat(data: ByteArray): Format {
    return FormatDetector.detect(data).orElseThrow { IllegalArgumentException("Unsupported format") }
}
```

### PNG, JPEGの処理

まず一番簡単なPNG, JPEGの場合です。これらの形式の場合、Scrimageの[ImmutableImage](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/ImmutableImage.java)として扱うことになります。

ここで画像のデータをImmutableImageに変換するには[ImageReader](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/nio/ImageReader.java)のインタフェースを実装したクラスを使います。[公式サイト](https://sksamuel.github.io/scrimage/)では`ImmutableImage.loader()`で形式に関係なく画像を読み込むことができると書いてありますが、実際にAPIをビルドする際はAWT関連のエラーが出るので、形式に応じて読み込むクラスを変える必要があります。

```kotlin
fun asImmutableImage(rawData: ByteArray): ImmutableImage {
    return ImageIOReader().read(rawData)
}
```

ImmutableImageに変換したら、リサイズ処理を行います。Scrimageではリサイズのためのメソッドが用意されているので、それを使ってリサイズを行います。

ここで注意すべきは、`resize()`や`resizeTo()`のようなメソッドがあるのですが、前者の場合はパーセントでのリサイズ、後者の場合は指定したサイズにリサイズするという違いがあります。これらの場合、元の画像のアスペクト比が保持されないため、`scaleTo()`や`scaleToWidth()`などのメソッドを使う必要があります。

今回はwidthのみを指定してアスペクト比を保持したままリサイズするため、`scaleToWidth()`を使います。

```kotlin
fun resizeImmutableImage(image: ImmutableImage, width: Int): ImmutableImage {
    return image.scaleToWidth(width)
}
```

最後に、リサイズした結果をByteArrayとして返すためのメソッドを用意します。どの形式に変換するかによって[ImageWriter](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/nio/ImageWriter.java)を実装するクラスを選ぶ必要があります。今回はWEBPにしたいので、[WebpWriter](https://github.com/sksamuel/scrimage/blob/master/scrimage-webp/src/main/java/com/sksamuel/scrimage/webp/WebpWriter.java)を使います。

```kotlin
fun encodeImage(image: ImmutableImage): ByteArray {
    return image.bytes(WebpWriter.DEFAULT)
}
```

最後にRouterでは、Content-Typeを指定してByteArrayを返すようにします。

```kotlin
call.respondBytes(
    bytes = resizedImage,
    contentType = ContentType("image", "webp")
)
```

### WEBPの場合

WEBPの場合は、上記データ読み込む時点で`WebpImageReader`を使う必要があります。その後のリサイズ処理はPNG, JPEGの場合と同じです。

ただ、形式がWEBPの場合には[scrimage-webp](https://github.com/sksamuel/scrimage/tree/master/scrimage-webp)が提供している[WebpImageReader](https://github.com/sksamuel/scrimage/blob/master/scrimage-webp/src/main/java/com/sksamuel/scrimage/webp/WebpImageReader.java)を使う必要があります。なので、形式に応じて読み込むクラスを変える必要があります。先ほどの`asImmutableImage`の引数に形式を追加して、形式に応じて読み込むクラスを変えるようにします。

```kotlin
fun asImmutableImage(rawData: ByteArray, format: Format): ImmutableImage {
    return when (format) {
        Format.WEBP -> WebpImageReader().read(rawData)
        Format.GIF, Format.PNG, Format.JPEG -> ImageIOReader().read(rawData)
    }
}
```

他の処理はPNG, JPEGの場合と同じです。

### GIFの場合

GIFの場合は、[AnimatedGif](https://github.com/sksamuel/scrimage/blob/master/scrimage-core/src/main/java/com/sksamuel/scrimage/nio/AnimatedGif.java)というクラスを使ってリサイズを行います。ImmutableImageと同じようにリサイズメソッドが用意されているので、それを使ってリサイズを行います。処理で使われるクラスが違うので、GIFの場合は別途メソッドを用意します。

```kotlin
fun asAnimatedGif(rawData: ByteArray): AnimatedGif {
    return AnimatedGifReader.read(ImageSource.of(rawData))
}
```

また、AnimatedGifの場合はframesというプロパティで各フレームのデータを保持していて、これらはImmutableImageとして扱うことができます。そのため、リサイズ処理は各フレームに対して行い、それをAnimatedGifに戻すという処理を行います。若干複雑ですが、以下のように書くことができます。

```kotlin
suspend fun resizeAnimatedGif(gif: AnimatedGif, width: Int): AnimatedGif {
    val resizedData = ByteArrayOutputStream().use {
        StreamingGifWriter().prepareStream(it, BufferedImage.TYPE_INT_ARGB).use { stream ->
            gif.frames.mapIndexed { index, image ->
                stream.writeFrame(image.scaleToWidth(width), gif.getDelay(index))
            }
            it.toByteArray()
        }
    return AnimatedGifReader.read(ImageSource.of(resizedData))
}
```

最後に、リサイズした結果をByteArrayとして返すためのメソッドを用意します。基本的にはImmutableImageと同じですが、GIFをWEBPに変換する場合は[Gif2WebpWriter](https://github.com/sksamuel/scrimage/blob/master/scrimage-webp/src/main/java/com/sksamuel/scrimage/webp/Gif2WebpWriter.java)を使います。これでWEBPに変換後も、GIFのアニメーションが保持されたままリサイズすることができます。

```kotlin
fun encodeGif(gif: AnimatedGif): ByteArray {
    return gif.bytes(Gif2WebpWriter.DEFAULT)
}
```

## 処理を共通化する

ここまででPNG, JPEG, WEBP, GIFの4つの形式に対してリサイズ処理を行うことができました。ただ、それぞれの形式に対して処理を書いていると、処理が重複してしまうため、共通化する必要があります。特にImmutableImageとAnimatedGifの処理は似ているため、それらを共通化することにします。

### 共通Interfaceを作る

ScrimageではImmutableImageとAnimatedGifは別のクラスであるだけでなく、共通のInterfaceを持っていないため、まずはそれを作成する必要があります。ここでは、ImageというInterfaceを作成し、それを実装するクラスを作成します。それぞれのクラスはWrapperとして作成し、それぞれのクラスのプロパティをInterfaceのプロパティとして持つようにします。

```kotlin
sealed interface Image {
    val width: Int
}

class AnimatedGifWrapper(
    val animatedGif: AnimatedGif
) : Image {
    override val width: Int
        get() = animatedGif.frames.first().width
}

class ImmutableImageWrapper(
    val immutableImage: ImmutableImage
) : Image {
    override val width: Int
        get() = immutableImage.width
}
```

### 画像取得の共通化

あとは外部に公開するAPIとして、`asImage`というメソッドを作成し、それぞれの形式に応じてWrapperを返すようにします。ここで、形式の判定は先ほど作成した`detectImageFormat`を使います。

```kotlin
fun asImage(rawData: ByteArray): Image {
    return when (val format = detectImageFormat(rawData)) {
        Format.GIF -> asAnimatedGifWrapper(rawData)
        Format.WEBP, Format.PNG, Format.JPEG -> asImmutableImageWrapper(rawData, format)
    }
}

private fun asAnimatedGifWrapper(rawData: ByteArray): AnimatedGifWrapper {
    val gif = AnimatedGifReader.read(ImageSource.of(rawData))
    return AnimatedGifWrapper(gif)
}

private fun asImmutableImageWrapper(rawData: ByteArray, format: Format): ImmutableImageWrapper {
    val image = when (format) {
        Format.WEBP -> WebpImageReader().read(rawData)
        Format.GIF, Format.PNG, Format.JPEG -> ImageIOReader().read(rawData)
    }
    return ImmutableImageWrapper(image)
}
```

### リサイズ処理の共通化

同じく、リサイズ処理も共通化します。ここでは、`resizeImage`というメソッドを作成し、それぞれの形式に応じてリサイズ処理を行うようにします。ここで、リサイズ処理は先ほど作成した`resizeImmutableImage`と`resizeAnimatedGif`を使います。AnimatedGifのリサイズ処理はまた別途`writeAnimatedGif`というメソッドを作成して分けています。

ここでImageはsealed interfaceとして作成しているため、分岐処理はwhen式を使って網羅することができます。

```kotlin
suspend fun resizeImage(image: Image, width: Int): Image {
    return when (image) {
        is AnimatedGifWrapper -> resizeAnimatedGif(image, width)
        is ImmutableImageWrapper -> resizeImmutableImage(image, width)
    }
}

private suspend fun resizeAnimatedGif(gifWrapper: AnimatedGifWrapper, width: Int): AnimatedGifWrapper {
    val gif = gifWrapper.animatedGif
    val resizedData = writeAnimatedGif { stream ->
        gif.frames.mapIndexed { index, image ->
            stream.writeFrame(image.scaleToWidth(width), gif.getDelay(index))
        }
    }
    val resizedGif = AnimatedGifReader.read(ImageSource.of(resizedData))
    return AnimatedGifWrapper(resizedGif, resizedData)
}

private suspend fun writeAnimatedGif(writeFunction: (StreamingGifWriter.GifStream) -> Unit): ByteArray {
    return withContext(Dispatchers.IO) {
        StreamingGifWriter().use {
            streamingGifWriter.prepareStream(it, BufferedImage.TYPE_INT_ARGB).use { stream -> writeFunction(stream) }
            it.toByteArray()
        }
    }
}

private fun resizeImmutableImage(imageWrapper: ImmutableImageWrapper, width: Int): ImmutableImageWrapper {
    val resizedImage = imageWrapper.immutableImage.scaleToWidth(width)
    return ImmutableImageWrapper(resizedImage, imageWrapper.format)
}
```

### 画像のエンコード処理の共通化

最後に、画像のエンコード処理も共通化します。ここでは、`encodeImage`というメソッドを作成し、それぞれの形式に応じてエンコード処理を行うようにします。

```kotlin
suspend fun encodeImage(image: Image): ByteArray {
    return when (image) {
        is AnimatedGifWrapper -> image.animatedGif.bytes(Gif2WebpWriter.DEFAULT)
        is ImmutableImageWrapper -> image.immutableImage.bytes(WebpWriter.DEFAULT)
    }
}
```

これで共通化も終わり、呼び出す側は`asImage`, `resizeImage`, `encodeImage`の3つのメソッドを使うだけでリサイズ処理を行うことができます。サイズのチェックもImageの方でプロパティ化しているため、それを使ってリサイズが必要かどうかの判定も行うことができます。

あとはImageWriterなど、毎回インスタンスを生成する必要のないクラスは、シングルトンとして作成しておくとよいでしょう。

## 最後に

だいぶ長くなってしまいましたが、ここまでで動的画像リサイズAPIの作成方法を紹介しました。Scrimageを使うことで、PNG, JPEG, WEBP, GIFの4つの形式に対してリサイズ処理を行うことができ、共通化することで処理の重複を防ぐことができました。

あとは、Cloud Run上で動かすためのDockerfileを作成し、Cloud Storageとの連携を行うことで、画像の配信を行うことができます。また、Cloud CDNを使うことで、2回目以降の呼び出しではキャッシュを返すようにすることで、負荷を軽減することができます。なかなか面白いプロジェクトでした。

では、また！
