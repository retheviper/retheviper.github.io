---
title: "Oracle JavaSE 8 Silverについて"
date: 2019-10-06
categories: 
  - java
image: "../../images/java.webp"
tags:
  - ocjp
  - certification
  - java
---

今回、Oracle認定JavaSE 8 Silverを受験しました。仕事でしばらくJavaを使うことになって、自分の実力がどのレベルなのかを確かめたかったので受けてみましたが、運よくも合格。Java Silverで検索すると合格した人たちはどう勉強したか、難易度はどうなのかについてのブログが多いのでここでは書きません。

それでもあえてポストとして資格試験のことを書く理由は、どうしても資格とは実際の業務で必要となる知識とは乖離があるものだからです。資格をまず取得して業務に入る人もいれば、私のように業務でのお作法を経験してから受験する人もいるだろうと思います。なので、私と全く同じ状況ではないとしてもテストで困る問題は多分ある程度その領域が重なるのでは、と思います。その結果として他の多くの方が書いているものとあまり変わらないことを述べているのではないか、とも思いますが…

もしこれからJavaSE Silverを受験しようとしている方がこのポストを見つけた場合に、少しでも役に立てるような情報になれるといいなと思い、自分の経験として意外とハマったところについて述べます。

## 全体的な印象

業務では使わない知識が要求される場合が少なくないです。ただそれが全部無駄な知識とは言えない印象でした。一旦覚えておくと業務でも活かせるのでは、と思う問題もありました。大きく分けてAPIに対する単なる知識を聞く問題と、コードを読んで理解する能力を要求するような問題があります。

意外と難しいなと思ったのは、IDEに依存できない問題でした。代表的にimportの書き方、コンパイルエラーの見つけだしなどがあります。実際の業務でIDEを使わず、単なるテキストエディタでJavaを書くことはないと思いますが、試験としてはそのような問題の頻度が決して低くないので、問題で提示されるコードはちゃんと読む必要があります。

それでは、以下ではテストの個別項目について感じたことを述べていきます。

### データ型

データ型で最も多く使われるのは、int/double/Stringの三つだと思います。テストでもbyte型やfloat型の範囲などを聞く問題はなかったですが、一部の問題でこれらが出てくる場合があります。例えばswitch文の条件文で使えないデータ型は何かを選ぶ問題など。Bronzeならもっとデータ型について詳しく聞く問題が出てくるのでBronzeの次にSilverを受けるならともかく、私みたいにSilverから挑戦する人なら少し困るかもしれません。

他にもデータ型と関連する問題の種類は、型変換(暗黙的な型変換とそうではない場合を選ぶ)があります。こちらもbyte, short, long, floatの特徴を確かに覚えないと、意外と正解がわからなくなる問題でした。

### String / Stringbuilder

StringやStringBuilderのメソッドに関する問題が多かったです。ただ普段はあまり意識してなかった部分ですが、StringはImmutableなオブジェクトになるためreplaceやsubstringなどのメソッドを使うと新しいインスタンスが返却されるということをちゃんと覚えておく必要があります。例えば以下のような問題があります。

```java
// 出力される文字列を当てる問題

String s = "this is string";
s.substring(0, 1);
System.out.println(s);
```

元のString自体は変わってないので最初に宣言した通り出力されるのが当たり前ですが、注意深くコードを読まないとsubstringで切り取った文字が出力されるように勘違いしやすいです。(自分だけかもしれませんが…)

### 配列

配列をあえて使う場合があまりなく、ほとんどListを使っていたのでこちらの問題でも苦戦しました。配列の宣言の仕方から、配列の中の要素を処理する方法などの問題がよく出てきましたが、配列の宣言には様々な方法があるので少し選択肢が怪しく思われます。

もちろんListは基本が可変・スレッドセーフではないので場合によっては業務でも配列を導入する必要はあるかもしれません。覚えておいて損ではないと思いますので、配列については確実に覚えておきたいと思いました。

### アクセス修飾子 / 継承

defaultとprotectedの違いを問う問題があります。普段からパッケージ構成や継承を考慮した実装をしていたならともかく、大抵の場合はpublicとprivateのみを使っていて他を忘れやすい場合もあると思うので、ちゃんと覚えておきたいものでした。

また、継承の問題でアクセス修飾子が重要なキーワードとなる問題もありました。例えばスーパークラスからオーバーライドーしているメソッドをより狭い範囲にするようなコードを提示して、そのコードを実行したらどうなるかを聞く問題など。正常実行されると思ってその結果を答えとして選んでも、「コンパイルエラー」が正解だったりします。

