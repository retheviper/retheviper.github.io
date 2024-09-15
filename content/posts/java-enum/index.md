---
title: "Enumを使いましょう"
date: 2019-10-27
categories: 
  - java
image: "../../images/java.webp"
tags:
  - enum
  - java
---

JavaのEnumは興味深いものです。読みやすく、同時に複数の値を持たせるということが魅力的です。なので複数のクラスで共通のコード値を扱う必要があったり、DBに連携する場合に使うと便利ですね。私も現在の仕事では積極的に活用しています。また、完全なクラスとして機能しているので単に定数のため使うだけでなく、処理を記述することも可能なので活用できる範囲はかなり広いのでは、と思います。

なので今回はJavaのEnumの活用法やメリットについて、自分の経験と調べたことをまとめ紹介したいと思います。

## 読みやすい

コードが読みやすいということは、つまり維持補修に有利ということですね。個人的にはプログラミングの段階は「とにかく動く物を作る」→「共通(重複)処理をメソッドやクラスで分離する」→「他人がみてもわかりやすいコードにする」順に進むべきだと思います。そして全ての段階が最初の設計からできていればなおさらですね。

テーブルにとあるコード値が項目として存在するとしましょう。DBの種類によってはその項目を`boolean`にすることもできますが、`char(1)`になる場合もあります。そしてこういった場合はそのコード値は二つの状態だけでなく、3つ以上の状態を持つ場合もありますね。サンプルとしてJava内での状態をStringとして持ち、DBには`char(1)`として記録するとしましょう。そういう場合は以下のようなコードが必要となるはずです。

```java
// Stringで記述されている状態をDBのコード値に変換するメソッドの例
public String toCodeValue(String status) {

    if (status.equals("TRUE")) {
        return "0";
    } else {
        return "1";
    }
}
```

このメソッドを使うコードは、以下のようになります。

```java
// itemオブジェクトにステータスを指定する
public void setStatusTrue(Item item) {

    item.setStatus("TRUE");
}
```

ハードコーディングされた`status`を定数かすることは可能です。定数は普通のクラス内でも持たせますね。しかし、そういう場合は定数がどのクラスで定義されているかをまず知る必要があります。とこで定数が定義されているかわからない場合は修正も難しくなるし、重複して同じ定数をそれぞれ違うクラスに作成することになる可能性もありますね。

そして、このコードでは処理の結果を見る前まで状態がどうなっているか確認できなくなる可能性があるという欠点があります。もしどこかで指定された`status`が`TRUE`でも`FALSE`でもない第三の文字列だったら？そういう場合は`if`の分岐を増やすしかないでしょう。

これをEnumを使うコードに変えてみましょう。コードは以下のようになります。

```java
public enum StatusEnum {
    
    TRUE("0"),

    FALSE("1");

    private Integer codeValue;

    StringEnum(Integer codeValue) {
        this.codeValue = codeValue;
    }

    public Integer getCodeValue() {
        return codeValue;
    }
}
```

定数名にそれぞれのコード値を設定し、フィールド・コンストラクタ・Getterを用意するだけです。構造からわかるように、このようなEnumクラスはLombokのアノテーションでも簡単に作ることができます。

```java
@Getter
@AllArgsConstructor
public enum StatusEnum {
    
    TRUE("0"),

    FALSE("1");

    private Integer codeValue;
}
```

作成したEnumを実際活用する場合のコードは、以下のようになります。

```java
// itemオブジェクトにステータスを指定する
Item item = new Item();
item.setStatus(StatusEnum.TRUE.getCodeValue()); // "0"となる
```

Enumを作成することで、入れたい値が明確になります。また、Enumは独立したクラスなのでパッケージを分けて保存することで管理がより簡単になります。あとでコード値が増える場合にもそちらのEnumを修正するだけで住みますね。また、Enum自体をフィールドとして宣言することも可能です。その場合は以下のようになりますね。

```java
// フィールドにEnumがある例
@Data
public Item {

    private StatusEnum status;
}
```

フィールドがEnumの場合は、値の指定がより簡単になります。

```java
Item item = new Item();
item.setStatus(StatusEnum.TRUE); // TRUEとして保存

// コード値を抽出したいときのコード
String itemCodeValue = item.getStatus().getCodeValue();
```

## 複数の値を持つことが可能

そこまで読み易くはならないのでは？と思われる可能性もありますね。確かに、一つの定数につき値が一つだけの場合はそうかもしれません。しかし、Enumの良いところは、定数に複数の値を持たせることも可能ということです。

