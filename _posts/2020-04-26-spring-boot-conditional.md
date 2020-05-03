---
title: "条件で動作するアノテーションを使う"
date: 2020-04-26
categories: 
  - spring
photos:
- /assets/images/sideimage/spring_logo.jpg
tags:
  - annotation
  - yaml
  - java
  - spring
---

アノテーションは普通のJavaでも使えるもので、様々なライブラリやフレームワークでは積極活用していますね。中でも、最もアノテーションを有効活用しているのはSpringではないかと思います。DIをするためでも、クラスの位置付けにも、なんでもアノテーションをつけることで簡単に定義できるようにしているからです。なのでSpringでWebアプリケーションを実装するときはどんなアノテーションがあるのかを調べてみるのが重要と思います。

なぜこのような話をするかというと、実は[前回のポスト](../../../02/25/spring-service-switch-di)でServiceを切り替える方法について説明しましたが、他に方法がないか探していたところ、Springならアノテーションでも条件によってBeanやConfigurationを登録する方法があるということをわかったからです。

また、その条件によって様々なアノテーションが存在していたため、それらの勉強をかねて、今回は様々な条件と場合を想定して作られたSpringのアノテーションについて紹介したいと思います。これらのアノテーションはSpringのauto-configurationに属するものらしいです。Spring Bootではすでに多くの設定が自動化されていますが、これをカスタムして使えるようにアノテーションを提供しているということですね。

※これらを全部試してみたわけではないですが、とりあえずはご紹介まで。

## 条件でDIする

以前のポストで紹介したコートから始めます。以下のようなことをすれば、application.ymlに記載した値によりどのリポジトリをDIするか決定できるという話をしました。

```java
@Configuration
public class SomeServiceConfig {

    // YAMLに設定した値を読み込む
    @Value("${settings.TestMode:false}")
    private boolean testMode;

    // YAMLの設定からどのImplクラスを使うかを決定してBean登録
    ＠Bean
    public SomeService someService(SomeRepository repository, SomeTestRepository testRepository) {
        return this.testMode ? new SomeTestSerivce(testRepository) : new SomeServiceImple(repository);
    }
}
```

同じようなことを、これから紹介するSpringのアノテーションで実装するとしたら、以下のように変わります。

```java
@Configuration
public class SomeServiceConfig {

    // 本番用のサービスクラス
    @Bean
    @ConditionalOnProperty(value = "settings.Testmode", havingValue = "false")
    public SomeService someService(SomeRepository repository) {
        return new SomeServiceImple(repository);
    }

    // テスト用のサービスクラス
    @Bean
    @ConditionalOnProperty(value = "settings.Testmode", havingValue = "true")
    public SomeService someTestService(SomeTestRepository testRepository) {
        return new SomeTestService();
    }
}
```

以上のコードからわかるように、`@ConditionalOnProperty`というアノテーションを使うと、とある条件によりメソッドの内容が実行されるようにすることができるようになります。わざわざ分岐を書いたり、カスタムな`@Value`アノテーションを用意するよりかなりすっきりしたコードになりますね。

また、このアノテーションはBeanアノテーションとだけ組み合わせができるわけでもないです。他のSpringのクラスアノテーション(Configuration、Component、Service、Repository、Controller)にも使えるので、より自由度が高いですね。

他にもSpring Bootのorg.springframework.boot.autoconfigure.conditionパッケージにはには`@Conditional...`といったアノテーションがいくつか用意されていて、これらを簡単に紹介したいと思います。

## 定義済みのConditionalアノテーション

#### ConditionalOnProperty

application.ymlなど、プロパティを書いたファイルやシステムプロパティに、指定したアノテーションがある場合実行されるアノテーションです。Spring Bootアプリケーションで最も一般的に使われるものらしいですね。括弧の中にはプロパティ名と値、そしてそのプロパティが存在しない場合も実行するかどうかを指定できます。

以下のコードは、use.testmodeというプロパティが存在していて、かつtrueの場合に実行されるConfigurationクラスの例です。matchIfMissingをtrueに指定すると、use.testmodeというプロパティが存在しなくても実行されるようになります。もちろん指定しなかった場合のデフォルト値はfalseとなります。

```java
@Configuration
@ConditionalOnProperty(value = "use.testmode", havingValue = "true", matchIfMissing = true)
public class TestConfig {
  // テストモードで使うConfiguration
}
```

#### ConditionalOnExpression

プロパティの記述方法による実行というアノテーションです。括弧で条件を指定できます。ここでいう条件(表現式)は、Valueアノテーションなどでも使われる[SpEL](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions)を使います。application.ymlなどに記載したプロパティが、括弧の中の条件と一致する場合に実行されますね。以下のコードはuse.testmodeとuse.submodeのどちらもtrueの場合に実行される例です。

```java
@Configuration
@ConditionalOnExpression("${use.testmode:true} and ${use.submode:true}")
public class TestSubConfig {
  // テストモード及びサブモードの両方がtrueの場合に使うConfiguration
}
```

#### ConditionalOnBean

指定したBeanが存在する場合に実行というアノテーションです。括弧の中にBeanとして登録されているかどうか判定したいクラスを指定するだけで使えます。ConditionalOnPropertyでとあるBeanが登録されたら、それに合わせて必要なサブモジュール的なものも登録したい、といった場合に使えるのかなと思います。以下のコードは、TestRepositoryというクラスがBeanとして登録されている場合に実行される例です。

