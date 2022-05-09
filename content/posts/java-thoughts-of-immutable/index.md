---
title: "インスタンスをImmutableにするための工夫"
date: 2019-08-25
categories: 
  - java
image: "../../images/java.jpg"
tags:
  - design pattern
  - java
---

Pythonのような本格的なオブジェクト指向言語ではあまり見かけられないことですが、Javaではいわゆる参照型の以外にもプリミティブ型というものがありますね。どうもJavaが初めて世に出た時代はまだオブジェクト指向という概念が生まれたばかりだったのでそうなったのではないかと思います。このプリミティブ型が存在するという点から、Javaは完全なオブジェクト指向言語ではないという話もあるようです。

プリミティブ型に対する定義は言語ごとに少し違うようですが、私が知っているのはJavaだけなのでJavaの基準からいうと、プリミティブ型はオブジェクトではないデータ型を指す言葉です。そしてそれは、メモリー上に載せたデータをどう持つかの観点がオブジェクトとは違うということです。オブジェクトはメモリー上のデータが位置する「アドレス」を指すことに対して、プリミティブ型はそれぞれ独立したメモリー領域にデータを載せます。

これを証明するのが、条件文での演算子の違いです。プリミティブ型で、二つの変数が同一なデータを持っているかを比較する演算子は`==`ですね。しかし、同じ方法でオブジェクト、よく挙げられている例としてStringだと、同じ方法を使えません。`equals()`を使わないとStringでの正確な値の比較はできなくなりますね。なぜならオブジェクトが持っている値そのものはメモリーのアドレスなので、「同じ値を入れた」つもりでもそれぞれのオブジェクトが指しているメモリーのアドレスは違う可能性があるからです。

このように、メモリー問題はプログラミングに対してはかなり重要なものです。いくらメモリーの絶対値が増えても、メモリーに載せられたデータをどう参照するか、どう扱うかを間違えたら思い通りにプログラムは動かない可能性があるからです。そしてまた重要なのは、メモリー上に載せたデータを参照する方法だけではなく、どう管理するかということです。正確なデータを入れたつもりが、途中で変わったりすると、参照の仕方が正しくてもプログラムが正常動作しない可能性がありますからね。

そういう意味で、今回はImmutableなクラスについて述べたいと思います。

## Immutable Objectとは

Immutavle Objectとは、簡単にいうと「一度インスタンスが生成されたら、そのインスタンスが持つデータが変わらない」オブジェクトのことです。逆に、インスタンスの生成以後もデータが変わる可能性のあるオブジェクトはMutableと言います。代表的なMutableクラスとしてはBeanを挙げられますね。Setterを通じて自由にデータを変えることができます。そしてImmutableなクラスとして代表的なのは、Stringと言います。Stringは値を代入しても、元のデータはGarbage Collectorの対象になるまでメモリーに残り、そのStringオブジェクトが指すメモリーのアドレスだけ変わるからです。

先に述べたように、プログラムの中でオブジェクトが持っているデータが途中で変わると、安定的な動作を保証できなくなります。そしてマルチスレッド環境では、二つのオブジェクトが同じメモリーアドレスを参照していると、スレッドそのものが止まってしまう可能性もあります。もしそのような場合が生じるとどこで問題が起きたか調べることも難しいですね。このような問題を回避するため、Immutableなクラスを作成するときは「インスタンスの生成以後はデータが変わらない」ことと、「クラスの持つデータが同じメモリーアドレスを参照しないように」します。

それでは、Immutableなクラスを作成する方法にはどんなものがあるか、見ていきましょう。

## setterは使わない

最近はかなりLombokを使う場合が多く、ある程度定型的なコードを生成してくれるので、Lombokのアノテーションを使って生成されるコードを持って説明したいと思います。Lombok自体の紹介は、[以前のポスト](../java-design-pattern-builder)を参照してください。

