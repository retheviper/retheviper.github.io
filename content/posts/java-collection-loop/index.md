---
title: "今更なループの話"
date: 2020-11-30
categories: 
  - java
image: "../../images/java.webp"
tags:
  - stream
  - java
  - collection
  - foreach
---

Javaはもともと手続き型な言語ですが、賢い方法で関数型な言語の特徴を受け止めていて、言語の中に共存させていますね。個人的には関数型プログラミングというものに憧れているので、Javaの中でも好んでStreamやLambdaを使っていて、個人的にもKotlinとSpring WebFluxで色々試しているところです。

ただ、Java 1.8から続いている話ですが、`Streamは果たして全てのForループを代替できるのか？`というものがありますね。そしてここでForループを代替ない理由としてよく挙げられているものが、性能・可読性・デバッグの難しさです。つまり内部的により複雑な処理を行っているため性能もよくないし、例外が発生した時も理由を特定するのが難しい上に、多くの人は[Method Chaining](https://en.wikipedia.org/wiki/Method_chaining)とLambdaに慣れてない、ということですね。

以上の理由から、私も普段はやはりList = ArrayListで、ループは拡張For文(たまに、Listの要素をいじって新しいインスタンスを新しく生成する場合はStream)というルールを当たり前のように守ってきていますが、ふとこれで本当に良いのか、いう疑問が湧いてきました。Javaも16までバージョンアップしていて、そろそろ関数型プログラミングに転換しても良い時期なのでは？だったり、自分の知っているものは正しいのかという検証をしてみたいという風にですね。

なので今更な感じではあるのですが、ちょっとしたベンチマークを兼ねて色々検証してみたり、考えてみました(本当は、ベンチマークがしてみたかっただけ)。

## ループの方法

今更な紹介となりますが、そもそも今回のポストが今更な話をしているので、Collectionに関する4つのループ文の細かい話もして行こうかと思います。

多くの場合、Collectionや配列のループ処理の方法は、以下の表通りに使われているのではないかと思います。

| 種類 | 使う場面 |
|---|---|
| For | インデックスが必要な時 |
| 拡張For | 他の方法を取る必要がない場合 |
| Iterator | 基本的に使わない |
| forEach() | 基本的に使わない |

上記のケースが成立する基準は、やっぱり`性能`になっているのではないかと思います。他にも可読性だとか、色々考慮する要素はあると思いますが、何よりも性能が基準として優先されているのは否定できない事実かと思います。なぜなら、他の要素はチューニングが難しいか、できないものであり(例えばセキュリティやバグ防止のためのバリデーションチェックは、効率的なコードに書き換えることはできても、そもそも無くすというのは論外になりますね)、全ての要件が満たされたアプリケーションでリファクタリングにより「目に見える形で」改善できるのは性能しかないからでしょう。そもそも、同じ処理をするなら性能が良い方が絶対いいですし。

なので、私が初めてループ処理に関して学んだ時は伝統的な形のFor文とWhileなのですが、のちにCollectionや配列だと拡張For文を使った方が良いという風に教わりましたが、その時も根拠としてあげられたのが「Forと拡張Forは性能上あまり違わない上に、拡張Forの方が常に要素数分だけループするのが保証されてあるから」ということでした。やはり性能から考えて、それから他のことも考慮するような話ですね。納得のいく話だったので、私自身もそれを信じて今までずっと拡張For文を使ってきました。

でも、実際はどうか検証してみたことはあまりなかったですね。ネットなどで調べてみても、拡張For文は既存のループの書き方を向上させたものであるとか、Iteratorの[Syntax Sugar](https://ja.wikipedia.org/wiki/%E7%B3%96%E8%A1%A3%E6%A7%8B%E6%96%87)に過ぎないとかの話もあリました。聞いた話では、もっとも性能が良いのは

ただ一つ、`Stream`と`forEach()`はどうでしょう。Javaでこれらが導入されてからもさらに時間が立っています。しかし、上述したとおり、依然として`Stream`や`forEach()`は`性能が劣る`から多く使われてないような気がしています(他にも、`あえて使う理由がわからない`・`わかりにくい`などの理由があると思いますが)。最初Java 1.8リリース当時にも、多くの人が性能のテストを行い、少なくとも性能面では既存の方式が有利という結論を出していて、今もそれはあまり変わってないようです。Javaのバージョンも16にまで上がったのですが、それまで行われたチューニングを踏まえても`Stream`や`forEach()`が持つ根本的なアーキテクチャ(?)的な理由から、既存の方式よりも性能が劣るのはしょうがない、という風に認識されています。

しかし、誰かにそう言われたから、そう思うというのはあまり良い考え方ではないでしょう。また、前述のとおり、Javaはすでに16までバージョンアップを重ねていて、大抵の変化というのは新しい機能の追加となっていますが、裏では何かJVMやコンパイラのチューニングなどでなんらかの目に見えない改善があったのかもしれません。関数型としてのコードの書き方に慣れているかどうかは、その人の問題として、性能面で改善されているとしたら、よりモダンな方法を使わない理由がないですね。そして、本当に拡張For文が全ての場合で良いかどうかの検証もあらかじめしておく必要があると思います。

以上の理由から、まず検証で使う4つのループの紹介と、そのベンチマークについて紹介したいと思います。

### For文

まずは伝統的な形のFor文です。一部では`c-style`とも呼ぶらしいですね。一番基本となるもので馴染みもありますが、やはり古い、という印象もあります。端的に、最近のいわゆる`モダン`な言語では、このような形のループは使えない場合もありますね。基本的に以下のような形です。

```java
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}
```

マイクロな最適化として、ループ対象のCollectionや配列の長さをあらかじめ宣言しておく場合もありますね。こうすると、ループ毎にループの対象となるCollectionや配列のサイズを毎回計算する必要がないため、少しは性能が有利になるという話があります。(コンパイラがこれぐらいの最適化は勝手にしてくれるという話もありますが)

```java
int size = list.size();
for (int i = 0; i < size; i++) {
    list.get(i);
}
```

この伝統的なFor文の良いところは、インデックスを基準にして処理をするため、インデックスが必要な場合はなんでもできるということです。例えば、以下のような場合があるでしょう。

```java
// 偶数のインデックスのみ処理をしたい
for (int i = 0; i < list.size(); i += 2) {
    System.out.println(list.get(i));
}

// 条件と一致する要素のインデックスが知りたい
for (int i = 0; i < list.size(); i++) {
    if (list.get(i).length() == 10) {
        System.out.println(i);
    }
}

// 前後の要素と比較したい
for (int i = 1; i < list.size(); i += 2) {
    System.out.println("インデックス" + i - 1 + "の長さ：" + list.get(i - 1).length() + "、インデックス" + i + "の長さ：" + list.get(i).length())
}
```

ただし、For文で指定されてあるインデックスが必ずループ対象の範囲内にあるかどうか、わからなくなる場合もあります。0から始まるインデックスで`i - 1`を指定してしまったり、iの範囲が対象のCollectionや配列よりも大きくなり例外を投げることになることもあるでしょう。また、インデックスを利用した場合、[マジックナンバー](https://ja.wikipedia.org/wiki/%E3%83%9E%E3%82%B8%E3%83%83%E3%82%AF%E3%83%8A%E3%83%B3%E3%83%90%E3%83%BC_(%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%A0))になってしまう可能性もあるので、バグが発生する可能性が上がったり、可読性が悪くなるなどの問題があります。なので、インデックスを基準に処理をしたい場合は慎重にコードを作成する必要がありますね。

### 拡張For文

いわゆる`for-each`文ですね。Colleciton/配列内の全要素を巡回しながら処理するには、これほど理解しやすく、安全なものはないかと思います。

```java
for (String element : list) {
    System.out.println(element);
}
```

最近は、Javaのみでなく他の言語でもこれが標準となっているようです(書き方は言語毎に少し違いますが)。それはつまり、インデックスによるループよりも、ループないで扱うオブジェクトを拡張For文で対象のCollection/配列内の要素に確実に制限した方が色々有利だということでしょう。実際、インデックスといいつつ、伝統的なForb文のものはCollection/配列のインデックスと同じものでもないですので、危険なコードでもありますから。

伝統的なFor文と比べ、拡張For文の中ではインデックスを利用することができないという問題があります。ただ、全く方法がないわけではないです。どうしてもインデックスを拡張For文の中で使いたい場合は、ループの外に定数を宣言するか、Collectionなら利用できる`indexOf()`か、`Collections.binarySearch()`を使う方法があります。

```java
// 定数を利用する方法
int i = 0;
for (String element : list) {
    System.out.println(element + "のインデックス：" + i);
    i++;
}

// indexOf()を利用する場合
for (String element : list) {
    System.out.println(element + "のインデックス：" + list.indexOf(element));
}

// Collections.binarySearch()を利用する場合
for (String element : list) {
    System.out.println(element + "のインデックス：" + Collections.binarySearch(values, value));
}
```

ただ、ループの中で`indexOf()`を使うのはあまり良い選択じゃないです。以下は`ArrayList.indexOf()`の実装になりますが、結局Collectionの中をループしながらインデックスを探すことになるので、実質的に二重ループになっちゃいます。なのでインデックスがどうしても必要な場合は、なるべく定数を使うか、伝統的なFor文を使うべきですね。

```java
// ArrayList.indexOf()
public int indexOf(Object o) {
    return indexOfRange(o, 0, size);
}

int indexOfRange(Object o, int start, int end) {
    Object[] es = elementData;
    if (o == null) {
        for (int i = start; i < end; i++) {
            if (es[i] == null) {
                return i;
            }
        }
    } else {
        for (int i = start; i < end; i++) {
            if (o.equals(es[i])) {
                return i;
            }
        }
    }
    return -1;
}
```

`Collections`の`binarySearch()`を利用する場合も、結局ループしながらインデックスを探すというのは変わりませんので注意を。以下はその実装です。

```java
// Collections.binarySearch()
public static <T> int binarySearch(List<? extends Comparable<? super T>> list, T key) {
    if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
        return Collections.indexedBinarySearch(list, key);
    else
        return Collections.iteratorBinarySearch(list, key);
}

private static <T> int indexedBinarySearch(List<? extends Comparable<? super T>> list, T key) {
    int low = 0;
    int high = list.size()-1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        Comparable<? super T> midVal = list.get(mid);
        int cmp = midVal.compareTo(key);

        if (cmp < 0)
            low = mid + 1;
        else if (cmp > 0)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found
}

private static <T> int iteratorBinarySearch(List<? extends Comparable<? super T>> list, T key) {
    int low = 0;
    int high = list.size()-1;
    ListIterator<? extends Comparable<? super T>> i = list.listIterator();

    while (low <= high) {
        int mid = (low + high) >>> 1;
        Comparable<? super T> midVal = get(i, mid);
        int cmp = midVal.compareTo(key);

        if (cmp < 0)
            low = mid + 1;
        else if (cmp > 0)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found
}
```

### Iterator

Iteratorは、個人的にあまり馴染まない(使いたくない)ものです。どのCollectionでもIteratorとして抽出ができてしまうので、CollectionよりもIteratorが主体になってしまうような感覚であり、定型文な書き方を矯正しているからです。少なくとも拡張ForではどんなCollectionのどんな要素を抽出して使っているのか明確ですが、Iteratorだとそれがわからないですね。

とにかく、そんなIteratorですが、ForでもWhileでもループをかけるという特徴があります。

```java
// Forを使う
for (Iterator iterator = values.iterator(); iterator.hasNext(); ) {
    System.out.println(iterator.next());
}

// Whileを使う
Iterator iterator = values.iterator();
while (iterator.hasNext()) {
  System.out.println(iterator.next());
}
```

Iteratrorを使う場合の問題は、使い方がいまいち直感的ではないということです。例えば以下のような例をみましょう。`getFoo()`と`getBar()`は、同じオブジェクトから呼ばれているように勘違いしやすいのではないでしょうか。

```java
for (Iterator iterator = list.iterator(); iterator.hasNext(); ) {
    System.out.println(iterator.next().getFoo());
    System.out.println(iterator.next().getBar()); // 注意！
}
```

面白いことに、拡張For文のバイトコードは、Iteratorを使うコードになるということです。なので少なくとも拡張For文は、Iteratorよりは発展した形と言えるのかもしれません。

### forEach()

モダンな書き方としてのforEach()ですね。拡張For文とあまり違わないのですが、Lambdaやメソッド参照が使えるというメリットがありますね。また、Kotlinのスコープ関数のように、処理の範囲がはっきりするという意味で良いのかもしれません。何よりコードが短くなるのが好きですね。

```java
list.forEach(System.out::println)
```

実装としても、拡張For文の中でLambdaを実行するという単純な構造になっています。なので単純に考えて、拡張For文よりは性能が劣る可能性がありますね。以下はIterableの実装です。

```java
// IterableのforEach()
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```

ただ、ArrayListの場合は実装が大きく違います。なので性能も大きく変わる可能性がありますね。以下はその実装です。

```java
// ArrayListのforEach()
public void forEach(Consumer<? super E> action) {
    Objects.requireNonNull(action);
    final int expectedModCount = modCount;
    final Object[] es = elementData;
    final int size = this.size;
    for (int i = 0; modCount == expectedModCount && i < size; i++)
        action.accept(elementAt(es, i));
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}

static <E> E elementAt(Object[] es, int index) {
    return (E) es[index];
}
```

## ベンチマークで検証してみると

この度も、JMHを使って簡単なベンチマークを作ってみました。実はstatic finalなフィールドとして宣言しておくとそのオブジェクトを全てのベンチマークで使い回してくれるのかな、と思っていましたが、どうやらそうではなかったみたいです。なので今回は、ちゃんと@Setupアノテーションを使ってフィールドの初期化をしてみました。実際のコードは以下のとおりです。

```java
@State(Scope.Thread)
public class LoopTest {

    private List<String> values;

    @Setup
    public void init() {
        final DecimalFormat format = new DecimalFormat("0000000");
        values = IntStream.rangeClosed(0, 9999999).mapToObj(format::format).collect(Collectors.toList());
    }

    @Benchmark
    public void indexLoop(Blackhole bh) {
        final int length = values.size();
        for (int i = 0; i < length; i++) {
            bh.consume(values.get(i));
        }
    }

    @Benchmark
    public void iteratorLoopFor(Blackhole bh) {
        for (Iterator iterator = values.iterator(); iterator.hasNext(); ) {
            bh.consume(iterator.next());
        }
    }

    @Benchmark
    public void iteratorLoopWhile(Blackhole bh) {
        final Iterator iterator = values.iterator();
        while (iterator.hasNext()) {
            bh.consume(iterator.next());
        }
    }

    @Benchmark
    public void extendedLoop(Blackhole bh) {
        for (String value : values) {
            bh.consume(value);
        }
    }

    @Benchmark
    public void forEachLoop(Blackhole bh) {
        values.forEach(bh::consume);
    }
}
```

そしてベンチマークの結果は、以下のとおりです。

```dos
Benchmark                    Mode  Cnt   Score   Error  Units
LoopTest.indexLoop          thrpt   25  27.737 ± 0.475  ops/s
LoopTest.iteratorLoopFor    thrpt   25  26.968 ± 0.556  ops/s
LoopTest.iteratorLoopWhile  thrpt   25  27.250 ± 0.557  ops/s
LoopTest.extendedLoop       thrpt   25  13.186 ± 0.152  ops/s
LoopTest.forEachLoop        thrpt   25  12.479 ± 0.104  ops/s
```

やはり、4つのループがそれぞれ違う結果を見せているのがわかります。少なくとも、ここでは伝統的なFor文を使った方がもっとも性能の面では有利のように見えますね。なるべく拡張For文を使った方が良い、という根拠として`性能はあまり変わらないから`というのはなんだったんだろう、と思うくらいの差があります。

しかし、本当にこれで、`性能が良い方を選べば良い`という結論を出して良いのでしょうか？

## 考えたいこと

処理としての結果が同じだとしたら、やはり性能の良い方を選びたくなるのは当然です。企業レベルの話だと、性能は費用と直結する問題でもありますしね。しかし、複雑化している現代のアプリケーションで考えるべきは、性能のみではありません。極端的な話だと、性能のためにをC、C++でWebアプリケーションを作るとしたら、他の言語に比べて生産性が下がってしまうでしょう。そして可読性や維持保守を考えず、性能を優先したコードだけを書いていくと、いわゆる[スパゲティコード](https://ja.wikipedia.org/wiki/%E3%82%B9%E3%83%91%E3%82%B2%E3%83%86%E3%82%A3%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%A0)になってしまう可能性もあります。

なので、性能のみではなく、アプリケーションを開発するときには色々と考慮すべき要素があるのは確かです。例えば、Readability(可読性)、Error-proneness(エラー発生可能性)、Capability(処理能力)などがあるでしょう。今までは性能ばかりの話をしてきましたが、これらの観点から4つのループを比較してみたらどうでしょう。

### 可読性とエラー発生可能性の側面から考える

拡張For文(forEach())では、Collectionそのものの要素をことができます。逆に、For文やIteratorでは可能ですね。ならば、Collectionや配列の中でとある条件と一致する要素だけのことしたい場合は、拡張For文よりもFor文やIteratorを使うべきであるようにも見えます。

しかし、観点を変えてみると、元のオブジェクトそのものが変わることで起こり得るサイドエフェクトが発生する場合も考えられます。こういう場合、元のオブジェクトを直接操作できるということはメリットではなくデメリットになってしまいますね。なので、どちらかというと、与えられたCollection/配列から条件に一致する要素だけを抽出して新しいCollection/配列のインスタンスを生成するのが正解の可能性もあります。そしてそれをよりわかりやすいコードとして実現できるのは拡張For文(forEach())ですね。例えば、以下のようにです。

```java
// 元のリストが変わってしまう
public void filterFor(List<String> list) {
    for (int i = 0; i < list.size(); i++) {
        if (list.get(i).length() > 10) {
            list.remove(i);
        }
    }
}

// 元のリストには影響がない - For文
public List<String> filterFor(List<String> list) {
    List<String> result = new ArrayList<>();
    for (int i = 0; i < list.size(); i++) {
        String element = list.get(i);
        if (element.length() <= 10) {
            result.add(element);
        }
    }
    return result;
}

// 元のリストには影響がない - 拡張For文
public List<String> filterForEach(List<String> list) {
    List<String> result = new ArrayList<>();
    for (String element : list) {
        if (element.length() <= 10) {
            result.add(element);
        }
    }
    return result;
}

// 元のリストには影響がない - Stream.forEach()
public List<String> filterStream(List<String> list) {
    return list.stream().filter(element -> element.length() <= 10).collect(Collectors.toList());
}
```

良いコードは、短く、わかりやすいコードなのではないかと思っています。そしてわかり安いコードは、誰がメンテしてもバグを起こす可能性は低くなるはずでしょう。そういう観点からすると、伝統的なFor文とIteratorは、今は使うべきではないのかもしれません。

### 処理能力の側面から考える

処理能力、というのはある程度性能ともつながるものですね。なので、性能という側面でもう一度考えてみます。互換性、汎用性などとも言える物かもしれません。ここで言いたいのは、Collection/配列がどんなものであれ、一定の性能を保証する実装を考える必要があるということです。

引数として`List`をとり、なんらかの処理をループで行うメソッドを実装するとしましょう。今まであげてきた、4つのループのパターンのうちどれを選ぶべきかは、その引数の実装クラスが何になるかわからない、という面も考慮する必要があります。なぜなら、Listは色々な実装クラスを持つインタフェースだからです。

引数としてListをまず宣言しておくと、言語の仕様としてはListの実装クラスはどれでも許容することになりますね。なので引数として入ってくるのは[ArrayList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/ArrayList.html)になる可能性もあり、[LinkedList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/LinkedList.html)にな可能性もあり、極端的には[AbstractList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/AbstractList.html)で個人がカスタマイズしたものが来る場合もあると予想できます。他にもJava 11を基準に、java.util.Listを継承しているCollectionの実装クラスの場合、例えば[AbstractSequentialList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/AbstractSequentialList.html)、[AttributeList](https://docs.oracle.com/en/java/javase/11/docs/api/java.management/javax/management/AttributeList.html)、[CopyOnWriteArrayList](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CopyOnWriteArrayList.html)、[RoleList](https://docs.oracle.com/en/java/javase/11/docs/api/java.management/javax/management/relation/RoleList.html)、[RoleUnresolvedList](https://docs.oracle.com/en/java/javase/11/docs/api/java.management/javax/management/relation/RoleUnresolvedList.html)、[Stack](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Stack.html)、[Vector](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Vector.html)などがあって、これらが全部Listになりえるので、どの実装でも対応する必要があります。

もちろん、Javaでとあるインタフェースを継承するということは、処理の前提条件と結果結果が明確であることと同じ意味なので、実装クラスが変わったって、処理の結果が大きく変わることはないです。ただ、Listの実装クラスが複数存在するのは、それらを使う目的によってどちらかに偏ったパフォーマンスを見せるためであることをまず理解する必要がありますね。これはつまり、同じ条件下でも実装クラスによって処理の性能が大きく変わる可能性があるということです。一般的に多く使われているListの実装クラスはArrayListですが、参照以外の性能が劣るという理由からLinkedListが使われる場面もあると予想できます。だとすると、ArrayListで性能がよかったものがLinkedListでもそうとは限らないものですね。

上記で実施したベンチマークだけをみて、性能は絶対これが有利だ、と言いきれない理由がここにあります。なぜなら、テストようのデータを`Collectors.toList()`を使ってListとして作成していますが、以下のコードでわかるように、常にArrayListが生成されているからです。

```java
public static <T> Collector<T, ?, List<T>> toList() {
    return new CollectorImpl<>((Supplier<List<T>>) ArrayList::new, List::add,
                                (left, right) -> { left.addAll(right); return left; },
                                CH_ID);
}
```

なので、ついでに他の実装クラスのベンチマークもしてみることにしました。ただ、Listの実装クラスを全部テストするのは無理があるので(特に、AbstractListやAbstractSequentialListは別途実装が必要ですし、CopyOnWriteArrayListはマルチスレッドでないと意味がないし、RoleListやVectorはほとんど使われてなく、Stackをループで利用するとは思わないので)、LinkedListの場合はどうかだけ確認してみました。まあ、ArrayListと違う反例は一つだけあったら十分ですしね。

幸い、Collectorsには`toCollection()`でCollectionの実装を指定できます。なので、上記のベンチマークのコードから、以下のような修正を入れるだけでListの実装を変えることができます。

```java
// LinkedListの場合
values = IntStream.rangeClosed(0, 9999999).mapToObj(format::format).collect(Collectors.toCollection(LinkedList::new));
```

LinkedListの場合、要素数が増えると急激に性能が低下する傾向があります。なので、ArrayListの時よりも要素数は2桁ほど減らしてベンチマークを実施しました。結果は以下です。

```bash
Benchmark                    Mode  Cnt    Score    Error  Units
LoopTest.indexLoop          thrpt   25    0.084 ±  0.005  ops/s
LoopTest.iteratorLoopFor    thrpt   25  854.459 ± 36.771  ops/s
LoopTest.iteratorLoopWhile  thrpt   25  839.233 ± 18.142  ops/s
LoopTest.extendedLoop       thrpt   25  659.999 ± 47.702  ops/s
LoopTest.forEachLoop        thrpt   25  780.463 ± 78.591  ops/s
```

ArrayListとは真逆の結果になったのがわかります。特に、インデックスによるループは使えるものにならないほど性能が低く、拡張For文よりも`forEach()`の性能が高いという、意外の結果となっています。このベンチマークでの数値が絶対的なものとは言えませんが、結果から推論できるのは、やはりArrayListのインデックスを利用する伝統的なFor文でのループが一番早かったからという理由だけで、全てのListをFor文で処理するというのは危ないということです。なので「どの実装クラスでも、平均的に良い性能を出してくれる」方式を選ぶ必要があるという結論を出せるでしょう。(それがおそらく拡張For文な気がします)

## 最後に

全ての場面で最適なコードを書くのは難しいことで、過去に書いたコードはいずれ改善しなければならないものとなりますね。あまりエンジニアとしての歴の長くない自分でも、たまに入社前のコードをみるとびっくりするくらいです。なんとか動くようなものは作ったものの、重複するコードや無駄なインスタンス作りなど、至る所に自分のミスが散らかっています。

なのでたまには、そのような過去の自分が書いたコードと向かい合って、それを直してみるのも良い経験になるのではないかと思ったりもします。特に今回みたいに、ループ処理は基本の中の基本ですが、その処理すらどれを選ぶかよくわかってないまま(そして副作用などは考えず)、ただひたすら書いてしまったものも多かったので、それに対する反省を兼ねて、そして自分の思うことの根拠を探すための勉強にもなりますので。そしてベンチマーク、意外と楽しいですので。これで自分の理論を証明していくのも良い経験ですね。

では、また！
