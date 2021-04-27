---
title: "年月を扱ってみる"
date: 2021-04-27
categories: 
  - kotlin
photos:
  - /assets/images/sideimage/kotlin_logo.jpg
tags:
  - kotlin
  - java
  - spring
---

Kotlin(Java)では、`java.time`パッケージのクラスで日付や時間を処理することができます。例えば[LocalDateTime](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/LocalDateTime.html)や[LocalDate](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/LocalDate.html)などがありますね。サーバサイドではこれらのクラスを使ってDBに日付や時間を入力したり、認証用のトークンの有効期間を設定したりの処理ができるようになります。他にも[Period](https://docs.oracle.com/javase/8/docs/api/java/time/Period.html)や[Duration](https://docs.oracle.com/javase/8/docs/api/java/time/Duration.html)があって、「期間」を扱うこともできますね。

ただ、「年月」という単位を扱いたい場合はどうしたらいいでしょうか。例えば、口座の入出金明細などを照会する時に、「2月から4月まで」という風に期間を設定するケースなどがあるとしたら、いらない「日」や「時間」まで含めるのはあまり効率的でなく、場合によってはバグの原因になるかも知れません。こういった場合は確かな「年月」としてデータを扱うか、数字として表現するかなどどちらかの方法を考える必要があるでしょう。

ということで、今回はこの年月を扱う方法について少し述べたいと思います。

## 年月を年と月に

年月を扱うということは、つまり、いつでも「年」と「月」という二つのデータとして分離できるようにしたいということにもなりますね。ここでは二つの方法で、「年月」を「年」と「月」の二つに分けて扱う方法について説明します。

### YearMonthとして

`LocalDate`や`LocalDateTime`では、基本的に[ISO-8601](https://www.iso.org/iso-8601-date-and-time-format.html)形式で日付を扱うことができます。もちろん、[DateTimeFormatter](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/format/DateTimeFormatter.html)を使って他の形式を指定することもできますが、扱うデータの形が違うだけで、本質的には「年月日」が基本となりますね。

`ISO-8601`の「年月日」形式で日付を扱っているということは、つまり、SpringでREST APIを作っている場合、リクエストの値が`ISO-8601`の形式を守っていれば`LocalDateTime`や`LocalDate`形式に自動変換されるということでもあります。例えば以下のようなリクエストのJSONがあるとしましょう。

```json
{
    "id": "1",
    "date": "2021-04-01"
}
```

Spring側では以下のようなコードで、リクエストのdateを`LocalDate`に変換することができます。

```kotlin
// リクエストボディ
data class DateRequest(val id: Int, val date: LocalDate)

// コントローラ
@PostMapping("/date")
fun date(@RequestBody request: DateRequest) {
    // ...
}
```

そして全く同じやり方で、`LocalDate`を[YearMonth](https://docs.oracle.com/javase/jp/8/docs/api/java/time/YearMonth.html)に変えることで年月に対応することができます。例えば以下のようなリクエストがあるとします。

```json
{
    "id": "1",
    "yearMonth": "2021-04"
}
```

ここで`yearMonth`を`YearMonth`に変えるだけです。以下のようになります。

```kotlin
// リクエストボディ
data class YearMonthRequest(val id: Int, val yearMonth: YearMonth)

// コントローラ
@PostMapping("/year-month")
fun yearMonth(@RequestBody request: YearMonthRequest) {
    // ...
}
```

`YearMonth`を使うことのメリットは、`LocalDateTime`や`LocalDate`と同じく`java.time`パッケージに属するオブジェクトなので、それらと互換性があり、相互変換が自由ということでもあります。例えば以下のように使えます。

```kotlin
>>> val yearMonth = YearMonth.now() // 現在の年月を取得
>>> println(yearMonth)
2021-04
>>> val localDate = yearMonth.atDay(1) // 年月に日を指定してLocalDateにする
>>> println(localDate)
2021-04-01
```

また、`YearMonth`は時間に関する便利なメソッドを多く提供しているので、単純に数値としての年月を扱うだけでなく、色々な要件に合わせて日付関連の処理が必要な場合に便利かも知れません。例えば以下のような機能が提供されます。

```kotlin
>>> val yearMonth = YearMonth.of(2021, 5)
>>> println(yearMonth)
2021-05
>>> println(yearMonth.getYear()) // 年を取得
2021
>>> println(yearMonth.getMonth()) // 月(Enum)を取得
MAY
>>> println(yearMonth.getMonthValue()) // 月(数字)を取得
5
>>> println(yearMonth.isLeapYear()) // うるう年であるかどうか
false
>>> println(yearMonth.atEndOfMonth()) // 月の最後の日(LocalDate)
2021-05-31
```

### 数字として

`YearMonth`で受け取って処理した方がもっとも綺麗な方法に見えますが、状況によっては素直に`Int`型で受け取った方が良い(もしくはそうするしかない)ケースもあるはずです。例えば以下のようなリクエストが送らられて来るようなケースですね。

```json
{
    "id": "1",
    "yearMonth": 202104
}
```

そもそも`year`と`month`のように別の項目になっていたとしたらもっとやりやすいのですが、このように年月が一つの`Int`型のデータとして送られてくる場合は自分で年と月を抽出する処理を作るしかないですね。例えば以下のようなextension functionを書くことができるでしょう。

```kotlin
// 年を抽出する
fun Int.extractYear(): Int = this / 100
// 月を抽出する
fun Int.extractMonth(): Int = this % 100
```

実際のコードを動かしてみると、ちゃんと意図通り動くのを確認できます。

```kotlin
>>> fun Int.extractYear(): Int = this / 100
>>> 202104.extractYear()
res4: kotlin.Int = 2021
>>> fun Int.extractMonth(): Int = this % 100
>>> 202104.extractMonth()
res6: kotlin.Int = 4
```

しかし、パラメータとして渡されたものはただの`Int`型なので、期待した通りの値ではない可能性もあるという問題があります。常に`YYYYMM`という形でデータが送られてくるかどうかをチェックする必要がありますね。

そういう場合に、上記のコードだとリクエストの`yearMonth`が正しい年月の形式になっているかどうかがわかりません。なので、正規式を用いたバリデーションチェックを挟むことにしたらより安全になるでしょう。例えば、以下のようなコードを使えます。

```kotlin
fun Int.toYearMonth(): Pair<Int, Int> =
    if (Regex("^(19|20)\\d{2}(0[1-9]|1[012])").matches(this.toString()))
        this / 100 to this % 100
    else
        throw IllegalArgumentException("cannot convert")
```

上記の関数は、以下のような使い方ができます。簡単に使えるのでいい感じですね。

```kotlin
>>> val (year, month) = 202104.toYearMonth()
>>> println(year)
2021
>>> println(month)
4
```

元の値を二つの`Int`に分けるために戻り値として`Pair`を使いましたが、場合によっては`YearMonth`の方が良いかも知れません。そういう場合は、以下のようなコードが使えます。

```kotlin
fun Int.toYearMonth(): YearMonth =
    if (Regex("^(19|20)\\d{2}(0[1-9]|1[012])").matches(this.toString()))
        YearMonth(this / 100, this % 100)
    else
        throw IllegalArgumentException("cannot convert")
```

## 年と月を年月に

さて、今回は逆に「年」と「月」を繋げて「年月」にする場合の処理を考えてみましょう。二つの`Int`を合わせて、一つの`Int`(YYYYMM)にする形です。ここでまず考えられる方法は二つです。`YearMonth`を使った方法と、文字列に変換してから処理するという方法です。

### YearMonthで

まず`YearMonth`を利用する場合は、年と月をそのまま引数として渡した後、`Int`に変換すれば良いですね。ただ、`YearMonth`は基本的に`ISO-8601`形式なので、2021年4月だと`2021-04`となるので`Int`へ変換ができません。なので、まず`String`に変えてから、`-`を消して`Int`に変換することにします。以上の処理は、以下のようなコードになります。

```kotlin
fun toYearMonth(year: Int, month: Int): Int = 
    YearMonth.of(year, month).toString().replace("-", "").toInt()
```

### 文字列で

文字列で処理する場合は、単純に[String templates](https://kotlinlang.org/docs/basic-types.html#string-templates)を使うことでも可能ですが、注意したいのは、月は1~12という範囲を持つので、単純にtemplateで年と月を繋げると`20214`のような形になり得る可能性もあるということですね。なので、`padStart()`を利用して、月が1~9の場合は先頭に`0`をつけるようにします。そのあとは`Int`に変換するだけですね。これは以下のようなコードになリます。

```kotlin
fun toYearMonth(year: Int, month: Int): Int = "${year}${month.toString().padStart(2, '0')}".toInt()
```

これらの方法は、引数が二つなので、`infix`として定義することもできます(好みの問題かと思いますが)。

```kotlin
>>> infix fun Int.toYearMonthWith(month: Int): Int = "${this}${month.toString().padStart(2, '0')}".toInt()
>>> 2021 toYearMonthWith 5
res10: kotlin.Int = 202105
```

## 最後に

いかがだったでしょうか。あまり難しいコードではなかったので、あえて記事にまでする必要があったのか、という気もしましたが、個人的には`YearMonth`というクラスの存在を初めて知ったのもあり、Kotlinならではのコード(extension function)を書いてみたく試したことを共有したいと思った次第です。もしKotlinやJavaで年月を扱う必要がある方には、少しでも役に立てるといいですね。

では、また！
