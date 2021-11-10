---
title: "色々な言語でやってみた（ソート編）"
date: 2021-11-10
categories: 
  - languages
photos:
  - /assets/images/sideimage/magic.jpg
tags:
  - kotlin
  - java
  - javascript
  - python
  - swift
  - go
---

今はどんなプログラミング言語を選んでもできることはあまり違わなく、まさに好みで選んでもいいと思えるくらいの時代となっていると思います。特に、[Kotlin/JS](https://kotlinlang.org/docs/js-overview.html)のようなトランスパイラーやFlutterのようなフレームワークも続々と登場している時代なので、こういう傾向はこれからもどんどん加速していくのではないかと思います。

しかしそのような変化がある一方で、今現在はプログラマに一人が扱えるプログラミング言語の数に対する要求も増えいている状況ではないかと思います。実際の業務ではさまざまな理由で使われる言語が決まっていて、自分が今まで触ったことのないものでも使えるようになる必要があり、一人のエンジニアが固定されたポジションでなく、さまざまな分野にかけて実装を行うケースもありますしね。いわゆる[Polyglot](https://en.wikipedia.org/wiki/Polyglot_(computing))の時代とも言えます。

なので、少なくともいろいろな言語の特徴を把握しておくということが大事になっているのではないかと思います。そして、そのような必要によるものでなくても、自分が普段接してない言語のコンセプトに触れてみることで、メインとなる言語への理解が深まることもあるのではないのかなと思ったりもします。これはどんな言語でもできることはあまり変わらないということともある意味通じているのですが、他の言語のコンセプトを受け入れた新しいAPIや機能を導入したり、そのようなライブラリが登場する場合もあるので。

さて、前置きが長くなりましたが、ということで、これからはたまにとある操作をするときにいろいろな言語ではどうやってできるのか、そしてそうした場合の特徴などを簡単に比べてみたいと思います。今回は、配列のソートになります。

## JavaScript

JavaScriptでは[Array.prototype.sort()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/sort)で配列のソートができます。なので、以下のようなコードを使えます。シンプルですね。

```javascript
> const a = [22, 1, 44, 300, 5000]
> a.sort()
[ 1, 22, 300, 44, 5000 ]
```

また、元の配列の値は変更せず、新しくソートされた配列を作りたい場合は以下の方法を使えます。

```javascript
> const a = [22, 1, 44, 300, 5000]
> const b = [...a].sort() // aをコピーしてソート
> console.log(b)
[ 1, 22, 300, 44, 5000 ]
```

ただ、ここで気づいた方もいらっしゃると思いますが、ソートされた値が期待通りにはなっていません。本当なら、`1, 22, 44, 300, 5000`になるのが普通でしょう。ここで昇順に値をソートしたい場合は、ソートの方法を自前で作成する必要があります。例えば以下のような方法がありますね。

```javascript
> const a = [22, 1, 44, 300, 5000]
> a.sort((a, b) => a - b)
[ 1, 22, 44, 300, 5000 ]
```

この`sort()`では、引数として渡す`compareFunction`（引数が二つ、戻り値はnumber）の戻り値の結果によって、以下のことが起こります。

- 0より小さいと、aのインデックスをbの先に置く
- 0だと、aとbは変更しない
- 0より大きいと、bのインデックスをaの先に置く

これはJavaをやっていた方だと、[Comparator](https://docs.oracle.com/javase/8/docs/api/java/util/Comparator.html)と同じだなとすぐわかる内容ですね。アロー関数の形もJavaのLambdaに似ているので、あまり違和感なく適応できるかと思います。かなりシンプルなのですが、number型の配列に対しては自前の`compareFunction`が必要となるということは大事なので、気を付ける必要はあるでしょう。

配列のインデックスを反転したい場合は、[Array.prototype.reverse()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/reverse)を使うだけで良いです。この場合はnumberの配列でも自前の`compareFunction`が必要ないので、便利ですね。

```javascript
> const a = [22, 1, 44, 300, 5000]
> a.reverse()
[ 5000, 300, 44, 1, 22 ]
```

## Java

では、次にJavaの方も見ていきましょう。先に述べた通り、`Comparator`を使うと簡単にソートの方法を実装できるので、基本的には同じです。ただ、Javaの場合だとそもそも[List.sort()](https://docs.oracle.com/javase/8/docs/api/java/util/List.html#sort-java.util.Comparator-)、[Collections.sort()](https://docs.oracle.com/javase/jp/8/docs/api/java/util/Collections.html#sort-java.util.List-)、[Arrays.sort()](https://docs.oracle.com/javase/jp/8/docs/api/java/util/Arrays.html#sort-int:A-)、[Stream.sorted()]など方法が色々あり、ソートしたいCollectionやArrayなどが`Immutable`であるかどうか、`Comparator`や[Comparable](https://docs.oracle.com/javase/8/docs/api/java/lang/Comparable.html)を自前で実装するか、それともスタンダードライブラリに用意されてあるものを使うかなどのさまざまな選択肢も考慮する必要があるということですね。

色々な選択肢がある中で、もっとも簡単なのは、`Collections.sort()`や`Arrays.sort()`を使う方法かなと思います。これを使う場合、primitive型やStringのListは短いコードでソートができるという（そして標準機能という）メリットがありますね。

```java
jshell> var a = new ArrayList<>() {{ add(22); add (1); add(44); add(300); add(5000); }};
a ==> [22, 1, 44, 300, 5000]

jshell> Collections.sort(a);

jshell> System.out.println(a);
[1, 22, 44, 300, 5000]
```

次に、`List.sort()`が簡単です。`Comparator`を引数として渡す必要がありますが、昇順・降順でソートしたい場合は既に用意されてあるメソッドを呼び出すだけですね。

```java
jshell> var a = new ArrayList<>() {{ add(22); add (1); add(44); add(300); add(5000); }};
a ==> [22, 1, 44, 300, 5000]

jshell> a.sort(Comparator.naturalOrder());

jshell> System.out.println(a);
[1, 22, 44, 300, 5000]
```

ちなみに`Comparator`で使える既定のソート方法は以下があります。

- 昇順: [naturalOrder()](https://docs.oracle.com/javase/jp/8/docs/api/java/util/Comparator.html#naturalOrder--)
- 降順: [reverseOrder()](https://docs.oracle.com/javase/jp/8/docs/api/java/util/Comparator.html#reverseOrder--)
- 逆順: [reversed()](https://docs.oracle.com/javase/jp/8/docs/api/java/util/Comparator.html#reversed--)

また、`Comparator`は、`Collections.sort()`の引数としても使えます。なので、降順にソートしたい場合は以下のようなコードを使えます。

```java
jshell> var a = new ArrayList<>() {{ add(22); add (1); add(44); add(300); add(5000); }};
a ==> [22, 1, 44, 300, 5000]

jshell> Collections.sort(a, Comparator.reverseOrder());

jshell> System.out.println(a);
[5000, 300, 44, 22, 1]
```

他に、元のListの値を変更せず、新しくソートされた結果を取得したい場合は、元のListをコピーする方法もありますが、もう一つの方法として`Stream`を使う方法を考えられます。

```java
jshell> var a = List.of(22, 1, 44, 300, 5000);
a ==> [22, 1, 44, 300, 5000]
jshell> var b = a.stream().sorted().collect(Collectors.toList());
b ==> [1, 22, 44, 300, 5000]
```

`Stream`でソートする場合でも、`Comparator`を使えます。

```java
jshell> var a = List.of(22, 1, 44, 300, 5000);
a ==> [22, 1, 44, 300, 5000]
jshell> var b = a.stream().sorted(Comparator.reverseOrder()).collect(Collectors.toList());
b ==> [5000, 300, 44, 22, 1]
```

また、DTOのListをソートしたい場合は、DTOが`Comparable`を継承するという方法も考えられますが、多くの場合はソート時の条件が明確にわかる`Comparator`を実装したいいかなと思います。汎用性や柔軟性を考えても、`Comparable`の場合、条件が変わるとクラスを修正する必要があるので、`Comparator`を使った方が無難かなと思います。

Arrayの場合、`Arrays.sort()`を利用してソートできる（もちろん`Comparator`も使えます）上に、ListやStreamに変換することもできるので上記の方法をそのまま使えます。なので選択肢はもっと多いわけですが、便利な（好みに合う）方法を選ぶといいかなと思います。個人的には`Arrays.sort()`に`Comparator`を渡した方が可読性という面で良さそうな気がします。

## Kotlin

Sytax Sugarをたくさん提供しているKotlinらしく、選べるソートのオプションがたくさんあります。なので、少しまとめてみました。

| Orderの種類 | ソート結果 | fun | 備考 |
|---|---|---|---|
| Natural | 呼び出し元 | [Array/MutableList.sort()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sort.html) | 昇順 |
|  |  | [Array/MutableList.sortDescending()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sort-descending.html) | 降順 |
|  |  | [Array/MutableList.reverse()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/reverse.html) | 逆順 |
|  | Array | [Array.sortedArray()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-array.html) | 昇順 |
|  |  | [Array.sortedArrayDescending()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-array-descending.html) | 降順 |
|  |  | [Array.reveredArray()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/reversed-array.html) | 逆順 |
|  | List | [Array/List.sorted()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted.html) | 昇順 |
|  |  | [Array/List.sortedDescending()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-descending.html) | 降順 |
|  |  | [List/MutableList.asRevered()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/as-reversed.html) | 逆順 |
| Custom | 呼び出し元 | [Array/MutableList.sortBy()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sort-by.html) | 昇順、selector((T) -> R)必要 |
|  |  | [Array/MutableList.sortByDescending()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sort-by-descending.html) | 降順、selector((T) -> R)必要 |
|  | List | [Array/Iterable.sortedBy()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-by.html) | 昇順、selector((T) -> R)必要 |
|  |  | [Array/Iterable.sortedByDescending()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-by-descending.html) | 降順、selector((T) -> R)必要 |
|  | Array | [Array.sortedArrayWith()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-array-with.html) | [Comparator](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-comparator/)必要 |
|  | List | [Array/Iterable.sortedWith()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/sorted-with.html) | [Comparator](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-comparator/)必要 |

かなり多い選択肢があるように見えますが、こうやって表としてまとめてみるとまぁまぁわかりそうな気はします。自前の比較処理を書く必要があるか、ソートした結果が元の配列かどうか、そしてArrayになるかListになるかなどいくつかの基準で分けられるということが分かれば大体どれを使った方がいいか悩む必要はないかなと思います。

なので、まずやりたいことを明確にした上で、どのAPIを使うかを選んで書くだけです。以下はListから、ソートされた新しいListを作成する例です。それぞれ昇順と降順の場合となっています。

```kotlin
>>> val a = listOf(22, 1, 44, 300, 5000)
>>> val b = a.sorted()
>>> println(b)
[1, 22, 44, 300, 5000]
>>> val c = a.sortedDescending()
>>> println(c)
[5000, 300, 44, 22, 1]
```

また、data classの配列をソートしたい場合は`sortBy`や`sortedBy`を使えます。ここで引数に必要なのは`(T) -> R`型のselectorですが、単純にどれを基準にソートするかを指定すれば良いだけですので実装は簡単です。以下の例を見てください。

```kotlin
>>> data class Data(val number: Int)
>>> val a = listOf(Data(22), Data(1), Data(44), Data(300), Data(5000))
>>> val b = a.sortedBy { it.number }
>>> println(b)
[Data(number=1), Data(number=22), Data(number=44), Data(number=300), Data(number=5000)]
>>> val c = a.sortedByDescending { it.number }
>>> println(c)
[Data(number=5000), Data(number=300), Data(number=44), Data(number=22), Data(number=1)]
```

他にも、より複雑な比較の条件を指定したい場合はJavaの場合と同じく、`Comparator`を実装すると良いでしょう。やはりJavaと似ているようで、より単純化した（そしてそのせいで選択肢は増えた）感覚ですね。

## Swift

Swiftでは、シンプルに元のCollectionをソートするかソートされた新しいCollectionを作るかの選択肢しかないようです。あまり変わったことはないですが、元のCollectionをソートする場合は以下のようになります。

```swift
  1> var a = [22, 1, 44, 300, 5000]
a: [Int] = 5 values {
  [0] = 22
  [1] = 1
  [2] = 44
  [3] = 300
  [4] = 5000
}
  2> a.sort()
  3> print(a)
[1, 22, 44, 300, 5000]
```

そして新しいCollectionを作成したい場合は以下のようになります。

```swift
  1> let a = [22, 1, 44, 300, 5000]
a: [Int] = 5 values {
  [0] = 22
  [1] = 1
  [2] = 44
  [3] = 300
  [4] = 5000
}
  2> let b = a.sorted()
b: [Int] = 5 values {
  [0] = 1
  [1] = 22
  [2] = 44
  [3] = 300
  [4] = 5000
}
  3> print(b)
[1, 22, 44, 300, 5000]
```

ただ、Swiftのソートが独特なのはどうやってソートするか、その方法を指定する時です。[sort()](https://developer.apple.com/documentation/swift/array/2296801-sort)でも[sorted()](https://developer.apple.com/documentation/swift/array/2296815-sorted)でも引数として`areInIncreasingOrder`という関数を渡すことができるようになっていますが、JavaScriptやJava、Kotlinで使われていた`compareFunction`や`Comparator`の戻り値が数字であったことに対して、`areInIncreasingOrder`はpredicate型として戻り値がBoolになっています。なので以下のような形でソートの方法を指定可能です。

```swift
let students: Set = ["Kofi", "Abena", "Peter", "Kweku", "Akosua"]
let descendingStudents = students.sorted(by: >)
print(descendingStudents) // "["Peter", "Kweku", "Kofi", "Akosua", "Abena"]"
```

他に、classのフィールドを基準にソートしたい場合は以下の方法を使えます。

```swift
class data { var number = Int() }

var datas : [data] = []
let data1 = data()
data1.number = 1
datas.append(data1)
let data2 = data()
data2.number = 2
datas.append(data2)

let descending = datas.sorted { $0.number > $1.number }
dump(descending)
/**
  result: [data] = 2 values {
    [0] = {
      number = 2
    }
    [1] = {
      number = 1
    }
  }
*/
```

## Go

Goにはジェネリックがないからか、[sort](https://pkg.go.dev/sort)というパッケージに、sliceの種類によってソート用のfuncが色々と用意されています。例えば以下のようなものがあります。

- func Float64s(x []float64)
- func Ints(x []int)
- func Strings(x []string)

なので、structのsliceではい場合はこれらの中でどれかを選んでソートすることになりますね。例えば以下のようになります。

```go
a := []int{22, 1, 44, 300, 5000}
sort.Ints(a)
fmt.Println(a) // [1 22 44 300 5000]
```

structの場合は、以下のような方法が使えます。ソートの基準がまた`bool`になっています。

```go
people := []struct {
  Name string
  Age  int
}{
  {"Gopher", 7},
  {"Alice", 55},
  {"Vera", 24},
  {"Bob", 75},
}
sort.Slice(people, func(i, j int) bool { return people[i].Name < people[j].Name })
fmt.Println(people) // [{Alice 55} {Bob 75} {Gopher 7} {Vera 24}]
```

面白いのは、Goのソートには[sort.SliceStable()](https://pkg.go.dev/sort#SliceStable)というものが別に存在しているということです。これは[安定ソート](https://ja.wikipedia.org/wiki/%E5%AE%89%E5%AE%9A%E3%82%BD%E3%83%BC%E3%83%88)を行うもので、その定義に関してはWikiでは以下のように述べています。

> 同等なデータのソート前の順序が、ソート後も保存されるものをいう。つまり、ソート途中の各状態において、常に順位の位置関係を保っていることをいう。

つまり、安定ソートの場合、ソートの基準となる値が同等の要素間の元の位置関係（インデックス）が保証されるということですね。その結果が実際どうなるのかを見てみましょう。

```go
people := []struct {
  Name string
  Age  int
}{
  {"Alice", 25},
  {"Elizabeth", 75},
  {"Alice", 75},
  {"Bob", 75},
  {"Alice", 75},
  {"Bob", 25},
  {"Colin", 25},
  {"Elizabeth", 25},
}

sort.SliceStable(people, func(i, j int) bool { return people[i].Age < people[j].Age })
fmt.Println(people) // [{Alice 25} {Bob 25} {Colin 25} {Elizabeth 25} {Alice 75} {Alice 75} {Bob 75} {Elizabeth 75}]
```

コードの実行結果でわかるように、`Alice 25`、`Bob 25`、`Colin 25`、`Elizabeth 25`と`Alice 75`, `Bob 75`, `Elizabeth 75`の元の順が維持されたままソートされたのがわかります。ここでもし`sort.Slice()`を使うと以下のようになります。

```go
sort.Slice(people, func(i, j int) bool { return people[i].Name < people[j].Name })
fmt.Println(people) // [{Alice 25} {Alice 75} {Alice 75} {Bob 75} {Bob 25} {Colin 25} {Elizabeth 75} {Elizabeth 25}]
```

安定ソートはそうでないソートに比べ性能が劣る可能性が高いので（元のインデックスをまで考慮しているので）、一つの値を基準にソートしても問題ない場合は`sort.Slice()`でも十分な気がしますが、そうでない場合は安定ソートを考慮する必要がありそうですね。

## Python

Pythonでは[list.sort()](https://docs.python.org/3/library/stdtypes.html#list.sort)か、[sorted()](https://docs.python.org/3/library/functions.html#sorted)を使えます。他の言語でも大体同じだったので命名だけでも推測が可能かと思いますが、前者は元のlistをソートするもので、後者は新しいlistを作り出すものです。

まず`list.sort()`は、以下のように使えます。他の言語とあまり変わらないですね。

```python
>>> a = [22, 1, 44, 300, 5000]
>>> a.sort()
>>> print(a)
[1, 22, 44, 300, 5000]
```

それに対して、`sorted()`は以下のように使えます。

```python
>>> a = [22, 1, 44, 300, 5000]
>>> b = sorted(a)
>>> print(b)
[1, 22, 44, 300, 5000]
```

また、これらの関数では`key`や`reverse`のようなパラメータを指定することで、どれを基準にソートするか、逆順にソートするかなどを指定できます。Pythonらしいシンプルさですね。

```python
class Data:
  def __init__(self, number):
    self.number = number
  def __repr__(self):
    return repr((self.number))
 
datas = [Data(1), Data(3), Data(2), Data(4)]
datas.sort(key=lambda data: data.number) # [1, 2, 3, 4]
sorted(datas, key=lambda data: data.number, reverse=True) # [4, 3, 2, 1]
```

## 番外：Stable sort

Goのソート方法の中で少し安定ソートの話が出ましたが、ここで比較した他の言語だとGoのように安定ソートとそうでないソートのどれを使うかという選択肢がなかったので、それぞれの言語での安定ソートはどうやって扱われているのかを表にしてみました。以下をご覧ください。

| 言語 | stable | non-stable | 備考 |
|---|---|---|---|
| Go | ⭕️ | ⭕️ | funcによって選べられる |
| Java | ⭕️ | ⭕️ | Streamはnon-stable |
| JavaScript | ⭕️ | ⭕️ | ブラウザのバージョンによる |
| Python | ⭕️ | ❌ | |
| Kotlin | ⭕️ | ❌ | SequenceでもStable |
| Swift | ❌ | ⭕️ | `stableを保証できない`と表現 |

多くの言語が安定ソートに対応していますが、少しづつ仕様が違う場合がありました。例えばJavaの場合、Streamによるソートは安定ソートではないため、安定ソートの結果を保証したい場合は既にソートされたCollectionを使うことをおすすめしています。Kotlinの場合はStreamに似たSequenceを使う場合でも、`stateful`なためか、安定ソートに対応していました。

また、JavaScriptの場合はブラウザのバージョンによって違いますが、最新のブラウザを使っている場合は大抵安定ソートに対応していました。ただ、JavaScirptを使った案件の場合はIEも対象ブラウザとして考慮される場合があるのですが、IEだと安定ソートに対応していないので確認が必要かなと思います。

Swiftの場合はまだソート時のデフォルト値をstableにするかどうかを検討している中で、APIとしてもGoのようにstableとそうでないものを分離するかどうかを検討しているらしいです。またどのアルゴリズムを使うかについて議論しているらしく、しばらくは安定ソートを期待できないかと思います。

KotlinとPythonはどの場合でも安定ソートとなるので、悩み事が一つ減るのが嬉しいですね。

## 最後に

今回は色々な言語のソートについて調べてみましたが、いかがでしたか。一度ソートしたデータはその後の要素に対するアクセスが早くなるので、チューニングの観点からは必要なものかと思います。そしてこうやって色々な言語のソートのAPIを調べてみると、その言語の設計思想や発展の過程のようなものが少し見えるようで面白く、勉強にもなりますね。個人的にはあまり意識してなかった安定ソートがかなり勉強になりました。

これからもこうやって色々な言語の使用やAPI、同じことをする場合の各言語による違いなどを比べてみたいと思います。時間と体力が十分であればの話ではありますが…！

では、また！
