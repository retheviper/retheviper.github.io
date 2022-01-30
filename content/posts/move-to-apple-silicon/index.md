---
title: "Apple Silicon Macに移行する"
date: 2021-12-19
categories: 
  - recent news
image: "../../images/magic.jpg"
tags:
  - mac
---

M1 Macが発売されてもう1年以上が経ちます。最初は流石に[Rosetta](https://ja.wikipedia.org/wiki/Rosetta)があるとはいえ、ネイティブアプリが少なく問題が起こったり性能が低下するケースも少なくなかったので、すぐにApple Silicon Macを購入しようとは思わなかったのです。それが今はネイティブアプリも増え、OSのアップデートもあったので十分移行しても良いタイミングになっている気がしました。そして今年、M1 ProとM1 Maxを搭載した新しいMacが発表され、既存のintelチップ搭載モデルと比べメリットが多いと思ったので購入を決めることになりました。

新しいMacについてはチップの性能という面でも、改善されたハードウェア（ディスプレイやスピーカ、キーボードなど）という面でもかなり良いものになっている印象はありますが、ここではあえてそれらについては述べません。多くのベンマークやレビューなどで明らかになっていることが多いと思いますので、ここではintel機からの移行経験に関して述べさせてください。

## 移行方法

まず、Apple Silicon Macに移行を決めてから、以下のような基準を立てました。

- 既存のMacからマイグレーションする
- なるべくネイティブアプリを使う

マイグレーションをすると決めた理由は、最初からアプリで使っている設定を最初から見直したくないものが多かったからです。アーキテクチャが違うので最初から設定する方と比べ何か問題になる可能性もあるかと思いましたが、特に問題はなかったので、設定がめんどくさい場合はマイグレーションをしても良さそうです。

また、移行には「移行アシスタント」を使いました。この場合、選べる移行の方法は以下の通りです。

- TimeMachine
- Macから移行
  - Wi-Fiを使う
  - Thunderboltケーブルで接続

TimeMachineを使うと最新の状態が反映されないので、使っているMacから移行することにしました。また、データのコピーにかかる時間が最も短いというのでThunderboltで二つのMacを直結して移行を行なっています。移行には時間は思ったよりそう掛からなく、終わった直後の状態は既存のものと変わらなかったです。

移行後はApple Siliconネイティブアプリを使いたいので、アプリの情報を一つ一つ確認してUniversalではないものからバイナリを切り替えていくことにしました。

## バイナリの確認

まずマイグレーションが終わった時点で、インストールされているアプリがApple Siliconネイティブであるかどうかを見分ける方法は以下があります。

- Finderから「情報を見る」
- アクティビティモニタの「種類」をみる
- 「このMacについて」→「システムレポート」→「ソフトウェア」→「アプリケーション」から探す
- [Is Apple Silicon ready?](https://isapplesiliconready.com)から探してみる

他にもintell版のアプリの場合は最初実行時にRosettaをインストールするか確認するダイアログが出るのでそれで確認するという方法もあります。主にターミナルで使うプログラミング言語などがそうですね。ただ、一度でもRosettaをインストールしたらintell版のバイナリでもダイアログなしで実行されてしまうので、全体の移行が終わるまではRosettaをインストールしない方が良いかと思います。

## アプリケーションの切り替え

以下はUniversal Binary(IntelとApple Siliconの両方に対応)を提供しているので、Intel機から移行した場合でも特に何もしなくて良いです。

- [Chrome](https://www.google.com/intl/ja/chrome)
- [Edge](https://www.microsoft.com/ja-jp/edge)
- [Firefox](https://www.mozilla.org/ja/firefox/new/)
- [EdgeView2](https://apps.apple.com/jp/app/edgeview-2/id1206246482)
- [Movist Pro](https://movistprime.com/en)
- [Amphetamine](https://apps.apple.com/jp/app/amphetamine/id937984704)
- [Bandizip](https://apps.apple.com/jp/app/id1265704574)
- [Obsidian](https://obsidian.md)
- [Magnet](https://apps.apple.com/jp/app/magnet-%E3%83%9E%E3%82%B0%E3%83%8D%E3%83%83%E3%83%88/id441258766)
- [Macs Fan Control](https://crystalidea.com/macs-fan-control/download)
- [iStat Menus](https://apps.apple.com/jp/app/istat-menus/id1319778037)
- [ユニコーン](https://apps.apple.com/jp/app/%E3%83%A6%E3%83%8B%E3%82%B3%E3%83%BC%E3%83%B3-%E5%BA%83%E5%91%8A%E3%83%96%E3%83%AD%E3%83%83%E3%82%AF%E5%BF%85%E9%A0%88%E3%82%A2%E3%83%97%E3%83%AA/id1231935892)
- [Microsoft Remote Desktop](https://apps.apple.com/jp/app/microsoft-remote-desktop/id1295203466)
- [Microsoft Word](https://apps.apple.com/jp/app/microsoft-word/id462054704)
- [Microsoft PowerPoint](https://apps.apple.com/jp/app/microsoft-powerpoint/id462062816)
- [Microsoft Excel](https://apps.apple.com/jp/app/microsoft-excel/id462058435)
- [Microsoft OneNote](https://apps.apple.com/jp/app/microsoft-onenote/id784801555)
- [Microsoft Outlook](https://apps.apple.com/jp/app/microsoft-outlook/id985367838)

一つ、Macs Fan Controlの場合、私はメニューバにCPUの温度を表示するために使っているのですが、温度を表示する項目を選択する際にintell機だと「CPU PECI」を選べる方が一般的かなと思いますが、Apple Siliconだとそのような項目がありません。しばらく全体的な温度をみたところ、「CPU Performance Core」が最も温度が高いように見えたので、温度を確認したい場合はそれを選んでおいた方がいいかなと思います。

### バイナリを切り替える必要があるケース

以下はApple Silicon用のバイナリを別途提供しているので、ホームページからダウンロードして既存のアプリを上書きするだけで対応できました。

- [Notion Desktop](https://www.notion.so/desktop)
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Postman](https://www.postman.com/downloads/)
- [Zoom](https://zoom.us/download)
- [Webex](https://www.webex.com/ja/downloads.html)
- [draw.io](https://github.com/jgraph/drawio-desktop/releases)

ただ、DockerはデスクトップアプリそのものはApple Siliconネイティブを使うとしても、イメージがamd64のみ対応するというケースも多いので、ここは色々と検証が必要かなと思います。私の環境ではまだMySQL（5.7）がApple Siliconに対応してなかったのですが、特に問題なく動いています。

## Homebrew

Rosettaを使って既存のintell版バイナリを使うこともできるらしいのですが、Apple Siliconに対応したバージョンはインストールされるパスが違うし（既存は`/usr/local`、Apple Silicon版は`/opt/homebrew`）、Apple Silicon版だとインストールされるパッケージは基本的にApple Siliconネイティブになるか、intell版でも新しくビルドしてくれるらしいので、使っていたintell版を消して新しくインストールし直すことにしました。

アンインストールとインストールは、別に何も意識する必要はありませんでした。公式ホームページに出ている通り、以下のコマンドを実行するだけで良いです。自動でintell版を消してくれて、新しくインストールする場合はApple Silicon版になります。

```bash
# アンインストール
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"

# インストール
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

ただ、homebrewでインストールしたパッケージ関しては自分の場合はとりあえず全部削除しておいて、後で必要になったらそれだけインストールしようという方針でしたが、以下のコマンドで使っていたパッケージのリストがバックアップできるらしいです。

```bash
/usr/local/homebrew/bin/brew bundle dump
```

## 開発環境

開発環境の構築に関しては色々なケースがあるので、こちらで別途説明します。

### JetBrains IDE

JetBrains社の製品（+ Android Studio）なら[Toolbox](https://www.jetbrains.com/ja-jp/toolbox-app)で簡単に管理ができるのですが、このアプリ自体もApple Silicon対応のバイナリに変える必要があります。

ToolboxをApple Siliconネイティブに切り替えた後は、メニューからIDEをアンインストール後に再インストールするだけです。もしToolboxを使ってない場合は、使っているIDEのバイナリをダウンロードし直す必要があります。

ちなみに、ToolboxからインストールされるIDEは`~/Applications/JetBrains Toolbox`の配下にLauncherが置かれ、それをシステムレポートなどで確認するとintel版のバイナリになっています。ただ、実行時はちゃんとApple Siliconネイティブになっているので（アクティビティモニタから確認可能）安心してください。

### Visual Studio Code

[Visual Studio Code](https://code.visualstudio.com/download)の場合は、Universal/Intel/Apple Silicon用のバイナリを全部提供していました。多分あえてIntelバージョンをインストールしてなかったら、自然にUniversalにアップデートされていたのではないかと思います。

あえてUniversalを使う必要はなく、サイズが小さいのでApple Siliconバージョンをダウンロードしたほうがいいかなと思います。

また、最近はモブプロなどでVisual Studio Live Shareを使うケースが多いかなと思いますが、こちらはまだApple Siliconに対応していません。こちらはGitHubのissueで[今後対応する予定](https://github.com/MicrosoftDocs/live-share/issues/4527#issuecomment-984823885)だというので、当面は待つしかないですね。

### Java

Javaの場合、intellだとどのベンダのものを選んでも大差ないですが、Apple Siliconだと少し話が変わってきます。なぜかというと、ベンダ別にApple Siliconに対応しているJDKのバージョンが違うからです。Rosettaを使っていない場合、「bad CPU type in executable」というエラーが発生するので、インストールされているバージョンが

各ベンダ別のApple Silicon対応済みのLTSバージョンのJDKの一覧は以下の通りです。

| JDK | 対応バージョン |
|---|---|
| [Amazon Corretto](https://docs.aws.amazon.com/corretto/latest/corretto-17-ug/downloads-list.html) | 17 |
| [Azul Zulu](https://www.azul.com/downloads/?version=java-11-lts&os=macos&architecture=arm-64-bit&package=jdk) | 1.8, 11, 17 |
| [Bellsoft Liberica](https://bell-sw.com/pages/downloads/#/java-11-lts) | 1.8, 11, 17 |
| [Eclipse Temurin](https://adoptium.net) | 17 |
| [Microsoft](https://docs.microsoft.com/ja-jp/java/openjdk/download) | 17 |
| [Oracle Java SE](https://www.oracle.com/java/technologies/downloads/#jdk17-mac) | 17 |
| [SapMachine](https://sap.github.io/SapMachine) | 11, 17 |

上記のJDKのうち、一部はintellijでもダウンロードできるものとなっていて、intellij上ではApple Silicon対応バージョンを`aarch64`と表示していますのでインストール時は必ず確認しましょう。

他にも[Red Hat](https://developers.redhat.com/products/openjdk/download)や[IBM Semeru](https://www.ibm.com/support/pages/java-sdk-downloads-version-110)などがありますが、こちらの場合はApple Silicon用のJDKを提供していません。

最近はどのJDKを選んでも特に問題はないかと思いますので、17を選ぶならOracleのでも良いし、他も好みで選んでも良さそうな気がします。私の場合はずっとAdpotOpenJDKを使っていたので、今回もTemurinを選びました。ただ、Temurinだと11がApple Siliconに対応してないので、そこはZuluを選んでいます。インストールにはhomebrewを使いました。

Apple Siliconとは直接的な関係はないですが、以下のコマンドで使うJDKのバージョンを簡単に切り替えできるのでさまざまなベンダのJDKを使ってみるのもありかもですね。

```bash
$ export JAVA_HOME=`/usr/libexec/java_home -v 11`
$ java -version
openjdk version "11.0.13" 2021-10-19 LTS
OpenJDK Runtime Environment Zulu11.52+13-CA (build 11.0.13+8-LTS)
OpenJDK 64-Bit Server VM Zulu11.52+13-CA (build 11.0.13+8-LTS, mixed mode)
$ export JAVA_HOME=`/usr/libexec/java_home -v 17`
$ java -version                                  
openjdk version "17.0.1" 2021-10-19
OpenJDK Runtime Environment Temurin-17.0.1+12 (build 17.0.1+12)
OpenJDK 64-Bit Server VM Temurin-17.0.1+12 (build 17.0.1+12, mixed mode)
```

GraalVMの場合は、まだApple Siliconに対応していません。ただ2020年からGithubの[issue](https://github.com/oracle/graal/issues/2666)がオープンの状態であって、Linux+aarch64に対応したバイナリは提供している状態なので、いずれはリリースされるかと思います。

### Kotlin

Kotlinは1.5.30から[Apple Siliconサポート](https://kotlinlang.org/docs/whatsnew1530.html#apple-silicon-support)が発表されていますが、これはKotlin/Nativeに関するものなのでKotlin/JVMの場合だとJavaの方だけ気をつけたらいいと思います。こちらもhomebrewでインストールし、特に問題はありませんでした。

### Gradle

Gradleの場合はv6.8.3を使うプロジェクトがあったので、Javaの設定が終わった後にビルドしてみると以下のようなエラーが出ました。（依存関係やプロジェクトの設定によってエラーの種類は変わる可能性があるかと思います）

#### Java 17で実行した場合

Java 17(Temurin)で実行した場合、Gradleそのものが実行時にエラーを吐きます。おそらくrefelction関係でdeprecatedになっていたAPIを使っているのが問題になったのではないかと思います。

```bash
> java.lang.IllegalAccessError: class org.gradle.internal.compiler.java.ClassNameCollector (in unnamed module @0x8f1317) cannot access class com.sun.tools.javac.code.Symbol$TypeSymbol (in module jdk.compiler) because module jdk.compiler does not export com.sun.tools.javac.code to unnamed module @0x8f1317
```

#### Java 11で実行した場合

Java 11で実行するとコンパイルまでは行われるようですが、テスト（junit）の実行で以下のような問題が起こりました。

```bash
*** java.lang.instrument ASSERTION FAILED ***: "result" with message agent load/premain call failed at src/java.instrument/share/native/libinstrument/JPLISAgent.c line: 422
FATAL ERROR in native method: processing of -javaagent failed, processJavaStart failed
Process 'Gradle Test Executor 1679' finished with non-zero exit value 134
```

調べてみると、Gradleはv6.9からApple Siliconに対応したようだったので、ラッパーを最新にバージョンアップ。`gradle/wrapper/gradle-wrapper.properties`のバージョン指定を変えるだけでも対応できますが、以下のコマンドを使っています。

```bash
./gradlew wrapper --gradle-version=7.3.1 --distribution-type=bin
```

いきなりv6.8.3からv7.3.1にアップデートしたのですが、テストまで正常終了しています。このプロジェクトはKotlin + Spring bootの構成なので、もし同じような構成のプロジェクトがあるとしたらJava/Kotlin/Gradleのバージョンアップをおこなってからアプリのビルドを試してみましょう。

### Ruby / Python / Go

RubyとPythonに関してはmacOS上ですでにインストール済みの状態ですが、バージョンやプロジェクトの設定などによっては問題が起こる可能性もあるのでhomebrewでインストールしました。このブログで使っている[jekyll](https://jekyllrb.com)はrubyをインストールし直したので同じくインストールし直す必要がありました。こちらもhomebrewでインストールができ、既存のプロジェクトにおいてはなんの問題もなく実行することができました。

Pythonの場合、既存のプロジェクトのパッケージを再インストールする必要がありました。でも`requirements.txt`があれば特に問題にはならないくらいです。

Goの場合も1.16からApple Siliconに対応しているので、特にバージョンの指定が必要なケースでなければ、homebrewで最新をインストールしても良いかなと思います。ただ、既存のプロジェクトのGOROOTやGOPATHの問題があるので、ホームページから別途ダウンロードして設定する必要のあるケースもあるかと思います。

## Rosettaを使うしかないケース

多くのアプリがApple Siliconに対応してきましたが、バージョンアップそのものが終わったり、Rosettaを通じて問題なく動く（からApple Silicon対応は後回しにするという政策の）アプリに関してはネイティブのバイナリが存在しない場合もあるので、仕方なくRosettaを使うしかないかなと思います。

ソースコードをダウンロードして、ローカルでビルドするという方法もあるかと思いますが、使われているSwiftのバージョンが低い場合はXCodeですぐにビルドできない場合もあったりしました。いつかはRosettaのサボートも終わりそうなので、長期的な観点ではこのようなアプリは他のものに代替した方が良いかも知れません。

### Mattermost

[Mattermost](https://mattermost.com/download)はSlackと似たようなコミュニケーションツールで、サーバにインストールすることで無料利用ができるしマークダウンのサポートが優秀だったりするのでプライペートでよく使っています。ただ残念なことに、こちらはまだApple Silicon用の正式リリース版がないようです。

正式リリースの予定はあるようなのでバージョンアップまでintell版を使うという選択肢もありますが、[GitHubのリポジトリ](https://github.com/mattermost/desktop/releases)を見るとUniversalとApple Silicon用のバイナリのベータ版も存在しているので、どうしてもRosettaを使いたくない場合はこちらを選んでみても良いかも知れません。

### KeyboardCleanTool

[KeyboardCleanTool](https://folivora.ai/keyboardcleantool)は、アプリを実行している間に全てのキー入力を無視するという単純なツールです。キーボードが汚れて拭きたいときによく使っていますね。残念ながらこちらもまだApple Siliconに対応していません。同じ会社で開発している[BetterTouchTool](https://folivora.ai)はUniversalバイナリで提供されていますが、その対応ができたのも11月のことなので他の製品が全部Apple Siliconに対応するにはかなり時間がかかるかも知れません。

このようなアプリは特にネイティブにならなくても困らないものなので、Apple Siliconネイティブ対応はかなり優先順位が低い感がありますね。

## OneDrive

Microsoft社の製品にしてはかなり珍しいケースですが、対応が遅れていますね。ただ、今月[PreviewとしてUniversalバージョンが利用できる](https://techcommunity.microsoft.com/t5/microsoft-onedrive-blog/onedrive-sync-for-native-arm-devices-now-in-public-preview/ba-p/3031668)ようになったらしいので、もうすぐApple Siliconネイティブ版が出るかも知れません。その際にはApp Storeで自動的にアップデートされるはずなので、待つだけですね。

### Flutter

Dartは[v2.14からNative対応](https://medium.com/dartlang/announcing-dart-2-14-b48b9bb2fb67)しているのですが、Flutterはまだ未対応らしく、公式を見ても[Rossettaを推奨](https://github.com/flutter/flutter/wiki/ㅅDeveloping-with-Flutter-on-Apple-Silicon)しています。なのでRosettaを入れて実行した方が早いですね。

他に、`flutter doctor`を実行していくつか問題が出るケースがあるかと思います。そういう場合は以下の手順で対応できました。

- `cmdline-tools component is missing`と出る場合
  - Android Studioから`Appearance & Behavior` -> `System Settings` -> `Android SDK` -> `SDK Tools` -> `Android SDK Command-Line Tools`にチェックを入れる
- `CocoaPods installed but not working`と出る場合
  - `brew install cocoapods`でインストール

## 最後に

移行が終わって本格的にApple Silicon Macを使ったのはまだ一週間経たないくらいの短い期間ですが、思ったより移行がスムーズで、ネイティブ対応済みのアプリも多かったので、M1が発表された直後の私のように互換性に疑問を持った方がいるとしたら（十分な事前調査を前提として）新しいMacに移行するのもありかも知れないと思います。

最初はAppleが公式的に全てのMacをApple Siliconに移行するまで2年という計画を立てたという話をしていましたが、ユーザとしてはより時間がかかるのではないかと思っていました。しかし、実際に触ってみると、今でも十分移行ができる状態になっていると思います。

では、また！
