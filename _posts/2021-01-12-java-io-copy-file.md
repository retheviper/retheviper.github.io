---
title: "今更なI/Oの話"
date: 2021-01-12
categories: 
  - java
photos:
  - /assets/images/sideimage/java_logo.jpg
tags:
  - java
  - io
  - nio
---

以前、Java 1.7から導入されたNIOに関してのポストを書いたことがありますが、いまだにJavaにおけるファイルのI/Oに関しては悩ましいところが多いです。恥ずかしいことですが、[Newsroom](https://ja.wikipedia.org/wiki/%E3%83%8B%E3%83%A5%E3%83%BC%E3%82%B9%E3%83%AB%E3%83%BC%E3%83%A0)のセリフでもあるように、「問題を解決する第一歩はそこに問題があるということを認識すること(First step in solving any problem is recognizing there is one)」ですね。なので、今までの自分が書いたコードを振り返り、どのように書いた方が良いかを反省することにしました。

なので今回は、今までなんとなく使ってきたコードたちを振り返り、なるべくどのような方法をとった方が良いかを考えてみようと思います。ただ、考えられる全てのケースを網羅するのは難しいと思うので、この度はあくまで`Javaのコードでファイルをコピーする`場合に限ります。なので、考えてみたいこと(検証対象)は以下の通りになります。

1. InputStreamとOutputStreamはどう作った方がいいか
  1. FileInputStreamとFileOutputStreamを使うか
  1. Filesのメソッドを使うか
1. ファイルコピーはどのような方法を使った方がいいか
  1. InputSteamからOutpuStreamへ書くか
      1. readAllBytes()
      1. transferTo()
  1. Filesのメソッドを使うか

今回はこれらの疑問について、よく使われているファイルコピーのコードを一つ一つ見ていきながら、考えてみたいと思います。

## InputStreamとOutputStreamはどう作るべきか

まずはInputStreamとOutputStreamです。今も多くの場合、メモリー問題を考えて、ファイルはなるべくStreamとして扱っているのではないかと思います。特に今のJavaがよく使われている分野はWebアプリケーションですが、そのWebアプリケーションを作るための代表的なフレームワークであるSpringでもファイルのアップロードやダウンロードはStreamの形式となっていますし、ローカルのものかネットワーク越しのものかを問わずファイルのデータを取り扱えるという意味ではFileやPathというオブジェクトを使う場合に比べ汎用性という面でも良さそうな気がします。

ただ、ローカルでファイルをコピーするために、InputStreamやOutputStreamを生成する方法は、Java 1.7以降だと二つの方法があります。一つはFileオブジェクトから`FileInputStream`・`FileOutputStream`を生成する方式であり、もう一つはPathオブジェクトとFilesクラスを利用して生成する方法ですね。

なるべくこれからのコードはNIOを使って書きたいと思っていますが、本当にそれだけで十分か、既存のコード(FileInputStreamとFileOutputStreamを利用する)までもNIOのものに変える必要があるかをまず確認してみたいです。

### コピーの方式

まずは、JavaでInputStreamとOutputStreamを利用して、ファイルをコピーするコードから見ていきましょう。

私を含め、初めてJavaに触れた多くの方々接することとなるファイルコピーのコードは、おそらく以下のようなものではないかと思います。いわば、最も一般的で、オーソドックスな形とも言えるでしょう。

```java
// byte[]を利用した例
private void copy(File source, File dest) throws IOException {
    InputStream is = null;
    OutputStream os = null;
    try {
        is = new FileInputStream(source);
        os = new FileOutputStream(dest);
        byte[] buffer = new byte[8192];
        int length;
        while ((length = is.read(buffer)) > 0) {
            os.write(buffer, 0, length);
        }
    } finally {
        is.close();
        os.close();
    }
}

// BufferedStreamを利用した例
private void copy(File source, File dest) throws IOException {
    int buff = 8192;
    InputStream is = null;
    OutputStream os = null;
    try {
        is = new BufferedInputStream(new FileInputStream(source), buff);
        os = new BufferedOutputStream(new FileOutputStream(dest), buff);
        int length;
        while ((length = is.read()) > 0) {
            os.write(length);
        }
    } finally {
        is.close();
        os.close();
    }
}
```

ここでまず、`FileInputStream`は`Files.newInputStream`、`FileOutputStream`は`Files.newOutputStream`に代替できます。まず目立つ違いとしては、`FileInputStream`・`FileOutputStream`は引数として`File`を取り、`Files.newInputStream`・`Files.newOutputStream`は引数として`Path`を取るという点がありますね。ただ、この違いは、`File`と`Path`の変換が自由なので、あまり決定的な違いとは言えません。つまり、どちらの方法にも簡単に切り替えができるということですね。

一見、`Files`クラスからInputStreamとOutputStreamのインスタンスを生成した方が、より最新のAPIを使っているので性能の面で良さそうな気はします。しかし、JavaのNIOは、必ず性能面で既存のIOと比べ優位にあるわけではないですね。実際、ファイルのI/Oに関しては、NIOを使ってもBlockingモードとしてしか動かないので、あまり性能は変わらないという話もあります。

そういう場合は、特に問題を起こしてないのに、あえて既存のコードをNIOに切り替える必要は無くなりそうな気もします。しかし、本当にそれで大丈夫でしょうか。

### FileInputStreamとFileOutputStreamの問題

実際は、そうでもないようです。`FileInputStream`・`FileOuputStream`には性能とは別の問題があります。GCによりアプリケーション全体が長くポーズの状態になる可能性があるということです。

#### finalize()のオーバライド問題

GCによりアプリケーション全体がポーズされるということは、つまり、メモリがフルになるということです。ファイルI/Oで、InputStreamとOutputStreamを使ってメモリがフルになるということは、ちゃんと`close()`されてないことですね。なので、`単純にclose()すれば良いだけなのでは？`と思われます。

しかし、本当の問題は`FileInputStream`・`FileOuputStream`のソースコードにあります。この二つのクラスは、`finalize()`メソッドをオーバーロードしていて、ちゃんと`close()`してもメモリー上にデータが残ってしまう可能性があるのです。この問題は、[こちらの記事](https://dzone.com/articles/fileinputstream-fileoutputstream-considered-harmful)に説明されてある通り、Jenkinsでも[問題視されたことがあり](https://issues.jenkins.io/browse/JENKINS-42934)、OpenJDKでも[finalize()を消す必要がある](https://bugs.openjdk.java.net/browse/JDK-8212050)と指摘されたことがあります。

JDKの対応としては、`FileInputStream`・`FileOuputStream`の`finalize()`はJava 9から`Deprecated`となり、Java 10からは別の実装を加えることで問題を解決していますが、Java 1.7や1.8を使う場合は依然として問題が起こり得るということになりますね。

なので、これからはなるべく`FileInputStream`・`FileOutptStream`の利用は避けるようにする必要があると思います。習慣は怖いですので。

## ファイルコピーはどのような方法を使った方がいいか

今までの結論で、InputStream及びOuputStreamのインスタンスはNIOを使うことにします。したがって前述のコードは以下のように直すことができますね。

```java
// byte[]を利用した例
private void copy(Path source, Path dest) throws IOException {
    InputStream is = null;
    OutputStream os = null;
    try {
        is = Files.newInputStream(source); // FileInputStreamを使わない
        os = Files.newOutputStream(dest); // FileOutputStreamを使わない
        byte[] buffer = new byte[8192];
        int length;
        while ((length = is.read(buffer)) > 0) {
            os.write(buffer, 0, length);
        }
    } finally {
        is.close();
        os.close();
    }
}

// BufferedStreamを利用した例
private void copy(Path source, Path dest) throws IOException {
    int buff = 8192;
    InputStream is = null;
    OutputStream os = null;
    try {
        is = new BufferedInputStream(Files.newInputStream(source), buff); // FileInputStreamを使わない
        os = new BufferedOutputStream(Files.newOutputStream(dest), buff); // FileOutputStreamを使わない
        int length;
        while ((length = is.read()) > 0) {
            os.write(length);
        }
    } finally {
        is.close();
        os.close();
    }
}
```

### try-with-resource

InputStreamやOutputStreamは最後に`close()`しないと、すでに使ったものでもメモリ上にデータが残ってしまいますね。なのでfinallyブロックでクローズするのが一般的かなと思いますが、こうした場合、finallyブロックでも追加の例外処理が必要になるケースもありますし、毎回`close()`するのは忘れられる可能性もあるので危険です。

なのでJava 1.7からは`AutoCloseable`と`try-with-resource`が導入され、以下のようにより簡潔かつ安全なコードを書くことができるようになりました。例えば上記のコードは、`try-with-resource`を使うと以下のようなコードに代替できますね。

```java
// byte[]を利用した例
private void copy(Path source, Path dest) throws IOException {
    try (InputStream is = Files.newInputStream(source);
        OutputStream os = Files.newOutputStream(dest)) {
        byte[] buffer = new byte[8192];
        int length;
        while ((length = is.read(buffer)) > 0) {
            os.write(buffer, 0, length);
        }
    }
}

// BufferedStreamを利用した例
private void copy(Path source, Path dest) throws IOException {
    int buff = 8192;
    try (InputStream is = new BufferedInputStream(Files.newInputStream(source), buff);
        OutputStream os = new BufferedOutputStream(Files.newOutputStream(dest), buff)) {
        int length;
        while ((length = is.read()) > 0) {
            os.write(length);
        }
    }
}
```

try-with-resourceでは、既存の方式と比べメリットしかないので、これは必ず使うことにします。

### readAllBytes()

次に考えられるのは、ファイルコピーでのBufferです。以上の例では、`byte[]`を使うか、`BufferedInputStream`・`BufferedOutputStream`を使っていますが、これは性能のためのものであるということは皆さんもご存知のはずなので、Bufferについては割愛します。

我々が知る限り、Bufferのサイズが大きければ大きいほど、性能はよくなります。なら、メモリが許容する限り、できるだけ大きいサイズのBufferを指定したら自然に性能はマシンが出せる最大限となるはずです。

そして、Java 9からは、InputStreamを一気に全部読み込み、`byte[]`として返す`readAllBytes()`というメソッドができました。このメソッドを使うと、`Integer.MAX_VALUE`サイズの`byte[]`を生成してInputStreamを全部読み込むことができます。理論的にはこれを使ったらファイルコピーもあっという間にできそうですね。

しかし、考えなくてはならないのが、そうやって読み込んだデータはメモリ上に残ってしまうということです。例えば複数のユーザが使っているWebアプリケーションで、数GBに達するファイルをアップロードする場合が予想されるのに、`readAllBytes()`を使ったらメモリはすぐ足りなくなるでしょう。いくらファイルコピーが早くなるとしても、同時に複数のユーザがファイルをアップロードする場合があれば、一周でのもメモリ上に大量のファイルデータが詰まってしまう可能性があるので、あまり良くない選択になります。なので、なるべく`readAllBytes()`の仕様は控えるべきでしょう。


### transferTo()

Java 9からは追加されたメソッドのうちには、InputStreamにはより簡単にOutputStreamにデータを転送することのできる`transferTo()`というメソッドもあります。`try-with-resource`に加え、`transferTo()`を使うとさらに簡潔なコードでファイルのコピーができるようになります。例えば以下のようなものですね。

```java
private void copy(Path source, Path dest) throws IOException {
    try (InputStream is = Files.newInputStream(source);
        OutputStream os = Files.newOutputStream(dest)) {
        is.transferTo(os);
    }
}
```

ソースコードを見るとわかることですが、`transferTo()`ではデフォルトのBufferサイズで作ったbyte[]を使ってコピーをしているので、デフォルト値のBuffer(`8192`バイト)を使う場合は、Bufferの指定もいらなくなるのが魅力的です。以下はソースコードです。

```java
public long transferTo(OutputStream out) throws IOException {
    Objects.requireNonNull(out, "out");
    long transferred = 0;
    byte[] buffer = new byte[DEFAULT_BUFFER_SIZE]; // 8192
    int read;
    while ((read = this.read(buffer, 0, DEFAULT_BUFFER_SIZE)) >= 0) {
        out.write(buffer, 0, read);
        transferred += read;
    }
    return transferred;
}
```

ただ気になるのは、`transerTo()`を使う場合は本当にBufferedが要らないかという点です。例えばInputStreamを`BufferedInputStream`でラップすると、せめてファイルを読み込む速度は上がるのではないかという疑問が湧いてきます。とにかく、もしものことなので、簡単なベンチマークも実施してみました(実はやってみたかっただけですが)。10GBほどのファイルを生成し、以下のケースでテストしてみました。

- InputStream → OutputStream
- BufferedInputStream → OutputStream
- InputStream → BufferedIOutputStream
- BufferedInputStream → BufferedIOutputStream

そしてコードは以下の通りです。

```java
@State(Scope.Benchmark)
@BenchmarkMode(Mode.AverageTime)
public class StreamBufferTest {

    private Path source;

    private Path output = Path.of("/Users/retheviper/temp/benchmarkOutput");

    // テスト用のファイルを作成する
    @Setup
    public void init() throws IOException {
        final String path = "/Users/retheviper/temp/benchmarkSource";
        final RandomAccessFile file = new RandomAccessFile(path, "rw");
        long size = (1024 * 1024 * 1024) * 10L; // 10GB
        file.setLength(size);
        this.source = Path.of(path);
    }

    @Benchmark
    public void noBuffer() throws IOException {
        try (InputStream in = Files.newInputStream(source);
             OutputStream out = Files.newOutputStream(output, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
            in.transferTo(out);
        }
    }

    @Benchmark
    public void withInputBuffer() throws IOException {
        try (InputStream in = new BufferedInputStream(Files.newInputStream(source));
             OutputStream out = Files.newOutputStream(output, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
            in.transferTo(out);
        }
    }

    @Benchmark
    public void withOutputBuffer() throws IOException {
        try (InputStream in = Files.newInputStream(source);
             OutputStream out = new BufferedOutputStream(Files.newOutputStream(output, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING))) {
            in.transferTo(out);
        }
    }

    @Benchmark
    public void withBothBuffer() throws IOException {
        try (InputStream in = new BufferedInputStream(Files.newInputStream(source));
             OutputStream out = new BufferedOutputStream(Files.newOutputStream(output, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING))) {
            in.transferTo(out);
        }
    }
}
```

そしてベンチマーク結果は、以下の通りです。

```dos
Benchmark                          Mode  Cnt   Score   Error  Units
StreamBufferTest.noBuffer          avgt   25  13.055 ± 0.583   s/op
StreamBufferTest.withInputBuffer   avgt   25  13.302 ± 0.460   s/op
StreamBufferTest.withOutputBuffer  avgt   25  13.663 ± 0.535   s/op
StreamBufferTest.withBothBuffer    avgt   25  12.668 ± 0.355   s/op
```

予想通り、`transferTo()`でのコピーの場合、BufferedInputStreamやBufferedOutputStreamを使わなくても性能はあまり変わりありませんでした。単純なファイルコピーではなかったり、InputStreamからOutputStreamというデータの転送ではない場合はまた必要となりそうな気はしますが、このメソッドが使える場合はあまり意識しなくても良さそうですね。

### Files.copy()がいい？

Java 1.7では、`Files.copy()`を通じて以下のファイルコピーができるようになっています。

- InputStream → Path
- Path → OutputStream
- Path → Path

そして一部では、JavaのNIOはネイティブコードで書かれてあるので、InputStreamからOutputStreamへの書き込みよりはFiles.copy()の方が性能がいいと言われる場合もありました。この話が本当さとすると少なくともローカルのファイルを扱う場合、InputStreamからOutputStreamへの書き込みよりはPathを使ったコピーが良さそうな気がします。

#### ソースで確認する

コードが違うと言われたからには、直接確認した方がいいですね。早速、以上であげた三つのメソッドのソースコードを確認することにします。まずは、`InputStream → Path`と`Path → OutputStream`です。こちらはシンプルに、`Path`からOuputStreamもしくはInputStreamを生成し、`transferTo()`を使ってコピーすることとなっています。(ただ、これはJava 11基準のソースコードなので、Java 9以前の場合は違うコードの可能性があります)

```java
// InputStream → Path
public static long copy(InputStream in, Path target, CopyOption... options) throws IOException {
    // コピー以外の処理は省略

    OutputStream ostream;
    try {
        ostream = newOutputStream(target, StandardOpenOption.CREATE_NEW,
                                            StandardOpenOption.WRITE);
    } catch (FileAlreadyExistsException x) {
        if (se != null)
            throw se;
        // someone else won the race and created the file
        throw x;
    }

    // do the copy
    try (OutputStream out = ostream) {
        return in.transferTo(out);
    }
}

// Path → OutputStream
public static long copy(Path source, OutputStream out) throws IOException {
    // ensure not null before opening file
    Objects.requireNonNull(out);

    try (InputStream in = newInputStream(source)) {
        return in.transferTo(out);
    }
}
```

ただ、やはり`Path → Path`の場合は全く違うコードになっています。コピー元とコピー先が同じファイルシステムの場合は[FileSystemProvider](https://docs.oracle.com/javase/7/docs/api/java/nio/file/spi/FileSystemProvider.html)を使い、そうではない場合はCopyMoveHelperを使うことになっていますね。

```java
// Path → Path
public static Path copy(Path source, Path target, CopyOption... options) throws IOException {
    FileSystemProvider provider = provider(source);
    if (provider(target) == provider) {
        // same provider
        provider.copy(source, target, options);
    } else {
        // different providers
        CopyMoveHelper.copyToForeignTarget(source, target, options);
    }
    return target;
}
```

ここで`CopyMoveHelper.copyToForeignTarget()`の場合は、結果的に`Files.copy(InputStream, Path)`を呼ぶことになるのですが、前者の場合は全く違う方式になるのでやはり性能の差が発生する可能性もありそうですね。整理すると、`同じシステム内で、Path → Pathでコピーする場合だけ性能がよくなる可能性がある`ということですね。

ここはまた検証が必要なところなので、またベンチマークを実施してみました。もちろんファイルシステムの違いにより結果は変わる可能性があるので、これが絶対的だとは言えませんが、何らかの違いがあるかも知れません。他の`Files.copy()`メソッドは実質的に`transferTo()`と同じものなので、今回の比較は`InputStream → OutputStream`と`Path → Path`だけになります。また、比較のためのテストケースが少ないので、今回は`transferTo()`のベンチマークよりもファイルサイズを大きくしてみました。以下は、そのテストコードです。

```java
@State(Scope.Benchmark)
@BenchmarkMode(Mode.AverageTime)
public class StreamCopyTest {

    private Path source;

    private Path output = Path.of("/Users/youngbinkim/Downloads/benchmarkOutput");

    // テスト用のファイルを作成する
    @Setup
    public void init() throws IOException {
        final String path = "/Users/youngbinkim/Downloads/benchmarkSource";
        final RandomAccessFile file = new RandomAccessFile(path, "rw");
        long size = (1024 * 1024 * 1024) * 10L; // 10GB
        file.setLength(size);
        this.source = Path.of(path);
    }

    @Benchmark
    public void streamToStream() throws IOException {
        try (InputStream in = Files.newInputStream(source);
             OutputStream out = Files.newOutputStream(output, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
            in.transferTo(out);
        }
    }

    @Benchmark
    public void pathToPath() throws IOException {
        Files.copy(source, output, StandardCopyOption.REPLACE_EXISTING);
    }
}
```

そして、ベンチマークの結果は以下の通りです。

```dos
Benchmark                      Mode  Cnt   Score   Error  Units
StreamCopyTest.streamToStream  avgt   25  12.128 ± 0.331   s/op
StreamCopyTest.pathToPath      avgt   25  12.257 ± 0.342   s/op
```

10GBのファイルでは誤差範囲以内の結果となったので、ファイルサイズだけを100GBに変えて同じくベンチマークを実施してみました。その結果は以下です。

```dos
Benchmark                      Mode  Cnt    Score   Error  Units
StreamCopyTest.streamToStream  avgt   25  160.046 ± 2.538   s/op
StreamCopyTest.pathToPath      avgt   25  153.505 ± 2.662   s/op
```

100GBになってからやっと差が見えてくる、ということになりますが、結論としてはやはり、`Path → Path`の方が早いという結果となりました。機会があれば、複数スレッドによるI/OやOSの違いまで考慮したベンチマークを書きたいものですが、とりあえずは予想通りの結果となったということで。

## 最後に

以上のソースコードとベンチマークでわかったことをまとめると、Javaでのファイルコピーは、とりあえず以下のようなことで結論づけができそうです。

- Java 1.7以上の場合
  - `FileInputStream`・`FileOutputStream`の代わりに`Files.newInputStream`・`Files.newOutputStream`を使う
  - `try-with-resource`を使う
  - コピー元とコピー先のどちらも同じファイルシステム上のパスであれば、両方`Path`が引数の`Files.copy()`を使う
- Java 9以上の場合
  - Bufferサイズが`8192`の場合は`transferTo()`を使う
    - `transferTo()`を使う場合、`BufferedInputStream`・`BufferedOutputStream`は必須ではない

多くの場合、エンタープライズアプリケーションはLTSである1.8や11を使うと思われるので、実質的には以上に並べた項目全てが当てはまると言えましょう。

かなり今更な感があるポストとなりましたが、個人的には自分の納得できる形で整理でき、スッキリしました。こうやって何気なく、「そう教わったから」使っていたコードを振り返ってみるのも良い勉強になりますね。次もまた、こうやってソースコードやベンチマークによる検証をやってみたいなと思います。

では、また！