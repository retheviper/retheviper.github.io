---
title: "デザインパターン、Builder"
date: 2019-07-14
categories: 
  - java
photos:
  - /assets/images/sideimage/java_logo.jpg
tags:
  - design pattern
  - java
---

以前、自分より開発者として日本就職が早かった大学の後輩がいて、どんな言語やフレームワークを勉強した方がいいかを聞いたことがあります。周りではC#をやるといい、ReactやNode.jsのような流行りのライブラリーを学んだ方がいいという人もいましたが、現場で使われている人の意見が聞きたかったからです。そして彼は、言語やフレームワークはメインとしている言語をマスターしたらいつでも変えられるもので、実質的に必要となるスキルはデザインパターンと言っていました。

それから本を買い、いくつかのデザインパターンを見たことはありますが、そのパターンたちをどう使ったらいいか一人で考えるのは難しいことでした。一人でコードを書く時は、自分が理解できるコードを書けばいいだけなのでそこまで考える余裕が無くなりますね。また、エンドユーザーだけを意識したコードになりがちだったのであまりパターンを含むコードの書き方をする必要もなかったです。

それが今はJavaでのフレームワーク開発に関わることとなり、自分の買いたコードを違う人が使えるように書けとの指示を受けましたが、いつも通りDTOを元とするWrapperクラスを生成して行けばいいのかなと思っていたら、実装してみてからはこれはダメだなと感じました。なぜなら数十から数百に至る変数の設定値があって、引数として渡すのも単純な数字や文字などではなかったからです。自分の作ったコードが使う側として接した場合は、これはダメと思うはず。

どう改善したらいいかと悩んでいたら、指示を出した方からBuilderパターンを使うといいだろうというアドバイうをもらいました。これなら引数は最低限にして、直観的に使えるらしいです。なので実際使ってみました。そして伝統的なDTOと比べてみると、確かになと思いましたね。何が違ってなぜ違うのかを、DTOとBuilderパターンの比べでこれから述べたいと思います。

## Telescoping Constructor Pattern

テレスコープとは望遠鏡のこと。どこが望遠鏡的かというと、だんだん伸びていくコンストラクターの形が望遠鏡の伸縮みたいでこんな名前になったようです。最近はJava Beanの中でもよく使われるパターンなのですね。オブジェクトを生成する時に引数の数によって値を入れる変数の数を調節できます。Method Overloading[^1]で複数のコンストラクターを用意するだけです。

例えばカフェでコーヒーを注文する過程を、Javaのクラスで具現化するとしましょう。カップのサイズ、ホットかアイスか、シロップは入れるか、クリームは入れるか…様々なオプションがありますね。これをTelescoping Constructor Patternで書くと、以下のようになります。

```java
public class CoffeeOrder{

    private final String size; 
    private final boolean hot;
    private final boolean addCream; 
    private final boolean addSugar;
    private final boolean takeout;
	
    public CoffeeOrder(Stirng size, boolean hot){
        this(size, hot, false, false, false);
    }

    public CoffeeOrder(Stirng size, boolean hot, boolean addCream){
        this(size, hot, addCream, false, false);
    }

    public CoffeeOrder(Stirng size, boolean hot, boolean addCream, boolean addSugar){
        this(size, hot, addCream, addSugar, false);
    }

    public CoffeeOrder(Stirng size, boolean hot, boolean addCream, boolean addSugar, boolean takeout){
        this.size = size;
        this.hot = hot;
        this.addCream = addCream;
        this.addSugar = addSugar;
        this.takeout = takeout;
    }
}
```
これでオーダを定義するクラスが一つ、できました。実際使ってみましょう。

