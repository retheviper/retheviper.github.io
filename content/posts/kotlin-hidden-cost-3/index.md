---
title: "Kotlinの隠されたコストーその３"
date: 2021-11-28
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---

Kotlinの隠されたコスト、その最後の記事となります。今までの記事もかなり興味深かったですが、今回はさらにKotlinならではの機能に触れているので、Kotlinそのものに対する理解も含めてみる必要があり、さらに深い内容となっているかと思います。

今回のアジェンダは、「委譲プロパティ」と「rangeを使ったループ」になります。この記事は[Exploring Kotlin’s hidden costs - Part 3](https://bladecoder.medium.com/exploring-kotlins-hidden-costs-part-3-3bf6e0dbf0a4)の内容を要約したものです。

## 委譲プロパティ

[委譲プロパティ](https://kotlinlang.org/docs/delegated-properties.html)とは、`getter`と`setter`が`委譲(delegate)`というオブジェクトによって実装された[プロパティ](https://kotlinlang.org/docs/properties.html)を指します。これによって再利用可能なカスタムプロパティを作ることができます。

```kotlin
class Example {
    var p: String by Delegate()
}
```

委譲オブジェクトはプロパティの設定と読み込みのため`getValue()`と`setValue()`を実装する必要があります。そしてこれらの関数はプロパティのメタデータ（プロパティ名）とオブジェクトのインスタンスを引数として必要とします。

クラスが委譲プロパティとして定義されると、コンパイラは下記のようなコードを生成します。

```java
public final class Example {
   @NotNull
   private final Delegate p$delegate = new Delegate();
   // $FF: 生成されたフィールド
   static final KProperty[] $$delegatedProperties = new KProperty[]{(KProperty)Reflection.mutableProperty1(new MutablePropertyReference1Impl(Reflection.getOrCreateKotlinClass(Example.class), "p", "getP()Ljava/lang/String;"))};

   @NotNull
   public final String getP() {
      return this.p$delegate.getValue(this, $$delegatedProperties[0]);
   }

   public final void setP(@NotNull String var1) {
      Intrinsics.checkParameterIsNotNull(var1, "<set-?>");
      this.p$delegate.setValue(this, $$delegatedProperties[0], var1);
   }
}
```

一部staticプロパティのメタデータがクラスに追加されます。そして毎回値の設定と読み込みが発生するたびにコンストラクタによる初期化が起こります。

### 委譲インスタンス

上記サンプルでは新しい委譲のインスタンスがプロパティの実装のため生成されています。委譲がstatefulの場合にこのようになります。たとえはローカルで計算されたプロパティを使うなどの場合です。

```kotlin
class StringDelegate {
    private var cache: String? = null

    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        var result = cache
        if (result == null) {
            result = someOperation()
            cache = result
        }
        return result
    }
}
```

またコンストラクタに追加のパラメータが渡されると、新しい委譲のインスタンスが必要となります。

```kotlin
class Example {
    private val nameView by BindViewDelegate<TextView>(R.id.name)
}
```

statelessであり、すでに渡されたオブジェクトのインスタンスとプロパティ名を保ちたいだけなら委譲クラスに`object`をつけてsingletonにする方法があります。たとえば下記のようなものです。

```kotlin
object FragmentDelegate {
    operator fun getValue(thisRef: Activity, property: KProperty<*>): Fragment? {
        return thisRef.fragmentManager.findFragmentByTag(property.name)
    }
}
```

また既存のオブジェクトを拡張して委譲することもできます。つまり、`getValue()`や`setValue()`を拡張関数として定義することもできるということです。Kotlinではすでに`Map`と`MutableMap`に拡張関数として委譲するパターンを使っています。（プロパティ名をキーで使っています）

もし一つのクラス内でローカルの委譲インスタンスに複数のプロパティを保持して再利用したいなら、そのクラスのコンストラクタでインスタンスを初期化しましょう。

Kotlin 1.1以降、[関数内のローカル変数を委譲プロパティにする](https://kotlinlang.org/docs/delegated-properties.html#local-delegated-properties)こともできます。この場合、委譲は後で初期化できます。

クラスに定義された委譲プロパティごとにオーバーヘッドとメタデータの追加が発生するのでなるべくプロパティを再利用できるようにした方が良いでしょう。また、定義したい項目が多い場合に、果たして委譲プロパティが良い選択肢であるかを考慮すべきです。

### ジェネリック委譲

委譲関数はジェネリックでも定義できます。なので委譲クラスをさまざまな型のプロパティとして定義することもできます。

```kotlin
private var maxDelay: Long by SharedPreferencesDelegate<Long>()
```

ただ、上記のようにprimitiveをジェネリック委譲を使う場合、`boxing`と`unboxing`が値の指定と読み込みで発生することに注意する必要があります。これはプロパティがnon-nullの場合でも起こることです。

なのでnon-nullなprimitive型の委譲プロパティを定義する場合はジェネリックで定義を避けたほうが良いです。

### スタンダード委譲（lazy()）

Kotlinでは[Delegates.notNull()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/not-null.html)、[Delegates.observable()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/observable.html)や[lazy()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/lazy.html)のような委譲のための標準機能が存在しています。

`lazy()`は読み込み専用の委譲プロパティのための関数です。初めて読み込みが発生する際、プロパティを初期化するた目にlambdaを指定できます。

```kotlin
private val dateFormat: DateFormat by lazy {
    SimpleDateFormat("dd-MM-yyyy", Locale.getDefault())
}
```

これはその値が実際に読み込まれるまで高いコストの初期化を遅延させるという、パフォーマンスと可読性の側面で優れた方法です。

ただ、`lazy()`はinline関数ではなく、引数として渡されたlambdaは別の`Function`クラスとしてコンパイルされ、戻り値の委譲オブジェクトもまたinline化されないことには注意する必要があります。

そして`lazy()`関数で見逃しやすいのは`mode`という引数で戻り値の委譲タイプを決められるということです。

```kotlin
public fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)
public fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
        when (mode) {
            LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
            LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
            LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
        }
```

`mode`を指定していない場合、デフォルトとしては`LazyThreadSafetyMode.SYNCHRONIZED`が使われますが、これは複数のスレッドで初期化ブロックが安全に実行されることを保証するためにコストの高い`double-checked lock`を行います。

シングルスレッドしかプロパティに対するアクセスがないというのがわかっているなら、無駄なロックは下げた方がいいでしょう。こういう場合は`LazyThreadSafetyMode.NONE`を使えます。

```kotlin
val dateFormat: DateFormat by lazy(LazyThreadSafetyMode.NONE) {
    SimpleDateFormat("dd-MM-yyyy", Locale.getDefault())
}
```

## Ranges

[Ranges](https://kotlinlang.org/docs/ranges.html)で限定された範囲の値のセットを定義できます。この値は`Comparable`なものならなんでも指定できますね。そして、この表現式を使うと[ClosedRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-closed-range/)というインタフェースの実装ができることになります。

### 包含テスト

rangeを使って範囲内に特定の値が含まれているかどうかを`in`や`!in`を使って検知することができます。

```kotlin
if (i in 1..10) {
    println(i)
}
```

rangeはnon-nullなprimitive型（`Int`, `Long`, `Byte`, `Short`, `Float`, `Double`, `Char`）に対する最適化が行われるので、コンパイルされた結果は以下のようになります。

```java
if(1 <= i && i <= 10) {
   System.out.println(i);
}
```

なので、オーバーヘッドや追加オブジェクトの割り当てなどは起こらないです。しかし、primitiveではない場合はどうでしょう。

```kotlin
if (name in "Alfred".."Alicia") {
    println(name)
}
```

Kotlin 1.1.50以前はコンパイル時に`ClosedRange`オブジェクトが常に生成されました。しかし、1.1.50からは以下のようになります。

```kotlin
if(name.compareTo("Alfred") >= 0) {
   if(name.compareTo("Alicia") <= 0) {
      System.out.println(name);
   }
}
```

rangeはまた、`when`の条件式でも使えます。`if-else`より可読性が良くなりますね。

```kotlin
val message = when (statusCode) {
    in 200..299 -> "OK"
    in 300..399 -> "Find it somewhere else"
    else -> "Oops"
}
```

ただ、rangeを使う場合、特定の値が含まれているかどうかをチェックするとき、指定された範囲とそれを使うコードの間に間があるとコストがかかることになります。たとえば以下のようなコードがあるとします。

```kotlin
private val myRange get() = 1..10

fun rangeTest(i: Int) {
    if (i in myRange) {
        println(i)
    }
}
```

この場合はコンパイルすると`IntRange`オブジェクトが追加されます。

```java
private final IntRange getMyRange() {
   return new IntRange(1, 10);
}

public final void rangeTest(int i) {
   if(this.getMyRange().contains(i)) {
      System.out.println(i);
   }
}
```

これはプロパティのgetterを`inline`として定義しても同じです。なのでなるべくrangeが使われるテストの方に直接書くことでオブジェクトが追加されない要因した方が良いです。また、primitiveではないオブジェクトを使う場合は定数として定義し、`ClosedRange`のインスタンスを再利用する方法があります。

### forループ

`Float`と`Double`を除いたprimitive型の範囲をループで使うのも良い選択です。

```kotlin
for (i in 1..10) {
    println(i)
}
```

コンパイルされた結果にはオーバーヘッドが発生しません。

```java
int i = 1;
for(byte var2 = 11; i < var2; ++i) {
   System.out.println(i);
}
```

逆順にループしたい場合は[downTo()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/down-to.html)を使えます。

```kotlin
for (i in 10 downTo 1) {
    println(i)
}
```

これにもまた、オーバーヘッドは発生しません。

```java
int i = 10;
byte var1 = 1;
while(true) {
   System.out.println(i);
   if(i == var1) {
      return;
   }
   --i;
}
```

[until](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/until.html)を使って特定の値未満にループするのも良いですね。

```kotlin
for (i in 0 until size) {
    println(i)
}
```

以前は少しコストがかかることになりましたが、Kotlin 1.1.4以降は以下のようなコードが生成されます。

```java
int i = 0;
for(int var2 = size; i < var2; ++i) {
   System.out.println(i);
}
```

ただ、そのほかは最適化があまり効いてないケースもあります。[reversed()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/reversed.html)を使う例があるとしましょう。

```kotlin
for (i in (1..10).reversed()) {
    println(i)
}
```

コンパイルされたコードがあまり綺麗とは言えません。

```kotlin
IntProgression var10000 = RangesKt.reversed((IntProgression)(new IntRange(1, 10)));
int i = var10000.getFirst();
int var3 = var10000.getLast();
int var4 = var10000.getStep();
if(var4 > 0) {
   if(i > var3) {
      return;
   }
} else if(i < var3) {
   return;
}

while(true) {
   System.out.println(i);
   if(i == var3) {
      return;
   }

   i += var4;
}
```

`IntRange`オブジェクトが範囲を再定義するため生成され、さらに`IntProgression`オブジェクトが逆順に要素を整列するために生成されます。

`progression`を作るのに二つ以上の関数が使われていると、二つ以上のオブジェクトを作るようなオーバーヘッドが発生することになります。

上記のルールは[step()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/step.html)を使う場合も同じで、`step 1`を指定しても状況は変わりません。

```kotlin
for (i in 1..10 step 2) {
    println(i)
}
```

さらに、生成されたコードで最後の値を読み込む時、`IntProgression`オブジェクトの最後の要素と`step()`で指定した範囲を考慮して追加の処理が行われます。上記のサンプルだと最後の要素は`9`です。

なので、`for`を利用したループをするときはなるべく`..`、`downTo()`、`until()`を利用してオーバーヘッドを避けた方が良いでしょう。

### forEachループ

`for`ループの代わりに、rangeに対してinline拡張関数の`forEach()`を使う場合も結果はあまり変わりません。

```kotlin
(1..10).forEach {
    println(it)
}
```

しかし、`forEach()`は`Iterable`に対してのみ最適化されてないです。これはつまり、iteratorを生成する必要があるということを意味します。なので、コンパイルされると以下のようになります。

```java
Iterable $receiver$iv = (Iterable)(new IntRange(1, 10));
Iterator var1 = $receiver$iv.iterator();

while(var1.hasNext()) {
   int element$iv = ((IntIterator)var1).nextInt();
   System.out.println(element$iv);
}
```

これは今までのサンプルよりもコストのかかるものです。`IntRange`オブジェクトを生成するだけでなく、`IntIterator`オブジェクトも生成しているからです。primitiveではない場合はさらにコストがかかるでしょう。

なので、rangeを使ったループが必要な場合は`forEach()`より`for`ループを使ってオーバーヘッドを減らした方が良いです。

### collectionインデックスループ

Kotlinのスタンダードライブラリは[indices](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/indices.html)という拡張プロパティで配列と`Collection`のインテックスを提供します。

```kotlin
val list = listOf("A", "B", "C")
for (i in list.indices) {
    println(list[i])
}
```

`indices`のコンパイルされた結果は良い最適化を見せてくれます。

```java
List list = CollectionsKt.listOf(new String[]{"A", "B", "C"});
int i = 0;
for(int var2 = ((Collection)list).size(); i < var2; ++i) {
   Object var3 = list.get(i);
   System.out.println(var3);
}
```

`IntRange`オブジェクトが作られてないです。では、自前で実装してみるとどうなるのでしょう。

```kotlin
inline val SparseArray<*>.indices: IntRange
    get() = 0 until size()

fun printValues(map: SparseArray<String>) {
    for (i in map.indices) {
        println(map.valueAt(i))
    }
}
```

拡張プロパティとして定義してコンパイルすると、あまり効率的ではないコードになっていることがわかります。`IntRange`オブジェクトが作られてます。

```kotlin
public static final void printValues(@NotNull SparseArray map) {
   Intrinsics.checkParameterIsNotNull(map, "map");
   IntRange var10000 = RangesKt.until(0, map.size());
   int i = var10000.getFirst();
   int var2 = var10000.getLast();
   if(i <= var2) {
      while(true) {
         Object $receiver$iv = map.valueAt(i);
         System.out.println($receiver$iv);
         if(i == var2) {
            break;
         }
         ++i;
      }
   }
}
```

この場合は代わりに`until()`と`for`ループを使った方が良いでしょう。

```kotlin
fun printValues(map: SparseArray<String>) {
    for (i in 0 until map.size()) {
        println(map.valueAt(i))
    }
}
```

## 最後に

いかがでしたか。個人的にはあまり委譲プロパティを使ったことがなく、そもそもの理解を兼ねてかなり勉強になりました。また、rangeに関しても、Javaでの習慣でテストクラスのフィールドとして定義していろいろな関数で使い回していましたが、まさかそれがよりコストのかかることだとは思ってなかったので少しショックでした。

また、改めてKotlinで提供している機能とAPIに対して正しく理解する必要があると思いました。そして[オッカムの剃刀](https://ja.wikipedia.org/wiki/%E3%82%AA%E3%83%83%E3%82%AB%E3%83%A0%E3%81%AE%E5%89%83%E5%88%80)でも話しているように、なるべくシンプルなロジックとコードを追求する必要があるとも思いましたね。intellijのメニューのうち、`Tools > Kotlin > Show Kotlin Bytecode` でいつでもJavaのコードにdecomplieされたコードを確認できるので、最新だとどのように変換されるのかを確認してみながらコードを最適化を行なった方が良いかもしれません。

今月はいつもの、自分の経験や仮説を紹介するようなポストでなく、ほぼ翻訳のみになってしまいましたが、私自身としてはかなり貴重な知識を得られたと思っています。またの機会で何か良いものがあったら、是非とも紹介させていただきたいですね。

では、また！
