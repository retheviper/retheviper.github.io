---
title: "Gradleからコマンドライン引数を渡す"
date: 2019-09-17
categories: 
  - gradle
photos:
  - /assets/images/sideimage/gradle_logo.jpg
tags:
  - groovy
  - java
  - gradle
---

最近仕事で作っているのは、固有ライブラリーです。ただ思っていたことと違ったのは、まず完全自作ではなく既存のライブラリーを改良するような物であるという点。そして二つ目はSpring Bootで作られているということです。Spring bootは今回初めて触れましたが、以前Spring MVCでWebサイトを作ってみた経験はありました。ただSpringはWebアプリケーションを作るためのフレームワークなので、これを持ってライブラリーを作ったり、Webページのないアプリケーションを作るということは想像もしたことがなかったです。

それが仕事で両方の使い道に触れることがあって、まだ世の中自分の知らないものばかりだなと思いました。プログラミングとは言語やフレームワークを使えるかどうかの問題だけでなく、どう活用してどんな物を作るかの問題も含めて考えなければならない物ですね。そして画面はなくてもMainクラスを作ることでJarから独立実行ができて、インポートしてはライブラリーとして使える独特？なしようとなりました。そしてもちろんそうするためには少し準備は必要となりました。

表題では短くなりましたが、今回の主題であり仕事での要件は実際はこういう物でした。

## Gradleのtaskで実行するSpring bootアプリケーションにして、起動時にはコマンドライン引数を渡す
 
こうして書いてみると複雑のようですが、実際はそこまで難しい概念ではないと思います。まずGradleはMavenのようなライブラリー管理ツールとして知られていて、`gradlew`に様々なオプションをつけることでJarのビルド、テストなどの行為ができますね。ここでしたいのは、そんなGradleでできるタスク(オプション)を追加することです。そして追加されたタスクを実行すると、Spring Bootプロジェクトとして作成されているライブラリーのMainクラスを起動させることです。

ただ、皆さんにもわかるように、Sping BootプロジェクトでMainクラスが用意されていると、普通に`java -jar project.jar`という風にコマンドラインから起動させることができます。なのになぜあえてGraleのタスクに入れようとしたかというと、以下のような理由があります。

### マルチプロジェクト構造となっている

現在の設計で、ライブラリー全体のプロジェクトは複数のサブプロジェクトを含むマルチプロジェクトとなっています。これでちゃんと伝わるか分かりませんが、とにかく表で表現すると以下のようになっています。

```
rootproject
┣ target
┣ generator
┗ runtime
```

簡単に説明しますと、現在のライブラリーはgeneratorというサブプロジェクトを利用して(runtimeはその時generatorから参照します)targetプロジェクトのコードを操作します。ここで最終的にgeneratorとruntimeはJarとして提供され、使用者はこのライブラリーでの処理を適用したいプロジェクトをtargetの位置に置いて使うことになります。こうなった場合に、コマンドラインからruntimeに依存しているgeneratorのJarを起動してtargetのソースファイルを操作するように指定するのはかなり面倒臭い作業となりますね。これを自動化したかったです。

### 起動時に渡したい引数が多い

Gradleのタスクとして実行したかったもう一つの理由は、ライブラリーが処理を行うために必要な引数の種類が7つくらいがあって、これを全部覚えるのは難しいからです。それにタイポにより処理に失敗する可能性も上がりますね。ここで思ったのが`.bat`ファイルを用意していて、使用者がそれを修正したらいいだけの話ではないかと一瞬思いましたが、やはりあまり良い方法ではなかったです。Gradle自体も今はそうなっていますが、別途のファイルを用意するということは、OSごとにそのファイルを作成する必要があるということですね。使用者がどんな環境で実行するかわからないのでとりあえず`.bat`と`.sh`の両方を準備する必要がありますね。

それに、このライブラリーがWindowsやLinuxでしか使用されないだろうと言い切れないので、そういう場合はより変数は多くなります。そうなるといちいちOSや環境に合わせて、コマンドライン引数を渡すための方法を作らなければならないですね。わざわざそんなことをするよりは、Gradleのタスクとして用意し(もちろん引数はファイルに記載して読み込まれるように)、環境のことはGradleにお任せした方がコードの管理や便宜性という面からして良さげな気がしました。統一感もあって、使用者にも良い印象になりそうですしね。
 

## [target] build.gradle
 
まずはタスクを実行したいtargetプロジェクトの方から始めましょう。Gradleプロジェクトは基本的に`build.gradle`というファイルを持ち、このファイルを修正することで依存関係やプラグインなど様々な設定ができますね。同じく、カスタムタスクを追加したい場合もこちらに追加したいタスクの内容を記載します。

```groovy
apply from: 'default.gradle' // このファイルを読み込むという意味

task taskname(type: GradleBuild) { // タスク名とタイプの設定
    group = 'application' // gradlew tasks コマンドから、applicationタブにこのタスクが追加される
    description = 'run program' // このタスクの説明
    dir = '../generator' // タスクの実行基準位置
    tasks = ['bootRun'] // 実行する内容
    startParameter.projectProperties = [args: '${defaultArgs}'] // コマンドライン引数としてdefaultArgsを読み込む
}
```

ここで`default.gradle`を読み込んでいる理由は、そのファイルにデフォルト値を記載しておき、タスクを実行する時に読み込んだ値をコマンドライン引数として使うためです。最後の行で`defaultArgs`と書いてありますが、これがデフォルト値を変数にしたものです。こうやってファイルを分離することで実際の使用者がこのタスクを実行するときは、`default.gradle`を修正して引数として渡される値だけを調節することになります。