```java
public class Cafe{

    public static void main(String[] args){
        // オーダーごとのオブジェクトを生成する
        CoffeeOrder order_1 = new CoffeeOrder("tall", false);
        CoffeeOrder order_2 = new CoffeeOrder("grande", false, true);
        CoffeeOrder order_3 = new CoffeeOrder("venti", true, true, false);
        CoffeeOrder order_4 = new CoffeeOrder("tall", false, false, true, false);
    }
}
```
このパターンのよくない点は、オブジェクトを生成する時、引数の意味を分かりにくいという点です。実際クラスの中身をみないと、連続している`false`や`true`の意味が分かりませんね。そして例えば、サイズとシロップだけを引数として入れたい場合は、それに合わせてまたコンストラクターを作成しなければならないです。変数が増えれば増えるほど、それに合わせコンストラクターを用意する必要があるという問題もあります。あとでオーダーのオプションが増えたり減ったりするとそれに対応するのが難しいですね。

## Java Bean, DTO(VO)

JavaでOOP[^2]の概念を学ぶ時、初めて接したのがこの`Java Bean`です。これも一つのパターンと言えますね。みなさんがよく知っているよう、`Getter`と`Setter`で値を渡すパターンです。同じくオーダーのクラスを作ってみましょう。

```java
public class CoffeeOrder{

    private final String size; 
    private final boolean hot;
    private final boolean addCream; 
    private final boolean addSugar;
    private final boolean takeout;

    public CoffeeOrder(){}

    public setSize(String size){
        this.size = size;
    }

    public getSize(){
        return this.size;
    }

    public setHot(boolean hot){
        this.hot = hot;
    }

    public getHot(){
        return this.hot;
    }

    public setAddCream(boolean addCream){
        this.addCream = addCream;
    }

    public getAddCream(){
        return this.addCream;
    }

    public setAddSugar(boolean addSugar){
        this.addSugar = addSugar;
    }

    public getAddSugar(){
        return this.addSugar;
    }

    public setTakeout(boolean takeout){
        this.takeout = takeout;
    }

    public getTakeout(){
        return this.takeout;
    }
}
```
コンストラクターとして引数を受け取るパターンも含める場合はありますが、Java Beanとしての特徴はこのGetterとSetterにあるので、ここでは省略。では同じく、これでオーダーを生成してみましょう。

```java
public class Cafe{

    public static void main(String[] args){
        // オーダーの内容はSetterで設定する
        CoffeeOrder order_1 = new CoffeeOrder();
        order_1.setSize("tall");
        order_1.setHot(false);

        CoffeeOrder order_4 = new CoffeeOrder();
        order_2.setSize("grande");
        order_2.setHot(false);
        order_2.setAddCream(false);
    }
}
```
さっきよりは個別項目ごとに値を設定することができ、それぞれのSetterをみてどんなオーダーを出しているのかがより明確になりますね。また変数が増えてもそれに合わせてGetterとSetterを用意するだけで良いです。

ただ、一つの注文を完成する時、オプションの数が増えると無題に長いコードになってしまうという問題がありますね。今は5つのフィールドを使っているだけですが、もし20、30のオプションがあったら？それをいちいち書くのはかなり時間もかかることですね。私が失敗したのはこの部分でした。なのでBuilderを使い、この問題を解決してみます。

## Builder Pattern

```java
public class CoffeeOrder{

    private final String size; 
    private final boolean hot;
    private final boolean addCream; 
    private final boolean addSugar;
    private final boolean takeout;
	
    public CoffeeOrder(Stirng size, boolean hot, boolean addCream, boolean addSugar, boolean takeout){
        this.size = size;
        this.hot = hot;
        this.addCream = addCream;
        this.addSugar = addSugar;
        this.takeout = takeout;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder{

        private final String size; 
        private final boolean hot;
        private final boolean addCream; 
        private final boolean addSugar;
        private final boolean takeout;
	
        public Builder(){}

        public CoffeeOrder size(String size){
            this.size = size;
            return this;
        }

        public CoffeeOrder hot(boolean hot){
            this.hot = hot;
            return this;
        }

        public CoffeeOrder addCream(boolean addCream){
            this.addCream = addCream;
            return this;
        }

        public CoffeeOrder addSugar(boolean addSugar){
            this.addSugar = addSugar;
            return this;
        }

        public CoffeeOrder takeout(boolean takeout){
            this.takeout = takeout;
            return this;
        }

        public CoffeeOrder build() {
            return new CoffeeOrder(size, hot, addCream, addSugar, takeOut);
        }
    }
}
```
Inner Classも入り、何か複雑になったように見えますが、実際使ってみるとそうでもないです。このようなBuilderクラスを使うとどうなるのか、また確認してみましょう。

