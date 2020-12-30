---
title: "9からの新メソッドめぐり"
date: 2020-12-30
categories: 
  - java
photos:
  - /assets/images/sideimage/java_logo.jpg
tags:
  - java
  - stream
  - collection
  - optional
  - string
---

仕事ではJava 11を扱うことが多いのですが、正直、自分の書いたコードを振り返ってみると、Java 9から新しく追加されたメソッドはあまり使ってないのが現実です。しかし、これら新しいメソッドたちは冗長さを隠してくれるシンタックスシュガーとして存在するだけでなく、性能や機能面でより優れているものもあるので、いますぐ使わないとしても目は通しておきたいものが多いなと思っています。

2021年は次のLTSバージョンとなる17の登場が予告されている時点なので、今更な感はありますが、そろそろ私もSEになってから2年になるので、今回は今年、自分が書いたコードへの反省を含め、Java 9〜11まで新しく追加されたメソッドたちの中から、良さそうな(よく使えそうな)ものを選別してみました。そして今回のポストは、そう選別したメソッドの簡単な紹介となります。

多くの場合、これらのメソッドを使える環境だとしたらJava 11を導入しているはずなのであまり意味はないのかも知れませんが、それぞれのメソッド名の右に、該当メソッドがどのバージョンから導入されたかを記入していますので参考にしてください。

## Stream

StreamこそJava 8のキモではないかと思います。そしてJava 9では、そのStreamの問題を色々と改善したり、より簡単に使えるようなメソッドを用意しています。なので、既存のforループにしか慣れてない人でも、簡単に入門できるようになったのではないかと思います。

### Iterate (9)

`iterate()`というメソッド名だけではすぐに意味がわからない場合もあると思いますが、このメソッドは伝統的なFor文と同じような構文でStreamでの処理を書くことができるようになります。つまり、「初期化・ループの継続条件・カウンタ変数の更新」を書くことで、Streaｍの要素数を決めることができるという意味です。例えば、以下のような書き方ができます。

```java
// 0~9までを出力
Stream.iterate(0, i -> i < 10, i -> i + 1).forEach(System.out::println);
```

これはつまり、以下のコードと同じ意味を持ちます。

```java
for (int i = 0; i < 10; i++) {
    System.out.println(i);
}
```

ただ、`iterate()`で指定できる初期化の値が数字という制限はないので(`T`です)、以下のようなこともできます。

```java
// Aで三角形を出力
Stream.iterate("A", s -> s.length() < 10, s -> s + "A").forEach(System.out::println);
```

また、ループの継続条件を指定しないこともできます。

```java
// Aで三角形を出力
Stream.iterate("A", s -> s + "A").forEach(System.out::println);
```

継続条件を指定しないと、無限ループになってしまうのでは？と思われそうですね。確かにそうですが、同じくJava 9でStreamの要素数の上限を指定できる新しいメソッドが追加されています。次に紹介するものがそれです。

### takeWhile (9)

以前、Streamの問題として「途中でやめられない」と書きましたが、Java 9から導入された`takeWhile()`メソッドを使うと、途中で処理を終了するようなことができるようになりました。既存にあった`limit()`の場合は、「指定された回数分」という限界がありましたが、こちらはPredicate型の条件を指定できるというところが違います。

```java
// AAAAAAAAAまで出力する
Stream.iterate("A", s -> s + "A")
    .takeWhile(s -> s.length() < 10)
    .forEach(System.out::println);
```

なので、`iterate()`の継続条件を書いてない場合には`takeWhile()`を使ってどの条件で処理が終わるかを明示した方が良いですね。

### dropWhile (9)

`dropWhile()`は、その名からも推測できますが、`takeWhile()`と真逆の機能をするメソッドです。このメソッドはStreamから与えられた条件と一致する要素を除いて、残りの要素を返却します。

```java
// AAAAAから出力する
Stream.iterate("A", s -> s.length() < 10, s -> s + "A")
    .dropWhile(s -> !s.contains("AAAAA"))
    .forEach(System.out::println);
```

### ofNullable (9)

Java 1.8のStreamでは、Null要素を追加するためにはまずその要素がNullかどうかをチェックして、Nullの場合に`Stream.empty()`を呼ぶような形にする必要がありました。いつものJavaのNullチェックですね。例えば以下のようなものです。

```java
// 要素のNullチェックを含むStreamのCollect
keyList.stream()
    .flatMap(k -> {
        Object value = map.get(k);
        return value != null ? Stream.of(value) : Stream.empty();
    })
    .collect(Collectors.toList());
```

