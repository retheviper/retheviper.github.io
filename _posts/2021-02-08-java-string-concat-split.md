---
title: "今更な文字列操作の話"
date: 2021-02-08
categories: 
  - java
photos:
  - /assets/images/sideimage/java_logo.jpg
tags:
  - java
  - string
---

もうこれで3回目の、「今更なシリーズ」です。このシリーズ自体は、またベンチマークとともに戻ってきました。さて、今回のテーマはJavaによる文字列の操作となりますが、その中でも`連結(Join)`と`分割(split)`について述べたいと思います。最初は単純に、文字列の分割は`String.split()`でやるしかないのに、連結の場合は`String.join()`とか`Collectors.joining()`とか、色々あるなと思ったのがきっかけです。同じことが複数のAPIでできるのは、単純に[シンタックスシュガー](https://ja.wikipedia.org/wiki/糖衣構文)な場合もありますが、実際は全く実装が違うケースもありますね。特に、Javaのように長い間使われてきた言語こそそのようなケースが多いかと思います。

また、単純なシンタックスシュガーに近い場合でも、その前後のコードや可読性など、周りの様相を考慮して適切なものを選ぶ必要がある場合もあります。例えば、以前紹介したInputStreamの`transferTo()`がそのようなケースですね。なので、一つのAPIを使う場合は、できればその実装がどうなっているかを確認してみるのも、良いコードを書くための工夫となるのではないかと思います。

ではでは、早速本題に入りましょう。まずは文字列の連結からです。

## Concatenating

文字列の連結といっても、色々なケースがありますね。そしてそういった場合は、`String.concat()`、`String.format()`などさまざまな方法があって、それら全部に対してシナリオを想定し検証するということは難しいと思います。なので今回は、「文字列の配列もしくはCollectionを、区切り文字でつないで一つの文字列にする」というケース一つに限定して述べたいと思います。

Javaでの区切り文字を使った文字列の連結には、主に以下のような方法が考えられます。これら一つ一つのAPIの特徴と、実際の使い方を持って比較して見た後、いつもの通りベンチマークをするということで性能を測定することとします。(`+`を使って文字列を繋ぐケースは、あまりよろしくないと思うのでケース外としています)

- `String.join()`
- `StringJoiner`
- `StringBuffer`
- `StringBuilder`
- `Collectors.joining()`

### StringBuffer || StringBuilder

純粋に、Collectionや配列になっている複数の文字列を連結する場合もあるとは思いますが、普通、文字列の連結が必要となる場合では、「とある規則によって」という条件がつくケースが多いかなと思います。例えば、ダッシュ(-)、アンダースコア(_)、カンマ(,)などで並ぶようにですね。そしてこのような規則がある場合、`StringBuffer`や`StringBuilder`を使った方法は他と比べて少し不利です。なぜなら、最後に区切り文字(delimiter)が付かないように制御するにはかなりコードの書き方に注意しなければならないからです。以下のコードが、そのようなケースです。

```java
// StringBufferを使う例
List<String> list = List.of("A", "B", "C");
String delimiter = ", ";

StringBuffer buffer = new StringBuffer();
// Listの要素と区切り文字を足す
for (String string : list) {
    buffer.append(string);
    buffer.append(delimiter);
}

String result = buffer.toString(); // A, B, C,
```

あえて、文字列の末尾に区切り文字が付かないようにするとしたら、おそらくこういうコードを書く必要があるでしょう。

```java
List<String> list = List.of("A", "B", "C");
String delimiter = ", ";
int limit = list.size() - 1;

StringBuffer buffer = new StringBuffer();
// Listの要素と区切り文字を足す(最後のインデックスの前まで)
for (int i = 0; i < limit; i++) {
    buffer.append(list.get(i));
    buffer.append(delimiter);
}
// 最後の要素を足す
buffer.append(list.get(limit));

String result = buffer.toString(); // A, B, C
```

こういう問題があるのに比べて、他の方法(`String.join()`、`StringJoiner`、`Collectors.joining()`)は、区切り文字が最後の要素の後に付かないので、よりシンプルなコードで書けるというメリットがありますね。なので、結論として`StringBuffer`や`StringBuilder`は、少なくとも「とある規則によって」複数の文字列を連結する場合には可読性という観点からしてあまり良い選択肢ではないということがわかります。

### StringJoiner

`StringBuffer`と`StringBuilder`ではループで文字列を連結して行くので、ループの中で条件分岐など他の処理も必要な場合に使えるのでは？と思われるかも知れません。しかし、そういう場合でも、`StringJoiner`を使ったほうが良いですね。なぜなら、使い方はほぼ変わらなく、特に操作をしなくても常に末尾に区切り文字が付かないからです。以下は、`StringJoiner`のもっともベーシックな使い方のコードとなります。

```java
List<String> list = List.of("A", "B", "C");
// 区切り文字を指定してインスタンスを作る
StringJoiner joiner = new StringJoiner(", ");
// あとは要素を足していく
for (String string : list) {
    joiner.add(string);
}
String result = joiner.toString(); // A, B, C
```

また、`StringJoiner`を使った場合は、PrefixとSuffixの指定も可能です。これらを指定した場合、文字列の先頭と末尾に指定したPrefixとSuffixが付くようになります。

```java
List<String> list = List.of("A", "B", "C");
// 区切り文字とPrefix、Suffixまで指定する
StringJoiner joiner = new StringJoiner(", ", "[", "]");
for (String string : list) {
    joiner.add(string);
}
String result = joiner.toString(); // [A, B, C]
```

使い方だけ見ても、区切り文字を持って文字列を繋ぐ場合は`StringBuffer`や`StringBuilder`より`StringJoiner`の方がより簡単であるということが分かります。

#### 番外：StringJoinerの実装

ついでに、StringJoinerはどんなコードで書かれているかを見ていきたいと思います。まずは`add()`ですが、これは面白くも、`ArrayList`の実装と似たような感じになっています。`StringJoiner`クラスはフィールドとして`String[]`を持っていて、`add()`がよばれる度にそれより大きいコピーを作っていく形です。

```java
public StringJoiner add(CharSequence newElement) {
    final String elt = String.valueOf(newElement);
    if (elts == null) {
        elts = new String[8];
    } else {
        if (size == elts.length)
            elts = Arrays.copyOf(elts, 2 * size);
        len += delimiter.length();
    }
    len += elt.length();
    elts[size++] = elt;
    return this;
}
```

そして`toString()`では、フィールドの`String[]`をループしながら、区切り文字とともに繋げて行くのが分かります。少し変わっているのは、性能を意識しているからか、`char[]`として文字列をつめた後から新しく`String`のインスタンスを作って返しているというところですね。

```java
public String toString() {
    final String[] elts = this.elts;
    if (elts == null && emptyValue != null) {
        return emptyValue;
    }
    final int size = this.size;
    final int addLen = prefix.length() + suffix.length();
    if (addLen == 0) {
        compactElts();
        return size == 0 ? "" : elts[0];
    }
    final String delimiter = this.delimiter;
    final char[] chars = new char[len + addLen];
    int k = getChars(prefix, chars, 0);
    if (size > 0) {
        k += getChars(elts[0], chars, k);
        for (int i = 1; i < size; i++) {
            k += getChars(delimiter, chars, k);
            k += getChars(elts[i], chars, k);
        }
    }
    k += getChars(suffix, chars, k);
    return new String(chars);
}
```

### String.join()

`String.join()`は、`InputStream.transferTo()`のように、あくまでシンタックスシュガーとして存在するものだと言えます。以下は実際のコードです。

```java
public static String join(CharSequence delimiter,
        Iterable<? extends CharSequence> elements) {
    Objects.requireNonNull(delimiter);
    Objects.requireNonNull(elements);
    StringJoiner joiner = new StringJoiner(delimiter);
    for (CharSequence cs: elements) {
        joiner.add(cs);
    }
    return joiner.toString();
}
```

引数に対するNullチェック以外は、Prefix・Suffixなしの`StringJoiner`での連結になっているということを確認できます。なので、より短いコードを書きたい場合は`StringJoiner`を使うよりも、こちらの方が便利ではありますね。

### Collectors.joining()

文字列の連結で`Stream`を利用する場合、他にも`filter()`、`map()`、`peek()`など、さまざまな処理をメソッドチェイニングで書けるというところが魅力的ですね。個人的には、処理の役割と目的・影響範囲が明確に見えるので、`Stream`による処理を好んで使っています。ただ、以前のポストでも書いたことがありますが、多くの場合に`Stream`は伝統的なループより性能面で不利ですので、時と場合によって適切に選ぶべきでしょう。

さて、そんな`Stream`ですが、中の実装はどうなっているのでしょうか。`Collectors.joining()`の場合、以下のような実装となっています。結局は、`StringJoiner`を内部で使っているだけですので、`String.join()`・`StringJoiner`と比べては、`Stream`によるコードの変化や性能に影響されるだけと言えるでしょう。

```java
public static Collector<CharSequence, ?, String> joining(CharSequence delimiter,
                                                            CharSequence prefix,
                                                            CharSequence suffix) {
    return new CollectorImpl<>(
            () -> new StringJoiner(delimiter, prefix, suffix),
            StringJoiner::add, StringJoiner::merge,
            StringJoiner::toString, CH_NOID);
}
```

### toString()

実は、Collectionの場合(`List<String>`)は、もっと簡単に文字列を作る方法がありますね。`toString()`を呼ぶことで、簡単にカンマ区切りの文字列が出来上がります。ただ、そうして文字列を作った場合、先頭と末尾に`[]`が入ってしまうので、場合によってはそれらを取り消すか、`substring()`で抽出するかの追加的な処理が必要となりますね。以下は、`substring()`を利用して`[]`の中の文字列だけを切り取るサンプルとなります。

```java
List<String> list = List.of("A", "B", "C");
String toString = list.toString(); // [A, B, C]
String result = toString.substring(1, toString.length() - 1); // A, B, C
```

また、もし区切り文字がカンマではない場合は、とりあえず`toString()`で文字列に変換した結果の文字列から、更に`replace()`を呼び出し、区切り文字だけを入れ替えるというやり方でも対応はできます。ただ、これは非常に非効率的なやり方ではあります。なぜなら、`replace()`のコードをみると、結局はループの中で`StrinbBuilder`を使って新しく作り出すような構造となっているからです。実際のコードは、以下の通りです。

```java
public String replace(CharSequence target, CharSequence replacement) {
    String tgtStr = target.toString();
    String replStr = replacement.toString();
    int j = indexOf(tgtStr);
    if (j < 0) {
        return this;
    }
    int tgtLen = tgtStr.length();
    int tgtLen1 = Math.max(tgtLen, 1);
    int thisLen = length();

    int newLenHint = thisLen - tgtLen + replStr.length();
    if (newLenHint < 0) {
        throw new OutOfMemoryError();
    }
    StringBuilder sb = new StringBuilder(newLenHint);
    int i = 0;
    do {
        sb.append(this, i, j).append(replStr);
        i = j + tgtLen;
    } while (j < thisLen && (j = indexOf(tgtStr, j + tgtLen1)) > 0);
    return sb.append(this, i, thisLen).toString();
}
```

少なくとも区切り文字がカンマではない場合は、`toString()`と`replace()`での文字列の生成よりは、他の方法をとったほうが性能面では有利ではないか、という推測が可能です。もちろん、要素数という変数があるので、実際の性能は測ってみないとわからないものですが…

### ベンチマークしてみる

では、文字列を連結するために使える色々なAPIと、その特徴を簡単に把握できたので、次に確認したいのは、やはり性能です。特に気になるのは、`String.join()`や`Collectors.joining()`でも結局は内部で`StringJoiner`を使っているというところです。それはつまり、`StringBuffer`や`StringBuilder`よりも`StringJoiner`が性能で有利だから、でしょうか。

これらのAPIを利用して、実際のアプリケーションに使われるビジネスロジックのコードを書く立場としては、それはコードを簡単に書ける方が良いのは当然ですが、そもそもこういうAPIの場合は、手間を省けるために性能は良くても複雑なコードで実装する可能性もあるのですので、疑問になります。しかも、多くの場合、文字列の操作では`StringBuilder`が早いと言われていますので、ますます性能差というのが気になってきます。なので、いつもの通りにベンチマークを実施してみました。

ベンチマークは、カンマ区切りで文字列を連結する例として作成しています。以下がそのコードです。

```java
@State(Scope.Benchmark)
public class StringConcatTest {

    private static final String DELIMITER = ", ";

    private List<String> target;

    @Setup
    public void init() {
        final DecimalFormat format = new DecimalFormat("0000000");
        this.target = IntStream.rangeClosed(0, 1000000).mapToObj(i -> format.format(i)).collect(Collectors.toList());
    }

    @Benchmark
    public void toString(final Blackhole bh) {
        final String toString = target.toString();
        bh.consume(toString.substring(1, toString.length() - 1));
    }

    @Benchmark
    public void stringJoin(final Blackhole bh) {
        bh.consume(String.join(DELIMITER, target));
    }

    @Benchmark
    public void collectorsJoining(final Blackhole bh) {
        bh.consume(target.stream().collect(Collectors.joining(DELIMITER)));
    }

    @Benchmark
    public void stringBuffer(final Blackhole bh) {
        final StringBuffer buffer = new StringBuffer();
        final int limit = this.target.size() - 1;
        for (int i = 0; i < limit; i++) {
            buffer.append(this.target.get(i));
            buffer.append(DELIMITER);
        }
        buffer.append(this.target.get(limit));
        bh.consume(buffer.toString());
    }

    @Benchmark
    public void stringBuilder(final Blackhole bh) {
        final StringBuilder builder = new StringBuilder();
        final int limit = this.target.size() - 1;
        for (int i = 0; i < limit; i++) {
            builder.append(this.target.get(i));
            builder.append(DELIMITER);
        }
        builder.append(this.target.get(limit));
        bh.consume(builder.toString());
    }
}
```

そして、結果は以下の通りです。

```dos
Benchmark                            Mode  Cnt   Score   Error  Units
StringConcatTest.toString           thrpt   25  41.445 ± 0.461  ops/s
StringConcatTest.stringJoin         thrpt   25  28.396 ± 0.447  ops/s
StringConcatTest.collectorsJoining  thrpt   25  31.024 ± 1.313  ops/s
StringConcatTest.stringBuffer       thrpt   25  30.570 ± 1.205  ops/s
StringConcatTest.stringBuilder      thrpt   25  45.965 ± 1.736  ops/s
```

この結果からわかるのは、やはり`StringBuilder`の性能は優秀ということですね。ただ、よく知られているように、`StringBuilder`はマルチスレッドを考慮したAPIではないので、スレッドセーフなAPIを使う必要のある環境であるなら、他のAPIを考慮すべきですね。そのような観点からすると、意外と、誤差範囲を踏まえて考えると`String.join()`が`Collectors.joining()`と大差ない性能を見せるという結果となりましたが…このような結果だとすると、気軽に`Stream`を使っても良さそうな気がします。

また、`toString()`の結果は、やはり早いものとなっていますが、ここで`replace()`を挟んだ瞬間性能は半分以下という結果となっています。なので、無理して`toString()`を使う必要はあまりないかな、と思いますね。文字列の連結という目的に合うコードかどうかもすぐわからないし…

もう一つ確かなのは、`StringBuffer`はもう使わなくても良さそうということですね。もうレガシーなコードとして残しておいて、これからはなるべく違うAPIを使うべきなのではないかと思います。

## Split

次に検証したいのは、文字列の分割です。先に述べたのように、文字列の分割は実質、`String.split()`しかない状態と言えますね。`substring()`でもなんとか分割はできるかも知れませんが、その場合はループと条件分岐なしでは話にならないので、そもそも論外かと思います。

ただ、ここで注目したいのは分割した後のことです。`String.split()`の戻り値は`String[]`なので、場合によって`Collection`に変えたくなりますね。なので、どちらかというと「配列をListに」する方法の検証ということとなりますが…とりあえずListをStringに変えてみたので、その逆の場合を考えてみるということで受け止めてくださると幸いです。

### Arrays.asList()

配列をListに変えるもっとも簡単な方法は、`Arrays.asList()`だと思います。コードも簡単ですね。

```java
String string = "A, B, C";
// まずは分割する
String[] array = string.split(", ");
// Listに変える
List<String> list = Arrays.asList(array);
```

ただ、こうやって生成したListのインスタンスは、[Immutable](https://ja.wikipedia.org/wiki/%E3%82%A4%E3%83%9F%E3%83%A5%E3%83%BC%E3%82%BF%E3%83%96%E3%83%AB)となってしまいます。中の要素を操作できないということですね。

もちろん、これは新しいListのインスタンスに要素をコピーすることで解決できます。もっとも簡単なのは、コンストラクタの引数としてListを渡す方法ですね。なので、「配列をMutableなListにする」もっとも簡単な方法は、おそらく以下のようになります。

```java
String string = "A, B, C";
// まずは分割する
String[] array = string.split(", ");
// Listに変える
List<String> list = Arrays.asList(array);
// MutableなListのインスタンスを作成する
List<String> mutableList = new ArrayList<>(list);
```

### Arrays.stream()

配列をListにするまたの方法は、`Stream`を利用することです。文字列の連結でも言及したことなのですが、`Stream`の場合は、`map()`や`filter()`のような中間操作のメソッドを使えるというメリットがありますね。また、`Collectors`のどのメソッドを呼ぶかによって結果として生成されるListがImmutableか、Mutableかを決定できるという面もメリット(可読性という観点で)ではないのかと思います。コードは`Arrays.asList()`と比べて少し複雑になっているように見えるかも知れませんが。

```java
String string = "A, B, C";
// まずは分割する
String[] array = string.split(", ");
// Listに変える(Mutable)
List<String> mutableList = Arrays.stream(array).collect(Collectors.toList());
// Listに変える(Immutable)
List<String> mutableList = Stream.of(array).collect(Collectors.toUnmodifiableList());
```

### ベンチマークしてみる

では、次にまたベンチマークとなります。コード自体は明らかに`Arrays.toList()`の方が簡単だったのですが、MutableなListを作るためにはListを生成した後にさらにインスタンスを作成する必要があるということで、性能面で損する可能性もあるのかなという気がします。なので、以上で紹介した`Arrays.asList()`と`Stream`によるListのインスタンスの作成を、Immutable・Mutableという二つのケースに分けて検証してみました。以下がそのベンチマークのコードです。

```java
@State(Scope.Benchmark)
public class StringSplitTest {

    private static final String DELIMITER = ", ";

    private String target;

    @Setup
    public void init() {
        final DecimalFormat format = new DecimalFormat("0000000");
        this.target = IntStream.rangeClosed(0, 1000000).mapToObj(i -> format.format(i)).collect(Collectors.joining(DELIMITER));
    }

    @Benchmark
    public void arraysAsListImmutable(final Blackhole bh) {
        bh.consume(Arrays.asList(target.split(DELIMITER)));
    }

    @Benchmark
    public void arraysAsListMutable(final Blackhole bh) {
        bh.consume(new ArrayList<>(Arrays.asList(target.split(DELIMITER))));
    }

    @Benchmark
    public void streamCollectImmutable(final Blackhole bh) {
        bh.consume(Arrays.stream(target.split(DELIMITER)).collect(Collectors.toUnmodifiableList()));
    }

    @Benchmark
    public void streamCollectMutable(final Blackhole bh) {
        bh.consume(Arrays.stream(target.split(DELIMITER)).collect(Collectors.toList()));
    }
}
```

そして結果は以下の通りです。

```dos
Benchmark                                Mode  Cnt  Score   Error  Units
StringSplitTest.arraysAsListImmutable   thrpt   25  8.316 ± 1.085  ops/s
StringSplitTest.arraysAsListMutable     thrpt   25  8.133 ± 0.435  ops/s
StringSplitTest.streamCollectImmutable  thrpt   25  6.086 ± 0.312  ops/s
StringSplitTest.streamCollectMutable    thrpt   25  7.247 ± 0.262  ops/s
```

ここでは、`Arrays.asList()`の方が、性能が高い結果となっていますね。途中で何かしらの操作が必要な場合は`Stream`の方が良いかと思いますが、そうではなく、単純に配列をListに変えたい場合はやはり`Arrays.AsList()`を使った方がコードもより簡単で、性能面でも少し優勢ということがわかりました。なので、(いつもそうですが)何をしたいかによって適切なコードを選ぶべきかんと思います。

## 最後に

他にも、文字列の操作に関しては[Baeldungさんの記事](https://www.baeldung.com/java-string-performance)がかなり良かったので、皆さんにもおすすめしたいと思います。最近は特に、アプリケーションでもっともよく扱うデータ型が文字列となっているので、文字列の操作に関してはなるべく性能と可読性という観点から良い書き方を取りたいものです。個人的には`Stream`が大好きなので、なるべくなんでも`Stream`で解決したいものですが…Javaだけでなく、プログラミング言語にとって「どんなケースでも正解」というものはないので。

しかし、Javaに触れてからもう3年も過ぎていますが、今更こんなことを考えるということが恥ずかしい限りですね…次からは、もっと興味深い(そしてこのブログを読まれる方々にも役立つような)ネタを探したいと思います。うまくいくかは少しわからない状態なのですが…！