---
title: "SwiftUIを触ってみた〜その1〜"
date: 2022-07-31
categories: 
  - swift
image: "../../images/swift.jpg"
tags:
  - swift
  - swiftui
  - gui
  - kotlin
  - compose
---

今までの自分のキャリアを振り返ってみると、仕事としての経験はバックエンドばかりで、画面側の実装にはあまり関わったことがありません。しかし、スタンドアロンのアプリを作るためには、ウェブ・モバイル・デスクトップを問わず画面が必要となるので、いつかは画面側の実装もできるようになる必要があるかなと常に思っているところです。

画面を作るといっても、どんな分野のエンジニアとしてキャリアパスを考えているか、どのような企業で働きたいか、慣れている言語は何であるかなど色々と考慮すべき要素は多いのですが、自分の場合はKotlinに慣れているのもあり、ウェブ・モバイル・デスクトップアプリに全部対応できるという点から[Jectpack Compose](https://www.jetbrains.com/lp/compose-mpp)を、また普段からMacとiPhone、iPadといったApple社の製品をよく使っている上、KotlinからのSwift入門が比較的簡単ということで[SwiftUI](https://developer.apple.com/xcode/swiftui/)を勉強したいと思っています。

さて、言語とフレームワークを決めてからは実践ですね。[公式のチュートリアル](https://developer.apple.com/tutorials/swiftui)が充実していたので、まずはこちらの方をすすめながら感じたSwiftやSwiftUIで印象的だった部分についてまとめてみたいと思います。もちろん、自分は仕事としてモバイルアプリの実装に関わったことがないのでコンテンツとしては粗末なものとなるかなと思いますが、もし自分のようにKotlinのバックグラウンドからSwiftに触れてみようと思っている方や、バックエンドのみのキャリアからGUIに初めて触れる方、もしくはKotlinとSwiftのどちらかに興味を持っている方には参考になる内容となればと思います。

## Swift

まずは言語そのものから。KotlinとSwiftはよく似ているという話を聞くことがありますが、正確に「どこが」というのはやはり触れてみる前はわからないものです。似ているという表現は共通点があるという意味なので、何に基準を置くかによって挙げられる共通点は色々と変わってくるものだからです。

例えば、言語デザインの観点でOOP志向的で、関数型的な要素があり、GCが存在する、ということでも共通点は発見できます。もしくは、言語の使用としてキーワードや書き方の印象が似ているという意味にもなれますね。細かくは、セミコロンを使わなくて良いという点も挙げられますね。

なので、まずは上記のチュートリアルを進行しながら、肌で感じた感覚から、Kotlinに比べたSwiftはどのようなものだったかを述べていきたいと思います。

### Kotlinに似ているもの

では、まずKotlinに似ているなと感じたところから述べていきます。似ているとしても、あくまで「肌の感触」なものなので、厳密には違う仕様になっているものも多いのですが、ここでの基準は「Kotlinでできたことをどれほど近い感覚で再現できるか」となっていますので（といっても個人的な感想ですが）、参考までに。

#### Computed properties

まず、プロパティの話からです。Kotlinではdata classを定義するとき、プロパティを以下のような二つの方法で定義することができます。

```kotlin
data class Student(
    val age: Int
) {
    val isAdult: Boolean
        get() = age >= 18
}
```

ここで`age`はインスタンスを作成するときに固定される単純な値となりますが、`isAdult`はgetterとして定義した処理(`age`が18以上かどうかという)の結果を返すように定義する形ですね。このような処理を伴うプロパティは、Swiftでも同じく[Computed Properties](https://docs.swift.org/swift-book/LanguageGuide/Properties.html#ID259)を通じて定義することができました。同じような処理を行う場合、以下のように定義できます。

```swift
struct Student {
    var age: Int

    var isAdult: Bool {
        get { return age >= 18 }
    }
}
```

まだ一つしたあげてないのですが、これだけでもなんとなく「KotlinとSwiftが似ている」の意味が少しは見えてきた気がしますね。処理を伴うプロパティを扱える、という仕様がそうですが、型の定義もそうで、キーワードは少し違うけど大体似たような感覚でコードが読めるというところがそうです。

ただ、やはり違う部分もありますね。data classに対して、SwiftはGoやRustのようにstructを使えるというところがそうかなと思います。もちろんSwiftにもClassはあるので、目的によってどれかを選ぶようになるらしいです。という面では、またなんとなくKotlinでdata classとclassを分けて使うという点と似ているような気もしますね。

#### Extension

次は、拡張です。Kotlinではオブジェクトについて、そのオブジェクトの外にメソッドやプロパティを定義することができますね。これらを拡張関数や拡張プロパティと呼び、以下のように定義することができます。

```kotlin
val Student.isUnderAge: Boolean
    get() = age < 18
```

以前このブログでも述べたことのある[Effective Kotlin](https://www.amazon.co.jp/dp/B08WXCRVD2/ref=dp-kindle-redirect?_encoding=UTF8&btkr=1)で提示されている活用方法ですが、ユースケースやドメインによって違う処理が必要となった場合は、class内に全てのメソッドやプロパティをを定義するよりはこのような拡張を使って、パッケージごとに定義することでアクセス制限を設ける方法があります。

同じようなことがSwiftでもできますが、書き方はやはり少し違いました。上記のようなプロパティをSwiftで同じ方法で実装する場合、以下のようになります。

```swift
extension Student {
  var isUnderAge: Bool {
    get { return age < 18 }
  }
}
```

上記のようにSwiftには[Extension](https://docs.swift.org/swift-book/LanguageGuide/Extensions.html)が別途キーワードとして存在していて、新しくclassやstructを定義するかのような感覚で関数やプロパティを付け加えることができます。個人的な感想としてはRustの[メソッド](https://doc.rust-jp.rs/book-ja/ch05-03-method-syntax.html)と似ている形で、一つの`extension`の中にまとめられるところがむしろKotlinより整頓された感覚なので良さげですね。Kotlinの場合、一つのオブジェクトに対しての拡張が複数あると少し汚くも見えるので…

#### String Interpolation

Javaの場合でもそうで、多くの言語では文字列と、違う変数として格納してある値を一つの文字列にまとめる場合は`format()`を使うか、文字列に変換して結合するケースが多いかなと思います。Kotlinでもそのような使い方はもちろんできますが、[String template](https://kotlinlang.org/docs/basic-types.html#string-templates)があるので、簡単に文字列の中で違う値を埋め込むことができます。例えば以下のようなものですね。

```kotlin
val world = "World"
println("Hello, $world")
```

Swiftでも[String Interpolation](https://docs.swift.org/swift-book/LanguageGuide/StringsAndCharacters.html#ID292)があるので、同じことができます。少し書き方が変わっているのですが、機能的にはほぼ一緒です。

```swift
let world = "World"
print("Hello, \(world)!")
```

#### Arguments

Kotlinでは、関数のパラメータにデフォルト値を設定することで、簡単にオーバーロードを実現でき、そのパラメータが渡されてない場合の処理にも対応できます。

```kotlin
// timesに指定した数値分、stringを標準出力する
fun printHello(string: String, times: Int = 1) {
    repeat(times) {
        println("Hello, $string")
    }
}

printHello("world") // timesに値を指定しなくても関数を呼び出せる
```

また、どのパラメータに値を指定したいかを明確にするときや、関数に定義されたパラメータの順番に関係なく値を指定したい場合など色々な場面で[Named Arguments](https://kotlinlang.org/docs/functions.html#named-arguments)を使うことができますね。例えば[joinToString()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/join-to-string.html)には`separator`、`limit`、`truncated`など6つのパラメータがあるのですが、デフォルト値が指定されていて、Named Argumentsにより以下のような使い方が可能です。

```kotlin
listOf("A", "B", "C", "D").joinToString(prefix = "[", postfix = "]")
```

Named ArgumentはKotlinにおいてはオプションで、基本的にはJava同様、関数に定義されてあるパラメータの順番に合わせて値を渡すだけでも問題ありません。しかし、Swiftではこれが逆になっていて、sturctのインスタンスを作る場合や関数を呼び出す場合は基本的にパラメータは基本的にNamed Argumentsのような形で渡す必要があります。これを[Argument Label](https://docs.swift.org/swift-book/LanguageGuide/Functions.html#ID166)と読んでいるそうです。

```swift
func printHello(string: String) {
    print("Hello \(string)!")
}

printHello(string: "world") // Function Argument Labelでstringを指定
```

ただ、これもKotlinと同様、デフォルト値を指定することができ、その場合はパラメータを省略することができます。

```swift
func printHello(string: String, times: Int = 1) {
    var count = 1
    repeat { // Kotlinのdo-whileループ的なもの
        print("Hello \(string)!")
        count += 1
    } while (count <= times)
}

printHello(string: "world") // timesを省略している
```

他にも、アンダースコアを使うことでArgument Labelを省略できるようにもなります。

```swift
func printHello(_ string: String, times: Int = 1) {
    var count = 1
    repeat {
        print("Hello \(string)!")
        count += 1
    } while (count <= times)
}

printHello("world") // stringを省略
```

関数を定義する側からしたらあまり似ていないような気もするのですが、呼び出す側としてはかなり似たような形でコードが書けるのが特徴的かなと思います。

#### Range

Kotlinでは[rangeTo()](https://kotlinlang.org/docs/ranges.html#:~:text=values%20using%20the-,rangeTo(),-function%20from%20the)を使って、簡単に数値の範囲を定義することができます。この関数は[operator](https://kotlinlang.org/docs/keyword-reference.html#operators-and-special-symbols)として定義されているので、`..`で簡単に使えます。こうやって定義したRangeでは、最小値と最大値の取得や、Listに変換するなど色々なことができます。

```kotlin
// Rangeの定義
val min = 10
val max = 20
val range = min..max

// 最小値と最大値の取得
println(range.start) // 10
println(range.endInclusive) // 20

// RangeをListにする
val intList = range.toList() // [10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
```

Swiftでも[Range Operator](https://docs.swift.org/swift-book/LanguageGuide/BasicOperators.html#ID73)を使って範囲を定義することができます。こちらも形は似ていて、`...`となります。ドットの数がKotlinより一つ多いという点を除くと全く同じ感覚で、最小値と最大値もまた名前が違うだけでプロパティとして取得できるという点もまた一緒です。

```swift
// Rangeの定義
let min = 10
let max = 20
let range = min...max

// 最小値と最大値の取得
print(range.lowerBound) // 10
print(range.upperBound) // 20

// RangeをArrayにする
let array = Array(range) // [10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
```

ただ、上記のコードを見ると気付きにくいところですが、Range実装については両言語での扱いが少し違うところがあります。Kotlinでは`rangeTo()`の戻り値が、元の値の型に合わせて`InteRange`や`LongRange`のようなものとなっていて、最小値と最大値をプロパティで取得する場合も`rangeTo`に渡された二つの値の型と一緒です。

しかし、Swiftの[Range](https://developer.apple.com/documentation/swift/range)は`Range<Bound>`という型で、当然Rangeから取得できる最小値や最大値も[Bound](https://developer.apple.com/documentation/swift/rangeexpression/bound/)の型となっています。IntやLongとはまた別の型になるので、場合によっては注意して使う必要があるかも知れません。

### Swiftだけのもの

今まではKotlinユーザの観点から、Kotlinとどれだけ同じ感覚でコードを書けるか、ということを述べていましたが、ここからは少し間隔が違うなと思ったところを少しまとめてみようと思います。

#### メソッド・プロパティコールでの省略

Kotlinでは、[apply()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html)のように自分自身を指しているのが明確な場合、`this`を省略することができます。以下のようにですね。

```kotlin
data class Student(
    val name: String,
    var age: Int = 0
)

val studentA = Student(name = "A").apply { age = 18 } // Student(name=A, age=18)
```

このように、`this`を使う場合か、明確に対象importしているなど特定のケースを除くとKotlinでは基本的に`Class.method()`のような形でどのクラスのメンバを呼び出しているかを表記するのが原則ですね。

しかし、Swiftの場合は少し状況が違います。もっとゆるい感じで、コンパイラを基準に対象が明確であれば、`.method()`のような形で省略できるような感覚です。以下はSwiftUIのチュートリアルで提示しているコードの一部を抜粋したものですが、`filter`が`FilterCategory`というenumであるため、`.all`という形で三項演算子の中で使われていることを確認できます。

```swift
struct LandmarkList: View {
    @State private var filter = FilterCategory.all

    enum FilterCategory: String, CaseIterable, Identifiable {
        case all = "ALL"
        case lakes = "Lakes"
        case rivers = "Rivers"
        case mountains = "Mountains"
        
        var id: FilterCategory { self }
    }

    var title: String {
        let title = filter == .all ? "Landmarks" : filter.rawValue
        return showFavoritesOnly ? "Favorite \(title)" : title
    }
}
```

#### Protocol

Swiftでは[Protocol](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html)というものがあり、JavaやKotlinのinterfaceと大体同じ感覚で使えます。ここまでだとあまり差はないように思いますが、実際にはstructやclass、enumなどを定義するときには、必要に応じでprotocolを採用(adopt)する必要があるというところが体験できる違いかなと思います。

例えば、Kotlinで一つのdata classを定義するとしたら、以下のようなメンバが自動て追加されます。

- equals()
- hashCode()
- toString()
- componentN()
- copy()

しかし、Swiftのstruct, class, enumなどにはこのようなメンバは基本的に追加されません。なので、必要なメンバがあればそれに関するprotocolを採用し、実装する必要があります。例えばハッシュ値が使いたい場合は[Hashable](https://developer.apple.com/documentation/swift/hashable)、
Jsonなどに変換するためには[Codable](https://developer.apple.com/documentation/swift/codable)、Listでループしたい場合は[Identifiable](https://developer.apple.com/documentation/swift/identifiable)、enumの全ケースを網羅してループしたい場合は[CaseIterable](https://developer.apple.com/documentation/swift/caseiterable)、同一化を比較したい場合は[Equatable](https://developer.apple.com/documentation/swift/equatable)を採用するなどです。

もちろんJavaやKotlinでも必要に応じてintefaceやannotationを使う必要はありますが、SwiftだとKotlinで気軽に使える機能がstructやclassなどを定義した時点では揃ってない可能性があるので、ここは気をつけるべきところですね。

#### some

Swiftでは少し変わった感覚のキーワードがあるます。そのキーワードの説明するために、まずは以下のようなprotocolとstructの定義があるとしましょう。

```swift
protocol Something {
    func define()
}

struct GoodThing: Something {
    func define() {
        print("It's good thing!")
    }
}
```

上記のようなコードがある場合、変数の型宣言や関数の戻り値で少し独特なキーワードを使うことができます。`some`というものです。実際使う時は、以下のようなコードとなります。

```swift
var good: some Something = GoodThing()

func returnSomething() -> some Something {
    return GoodThing()
}
```

これだけでは`some`というキーワードが一体どんなものかわからないですね。ここでKotlinの概念を持ってくるとどうでしょうか。実は、Kotlinでもこれによく似た機能があります。`<T extends Something>`です。KotlinやJavaの経験がある型ならこれで十分に何を意味しているかがしっくり来るかなと思います。

つまり、`some`はとあるprotocolを満足する何かしらのインスタンスを示すものです。Swiftではそれを満足するオブジェクトであってもprotocolを直接変数の型や関数の戻り値として定義して直接使うことはできない場合があります。その場合に`some`を使うことで問題を回避できます。JavaやKotlinでinterfaceを使って、その具体的な実装は問わなく使うのと一緒だと言えます。このキーワードのおかげで、SwiftUIでは[View](https://developer.apple.com/documentation/swiftui/view)を満足して入れば画面を構成するどんなコンポーネントとして扱えるようになります。

ただ、interfaceを扱うのとは概念的に同じだとしても、コードを書く側の感覚としては全く違うのでここは注意しなければならないと思います。

#### Compiler Control Statements

Swiftには[Compiler Control Statements](https://docs.swift.org/swift-book/ReferenceManual/Statements.html#ID538)という仕様があり、コンパイル時の処理を指定できます。例えば、SwiftUIのチュートリアルでは一つのアプリを実装して、OSによって違う機能を実現するためにこれを利用しているケースがあります。以下がその例です。

```swift
// watchOSで起動する場合は、通知を使う
#if os(watchOS)
WKNotificationScene(controller: NotificationController.self, category: "LandmarkNear")
#endif

// macOSで起動する場合は、設定を使う
#if os(macOS)
Settings {
    LandmarkSettings()
}
#endif
```

Kotlinの場合もAndroidでアプリを実装する場合はこのような設定が必要になる場面もあるかも知れませんが、バックエンドの経験上ではコードによりコンパイラをコントロールするというケースはあまりなかったので、かなり新鮮な感覚でした。

## 最後に

いかがでしたか。SwiftUIの話をするつもりが、Swiftのことだけでかなりの量になってしまったので、SwiftUIについては次のポストで述べようかなと思っています。しかし、Swiftだけでもかなり興味深いところが多かったので、やはりチュートリアルを触ってみて色々な経験ができたので良い選択をしたかなと思います。

また、やはりKotlinとSwiftがなんとなく似ている部分があるのは感覚的には確かなので、やはりどちらかの経験があると残りの片方への入門もしやすくなるのかなという感覚はあります。これは外国語の教育（自分の専攻です）でいうスキーマ、いわゆるバックグラウンドの知識ある故のことだろうなと思うと、少しうれしくもなりますね。やはりKotlinやってよかったなと思います。

では、また！
