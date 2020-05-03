---
title: "newしたインスタンスの中でBeanを使いたい"
date: 2019-12-22
categories: 
  - spring
photos:
- /assets/images/sideimage/spring_logo.jpg
tags:
  - dependency injection
  - java
  - spring
---

一般的なJavaプロジェクトなら、外部設定ファイル(YAML)を記載してその値を読み込む場合なら私の[以前のポスト](../../../11/24/java-use-yaml-as-configuration-file)のようにできます。しかし、今回はSpringプロジェクトとして同じようなことをするようになりました。SpringはYAMLを読み込む時に固有の仕様や使い方がありますね。そしてそうやって読み込んだYAMLの値はBeanに設定することができて、アプリケーションの中ではどこでも`@Autowired`を使って呼び出せるというメリットがあります。

しかし、そんな便利なDIですが、使い方の難点もあります。例えば、普通にnewして使うインスタンスのなかで`@Autowired`は使えない問題があるということです。今回もかなりハマっていたことなのですが、Builderでオブジェクトを作成するようにして、使用者が指定してない値はYAMLから取得したBeanを使いたかったです。でもBuilderだと新しいインスタンスを作ってしまうので、Beanを読み込めなくなっていたのでかなりはまりました。

結果的には違う方法をとると、`@Autowired`なしでもBeanを取得することができるということがわかったので、今回のポストではそれに至るまでの過程をコードを持って述べていきたいと思います。YAMLの作成から、newしたインスタンス内でBeanを取得して使う方法を紹介します。

## YAMLからBeanを作る

Springではapplication.ymlに以下のように記載して、特定のYAMLを読み込むという指定ができます。

```yaml
spring:
  profiles:
    active: buildingdefault
```

ここでactiveに記載したものを使って、カスタムYAMLファイルを準備します。ファイル名のプレフィックスとしては`application-`が入ります。なので今回のファイル名は`application-buildingdefault.yml`になりますね。

ファイルを作成して、以下のように項目と値を記載します。

```yaml
settings:
  material: "cement"
```

作成したYAMLファイルはsrc/main/resourceにおきます。そしてこれからはSpringでYAMLを読み込むためのクラスを作成します。

SpringでYAMLを読み込み、Beanを作成する方法は二つがあります。一つ目はまず、フィールドにアノテーションをつけてYAMLの項目と紐づくことです。

```java
@Getter
@Component
public class DefaultSettings {

    @Value("${settings.material}")
    private String material;
}
```

フィールドに`@Value`をつけて、アノテーションの引数としてYAMLの項目名を入力します。こうすることでYAMLから読み込まれた値はString形でBeanに取り込まれます。フィールドは必ずStringである必要はなく、intやdoubleなどのプリミティブ型はもちろん、ENUMにも対応しています。Localeならja_JPなどとYAMLに記載しておくと、ちゃんと取り込まれます。

YAMLの値をBeanにするもう一つの方法は、フィールドではなくクラスにアノテーションをつけることです。以下のように`@ConfigurationProperties`の引数にprefixを指定すると、指定した項目の配下にあるもの全てがフィールドのマッピング対象となります。

```java
@Data
@Configuration
@ConfigurationProperties(prefix = "settings")
public class DefaultSettings {

    private String material;
}
```

## YAMLから複数の設定を読み込みたい時

YAMLから設定値を読み込む際に、設定を複数を記載して状況に合わせて使いたい場合もあります。もちろんYAMLでは配列での記載ができますし、Springで読み込む時もこれをListにすることができます。なのでどうやって複数の設定をBeanにするかを説明します。

YAMLでは以下のように記載します。

```yaml
settings:
  - preset-name: "default"
    material: "cement"
  - preset-name: "cabin"
    material: "wood"
```

ここでpreset-nameは、実際Javaで設定を使う時にそれぞれの設定セットを区別するためのキー的なものです。なくても値を読み込むには問題がないですが、こうやって名前をつけておくとのちにどれがどれかを分かりやすくなりますね。

YAMLの記載が終わったら、それぞれの設定セットに合わせてBeanクラスを作成しておきます。

```java
@Data
public class Material {

    private String presetName;

    private String material;
}
```

最後に、YAMLを読み込むクラスを作成します。このクラスにBeanのListをフィールドとして記載すると、Springアプリケーションの起動と同時にこれらの設定が読み込まれることを確認できます。