例えば二つ以上のDBを使っていて、同じ項目を片方のテーブルは`char(1)`、またの方は`boolean`で管理しているとしましょう。Enumでは両方を一つの定数として管理することができます。それがどういうことか、以下のコードで確認してみましょう。

```java
@Getter
@AllArgsConstructor
public enum MultiValueEnum {
    
    Y("0", true),

    N("1", false);

    private Integer charValue;

    private boolean booleanValue;
}
```

コードを見ると簡単に理解できると思いますが、どちらのGetterを使うかによって同じ定数でも違うデータ型のコード値を返却します。これを実際のコードで使うとしたら、以下のようになります。

```java
// DBの項目がchar(1)
Item item = new Item();
item.setStatus(MultiValueEnum.Y.getCharValue()); // "0"

// DBの項目がboolean
item.setStatus(MultiValueEnum.Y.getBooleanValue()); // true
```

複数の値を指定可能ということは、配列やListでも可能ということではないか？と思う方もいるかもしれません。結論からいうとYesです。自分も最初からわかっていたわけではありませんが、調べてみると[Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)を使うと値としてListを持たせ、さらにそのListの中の値との比較もできるようです。この場合のコードは以下のようになります。

```java
@Getter
@AllArgsConstructor
public enum ListValueEnum {
    
    Y(Arrays.asList("Good","Excellent")),

    N(Arrays.asList("Bad", "Unavailable")),

    NULL(null);

    private List<String> codeValueList;

    // Listの中に値があるかチェック
    public boolean hasCodeValue(String codeValue){
        return codeValueList.stream().anyMatch(code -> code.equals(codeValue));
    }

    // Listの中の値と一致する場合、その定数を返す("Good"ならListValueEnum.Yを返却)
    public ListValueEnum findByValue(String codeValue) {
        return Arrays.stream(ListValueEnum.getCodeValueList())
        .filter(listValueEnum -> listValueEnum.hasCodeValue(codeValue))
        .findAny().orElse(NULL); // 当てはまる値がない場合はNullを返す・
    }
}
```

まだStreamは一部でしか使ってみたことがないのでこういう活用方法は考えてみたことがないですが、どこかで使えそうなコードです。

また、フィールドとして使うEnumに特に値がない場合は以下のようなアノテーションを使うことも可能です。

```java
@Data
public class Item {

    // StringEnum.YESのまま使う場合
    @Enumerated(EnumType.STRING)
    private StringEnum codeValue1;
}
```

## Classらしき活用

Enumはクラスなので、もちろん処理を持たせる方法もあります。処理を持たせるというのは難しく感じるかもしれませんが、処理をメソッドと思えば良いだけの話です。メソッドのフィールド化する方法についてはLambdaに関する[以前のポスト](../java-functional-interface)を参照してください。

```java
@AllArgsConstructor
public enum CalculateEnum {

    // Lambdaでフィールドを指定
    TYPE_1(num -> num),

    TYPE_2(num -> num + 10);

    private Function<Integer> calculate;

    // 値を入れると処理結果が帰ってくる
    pulbic Integer calculate(Integer number) {
        return calculate.apply(number);
    }
}
```

このような形のEnumを使う場合のコードは以下のようになります。

```java
// 処理結果を取得
Integer type_1 = CalculateEnum.TYPE_1.calculate(10); // 10

Integer type_2 = CalculateEnum.TYPE_2.calculate(10); // 20
```

## 最後に

共通の部品として使われるものが重複されていると管理もその値の理解も難しくなり、余計なコードが増えるますね。これをEnumで克服できるということを伝えたかったのですが、いかがでしょうか。ただ、定数を必ずEnumにする必要があるかは、よく考えてみる問題だと思います。場合によってはテーブルとして管理した方が良いのかもしれません。そしてある特定のクラス内でしか使われていないなら、あえてEnumを作る必要もないでしょう。

それでもこのようにAPIの活用方法を理解し、覚えておくと、どこかで使っていい場合が現れるのではないかと思います。最初はLambdaをただの「読みづらいコード」としか認識していなかった私も、Functionの存在を知ってからは積極的に使っています。知識のみでなく、その知識を適材適所で活かしていけるようになることが真のプログラマーと思います。まずその判断は難しいかもしれませんが、知識を先に持つことで見えてくるものもあるのではないでしょうか。なので、これからも新しく得られた知識があれば、このブログで紹介していきたいと思います。
