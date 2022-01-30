---
title: "Javaはこう進化して欲しい"
date: 2020-02-03
categories: 
  - java
image: "../../images/java.jpg"
tags:
  - java
---

Javaは長い間業界で生産性、性能、安定性で評判がよく、最近はバージョンアップも様々な機能が追加されています。仕事では主に11バージョンを使っていますが、次のLTSバージョンである17がリリーズされたらそちらを使うことになるのではないかと思います。なので今も新しいバージョンが発表されると一応更新履歴には目を通していますが、言語仕様そのものが変わる場合は新しいAPIの追加と比べ少ない気がしなくもないです。

Javaが人気を得た理由の一つの生産性という部分では、今じゃPythonやJavaScriptなどに比べて劣る部分もあり、「コードが読みやすい」というメリットも、いつの間にか「冗長すぎる」という評価と変わってしまいましたね。自分はJavaが好きで、まだ言語そのものの仕様まで見切ってはいないですが、それでも使いながらこれは不便だな、これは他の言語と同じくなって欲しいなと思う時もあります。今回のポストでは仕事でJavaを扱いながら感じた不便なところ、また他の言語と比べて改善したいところについて述べたいと思います。こうして他の言語と比べ、Javaがどんなものであるかを把握していくのもまた一つの勉強になるのでは、と思いますので。

## Optional表記の改善

以前のポスト](../../../01/05/java-optional)で紹介したOptionalですが、これはJavaだけでなく他の言語でも多く採用しているAPIの一つですね。むしろ、JavaのOptionalが他の言語から影響され導入されたものらしいです。今は私自身もよく使っていて、すごい便利だと思っていますが、それでも他の言語と比べたらやはり不便と思うところがあります。

他の言語と比べてコードが冗長であることがJavaの特徴と先に述べましたが、実際のコードではどうかをまず比較してみましょう。複雑にネストしているオブジェクトのとあるフィールドを読み、Nullだった場合はデフォルト値を返却する例を持って説明します。

### Java

Javaのコードでは、最初のオブジェクトをOptionalでラップし、ネストされているフィールドやメソッドに対してmap()をチェーニングしていくことで次から次へとラップの対象を変えていきます。そして最後に、ターゲットのオブジェクトがNullだった場合はorElse()などのメソッドでデフォルト値を設定しますね。

```java
// 元のオブジェクト
SomeClass object;

// 複雑にネストされている
Optional.ofNullable(object).map(obj::getProp1).map(prop1::getProp2).map(prop2::getProp3).orElse("default");
```

### C#

言語そのものがJavaと似ているC#ですが、より若いからか、Javaと比べ進んでいる部分がよく見当たるC#です。こちらでもKotlinとコードの書き方は同じです。違うのは、オブジェクトそのものがNullになる可能性を事前に宣言しないということだけですね。

```csharp
object?.prop1?.prop2?.prop3? ?? "default";
```

### JavaScript

JavaScriptのOptionalもまた、C#とあまり変わらないです。

```js
object?.prop1?.prop2?.prop3? ?? "default"
```

### Swift

Swiftでもそう変わりません。

```swift
object?.prop1?.prop2?.prop3 ?? "default"
```

### Kotlin

Kotlinもデフォルト値指定のためのElvis opertor特有の表現を覗くと、一緒ですね。

```kotlin
object?.prop1?.prop2?.prop3 ?: "default"
```

他の言語の例と比べて見ると、JavaのOptionalは確かに冗長な印象ですね。なので今後Optionalを言語の基本仕様として導入し、ラップをするのではなく?として表現できるようにしたらどうかという気もします。?を導入したところでコードの読みやすさを損ねるわけでもないですので。

## Multiple Return Statements

Javaの仕様ではメソッドの戻り値となれるオブジェクトは常に一つのみですが、Pythonのような言語では戻り値を複数指定することができます。もちろん、Javaの戻り値が一つという制約を乗り越えるためによくBeanやCollectionに複数のオブジェクトやデータを入れて返すことはできるのでこれはシンタックスシュガー的なものになるだけですが、それでも便利な方法があったら使いたくもなります。

