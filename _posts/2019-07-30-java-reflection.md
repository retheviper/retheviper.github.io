---
title: "ReflectionとGenericを活用する"
date: 2019-07-30
categories: 
  - java
photos:
  - /assets/images/sideimage/java_logo.jpg
tags:
  - reflection
  - generic
  - java
---

今回の仕事で学んだことは、自分のコードを他人がライブラリーとして使うときはどのように実装していくかの方法です。とある機能をするスクリプトを書くこと、エンドユーザーが使うUIやそコンテンツを処理するロジックなどだけに関わったことのない自分にとってはとても新しい経験になりました。今までだと自分が担当した機能を具現し、それを最適化していいだけでした。でも、ライブラリーは基本的にコードを扱える人が使うものなので設計が全く違いますね。

そして、難しかった部分の一つは、柔軟性を持たせることでした。例えばデータを受け取り、処理していく中で、今までは自分が実装したBeanにデータがマッピングされていることを第一の前提条件としていました。それが今の仕事では、「どんなBeanが入ってくるかわからないから、それに対応する」ようにする必要がありました。

まずクラスやインスタンスを引数として受け取る方法を知る必要がありますね。最初はObjectそのものを使おうと思いましたが、調べてみると`Generic`というものがあったので、そちらを使うことにしました。

次にそのGenericを使って、引数として受け取ったBeanです。自分が設計したBeanだけを使うなら、Beanが持つフィールドのデータ型も知っていいて、Getter/Setterからデータのやりとりができますね。しかし自分が作ったものではないと、フィールドのデータ型もGetter/Setterメソッドもどう呼ぶかもわからなくなります。ここでどう対応したらいいだろうか…と悩み、探し出した答えが`Reflection`でした。

今回のポストはその二つを使い、どう「自分が作っていないBeanを処理」したかについて述べ地と思います。

## Generic

Genericとは、「データのタイプを一般化」するということです。総称型とも呼ぶらしいです。クラスの中で使うタイプを、クラスの外部から設定するとき用いられると言いますね。今までは主に複数のデータ型やオブジェクトを処理したい場合は`Object`を使ってきました。Javaでは一部のPrimitive(参照型)を覗くと、ほとんどのデータががオブジェクトとして扱われているので、もっとも上位のタイプであるObjectに変えられらからです。

しかし、Objectを引数として使うと、メソッドの中のフィールドが何があるかわからなくなります。また、Getter/Setterによる値の指定・取得などの操作もできませんね。なのでオブジェクトそのものを受け取るだけでなく、インスタンスの中を直接覗く必要があります。

ここでGenericを使うと、Objectと同じ機能(どんなインスタンスやクラスでも受け取れる)をしながらも、最初に具体的なタイプが決まるのでキャストがいらなくなります。これなら性能的にもよく、より安全な挙動を期待できますね。あとは実際どんな構造をしているかわからないObjectに対し、Reflectionを活用して、クラスやインスタンスの中を覗いたらメソッドもフィールドも取得できるというメリットもあります。

まずインスタンスを引数として受け取る方法から見てみましょう。引数として`T`[^1]を指定します。これでGenericタイプの引数を受け入れられます。つまり、インスタンス自体が引数となるということです。ただ、引数が`T`の場合はメソッドの戻り値の前でも`<T>`を宣言する必要があります。コードで表現すると以下のようになります。

```java
// Beanのインスタンスを受け取るメソッド
public <T> boolean isBean (T parameter) {
    // ... 何かの処理
}

// 使用例
BeanObject beanObject = new BeanObject();

if (isBean(beanObject)) {
    // ... 何かの処理
}
```

Listの中にもGenericを使うことができます。たとえば以下のように書くと、どんなタイプも受け入れられるようになりますね。

```java
List<T> list = new ArrayList<>();
```

インスタンスではなく、クラスそのものをGenericで受け入れるには以下のように書きます。

```java
// Beanのクラスを受け取るメソッド
public boolean isBean (Class<?> parameter) {
    // ... 何かの処理
}

// 継承もできる(限定)
public boolean isStringBean (Class<? extends String> parameter) {
    // ... 何かの処理
}

// 使用例
if (isBean(BeanObject.class)) {
    // ... 何かの処理
}
```

こうやってGenericの引数を渡してもらう準備は完了です。次は、そうやって受け取った引数を扱うReflectionについて調べてみましょう。

## Reflection

クラスの具体的なタイプをわからなくても、メソッド・コンストラクター・フィールドなどににアクセスできるようにしてくれるAPIをReflectionと言います。取得したメソッド・コンストラクター・フィールドはいずれもそのまま使ったり、値(戻り値)を取得したり、つけられているアノテーションを取得するなど、普通のクラスでできることは全部できます。

では、このReflectionを実際どう使うかを以下のコードで紹介します。まずはインスタンスからクラスを取得する方法と、さらに取得したクラスからコンストラクターを取得するコードです。

