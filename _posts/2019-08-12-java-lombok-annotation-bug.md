---
title: "Lombokのバグにあった話"
date: 2019-08-12
categories: 
  - java
photos:
- /assets/images/sideimage/java_logo.jpg
tags:
  - lombok
  - java
---

[前回のポスト](../java-design-pattern-builder)で、Builderパターンとともに`Lombok`を紹介しました。Beanだけでなく、Immutableなクラス[^1]やBuilderを作れるなど便利な機能が集まっていて、さらにアノテーションで様々なオプションが指定できたり(フィールドのアクセスレベルを指定できるなど)、必要なメソッドは追記しても良いなど使い道が多いですね。ただ、今回はそのLombokを使いながらバグらしき現象を発見したのでポストします。

## バグの発生したところ

どこでバグが発生したかを述べる前に、まずどんなLombokのアノテーションを使っていて、それらがそれぞれどんな機能をしているかを紹介したいと思います。なぜなら今回の場合は二つ以上のLombokアノテーションを組み合わせて使っていて、ほかにも自分と同じような組み合わせでLombokを使って同じバグを経験される方がいるかも知れないからです。

Lombokの`＠Builder`には、`(toBuilder=true)`のオプションをつけられます。これを使うと、基本的に新しいインスタンスの生成はstaticメソッドを呼ぶ事でできるようになって、すでに存在しているインスタンスから一部の値を変えて違うインスタンスに再生成する事ができます。コードで表現すると以下のようになります。

```java
// @Builderだけを使って新しいインスタンスを生成
House house2 = House.builder().type("wooden").build();

// toBuilderオプションで既存のインスタンスから値を変えて再生成
House house3 = house2.tobuilder().type("block").build();
```

そしてBuilderを使いながらも、元のクラスのフィールドの中でBuilderにデフォルト値として渡したいものもあります。つまり、Nullにしたくない場合ですね。自分の場合は、Listでした。インスタンスが生成される場合にはとにかくListを初期化して、Builderを使ってListそのものを代入したり、`addList()`のようなメソッドを作って個別の要素を追加できるようにしたかったです。

これを実現するためには、元のクラスでListを初期化して、その状態でBuilderに渡す必要はありました。Lombokのアノテーションではフィールドに`@Builder.Default`をつけることでできるようになります。そしてBuilderのアノテーションでは生成されない、`addList()`的なメソッドだけを実装することですね。コードで表現すると以下のようになります。

```java
@Builder(toBuilder=true)
public class Wallet {

    // Builderでインスタンスが生成されるときはNullにしたくない
    @Builder.Default
    List<String> cards = new ArrayList<>(); 

    // カスタムメソッドをBuilderに追加するための宣言
    public class WalletBuilder {

        // 元のBuilderではListそのものを代入する方法しかないので、個別要素を追加できるメソッドを書く
        public WalletBuilder addCard(String card) {
            this.cards.add(card);
            return this;
        }
    }
}
```

こうした実装で期待していた動きは以下のようなものでした。

```java
// インスタンスを生成しながらListにAddする
Wallet myWallet = Wallet.builder().addCard("Apple Card").build();

// 既存のインスタンスにAddする
Wallet newWallet = myWallet.toBuilder().addCard("American Express Card").build();
```

しかし、実際テストをしてみるとこの二つのアノテーションによりバグが発生したのです。

## それでどんなバグが？

既存のインスタンス(Listの操作を一切していない)では`add`が思い通りになったのですが、インスタンスを生成すると同時に`add`した場合はNPE[^2]が発生していました。場所を調べてみると`this.cards.add(card);`のところで例外が発生していたので、生成されてないオブジェクトに要素を追加しようとしていたとのことですね。つまり、Listがちゃんと初期化されてないという事です。

少し調べてみると、[Githubでのイシュー](https://github.com/rzwitserloot/lombok/issues/1347)がありました。2017年に書かれたもので今となってはずいぶん古い感じもしましたが、読んでみると今の自分が経験している現象と似ていましたね。しかも、Lombokの`1.18.2`バージョンで解消されたという話もありましたが、今の自分が使っているバージョンは`1.18.8`でした。解消されているはずがちゃんと想定通りならなかったのか、バージョンアップにより再発したのかわかりませんが、ともかく同じ現象が起きていましたね。

## 解決策

それではどう解決したらいいか？他にも方法があるのかも知れませんが、自分の場合は`toBuilder=true`と`@Builder.Default`の両方を使わない事で解決できました。Builderにちゃんとフィールドが渡らない時点で後者は意味がなくなりました。そして`toBuilder`の場合も、二つのメソッドが追加されるだけなのでそれを手書きで確実に値を渡せるようにしました。上で提示した`Wallet`クラスをこのやり方で直すと、以下のようなコードになります。

```java
// toBuilderオプションを使わない
@Builder
public class Wallet {

    // @Builder.Defaultを使わない
    List<String> cards = new ArrayList<>();

    // toBuilderメソッドも手書きする
    public WalletBuilder toBuilder() {
        return new WalletBuilder().cards(this.cards);
    }
}
```

これで元の想定通り、インスタンスを生成する時もちゃんと初期化されたListが渡るようになりました。めでたしめでたし。

## 教訓？

ライブラリーを使ってコードの量を減らし、自動化することは生産性の向上という面では大事な事ではありますが、人間の書いたコードはどこでバグが発生するかわからないので(想定していない使い方をする場合もありますし)、たまには手間がかかっても確実なコードを書くのが安全な場合もありますね。特にこの場合、アノテーションを諦めずコードを直そうとしていたらいつまでたってもバグは回避できなかったのかも知れません。そういう意味で、よい勉強になったと思える事件ではなかったのだろうかと思います。

もしこのポストを読まれる方の中、私と同じような実装を考えている方がいたら、こんなこともあるんだなと参考できるようなことになっていると嬉しいです。それではまた会いましょう！


[^1]: 一度インスタンスを生成すると、途中で値を変える事が出来ないクラス。Stringの場合がそうです。値を代入すると、メモリーに載せてある値を捨てて新しく生成します。
[^2]: Null Pointer Exception、ヌルポとも呼ぶ例外です。参照しようとしているオブジェクトがメモリー上にありませんよーとのことで、Javaで最も遭遇しやすい例外ですね。