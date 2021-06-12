---
title: "パターンと一致する文字列かを判定する"
date: 2020-11-22
categories: 
  - java
photos:
  - /assets/images/sideimage/java_logo.jpg
tags:
  - java
  - pattern
  - regular expression
---

一般的に、アプリケーションには要求される業務的な要件やセキュリティの観点から考慮すべきことがあるので、なんらかの機能を作るときはそれが「動くか」だけでなく、任意のロジックが必要となる場合がありますね。なのでその機能が動くにはとある場合で、動くときはとある条件に合わせて処理をする、といった制限が要求されることがあります。

今回のポストも、またそのような業務上の要件から生まれた話です。現在、私が関わっている案件では、EC2で起動するSpring Boot基盤のアプリを作っています。このアプリでは、ファイルのデータとアップロード先のパスを指定すると、S3にアップロードするという単純な機能があり、それは自分の担当となっています。

単純にアップロードパスとデータがあれば、動く機能を作るのは単純です。SpringにはSpring Cloudというフレームワークがあるので、すでに[ResourceLoader](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/ResourceLoader.html)というクラスを持ってファイルアップロードを実現できます。Spring Cloudを使わない場合でも、[AWS SDK](https://aws.amazon.com/jp/sdk-for-java)を使うと簡単に実装ができます。事実、こちらの昨日もアップロード先のパスとファイルだけあれば良いものとなっているので、実装というまででもないですね。

ただ、この機能が呼び出されたとき、渡されたアップロード先のパスが「正しいもの」であるかを確認する必要がありました。つまり、業務上ファイルをS3に格納する際に決まったパスのルールがあって、この機能からはパラメータとして渡されたパスが規定のパターンと一致するかどうかを一度チェックする必要がありました。

渡されたパスが「正しいもの」かどうかをチェックするための機能は、何で作ったら良いでしょうか。そしてどう作った方が良いでしょうか。色々な方法があるかと思いますが、まずここでは自分がどう実装したかを紹介していきたいと思います。

## 文字列のパターンは正規表現で

まず、ファイルのアップロード先(保存先)パスは文字列であり、特定のパターンである必要があります。文字列が特定のパターンで構成されているかどうかの判定は、[正規表現](https://ja.wikipedia.org/wiki/%E6%AD%A3%E8%A6%8F%E8%A1%A8%E7%8F%BE)を使いますね。なので、「正しいもの」としてのパスのパターンは、正規表現としてあらかじめ宣言しておいて、渡されたパラメータがそれと一致するかをチェックすることとします。ただ、Javaでは正規表現を使って文字列のパターンを判定する方法がいくつかありますので、それらのうちにどれを選ぶべきかを考える必要があります。例えば、以下の方法がありますね。

1. `Pattern.matches()`を使う
1. `Pattern`から`Matcher`を取得し使う
1. `Pattern`から`Predicate`を取得し使う
1. `String.matches()`を使う

そして、これらの方法は、以下のコード通りに使うことができます。

```java
// 正規表現の例
String patternRegex = "^[0-9]*$";
// 正規表現で判定したい文字列
String value = "123456789"; 

// Pattern.matches()を使う場合
boolean patternMatches = Pattern.matches(patternRegex, value);

// Matcherを使う場合
Pattern pattern = Pattern.compile(patternRegex);
Matcher matcher = pattern.matcher(value);
boolean matcherFind = matcher.find(); // 部分一致
boolean matcherMatches = matcher.matches(); // 完全一致

// Predicateを使う場合
Pattern pattern = Pattern.compile(patternRegex);
boolean matcherFind = matcher.asPredicate().test(value); // 部分一致
boolean matcherMatches = matcher.asMatchPredicate().test(value); // 完全一致

// String.matches()を使う場合
boolean stringMatches = value.matches(patternRegex);
```

ここで`Matcher`や`Predicate`を使う場合、部分一致を選べられるので、部分一致の場合はこれらを使うしかなさそうです。しかし、完全一致が必要な場合は何を基準に、どれを選ぶべきでしょうか。どれも同じような結果を出すのであれば、より効率的な方法を選びたくなります。そして、この場合、考えられるのは性能です。つまり、どれを使った時にもっとも早く判定の結果を得られるかということです。

## どれも同じなら性能で

前述の通り、文字列が与えられた正規表現のパターンと一致するかどうかを判断する様々な方法があるので、中でももっとも早いのはどれか、測定したいと思います。いわゆるストップウォッチ方式(処理終了時点の時間から、処理開始時点の時間を引く)が簡単ですが、より正確な比較がしたかったためOpenjdkから提供する[JMH](https://openjdk.java.net/projects/code-tools/jmh)を使ってベンチマークを作りました。Java特有の起動が遅い問題で測定に影響が出るのを防ぐためか、何回かのウォーミングアップも含めて測定をしてくれるので、良いですね。

実際にベンチマークを行うため使ったコードは、以下の通りです。

```java
public class RegexTest {

    private static final String PATTERN_REGEX = "^[0-9]*$";

    private static final DecimalFormat DECIMAL_FORMAT = new DecimalFormat("0000000");

    private static final Pattern PATTERN = Pattern.compile(PATTERN_REGEX);

    private static final Predicate PREDICATE = PATTERN.asPredicate();

    private static final Predicate MATCH_PREDICATE = PATTERN.asMatchPredicate();

    private static final List<String> VALUES = IntStream.rangeClosed(0, 9999999).mapToObj(DECIMAL_FORMAT::format).collect(Collectors.toList());

    @Benchmark
    public void patternMatches(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(Pattern.matches(PATTERN_REGEX, value));
        }
    }

    @Benchmark
    public void matcherFind(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(PATTERN.matcher(value).find());
        }
    }

    @Benchmark
    public void matcherMatches(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(PATTERN.matcher(value).matches());
        }
    }

    @Benchmark
    public void predicate(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(PREDICATE.test(value));
        }
    }

    @Benchmark
    public void matchPredicate(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(MATCH_PREDICATE.test(value));
        }
    }

    @Benchmark
    public void stringMatches(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(value.matches(PATTERN_REGEX));
        }
    }
}
```

そして、測定の結果は以下の通りです。実際の出力はメソッド名の順番が違いますが、コードでの並び順に合わせて変えています。

```dos
Benchmark                  Mode  Cnt  Score   Error  Units
RegexTest.patternMatches  thrpt   25  0.591 ± 0.013  ops/s
RegexTest.matcherFind     thrpt   25  1.525 ± 0.022  ops/s
RegexTest.matcherMatches  thrpt   25  1.481 ± 0.030  ops/s
RegexTest.predicate       thrpt   25  2.050 ± 0.182  ops/s
RegexTest.matchPredicate  thrpt   25  1.733 ± 0.236  ops/s
RegexTest.stringMatches   thrpt   25  0.609 ± 0.005  ops/s
```

この結果からして、性能面では確かに`Matcher`か`Predicate`を使った方が良いと言えるでしょう。ただ、ベンチマークの結果としては`Predicate`が誤差を含めても性能がもっとも良いこととなっていますが、`Pattern.asPredicate()`はJava 1.8、`Pattern.asMatchPredicate()`はJava 11から導入されたので、JDKのバージョンに合わせて適切な方を選ぶ必要があります。

ただ、結果だけでなく理由も知りたくなります。性能がよかった`Matcher`と`Predicate`の場合、テストではあらかじめインスタンスを作成しておいたという共通点があります。なので、性能の低い`Pattern.matches()`と`String.matches()`の場合、メソッドが呼び出されるたびにインスタンスを作成しているため遅くなっているのではないか、という推測もできますね。実際はどうか、コードをみていきましょう。

まず`Pattern.matches()`ですが、実際のコードは以下の通りです。

```java
// Pattern.matches
public static boolean matches(String regex, CharSequence input) {
    Pattern p = Pattern.compile(regex);
    Matcher m = p.matcher(input);
    return m.matches();
}
```

これをみると、`Pattern`と`Matcher`のインスタンスがメソッドを呼び出すたびに生成されるということがわかります(実際、`Pattern.compile()`と`Pattern.matcher()`のコードを追ってみるとインスタンスを作成するのがわかります)。なのでこちらが遅くなるのは当然のことですね。

それでは、`String.matches()`の場合はどうか、同じくコードから確認しましょう。実際のコードは以下の通りです。

```java
// String.matches
public boolean matches(String regex) {
    return Pattern.matches(regex, this);
}
```

これもまた、単に`Pattern.matches()`を呼び出しているだけなので、遅いわけですね。ただ一つ違う点は、比較対象となる自分自身のインスタンスが必要なため、`Pattern`とは違ってstaticメソッドではないというところといえますが、これは性能に影響する部分ではないので、ベンチマークでも誤差範囲の中の結果となったと思います。

## 実際のValidatorを作る

では、性能で`Matcher`と`Predicate`が有利であるということがわかったので、あとはこれを利用して、渡されたパスが許容できるものかどうかを判定するValidatorを作ります。今の案件ではJava 11を使うので、`Predicate`を選びました。

パスのパターンは複数あるので、配列やリストとしてパターンを指定して起きます。また、`Predicate`で判定するので、あらかじめ指定したパターンでインスタンスを作成しておいて、判定が必要なときはパターンの配列やリストをループさせて、一致するものがあるかどうかを返すと良いでしょう。この要件から、実際のコードは以下のようになりました(パスの正規表現は、実際の業務とは違うものとなっていますが)。


```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class StorageValidator {

    /**
     * 許容されたパスのパターン
     */
    private static final List<Predicate<String>> PATH_PATTERN_MATCHERS = List.of(
            createMatcher(
                    "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"),
            createMatcher(
                    "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail_backup\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"));

    /**
     *
     * 与えられた文字列が、SPLで利用できる有効なファイルアップロードパスであるかどうかを判定する.
     *
     * @param path 判定対象の文字列(ファイルパス)
     * @return 判定結果
     */
    public static boolean isValidUploadPath(String path) {
        return PATH_PATTERN_MATCHERS.stream().anyMatch(predicate -> predicate.test(path));
    }

    /**
     *
     * 与えられた正規表現から、{@link Predicate}型のパターンマッチャーオブジェクトを返す.
     *
     * @param pattern 正規表現
     * @return 生成されたパターンマッチャーオブジェクト
     */
    private static Predicate<String> createMatcher(String pattern) {
        return Pattern.compile(pattern).asMatchPredicate();
    }
}
```

以上で、渡されたパスが想定のパターンと一致するものかどうか判定することができるようになりました。簡単ですね。

### 番外：Kotlinで書いてみたら？

今回のポストの内容とはあまり関係のないものですが、ちょっとした興味本位から、Javaで作成したValidatorをKotlinのコードに変えてみました。幸い、intellijには、Javaで書かれたコードをKotlinに変えてくれる便利な機能がありますので簡単にできます。そもそもKotlinを作ったのがJetbrain社なので、Kotlinを普及するための機能であるとは思いますが、おかげでJavaプログラマがKotlinに入門するのも簡単になりますね。

`static final`なフィールドをKotlinでは`companion object`として扱うことになるらしく、コード自体はそう変わらない感覚です。ただ、Kotlinでは`stream()`を呼び出さなくてもCollectionから直接呼び出すことのできるメソッド(`any`)があったり、`List.of()`も`listOf()`で代替できるのですが、自動変換ではそこまでしてくれなかったので、そういうところ自分で変えるしかないです。それで完成したコードは、以下の通りです。

```kotlin
class StorageValidator {

    companion object {
        private val PATH_PATTERN_UPLOAD = listOf( // 画像の保存先パスのパターン(正規)
            Pattern.compile(
                "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"
            )
                .asMatchPredicate(),
            Pattern.compile(
                "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail_backup\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"
            )
                .asMatchPredicate()
        )
    }

    fun isValidUploadPath(path: String): Boolean {
        return PATH_PATTERN_UPLOAD.any { predicate -> predicate.test(path) }
    }
}
```

## 最後に

本当は、正規表現を使って文字列がパターンと一致するかどうかを判定する、という機能を作るのはそう難しいことではないですね。どちらかというと、正規表現そのものの書き方が難しいのでは、と思います。ただ、最近は[RegExr](https://regexr.com)、[RegEx Testing](https://www.regextester.com)、[regular expressions 101](https://regex101.com)など、その場で入力した正規表現をテストしながら作成できるオンラインのツールなども多いので、じっくり時間をかけるといくらでも必要なパターンに合わせたものを作ることができると思います。

個人的な感想としては、書いたコードは短く簡単でしたが、久々に色々と考えられる(効率の面で)チャンスとなったのでなかなか面白い作業になったと思っています。これからもこのような要件があるとしたら、また違う方法で挑戦してみたくなりますね。では、また！