## [target] default.gradle

次に、`build.gradle`でタスクを実行する時に読み込まれるファイルの設定は以下のようになります。ここでは読み込まれる対象としての設定と、変数の形で宣言したコマンドライン引数を記載するだけです。

```groovy
ext { // 読み込まれる対象と表記
    defaultArgs = '-arg1 value -arg2 value' // arg1とarg2の二つの引数がある場合
}
```

こちらは簡単ですね。引数名が書いてあるので順番は関係なく、あとはそれぞれの値を変えるだけでよくなります。

## [generator] build.gradle

それでは続いて、タスクで実行される側の設定です。 targetのタスクでgeneratorを`bootRun`すると指定していたので、それに合わせて`bootRun`時の挙動を設定します。例えば引数をどんな形で受け取るか、メインクラスはどれかという設定ですね。

```groovy
bootRun { // bootRun時の挙動
    if (project.hasProperty('args')) {
        args project.args.split('\\s+') // コマンドライン引数がある場合、空白を基準に分割する
    }
}

jar { // Jarとしての設定
    manifest {
        attributes 'Main-Class': 'com.package.Main' // メインクラスのクラスパス(パッケージとクラス名)の指定
    }
}
```

コマンドライン引数を分割する理由は、皆さんも予想しているとは思いますが、Javaのメインメソッドは普通文字列の配列として引数を受け入れるからです。こうやって分割しておくと実際の処理で引数のパースが簡単になりますね。

ここまでこればGradleの設定は終わりです。あとはJavaでのメインクラスの設定です。

## [generator] Main.java

generatorのJarを実行した時に呼ばれるメインクラスを作ります。ここでは一般的なJavaのメインメソッドと、Spring bootとしてのメインクラスの作法、JCommanderでコマンドライン引数をパースするための作法、Lombokが混在していますのでそれぞれに対する知識のない方には少し難しいコードになっているかも知れません。

ただ、実行されている時の動作としては単純なものになっているので、Springのアノテーションにある程度慣れている方ならすぐに理解できると思います。まずコードは以下のようになります。

```java
@SpringBootApplication // Spring bootとしてのメインクラスにつける
@EnableAutoConfiguration(exclude = { DataSourceAutoConfiguration.class }) // H2関連エラーが出たので付けました
public class Main implements CommandLineRunner {

    @Autowired
    private CoreProcessClass coreProcessClass; // @Componentとなっている実際の処理クラス

    @Override
    public void run(final String... args) throws Exception { // CommandLineRunnerを継承するして実行時の動作をオーバーライドする
        final CommandLineOptions options = CommandLineOptions.parse(args); // パースと同時にBeanを生成
        coreProcessClass.startProcess(options.getArg1, options.getArg2); // 本処理開始
    }

    public static void main(final String[] args) {
        SpringApplication.run(Main.class, args); // メインメソッドとしては引数を渡すだけ
    }

    @Data
    public static class CommandLineOptions { // コマンドライン引数をパースするクラス

        // JCommanderを使用した引数の設定
        @Parameter(names = "-arg1", description = "File", required = true, converter = ArgumentsToFileConverter.class) // 引数は文字列なので、コンバータクラスを使う
        private File arg1;

        @Parameter(names = "-arg2", description = "String", required = true) // 普通の文字列の場合
        private String arg2;

        private CommandLineOptions() {}

        public static CommandLineOptions parse(final String... args) { // 実際のパースを行うメソッド
            try {
                final CommandLineOptions options = new CommandLineOptions();
                JCommander.newBuilder()
                        .addObject(options)
                        .acceptUnknownOptions(true)
                        .build()
                        .parse(args);
                return options;
            } catch (ParameterException e) {
                e.getJCommander().usage();
                throw e;
            }
        }
    }

    public class ArgumentsToFileConverter implements IStringConverter<File> { // JCommanderで引数をオブジェクトに変えるためのクラス

        @Override
        public File convert(final String argument) {
            return new File(argument);
        }
    }
}
```

[JCommander](http://jcommander.org)を使うことでコマンドラインのパースは簡単にできます。ここでいうパースは単純に文字列だけを意味することではなく、必須項目としての指定(ない場合は例外となる)やオブジェクト変換などの様々なことができるという意味です。例えば引数として渡した文字列があるものはファイルパスだとしたら、それを読み込んでFileオブジェクト化したり複数の引数をListとして取得することもできます。

そしてパースしたオブジェクトを、本処理で使われるオブジェクトに渡すだけで終わり。意外と簡単に終わりますね。

## 最後に

実は、今まで説明した内容は自分で考えて作り出したものではないです。最初はGradleタスクを作るために導入した[ライブラリーの問題](../../../09/08/java-module-conflict)があってなかなか仕事が進まなく、すでに出来上がっていたものを参考にしたものにすぎません。でもここであえて紹介するのは、皆さんに共有する価値があると思ったからです。

特にサーバーで動くアプリケーションを作る場合はやはりコマンドラインで起動させる場合が多いですし、環境によっては引数で違う値を渡す必要があるかもしれません。そのような時に、このようにファイルから引数を渡して実行するタスクを作成して環境ごとに設定を変えたり違うファイルを読むようにするとかなり便利そうですね。なので皆さんにもぜひ紹介したいと思いました。

そして個人的には、今は自分も先輩方の成したものから学ぶばかりですが、いつかはこんな考え方もあるんだと後輩に伝えられたらいいなと思わせる、大事な経験でした。これから先はまだまだ遠いですね！