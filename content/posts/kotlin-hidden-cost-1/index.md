---
title: "Kotlinの隠されたコストーその１"
date: 2021-11-14
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---

Kotlinは便利ですが、何が便利かというと代表的に挙げられるものがたくさんのシンタクスシュガーではないかと思います。同じJVM言語のJavaと比べ、多くの場合でコード量が劇的に減るのが嬉しいという評価も多いものですね。しかし、この便利さの裏には隠されたコスト（性能面での）があるという話があります。今回はそれについて説明している良い記事を見つけたので、共有したいと思います。ただ、翻訳よりは要約に近いものなので、そこはご了承ください。

ちなみにここで紹介している記事（[Exploring Kotlin’s hidden costs - Part 1](https://bladecoder.medium.com/exploring-kotlins-hidden-costs-part-1-fbb9935d9b62)）は、2017年に作成された（Kotlinがまだ1.1だったころ）ので、1.5にまでバージョンアップを成している今からすると、コンパイラの改善などで少し状況が違うケースもあるかと思いますが、述べている内容のレベルが高いので一度は目を通してみても良いかなと思います。また、記事で紹介しているKotlinのBytecodeに対しても、直接最近のKotlinが生成しているコードと比較してみるのも面白いかもですね。

また、今回紹介している記事は`Part 1`ですが、そのほかにも`Part 2`や`Part 3`の記事がありますので、今後も順次紹介させていただきたいと思います。では、まず`Lambda表現式とcompanion object`編を、どうぞ。

## 高階関数とLambda表現式

例えば以下のような関数を定義しておいたとしましょう。渡されたパラメータをDBのトランザクションの中で実行し、実行結果の行数を返すものです。

```kotlin
fun transaction(db: Database, body: (Database) -> Int): Int {
    db.beginTransaction()
    try {
        val result = body(db)
        db.setTransactionSuccessful()
        return result
    } finally {
        db.endTransaction()
    }
}
```

上記の関数は、Lambdaを渡して以下のように使えますね。

```kotlin
val deletedRows = transaction(db) {
    it.delete(“Customers”, null, null)
}
```

KotlinはJava 1.6のJVMから使えますが、Java 1.6のJVMではLambdaに対応していないのです。なので、Kotlinはその互換性を維持するためにLambda（匿名関数も）を`Function`というオブジェクトを生成することで対応しています。

### Functionオブジェクト

では、実際コンパイルされたLambda（body）がJavaのコードとしてはどうなっているかをみていきましょう。（ここでは、Intellij/Android Studioの`Show Kotlin Bytecode`の`Decompile`機能を使っています）

```java
class MyClass$myMethod$1 implements Function1 {
   // $FF: 生成されたメソッド
   // $FF: ブリッジメソッド
   public Object invoke(Object var1) {
      return Integer.valueOf(this.invoke((Database)var1));
   }

   public final int invoke(@NotNull Database it) {
      Intrinsics.checkParameterIsNotNull(it, "it");
      return it.delete("Customers", null, null);
   }
}
```

これを見るとわかりますが、Lambda（匿名関数）を使う場合、コンパイルされた結果としては基本的に3、4個のメソッドが追加で生成されるということになります。ここで追加された`Function`オブジェクトのインスタンスは、必要な時にだけ生成されます。正確には、以下のような動作をします。

- value captureがある場合、毎回パラメータが渡されるたび`Function`のインスタンスが生成され、GCの対象になる
- value captureがない場合、`Function`はSingletonとしてインスタンスが生成され再利用できる

先ほどのコードでは、value captureがないため、Lambdaの呼び出し元は以下のようなコードとしてコンパイルされます。

```java
this.transaction(db, (Function1)MyClass$myMethod$1.INSTANCE);
```

しかし、value captureのある高階関数を繰り返し呼び出す場合はGCによる性能の低下を考えれます。

### Boxingオーバーヘッド

Lambdaに対応しているJava 1.8以降のバージョンでは、`Function`インタフェースを複数提供していることでなるべくboxing/unboxingを避けようとしています。しかし、Kotlinでコンパイルされた場合はgenericを利用しています。

```kotlin
/** 引数を一つ受け取る関数 */
public interface Function1<in P1, out R> : Function<R> {
    /** 引数を受け取り関数を実行する */
    public operator fun invoke(p1: P1): R
}
```

これらをみてわかるのは、高階関数でパラメータとして渡された関数を呼び出す時に、その関数にprimitiveタイプの値が存在する場合（パラメータ、もしくは戻り値）boxing/unboxingが起こるということです。先ほどのコンパイルされたLambdaにおいて、戻り値が`Integer`としてboxingされたのを確認できましたね。

primitiveタイプを使用するLambdaをパラメータとしてとる高階関数は、参照回数が少なければあまり意識しなくてもよいコストになりますが、そうでない場合は性能に影響があると推定できます。

### Inline関数

幸い、Kotlinでは`inline`と言うキーワドを提供しています。これを使うと高階関数をインライン化できますね。インライン化されると呼び出し元のコードに`Function`の中身を直接含ませてコンパイルします。なので、インライン化された場合は以下のような面で性能の向上を考えられます。

- Functionオブジェクトのインスタンスが生成されない
- primitiveタイプを使う関数に対してboxing/unboxingが起こらない
- メソッドカウントが増えない（Androidの場合、アプリが参照できるメソッドの数字に制限がある）
- 関数の呼び出しが増えない（CPU依存が高く、呼び出される頻度の高いコードのパフォーマンスの改善を期待できる）

インライン化された場合のコードを確認してみましょう。`transaction`関数が消え、`db.delete`を直接呼び出しているのがわかります。また、戻り値の`result`もWrapperクラスからprimitiveタイプになっているのがわかります。

```java
db.beginTransaction();
try {
   int result$iv = db.delete("Customers", null, null);
   db.setTransactionSuccessful();
} finally {
   db.endTransaction();
}
```

ただ、`inline`キーワードを使うときは以下のことを考慮しなければならないです。

- インライン関数は自分自身を直接呼び出したり、他のインライン関数から呼び出せない
- クラスに定義されたpublicなインライン関数はそのクラスのpublic関数とフィールドのみアクセスできる
- コンパイルされたコードが大きくなる（繰り返し参照される場合はより大きくなる）

なるべく高階関数をインライン化し、必要であれば長いコードブロックをインラインではない関数に写した方がいいです。また、性能が大事なところでは呼び出された関数をインライン化することも考えられます。

## Companion object

Kotlinではクラスがstaticなフィールドやメソッドを定義できません。その代わりに`companion object`を使うことになっていますね。

### クラスのprivateフィールドをcompanion objectからアクセスする

以下のような例があるとしましょう。

```kotlin
class MyClass private constructor() {

    private var hello = 0

    companion object {
        fun newInstance() = MyClass()
    }
}
```

上記のコードがコンパイスされると、`companion object`はSingletonクラスになります。なので、クラスのprivateフィールドに外部クラスからアクセスできるようにする必要があり、コンパイラが`getter`、`setter`を追加で生成することになるということです。生成されたメソッドは`companion object`から参照されることになります。以下を見てください。

```text
ALOAD 1
INVOKESTATIC be/myapplication/MyClass.access$getHello$p (Lbe/myapplication/MyClass;)I
ISTORE 2
```

Javaだとこれを避けるためにアクセス制限を`package`単位にすることができましたが、Kotlinではそのようなキーワードがないですね。`public`や`internal`を使う場合も`getter`と`setter`は基本的に生成されます。また、これらのメソッドはinstanceメソッドであり、staticメソッドよりもコストが高いですね。なので、最適化のためフィールドのアクセス制限を変えるということは避けた方が良いです。

もし`companion object`からクラスのフィールドに頻繁なアクセスが発生するとしたら、この隠れているメソッドの呼び出しを避けるためにフィールドの値をキャッシュするという方法も考慮できます。

### Companion objectの定数にアクセスする

Kotlinでは、クラス内のstaticな定数は`companion object`の中に定義するのが一般的です。

```kotlin
class MyClass {

    companion object {
        private val TAG = "TAG"
    }

    fun helloWorld() {
        println(TAG)
    }
}
```

一見シンプルで良さげなコードですが、Kotlin 1.2.40以前の場合だとかなり裏のコードは汚くなっています。

#### Kotlin 1.2.40以前の場合

`companion object`に定義されたprivateな定数にアクセスする場合、上記のようなこと（`getter`を利用する）が起こります。

```text
GETSTATIC be/myapplication/MyClass.Companion : Lbe/myapplication/MyClass$Companion;
INVOKESTATIC be/myapplication/MyClass$Companion.access$getTAG$p (Lbe/myapplication/MyClass$Companion;)Ljava/lang/String;
ASTORE 1
```

問題はこれだけではありません。生成されたメソッドは実際の値を返すわけでなく、instanceメソッドとして生成された`getter`を呼び出すことになります。

```text
ALOAD 0
INVOKESPECIAL be/myapplication/MyClass$Companion.getTAG ()Ljava/lang/String;
ARETURN
```

定数が`public`になっている場合はダイレクトにアクセスできるようになりますが、依然として`getter`メソッドを通して値にアクセスことになります。

そして定数の値を格納するために、Kotlinコンパイラは`companion object`ではなく、それを持つクラスの方に`private static final`フィールドを生成します。さらに`companion object`からこのフィールドにアクセスするため、またのメソッドを生成することとなります。

```text
INVOKESTATIC be/myapplication/MyClass.access$getTAG$cp()Ljava/lang/String; 
ARETURN
```

こういう長い道のりで、やっと値を読み込むことになります。

```text
GETSTATIC be/myapplication/MyClass.TAG : Ljava/lang/String; 
ARETURN
```

まとめると、Kotlin 1.2.40以前のバージョンを使っている場合は以下のようになります。

- `companion object`から静的メソッドを呼び出す
  - `companion object`からinstanceメソッドを呼び出す
    - クラスのstaticメソッドを呼び出す
      - staticフィールドから値を読み込む

これをJavaのコードで表現すると以下の通りです。

```java
public final class MyClass {
    private static final String TAG = "TAG";
    public static final Companion companion = new Companion();

    // 生成されるメソッド
    public static final String access$getTAG$cp() {
        return TAG;
    }

    public static final class Companion {
        private final String getTAG() {
            return MyClass.access$getTAG$cp();
        }

        // 生成されるメソッド
        public static final String access$getTAG$p(Companion c) {
            return c.getTAG();
        }
    }

    public final void helloWorld() {
        System.out.println(Companion.access$getTAG$p(companion));
    }
}
```

よりコストの低いBytecodeを生成することも可能ですが、それは簡単ではないです。

まず`const`キーワードを使ってコンパイルタイム定数を定義することでメソッドの呼び出しをなくすことができます。しかし、KotlinではprimitiveかStringに対してのみ可能な方法です。

```kotlin
class MyClass {

    companion object {
        private const val TAG = "TAG"
    }

    fun helloWorld() {
        println(TAG)
    }
}
```

または`@JvmField`を使ってJavaのアプローチを取る方法を考えられます。こうすることで`getter`や`setter`が生成されず、フィールドに直接アクセスができるようになります。ただ、`@Jvm`系のアノテーションはJavaとの互換性のためのものであるのでこれが果たして良い方法かどうかを考えた方が良いでしょう。そして`public`なフィールドのみ可能な方法です。

Androidの開発の場合だと、`Parcelable`オブジェクトを自前で実装する場合のみ有効な方法に思われます。例えば以下のようにですね。

```kotlin
class MyClass() : Parcelable {

    companion object {
        @JvmField
        val CREATOR = creator { MyClass(it) }
    }

    private constructor(parcel: Parcel) : this()

    override fun writeToParcel(dest: Parcel, flags: Int) {}

    override fun describeContents() = 0
}
```

最後の方法として、[ProGuard](https://developer.android.com/studio/build/shrink-code)やR8のようなツールを使ってBytecodeの最適化を狙うという方法があるでしょう。

#### Kotlin 1.2.40以降の場合

Kotlinn 1.2.40からは、`companion object`に定義された値はメインクラスの方に格納されるということには変わりがありませんが、メソッドの生成と呼び出しなしで直接アクセスができるようになりました。これをJavaのコードとして表現すると以下の通りです。

```java
public final class MyClass {
    private static final String TAG = "TAG";
    public static final Companion companion = new Companion();

    public static final class Companion {
    }

    public final void helloWorld() {
        System.out.println(TAG);
    }
}
```

また、上記のように`companion object`にメソッドが一つもない場合は、ProGuardやR8によるツールと使うとクラス自体が消えることで最適化されます。

ただ、`companion object`に定義さえたメソッドの場合はコストが少しかかります。フィールドがメインクラスの方に格納されてあるため、`companion object`に定義されたprivateフィールドにアクセスするためには依然として生成されたメソッドを経由することになります。

## 最後に

今回は人の書いた記事を読んだだけですが、かなり勉強になる内容でした。特に私個人としては、intellijを使っていると何を基準に`inline`キーワードを使った方がいいという警告が出るのか悩ましい場面がありましたが、それが少し理解できました。`companion object`に関する話も、今は問題が解決されたものの、何も考えず「定数だから`companion object`だな」と思っていた自分を反省することになりましたね。そしてこの後の記事でも面白い内容が色々と出てくるので、またの機会でぜひ紹介したいと思います。

では、また！
