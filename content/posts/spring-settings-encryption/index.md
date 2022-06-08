---
title: "Jasyptでプロパティを暗号化する"
date: 2020-03-16
categories: 
  - spring
image: "../../images/spring.jpg"
tags:
  - yaml
  - java
  - spring
---

Springでは、application.propertiesやapplication.ymlファイルに別途設定したい項目を定義することによってデータをアプリケーションの外に出すことができます。こうしてデータとアプリケーションを分離するのは、ハードコードすることによって変更が発生した場合にアプリケーションそのものを修正する不便からの開放されることができます。

しかし、こうやって別途の外部ファイルに様々な設定値を書くということは、セキュリティ問題と繋がることになる場合もあるでしょう。例えばDBなどのクレデンシャル情報や、企業の業務と関連したコードなどが平文で記載されているとしたら、ハッキングでなくても、セキュリティ問題となる可能性は十分です。こういう場合に、記載したい各種設定値などを暗号化できるといいでしょう。それを可能にするAPIがすでに存在していて、今回紹介したいものは[Jasypt](http://www.jasypt.org)というものです。

Jasyptを使うと、平文を暗号化したり、暗号文を複合して平文に戻すことができます。Spring Boot Starterも提供しているのですでに作成済みのSpring Bootアプリケーションに暗号化機能を追加するのも簡単。では、Jasyptを使ってどうやって暗号化と複合の機能をSpring Bootアプリケーションに導入できるかを今回のポストで説明しましょう。

## Jasyptによる暗号化のフロー

Jasyptを使ってのSpring Bootアプリケーションの外部ファイルに記載した設定値の暗号化と複合は、以下のようになります。

1. EncryptorクラスをBean登録
2. Encryptorクラスで平文を暗号化
3. YAMLに暗号文を記載
4. アプリケーションを起動時、YAMLの暗号文を複合して使用

Jasyptで提供しているEncryptorは、デフォルトとして提供されるクラスもありますが、カスタマイズもできるので今回はその方法を紹介していきます。

## 依存関係を追加

Spring bootを基準に、依存関係は以下のように追加します。

Mavenの場合

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.2</version>
</dependency>
```

Gradleの場合

```groovy
dependencies {
    implementation 'com.github.ulisesbocchio:jasypt-spring-boot-starter:3.0.2'
}
```

Jasypt本体だけを普通のJavaアプリケーションで使うこともできますが、今回はSpring Bootアプリケーションを基準にしているので、設定が簡単な方を紹介しています。

## カスタムEncryptorを作る

まずはSpringのBeanとしてEncrytorを登録します。Encryptorには様々なオプションが指定できますが、実際重要なのはパスワードとアルゴリズムです。パスワードとアルゴリズムが一致しない場合は複合ができないからです。ここではYAMLファイルにカスタム設定としてEncryptor用のパスワードを記載し、それを持ってEncryptorをBean登録する場合のコードを紹介します。

```java
@Configuration
public class StringEncryptorConfig {

    // YAMLから読み込むパスワード
    @Value("${settings.jasypt.password}")
    private String password;

    // encryptorBeanという名前でEncryptorをBean登録する
    @Bean(name = "encryptorBean")
    public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword("password");
        config.setAlgorithm("PBEWithMD5AndDES");
        // 以下は必須項目ではない
        config.setKeyObtentionIterations("1000");
        config.setPoolSize("1");
        config.setProviderName("SunJCE");
        config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
        config.setStringOutputType("base64");
        encryptor.setConfig(config);
        return encryptor;
    }
}
```

これでEncryptorのBean登録の設定は完了です。次は外部設定ファイルの設定ですね。

## 外部ファイルの設定

外部設定ファイルでは、Beanとして登録したEncryptorの名前を指定し、暗号化したプロパティを記載します。Encryptorの名前が一致しなかったり、記載されてない場合はアプリケーションの起動時のエラーとなるので注意しましょう。

application.propertiesの場合

```properties
jasypt.encryptor.bean=encryptorBean
properties.password=ENC(askygdq8PHapYFnlX6WsTwZZOxWInq+i)
```

application.ymlの場合

```yml
jasypt:
  encryptor:
    bean: encryptorBean
properties:
  password: ENC(askygdq8PHapYFnlX6WsTwZZOxWInq+i)
```

ここでお気づきの方もいらっしゃるだろうと思いますが、暗号化した項目は必ず`ENC()`で囲まなければなりません。そうしなかった場合は、JasyptのEncryptorは設定値をそのまま文字列として読み込みますので複合されません。

## 暗号化のアルゴリズム

Jasyptのパッケージを辿ると、基本的にいくつかのEncryptorが定義されていることがわかります。文字列だけでなく、数字タイプやバイナリーも暗号化できるので必要に応じてはSpringではなく、普通のJavaアプリケーションでもインポートして使うことができます。

今回は文字列の暗号化だけを紹介しますが、この文字列のEncryptorには以下のようなものが予め定義されています。

```java
// デフォルトとして使われるEncryptor
public BasicTextEncryptor() {
    super();
    this.encryptor = new StandardPBEStringEncryptor();
    this.encryptor.setAlgorithm("PBEWithMD5AndDES");
}