Lombokでは、アノテーションをつけることで簡単にBeanを生成できます。クラスの上に`@Data`をつけることで、簡単にSetterとGetterができますね。これを使った場合、直接メソッドを手で書くより安定的でコードの量も減るため積極的に使えます。

しかし、Setterが生成されるということはImmutableなオブジェクトにならないことを意味します。次は実際、`@Data`をつけることで生成されるコードの例です。

```java
// @Dataの場合
public class Car {

    // フィールドだけを定義
    private String name;
    private String color;

    // 以下のメソッドたちが自動生成
    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }

    public void setColor(String color) {
        this.color = color;
    }

    public String getColor() {
        return this.color;
    }
}
```

こうなった場合、フィールドが`final`で守られていないといつでもSetterにより値が変わる可能性があります。そして一部Setterメソッドが使われていない場合は全フィールドに値が設定されてないままnullになってしまう可能性もありますね。これはImmutableの定義にふさわしくないコードとなっています。

幸い、Lombokで提供するアノテーションの中にはImmutableなクラスを生成するためのアノテーションもあります。`@Value`というものです。これを使うと、`@Data`と同じ機能をしながら(インスタンスの生成とフィールドの値を指定する方法は変わりますが)もImmutableなクラスを生成することができます。こちらのアノテーションを使ったコードは、以下のようになります。

```java
// @Valueの場合
public class Car {

    // フィールドだけを定義
    private String name;
    private String color;

    // 以下のメソッドたちが自動生成
    public Car(String name, String color) {
        this.name = name;
        this.color = color;
    }

    public String getName() {
        return this.name;
    }

    public String getColor() {
        return this.color;
    }
}
```

最初、インスタンスを生成するときはコンストラクターで全フィールドを引数として指定します。フィールドが多いとどれがどれかわからなくなりますが、フィールドの値指定が漏れる可能性は無くなります。そしてSetterがないため一度生成されたインスタンスに対してはフィールドを変更できなくなります。

そしてこの`@Value`アノテーションの良いところは、Builderパターンと両立できるというところです。`@Builder`をつけることで、インスタンスの生成時にそれぞれのフィールドがどんなものであるかを明確に確認できますね。ただ、Builderパターンでは全フィールドに値を指定する義務はないので注意が必要です。この問題は、手で`build()`メソッドを書くことで回避できます。ある意味、オーバーライドに近いことだと言えますね。コードで表現すると、以下のようになります。

```java
@Value
@Builder
public class Car {

    private String name;
    private String color;

    // 以下のコードだけを作成
    public static CarBuilder {

        // nullのフィールドがあったらNPEを発生させる
        public Car build() {
            if (this.name == null) {
                throw new NullPointException();
            }
            if (this.color == null) {
                throw new NullPointException();
            }
            return new Car(this.name, this.color);
        }
    }
}
```

この場合はやはりコードが複雑になり、フィールドが増えるとまたそれに対応しなければならなくなりますね。フィールドがまた`List`だったりすると、ループでnull検査をする必要もあるはずです。このように`@Value`だけを使う場合に比べ、コードが複雑になっているので、便宜生と安全性でどちらを選ぶかを考える必要がありますね。

## final宣言

Beanを使うしかない場合もありますね。例えばフィールドがnullになっても良い場合もあるはずです。それともそのBeanを持って処理をするメソッドでnullを検査するなど、何かの措置をしといたら良い場合もあるはずでしょう。

そしてBeanを使う場合、フィールドを`private`に宣言して外部からの直接的なアクセスを防ぐということは常識となっています。`public`で宣言されたフィールドだと、どこでもアクセスできるようになり知らないうちに値が変更される可能性がありますからね。それを防ぐために、Beanのフィールドに直接的なアクセスを許容しなく、Setterメソッドで値を指定してGetterメソッドで値を参照することは暗黙のルールとなっています。

