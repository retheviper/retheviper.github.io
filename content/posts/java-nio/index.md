---
title: "IOからNIOへ"
date: 2020-01-20
categories: 
  - java
image: "../../images/java.jpg"
tags:
  - nio
  - java
---

私がそうでしたが、今でも初めてJavaでのI/Oを学ぶとしたら、やはりFileオブジェクトを生成してInputStreamで読み込んだり、OutputStreamで書き込むのが一般的なのではないかと思います。ここで少し発展すると、WriterやReaderなどのクラスを使ったり、StreamをBufferで包んだり、SerializableでオブジェクトのI/Oを行ったりするレベルまで行くのでしょう。

昔のAPIだとしても、動作や性能に大した問題がなければあえて新しいAPIにコードを全部変える必要はないと思います。むしろ無理やり新しいAPIに書き換えたコードが問題を起こす可能性もあるし、常に優秀とは言えませんので。例えばJava 1.8で追加されたforEach()は便利で、Lambdaが好きな私は多くの場面で使っていますが、実際は今までのJVMは伝統的なforループに最適化されているのでforEach()は性能で劣るらしいです。今後、forEach()の性能がより良くなる可能性もなくはないでしょうが、最近のJavaのバージョンアップ履歴をみると関数型APIの性能改善にどれだけ時間がかかるかは少し謎です。

新しいAPIを使うということにはこのような問題もあり、慎重ではければならないのですが、それでもAPIが新しくなるのには何らかの理由があるためなので、新しくコードを書いたり簡単なコードに変えたりするなどの理由があれば、積極的に新しいAPIを導入してみるということもそう悪くないと思います。今回紹介したいAPIもまたそのようなものです。ファイルI/Oを扱う新しい方式、NIOです。(と言っても、Java 1.7から導入されたので今はあまり新しくもないですが…)

## NIOって何？

NIOは、Javaの新しいI/Oのことです。Newの略かと思いがちなのですが、実際はNon-blockingの略らしいですね。JavaはCやC++と比べ遅いですが、その理由の一つがI/Oだったらしいです。なのでそれを改善するために出たのがこのNIOですと。

BlockingかNon-blockingかによる違い、Stream基盤かChannel基盤かという違いなど様々な違いがありますが、一般的には頻繁なI/Oが要求される場合ではNIOを選択した方がより良い性能を期待できると言います。他には以下のようなメリットがあります。

- スレッドのブロックが発生しない
- コードがより簡潔
- コピー、移動、読み込みのオプション指定が簡単
- 基本的にBufferを使うので、Buffered~でのラッピングが要らなくなる

あまりJVMの構造に詳しくないので、私がここで浅い知識を持って説明するようなことはしません。ただ、自分の観点からしてコードがより簡潔になるということは確かなメリットです。なので皆さんにもぜひ使ってみていただきたいと思います。

それでは、実際のコードでNIOをどう使うかについて説明して行きます。

## File → Path

NIOではFileオブジェクトよりPathオブジェクトを使います。PathはFileオブジェクトに比べ、ファイルパスをディレクトリとファイル名で分離して指定できるのが最大のメリットです。

例えばファイルパスが複数のフォルダでネストされているとしましょう。

```java
// 複数のディレクトリとファイルがそれぞれ文字列として存在(path/to/file.txt)
String rootDirectory = "path";
String toDirectory = "to";
String fileName = "file.txt";
```

この複数の文字列からインスタンスを作成するとしたら、Fileのコンストラクターは引数が一つの文字列なので以下のようになります。ディレクトリの文字列にスラッシュが含まれてないので、文字列を結合しながらスラッシュも一緒にいれる必要があります。

```java
// Fileオブジェクトの生成
File file = new File(rootDirectory + "/" + toDirectory + "/" fileName);
```

しかし、Pathの場合は指定できる文字列が複数でも構いません。ディレクトリとファイル名の文字列を順番通り指定するだけで良いです。

```java
// Pathオブジェクトの生成
Path path = Paths.get(rootDirectory, directory, fileName);
```

このように、インスタンスの作成がより便利なのがPathです。また、どうしてもFileオブジェクトが必要な場合があるとしても、FileのメソッドからPathに変換できる機能があるので便利ですね。もちろん、その逆もできます。

```java
// PathからFile生成
Path path = file.toPath();

// FileからPath生成
File file = path.toFile();
```

他にもtoURI()メソッドでURIオブジェクトを生成できるなど、PathにはFileと同じ機能をするようなメソッドが多いので、どちらか便利な方を使いましょう。

## Files