これを、Java 9ではより簡単なコードで書くことができます。`Optional`の`ofNullable()`とあまり変わらない感覚ですね。

```java
keyList.stream()
  .flatMap(k -> Stream.ofNullable(map.get(k)))
  .collect(Collectors.toList());
```

## Collectors

Streamの要素を集約するためのCollectorを提供する`Collectors` APIですが、こちらの変化は主にシンタックスシュガーなものが多い印象です。主にStreamでしかできなかったことや、既存のCollectorsのみだとかなり長くなるコードを簡潔に書くことができるようになっています。

### filtering (9)

`Stream`の`filter()`と同じ処理を、`Collector`でもできるようになりました。どちらを使うかは好みの問題な気がしますが、`Collector`そのものを共通化するなどの処理ができそうな気はしますね。

```java
// 0~9までのリスト
List<Integer> numbers = Stream.iterate(0, i -> i < 10, i -> i + 1).collect(Collectors.toList());

// Stream.filter()
numbers.stream()
    .filter(e -> e > 5)
    .collect(Collectors.toList()); // 6, 7, 8, 9

// Collectors.filtering()
numbers.stream()
    .collect(Collectors.filtering(e -> e > 5, Collectors.toList()));  // 6, 7, 8, 9
```

### flatMapping (9)

これもまた名前から推測できると思いますが、`Collectors`でCollectionに変えるとき、要素のflatMappingをできるようにしてくれるようなものです。具体的には、以下のサンプルコードを参照してください。

例えば、以下のようなクラスがあるとします。

```java
public class Selling {
   String clientName;
   List<Product> products;
}
public class Product {
   String name;
   int value;
}
```

そして、このSellingのリストを、「clientNameをKeyに、productsをValueにしたMapにしたい」場合はどうしたら良いでしょうか。例えば以下のような方法を考えられます。

```java
Map<String, List<List<Product>>> result = operations.stream()
                .collect(Collectors.groupingBy(Selling::getClientName, Collectors.mapping(Selling::getProducts, Collectors.toList())));
```

しかし、問題は、`List<Product>`をさらにListの中に入れてしまうことになります。これは本来の目的ともズレていて、無駄な処理が発生し、Valueを持ち出すときも不便なはずです。

これを`Map<String, List<Product>>`の形に変えるとしたら、以下のような方法が使えます。自作のCollectorを作るのですね。

```java
Map<String, List<Product>> result = operations.stream()
    .collect(Collectors.groupingBy(Selling::getClientName, Collectors.mapping(Selling::getProducts,
        Collector.of(ArrayList::new, List::addAll, (x, y) -> {
            x.addAll(y);
            return x;
        }))));
```

ただ、毎回このような自作Collectorを作るというのはあまり効率的ではない方法ではないかと思います。それに、自作のCollectorを普段から使ってない場合はコードだけみても少しわかりづらくもありますね。なので、ここは新しく追加された`flatMapping()`で変えてみると以下のようになります。より簡潔ですね。

```java
Map<String, List<Product>> result = operations.stream()
    .collect(Collectors.groupingBy(Selling::getClientName, Collectors.flatMapping(selling -> selling.getProducts().stream(), Collectors.toList())));
```

### toUnmodifiable (10)

Java 10では`Collectors`に以下の三つのメソッドが追加されています。

- `toUnmodifiableList()`
- `toUnmodifiableSet()`
- `toUnmodifiableMap()`

これらのメソッドを使うと、既存の`Collections`を呼ぶ必要なく、簡単に(もっと短いコードで)UnmodifiableなCollectionを作ることができます。

```java
// Collections.unmodifiableList
List<Integer> collectionsUnmodifiable = Collections.unmodifiableList(Stream.iterate(0, i -> i < 10, i -> i + 1).collect(Collectors.toList()));

// Collectors.toUnmodifiableList
List<Integer> collectionsUnmodifiable = Stream.iterate(0, i -> i < 10, i -> i + 1).collect(Collectors.toUnmodifiableList());
```

引数は、既存の`toList()`・`toSet()`・`toMap()`と同じなので(`toMap()`だけ、KeyとValueのマッピングを指定する必要がありますね)、既存のメソッドと同じ感覚で使うことができます。

## Collections

Collections APIの新しいメソッドは、かなり現代的な書き方を可能にします。Kotlinのような言語がJavaの冗長さを回避するための工夫をしているのであれば、Java側に新しく追加されたメソッドはそれをさらにJavaに似合うような形で受け入れたような印象です。(というか、それしか方法はなかったかも知れませんが…)