Setterメソッドによってフィールドに直接アクセスせずに値を指定することで最低限の安全は確保したと言いたいところですが、実はそうでもありません。なぜなら、依然としてフィールドの値は何度でも変わる可能性があるからです。一度Setterで値を入れて、その後にまたどこかでSetterを読んでいたら、Beanの持つフィールドの値は上書きされます。

これを防ぐためには、フィールドに`final`を使うべきです。final宣言されたフィールドは、初期化以後にその値が代入されないため、安定性が上がります。final宣言されたフィールドに値を代入しようとするとコンパイルエラーとなるため、エラーを見つけやすいというところも良いですね。

また、全フィールドがfinalで宣言されている場合、`@Data`アノテーションは実質的に`@Value`アノテーションと同じコードを生成します。もちろん、場合によってはfinalではないフィールドを持たせることもできます。そういう場合のコードは以下のようになります。場合によってはこれも必要かもですね！

```java
// @Dataの場合
public class Car {

    // colorだけがfinal
    private String name;
    private final String color;

    // 以下のメソッドたちが自動生成
    public Car(String color) {
        this.color = color;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }

    public String getColor() {
        return this.color;
    }
}
```

ただ、finalとなっているのはあくまでもこのクラスのフィールドのみということに気をつけなければならないです。Carオブジェクトを生成時に使われたデータが、以後も固定されて更新ができなくなります。

## 浅いコピーと深いコピー

Setterメソッドのもう一つの問題は、オブジェクトをコピーした場合に、コピー先のオブジェクトの値を変えるためにSetterを使うとそれがコピー元に影響するということです。簡単に以下のようなコードを作成したとしましょう。

```java
// @DataのCarクラス
Car car1 = new Car();
car1.setName("My car");
car1.setColor("red");

// 友達が私と同じ車を買った
Car car2 = car1;
car2.setName("Your car");

// 出力してみる
System.out.println(car1);
System.out.println(car2);
```

このコマンドをコンパイルして実行してみると、どちらのCarも`name`が`Your car`になっていることを確認できます。なぜこうなったのでしょう？先はStringがImmutableと言っていましたけどね。

これは、`car1`の値だけを`car2`にコピーするという意図から書かれたコードが、実は`car1`と`car2`は「同じメモリーアドレスを参照する」というコードになってしまったからです。同じアドレスから値を参照するため、そのアドレスの値が変わると両方に影響するわけです。

このようにオブジェクトは参照が変わるだけなので、代入だけではそれぞれが独立していると言えなくなります。プリミティブ型が値そのものを保存するので代入でも安全なのとはまた違うところですね。このようにオブジェクトの「参照」だけが変わる状況を「浅いコピー」と言います。

今までの展開から推測できるように、オブジェクトの参照を分離する必要があるでしょう。参照が独立していると、片方の値が変わっても他には影響ないはずですからね。参照がオブジェクトごとに違うというのは、同じ値を持ってメモリーに新しいオブジェクトを生成するということと同じ意味です。そしてこれを「深いコピー」と呼びます。

深いコピーには様々な方法があります。まずはフィールドのオブジェクトを新しく生成することです。例えば、以下のような方法がありますね。

```java
// インスタンスを生成して値を入れてみる
Car car2 = new Car();
car2.setName(new String(car1.getName()));
car2.setName(new String(car1.getColor()));

// 値を変えてみる
car2.setName(new String("Your car"));
```

同じく出力してみると、今回はちゃんと`car2`の値だけが変わったことを確認できます。しかし、全てのフィールドに対してこうするのはあまり便利ではないですね。とこかでメソッド化することはできないのでしょうか？

簡単なのは、インタフェースを利用することです。`Cloneable`を継承することで簡単にオブジェクトをクローンできるようになります。ただ少し、メソッドを作成する必要はありますがね。

```java
@Data
public class Car implements Cloneable {

    private String name;
    private String color;

    // cloneメソッドを作る  
    public Car clone() throws CloneNotSupportedException {
        return (Car) super.clone();
    }
}
```

