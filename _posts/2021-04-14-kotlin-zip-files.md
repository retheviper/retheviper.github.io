---
title: "KotlinでZIP圧縮してみる"
date: 2021-04-14
categories: 
  - kotlin
photos:
  - /assets/images/sideimage/kotlin_logo.jpg
tags:
  - kotlin
  - java
---

サーバサイドの機能を作っていると、ファイルダウンロード機能が必要な時があります。ただ、ストレージに保存されてあるファイルをそのまま返すということだけでなく、場合によってはファイルを生成してそのまま返したり、複数のファイルをまとめて転送する必要もありますね。

リクエストごとに一つのファイルをダウンロードさせるとしたら、実装はそう難しくないものですが、複数のファイルをダウンロードさせるという場合は少し複雑になりますね。ファイルを一つにまとめて送るとしたら、ZIPに圧縮した方が良いでしょう。幸い、Javaでは基本的に[ZipOutputStream](https://docs.oracle.com/javase/jp/8/docs/api/java/util/zip/ZipOutputStream.html)というAPIを提供しているので、エントリに圧縮対象のファイルを追加したあとZIPファイルを出力だけで良いです。

ただ、単純にファイルが複数だるだけでなく、ディレクトリが多重にネストされてあったりする場合は、ディレクトリ構造を維持しつつそのまま圧縮するとかの追加的な処理が必要となります。そして場合によっては含めたくないファイルがあるケースもあったりしますね。そしてなるべくファイルの数に関係なく(ファイルが一つであれ、ディレクトリであれ)一つの機能で済ませたいものです。なので、今回はそのようなユースケースに合わせた簡単なメソッドを作る方法を、JavaのコードからKotlinへ移行していく過程を簡単に紹介したいと思います。

今回紹介しますコードは、はBaeldungの[JavaでZipを圧縮する方法](https://www.baeldung.com/java-compress-and-uncompress)に関する記事に紹介されてあるものをベースにしています。

## Java

まずJavaのコードを見ていきましょう。上記の記事には、以下のようなコードが紹介されています。

```java
public class ZipDirectory {
    public static void main(String[] args) throws IOException {
        String sourceFile = "zipTest";
        FileOutputStream fos = new FileOutputStream("dirCompressed.zip");
        ZipOutputStream zipOut = new ZipOutputStream(fos);
        File fileToZip = new File(sourceFile);

        zipFile(fileToZip, fileToZip.getName(), zipOut);
        zipOut.close();
        fos.close();
    }

    private static void zipFile(File fileToZip, String fileName, ZipOutputStream zipOut) throws IOException {
        if (fileToZip.isHidden()) {
            return;
        }
        if (fileToZip.isDirectory()) {
            if (fileName.endsWith("/")) {
                zipOut.putNextEntry(new ZipEntry(fileName));
                zipOut.closeEntry();
            } else {
                zipOut.putNextEntry(new ZipEntry(fileName + "/"));
                zipOut.closeEntry();
            }
            File[] children = fileToZip.listFiles();
            for (File childFile : children) {
                zipFile(childFile, fileName + "/" + childFile.getName(), zipOut);
            }
            return;
        }
        FileInputStream fis = new FileInputStream(fileToZip);
        ZipEntry zipEntry = new ZipEntry(fileName);
        zipOut.putNextEntry(zipEntry);
        byte[] bytes = new byte[1024];
        int length;
        while ((length = fis.read(bytes)) >= 0) {
            zipOut.write(bytes, 0, length);
        }
        fis.close();
    }
}
```

`zipFile`メソッドをみると、引数の`fileToZip`にZIPで圧縮したいファイルやディレクトリのパスを指定して、`fileName`にはファイルもしくはディレクトリ名、`zipOut`には圧縮後のZIPのファイル名を指定するようになっています。

そして実装としては、指定したファイルやディレクトリに`hidden`属性がある場合は圧縮しなく、圧縮元のファイルがディレクトリである場合は中のファイルを全部ZIPに含ませるという処理が含まれてありますね。対象のファイルとディレクトリを全部エントリに追加した後は、圧縮元を読み込んでZipOutputStreamに書き込むという処理となっています。これをKotlinのコードに変えてみましょう。

## Kotlinのコードに変えてみる

JavaのコードをKotlinのコードに変えるのはそう難しくありません。Intellijの場合、すでにJavaのコードを貼り付けると自動でKotlinのコードの変換してくれる機能を搭載していますので。ただ、それだけでは十分ではないですね。簡単に変換ができるとしても、それが本当に`Kotlinらしいコード`になっているとはいえない場合があります。

そして、処理自体もより単純に、もしくは読みやすいコードにする方法もあるはずですね。上記のJavaコードをまずKotlinに変えて、色々改善したいところを含めて変えていきます。

### Kotlinらしいコードに変える

Intellij 2021.1を基準に、Javaのコードをそのまま貼り付けると以下のようなコードに自動変換されます。

```kotlin
@Throws(IOException::class)
private fun zipFile(fileToZip: File, fileName: String, zipOut: ZipOutputStream) {
    if (fileToZip.isHidden) {
        return
    }
    if (fileToZip.isDirectory) {
        if (fileName.endsWith("/")) {
            zipOut.putNextEntry(ZipEntry(fileName))
            zipOut.closeEntry()
        } else {
            zipOut.putNextEntry(ZipEntry("$fileName/"))
            zipOut.closeEntry()
        }
        val children = fileToZip.listFiles()
        for (childFile in children) {
            zipFile(childFile, fileName + "/" + childFile.name, zipOut)
        }
        return
    }
    val fis = FileInputStream(fileToZip)
    val zipEntry = ZipEntry(fileName)
    zipOut.putNextEntry(zipEntry)
    val bytes = ByteArray(1024)
    var length: Int
    while (fis.read(bytes).also { length = it } >= 0) {
        zipOut.write(bytes, 0, length)
    }
    fis.close()
}
```

ここでもっとKotlinらしいコードに変えたい部分は、`InputStream`や`OutputStream`の使い方です。Javaでも`try-with-resource`があって、Kotlinには`use()`があるのでそちらを使った方が`close`よりも良い気がします。

また、`if`は`when`に変えたり、`for`を`forEach()`に変えたりなどでよりやりたいことを明確にすることができるようにも見えます。個人的にはスコープをあえて分けたほうが責任が明確になり、処理を追うときに混乱しないのでなるべくスコープ関数やCollection専用のオペレーションを積極的に使用して処理の単位を分けられるところはきちんと分けたいと思います。Javaのやり方をとっても処理としては全く問題がありませんが、せっかくなのでKotlinならではのコードを描きたいものです。

あえてIOExceptionを投げるという表示をしておくというのも、ランタイム時の例外の処理を強制してないKotlinには相応しくないのではないかという気もするので、アノテーションは削除することとします。

### IOからNIOに変える

NIOに関しては以前のポストで何回か言及したことがありますが、サーバのように頻繁かつ同時実行数が多いケースは積極的に採用した方が良いと思います。また、Java 1.8以降から追加されたメソッドでかなり便利に使える機能が多いので、IOをNIOに変えるだけでコードの量をかなり減らせる可能性もあります。

特にディレクトリを指定した場合、そのディレクトリの子要素を循環するにはNIOの`Files`が提供する機能が強力なので、今回はそれを積極活用することにします。

### シグニチャーを変える

上記のメソッドでは、三つの引数を取っていますが、実際に必要なのは圧縮元のパスと、圧縮先のパスのみですね。ZipOutputStreamを呼び出し元で渡す理由は特になく、むしろこのメソッドを利用する度に定義する必要があるので不便ですね。そして、メソッドの中で単純にエントリを追加していて、呼び出し元とオブジェクトに対する処理の職務を分担するという構造もあまりよくないかと思います。なので、ZipOutputStreamの生成と使用はメソッドの中で完結するように変えることにします。

こうすることで、メソッドの外側(呼び出し元)での使い方はもっと簡単になりますし、圧縮元のデータを読み込む際に使う`InputStream`は中で閉じているのに引数の`OutputStream`は外で閉じるという複雑な状況は避けられます。

### 再帰を無くす

圧縮元のパスがディレクトリである場合は、さらにネストされたディレクトリやファイルもまとめて圧縮するために再帰を使うようになっています。再帰はアルゴリズムとしては重要ではあるものの、処理が全部終わるまでメモリに全データと処理を詰めておくので処理の効率という面ではあまりよくない場合もありますね。やりたいのは単純に`hidden`属性を持つファイルやディレクトリを除外すること、そしてそれ以外のファイルやディレクトリは全部ZipOutputStreamのエントリに入れたいという単純な事です。

幸い、NIOを使うことでディレクトリの子要素を全部取得することができますし、取得した子要素は`Stream<Path>`として取得できるので、`filter()`や`forEach()`のようなメソッドが使えます。これで十分、再帰を使わずに目的を達成できそうですね。

## 完成したコード

以上のことを反映し、修正したコードは以下の通りになります。

```kotlin
object ZipService {

    fun archive(source: Path, target: Path): Unit =
        ZipOutputStream(Files.newOutputStream(target)).use { zos ->
            when {
                Files.isHidden(source) -> return
                Files.isDirectory(source) -> {
                    Files.walk(source)
                        .filter { Files.isDirectory(it).not() }
                        .filter { Files.isHidden(it).not() }
                        .forEach {
                            zos.putNextEntry(ZipEntry(it.toString())).also { zos.closeEntry() }
                        }
                }
                else -> zos.putNextEntry(ZipEntry(source.toString()))
            }
        }
}
```

簡単に説明しますと、`object`として宣言したSingletonクラスにおくことでどこでも活用できるユーティリティクラスにして、メソッドのシグニチャはより単純なものにしました。引数の`source`には圧縮元のファイルやディレクトリを、`target`には圧縮先のZIPファイルを指定する事になっています。ZipOutputStreamはメソッドの中で生成して、`use()`を使って自動にクローズされるようにしています。

また、内部の分岐は`when`を使ってより単純化して、圧縮元がディレクトリである場合は`Files.walk()`を使って子要素を全部取得するようにしています。取得した子要素は`filter()`でディレクトリでも`hidden`でもない場合はエントリに追加するようにしています。より短く、単純なコードの出来上がりです。

あと、少しだけ補足しますと、あくまで自分の好みですが、Kotlinらしさを追求したいがため`also()`や`not()`を使っています。なのでここはまだ変えられる余地はたくさんあるのかも知れませんね。

## 最後に

`Kotlinらしいコード`と述べましたが、上記のコードはあくまで`Kotlin/JVM`でのみ有効ですね。なのでもし`Kotlin/Native`や`Kotlin/JS`などで使うには、別の方法を探す必要があるはずです。また、`Files.walk()`はJava 1.8から追加されたメソッドなので、1.7の場合は`Files.walkFileTree()`を、その以前なら仕方なくNIOではない別の方法を使う必要があると思います。

なので、`Kotlin/JVM`(Java 1.8以上)ではこれが最善なのかも知れませんが、また色々と研究の余地はありそうですね。こうやってJavaのAPIをKotlinの作法で切り替えていくのも、それなりに価値のあることではないかと思います。

では、また！
