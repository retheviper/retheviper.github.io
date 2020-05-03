---
title: "Nullチェックの地獄から脱出したい"
date: 2020-01-05
categories: 
  - java
photos:
- /assets/images/sideimage/java_logo.jpg
tags:
  - optional
  - java
---

Javaでアプリケーションを組みながら最も遭遇率の高い例外が何かとしたら、それはNullPointerException(NPE)でしょう。初めてプログラミングを接する人に取っては「空白」と「Null」がどう違うのかを理解することもかなり難しいことではないのかと思いますが、Nullを理解できたとしても予想してなかったところで出てくるNPEで苦労する場合は決して少なくないと思います。ある意味、Javaでのアプリケーション開発はNPEとの戦いであるといっても過言ではないのではないでしょうか。

なので今回は、NPEに対処するための方法を紹介します。Nullをチェックし、安全なコードを書く方法を探してみましょう。

## Nullチェックで十分か？

とある学生の名前を取得するメソッドがあるとしましょう。データオブジェクトを引数として渡すと、そこから順番に学校、学年、組、学生の情報を取得して最後に学生の名前をStringとして返却するようなものです。これをコードで表現したら、例えば以下のように表現できます。

```java
public String getStudentName(Data data) {
    School school = data.getSchool();
    Grade grade = school.getGrade();
    ClassName className = grade.getClassName();
    Student student = className.getStudent();
    return student.getName();
}
```

これをより簡潔なコードで表現するとしたら、以下のようになるでしょう。

```java
public String getStudentName(Data data) {
    return data.getSchool().getGrade().getClassName().getStudent().getName();
}
```

このメソッドが意図通りに作動するとしたら、シグネチャーとコードだけで意図と結果が明確なものとなるはずです。しかし、皆さんにもわかるように、このコードにはいつどこでも例外が発生する可能性があります。

学生の名前フィールドがNullだとしたら？いや、そもそも学生が、もしくは組が、学年が、学校がNullだったら？引数がNullだとしたら？どれもNPEになりうる可能性があるので、極めて危険なコードとなっています。

ここでまず考えられる対策は、事前にNullチェックの処理を入れNullでない場合にだけ次の処理に移行するようなコードを書くことでしょう。そしてNullだった場合にまた適切な処理(もしくはデフォルト値)を書くことで意図した通りに動かすことができます。

では、上のコードにNullチェックの処理を入れ、をNull Safeなコードに変えてみましょう。例えば以下のように変えることができるでしょう。

```java
public String getStudentName(Data data) {
    if (data != null) {
        School school = data.getSchool();
        if (school != null) {
            Grade grade = school.getGrade();
            if (grade != null) {
                ClassRoom classRoom = grade.getClassRoom();
                if (classRoom != null) {
                    Student student = classRoom.getStudent();
                    if (student != null) {
                        String studentName = student.getName();
                        if (studentName != null) {
                            return studentName;
                        }
                    }
                }
            }
        }
    }
    return "Sato"; // Default value
}
```

以上のコードはネストしすぎて、極めて読みづらいコードとなっています。なのでもし一つの項目でもNullチェックが抜けているとしてもわからなくなります。また、コードを直すこともかなり困難になります。メソッドの目的はあくまで、「学生の名前が知りたい」というシンプルな要求に応えるためのものだったのですが、もはやNullチェックが入りすぎてなんのためのロジックなのかわかりづらいですね。

ネストしている処理を避けるためif文をバラバラにしても結果はあまり変わりません。以下のコードをご覧ください。

```java
public String getStudentName(Data data) {
    if (data == null) {
        return "Sato"; // Default value
    }
    
    School school = data.getSchool();
    if (school == null) {
        return "Sato"; // Default value
    }

    Grade grade = school.getGrade();
    if (grade == null) {
        return "Sato"; // Default value
    }
    
    ClassRoom classRoom = grade.getClassRoom();
    if (classRoom == null) {
        return "Sato"; // Default value
    
    Student student = classRoom.getStudent();
    if (student == null) {
        return "Sato"; // Default value
    }

    String studentName = student.getName();
    if (studentName == null) {
        return "Sato"; // Default value
    }

    return studentName;
}
```

