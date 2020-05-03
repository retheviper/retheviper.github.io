---
title: "ServiceのImplクラスをYAMLで選択する"
date: 2020-02-25
categories: 
  - spring
photos:
- /assets/images/sideimage/spring_logo.jpg
tags:
  - yaml
  - java
  - spring
---

Springではビジネスロジックを書く場合、一般的にServiceというクラスを作成することになります。Serviceは重要な処理が入るため開発やテストでは重要なクラスですが、開発をしていると、状況によっては実装しても動かせない場合もあります。例えばまだ環境が整っていない、他のクラスに依存する設計となっているがそのクラスがまだ存在していないなどの場合ですね。こういう時は実際の処理が行われず、常に同じ結果を返すクラスを書いておく場合もあります。

こういう場合に、予め複数のServiceクラスを書いておいて、外部の設定ファイル(`application.yml`)に開発モード、テストモードなど状況に合わせてどちらのServiceクラスを使うかを選択できたら便利でしょう。実際、ServiceクラスはInterfaceと実装クラス(Implという名の)を分けて書く場合が多いので、複数のImplクラスを作って置いて、場合によって違うものがBeanとして登録されるようにすることも不可能ではありません。

なので今回は、YAMLの設定を読み、場合によってどのServieImplをBeanとして登録するかを決める方法を紹介します。

## Serviceの元の構成

一般的なServiceクラスの構成は以下のようになります。Interfaceを作成して、それを具現化するサービスを作ることですね。そして中にはDBなどアプリの外部との連携を担当するクラスをDIして使ったりします。

```java
public interface SomeService {

    public void doSomething();
}

@Service
public class SomeServiceImpl implements SomeService {

    private SomeRepository repository;

    @Autowired
    public SomeServiceImpl(SomeRepository repository) {
        this.repository = repository;
    }

    @Override
    public void doSomething() {
        // ...
    }
}
```

ここで、テスト時に使いたいImplクラスを以下のように作成したとします。

```java
@Service
public class SomeTestServiceImpl implements SomeService {

    private SomeTestRepository testRepository;

    @Autowired
    public SomeServiceImpl(SomeTestRepository testRepository) {
        this.testRepository = testRepository;
    }

    @Override
    public void doSomething() {
        // ...
    }
}
```

こういう場合は、アプリケーションを起動するとSpringではInterfaceの実装クラスはどれ？と聞いてくることになります。一つのInterfaceに対して二つの実装クラスが存在していて、両方Beanとして登録しようとしているからです。

## アノテーションを削除

Serviceクラスには一般的に`@Service`をつけることになります。このアノテーションをつけると、Springではこのクラスを自動的にBeanとして登録することになります。なので一つのInterfaceに対して複数の`@Service`のついたクラスを作成すると、どれを使いたいかSpringとしてはわからなくなります。なので、ここでは`@Service`アノテーションは使わないことにします。

## YAMLの作成

YAMLの作成は簡単です。今回はbooleanを使って、trueになっていればテストモード(SomeTestServiceImplを使用)、falseになっていれば通常モード(SomeServiceImplを使用)で動くようにします。例えば以下のようなものです。

```yml
settings:
  TestMode: true
```

application.ymlに直接このカスタムプロパティを書いても良いのですが、自作の設定なので、適当な名前をつけて別途のファイルにしても良いです。別途ファイルにした場合は、application.ymlでそのファイルを含むようにすることを忘れないようにしましょう。

```yml
spring:
  profile:
    include: settings # ファイル名がapplication-settings.ymlの場合
```

## Configuration設定

アノテーションを外したら、ServiceImplはBeanとして登録できなくなります。しかし、使いたいServiceImplクラスを選ぶということは、状況によって使いたいクラスをBeanとして登録したい、ということです。なのでどこかでクラスを選び、Beanとして登録するようにする必要がありますね。また、YAMLに書いた設定を読み込む必要もあります。これらをまとめて`@Configuration`のついたクラスとして実装しましょう。

```java
@Configuration
public class SomeServiceConfig {

    // YAMLに設定した値を読み込む
    @Value("${settings.TestMode:false}")
    private boolean testMode;

    // YAMLの設定からどのImplクラスを使うかを決定してBean登録
    @Bean
    public SomeService someService(SomeRepository repository, SomeTestRepository testRepository) {
        return this.testMode ? new SomeTestSerivce(testRepository) : new SomeServiceImple(repository);
    }
}
```

Springでは`@Value`や`@ConfigurationProperties`を使うことでYAMLに指定した値を読み込むことができます。`@ConfigurationProperties`だとクラス全体のフィールドに対してYAMLの値をマッチすることができますが、ここでは一つの値を読み込みたいだけなので、個別フィールドに対して使える`@Value`を使います。YAMLファイルがない場合は例外となるため、デフォルト値としてfalseを指定しておきました。

Bean登録は普通に`@Bean`アノテーションをつけ、新しいインスタンスを作成して返すだけです。今回の例ではSerivceImplで依存しているRepositoryクラスをコンストラクターにAutowiredを使って注入しているため、そのインスタンスも必要となりますね。メソッドの引数にRepositoryを書いておけば、それがBeanとして登録されているクラスだと自動的に引数として入ってきます。なのであとはこれがテストモードであるか、通常モードであるかによってそれぞれのコンストラクターに合わせた引数を渡し、インスタンスをリターンすればBean登録も完了となります。簡単ですね！

## アノテーションを使う場合

Beanを状況により切り替えたいといった場合に、`@Profile`アノテーションを使う方法もあります。こちらもやり方は難しくありません。まずYAMLファイルを以下のように定義したとします。

```yml
spring:
  profile:
    active: dev
```

YAMLの定義ができたら、あとはどのプロファイルを使うかをアノテーションで指定します。以下のコードのようにです。

```java
@Configuration
public class SomeServiceConfig {

    // devやdebugの場合はこちらをBean登録する
    @Bean
    @Profile({"dev", "debug"})
    public SomeService someService(SomeTestRepository testRepository) {
        return new SomeTestSerivce(testRepository);
    }

    // prodの場合はこちらをBean登録する
    @Bean
    @Profile("prod")
    public SomeService someService(SomeRepository repository) {
        return new SomeServiceImple(repository);
    }
}
```

`@Profile`アノテーションでは指定できるプロファイルを配列で指定できるため、YAMLの記載によってどんなBeanを登録するかを簡単に指定できます。どちらの方法をとっても良いので状況によって適切な方法を選びましょう。

## 最後に

最近はDjangoやExpressなど、他の言語のウェブフレームワークにも触れてみたいと思っていますが、日々新しい発見と勉強が続いているので、なかなかSpringから離れることができません。こうやってできること、知らなかったことを発見するたびに、他にも良いフレームワークがありながらSpringがエンタープライズ市場で長い間生き残ることができたのはこのようにできることが多いからなのではないか、という気もします。Springだけでもしばらくブログに載せる記事のネタは尽きないかもですね！