### Factory Method (9)

Java 9では、[ファクトリーメソッド](https://ja.wikipedia.org/wiki/Factory_Method_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3)でCollectionの作成ができるようになりました。使い方としては、既存の`Arrays.asList()`と似ているような感覚です。

```java
// Listの作成
List<String> list = List.of("A", "B", "C");

// Setの作成
Set<Integer> set = Set.of(1, 2, 3);
```

Mapの場合は、KeyとValueを順番に並ぶことでインスタンスを作成できますが、エントリーセットを定義することもできます。

```java
// KeyとValueのセットで定義する
Map<String, String> map = Map.of("foo", "a", "bar", "b", "baz", "c");

// エントリーセットを定義する
Map<String, String> map = Map.ofEntries(
  new AbstractMap.SimpleEntry<>("foo", "a"),
  new AbstractMap.SimpleEntry<>("bar", "b"),
  new AbstractMap.SimpleEntry<>("baz", "c"));
```

これらのファクトリーメソッドで作成したCollectionの特徴は、最初からUnmodifiableなオブジェクトになるということです。なので、例えばアプリケーションの起動時にフィールドに定数をCollectionとして定義する、という場合に使えます。つまり、以下のような既存のコードを代替できるようなものです。

```java
// もっとも基本的な方式
Set<String> set = new HashSet<>();
set.add("foo");
set.add("bar");
set.add("baz");
set = Collections.unmodifiableSet(set);

// Double-brace initialization 
Set<String> set = Collections.unmodifiableSet(new HashSet<String>() {{
    add("foo"); add("bar"); add("baz");
}});
```

また、このファクトリーメソッドで作ったCollectionは以下のような特徴を持ちますので、必要に応じて使うのが大事ですね。

- Immutable(Unmodifiable)になる
- Null要素を指定できない
- 要素がSerializableだとCollectionもSerializableになる

#### copyOf (10)

List, Set, Mapに`copyOf()`というメソッドが追加されています。引数にそれぞれのCollectionを渡すと、Unmodifiableなコピーすることができます。

```java
// コピー元のリスト
List<String> original = ...

// コピーする
List<String> copy = List.copyOf(original);
```

## Optional

Optionalは積極的に使われていますか？私の場合は、Streamが返すもの以外で、自分でOptionalを使う場合はあまりないです。色々制約が多いので、複雑なNullチェックが必要な場合ではないと使いづらい気もしますね。ただ、9と10で追加されたメソッドでかなり便利に使えるものとなったので、たまには良いのかも知れません。

### or (9)

Optionalの中身がNullの場合に実行されるメソッドです。既存の`orElse()`や`orElseGet()`と何が違うかというと、こちらはOptionalの中身ではなく、またのOptionalを返すということです。引数としてはSupplierをとります。

```java
String string = null;
Optional<String> optional = Optional.ofNullable(string);
System.out.println(optional.or(() -> Optional.of("default")).get()); // "default"
```

### orElseThrow (10)

Optionalの中身がNullの場合は例外を投げる分岐です。NullのOptionalはもともと`NoSuchElementException`を投げますが、ビジネスロジックなどによりカスタマイズした例外を投げたい場合などはこちらを使えますね。引数としてはSupplierをとります。

```java
String string = null;
Optional<String> optional = Optional.ofNullable(string);
String throwing = optional.orElseThrow(RuntimeException::new); // RuntimeException
```

### ifPresentOrElse (9)

Optionalの中身がNullかどうかによって二つのアクションを指定して、分岐処理ができるようなメソッドです。第一引数としてはConsumerを指定することで中身がNullではない場合の処理を、第二引数としてはRunnableとして中身がNullだった場合の処理を書きます。

```java
Optional<String> hasValue = Optional.of("proper value");
hasValue.ifPresentOrElse(v -> System.out.println("the value is " + v), () -> System.out.println("there is no value")); // the value is proper value

Optional<String> hasNoValue = Optional.empty;
hasNoValue.ifPresentOrElse(v -> System.out.println("the value is " + v), () -> System.out.println("there is no value")); // there is no value
```

### stream (9)

Optionalを要素が一つか、Null(`Stream.empty()`)のSteamに変えるメソッドです。もともとStreamから要素を取得するときもOptionalになっていたので、このようなメソッドが追加されたのも当たり前といえば当たり前ですね。要素が多くて一つなのにStreamに変える意味があるかというと、他のStreamと結合ができたりもするので色々と活用できる余地はありそうです。

```java
Optional<String> optional = Optional.of("value");
Stream<String> stream = optional.stream();
```

## String

String APIの場合は、主にJava 11でかなりの変化がありました。Webアプリケーションのみならず、最近のアプリケーションは文字列を扱う場合が多いので、このような変化はありがたいですね。

### repeat (11)

指定した数値分、文字列を繰り返します。同じ文字列の単純な繰り返しだとすると、StringBuilderやStrinbBufferなしでも簡単に使えるこちらのメソッドの方が良いですね。

```java
String a10 = "A".repeat(10); // "AAAAAAAAAA"
```

### strip (11)

文字列の前後の空白を除外するために、今までは`trim()`を使うケースが多かったのではと思いますが、Java 11からは`strip()`が追加され、`trim()`を代替できます。この二つが何が違うかというと、まずそれぞれのメソッドで定義している「空白」が違います。`trim()`はUnicodeを考慮してなかったため、半角スペースのみの対応となっていましたが、`strip()`はUnicodeで指定されたWhitespace全部を対象とするので、全角スペースや改行にも対応できます。どの文字がWhitespaceとして扱われるかは、`Character.isWhitespace()`のメソッドが基準となるので、[そちらのJavaDoc](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/Character.html#isWhitespace(char))を参照してください。

```java
String stripped = "\n  hello world  \u2005".strip(); // "hello world"
```

また、`strip()`では前後の空白が全部削除されるのですが、文字列の前後を基準にして片方だけ削除したい場合は、前からだけを削除する`stripLeading()`や後ろからだけを削除する`stripTrailing()`も使えます。

```java
String stripLeading = "\n  hello world  \u2005".strip(); // "hello world   "
String stripTrailing = "\n  hello world  \u2005".stripTrailing(); // "\n  hello world"
```

今までの説明だけでも十分`strip()`を使う理由はあると思いますが、実はもう一つがあります。性能です。性能という面では`strip()`の方が`trim()`より[5倍も早い](https://stackoverflow.com/questions/53640184/why-is-string-strip-5-times-faster-than-string-trim-for-blank-string-in-java)と言われているので、なるべく`trim()`よりは`strip()`を使うべきですね。

### isBlank (11)

すでに`isEmpty()`というメソッドがありますが、このメソッドと`isBlank()`の違いは、`trim()`と`strip()`の関係と似ています。同じく、`isEmpty()`と比べると`isBlank()`の方がUnicodeに対応しているのでより多くのケースのWhitespaceに対応できて、性能でも優れています。

```java
boolean isEmpty = "\n    \u2005".isEmpty(); // false
boolean isBlank = "\n    \u2005".isBlank(); // true
```

### lines (11)

文字列に改行コード(`\n`・`\r`・`\r\n`)を基準に分けた`Stream<String>`を返却します。

```java
String multipleLine = "first\nsecond\nthird";
long lines = multipleLine.lines().filter(String::isBlank).count(); // 3
```

## Prediacte not (11)

LambdaやMethod Referenceで定義したPredicateの結果がFalseかどうかを判断するためのメソッドです。単純にtrueの否定になるだけなのですが、このメソッドの引数はPredicateなので、LambdaやMethod Referenceを使ってより単純に表現できるのがメリットといえますね。

```java
// 否定の条件式を使う場合
list.stream()                       
    .filter(m -> !m.isPrepared())
    .collect(Collectors.toList());

// Predicate.not()を使う場合
list.stream()                          
    .filter(Predicate.not(Man::isPrepared))
    .collect(Collectors.toList());
```

## 最後に

2021年に次のLTSであるJava 17がリリースされると、今のJava 11を使う現場の場合は多くがJava 17に移行するのではないかと思います。12から16まで、さまざまなAPIや機能、JVMの改善などが含まれていて、すでに多くのブログなどで紹介されていますが、また既存のAPIにはどのような変化があるかまでは完全に把握していない状態です。なので、Java 17のリリースに合わせて、もう一度12〜17までの新しいメソッドの整理と紹介を行おうと思います。これだけでもかなり勉強になりますし、業務で使えそうなテクニックも増えていく感覚ですね。

また、今年のポスティングはこれで終了となります。色々と大変な一年だったのですが、なんとか年末を迎えることができましたね。その間、このブログにも多くの方々がいらしてくださいました。まだジュニアレベルでしかない駆け出しエンジニアのブログなのであまり情報取集には役立たないかも知れませんが、少しでも私の書いたポストを読んでくださりありがとうございます。来年からは、より面白く、より良い情報を収取してブログに載せたいですね。

では、また！