if文のネストを無くして読みやすくしてみようとしました。でも、このやり方だとむしろreturnが多すぎてこれはこれであまりよくない処理になっています。

このようなコードはどう直したらいいのか？という時に、使えるAPIをJavaでは用意しています。今回の主題であるOptionalです。

## Optionalを導入する

現代の言語はこのNullによって起こり得る問題を最初からブロックするため最初からNullを代入することを許さなかったり、Nullになりえるオブジェクトを扱えるAPIを提供したりするようです。例えばKotlinやSwiftではNullableやOptionalを制約の一つとして使っているようで(触ってみたことがないのでこうとしかいえませんが)、Pythonの場合もUnionやOptionaと言ったAPIが用意されているようです。そしてJavaもそう言ったトレンドに答えるべく、Java 1.8でOptionalをAPIとして導入しています。

Optionalは、Nullになる可能性のあるオブジェクトに対しての新しい(といってもJava 1.8で導入されたのでもうそんなに新くもないですが)方法です。基本的には関数型言語から影響を受けて作られているらしいですね。

私自身は関数型言語に詳しくないのですが、確かにこのOptionalの使い方をみるとLambda同様、元のJavaの思想とはかなり違うもののような気がします。なぜなら、オブジェクトのNullチェックを比較演算してその後の処理を決めるわけではなく、メソッドの連鎖で決めていくような形になっていて、書き方がかなり異質的だからです。

なら、そんな異質的なAPIをなぜ使うのか？それはOptionalがどんなものであり、どんな特徴を持っているかをまず見て判断することにしましょう。

#### 使い方が簡単

map()やfilter()などCollection[^1]やStream[^2]と似たような機能をするメソッドがあり、さらに引数としてLambdaを使えるので、CollectionやStreamに慣れていると簡単に適応できます。

Optionalを効率的に使うためにはメソッドチェーニング[^3]やLambdaにまずなれる必要があるので、まずはjava.util.functionsになれるとしましょう。[以前のポスト](../../../../2019/08/06/java-function)を参考にしてください。

#### 見ただけでわかる

Optionalはオブジェクトを包み、そのオブジェクトがNullである場合の処理のため作られたAPIです。なのでOptionalで包まれているオブジェクトがあると、そのオブジェクトはNullになる可能性があることを明らかにしているということです。なので戻り値だけでNullになる可能性があるコードを見分けることができるようになります。

#### 可読性が上がる

Nullチェックという本来の目的に充実しながらも、コードが簡潔になるので読みやすいコードになります。取得したいオブジェクトがネストしている場合もOptionalで対応できます。最初のオブジェクトのNullチェックをして、さらにネストしているオブジェクトをNullチェックしていくような形です。

## OptionalでNullチェックを変えてみましょう

では、実際のコードを持ってOptionalでのNullチェックがどう可能になるのかをコードを持ってみてみましょう。さっきのメソッドは以下のように変えることができます。

```java
public String getStudentName(Data data) {
    return Optional.ofNullable(data)
                    .map(Data::getSchool)
                    .map(School::getGrade)
                    .map(Grade::getClassRoom)
                    .map(Class::getStudent)
                    .map(Student::getName)
                    .orElse("Sato");
}
```

Optionalが初めての方にはどんなことをしているか一見わからなくなるのではと思いますが、それでもコードの量が減り、可読性がよくなったのはわかるでしょう。もちろん、Nullチェックが省略されているわけでもありません。このように簡潔で分かり安く、安全なNullチェックを可能にするのがOptionalです。

## Optionalのメソッド

OptionalはSingletonのjava.util.Optional<T>をインポートしてオブジェクトを包み、包まれたオブジェクトがNullか否かによってどんな挙動をするかのメソッドを持っています。これからそれらのメソッドを一つづつ見ていきましょう。

#### empty()

空のOptionalを作成します。空のOptionalはその名の通り空で、中にラップされたオブジェクトがNullの状態です。

```java
Optional<String> emtpyName = Optional.empty(); // StringはNull
```

