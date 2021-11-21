---
title: "Kotlinの隠されたコストーその２"
date: 2021-11-21
categories: 
  - kotlin
photos:
  - /assets/images/sideimage/kotlin_logo.jpg
tags:
  - kotlin
  - java
---

今回はまたKotlinの隠されたコストに対するポストです。今となってはあまり気にすることはないかも知れませんし（検証は必要そうですが、バージョンアップごとにコンパイラが生成するコードを追うのは大変そうですね…）、極限のチューニングをするよりもマシンスペックを上げた方がよい時代になったとはいうものの、この記事で紹介していることをコーディングの習慣として身につけておくと良いかなと思います。

前回は高階関数とLambda、そしてcompanion objectに関する記事を紹介しました。今回はローカル関数、Null安定性、Varargsに隠されたKotlinのコストについて述べます。この記事は[Exploring Kotlin’s hidden costs - Part 2](https://bladecoder.medium.com/exploring-kotlins-hidden-costs-part-2-324a4a50b70)の内容を要約したものです。

## ローカル関数

関数内に定義した関数を「ローカル関数」と言います。これらローカル関数は、アウター関数（ローカル関数が定義された関数）の範囲にアクセスできます。例えば以下だと、`sumSquare`で`someMath`のパラメータにアクセスしているのがわかります。

```kotlin
fun someMath(a: Int): Int {
    fun sumSquare(b: Int) = (a + b) * (a + b)

    return sumSquare(1) + sumSquare(2)
}
```

ローカル関数は基本的にLambdaと似ていますが、他に制限があります。ローカル関数そのものと、ローカル関数を含む関数もまた`inline`として定義できません。なので関数の呼び出しにかかるコストを避ける方法がありません。

コンパイルされたローカル関数は`Function`オブジェクトに変わります。なので前回の記事で述べた「インライン化してないLambda」と同じ問題を持っています。上記のコードをJavaのコードで表すと以下のような形になります。

```java
public static final int someMath(final int a) {
   Function1 sumSquare$ = new Function1(1) {
      // $FF: 生成されたメソッド
      // $FF: ブリッジメソッド
      public Object invoke(Object var1) {
         return Integer.valueOf(this.invoke(((Number)var1).intValue()));
      }

      public final int invoke(int b) {
         return (a + b) * (a + b);
      }
   };
   return sumSquare$.invoke(1) + sumSquare$.invoke(2);
}
```

ただ、Lambdaと比べ一つ性能が劣化されない点があります。関数のインスタンスが呼び出し元からわかるので、ジェネリックなインタフェースを使わず、匿名クラスになりメソッドが直接呼び出されます。これは外の関数からローカル関数を呼び出す際に、`casting`や`boxing`が発生しないということを意味します。実際のBytecodeを見ると以下の通りです。

```text
ALOAD 1
ICONST_1
INVOKEVIRTUAL be/myapplication/MyClassKt$someMath$1.invoke (I)I
ALOAD 1
ICONST_2
INVOKEVIRTUAL be/myapplication/MyClassKt$someMath$1.invoke (I)I
IADD
IRETURN
```

ここでメソッドが2回呼び出されていますが、メソッドの引数も戻り値も`int`型になっていて、`boxing`と`unboxing`がないのを確認できます。

ただ、依然としてメソッドが呼び出されるたびに`Function`オブジェクトのインスタンスを生成していますが、ローカル関数をvalue caputeなしのものに代替することでこの問題は回避できます。

```kotlin
fun someMath(a: Int): Int {
    fun sumSquare(a: Int, b: Int) = (a + b) * (a + b)

    return sumSquare(a, 1) + sumSquare(a, 2)
}
```

上記のようにすることで、`Function`オブジェクトのインスタンスは再利用できるようなものになります。こうすることで既存のprivate関数に比べ、ローカル関数のデメリットは追加のクラス（メソッドを含む）を生成するということだけになります。

ローカル関数はprivate関数の代替として、アウター関数の変数にアクセスできるというメリットがあります。ただこれによって`Function`オブジェクトを生成するというコストがかかりますので、non-capturingにする工夫が必要です。

## Null安全性

Kotlinの最も良い機能の一つは明視的にnullになり得る型とそうでない型を区別できるということです。これによってコンパイラがランタイムで予期せぬ`NullPointerException`を投げるのを防止できます。

### Non-nullパラメータのランタイムでのチェック

例えば以下のような関数があるとします。

```kotlin
fun sayHello(who: String) {
    println("Hello $who")
}
```

これはJavaのコードで以下のようになります。

```java
public static final void sayHello(@NotNull String who) {
   Intrinsics.checkParameterIsNotNull(who, "who");
   String var1 = "Hello " + who;
   System.out.println(var1);
}
```

`@NotNull`アノテーションが追加され、Java側にnullが渡されてはいけないということを知らせています。

しかし、アノテーションは呼び出し側にnull safetyを強制するものではありません。なのでstaticメソッドを呼び出してパラメータをもう一度確認しています。この関数は`IllegalArgumentException`を投げて呼び出し元の修正を簡単にします。

publicな関数には常にnon-nullなパラメータに対して`Intrinsics.checkParameterIsNotNull()`でのチェックがが追加されますが、privateな関数に対しては追加されません。なぜなら、Kotlinクラスはnull safeであることをコンパイラが保証するからです。

このNullチェックによるパフォーマンスへの影響は無視しても良いほどでテストにも有用ですが、ビルド時にもっと時間がかかる原因になります。これに対してはコンパイラのオプションに`-Xno-param-assertions`を追加するか、[ProGuard](https://www.guardsquare.com/proguard)のルールに以下の設定を追加することでランタイムNullチェックをなくすことができます。

```java
-assumenosideeffects class kotlin.jvm.internal.Intrinsics {
    static void checkParameterIsNotNull(java.lang.Object, java.lang.String);
}
```

ただ上記のルールを追加する場合、AndroidのProGuardのOptimization設定が有効になっているかのチェックがまず必要です。この設定はデフォルトでは無効になっています。

### Nullable primitive型

まず先に覚えておくべきことは、nullableで宣言したprimitive型は常にJavaの`int`や`float`などの代わりに`Integer`、`Float`といった`boxed reference`型が使われるので追加のコストが発生するということです。

[autoboxing](http://docs.oracle.com/javase/8/docs/technotes/guides/language/autoboxing.html)とnull-safetyを無視するのでJavaでは`Integer`でも`int`でもコードはあまり変わらないJavaに対して、Kotlinだとnullableに対して安全なコードを書くように強制しているので、non-nullの方を使った方が良いというのが明確にわかります。

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}

fun add(a: Int?, b: Int?): Int {
    return (a ?: 0) + (b ?: 0)
}
```

なので、なるべくコードの可読性と性能を考慮してnon-nullの方を選んだ方が良いです。

### 配列

Kotlinには、以下の3通りの配列があります。

- `IntArray`、`FloatArray`のようなもの：primitive型の配列。`int[]`、`float[]`のような型にコンパイルされる。
- `Array<T>`：non-nullオブジェクトの型が指定された配列。primitiveに対して`boxing`が起こりえる。
- `Array<T?>`：nullableオブジェクトの型が指定された配列。明確に`boxing`が起こる。

もしnon-nullなprimitive型の配列が必要な場合は、なるべく`Array<Int>`の代わりに`IntArray`を使いましょう。

## Varargs

KotlinではJavaとは書き方が少し違いますが、[可変長引数](https://kotlinlang.org/docs/functions.html#variable-number-of-arguments-varargs)を定義することができます。

```kotlin
fun printDouble(vararg values: Int) {
    values.forEach { println(it * 2) }
}
```

Javaと同じく、`vararg`はコンパイルされると指定した型の配列になります。そして上記の関数は以下のように、３つの方法で呼び出すことができます。

### 複数のパラメータを渡す

```kotlin
printDouble(1, 2, 3)
```

Kotlinのコンパイラはこれを新しい配列の生成と初期化に変えます。これはJavaと一緒です。

```kotlin
printDouble(new int[]{1, 2, 3});
```

これはつまり新しい配列を作るためのオーバヘッドがあるということです。ただJavaと変わらないやり方です。

### 配列を渡す

Javaでは配列をそのまま渡すことができますが、Kotlinだとそれができず、`spread operator`を使う必要があります。

```kotlin
val values = intArrayOf(1, 2, 3)
printDouble(*values)
```

Javaでは配列の参照が`as-is`として関数に渡され、新しい配列の割り当ては起こりません。しかし、Kotlinの`spread operator`は以下のようなことをします。

```java
int[] values = new int[]{1, 2, 3};
printDouble(Arrays.copyOf(values, values.length));
```

配列のコピーが関数に渡されるので、より安全なコードといえます。呼び出し側には影響なしで、配列を修正できますので。しかしメモリを追加的に消費してしまいます。

### 配列と他の引数を混ぜて渡す

`spread operator`の良い点は、配列と他の引数を混ぜて渡すこともできるということです。

```kotlin
val values = intArrayOf(1, 2, 3)
printDouble(0, *values, 42)
```

この場合はどうコンパイルされるか気になりませんか？結果はかなり面白いです。

```java
int[] values = new int[]{1, 2, 3};
IntSpreadBuilder var10000 = new IntSpreadBuilder(3);
var10000.add(0);
var10000.addSpread(values);
var10000.add(42);
printDouble(var10000.toArray());
```

配列を新しく生成するだけでなく、一時的なビルダオブジェクトを使って配列の最終的なサイズを計算しています。なので配列を渡す時よりもコストは追加されます。

なので、呼び出される回数の多くパフォーマンスが重要なコードに対してはなるべく可変長引数より実際の配列をパラメータとして使った方が良いです。

## 最後に

いかがでしたか。個人的にprivate関数をよく使うので、よりスコープを制限できるという面でローカル関数を積極的に使いたいと思っていましたが、ここでも隠されたコストがあるというというのは興味深かったです。primitive型についてはJavaがそうだったので、なんとなく`boxing`が起こるんじゃないかなと思っていたものの、nullableに対してのみそうだというのも面白かったですね。逆に、primitiveのままになるnon-null型に対してはどうやってチェックが走るのだろうという新しい疑問もありました。（例えば`int`だとデフォルト値の`0`が常に割り当てられるので）

あと、配列の場合はJavaでも`IntStream`、`DoubleStream`などがあったのでなんとなくすぐ理解ができましたが、まさか`varargs`で渡したパラメータに対して色々とコストが追加されるとは思わなかったです。そもそもあまり配列を使わないので、可変長引数を使う場面もなかったのですが…よく使わないものほど重要なことを忘れやすそうなので、これは覚えておかないとですね。色々と勉強になりました。

では、また！