継承に関してはかなり問題の量が多いような印象を受けました。Interfaceとabstractの違い、継承の仕方、インスタンスと参照の違い、キャストなどバリエーションも多く、どれも実務では使われているので慣れてはいるものの、問題自体もそのためか知識だけよりは少し複雑な方法でミスを招こうとしている印象でした。コードを注意深く読む必要があります。

### ラベル

ラベルについてはこの度初めて接したので全然知らなかったですね。しかし二重ループで使うとかなりパフォーマンスの改善を図れると思いますので、これはテストだけではなく覚えておくと良いこつと思います。ただ、ループ以外で使うことはそんなにないかも…とも思います。

問題としてはラベルがついたループ文ないでif文を書き、結果がどうなるかを聞く問題が少しありました。例えば以下のようなものです。

```java
// 出力がどうなるかを問う問題

int num = 0;

x:
for (int i = 0; i < 10; i++) {
    if (i % 5 == 0) {
        continue x;
    }

    System.out.println(num);
    num++;
}
```

ラベル自体よりは条件文とcontinue / breakの違いをちゃんと理解しているかを聞くような問題とも言えると思いますが、とにかくラベルとはなんぞや？となると、一旦答えることができなくなります。もちろん知識としてラベルをつけることができるのはどこか、という問題もありました。

### 例外

Javaの例外の特徴や、try-catch文での挙動に関する問題が出てきます。業務ではカスタム例外を作ることもあるのでExceptionとRuntimeExceptionの違いについて理解しているならそこまで問題となることはないと思いますが、提示されるコードで見逃す可能性があるのは「throws宣言があるかどうか」、「catchでスーパークラスの例外を宣言しておいてその次にサブクラスをまた書いてコンパイルエラーになる場合であるか」のような部分です。上位の例外をcatchすると、下位の例外は書く必要がないので注意しましょう。

IDEを使える環境だったらあまり気にする必要ない部分なので、意外と見逃せやすい部分ではないかと思います。

### Lambda

JavaSE SilverではJava8ならではのAPIについて聞く場合が少なくないです。Lambdaもそうですね。ただ、大抵のLambda関連の問題は正しい書き方やFunctionの種類に関する知識を問う問題です。慣れてなくても、覚えておくと大丈夫でした。

書き方については括弧とreturnの省略が正しく書いてあるのかについての問題で、例外同様IDEだとすぐコンパイルエラーになるため見逃しやすい部分でした。

### LocalDate / LocalTime / LocalDateTime

今まで自分が使っていた日付関連のAPIは`java.util.Date`と`java.sql.Date`しかなかったので、苦戦した問題でした。こちらもStringと同じくImmutableなので、似たような問題が出てきます。例えば以下のような問題です。

```java
// 出力がどうなるかを問う問題

LocalDate date = LocalDate.now();
date.plusDays(5);
System.out.println(date);
```

Stringと同じく、Local〜類のAPIは値を操作するメソッドが新しいインスタンスを返却することを覚えておくと良いだけです。ただ、時間や日付の出力フォーマットに関する問題が出るのでこちらは覚えておく必要がありました。APIを見ると`java.sql.Data`にも変換ができるのですが、変換に関する問題は出てませんでした。

### その他

問題集だけだと複数の選択肢を選ぶ問題なのに一つしか選んでなかったりしていて、本場でこうなると困るな…と思っていましたが、実際のテストでは単一選択肢だとラジオボタン、複数だとチェックボックスで選ぶようになっているので心配ありませんでした。チェックスボックスなのに一つだけ選んでいたりすると次の問題に移る前に警告も出ました。

一部ループでの処理などを直接計算する必要がある場合もありますが、多くは時間がかからなく即答できるものです。時間が足りないということはほぼないと思いますので、問われているAPIに関してその特徴をよく覚えると簡単に合格できる資格なのでは、と思います。

ただ逆に、私みたいに業務に慣れていて基本を忘れていたり知らなかったりするとハマるところも少なくないと思います。先に述べたように時間が足りなくなることはあまりないので、コードが提示される問題は注意深く読みましょう。そしてコードが提示される問題の答えは実質的に「処理の結果が出力される」、「コンパイルエラーになる」、「実行時に例外がスローされる」の三つだけなので、まずどれに当て嵌まるかを考えてみると良いと思います。

それでは、これから受験する皆さんも合格できますように！