#### get()

Optionalでラップされたオブジェクトを取得する時に使います。

```java
String value = "Sato";
Optinal<String> name = Optional.of(value);

System.out.println(name.get()); // Sato
```

#### of(T value)

引数として渡したオブジェクトを包むOptionalを生成します。ただ、引数のオブジェクトがNullの場合はget()の結果もNullになります。

```java
String value = null;
Optinal<String> name = Optional.of(value);

name.get(); // Null
```

#### ofNullable(T value)

引数として渡したオブジェクトを包むOptionalを生成するということではof()と同じですが、引数のオブジェクトがNullだった場合はempty()で生成されたOptionalを返却します。

```java
String value = null;
Optinal<String> name = Optional.ofNullable(value);

name.get(); // Optional<String>
```

#### map(Function<? super T, ? extends U> mapper)

CollectionやStreamのmap()と似たようなメソッドです。複雑にネストされているフィールドを安全にチェックする時に使います。mapで取り出したオブジェクトは自動的にOptionalでラップされたクラスとなります。

```java
String name = "Sato";
Student sato = new Student(name);

Optional<Student> student = Optional.ofNullable(sato);
String nameOfSato = student.map(Student::getName).get(); // Optional<Student> -> Optional<String>
```

ここで使われている::での表現式はMethod Referenceといい、ターゲットレファレンスとメソッドを書くだけで一般的なLambdaと同じ効果を期待できる書き方です。Lambdaで既存のコードをより簡潔に書くことができるようになりましたが、さらに引数の変数名を省略できるようにしたものですね。変数名を書かなくても指している対象が明確でメソッドも一つだけを呼ぶ場合に使います。

```java
// 引数を標準出力するLambdaの一般的な書き方
Consumer<String> print = name -> System.out.println(name);

// Method Referenceに変えた形
Consumer<String> print = System.out::print;

// インスタンスの生成
Supplier<String> = String::new;
```

#### filter(Predicate<? super T> predicate)

filter()もまたCollectionやStreamのメソッドに慣れているなら簡単に使えるメソッドの一つです。条件と一致する場合(PredicateによりTrueとなる)にだけ値を返却します。単にNullかどうかの判定だけでなく、何かの処理を付け加えたい時に使います。

```java
// 伝統的なパターン
public String getSato(Student student) {
    String name = student.getName();
    if (name != null && name.equals("Sato")) {
        return name;
    }
}

// filter
public String getSato(Student student) {
    return Optional.ofNullable(student)
            .filter(s -> s.getName().equals("Sato"))
            .map(Student::getName)
            .get();
}
```

Optionalの要素は一つしかないのでfilterで指定した条件の結果がfalseの時は以後のメソッドが無視されます。

#### isPresent()

OptionalでラップしたクラスがNullであるかを判定するためのメソッド。Nullでない場合はTrue、Nullの場合はFalseとなるシンプルなものです。

```java
String name = "Sato";
Optional<String> studentName = Optional.ofNullable(name);
studentName.isPresent(); // true
```

#### ifPresent(Consumer<? super T> consumer)

ラップされたオブジェクトがNullでない場合にだけ実行するメソッドを記述します。

```java
Optional<String> name = Optional.ofNullable(student.getName());
name.ifPresent(n -> System.out.println(n));
```

#### orElse(T other)

引数として渡したオブジェクトがNullの場合にデフォルト値を使います。このメソッドを使った場合はget()は記述しなくてもよくなります。

```java
String defaultName = "Sato";
Optional<String> name = Optional.ofNullable(student.getName());
String result = name.orElse(defaultName); // student.getName()がNullの場合defaultNameになる
```

#### orElseGet(Supplier<? extends T> other)

引数として渡したオブジェクトがNullの場合にデフォルト値として指定したLambdaを実行し、その結果を返却します。このメソッドを使った場合はget()は記述しなくてもよくなります。

```java
Optional<String> name = Optional.ofNullable(student.getName());
String result = name.orElseGet(() -> student.getNumber + "の名前がありません"); // student.getName()がNullの場合Lambdaを実行する
```

