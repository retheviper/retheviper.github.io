---
title: "Kotlinで和暦を使う"
date: 2021-12-05
categories: 
  - kotlin
image: "../../images/kotlin.jpg"
tags:
  - kotlin
  - java
---

帳票などで、たまに和暦を処理する必要な時がありますね。例えば元号を表記するとか、和暦の年度を表記するなどの場合があるかと思います。Kotlin(JVM)の場合、西暦だとJavaのAPIの`Date`や`LocalDate`などのAPIを使うと簡単ですが、和暦が必要となるのはごく一部のケースなので方法がなかなか分かりづらいかと思います。なので、今回はKotlinで和暦を扱う方法について少しまとめてみました。

## JapanseEra / JapaneseDate

Javaでは、1.8から和暦で日付を扱える[JapaneseDate](https://docs.oracle.com/javase/jp/11/docs/api/java.base/java/time/chrono/JapaneseDate.html)及び元号を扱える[JapaneseEra](https://docs.oracle.com/javase/jp/11/docs/api/java.base/java/time/chrono/JapaneseEra.html)というAPIを提供しています。なので`JapaneseDate`のインスタンスを作り、そこから`JapaneseEra`を取得することで簡単に元号の情報を取得できるようになります。実際の使い方は以下の通りです。

```kotlin
// 現在日付のJapaneseDateを取得
val japaneseDate = JapaneseDate.now()

// JapaneseEraの取得
val japaneseEra = japaneseDate.era
```

`JapaneseDate`の場合、`LocalDate`と同じく[ChronoLocalDate](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/chrono/ChronoLocalDate.html)を継承しているのでインスタンスを作成する方法はそう変わりません。なので、以下のようなこともできます。

```kotlin
// LocalDateをJapaneseDateに変換
val japaneseDateFromLocalDate = JapaneseDate.from(LocalDate.now())

// 特定の日付を指定してJapaneseDate
val japaneseDateFromSpecificDate = JapaneseDate.of(2000, 12, 31)
```

## 元号を日本語で表記する

和暦を扱う場合にやりたいことは大きく二つかと思います。一つは、元号を文字列として扱うこと、そしてもう一つは、和暦での年度を数字として扱うことです。まずは、元号を文字列として取得できる方法について説明します。

まず上記で紹介した通り、`JapaneseDate`のインスタンスを取得した上で、さらにそのオブジェクトが保持している`JapaneseEra`を取得する必要があります。その後、`JapaneseEra.getDisplayName()`という関数に[TextStyle](https://docs.oracle.com/javase/jp/8/docs/api/java/time/format/TextStyle.html)と[Locale](https://docs.oracle.com/javase/jp/11/docs/api/java.base/java/util/Locale.html)を指定して文字列を取得することができます。前者は文字の出力型を指定する列挙型定数で、後者は言語の指定と思ってください。

`TextStyle`の場合、以下のような値があります。他の言語だと指定したものによって出力がかなり変わってくるかもしれませんが、日本語の場合は`FULL`と`NARROW`だけで十分ではないかと思います。

| 定数 | 出力例 |
|---|---|
| `FULL` | 昭和 |
| `FULL_STANDALONE` | 昭和 |
| `NARROW` | S |
| `NARROW_STANDALONE` | S |
| `SHORT` | 昭和 |
| `SHORT_STANDALONE` | 昭和 |

`Locale`の場合、`Locale.JAPAN`や`Locale.JAPANESE`のどちらを指定しても結果は同じです。ただ、実装としては以下のようになるのでなるべく`Locale.JAPAN`を使った方が良さそうです。

| Locale | 作られるBaseLocaleの設定 |
|---|---|
| `JAPAN` | `language = ja, region = JP` |
| `JAPANESE` | `language = ja` |

以下はこれらの定数を渡して元号を文字列として取得する例です。

```kotlin
val today = JapaneseDate.now()
val era = today.era

// 元号を漢字で取得
val eraName = era.getDisplayName(TextStyle.FULL, Locale.JAPAN) // 令和
```

元号だけでなく、年度までも合わせて表記したい場合もあるかと思います。その場合に使えるものは[DateTimeFormatter](https://docs.oracle.com/javase/jp/11/docs/api/java.base/java/time/format/DateTimeFormatter.html)です。これも`JapaneseDate`が実質`LocalDate`と同じく`ChronoLocalDate`を継承しているから可能なことですね。

```kotlin
// 日付を日本語で表記する
val formatter = DateTimeFormatter.ofPattern("Gy年", Locale.JAPAN)
val todayString = formatter.format(JapaneseDate.now()) // 令和3年
```

もしJava 1.8以前のバージョンを使うなどで`LocalDate`や`JapaneseDate`が使えなく、`java.util.Date`の方を使うしかない場合は、以下のような方法で年号と年度の取得が可能です。

```kotlin
val format = SimpleDateFormat("Gy年", Locale("Ja", "JP", "JP"))
val year = format.format(Date()) // 令和3年
```

`java.util.Date`を使う場合は、`Locale`に第3引数の`variant`まで指定する必要があるので、既存の列挙型として定義されたものは使えません。

また、`Locale.ENGLISH`などに設定すると、`JapaenseDate`を使っている場合でも取得した結果は`AD2021年12月5日`になります。

### 合字で表記する

年号については、Unicodeで合字を取得して使いたい場合もあるかと思います。その場合は、以下のようにUnicodeのMapなどを定義しておいて取得するのが良いかと思います。拡張関数などを定義するのも良いでしょう。

```kotlin
val eraUnicodeMap = mapOf(
    JapaneseEra.MEIJI to "\u337e", // ㍾
    JapaneseEra.TAISHO to "\u337d", // ㍽
    JapaneseEra.SHOWA to "\u337c", // ㍼
    JapaneseEra.HEISEI to "\u337b", // ㍻
    JapaneseEra.REIWA to "\u32ff" // ㋿
)

val era = JapaneseDate.now().era
// 元号を合字で取得する
val eraUnicode = eraUnicodeMap[era] // ㋿
```

上記のサンプルでは`JapaneseEra`が列挙型なのでそのままキーとしていますが、`JapaneseEra`は数値としての情報も持っているのでそちらを使う方法もあるでしょう。それぞれの値に対する数値は以下の通りです。

| JapaneseEra | 数値 |
|---|---|
| MEIJI | -1 |
| TAISHO | 0 |
| SHOWA | 1 |
| HEISEI | 2 |
| REIWA | 3 |

2021年から2022年の3月の場合は令和3年なので、`JapaneseEra.REIWA.value`の値が年度だと勘違いされやすいかなと思います。実際の年度の情報は`JapaneseDate`の方にあるので注意しましょう。

## 年度を数字で表示する

`JapaneseEra`は元号を得るために使う列挙型定数のクラスなので、これ自体は`JapaneseDate`の日付情報を持っていません。なので参照できる情報は、あくまでも元となる`JapaneseDate`が属した元号の情報のみです。

なので数値としての年度は、列挙型の[ChronoField](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/temporal/ChronoField.html)を`JapaneseDate.get()`に渡して取得する必要があります。

```kotlin
val today = JapaneseDate.of(2010, 12, 31) // 平成22年

// 年度をIntとして取得する
val year = today.get(ChronoField.YEAR) // 2010
val yearOfHeisei = today.get(ChronoField.YEAR_OF_ERA) // 22
```

これは`JapaneseDate`が`LocalDate`と違って、直接`year`をgetterで取得できないからです。実際オブジェクトの中を覗いてみると、`LocalDate`は年月日をintとshortのフィールドとして保持していることに対して、`JapaneseDate`は`LocalDate`とint型の`yearOfEra`を持っていて、`get(ChronoField.YEAR_OF_ERA)`を通じてはじめて`yearOfEra`を取得できることになります。getterを用意していないのはおそらく`LocalDate`と`yearOfEra`という二つの概念があるからなのではないかと思います。もちろん、Kotlinなのでこれは簡単に拡張関数を書くことでgetterを作ることはできますね。

また、日付のオブジェクトとして`LocalDate`を使っている場合は場合は`ChronoField.YEAR_OF_ERA`を渡しても西暦の年度が返ってくるので、和暦を使うために`JapaneseDate`を使っているかどうかをまず確認しましょう。

### 年度を2桁の文字で表示する

厳密に言って和暦とは関係のないことですが、年度を取得して使う場合、一貫して先端に「0」のついた2桁の文字列として扱いたい場合もあるかと思います。`JapaneseDate`を通じて年度を取得した場合は`Int`型になるので、1〜9の間は1桁の数字となるわけですが、これを01〜09に表示したい場合は以下の方法が使えます。

#### DecimalFormatを利用する

一つは、JavaのAPIである[DecimalFormat](https://docs.oracle.com/javase/jp/11/docs/api/java.base/java/text/DecimalFormat.html)を使うことです。小数点の範囲などをわかりやすく指定できるので個人的には好むやり方です。

```kotlin
val today = JapaneseDate.now() // 令和3年

// 数字を表示するためのフォーマットを指定
val decimalFormat = DecimalFormat("00")
val year = decimalFormat.format(today) // 03
```

#### String.formatを利用する

もう一つの方法は、Kotlinのスタンダードライブラリの機能である[String.format()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/format.html)を使うことです。性能注視なら、こちらの方法が良いかなと思います。

```kotlin
val today = JapaneseDate.now() // 令和3年

// 数字を表示するためのフォーマットを指定
val year = "%02d".format(today) // 03
```

## 番外：kotlinx-datetime

Kotlinには元々日付や時間を扱うAPIがなかったのですが、2020年から[kotlinx-datetime](https://github.com/Kotlin/kotlinx-datetime)を提供しています。なのでKotlin/JSやKotlin/Nativeなど、JVM上で動かない場合でも日付を扱える公式のAPIができたわけですが、いくつかの懸念があるのでこれを導入するには検討が必要かと思います。

### Pre-releaseの段階

`kotlinx-datetime`はまだpre-releaseの段階で、2021年10月に`v0.3.1`がリリースされています。なので色々とバグがあったり、思い通りにならない可能性があります。また、開発途中のものなので仕方ありませんが、現時点で提供している機能も`java.time`のAPIに比べて少なく、簡単に年号の計算などができるわけではありません。今は必要最低限の機能だけを提供していると思って良いでしょう。

### マルチプラットフォーム向け

Kotlinのスタンダードライブラリ、及び`kotlinx`として提供されるライブラリはマルチプラットホームを考慮した実装となっているため、プラットホームが違っても同じ使い方ができるというメリットがありますが、かえってデメリットになる場合もあります。実際、`kotlinx-datetime`のJVMの実装は内部的に`jata.time`のAPIに依存しているため、JVMだけを使う場合はあえて導入する必要がないともいえます。

また、プラットフォームごとに実装が違うということはどこかで予期せぬ例外が発生したり、期待した結果にならないケースも発生しえる、ということにもなるかと思います。

## java.timeの懸念

`JapaneseEra`では明治以前（慶応など）の元号は使えませんが、おそらくその理由は和暦でグレゴリウス暦が使われたのは明治からだったという歴史的な背景があるのではないかと思います。また、`JapaneseDate`でも明治6年(西暦1873年1月1日)以前の日付を指定すると以下のように例外が発生します。

```shell
Exception in thread "main" java.time.DateTimeException: JapaneseDate before Meiji 6 is not supported
	at java.base/java.time.chrono.JapaneseDate.<init>(JapaneseDate.java:333)
	at java.base/java.time.chrono.JapaneseDate.of(JapaneseDate.java:257)
```

なので、単純に帳票を作るなどのケースでなく、歴史的な研究のための日付計算ではここで紹介した方法は使えないケースもあるかと思います。

また、JDKのバージョンなどの問題があるためか、`JapaneseEra.REIWA`の取得ができなく、エラーとなるケースがあるので注意する必要があります。この場合でも`value`の値の取得は問題ないので、少し可読性は低下しながら分岐などの判定に定数をそのまま使うのは避けたほうが良さそうです。（正確な理由はわかりませんが…）

## 最後に

いかがでしたか。少し興味本位で調べ始めたもののまとめではありますが、本業の方で実際に必要な処理でもあり、これをどうやって拡張関数として落とせるかということも考えられる良い機会となったかなと思っています。

また、JavaのAPIに関しては[Javaバージョン別の改元(新元号)対応まとめ](https://qiita.com/yamadamn/items/56e7370bae2ceaec55d5)という良い記事があったので、興味のある方はご一読ください。

では、また！