```java
public class Cafe{

    public static void main(String[] args){
        // オーダーを生成してBuilderを使う
        CoffeeOrder order_1 = new CoffeeOrder();
        order_1.Builder().size("tall").hot(false).build();

        CoffeeOrder order_2 = new CoffeeOrder().Builder()
                            .size("grande")
                            .hot(false)
                            .addCream(true)
                            .takeout(true)
                            .build();
    }
}
```
Setterと似たような使い方で、一回だけで複雑なオプションを全部処理できます。また、生成と同時にもオーダーを完成できます。Builderが戻り値として自分自身を使っていて、連続してメソッドを呼び出すことができるからです。これならいくら変数が増えても対応できますね！

## Lombokを使う

以上のパターンは[Lombok](https://projectlombok.org)を使うとアノテーションだけで設定できるらしいです。例えばコンストラクターは`@NoArgsConstructor`や`@RequiredArgsConstructor`、`@AllArgsConstructor`でできます。Java Beanなら`@Data`をつけることでGetterとSetterができるらしいですね。また、Builderの場合は`@Builder`でできると言います。以下はLombokを使った場合の例です。

```java
import lombok.Builder;

@Builder
public class CoffeeOrder{

    private final String size; 
    private final boolean hot;
    private final boolean addCream; 
    private final boolean addSugar;
    private final boolean takeout;
}
```

ちなみに`@Builder(toBuilder = true)`にすると、インスタンスの新規生成では`CoffeOrder.builder()`から直接Builderにアクセスできるようになります。また、既存のインスタンスの値を受け継ぐ場合は`order_1.toBuilder()`を使えるようになります。実際は以下のコードになるようなものとなります。

```java
import lombok.Builder;

@Builder(toBuilder = true)
public class CoffeeOrder{

    private final String size; 
    private final boolean hot;
    private final boolean addCream; 
    private final boolean addSugar;
    private final boolean takeout;

    // ... 基本的なBuilder

    public CoffeeOrderBuilder toBuilder() {
        return new CoffeeOrderBuilder().size(this.size).hot(this.hot).addCream(this.addCream)
                                       .addSugar(this.addSugar).takeout(this.takeout);
    }
}
```

そしてBuilderを使うとき、親クラスのフィールドをそのまま継承したい場合はフィールドに`@Builder.Default`をつけることでそのまま受け継がれます。他にもフィールドにつけてアクセスレベルを指定できるなど便利な機能が多いので、ぜひ使いたいものですね。

## 最後に

デザインパターンの種類に何があって、どんな構造をしているかを把握することも大事ですが、何より大事なことは適材適所に使えることではないかと思いました。最初から自分がBuilderパターンを知っていたとしても、それを使ったらいいと言われなかったら果たして使おうとしていただろうかと思うと、そうでもないような気がしますね。なのでこれからはデザインパターン自体の研究とともに、それをどの場合に使えるかという面から考察していきたいと思います。

それでは、また会いましょう！


[^1]: 引数の数や種類を変えることで、同名のメソッドを複数作成する記法。
[^2]: Object Oriented Programing.オブジェクト指向プログラミングとも言いますね。コードをひたすら上から下まで流れる処理として扱うのではなく(手続き型プログラミング)、隔離されたオブジェクト間のデータ交換として成立するプログラミングのパラダイムです。