### Java

メソッドの処理結果として複数のデータを受け取りたい場合、Javaだと先に述べたようにBeanやCollectionを使うことになりますね。以下は戻り値が複数の数字である例です。

```java
// 複数の戻り値を持つメソッド
public List<Integer> multipleReturn() {
    return Arrays.asList(1, 2);
}

// 戻り値の取得
List<Integer> data = multipleReturn();
```

### C#

C#でもJavaと似たようなやり方で複数のデータを取得することができますね。実際はref/outパラメータを使ったり、structやclassを使う方法もあるらしいですが、Javaと比べより便利ではないかと思うのはTupleを使う場合です。C#のバージョンによって書き方が違っていて、昔の書き方ではJavaでCollectionを使うのとあまり変わらないものの、新しい書き方ではかなり便利なものとなっています。以下はその二つの例のコードです。

```csharp
// 複数の戻り値を持つメソッド(7以前)
public Tuple<int, int> oldMultipleReturn()
{
    return Tuple.Create(1, 2);
}

// オブジェクトとして取得
var result = oldMultipleReturn();

// 複数の戻り値を持つメソッド(7以後)
public (int, int) newMultipleReturn()
{
    return (1, 2);
}

// 変数として取得
(int one, int two) = newMultipleReturn();
```

### Python

Pythonの例では、7以後のC#と似たような感覚でコードを書けます。オブジェクト(tuple)として戻り値を全部取得するか、個別の変数として取得するか両方一つのfunctionでできるのがより便利な気もしますね。

```python
# 複数の戻り値を持つfunction
def multiple_return():
    return 1, 2

# 個別の戻り値を取得
a, b = multiple_return()
print(a) # 1

# 戻り値をtupleとして全取得
d = multiple_return()
print(d) # (1, 2)
```

### JavaScript

ES6から導入された書き方ではPythonと似たようなコードで複数の戻り値を取得できます。

```js
// 複数の戻り値を持つfunction
funtion multipleReturn() {
    return {
        first: 1,
        second: 2
    }
}

// 個別の戻り値を取得
var (first, second) = multipleReturn()
```

### Swift

SwiftはやはりOptionalと同じく、JavaScriptとあまり変わりません。

```swift
// 複数の戻り値を持つfunction
func multipleReturn() -> (Int, Int) {
    return (1, 2)
}

// 個別の戻り値を取得
let (first, second) = multipleReturn()
```

### Kotlin

KotlinではPairかTripleなどがあり、使い方は簡単です。

```kotlin
// 複数の戻り値を持つfunction
fun multipleReturn(): Pair<Int, Int> {
    return 1 to 2
}

// 個別の戻り値を取得
val (first, second) = multipleReturn()
```

Javaでのコードの書き方の方がメソッドの役割をわかりやすいというメリットはありますが、戻り値のオブジェクトやデータをそのまま変数として使えるという面ではPythonのやり方がより便利ですね。このように複数の戻り値を持つメソッドを定義できるのは現代プログラミング言語ならどれもが持っている機能のようです。Javaにもいつかは導入されるのでしょうか？

## 引数の種類をor指定

たまに、一つのメソッドで引数の型を複数指定できたら便利ではないだろうかと思うことがあります。Javaではこれをオーバーロードで実現していますね。

### Java

```java
public void doSomething(String value) {
    // Stringの場合の処理
}

public void doSomething(int value) {
    // intの場合の処理
}
```

もちろん、引数の型をObjectとして宣言し、内部ではinstansofを使って判定することもできます。しかし、前者ならやりたいことに比べコードの量が増えすぎる問題がありますし、後者なら意図した型以外のObjectが渡された場合の挙動がおかしくなる可能性もあります。

### TypeScript