// より強いアルゴリズムを使うEncryptor
public StrongTextEncryptor() {
    super();
    this.encryptor = new StandardPBEStringEncryptor();
    this.encryptor.setAlgorithm("PBEWithMD5AndTripleDES");
}

// AES256を使う最も強力なEncryptor
public AES256TextEncryptor() {
    super();
    this.encryptor = new StandardPBEStringEncryptor();
    this.encryptor.setAlgorithm("PBEWithHMACSHA512AndAES_256");
    this.encryptor.setIvGenerator(new RandomIvGenerator());
}
```

ここに記載されているアルゴリズムは、そのままBeanとして定義するカスタムEncryptorにも使えます。ただし、アルゴリズムが複雑なものであるとそれだけ暗号化した結果は複合が難しくなるので安全ですが、アプリケーション起動が遅くなる可能性もあるので場合によって適切なものを選びましょう。また、AES256を使う場合はIvGeneratorも指定する必要があるということに注意しましょう。

コード内で暗号化がしたい場合は、Bean登録したEncryptorを呼ぶか、新しいEncryptorのインスタンスを作成してencrypt()メソッドを呼び出すとできます。当然のことですが、同じパスワードを指定しないと正しく暗号化と複合ができないということに注意しましょう。

## コマンドラインツール

Jasyptを[ダウンロード](https://github.com/jasypt/jasypt/releases/download/jasypt-1.9.3/jasypt-1.9.3-dist.zip)すると、コマンドラインツールで暗号化や複合ができるようになります。リンクからditributableバーションをダウンロードして解凍すると、binフォルダの中にbatファイルとshファイルが入っています。格ファイルの機能は以下となります。

1. encrypt.sh(bat): パスワードベースで平文を暗号化する
2. decrypt.sh(bat): パスワードベースで暗号文を複合する
3. digest.sh(bat): 複合できないHashコードを生成する
4. listAlgorithm.sh(bat): 暗号化に使えるアルゴリズムの種類を羅列する

encryptとdecryptでは、パスワードと暗号化・複合したい文をコマンドライン引数として入力するとその結果が標準出力で画面に表示されます。また、オプションとしては使いたいアルゴリズムを指定することもできます。使い方は以下のコマンドになります。

```shell
bin % ./encrypt.sh password=password input=this_is_input
```

このコマンドでの出力結果は以下です。

```shell
----ENVIRONMENT-----------------

Runtime: AdoptOpenJDK OpenJDK 64-Bit Server VM 11.0.6+10 



----ARGUMENTS-------------------

input: this_is_input
password: password



----OUTPUT----------------------

2lgKlL4gECBBtjch4WZITWDBHWhIxvVz
```

また、listAlgoritymを実行すると、以下のように現在のシステムで使えるアルゴリズムのリストが出力されます。

```shell
PBE ALGORITHMS:      [PBEWITHHMACSHA1ANDAES_128, PBEWITHHMACSHA1ANDAES_256, PBEWITHHMACSHA224ANDAES_128, PBEWITHHMACSHA224ANDAES_256, PBEWITHHMACSHA256ANDAES_128, PBEWITHHMACSHA256ANDAES_256, PBEWITHHMACSHA384ANDAES_128, PBEWITHHMACSHA384ANDAES_256, PBEWITHHMACSHA512ANDAES_128, PBEWITHHMACSHA512ANDAES_256, PBEWITHMD5ANDDES, PBEWITHMD5ANDTRIPLEDES, PBEWITHSHA1ANDDESEDE, PBEWITHSHA1ANDRC2_128, PBEWITHSHA1ANDRC2_40, PBEWITHSHA1ANDRC4_128, PBEWITHSHA1ANDRC4_40]
```

このリストの中のアルゴリズムはEncryptorをBean登録する時指定できるもののリストでもあるので、必要に応じて適切なものを選びましょう。強力なアルゴリズムを使うとアプリケーションの起動が遅くなる可能性もあります。(Spring BootアプリケーションのYAMLファイルは起動時に読み込まれますので)

## 最後に

アプリケーションの作りで、セキュリティの重要性はいうまでもなく高いものですね。先にも述べましたが、Springの設定ファイルでは特に、DBや外部システム連携のための接続情報などの敏感な情報が書かれることが少なくないため、外部設定ファイルがそのまま流出されたら困ることも起こり得ると思います。普段からそのようなことが怒らないように気を付けることももちろん大事ですが、こうやって暗号化によって情報を守るという手段もまた良い方法になるのでは、と思います。

特に、JasyptのEncryptorは外部設定ファイルだけでなく、コードの中でも使えるので、活用できる範囲が広いですね。敏感な情報を扱っている場合は、アプリケーションの中でも積極的に活用していきたいものです。性能も安定性も大事ですが、何より情報が漏れないように、ですね。
