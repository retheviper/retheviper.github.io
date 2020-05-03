---
title: "Javaの色々なコーディングスキル"
date: 2019-11-17
categories: 
  - java
photos:
- /assets/images/sideimage/java_logo.jpg
tags:
  - java
---

今回のポストでは、スキルといっても大したものはないですが、自分がコードを書きながらこれは便利だな(もしくは単に格好いいな)と思ったコーディングスキル的なものをいくつか集めました。

## StreamでListの変換

以下のような二つのクラスがあるとしましょう。コードの量を減らすためLombokを使っていると仮定します。

```java
@Getter
public class Item {

    private String value;
}

@Setter
public class Product {

    private String value;
}
```

業務的にこの二つのクラスのフィールドである`value`が同一なものと仮定します。そうすると、Itemクラスから`value`を取得してProductクラスに取り込むことが必要な状況もあるでしょう。このような場合、オブジェクトがそれぞれ一つだけだとあまり問題にならないですね。

```java
public Product setProductValueFromItemValue(Item item) {

    Product product = new Product();
    product.setValue(item.getValue());
    return product;
}
```

複数の項目をマッピングする必要があるなら、[ModelMapper](http://modelmapper.org/)のようなライブラリを使う方法もあります。同じような名前のGetter/Setterがあると自動でマッピングしてくれるので便利ですね。

```java
public Product setProductValueFromItemValue(Item item) {

    ModelMapper mapper = new ModelMapper();
    Product product = mapper.map(item, Product.class);
    return product;
}
```

しかし、これらがListやMapに入っているとどうするのでしょうか。普通にFor文のなかで同じくマッピングをさせる方法がありますね。

```java
public List<Product> itemListToProductList(List<Item> itemList) {

    List<Product> productList = new ArrayList<>();
    for (Item item : itemList) {
        productList.add(mapper.map(item, Product.class));
    }
    return productList;
}
```

これをStreamとLambdaを利用して、より簡単なコードにすることができます。

```java
public List<Product> itemListToProductList(List<Item> itemList) {

    List<Product> productList = itemList.stream().map(item -> mapper.map(item, Product.class)).collect(Collectors.toList());
    return productList;
}
```

やっていることはFor文とあまり変わりません。元のリストから要素を一つづつ取り出し、マッピングして新しいオブジェクトを作る。そしてそれを取り集めて新しいListを生成していますね。ただ、`map()`の引数はLambdaなのでただのマッピングだけでなく、より複雑な処理を入れることもできます。同じことをするとしてもより簡単で短いコードが完成されます。

## CollectionでImmutable

Immutable、すなわち不変クラスについては[前回のポスト](../../../08/25/java-thinking-of-immutable)でも扱いました。今回はCollectionを使って、そのクラスのListやMapもImmutableにする方法について述べます。下のコードは、Listの例です。

```java
public List<Item> returnAsUnmodifiableList(List<Item> list) {

    return Collentions.unmodifiableList(list);
}
```

同じ方法で、`Collentions.unmodifiableMap()`でラッピングするとMapもImmutableにできます。こう変換されたListやMapは変更が不可能になるため設定系などのデータを持っている場合に有効活用できます。ただ、Nullが入るとNullPointerExceptionが発生するため注意しましょう。包みたいListがNullになる可能性がある場合は`Collection.emptyList()`を代わりに入れることができます。

逆に、ImmutableになったListやMapを変更したい場合は新しいオブジェクトに複製します。

```java
public List<Item> returnAsModifiableList(List<Item> list) {

    return new ArrayList<>(list);
}
```

ただ、こうしてオブジェクトを複製してデータを変更する場合、元のImmutableなListにも反映されるので注意する必要があります。

## カスタムクラスをIterableにする

とあるクラスの中に、子要素のクラスがListとして入っているとします。例えば、以下のようなイメージです。

```java
public class Container {

    private List<Baggage> baggages = new ArrayList<>();
}
```

場合によっては、このクラスの中の子要素を全部取り出してFor文を書きたい場合もあるはずです。そういう場合は普通、こんな形で使うのではないかと思います。

```java
public void printBaggageNames(Container container) {

    List<Baggage> baggages = container.getBaggages();
    for (Baggage baggage : baggages) {
        System.out.println(baggage.getName());
    }
}
```

でも、このクラス自体を拡張For文のなかで使えるとしたら、より便利になりますね。つまり、以下のように使えるとしたいということです。

```java
public void printBaggageNames(Container container) {

    for (Baggage baggage : container) {
        System.out.println(baggage.getName());
    }
}
```

もしこれができたら、Getterは要らなくなって、よりシンプルなコードがかけますね。また、Listそのものを取得させるわけではないので、Immutableにする必要もなくなります。

これはIterableを使うことで具現化できます。

```java
public class Container implements Iterable<Baggage> {

    private final List<Baggage> baggages = new ArrayList<>();

    @Override
    public Iterator<Baggage> iterator() {
        return baggages.iterator();
    }
}
```

こうすることで簡単に、親クラスから子要素を拡張For文の中で使えるようになります。簡単ですね！

## 最後に

そこまで高級スキル的なものはなかったのですが、覚えておくとどこかで必ず役に立ちそうなスキルをいくつか集めてみました。これらは実際の仕事でも使っているものであって、とりあえず「動けばいい」レベルを超えて行きたい時に有効活用できるようなものではないかと思いました。こういう細かいところでのスキルの差が、プログラマーとしての実力に繋がるものではないでしょうか。そう思って、今後からも何かわかったらまたポストとして作成したいと思います。

では、また！