```java
@Configuration
@ConditionalOnBean(TestRepository.class)
public class TestBeanConfig {
  // テスト用のBeanが登録された場合に使うConfiguration
}
```

#### ConditionalOnMissingBean

ConditionalOnBeanとは逆のものです。こちらは、指定したクラスがBeanとして登録されてない場合に実行というものとなります。以下のコードは、RepositoryがBeanとして登録されてない場合は自動的にTestRepositoryをBeanとして登録する例です。

```java
@Configuration
public class AlternativeConfigutration {

    @Bean
    @ConditionalOnMissingBean
    public Repository testRepository() {
        return new TestRepository();
    }
}
```

#### ConditionalOnResource

リソースファイルが存在する場合に実行するというアノテーションです。例えばLogBackを使う場合、xmlファイルが必要となりますが、そのxmlファイルが存在する場合はConfigurationも実行するという構成にしたい場合に使えますね。以下のコードは、そのような場合の例です。

```java
@Configuration
@ConditionalOnResource(resources = "/logback.xml")
public class LogbackConfig {
  // リソースフォルダにlogback.xmlが存在する場合に使うConfiguration
}
```

#### ConditionalOnClass

指定したクラスが存在する場合に実行するというアノテーションです。ConditionalOnBeanと似ていますが、こちらはBeanではなくても良いという違いがありますね。例えば依存関係でとあるライブラリがあるかどうかで使えると思います。以下のコードはcom.custom.library.moduleパッケージのSomeClassというクラスが存在する場合に実行される例です。

```java
@Configuration
@ConditionalOnClass(name = "com.custom.library.module.SomeClass")
public class CustomLibraryConfig {
  // カスタムライブラリのSomeClassがある場合使うConfiguration
}
```

#### ConditionalOnMissingClass

ConditionalOnClassの逆の場合のアノテーションです。こちらはConditionalOnMissingBeanと似ていますね。同じく、指定するクラスはBeanでなくても良いです。以下のコードは、上のConditionalOnClassの逆の場合の例です。

```java
@Configuration
@ConditionalOnMissingClass(name = "com.custom.library.module.SomeClass")
public class NoCustomLibraryConfig {
  // カスタムライブラリのSomeClassがない場合使うConfiguration
}
```


#### ConditionalOnJava

アプリケーションがJavaのどのバージョンで実行されているかのによるアノテーションです。JavaのバージョンによってAPIの仕様が変わる場合があるので、複数の環境でアプリケーションを実行する必要がある場合は使うことを考えられますね。以下のコードは、Javaのバージョンが1.8の場合の例です。

```java
@Configuration
@ConditionalOnJava(JavaVersion.EIGHT)
public class JavaEightConfig {
  // Javaのバージョンが1.8の場合に使うConfiguration
}
```

## カスタムCondition

Conditionインタフェースを実装することで、カスタムConditionを作ることもできます。例えば以下のコードのように、アプリケーションが実行されるOSがLinuxの場合のConditionを自作することができます。

使い方は簡単で、戻り値がbooleanであるmatches

```java
class OnUnixCondition implements Condition {

    // OSがLinuxかどうかを判定するConditionとなる
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return SystemUtils.IS_OS_LINUX;
    }
}
```

実装したConditionは、Conditionalアノテーションに指定して使います。

```java
@Configuration
public class OsConfig {

    // LinuxではBeanが登録される
    @Bean
    @Conditional(OnUnixCondition.class)
        public UnixBean unixBean() {
        return new UnixBean();
    }
}
```

AnyNestConditionクラスを継承すると、より複雑な条件の指定ができます。上で実装したOnUnixCondition以外でも、Windowsで実行されているかどうかを判定するOnWindwosConditionクラスを実装したとしましょう。そういった場合は、以下のように実装することができます。

```java
public class OnWindowsOrUnixCondition extends AnyNestedCondition {

    OnWindowsOrUnixCondition() {
        super(ConfigurationPhase.REGISTER_BEAN);
    }

    @Conditional(OnWindowsCondition.class)
    public static class OnWindows {
        // Windowsの場合
    }

    @Conditional(OnUnixCondition.class)
    public static class OnUnix {
        // Linuxの場合
    }
}
```

実装したConditionは、また同じ方法でConditionalアノテーションに指定します。

```java
@Configuration
public class OsConfig {

    // WindowsかLinuxのどちらだとBeanが登録される
    @Bean
    @Conditional(OnWindowsOrUnixCondition.class)
    public WindowsOrUnixBean windowsOrUnixBean() {
        return new WindowsOrUnixBean();
    }
}
```

## 最後に

こちらで紹介したアノテーション以外でも、org.springframework.boot.autoconfigure.conditionのパッケージの下には様々なクラスが存在しています。例えばWebアプリケーションかどうか、クラウドプラットフォームかどうかのアノテーションが用意されていて、のちにまた様々な条件が追加される可能性もありますね。

これらConditionalアノテーション群は、Spring BootのAuto Configurationでも使われているものらしいです。あので以前私が紹介したように自分でプロパティを直接読み込み、if文を書くよりは安定的な方法であると思います。また、様々な条件に対応するアノテーションがそれぞれ存在していて、カスタムCondtionを作ることで共通化できる部分もあると思うので、いろいろ便利ですね。

Javaそのものもそうですが、Springの世界もまだいろいろと奥深いと感じました。これからも勉強ですね。