こうすると、以下のような方法で深いコピーができるようになります。

```java
try {
    car2 = car1.clone();
    car2.setName("Your car");
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
```

ただ、この方法を使うときに注意すべきことがあります。クラスに`clone()`を定義したからって、全てのフィールドに対して深いコピーを保証するわけではないということです。例えば`Car`クラス内にさらに`Engine`のような、Beanクラスがフィールドとして定義されていると、そのフィールドは浅いコピーになる可能性があるということです。これを回避するためには、`Engine`クラスにも`Cloneable`を継承させる必要があります。以下のようにです。

```java
@Data
public class Car implements Cloneable {

    private String name;
    private String color;
    // 追加したフィールド
    private Engine engine;

    // フィールドも深いコピーをさせる
    public Car clone() throws CloneNotSupportedException {
        Car car = (Car) super.clone();
        car.engine = this.engine.clone();
        return car;
    }
}

@Data
public class Engine implements Cloneable {

    private String name;
    private String cylinders;

    // cloneメソッドを作る  
    public Engine clone() throws CloneNotSupportedException {
        return (Engine) super.clone();
    }
}
```

ListやMapの場合はどうクローンしたら良いでしょうか？同じくオブジェクトをクローンする観点からすると、両方とも方式は似ています。ループによるクローンですね。例えば以下のような方法でクローンができます。

```java
// Listのコピー
List<Car> carList1 = new ArrayList<>();
List<Car> carList2 = new ArrayList<>();
for (Car car : catList1) {
    carList2.add(car.clone());
}

// Mapのコピー
Map<String, Car> carMap1 = new HashMap<>();
Map<String, Car> carMap2 = new HashMap<>();
for (Entry<String, Car> entry : carMap1.entrySet()) {
    carMap2.put(entry.getKey(), entry.getValue().clone());
}
```

これでImmutableなクラスを作り、コピーもできるようになりますね。

## Immutableなクラスで注意すること

[Reflectionに関するポスト](../java-reflection)でも紹介したように、Reflectionを使うとフィールドに直接アクセスができ、privateで宣言されていてもアクセスを可能にすることもできます。つまりいくらImmutableなクラスを作ったとしても、Reflectionを使うとフィールドの値を変えることはできるということですね。

そしてImmutableなクラスを作るということは、結局メモリーの使用量が上がるということでもあります。常に新しくオブジェクトを生成し、それぞれがメモリーを占有することになりますからね。もちろん現代のマシンのメモリーは多少のオブジェクトが作られても耐えられるメモリーを持っていますし、GCも活発に動いてくれるので一般的には心配するようなことではないですがね。

## Singletonクラスの変数には注意

Immutableとも関係があることですが、以前紹介した[Singletonクラス](../java-design-pattern-singleton)を作成する場合にも、そのクラスが持つフィールドには注意しなければならないです。このクラスはインスタンスが生成されるとアプリケーションが終了するまで一つのインスタンスが使われるため、フィールドの値が変更される可能性があったら致命的です。処理ごとに結果が変わる可能性があるからです。なのでSingletonクラスにはなるべくフィールドを持たせないようにするか、final宣言をしておくなど、フィールドの値が変わる可能性を最初から封鎖しておく必要があります。

## 最後に

個人的に、プログラミングの始まりが「どう実現するか」だとすると、プログラミングの完成は「どう安定させるか」であると思います。もちろん、性能を改善させたり、維持補修を簡単にさせたりするテクニックも大事ですが、安定的に動作するプログラムを作るということがもっとも難しいものですからね。要件が増えれば増えるほど、コードは複雑になり、例外が発生する可能性も高くなります。Immutableなクラスを作るということは、そのような例外を回避するための一歩であるゆえ、信頼できるプログラムを作り出せるという面で大事な知識なのではないかと思います。

これからもこのような知識に触れ、身につけていきたいですね！