TypeScriptでは、これを簡単に引数のタイプを複数指定できるようにすることで解決しています。

```ts
function checkString(v: string | number) {
    if (typeof v === "string") {
        return true
    }
    return false
}
```

これまで引数の種類だけ違う場合はオーバーロードして、共通処理だけprivateメソッドで書いていましたが、これならよりコードを簡単に把握できそうですね。ぜひ導入して欲しい機能の一つです。

## Ternary Operator with Throw

結果が二択しかない場合は、なるべくifより三項演算子を使った方がコードも短くなり便利と思います。しかし、Javaの三項演算子では例外を投げることができません。条件式に当てはまらない場合はどうしても以下のようなコードを書くしかないです。

```java
// xが0だとnumberも0で、0ではなかった場合は例外とする
int number = x -> {
    if (x == 0) {
        return x;
    } else {
        throw new RuntimeException();
    }
}
```

無理やり三項演算子で例外を投げようとしたら、以下のような方法はありますね。

```java
// Genericな戻り値を持っていて、例外を投げるだけのメソッド
public <T> T throwSomeException() {
    throw new RuntimeException();
}

// elseでメソッドを呼ぶ
int number = x == 0 ? x : throwSomeException();
```

個人的にif文は二択しかない結果のために使うのはスペースの無駄遣いと思いますし、無理やりメソッドを作ってまで三項演算子を使う必要はないので、三項演算子でelseの場合には単純に例外を投げられるといいな、と思っていました。そして調べてみると、他の言語ではそれができるようです。

### C#

C#では、二つのやり方があります。まず7以前だと、elseの場合に例外を投げるFuncを実行させることで実現できます。そして7以後では普通に三項演算子でthrowできるようです。まさに私が望んでいた形ですね。

```csharp
// 7以前
int number = x == 0 ? x : new Func<int>(() => { throw new Exception(); })();

// 7以後
int number = x == 0 ? x : throw new Exception();
```

### Kotlin

Kotlinでは三項演算子がなく、if-elseを使うことになるということだけで、簡単な形になっています。

```kotlin
val i = if (x == 0) x else throw Exception("error")
```

### JavaScript

JavaScriptでは、7以前のC#と似た形で例外を投げることができます。

```js
var number = (x == 0) ? x : (function() { throw "error" }());
```

わざわざ関数を実行してまで三項演算子で例外を投げたくはないですが、こういうやり方があるということがわかっただけでもかなり興味深いですね。JavaにもC#の7以後のような書き方ができるといいな、と思います。

## アクセス修飾子の拡張

Javaでのアクセス修飾子は、public・private・protectedをよく使っています。しかし、自作ライブラリーを作る場合はpublicとprivateの中間的なものもあって欲しいな、と思う時もあります。例えばJarにまとめた時、Jar以外ではアクセスできないようなアクセス制限をかけられるようなものですね。

Java 9からモジュールが導入されましたが、自分が経験した問題もあるのでなるべくモジュールを使いたくはないなと思っているので…同じプロジェクトの中ならパッケージが違ってもアクセスできるような修飾子があったらいいな、と思います。また、protectedならサブパッケージでも参照できるなど。同じモジュール内でのみのアクセス修飾子はC#とSwift、KotlinでInternalとして提供しているので、Javaにも導入されるといいですね。

## 最後に

最近のJavaの更新履歴をみると、続々と便利な機能が導入され続けています。特に14では、recordでLombokの`@Data`と同じ機能を持つクラス宣言ができるようになるらしいです。次のLTS版は17なので、まだ十分色々と改善される余地はありますね。1.8でも便利な機能は多いですが、これからもどんどん他の言語の良い点を吸収して変転できるといいなと思います。

また、こうやって他の言語ではどうしているかを調べてみるのも良い勉強となりました。特にTypeScriptは最近注目している言語なので、機会があれば経験してみて、Javaとの比較もしてみたいものですね。では、また！
