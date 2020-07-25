---
title: "二つのListを結合する"
date: 2020-07-26
categories: 
  - java
photos:
  - /assets/images/sideimage/java_logo.jpg
tags:
  - stream
  - java
---

よく訪問しているサイトに、とある質問がありました。質問の内容とは、`List二つを、重複する要素なしで一つにまとめる方法`ということでした。SQLなら簡単に解決できそうな問題でもありますが、クエリーを修正できない状態だったり、複数のAPIを呼び出してその戻り値を扱う場合は直接コードを書いて併合するしかないですね。今まであまり経験したことのない状況でしたが、個人的にも興味が沸いたのでいろいろ試しながらコードを書いてみました。

## 問題のコード

質問の作成者がやりたいことは、`List<Map<String, Object>>`が二つあり、Listの中のMapの重複しない要素だけを一つのListとしてまとめたいということです。ここで重複の条件は、Mapのキーでした。

今はコードは存在するものの、重複チェックのためのロジックが複雑になりすぎて、負荷も高く性能面でも問題があるらしいです。まず彼の後悔しているコードは、以下のようなものでした。

```java
List<Object> list1 = ...
List<Object> list2 = ...

for (Object object : list1) {
    Map<String, String> map = (HashMap<String, String>) object;
    String a1 = map.get("a");
    for (Object object2 : list2) {
        Map<String, String> map2 = (HashMap<String, String>) object2;
        String a2 = map2.get("a");
        if (a1.equals(a2)) {
            Set<String> keys = map2.keySet();
            for (String key : keys) {
                String v = map2.get(key);    
                map.put(key, v);
            }
        }
    }
}
```

コードをみると、3重ループになっていて、KeyとValueが一致する項目がないかを一つ一つチェックしています。そして実際のコードなのかよくわかりませんが、このコードだとキーがリテラルになっているので全体のキーを循環できるようなループがまた追加されるべきではないかと思います。そうするとまたループが追加されたりして、より複雑なコードになりそうですね。これをListの個数分回すとしたら、それは負荷が多いだろうなと思います。なのでこれをなるべく短く、より単純なコードにしてみたいと思います。

Best Practiceではないのかも知れませんが、ひとまず自分の考えてコードを紹介します。

## Streamを使ってみる

Listの要素を一つづつ処理したい場合は、まずStreamを使う方法がないかをまず考えてみました。ネットで調べてみるとやはりStreamを使って複数のListをマージする方法がいくつかあります。それらを使って検証してみました。

### 結合と重複の除外

`Stream.concat()`を使うと、二つのStreamをつなげることができます。また、Streamでは`distinct()`で重複を除外することができます。これらの組み合わせを使うと、二つのListを重複する要素なしで結合することができます。まず簡単な例題を使うと以下のようなコードになります。

```java
// 結合したいList
List<String> list1 = List.of("a", "b", "d", "e");
List<String> list2 = List.of("b", "c", "d", "f");

// 結合と重複の除外("a", "b", "c", "d", "e", "f")
List<String> concat = Stream.concat(list1.stream(), list2.stream()).distinct().collect(Collectors.toList());
```

`Stream.concat()`の引数は二つしか指定できないので、3つ以上のListを繋ぐ場合はループを使うのを考慮すると良いです。また、`distinct()`の場合、`equals()`がちゃんと定義されてあるという前提ならどんなオブジェクトでも重複検査ができます。なのでLombokの`@Data`のようなアノテーションのついたクラスでも重複を除外して一つのListに納めることができます。

### 限界

質問の作成者は、Listの中のMapの要素に対して重複チェックを行いたいと行っていますが、この方法ではそのようにはなりません。なぜなら、Mapそのものを`equals()`で比較してしまい、中の要素一つ一つに対してはチェックしない構造となっているからです。なのでこのようなコードだと、二つのListを繋いだような物ができてしまうだけです。

## Forループを使ってみる

