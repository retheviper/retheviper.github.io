---
title: "JVM言語経験者がGoを触る時のハマりどころ"
date: 2022-04-17
categories: 
  - go
image: "../../images/go.jpg"
tags:
  - java
  - kotlin
  - go
---

JavaとPython、そして少しのJavaScriptの経験してなかった私が、転職先でGoとKotlinを触って1年が経ちました。最近のプログラミング言語は大体収斂進化している傾向があるので、一つの言語ができれば大体他の言語もできる、もしくは読めるようになると言います。

しかし、言語が違うということは、その設計思想が違うということなので、同じ結果を期待して書いたつもりのコードが全く思い通りにならないケースもあります。理由は大きく二つ、「あの言語ではこうだったから、この言語でもそうだろう」という慣性と「これはこの言語の特別な仕様だろう」という思い込みなのではないかと思います。（実際私がそうでしたが）

というわけで、今回はJava/Kotlinをバックグラウンドとして持つエンジニアがGoを触るときの落とし穴的な部分を一部紹介したいと思います。あくまで個人的な経験によるものなのですが、これからGoでコードを書くことになる方には少しでも参考になればと思います。

## time

Goでは時間を扱うためのスタンダードライブラリとして[time](https://pkg.go.dev/time)が存在していて、以下のように使えます。

```go
// 現在時間を取得
now := time.Now()
// 時間を指定して取得(2022-04-01 12:30:00 +0000 UTC)
someDay := time.Date(2022, 4, 1, 12, 30, 0, 0, time.UTC)
```

Java/Kotlinだとこれに対応するAPIとして[java.time](https://docs.oracle.com/javase/jp/8/docs/api/java/time/package-summary.html)がありますね。Goと比べて違う点は、「時間」のみでなく、もっと細かい単位でクラスを分けているというところと言えますでしょう。

```java
// 年度
Year year = Year.now();
// 年月
YearMonth yearMonth = YearMonth.now();
// 年月日
LocalDate date = LocalDate.now();
// 時間
LocalDateTime time = LocalDateTime.now();
```

ただ、違う点はここだけではありません。当然ながら、言語が違うとライブラリの実装も変わってくるものなので、処理の結果も違うケースがありますね。代表的には、「月」単位の時間を扱う時、実装によってはGoでは意図通りの範囲にならないケースがあります。例えば以下のようなコードがあるとします。

```go
func getOneMonthBefore(t *time.Time) time.Time {
    // 月に -1 を指定して返す
    return t.AddDate(0, -1, 0)
}
```

一件なんの問題もなさそうなコードですが、一部のケースで以下のような問題が発生します。

```go
date := time.Date(2022, 3, 31, 0, 0, 0, 0, time.UTC)
oneMonthBefore := getOneMonthBefore(&date) // 2022-03-03
```

上記のコードの結果が「2月28日」でなく、「3月3日」になるのは、処理が以下のように行われるからです。

1. `2022-03-31`から1ヶ月前の`2022-02-31`になる
2. `2022-02-31`という日付は存在しないので、2月の末日から日付の補正を行う
3. 2月の末日である28日から、31日の差分ほど日付をプラスする

というわけで「基準となる月より先月の日付が少ない場合」にこのような結果を得られるわけです。ただ、月末から1ヶ月前というものは、意図としては`2022-02-28`を期待するはずですね。人間の思う処理と、実際のコードが算出する結果が違うというのは十分にあり得る状況ですが、[AddDate](https://pkg.go.dev/time#Time.AddDate)のドキュメントでは上記のような処理になるという話は特に言及されてないので誤解する可能性もあるのかなと思います。

また、Java/Kotlinで使っている`LocalDate`の場合は期待通り`2022-02-28`になるので、Java/Kotlinの経験のあるエンジニアが無意識的にこのような問題を起こすコードを書く可能性もあるかなと思います。ちなみに、`LocalDate`を使ったコードがGoと違う結果になるのは、[minusMonth()](https://docs.oracle.com/javase/jp/8/docs/api/java/time/LocalDate.html#minusMonths-long-)では最後に以下のメソッドを呼び出すからです。

```java
private static LocalDate resolvePreviousValid(int year, int month, int day) {
    switch (month) {
        case 2:
            day = Math.min(day, IsoChronology.INSTANCE.isLeapYear(year) ? 29 : 28);
            break;
        case 4:
        case 6:
        case 9:
        case 11:
            day = Math.min(day, 30);
            break;
    }
    return new LocalDate(year, month, day);
}
```

なので、Goでも3月31日から1ヶ月前の日付が2月28日になるという処理を期待したい場合は、以下の二つの方法を考慮した方がいいかなと思います。

1. LocalDateと同じく、閏年と月別の末尾を考慮した処理を足す
2. `AddDate()`で得られた月が基準となるtimeと同じ月である場合、先月の末日を返す

前者の場合は月を計算した後、上記の`resolvePreviousValid()`と同じ処理を足すことで実現でき、後者の場合は、以下のように末日を取得することが可能なので参考にしてください。

```go
date := date.AddDate(2022, 3, 0, 0, 0, 0, 0, time.UTC) // 3月0日を指定すると2月28日になる
```

## map

Goで変数を宣言する方式は以下の二つがありますね。

```go
// 型だけを宣言
var intSlice []int
// 初期化と共に宣言
stringSlice := make([]string, 10)
```

問題は、宣言の仕方によって要素を足す場合に問題が起こり得るということです。先にsliceの例を見ましょう。`var`で宣言した場合でも`make()`で初期化した場合でも[append](https://pkg.go.dev/builtin#append)を使った要素の追加には問題がありません。

```go
var intSlice []int
// sliceに値を追加する
intSlice = append(intSlice, 1) // [1]
```

ただ、mapの場合は`var`で宣言すると問題が起こる可能性があります。以下のコードは、nil pointerとなります。

```go
var stringMap map[string]string
stringMap["A"] = "a" // panic: assignment to entry in nil map
```

これは`var`で宣言した変数は基本的にnilになるからですね。nilのmapに要素を追加しようとしたのでエラーが発生するのは当たり前ですが、Goland(Intellij)上では警告も表示されず、コンパイルも無事通るので実際にコードを実行するまでこのコードが動くかどうかはわかりません。むしろ、sliceを先に扱っていたなら、nilでもappendできるので「Goではこれでいいのかな」と思い込みやすいかなと思います。

JavaやKotlinでもインスタンスを生成していないMapに対して要素を足すことはできないのですが、ここはJavaやKotlinの週間というよりは「Goの特殊性」と考えてしまうケースだと思いますので、要注意なところなのではないかと思います。

## switch

Goの[switch](https://gobyexample.com/switch)はJavaとよく似ています。ただ、形が似ているだけで、決定的な違いがあります。まずはJavaのswitchから見ていきましょう。以下のようなコードがあるとします。

```java
int i = 1;
switch (i) {
    case 0:
       System.out.println("zero");
    case 1:
       System.out.println("one");
    case 2:
       System.out.println("two");
    default:
       System.out.println("else");
}
```

Javaのswitchは、`break`を明示的に書かない限り、条件が一致するcaseに分岐されたとしても、その下のcaseでも流れてしますね。なので、上記のコードを実行した結果は以下のようになります。

```text
one
two
else
```

Kotlinでは`when`式になり、`break`なしでも条件と一致するコードブロックを実行することで処理は終了します。例えば以下のようなコードを書くとしましょう。

```kotlin
val i = 1
when (i) {
    0 -> println("zero")
    1 -> println("one")
    2 -> println("two")
    else -> println("else")
}
```

実行した結果はJavaと違うのがわかります。省略されているだけで、一つの枝ごとに処理が`break`するからです。

```text
one
```

ここでGoのswitchの場合はどうなるかを見ていきましょう。形上はJavaと似ていますが、結果もそうでしょうか？

```go
i := 1
switch i {
case 0:
    fmt.Println("zero")
case 1:
    fmt.Println("one")
case 2:
    fmt.Println("two")
default:
    fmt.Println("else")
}
```

上記のコードを実行した結果は、Kotlinと同じです。つまり、「one」と出力されるということです。これはGoのswitchもまた、Kotlinと同じく枝ごとに`break`するからです。なので、Javaの場合と同じ結果が欲しい場合は、`fallthrough`を追加し、次の枝に進むということを明示的に書く必要があります。以下のようにです。

```go
i := 1
switch i {
case 0:
    fmt.Println("zero")
    fallthrough
case 1:
    fmt.Println("one")
    fallthrough
case 2:
    fmt.Println("two")
    fallthrough
default:
    fmt.Println("else")
}
```

Javaの経験がある場合、スタイルが似ているのでつい挙動も同じだろうと思って`fallthrough`を省略してしまうというケースもあり得るかなと思います。ここは言語が違うだけ使用も違うということなので、要注意ですね。

## if

Goではif文の条件がおかしいと思われる場合、コンパイルが通りません。例えば以下の例を見てください。

```go
type Role int

const (
    SystemAdmin = 1
    Operator    = 2
    Developer   = 3
)

type User struct {
    Name string
    Role Role
}

// SystemAdminかDeveloperではない場合はエラーを返す
func checkRunnableUser(u User) error {
    if u.Role != SystemAdmin || u.Role != Developer {
        return errors.New("user is not runnable")
    }
    return nil
}

func Test_checkRunnableUser(t *testing.T) {
    u := User{Name: "John", Role: Operator}
    err := checkRunnableUser(u)
    if err != nil {
        t.Errorf("unexpected error: %s", err)
    }
}
```

上記のコードをコンパイルしようとする場合、Goland(Intellij)では条件に警告が表示され、コンパイルすると`suspect or: u.Role != SystemAdmin || u.Role != Developer`というエラーメッセージが表示されるのを確認できます。エラーメッセージでもわかるように、これはif文の条件が間違っているからですね。「UserのRoleがSystemAdminかDeveloperの場合のみ許容する」という要件を満たすためには、`or`ではなく`and`を使う必要があります。なので、if文の条件を以下のように修正すると意図通りに動くし、IDE上の警告やコンパイル時のエラーも発生しなくなります。

```go
// SystemAdminかDeveloperではない場合はエラーを返す
func checkRunnableUser(u User) error {
    if u.Role != SystemAdmin && u.Role != Developer {
        return errors.New("user is not runnable")
    }
    return nil
}
```

Javaの場合だと、Intellijでは条件が怪しいという警告は表示されるものの、Kotlinと同じくコンパイル時のエラーは発生しません。なので実行はできるようになりますが、警告の出ている箇所を確認していないと実際に実行してみるまでロジックが間違えていることには気づかなくなりますね。

```java
enum Role {
    SYSTEM_ADMIN, OPERATOR, DEVELOPER
}

record User(String name, Role role) {}

static void checkRunnableUser(User user) {
    if (user.role() != Role.SYSTEM_ADMIN || user.role() != Role.DEVELOPER) {
        throw new IllegalArgumentException("user is not runnable");
    }
}

public static void main(String[] args) {
    checkRunnableUser(new User("John", Role.SYSTEM_ADMIN));
}
```

しかし、問題は同じ処理をKotlinで書いてみると、Intellijで警告が出ることもなく、コンパイルも通るということです。Javaのケースと同じくランタイムでエラーが発生するコードになりますが、警告すら表示されないのでコードを注意深く確認しないと意図通りに動作している理由が何かを見逃しやすくなっているのではないかと思います。

```kotlin
enum class Role(val value: Int) {
    SystemAdmin(1),
    Operator(2),
    Developer(3)
}

data class User(val name: String, val role: Role)

fun checkRunnableUser(user: User) {
    if (user.role != Role.SystemAdmin || user.role != Role.Developer) {
        throw IllegalAccessException("user is not runnable")
    }
}

fun main() {
    checkRunnableUser(User(name = "John", role = Role.Operator))
}
```

コンパイルタイムでエラーを事前に検知できるという点は確かにGoのコンパイラの方が優秀かなと思います。ただ、KotlinやJavaに慣れている場合、条件がおかしいということに気づくより、「`const`を使っているせいか」「switchを使うべきか」など、問題の本質に気づかないようになる可能性もあるのではないかと思います。

これは、そもそも正しく条件を書くことが何よりも大事であることでありながら、他の言語で形成された習慣で違う言語のコードを書くと問題を起こし得るということを実例として適切ではないかと思いますね。

## 最後に

幾つかの例を挙げましたが、まだ自分もGoでアプリを書いた歴も短く、言語に対しての理解も深くないのでこれからも色々と問題に遭遇する可能性はあるのかなと思います。その度はまたこうやってブログに載せていきたいと思います。ブログのネタができるという面では嬉しいですが、失敗からのポストは結局自分が辛くなることなので、うれしくはないですね…

とにかくここであげた問題は全て自分が経験したものですが、大事なのは、違う言語に挑戦するときは自分の持つバックグラウンドの知識を活かしながらも、それを偏見にしたいこと、そして先走らないことかなと思いました。Goに限らず、新しいものに触れるときは常に注意しないと、という感じですね。

では、また！