昔ながらのI/Oでファイルのコピーや削除などの操作を行うためにはInputStream、OutputStream、Writer、Readerなどのクラスを活用してきました。NIOでは主にこれらの作業をFilesクラスを持って行います。また、FilesクラスにはWriterとReader、InputStreamとOutputStreamを生成する機能もあるので使い勝手が良いクラスです。

### ファイルのコピー

Filesクラスでのファイルコピーは簡単です。以下のコードを見てください。基本的にコピー元とコピー先のファイルをPathオブジェクトとして指定するだけです。

```java
// PathをPathにコピー
Files.copy(source, target);
```

FilesクラスでのコピーにはENUMでコピー時のオプションを指定することもできます。

```java
// オプションを指定(ファイル属性もコピー)
StandardCopyOption option = StandardCopyOption.COPY_ATTRIBUTES;
Files.copy(source, target, option);
```

また、実際存在するファイルではなく、InputStreamをコピー元として指定することもできます。この場合、データをファイルに書き込むということもできますね。

```java
// InputStreamをPathにコピー
Files.copy(sourceStream, target);
```

### ファイルの削除

Filesクラスでのファイル削除はコピーと同じく、Pathオブジェクトを引数として渡します。

```java
// 削除
Files.delete(path);
```

戻り値がbooleanのメソッドも用意されています。ファイルが存在する場合は削除して、その結果をbooleanとして返します。

```java
// 存在する場合削除
Files.deleteIfExists(path);
```

### ファイルの移動

ファイルの移動は、コピーと削除の組み合わせみたいなものですね。また、ファイル名を変える場合にも使えます。基本がコピーだからか、コピーの時と同じオプションを使えます。

```java
// 移動もしくはリネーム
Files.move(path);

// オプションを指定(上書きする)
StandardCopyOption option = StandardCopyOption.REPLACE_EXISTING;
Files.move(path, option);
```

### ファイルの書き込み

InputStreamをcopy()で使えるのですが、ファイル書き込みの場合のメソッドもあります。

```java
// Pathにデータを書き込む
Files.write(path, content);
```

write()メソッドの引数として渡せるのは`byte[]`、`List<String>`などがあります。また、コピーの場合のようにオプションが指定できます。こちらのオプションではファイルが存在する場合上書きするか、追記するかを選べるので場合によってはcopy()と分けて使えます。

```java
// オプション指定(追記)
StandardOpenOption option = StandardOpenOption.APPEND;
Files.write(path, content, option);
```

### ファイルの読み込み

書き込みが文字列かbyte[]で分けられているように、読み込みも同じ形でファイルを取得できるメソッドがあります。文字列取得の場合、シンタックスシュガーとして結果物がStreamかListかくらいの違いがあります。

```java
// 文字列として全行を読み込む
Stream<String> lines = Files.lines(path);
List<String> liness = Files.readAllLines(path);

// byte[]として読み込む
byte[] bytes = Files.readAllBytes(path);
```

Fileがそうであるように、Pathもまたファイルではなくディレクトリになれるので、Filesのメソッドもそれに対応しています。list()メソッドではディレクトリないのエントリをPathとして取得してStreamを生成します。

```java
// ディレクトリ内のエントリを要素として持つStream取得
Stream<Path> files = Files.list(path);
```

### I/Oとの組み合わせで使う

先に述べたように、Filesのメソッドの一部は昔ながらのI/Oと組み合わせて使えるものもあります。その一部を紹介します。

```java
// 読み込みの場合
InputStream is = Files.newInputStream(path);
BufferedReader br = Files.newBufferedReader(path);

// 書き込みの場合
OutputStream os = Files.newOutputStream(path);
BufferedWriter bw = Files.newBufferedWriter(path);
```

もちろんOpenOptionの指定もできます。

```java
// ファイルがない場合は作成する
StandardOpenOption option = StandardOpenOption.CREATE;
InputStream is = Files.newInputStream(path, option);
```

## 最後に

どうでしたか。同じ機能をするだけならあまり使いたくなるメリットはないように見えるかも知れませんが、実際使ってみると、ENUMによるオプション指定でやりたいことが明確となって、コードの量も減らすことができる便利なクラスを提供するのがNIOだと思います。特にFileはそのまま使うとしても、Filesのメソッドは便利かつ強力なので、皆さんにぜひお勧めしたいものです。

他にもFilesクラスには双方通信ができるというChannelクラスを提供するメソッドや、ファイルの属性、シンボリックリンクを取得したり指定したPathがディレクトリかを確認したり、二つのPathが同じファイルかをチェックするなど便利なメソッドが多いので、ぜひ使ってみてください。

では、また！