今回は質問者のコードを直して、より効率的に変えてみます。あえてForループを使っているのは、条件に一致した場合に一回だけ`put()`を実行するためです。`Stream`や`forEach()`は全ての要素に対して処理を行うためのものなので、除きました。

Mapには`put()`以外でも`putAll()`があるので、要素を巡回しながら一つづつ一つでもループは消すことができます。そして`putAll()`を実行したら、次の要素までチェックする必要はなくなるので`continue`を実行して次のループは飛ばすようにして、無駄な処理を無くします。そうするとまず以下のようにコードを帰ることができますね。

```java
for (Object object : list1) {
    Map<String, String> map = (HashMap<String, String>) object;
    String a1 = map.get("a");
    for (Object object2 : list2) {
        Map<String, String> map2 = (HashMap<String, String>) object2;
        String a2 = map2.get("a");
        if (a1.equals(a2)) {
            // ループを一回無くす
            map.putAll(map2);
            continue;
        }
    }
}
```

ここで、MapのKeyを指定しているところも直します。リテラルで指定しなく、Entryを巡回しながら比較することにするのでループは一つ増えます。そして、Entryは`Set`で取得できるので、Collectionのメソッドである`contains()`を使って比較することができます。なので比較したいMapのうち、どちらかのEntryを巡回しながら要素が違う方のMapに入っているのかを確認するだけで良いですね。これを反映して直したコードが以下です。

```java
for (Object object : list1) {
    Map<String, String> map = (HashMap<String, String>) object;
    for (Object object2 : list2) {
        Map<String, String> map2 = (HashMap<String, String>) object2;
        // Entryで比較する
        for (Map.Entry<String, DataClass> entry : map2.entrySet()) {
            if (map1.entrySet().contains(entry)) {
                map1.putAll(map02);
                continue;
            }
        }
    }
}
```

あとは最初からあらかじめListの型変換をして、ループ毎に型変換をしないようにすることですね。あまりこれで性能の改善は期待できないのかも知れませんが…

```java
// あらかじめ型変換をしておく
List<Map<String, String>> convertedList1 = (List<Map<String, String>>) list1;
List<Map<String, String>> convertedList2 = (List<Map<String, String>>) list2;

for (Map<String, String> map : convertedList1) {
    for (Map<String, String> map2 : convertedList2) {
        for (Map.Entry<String, DataClass> entry : map2.entrySet()) {
            if (map1.entrySet().contains(entry)) {
                map1.putAll(map02);
                continue;
            }
        }
    }
}
```

あとは場合によってMapのKeyをソートするなどの処理が必要になるかもですが、一旦これで要件は満たしたような気がします。

## 条件が違う場合

質問者のコードからは、推論するしかないことですが、もしListのインデックスを基準に比較するという条件があるとしたら、コードはより減らすことができます。list1とlist2の同じインデックスに、同じ要素を持つMapがあるかを確認するということです。もしこの条件があるとしたら、ループは2重に納めることができます。以下はその場合のコードです。

```java
for (Map<String, String> map : list1) {
    for (Map.Entry<String, String> entry : map.entrySet()) {
        // List2と同じインデックスを比較する
        Map<String, String> map2 = list2.get(list1.indexOf(map));
        if (map2.entrySet().contains(entry)) {
            map.putAll(map2);
            continue;
        }
    }
}
```

ただ、この方法を使うためには二つのListが同じサイズを持っているという前提が必要となるのでそこには気をつけなくではならなくなります。

## 最後に

たまにこうやって、自分ではまだ遭遇していない場合に対して考えられるチャンスとなるので、コミュニティに注目するのは良い経験となると思います。調べている間に普段は使ってみたことのないメソッドやAPIを調べてみたり、自分が書いていたコードを振り返ってみる良い機会にもなりますね。

他にもKeyは構わなく、重複するValueがある場合のみチェックするとか、要素のフィールドが重複している場合をチェックするかなどさまざまなバリエーションを考えられると思います。どれも面白い主題なのですが、今回の主題とは少し乖離があるため、機会があればいつかそのようなケースに対してのコードもブログに載せたいと思います。では、また！