#### orElseThrow(Supplier<? extends X> exceptionSupplier)

引数として渡したオブジェクトがNullの場合に例外を投げます。このメソッドを使った場合はget()は記述しなくてもよくなります。

```java
Optional<String> name = Optional.ofNullable(student.getName());
String result = name.orElseThrow((BusinessException::new);
```

## Optionalで注意すべきこと

Nullチェックで便利で安全なOptionalですが、全ての状況でNullに関する処理を全部Optionalに変える必要はありません。Optionalの導入を検討する時、注意すべきことについて説明します。

#### 性能を意識する

すでに気づいている方もいらっしゃると思いますが、Optionalはオブジェクトをラップするものなので必然的に性能の低下と繋がります。なのでNullチェックがいる場面では一旦Optionalを使う、ということはあまり良い考えではありません。簡単なNullチェックはOptionalでなくてもできますし、早いです。

Optionalを使ってオブジェクトがNullの場合の処理を書く際もorElse()よりはorElseGet()を使った方が良いです。orElse()はNullではない場合も必ず実行されるからです。それに対してorElseGet()の場合はLazy[^4]なメソッドなのでより良い性能を期待できます。

ただ、場合によっては(staticなデフォルト値をフィールドとして持っているなど)、orElse()の方を使った方が良いケースもあるのでその場の判断が重要です。返却したいデフォルト値のインスタンスがどこで作成されるかの時点をよく把握しましょう。

```java
// よくない例(Nullではない場合捨てられるインスタンス)
public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return Optional.ofNullable(student).orElse(Student::new);
}

// 良い例
public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return Optional.ofNullable(student).orElseGet(Student::new);
}
```

また、戻り値としてNullもしくは決まったデフォルト値を期待する場合はOptionalよりもNullチェックの方が良い場合もあります。

```java
// よくない例(常に同じデフォルト値が決まっている場合)
private static Student defaultStudent;

public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return Optional.ofNullable(student).orElse(defaultStudent);
}

// 良い例
public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return student != null ? student : defaultStudent;
}
```

#### isPresent()とget()の組み合わせはNG

isPresent()でオブジェクトがNullかを確認したあと、get()でオブジェクトを取得するようなコードは結局普通のNullチェックと変わりません。デフォルト値を使いたい場合はorElseGet()を、例外としたい場合はorElseThrow()を活用しましょう。

```java
// よくない例
public String getStudent(String name) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));

    if (student.isPresent()) { // (value != null)の方が良い
        return student.get();
    } else {
        throw new NullPointerException();
    }
}

// 良い例
public String getStudent(String name) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));
    return student.orElseThrow(NullPointerException::new);
}
```

また、オブジェクトがNullでない場合にだけ処理を行いたい場合なら、ifPresent()を使いましょう。

```java
// よくない例
public void adjustScore(String name, int score) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));
    
    if (student.isPresent()) {
        student.get().setScore(score);
    }
}

// 良い例
public void adjustScore(String name, int score) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));
    student.ifPresent(s -> s.setScore(score));
}
```

#### フィールドでは使わない

そもそもOptionalはフィールドとして使われる場合を想定していないようです。なぜなら、OptionalはSerializableを継承してないからです。なのでDTOなどでフィールドとしてOptionalを使うとNullチェック以前に問題が起こる可能性があります。

```java
// よくない例
@Data
public class Student implements Serializable{

    private Optional<String> name; // 直列化できない
}

// 良い例
@Data
public class Student implements Serializable{

    private String name; // 直列化できる
}
```

#### 引数では使わない

メソッドやコンストラクターの引数としてOptionalを使うと、それを呼び出すたびに引数としてOptionalを生成する必要があります。また、内部的にOptionalでNullチェックのロジックが入るのでコードも複雑になりますね。こういう場合、内部でどんな処理が行われ、期待通りの処理になっているかわからなくなるので不便です。

なのでメソッドやコンストラクターの引数は普通のオブジェクトにして、Nullチェックをした方が使いやすく意図した処理を期待できるようになります。