```java
public <T> boolean isBean (T object) {

    // インスタンスからクラスを取得
    Class<?> objectClass = object.getClass();
    // クラスからさらにインスタンスを生成する
    Object instance = objectClass.newInstance();

    // クラスのパッケージ名を取得
    String package = objectClass.getPackage();
    // パッケージを含むクラス名を取得
    Stirng classNamePackageInvolved = objectClass.getName();
    // クラス名だけを取得
    String className = objectClass.getSimpleName();   
    
    // 配列でpublicコンストラクターたちをを取得
    Constructor[] constructors = objectClass.getConstructors();
    // 特定のpublicコンストラクターを取得
    Constructor constructor = objectClass.getConstructor(parameter1, parameter2, ...);
    // 取得したコンストラクターからインスタンスを生成
    Object instance2 = constructor.newInstance();

}
```

クラス自体を扱うことができるので、その中身がわかれば新しいインスタンスを生成して使うこともできますね。それでは次に、フィールドを取得する方法を紹介します。

## Fieldの取得

```java
public boolean isBean (T object) {

    Class<?> objectClass = object.getClass();

    // 配列でpublicフィールドたちを取得
    Field[] fields = objectClass.getFields();
    // 特定のpublicフィールドを取得
    Field field = objectClass.getField("フィールド名");
    // 配列で全フィールドたちを取得(public以外も)
    Field[] declaredFields = objectClass.getDeclardFields();
    // 特定のフィールドを取得(public以外も)
    Field declaredFiled = objectClass.getDeclaredField("フィールド名");

    // Fieldでできること
    // フィールドに値を設定
    field.set(object, parameter);
    // フィールドの値を取得
    String fieldValue = field.get(object);
    // フィールドのアノテーションを取得
    Annotation[] annotations = field.getAnnotations();
}
```

ここで注意すべき部分は、フィールドを取得するときはクラスからであって、実際値を代入したり取得するときはその対象としてインスタンスを使うということです。クラスが設計図であり、インスタンスがその設計図で生成されたものだということが明確になる瞬間ですね。Reflectionを使うことのメリットはここにもあるのかも知れません。

次には、メソッドをみていきます。

## Methodの取得

メソッドもフィールドの場合とそう変わりません。クラスからメソッドを取得し、そのメソッドからさらに色々できるようになります。またコードを以下に用意しました。

```java
public boolean isBean (T objectClass) {

    Class<?> objectClass = object.getClass();

    // 配列でpublicメソッドたちを取得
    Method[] methods = objectClass.getMethods();
    // 特定のpublicメソッドを取得 
    Method method = objectClass.getMethod("メソッド名", parameter1, parameter2, ...);
    // 配列でメソッドたちを取得(public以外も)
    Method[] declaredMethods = objectClass.getDeclaredMethods();
    // 特定のメソッドを取得(public以外も)
    Method declaredMethod = objectClass.getDeclaredMethod("メソッド名", parameter1, parameter2, ...);
    
    // Methodでできること
    // メソッド名の取得
    String methodName = method.getName();
    // メソッドの実行
    Object methodInvoked = method.invoke();
    // 引数のアノテーションを取得
    Annotation[] parameterAnnotations = method.getParameterAnnotations();
    // 引数を取得
    Parameter[] parameters = method.getParameters();
}
```

アノテーションの場合はクラスでもフィールドでも取得できますが、メソッドの場合はそれに加えて引数のアノテーションも取得できるということが特徴です。また、引数そのものを取得することもできますね。もちろん`Parameter`クラスでも引数名を取得するなど色々な操作ができます。

## 結論

結局、道が見えてくると、解決の方法も見えてくるものです。メソッドをやコンストラクターに触れる必要もなく、引数で受け取ったインスタンスからフィールドを取得して、その値をObjectに代入する。そしてinstanceofを活用して分岐させ、キャストすることでデータを扱う、というシンプルな構造で自分のタスクは完成されました。簡単なコードで表現すると以下のような形ですね。

```java
// クラスをわからないBeanを引数とするメソッド
public <T> processSomething(T bean) {
    // 複数のタイプのオブジェクトがある
    String stringObject;
    Integer intObject;

    // クラスとフィールドの取得
    Class<?> beanClass = bean.getClass();
    Field[] beanFields = beanClass.getDeclardFields();

    // ループで個別フィールドを取得し、一致するタイプのオブジェクトにフィールドの値を入れる
    for (Field field : beanFields) {
        // privateフィールドの場合はアクセスできない場合があるためアクセス可能にする
        if (field.canAccess(bean)) {
            field.setAccessible(true);
        }

        // フィールドの値を取得し、型を判断
        Object value = field.get(bean);
        if (value instanceof String) {
            stringObject = (String)value;
        } else if (value instanceof Integer) {
            intObject = (int)value();
        }
    }
}
```

どうですか。方法がわかれば簡単にできるものなのでは。さらに応用して、インスタンスの一部の値だけを修正して返すなどの挙動もできそうです。色々活用できそうな面が多いですね。

## 最後に

このようにだいたいフィールドとメソッドがわかれば、これでどんなインスタンスやクラスが入ってきても、対応できそうな気がします。実際どうかはコンパイルして実行してみない限りわからないものですが…でも一つ、賢くなったような気はします。皆さんもぜひ、Reflectionを通じてクラスとインスタンスに対する理解を深めてみてください。

それでは、また！

[^1]: Typeのこと。ただ、E(Element)やK(Key)、N(Number)、V(Value)も使えるらしいです。
