---
title: "Kotlinプログラマが見たRust"
date: 2022-03-27
categories: 
  - rust
image: "../../images/rust.jpg"
tags:
  - rust
  - python
  - kotlin
  - java
  - go
---

Rustの勉強を始めたい、と思ったのはおよそ2年前のことです。当時はJavaとPythonを主に触っていたので、パフォーマンスがクリティカルな部分では対応しきれない部分があると思い、ネイティブにコンパイルされる言語に触れてみる必要があると思いました。そしてできれば、GCのなくポインタを扱う言語でアプリを書いてみたら、本業と言えるJavaの理解もより深くなるのではないかと思った次第です。

そこで候補として考えたのがGoとRustです。ただ、Goは世間の評価はともかく、自分の立場からすると少し追求している目標とずれているところがあるなと思いました。特に転職してからGoとKotlinという言語を並行で触っていると、良くも悪くも自分がやりたいことがなんなのかわかってきた気分にもなったのです。

そこで、そろそろ次の候補として考えていたRustに触れてみたいと思った次第です。これもまた、世間の評価は置いといて、実際自分に合うかどうかを確認してみたくなりました。最近は色々な言語が扱える[Polyglot Programmer](https://medium.com/@guestposts_92864/what-is-a-polyglot-programmer-and-why-you-should-become-one-e5629bf720c2)の時代だという概念もあり、多くのプログラミング言語が互いの良いところを吸収しながらどれも似たようなものになったという評価もありますが、私の場合は、あくまで自分に合うのは何かを探るという感覚としてRustという新しい言語を接してみたいと思っています。

なので、今回はまずこちらの[The Rust Programming Language](https://doc.rust-jp.rs/book-ja/title-page.html)を読みながら、興味深かった部分について、自分が今まで経験してみた他のプログラミング言語と比べながら感想を述べたいと思います。ドキュメントが長く、自分の理解もまだ浅いのでまずは一部だけを紹介しましょう。

## Loop

Rustでは伝統の`for`と`while`以外にも、ループの条件を指定しない(無限ループ)`loop`というのがありました。特定の条件でループを終了したい場合のみ、`break`することで終了できます。例えば以下のような形です。

```rust
loop {
    // do something
}
```

他の言語だと、普通は`while(true)`のような形が多いかと思います。例えばPythonは以下のようになりますね。

```python
while true:
  # do something
```

KotlinやJavaでも事情は変わりません。以下のようになりますね。

```java
while(true) {
    // do something
}
```

Kotlinの場合だと、拡張関数があるので`loop`というものを定義したらどうかと思いましたが、そうするとコンパイラ上でループだと認識されないので、`break`を書くとコンパイルエラーとなります。なので以下のように拡張関数と作ることはできませんでした。

```kotlin
fun test() {
  loop {
    break // コンパイルエラー
  }
}

fun loop(doSomething: () -> Unit) {
  while(true) {
    doSomething()
  }
}
```

Goの場合は、`for`に条件式を書かないことでシンプルな無限ループを書くことができます。

```go
for {
    // do something
}
```

個人的に`while(true)`や条件式を指定しない`for`は、慣習でしかなく、直感的な理解を招くものではないと思いますので、`loop`というキーワードを設けた方がコードの可読性という面ではよりわかり安いものなのではないかと思いました。細かい部分ではありますが、一回仕様としてとして決まるとなかなか変更できないものなので、どんなキーワードを使うかを決めるということも言語の設計においては大事だという気がします。

### Array

Rustでは配列のindexを基準に一部を抽出するとき、以下のような書き方をします。参照(`&`)を利用して定義する必要があって、標準出力する形も少し独特ですね。また、indexを指定して切り出したものは「所有権のない別のデータ型」として定義されています。ここで切り抜いた配列の一部を、Rustでは`slice`と呼んでいるらしいです。

```rust
let arr = [0, 1, 2, 3, 4];
let slice = &arr[1..3];

println!("{:?}", slice); // [1, 2]
```

Pythonでもかなり似たような感じでコードが書けます。以下は上記と同じ挙動をするコードの例です。ただ、ここで切り抜いた`slice`のデータ型は同じく`list`になるというのがRustとの違いですね。

```python
list = [0, 1, 2, 3, 4]
slice = list[1:3]
print(slice) # [1, 2]
```

Kotlinの場合は、[List](https://kotlinlang.org/docs/collections-overview.html#list)の関数に[Range](https://kotlinlang.org/docs/ranges.html)オブジェクトを渡すことで同じことができます。少し問題になるのは、Kotlin特有の`Range`の書き方がどのような範囲を示すのか覚えてないとその範囲が分かりづらいということです。幸い、ここはIntellij Idea 2021.3のアップデートで[ヒントを表示](https://blog.jetbrains.com/idea/2021/10/intellij-idea-2021-3-eap-5/#inline_hints_for_ranges)してくれるようになったので、これ以前のバージョンを使っている場合はアップデートした方が良いですね。

```kotlin
val list = listOf(0, 1, 2, 3, 4)
val subList = list.slice((1..2))
println(subList) // [1, 2]
val subListUntil = list.slice((1 until 3))
println(subListUntil) // [1, 2]
```

Javaの場合は、インデックスの範囲を[List.subList()](https://docs.oracle.com/javase/jp/8/docs/api/java/util/List.html#subList-int-int-)に渡すことで同じことができます。

```java
List<Integer> list = List.of(0, 1, 2, 3, 4);
List<Integer> subList = list.subList(1, 3);
System.out.println(subList); // [1, 2]
```

Goの場合は、Pythonと全く同じ方法で定義ができますね。また、配列からインデックスの範囲を指定して切り取ったviewを`slice`と呼ぶのはRustと一緒です。ただ、Goのsliceはarrayと違って、可変長ですね。

```go
arr := []int{0, 1, 2, 3, 4}
slice := arr[1:3]
fmt.Println(slice) // [1 2]
```

## Immutability

Rustでの変数の宣言は基本的に`let`一つで、不変になります。もちろん可変できる変数を定義するのは不可能ではなくて、以下のように`mut`キーワードを使うことで値を再代入することはできます。例えば以下のようにです。

```rust
fn main() {
    let mut x = 5;
    println!("The value of x is: {}", x); // The value of x is: 5
    x = 6;
    println!("The value of x is: {}", x); // The value of x is: 6
}
```

他のプログラミング言語だと、変数の宣言時にその変数の可変性をあらかじめキーワードで表現するようになっているケースが多いかと思います。もしくは、基本的に変数は可変で、不変にしたい場合にだけ特別なキーワードを使うとかですね。しかし、Rustでは変数は基本的にimmutableであるというのが特徴的です。GCのない言語として、メモリの安全性を確保するための工夫がここで現れていると言っていいでしょうか。

もちろん、Pythonのように変数の宣言と再代入の区別が付かないケースもありますね。

```python
x = 5
print("The value of x is: {}".format(x)) # The value of x is: 5
x = 6
print("The value of x is: {}".format(x)) # The value of x is: 6
```

Kotlinの場合は不変だと`val`、可変だと`var`で宣言するようになっていますね。

```kotlin
val x = 5
x = 5 // コンパイルエラー
var y = 6
y = 7 // OK
```

Javaの場合は、Rustと逆です。`final`をつけない場合、基本的に再代入ができる構造ですね。

```java
int x = 5
x = 6 // OK
final int y = 6
y = 7 // コンパイルエラー
```

Goの場合は、変数をimmutableにできる方法はないようですね。なので、再代入は自由ですが、逆にJavaの`final`のようなキーワードが欲しい気分にもなります。

```go
x := 5
x = 6
fmt.Println(x) // 6
```

## Shadowing

これは全く予想できなかった部分ですが、Rustのドキュメントには変数にシャドーイングを使えると紹介されています。`mut`キーワードをつけると再代入は可能なので、それで良いのではという気もしますが、変数を不変にしながら、違うデータ型として定義し直す場合などに使えるという説明でした。

Rustではシャドーイングを使って以下のようなコードを作成できます。

```rust
fn main() {
    let x = 5;
    let x = x + 1; // 6
    let x = x * 2; // 12

    println!("The value of x is: {}", x); // The value of x is: 12
}
```

Pythonの場合も似たようなことができます。同じ挙動をするコードを以下のように書くと、問題なく動きます。変数の宣言と再代入が厳密に区別されない故のことかと思いますが、形的にはRustと全く一緒と言えますね。

```python
x = 5
x = x + 1
x = x * 2
print("The value of x is: {}".format(x)) # The value of x is: 12
```

Kotlinの場合、シャドーイングは一部の場合のみ可能です。関数の引数と、その関数で宣言している変数名が一致する場合ですね。

```kotlin
fun shadow(value: Int) {
  val value = value + 1 // Name shadowed: value
  println(value) // valの方が出力される
}
```

Javaでは、シャドーイングができないです。ただ、以下のような形は可能です。

```java
class Clazz {
  private int value = 0;

  public void setValue(int value) {
    this.value = value;
  }
}
```

Goの場合、少し複雑になります。以下の例をみると、`x`の宣言と代入を2回していますが、スコープが分かれてあるから可能なことです。Kotlinのケースと似ているとも言えますね。

```go
x := 0
fmt.Println("Before the decision block, x:", x) // Before the decision block, x: 0

if true {
  x := 1
  x++
}
fmt.Println("After the decision block, x:", x) // After the decision block, x: 0
```

上記のコードは、以下のようにif文での`x`に対して再代入することで挙動が変わります。

```go
x := 0
fmt.Println("Before the decision block, x:", x) // Before the decision block, x: 0

if true {
  x = 1
  x++
}
fmt.Println("After the decision block, x:", x) // After the decision block, x: 2
```

このように他の言語だとなるべく使わないように誘導しているシャドーイングですが、Rustでは一つの機能として紹介しているのが面白いところでした。これもまた、後述する「所有権」というものと強く関係しているような気がします。

## Ownership

他の言語と比べたときに、Rustならではの特徴と言えるものは所有権ではないでしょうか。今まで私はGCのない言語を触ってみたことがないので、これはかなり興味深い概念でした。例えばKotlinの場合はNativeでコンパイルする場合、[参照カウント](https://blog.jetbrains.com/kotlin/2021/05/kotlin-native-memory-management-update/#kn-gc)を使うと言われています。JavaやPython, Goの場合はGCが働いて参照されていないオブジェクトが占めているメモリを解放することになりますね。

しかし、Rustでは定数、不動小数点数、論理値、文字というスカラー型を除いた全ての参照型に関しては「一度使われたらメモリは解放される」「スコープを外れたら解放される」という原則を持っているようです。参照型とスカラー型という区分はJavaのプリミティブ型と参照型の関係を思い出させるところがありますね。より積極的かつ攻撃的なメモリ解放が行われるという違いはありますが。

基本的には一回使った変数に対しては2回使えなかったり、値の変更ができないかと思った方が良い、ということかなと思いますが、他にも色々と興味深いものがありました。

### Move

所有権と関係する概念で、ムーブがあります。変数とデータが実際どうやって相互作用するかによるものらしいです。早速下のコードを見ていきましょう。なんの問題もなさそうなものです。

```rust
let s1 = "hello";
let s2 = s1;

println!("s1 = {}, s2 = {}", s1, s2); // s1 = hello, s2 = hello
```

ただ、上記の`String literal`を`String`に変えたら問題が起こります。以下のコードを見ましょう。

```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{}, world!", s1);
```

上記のコードは、コンパイルしようとすると以下のようなエラーが発生します。

```shell
error[E0382]: use of moved value: `s1`
 --> src/main.rs:4:27
  |
3 |     let s2 = s1;
  |         -- value moved here
4 |     println!("{}, world!", s1);
  |                            ^^ value used here after move
  |
  = note: move occurs because `s1` has type `std::string::String`,
which does not implement the `Copy` trait
```

つまり、`s1`のデータが`s2`に移動したのでもう使えないということです。なので、二つの変数に同じデータを保証したい場合は、明示的に値をコピーする必要があります。例えば以下のようにです。

```rust
let s1 = String::from("hello");
let s2 = s1.clone();
println!("s1 = {}, s2 = {}", s1, s2); // s1 = hello, s2 = hello
```

なぜこうなっているかというと、Rustでは変数がスコープの外に移動するときにメモリの解放が起こりますが、ここで複数の変数が同じポインタを使っている場合は二重解放が起こる危険があるからと説明されています。また、`String literal`と違って`String`はimmutableではないので、s1の再代入でs2のデータまで変わってしまうという問題を防ぐための意図もあるような気がします。

実際このような代入が問題となる言語のケースもありますね。例えばPythonの方を見ましょう。二つの変数が同じポインタを使っているので、再代入で両方とも値が変更されたのを確認できます。

```python
s1 = "hello"
s2 = s1
print("s1 = {}, s2 = {}".format(s1, s1)) # s1 = hello, s2 = hello

s1 = "world"
print("s1 = {}, s2 = {}".format(s1, s1)) # s1 = world, s2 = world
```

JavaではStringをimmutableとして扱っているため、s1の値を再代入してもs2に影響はありません。KotlinもJVMの場合は、基本的にJVMのバイトコードを生成するためか同じ挙動をします。以下をご覧ください。

```kotlin
var s1 = "hello"
val s2 = s1
println("s1 = $s1, s2 = $s2") // s1 = hello, s2 = hello

s1 = "world"
println("s1 = $s1, s2 = $s2") // s1 = world, s2 = hello
```

Javaの場合も前述した通りです。

```java
var s1 = "hello";
var s2 = s1;
System.out.println(String.format("s1 = %s, s2 = %s", s1, s2)); // s1 = hello, s2 = hello

s1 = "world";
System.out.println(String.format("s1 = %s, s2 = %s", s1, s2)); // s1 = world, s2 = hello
```

Goでも変数はimmutableとして定義できませんが、この再代入により値が変わる可能性があるものに対しては安全性を担保されています。

```go
s1 := "hello"
s2 := s1
fmt.Println(fmt.Sprintf("s1 = %s, s2 = %s", s1, s2)) // s1 = hello, s2 = hello


s1 = "world"
fmt.Println(fmt.Sprintf("s1 = %s, s2 = %s", s1, s2)) // s1 = world, s2 = hello
```

Rustで明示的にコピーをしない場合はデータそのものが移動してしまうというのは確かにコーディング時には気を使わないといけないものですが、幸いコンパイルタイムで確認できる問題であり、他の言語を扱うときには思わぬ挙動をする可能性がある習慣を矯正してくれる可能性もあるかなと思うと、良い仕様ではなイカという気もしますね。

## Closure

Rustではclosureを関数内の関数として定義することももちろん可能ですが、`|val| val + x`の形式で書きます。他の言語でlambdaと呼ばわれているものですね。多少は独特な書き方な気もしますが、型の省略が可能なのが他の言語と比べ便利なものな気がします。もちろん型の明示的な表記もできるので、以下のような使い方ができます。

```rust
fn main() {
    // i32の引数を必要とする場合
    let closure_annotated = |i: i32| -> i32 { i + 1 };
    let closure_inferred  = |i     |          i + 1  ;

    let i = 1;
    println!("closure_annotated: {}", closure_annotated(i)); // closure_annotated: 2
    println!("closure_inferred: {}", closure_inferred(i)); // closure_inferred: 2

    // 引数がない場合
    let one = || 1;
    println!("closure returning one: {}", one()); // closure returning one: 1
}
```

Pythonの場合は以下のように書くことができますね。もちろん、関数の中に関数を定義することもできますが、lambdaを使った方がが便利なのかなと思います。ただ、[3.5から型ヒントを使える](https://docs.python.org/3/library/typing.html)ようになっていて、コンパイルタイムで確実にエラーをチェックしたい場合は明示的に型を書いたほうが良さげな気はします。

```python
closure = lambda x : x + 1
print(closure(1)) // 2
```

Kotlinでも簡単に定義はできるものですが、少なくとも引数の型は書く必要があります。もしくは、変数に型を指定することが必要ですね。

```kotlin
val closure = { x: Int -> x + 1 }
println(closure(1)) // 2
```

Javaではメソッド内にメソッドを定義することができなく、1.8から追加された`Functional Interface`を使う必要があります。また10からは`var`で型推論を使えるようになりましたが、Functional Interfaceをvarとして宣言するのはできないという制約があります。他の言語と比べると最も制約が多いですね。

```java
Function<Integer, Integer> closure = i -> i + 1;
System.out.println(closure.apply(1)); // 2
```

Goの場合は関数内に関数を定義するのは不可能ではないものの、他の言語のlambdaのような書き方はできず、匿名関数として定義ができます。また型を明示する必要があるので、名前を除いて完全な関数を定義して変数に代入しているようなものになりますね。

```go
closure := func(x int) int {
  return x + 1
}
fmt.Println(closure(1)) // 2
```

また、closureにおいてRustの特徴はもう一つあります。closureを引数とする関数を定義するときの書き方です。closureに対してgenericを使って、whereというキーワードで関数の中にclosureを書いていくような形です。他の言語だとclosureが引数でも書き方は大きく変わらないのですが、Rustでは全く違う形になっているのが興味深いですね。例えば以下のようなコードになります。

```rust
// Fというclosureを引数とする関数
fn apply_to_3<F>(f: F) -> i32 where
    // Fはi32を受け取ってi32を返すclosure
    F: Fn(i32) -> i32 {
    f(3)
}

fn main() {
    let double = |x| 2 * x;
    println!("3 doubled: {}", apply_to_3(double)); // 3 doubled: 6
}
```

## 最後に

まだドキュメントの半分の読んでなく、実際に何かしらのアプリを作ってみたわけでもないので今回のポストだけでは十分ではないというのは十分承知のつもりですが、久々に違う言語を学びながら、色々と興味深いところが多かったのでひとまず感想を書いてみました。

噂ではRustのコンパイラは優秀で、そのコンパイラの指示通りにアプリを組むだけでのかなり勉強になる瞬間が多いというのと、言語自体の設計が良いという話だったので、これからも勉強しながら気づいたことや感じたこと、学んだことについてブログにまとめていきたいと思います。今回のポストだけでの企画として終わらせたくないので、今年はこれで頑張っていきたいですね。

では、また！
