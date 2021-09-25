---
title: "Java 17は何が変わったか"
date: 2021-09-25
categories:
  - Java
photos:
  - /assets/images/sideimage/java_logo.jpg
tags:
  - java
  - kotlin
---

今月は新しいLTSバージョンであるJava 17のリリースがありました。まだJava 1.8を使っている案件も多いかなと思いますが、Java 1.8は2022年まで、Java 11は2023年までのサポートとなるので、いずれにせよJava 17に移行する必要はあるかなと思います。特にJava 9からモジュールが導入されたため、8からの移行はかなり大変だったらしいですが、11から移行する場合はそれほどでもないと言われているので、今からでも17では何が変わっているか、目を通しておくのもそう悪くはないでしょう。

現時点では[Eclipse Temurin](https://adoptium.net)(旧AdoptOpenJDK)、[Zulu](https://www.azul.com/downloads/)などの有名JDKはほとんどが17のリリースを完了しているか、対応の最中にありますね。また、[Oracle JDK 17は無料に](https://www.itmedia.co.jp/news/articles/2109/15/news147.html)なったので、こちらを選ぶもの悪くない選択肢の一つかも知れません。

また、こういう無料化やJDKの多様化のみでなく、GoogleとOracleの訴訟の件もGoogleの勝利で終わったので、AndroidでもJava 17を使える可能性ができた以上、これからJava 17を使える場面は増えてくるかも知れません。実際、まだ遠い話ではあります、Springを使う場合、2022年の[Spring 6はJava 17がベースラインとなる](https://spring.io/blog/2021/09/02/a-java-17-and-jakarta-ee-9-baseline-for-spring-framework-6)らしいですね。なので、Java 11は採択されてなかった現場でも、サポート期間などを考慮して17に転換する可能性はあると思います。

というわけで、今回はそんなJava 17では何が変わったかを述べていきますが、大きく分けて新しい予約語の追加、新しい書き方など言語スペックとして変わったものと、新しく追加されたAPIという二つの観点でその変化を辿っていきたいと思います。案件によってはJava 1.8から17に移行するケースもあるかと思いますが、9〜11までの間にあった変更事項や新しいAPIなどはこのブログでも扱っていて、他でも参考にできるサイトが多いと思いますので、今回は8~11までの変化については割愛し、11〜17の間の変化だけを扱うことにさせてください。

## 言語スペック

### New macOS Rendering Pipeline (17)

macOSでは長い間、SwingなどJavaの2Dレンダリングに[OpenGL](https://www.opengl.org/)を使っていましたが、新しい[Metal framework](https://developer.apple.com/metal/)を導入しながら、10.14からOpenGLは`deprecated`となりました。

従ってJava側でも、Metalを利用する新しいグラフィック・レンダリング・パイプラインを実装するという[Project Lanai](https://openjdk.java.net/projects/lanai/)が進められていましたが、17から[New macOS Rendering Pipeline](https://openjdk.java.net/jeps/382)という名で導入されました。JavaであまりGUIを使うことないのでは？と思いがちかと思いますが、intellijのようなJavaベースのIDEでも画面描画で性能向上があるという噂です。ただ、intellijでは基本的に[Jetbrains Runtime](https://confluence.jetbrains.com/display/JBR/JetBrains+Runtime)を使っていて、現時点ではそれがJava 17に対応していないので少し待つ必要はあります。

### macOS/AArch64 Port (17)

17からはM1など、[Apple Siliconを搭載した新しいMacに対応](https://openjdk.java.net/jeps/391)しました。[Zulu](https://www.azul.com/downloads/)などの他のJDKでは独自に対応してるケースもありましたが、OpenJDK(OracleJDK)で対応したことで、これをベースとする[Eclipse Temurin](https://adoptium.net/)や[Microsoft Build of OpenJDK](https://www.microsoft.com/openjdk)のような他のJDKでも自然にARMベースMacでネイティブとして使えるということになると思います。

### Record (17)

14からPreviewとして導入された`Record`が、17ではstableになり正式に導入されました。指定したフィールドを`private final`にして、コンストラクタ、`getter`、`toString`、`hashcode`、`equals`などを自動生成してくれるものです。最初は`Lombok`の[@Data](https://projectlombok.org/features/Data)のようなものかと思いきや、実際は[@Value](https://projectlombok.org/features/Value)に近いものになっていますね。値はコンストラクタでしか渡せなくて、後から変更はできなくなります。こういうところは、フィールドを`val`として指定したKotlinの`data class`に近い感覚でもあります。なので、実際の使用例を見ると、以下のようになります。

```java
// Recordの定義
record MyRecord(String name, int number) {}

// インスタンスの作成
MyRecord myRecord = new MyRecord("my record", 1);

// フィールドの取得
String myRecordsName = myRecord.name();
int myRecordsNumber = myRecord.number();
```

Kotlinでは[Named Arguments](https://kotlinlang.org/docs/functions.html#named-arguments)に対応しているのですが、Javaではまだそのような機能がないので、`Record`だとフィールドが多くなるとどれがどれだかわからなくなりそうな気はします。これに対してKotlin側で`Record`を使う場合、何らかのラッパークラスを作って対応するなどの方法は考えられますね。もしくは普通に`setter`をもつDTOを定義するか、builderパターンを利用する方が良いでしょう。

また、`Record`では`getter`名もフィールド名そのままになるという特徴もありますが、自動生成されるコンストラクタをカスタマイズするときも少し書き方が違うという特徴があります。

```java
record MyRecord(String name, int number) {
    // コンストラクタにバリデーションをつける例
    public MyRecord {
        if (name.isBlank()) {
            throw new IllegalArgumentException();
        }
    }
}
```

他に、`Record`として定義しても実際は`Class`が作られることになるので、以下のようなこともできます。

- コンストラクタを追加する
- `getter`をオーバライドする
- インナークラスとして`Record`を定義する
- インターフェイスを実装する

また、`Reflection`でもクラスが`Record`であるかどうかを判定する[isRecord](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Class.html#isRecord())も追加されています。

### Text Blocks (15)

Javaでは長い間、HTMLやJSON、SQLなどをリテラルとして使うためにはエスケープや文字列の結合などを使う必要がありました。これはあまり可読性という面でよくなく、コードの修正も難しくなる問題がありましたね。例えば、HTMLを表現するとしたら以下のようなことをしていたかと思います。

```java
String html = "<html>\n" +
              "    <body>\n" +
              "        <h1>This is Java's new Text block!</h1>\n" +
              "    </body>\n" +
              "</html>\n";

String query = "SELECT \"EMP_ID\", \"LAST_NAME\" FROM \"EMPLOYEE_TB\"\n" +
               "WHERE \"CITY\" = 'INDIANAPOLIS'\n" +
               "ORDER BY \"EMP_ID\", \"LAST_NAME\";\n";
```

幸い、15から[Text Blocks](https://openjdk.java.net/jeps/378)が導入され、他の言語のように簡単かつ可読性の高い文字列を定義することができるようになりました。これを使うとエスケープを意識しなくて良いので、複数行でなくても色々な分野で有効活用できそうですね。`Text Blocks`を使って上記のコードを変えると、以下のようになります。

```java
String html = """
              <html>
                  <body>
                      <h1>This is Java's new Text block!</h1>
                  </body>
              </html>
              """;

String query = """
               SELECT "EMP_ID", "LAST_NAME" FROM "EMPLOYEE_TB"
               WHERE "CITY" = 'INDIANAPOLIS'
               ORDER BY "EMP_ID", "LAST_NAME";
               """;
```

Kotlinでは全く同じ書き方で同じことができるので、ここでは割愛します。

### Sealed Class (17)

JDK 15からPreviewで導入された[sealed classes](https://openjdk.java.net/jeps/409)が、Stableとなりました。`class`や`interface`を`sealed`にすれば、それを拡張・実装できるクラスやインターフェイスを限定できるようになります。こうすることで、ライブラリなどで勝手に拡張して欲しくないクラスやインターフェイスを守ることができますね。また、将来的には`sealed`として定義されてあるクラスの子クラスを`switch`の`case`に指定するときは全部のケースが指定されているかどうかをコンパイラがチェックするようにするとかの話もあるようです。以下は、`sealed`クラスが`permits`キーワードを使って継承できるクラスを指定する例です。

```java
public abstract sealed class Shape permits Circle, Rectangle, Square, WeirdShape { }
```

Kotlinでも[Sealed Classes](https://kotlinlang.org/docs/sealed-classes.html)は存在していますが、`interface`を`sealed`にするためには1.5以降を使う必要があって、拡張・実装できるクラスやインターフェイスを指定するわけではなく、コンパイルされたモジュール以外で`sealed`として定義されているクラスやインターフェイスを拡張・実装できない仕様となっています。なので書き方的には、以下のようになります。より簡単ですね。

```kotlin
sealed interface Error

sealed class IOError(): Error

class FileReadError(val f: File): IOError()
class DatabaseError(val source: DataSource): IOError()

object RuntimeError : Error
```

また、Javaの場合は`Record`と同じく、このクラスが`sealed`であるかどうかを判定する[isSealed](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Class.html#isSealed())が追加されています。

### Switch Expressions (14)

Java 12からPreviewで[Switch Expressions](https://openjdk.java.net/jeps/361)が導入され、14からはStableになっています。従来の`switch`を改善したもので、以下のようなことができるようになりました。

- `case`をまとめて指定できる
- `case`の処理をラムダのような書き方で記述できる
- `case`の処理を戻り値にして、`switch`を式として使える

例えば、`day`というenumの値を見て、int値を返すメソッドを実装するとしましょう。従来の方法では以下のようになるはずです。

```java
int numLetters;
switch (day) {
    case MONDAY:
    case FRIDAY:
    case SUNDAY:
        numLetters = 6;
        break;
    case TUESDAY:
        numLetters = 7;
        break;
    case THURSDAY:
    case SATURDAY:
        numLetters = 8;
        break;
    case WEDNESDAY:
        numLetters = 9;
        break;
    default:
        throw new IllegalStateException("Wat: " + day);
}
```

上記の処理は新しい`switch`では以下のように書くことができます。

```java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY                -> 7;
    case THURSDAY, SATURDAY     -> 8;
    case WEDNESDAY              -> 9;
};
```

Kotlinだと以下のようになるはずですね。

```kotlin
val numLetters = when (day) {
    Day.MONDAY, Day.FRIDAY, Day.SUNDAY -> 6
    Day.TUESDAY -> 7
    Day.THURSDAY, Day.SATURDAY -> 8
    Day.WEDNESDAY -> 9
}
```

また、`when`だとargumentなしでも使えて分岐を条件文によるものにすることもできるなどの特徴もあるので、使い勝手はJavaの`switch`よりいいかなと思います。ただ、Javaでもバージョンアップと共に後述する機能も追加されてあるので、今後Kotlinのように色々と改良が行われる可能性はあるかと思いますね。

### Pattern Matching for instanceof (16) / switch (17)

Java 14からは、[Pattern Matching for instanceof](https://openjdk.java.net/jeps/394)が導入され、16ではStableになりましt。今までは`instanceof`を使ってオブジェクトのインスタンスの種類を判定した後、そのインスタンスの種類にあった処理を行うには以下のようにキャストが必要でしたね。

```java
static String formatter(Object o) {
    String formatted = "unknown";
    if (o instanceof Integer) {
        formatted = String.format("int %d", (Integer) i);
    }
    // ...
}
```

一度どれのインスタンスかわかった上でさらにキャストをする必要はあるのはだるいし、ミスをしたら例外の原因にもなり得る問題がありますね。なので、`Pattern Matching`を利用して、キャストをなくすことができるようになりました。`instanceof`を使った条件文の中に、キャストする変数名を指定しておくと、`if`分の中でそのまま自動にキャストされた変数を使えるようになります。なので、以下のようなことができるようになります。

```java
static String formatter(Object o) {
    String formatted = "unknown";
    if (o instanceof Integer i) {
        formatted = String.format("int %d", i);
    } else if (o instanceof Long l) {
        formatted = String.format("long %d", l);
    } else if (o instanceof Double d) {
        formatted = String.format("double %f", d);
    } else if (o instanceof String s) {
        formatted = String.format("String %s", s);
    }
    return formatted;
}
```

さらに、17からはPreviewとして[Pattern Matching for switch](https://openjdk.java.net/jeps/406)が導入されています。これを使うと、`instanceof`なしで、`switch`文を使ったよりシンプルな処理を書けるようになります。これを先に紹介した`Switch Expressions`と組み合わせることで、上記の処理は以下に変えることが可能になります。かなりシンプルになったのがわかりますね。

```java
static String formatterPatternSwitch(Object o) {
    return switch (o) {
        case Integer i -> String.format("int %d", i);
        case Long l    -> String.format("long %d", l);
        case Double d  -> String.format("double %f", d);
        case String s  -> String.format("String %s", s);
        default        -> o.toString();
    };
}
```

### Packaging Tool (16)

実行できるバイナリを生成する[Packaging Tool](https://openjdk.java.net/jeps/392)が導入されています。これを使うと、Java runtimeとライブラリ、それぞれのOSにあった実行ファイルが一つのパッケージになる機能です。Java runtimeが含まれるということはOSのJavaのバージョンに関係なく実行できるものになるという意味なので、Javaのバージョンを固定したり、複数のアプリでそれぞれ違うバージョンのJavaを使って起動したい場合は役立つ機能かも知れません。

## API

Java 17からは、APIドキュメントから、新しく追加されたAPIの一覧だけを見られるタブができたということです。今回は11以降に追加されたもののみですが、今後新しいLTSバージョンがリリースすると、17以降のものをこちらから確認できそうですね。新しいAPIの一覧は[こちら](https://download.java.net/java/early_access/jdk17/docs/api/new-list.html)から確認できます。

ここで全てのAPIの詳細まで探るのは難しいと思いますので、個人的に興味深いと思ったのを一部紹介したいと思います。

### @Serial (14)

`java.io`パッケージに、[Serial](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/Serial.html)というアノテーションが追加されました。これは[Serializable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/Serializable.html)を実装したクラスで、そのシリアライズのメカニズムを`@Override`するような機能のようです。例えば以下のようなことができます。

```java
class SerializableClass implements Serializable {

    @Serial
    private static final ObjectStreamField[] serialPersistentFields;

    @Serial
    private static final long serialVersionUID;

    @Serial
    private void writeObject(ObjectOutputStream stream) throws IOException {}

    @Serial
    private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException {}

    @Serial
    private void readObjectNoData() throws ObjectStreamException {}
    
    @Serial
    Object writeReplace() throws ObjectStreamException {}

    @Serial
    Object readResolve() throws ObjectStreamException {}
}
```

このアノテーションをつけることで、コンパイルタイムでエラーをキャッチできるのも特徴的です。例えば、このアノテーションを以下のようなクラスのメンバに使う場合はコンパイルエラーとなります。

- Serializableを実装してないクラス
- Enumのように、Serializeの効果がないクラス
- [Externalizable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/Externalizable.html)を継承しているクラス

このようなアノテーションが追加されたことによって、JacksonやGsonなどのライブラリの実装にも何か影響があるかも知れません。

### String

同じ文字列だとしても、Javaでは`java.lang.String`、Kotlinでは`kotlin.text.String`を使うことになるので、Kotlinを使う場合はあまりJavaのAPIを使うことはないかと思います（また、JavaでのString関連のAPIは、Kotlinだと`deprecated`になるケースが多いです）。なので、ここでは新しいAPIと、Kotlinで同じような処理をするために使える方法を中心に紹介します。

#### formatted (15)

Javaでは`String.format()`をで文字列をフォーマットとして使うことができました。多くの場合、文字列は`+`を使うよりフォーマットを使った方が性能が良いと言われていて、よく使っていたものです。

```java
String name = "formatted string";

// 15以前
String formattedString = String.format("this is %s", name);

// 15以降
String newFormattedString = "this is %s".formatted(name);
```

Koltinだと[String.format](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/format.html)と[String Templates](https://kotlinlang.org/docs/basic-syntax.html#string-templates)が使えます。

```kotlin
val name = "formatted string"

// Format
val formattedString = "this is %s".format(name)

// String Template
val templateString = "this is $name"
```

#### indent (12)

[indent](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/String.html#indent(int))では、対象の文字列に引数で指定した分のwhite spaceを入れます。引数が`int`型なので、負数を渡すことでwhite spaceを減らすこともできます。

```java
String nonIndent = "A";
// インデントを追加
String indented10 = nonIndent.indent(10); // "          A"
// インデントを削除
String indented5 = indented10.indent(-5); // "     A"
```

Kotlinの場合は、インデントを追加するための[prependIndent](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/prepend-indent.html)や代替するするための[replaceIndent](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/replace-indent.html)などがあり、渡すパラメータも文字列となるのでJavaのものとは少し使い方が違います。

```kotlin
val nonIndent = "A"
// インデントを追加
val prepended = nonIndent.prependIndent("     ") // "     A"
// インデントを代替（なかった場合は追加）
val replaced = prepended.replaceIndent("|||||") // "|||||A"
```

#### stripIndent (15)

`Text Block`で複数行の文字列を扱う場合、ソースコード上の可読性の都合で任意のインデントを入れたら実際のデータとしては扱いづらい場合もあるはずです。ここでインデントを削除するためののものが[stringIndent](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/String.html#stripIndent())です。

Kotlinでは[trimIndent](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/trim-indent.html)が同じ役割をしています。

#### transform (12)

文字列に対して[Function](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/function/Function.html)を実行するという単純なAPIです。`replace`では不可能な、条件による処理などが必要なときに使えそうです。実装を見ると極めて単純です。

```java
public <R> R transform(Function<? super String, ? extends R> f) {
    return f.apply(this);
}
```

Kotlinでは文字列でも`map`・`filter`・`reduce`のような高階関数が使えるのでこれらを使うこともできますね。もしくは以下のような拡張関数を定義することで同じことができるかと思います。

```kotlin
fun <R> String.transform(f: (String) -> R): R = f(this)
```

#### translateEscapes (15)

エスケープになっている一部の文字をリテラルに変えてくれる機能です。こちらはコードを見た方が理解が早いかなと思います。

```java
String string = "this\\nis\\nmutli\\nline";
String escapeTranslated = string.translateEscapes() // "this\nis\nmutli\nline"
```

以前は`Matcher`と正規式を組み合わせるなど独自の処理を書くか、ライブラリに依存していたと思いますので、こういうのができると嬉しいですね。変換されるエスケープ文字は以下の通りです。

| Escape | Name | Translation |
|---|---|---|
| `\b` | backspace | U+0008 |
| `\t` | horizontal tab | U+0009 |
| `\n` | line feed | U+000A |
| `\f` | form feed | U+000C |
| `\r` | carriage return | U+000D |
| `\s` | space | U+0020 |
| `\"` | double quote | U+0022 |
| `\'` | single quote | U+0027 |
| `\\` | backslash | U+005C |
| `\0 - \377` | octal escape | code point equivalents |
| `\<line-terminator>` | continuation | discard |

Kotlinでは似たようなAPIがないので、必要なら独自の処理を書いた方が良さそうです。（ライブラリは知らず…）

### Map.Entry.copyOf (17)

`Map.Entry`のコピーを作成します。コピーしたエントリは元のMapとは何の関係もないデータとなります。以下のようなサンプルコードを[公式ドキュメント](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.Entry.html)から提示していますね。

```java
var entries = map.entrySet().stream().map(Map.Entry::copyOf).toList();
```

ちなみに`Map`そのもののコピーは、10から追加された[copyOf](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.html#copyOf(java.util.Map))でできます。

```java
var copiedMap = Map.copyOf(map);
```

Kotlinだと、`Entry`のコピーは以下のようにできます。型は`List<MutableMap.MutableEntry<K, V>>`となります。

```kotlin
// Map.Entryを使う場合
val entriesJava = map.entries.map { Map.Entry.copyOf(it) }

// KotlinのMap.Entryを使う場合
val entriesKotlin = map.entries.toSet()
```

また、Kotlinでの`Map`のコピー方法は以下のようにできます。

```kotlin
val copiedMap = map.toMap()
```

### Stream

#### mapMulti (16)

16からStreamに[mapMulti](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html#mapMulti(java.util.function.BiConsumer))というメソッドが追加されました。基本的には「Streamの要素に1:Nの変換を適用して結果をStreamを返す」という処理なので、[flatMap](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html#flatMap(java.util.function.Function))に似ていますが、以下のケースでは`flatMap`を使う場合より良いと言われています。

- 要素を減らす場合
- 要素をStreamに変換するのが難しい場合

まずはオブジェクトがネストされているCollectionに対して`flatMap`を使う場合を考えてみましょう。要素を減らすケースでは、`flatMap`でまず全ての要素を展開し、`filter`を使って条件に合う要素だけを取る必要があります。ここで要素を展開するには、全ての要素を`Stream`に変換しなければならないので、全ての要素のグループに対して`Stream`のインスタンスを作ることになります。また、オブジェクトがネストしている場合は、その個別の要素に対してどうやって`Stream`に変換するか、処理の中で定義する必要があります。

問題は`Stream`のインスタンスを毎回作るためオーバヘッドが発生することにもなるし、要素がさまざまな型のオブジェクトである場合は`Stream`に変換する処理を書くのも大変ということです。例えば以下のようなListがあるとしましょう。

```java
List<Object> numbers = List.of(List.of(1, 2L), 3, List.of(4, 5L, 6), List.of(7L), 8L);
```

このListから、`Integer`のみを抽出して別のListにしたい場合はどうしたら良いでしょうか。まず`flatMap`を使うとしたら、以下のような処理を書くことになるかと思います。

```java
List<Integer> integers = list.stream()
        .flatMap( // 要素をStreamに変換する
                it -> {
                    if (it instanceof Iterable<?> l) {
                        return StreamSupport.stream(l.spliterator(), false);
                    } else {
                        return Stream.of(it);
                    }
                })
        .filter(it -> it instanceof Integer) // Integerのみを取る
        .map(it -> (Integer) it) // ObjectからIntegerへキャスト
        .toList();
```

これを`mapMulti`を使って処理する場合は以下のようになります。よりシンプルになりましたね。

```java
class MultiMapper {
    static void expandIterable(Object e, Consumer<Integer> c) {
        if (e instanceof Iterable<?> i) {
            i.forEach(ie -> expandIterable(ie, c));
        } else if (e instanceof Integer i) {
            c.accept(i);
        }
    }
}

List<Integer> integers = list.stream().mapMulti(MultiMapper::expandIterable).toList();
```

他にも[mapMultiToInt](https://docs.oracle.com/en/java/javase/17/docs/api/new-list.html#:~:text=java.util.stream.Stream.mapMultiToInt(BiConsumer%3C%3F%20super%20T%2C%20%3F%20super%20IntConsumer%3E))、[mapMultiToLong](https://docs.oracle.com/en/java/javase/17/docs/api/new-list.html#:~:text=java.util.stream.Stream.mapMultiToLong(BiConsumer%3C%3F%20super%20T%2C%20%3F%20super%20LongConsumer%3E))、[mapMultiToDouble](https://docs.oracle.com/en/java/javase/17/docs/api/new-list.html#:~:text=java.util.stream.Stream.mapMultiToDouble(BiConsumer%3C%3F%20super%20T%2C%20%3F%20super%20DoubleConsumer%3E))などのメソッドも追加されていますので、数字を扱う場合はこちらを使った方が便利でしょう。例えば、上記の`mapMulti`を`mapMultiToInt`で書く場合は以下のようになります。

```java
class MultiMapper {
    static void expandIterable(Object e, IntConsumer c) {
        if (e instanceof Iterable<?> i) {
            i.forEach(ie -> expandIterable(ie, c));
        } else if (e instanceof Integer i) {
            c.accept(i);
        }
    }
}

List<Integer> integers = list.stream().mapMultiToInt(MultiMapper::expandIterable).boxed().toList();
```

`mapMultiToInt`の戻り値は[IntStream](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/IntStream.html)なので、`Stream<Integer>`に変換するために[boxed](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/IntStream.html#boxed())を呼び出し、`Consumer`が`IntConsumer`に変わり、`mapMulti`の型指定が変わるなど少しの違いがあります。

Kotlinではそもそも`flatMap`を`Stream`として扱わないので、そもそもの処理を違う観点から考える必要があります。幸い、KotlinのCollectionには色々なAPIがあるので、そこまで難しくはないです。例えば、オブジェクトのインスタンスを基準に要素を集約したい場合は以下のようなコードを書くことができます。

```kotlin
val list = listOf(listOf("A", 'B'), "C", setOf("D", 'E', "F"), listOf('G'), 'H')

val result: List<String> = list.flatMap {
    if (it is Iterable<*>) {
        it.filterIsInstance<String>()
    } else {
        listOf(it).filterIsInstance<String>()
    }
} // [A, C, D, F]
```

ただ、Javaでは`List.of(1, 2L)`でListを作成した場合、1はint、2LはLongとして扱われますが、Kotlinでは`listOf(1, 2L)`が`List<Long>`となってしまうので、そもそもの型に注意する必要があります。

```kotlin
val list = listOf(listOf(1, 2L), 3, setOf(4, 5L, 6), listOf(7L), 8L)

val result = list.flatMap {
    if (it is Iterable<*>) {
        it.filterIsInstance<Int>()
    } else {
        listOf(it).filterIsInstance<Int>()
    }
} // [3]
```

#### toList(16)

Streamの終端処理として使用頻度の高い「Listに集計する」をシンタックス・シュガーとして作ったような感覚のメソッドです。ここはKotlinの機能をJavaが受け入れたような気もしますね。処理の結果として生成されるListは`Unmodifiable`です。

```java
List<String> list = List.of("a", "B", "c", "D");

// 旧
List<String> upper = list.stream().map(String::toUpperCase).collect(Collectors.toUnmodifiableList());

// 新
List<String> lower = list.stream().map(String::toLowerCase).toList();
```

Kotlinでは基本的にCollectionで高階関数を呼び出した結果が`Unmodifiable`なListになるのですが、`stream`に変換して使うこともできるので、場合によっては便利なのかも知れませんね。

### Collectors.teeing (12)

Collectorsに、二つの`Collector`を結合する[teeing](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Collectors.html#teeing(java.util.stream.Collector,java.util.stream.Collector,java.util.function.BiFunction))というメソッドが追加されました。ちなみに`Tee`は二つの水道管を接続して一つにしてくれる「T字継手」の意味を持つらしいです。引数に二つの`Collector`と、それを結合する処理の`BiFunction`を指定する形となっています。

例えば以下のような`Stream`があるとしましょう。

```java
record Member(String name, boolean enabled) {}

/**
* [
*    Member[name=Member1, enabled=false],
*    Member[name=Member2, enabled=true],
*    Member[name=Member3, enabled=false],
*    Member[name=Member4, enabled=true],
* ]
*/
Stream<Member> members = IntStream.rangeClosed(1, 4).mapToObj(it -> new Member("Member" + it, it % 2 == 0));
```

これを`teeing`を使って、`Member`の`enabled`を基準に二つのListに分けるとしたら以下のようになります。

```java
/**
* [
*    [
*       Member[name=Member2, enabled=true],
*       Member[name=Member4, enabled=true]
*    ],
*
*    [
*       Member[name=Member1, enabled=false],
*       Member[name=Member3, enabled=false]
*    ]
* ]
*/
List<List<Member>> result = members.collect(
        Collectors.teeing(
                Collectors.filtering(
                        Member::enabled,
                        Collectors.toList()
                ),
                Collectors.filtering(
                        Predicate.not(Member::enabled),
                        Collectors.toList()
                ),
                (list1, list2) -> List.of(list1, list2)
        )
);
```

Kotlinではそもそも`collect`する必要がないので、`Collection`の高階関数を使った処理をした方が良いでしょう。（Javaでもそうした方がわかりやすいような…）

## 最後に

いかがだったでしょうか。さすがに全ての変更事項を整理するのは難しかったので、目立っている変化だけをいくつか取り上げてみましたが、それでもかなりの量ですね。ただ確かなのは、Java 17が11よりもさらにモダンな言語になったバージョンであるので、Javaを使っている案件なら十分導入する価値がありそうです。また、Java 15からは11に比べてG1GCの改良もあったので、[性能向上もあった](https://www.optaplanner.org/blog/2021/01/26/HowMuchFasterIsJava15.html)ようですので、性能という面でも良いですね。

Kotlinを使っている場合でも、APIだけを見るとあまりメリットはないかも知れませんが、JVMを使っている限り性能向上などの恩恵を受けることはできると思われるので、導入を考慮しても良いかなと思います。また次のLTSでは色々と面白いAPIが続々と登場するかも知れませんしね。

では、また！
