---
title: "関数型インタフェースを使う"
date: 2019-08-06
categories: 
  - java
photos:
  - /assets/images/sideimage/java_logo.jpg
tags:
  - functional interface
  - lambda
  - java
---

今回も、いつもと同じく仕事で得られた知識です。とあるIterableなクラスを作り、Forループをさせる必要がありました。これはそんなに難しいことではなかったです。まずフィールドにリストを持たせる。またIterableをインプリメントし、戻り値がIteratorであるメソッドを作るだけでしたね。しかし問題は、そのクラスのループの途中で、「使用者が指定したルールによりループを終了させる機能のあるクラスを作ること」でした。つまり、ループを終了させるには引数でループ中の要素を受け入れる引数を持つメソッドが内臓されていて、そのメソッドはインスタンスごとの基準となるデータを持っているということですね。

## メソッドをフィールドとして使う？

メソッドとして機能しながらフィールドみたいにデータを持つ？それをさらに使用者が指定できるようにする？難しい注文だったので一瞬迷いましたが、「functionを使うといい」というアドバイスを受け、調べてみました。なるほど、これを使ったらフィールドとして宣言しながらもメソッドの機能を期待でき流みたいです。早速適用してみて、それがどう機能するかをまず紹介したいと思います。

まずはIterableなクラスを用意します。ループの対象となるのはこちらです。

```java
// Lineというクラスをリストとして持ち、Iterate可能なクラス
public class Factory implements Iterable<Line> {

    private List<Line> lines = new ArrayList<>();

    public Iterator<Line> iterator() {
        return this.lines.iterator();
    }
}
```

次はこのFactoryクラスを持ってループをさせる例です。ループの途中で`Rule`クラスの`isEnd(Line)`での判定が入ります。Lineインスタンスの中で何か条件に当たるものがあったら、戻り値がTrueとなってループから抜ける構造です。

```java
// Iterableクラス
Factory factory = new Factory();

// forループでFactoryの中のLineオブジェクトを処理
for (Line line : factory) {
    // Ruleクラスのループ終了判定メソッドを使う
    if (Rule.isEnd(line)) {
        break;
    }
    // ... 何らかの処理
}
```

ここで判定を行うRuleクラスの場合は、以下のようになっています。

```java
public class Rule {

    // 判定のルールをフィールドとして持っている
    private Predicate<Line> endRule = line -> line.isBroken();

    // 引数のLineがPredicateの条件に当たるかを判定するメソッド
    public boolean isEnd(Line line) {
        return endRule.test(line);
    }

    public class RuleBuilder {

        // 中身は普通のBuilder

        private Predicate<Line> endRule;

        public RuleBuilder endRule(Predicate<Line> endRule) {
            this.endRule = endRule;
            return this;
        }
    }
}
```

まずPredicateが何であり、Lambdaだけであれができる？と思われそうなコードです。しかしちゃんと動いてくれています。それはなぜか？今までよくしらなかったですが、Java8から追加された`java.util.function`によるマジックでした。フィールドはデータを持つためのものである、とは思っていましたが、そのデータというのがメソッドそのものにも慣れるとは知らなかったですね。

それでは、その`java.util.function`が一体どんなものかを、それに含まれている様々なインタフェースたちを持って紹介したいと思います。

## Functional Interface

`java.util.function`に含まれている様々なインタフェースたちを、関数型インタフェースと呼ぶらしいです。Java8で追加されたLambdaを「実装すべき抽象メソッドが一つしかないインタフェースを具現化したもの」と言いますが、ここでいう「実装すべき抽象メソッドが一つしかないインタフェース」のことが関数型インタフェースです。

言葉として表現すると難しいですが、要は一つです。中身をLambdaで満たせて完成するインタフェース。様々なタイプのものがあって、ぞれぞれの特徴は少しづつ違いますが、どんなことがして欲しいかによって選択するものが違うだけで、実際はそんな難しくもないです。むしろ難しいと言えばLambdaの方かな…

とにかくこれらの関数型インタフェースを、一つづつ紹介しましょう。

### Function

Functionは、そのなの通り典型的な関数です。引数と戻り値を指定して宣言します。実行は`apply(適用)`となります。コードで見ると以下のようになります。

```java
// Integerが引数で、Stringが戻り値となる例
Function<Integer, String> function = number -> String.valueof(number);

// Functionの実行
String result = function.apply(12);
```

#### BiFunction

Function以外にも「Bi」が付くいくつかの関数型インタフェースがあります。何が違うかというと、そのなのとおり引数が二つ。他は元のものとほぼ一緒です。

```java
// 二つのStringが引数で、Integerが戻り値となる例
BiFunction<String, String, Integer> biFunction = (string1, string2) -> Integer.parseInt(string1) + Integer.parseInt(string2);

// BiFunctionの実行
int result = biFunction.apply("1", "2");
```

### Predicate

先に紹介したものですね。`Predicate`は「述語」の意味を持っています。その名の通り、「引数がTrueかFalseかを述べる」ようなものです。引数は一つです、戻り値がBooleanです。実行は`test`です。

```java
// 引数がStringの例
Predicate<String> predicate = string -> string.isEmpty();

// Predicateの実行
boolean result = predicate.test("空じゃない！");
```

#### BiPredicate

引数が二つのPredicateです。

```java
// 引数がStringの例
BiPredicate<String, Integer> biPredicate = (string, number) -> string.equals(Integer.toString(number));

// BiPredicateの実行
boolean result = biPredicate.test("1", 1);
```

### Consumer

`Consume`は消費するという意味がありますね。引数を受けて戻り値はない(`void`となる)ものです。実行するときは`accept(受納)`となります。

```java
// 引数がStringの例
Consumer<String> consumer = string -> System.out.println(string);

// Consumerの実行
consumer.accept("吸収！");
```

#### BiConsumer

引数が二つのConsumerです。

```java
// 引数がStringとIntegerの例
BiConsumer<String, Integer> biConsumer = (string, number) -> System.out.println(string + "：" + number);

// BiConsumerの実行
biConsumer.accept("今年儲かる確率は", 0);
```

### UnaryOperator

`Unary`は「単項」の意味。`Operate`は作用するという意味を持っていますね。引数と戻り値が同じもので、引数に何かの操作をしてから返すという印象です。

```java
UnaryOperator<String> uOperator = string -> string + "完成されます";

// UnaryOperatorの実行
String result = uOperator.apply("この文字を入れると");
```

#### BinaryOperator

引数が二つのOperatorです。

```java
BinaryOperator<String> biOperator = (string1, string2) -> string1 + string2 + "ではないです";

// BinaryOperatorの実行
String result = biOperator.apply("私は", "大丈夫");
```

### Supplier

`Supply`は「補給」の意味。Consumerとは真逆のもので、引数がなく戻り値だけがあるものです。実行は`get`となります。引数がないためこちらはBiSupplierのようなインタフェースがないです。

```java
Supplier<String> supplier = () -> "例えば引数なしで文字列が帰ってくる！";

// Supplierの実行
String result = supplier.get();
```

## 最後に

Java8が出てから数年、もうJavaも12までバージョンアップしています。でもまだJava8が使われている場面は多く、なるべくJava8の機能を最大限に活かしたコードを書きたいものですね。LambdaもStreamも難しいですが、Functionみたいにどこかで使うことになってまた今まではできなかったことをできるようになりたいです。

今回も色々と勉強になりました。Javaの世界はまだまだ広くて奥深いものですね！