```java
// よくない例
public class Student {

    private String name;

    public Student(Optional<String> name) { // インスタンスを作成するたびOptionalも必要となる
        this.name = name.orElseThrow(NullPointerException::new); // OptionalでNullチェックおよび代入が必要
    }
}

// 良い例
public class Student {

    private String name;

    public Student(String name) {
        this.name = name; // 期待通りの処理
    }
}
```

#### Collectionの中では使わない

Collectionの中の要素は無理やり入れない限りNullが入らない場合もあれば、Nullチェックに対応するメソッドを含めている場合もあります。そして中の要素は複数になるので、Optionalを要素として使う場合は性能の低下が必然的に起こります。なので要素ではなるべくOptionalを使わないようにしましょう。また、フィールドや引数と同じく要素を追加したり取得する場合に毎回Optionalを経由しなければならないという不便さがあります。

```java
// よくない例
List<Optional<String>> names = new ArrayList<>();
names.add(Optional.ofNullable(name1)); // 要素を追加するたびラップが必要

// 良い例
List<String> names = new ArrayList<>();
names.add(name1);
```

#### CollectionはCollectionで

Collectionが戻り値のメソッドの場合、NullだとCollections.emptyList()やCollections.emptyMap()などで空のCollectionを返却した方が良い場合が多いです。Collectionは

また、Spring Data JPAを使っている場合はそもそも戻り値がNullだと、自動的に空のListを生成してくれるので尚更Optionalを使う必要がありません。

```java
// よくない例
public Optional<List<Student>> listStudent() {
    List<Student> students = this.repository.listStudent();
    return Optional.ofNullable(students);
}

// 良い例
public List<Student> listStudent() {
    List<Student> students = this.repository.listStudent();
    return students != null ? students : Collections.emptyList();
}
```

### int/long/doubleはOptionalでラップしない

Optionalのバリエーションでは、一部プリミティブ型のためのクラスも用意されています。int/long/doubleの場合がそうです。これらはOptionalInt、OptionalLong、OptionalDoubleで包む方が良いです。

```java
// よくない例
Optional<Integer> count = Optional.of(100);
int countNum = count.get();

// 良い例
OptionalInt count = OptionalInt.of(100);
int countNum = count.getAsInt();
```

## 最後に

今は使われているJavaのバージョンが古くても、公式サポートなどの理由でJava 1.8以上にバージョンアップするところも多いと聞きます。ならばJavaプログラマーとして、未来に備えJava 1.8の重要なAPIに慣れて置いた方が良いでしょう。そういう意味でFunctionやOptionalは皆さんにもぜひ使ってみて欲しいAPIでもあります。そもそもJavaがこんなにメジャーな言語になり得たのは、開発しやすいというメリットがあったからなので、さらに開発が楽になるAPIは覚えておいて損はないでしょう。

Javaもかなり古い言語ですが、最近は急激なバージョンアップと共に関数型言語など最近のトレンドを反映して変化しているところもあります。今は性能も書きやすさも優秀な言語が溢れ出している時代ですが、こんなJavaの変化がどこまで続き、いつまで生き残ることができるか気になります。JVMは依然として強力ですが、LLVMなどより性能が優れた技術も続々と登場していますしね。でも、Javaの変化に適応し、大体のAPIを使うことができたら、他の言語にも適応しやすくなるのではと思います。そういう理由ででも、みなさん、Java 1.8以後のAPIは注目してください。では！

[^1]: List, Set, Mapなど複数の要素を持つオブジェクトのことを指します。
[^2]: ファイルの入出力で使われるInputStreamやOutputStreamではなく、Collectionの要素を一つずつ巡回しながら特定のメソッド(主にLambda)を実行できるようにしてくれるJava 1.8のAPIです。
[^3]: 戻り値が自分自身のため、何度もメソッドをつなげて書くことのできる仕組み。Builderパターンが代表的なメソッドチェーニングの例です。
[^4]: プログラミングでLazyということは、とある処理が常にではなく、呼ばれた際に初めて実行される仕組みのことを意味します。必要な時だけ処理が始まるので不要な処理が減ります。