```java
@Data
@Configuration
@ConfigurationProperties(prefix = "settings")
public class MultiSettings {

    private List<Material> presets;
}
```

のちにこのクラスからListを取得して、presetNameで各設定値を探すだけで簡単に使えるようになります。

## BuilderからBeanを使う(失敗の例)

今までの設定で、普通のSpringアプリケーション内ではBeanをDIして使うことができるようになります。しかし、今回はDIなしてBeanを取得する方法を説明するためのポストになっていますので、その過程を説明します。

まず自分がやりたかったことは、先に述べましたが、Builderの中でYAMLの値を読み込んでいるBeanを使うことでした。ここでYAMLに記載した値はデフォルト値として使われて、必要に応じて一部の項目だけbuild()時に上書きしたいです。まず試して、ダメだったコードは以下のようなものです。

```java
public class Building {

    public static BuildingBuilder() {
        return new BuildingBuilder();
    }

    public static class BuildingBuilder {

        // DIができない
        @Autowired
        private DefaultSettings settings;

        private String material;

        public BuildingBuilder() {
            this.material = this.settings.getMaterial();
        }

        ...
    }
}
```

Builderを使うと、まずBuilderのインスタンスを新しく生成するしかないです。そしてnewしたインスタンスの中では`@Autowired`で記載していても、DIがまともにできません。実際上のようなコードを書くと、BeanのフィールドがNullになっていることを確認できます。

なのでDIのことは忘れて、newしたインスタンスの中でBeanを取得できる方法をとります。

## ApplicationContextProviderを作る

ApplicationContextは、SpringでBeanの生成やオブジェクト間の関係設定など様々な機能を担当するインタフェースです。ここで重要なのは、ApplicationContextがSpringアプリケーションを起動する時予め登録されたBeanを生成して管理するということです。つまり、このインタフェースにアクセスできればBeanを取得できるということになります。

ただ、ApplicationContextそのものはあくまでインタフェースであるため、インスタンスを取得するためにはその役割をするクラスを作成する必要があります。以下のコードでインスタンスを取れるようになります。

```java
@Component
public class ApplicationContextProvider implements ApplicationContextAware {
     
    private static ApplicationContext context = null;
 
    public static ApplicationContext getApplicationContext() {
        return this.context;
    }
 
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        this.context = context;
    }
}
```

構造は簡単で、フィールドにApplicationContextがあって、それに対するGetterとSetterがあるだけです。これで動くのも不思議ですが、Springアプリケーションが動作すると自動的にApplicationContextのインスタンスがSetterを通じてフィールドにセットされます。ただ、このクラスのインスタンをnewしては使えなくなるのでフィールドとGetterはstaticにしておきます。

## BuilderからBeanを使う(成功の例)

それでは、ApplicationContextのインスタンスを取得できるようになりましたので、Builderを修正します。

```java
public class Building {

    public static BuildingBuilder() {
        return new BuildingBuilder();
    }

    public static class BuildingBuilder {

        private DefaultSettings settings;

        public BuildingBuilder() {
            this.settings = ApplicationContextProvider.getApplicationContext().getBean(DefaultSettings.class);
        }

        ...
    }
}
```

さっき作成したApplicationContextProviderクラスからApplicationContextを取得して、さらにgetBean()を呼び出します。このgetBean()に引数として取得したいBeanのクラスを渡すと、そのBeanのインスタンスを取得することができます。もちろんコンストラクターではなく、フィールドそのものに書くこともできます。そうする場合は以下のようになりますね。

```java
public class Building {

    public static BuildingBuilder() {
        return new BuildingBuilder();
    }

    public static class BuildingBuilder {

        private DefaultSettings settings = ApplicationContextProvider.getApplicationContext().getBean(DefaultSettings.class);

        public BuildingBuilder() {
        }

        ...
    }
}
```

修正したコードを動かしてみると、BeanのフィールドがNullではなくちゃんとYAMLから読み込んだ値が入っていることを確認できます。

## 最後に

Springを使いながら、恥ずかしくも実際アプリケーションの内部ではどんなことが起きているかを知らなかったので今回は失敗したのではないかと思います。ただ単に動くことを確認するだけでなく、こうして自分の使っている言語やフレームワークの特性をちゃんと理解していないとこのようにハマることはなかったでしょう。なので新しい知識を得た同時に、自分に対する反省もすることになりました。これからはちゃんと自分が使っているものはどう、なぜ動くのかをちゃんと理解してから使わないとですね。