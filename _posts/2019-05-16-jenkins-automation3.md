---
title: "Jenkinsで何もかも楽にしたい(3)"
date: 2019-05-16
categories: 
  - jenkins
photos:
  - /assets/images/sideimage/jenkins_logo.jpg
tags:
  - ci/cd
  - jenkins
---

仕事でJenkinsを使って、自動化したいと言われたのはいくつかのタスクがあります。そしてそのタスクを実行するにもまたいくつかの手順が入りますね。こんなことがしたいという想像力ももちろん大事なものですが、何かをするにはそれに必要な環境を整えることが何よりも大事なのではないかと思います。今回のポストでは仕事で与えられた作業と、それを準備した過程を述べたいと思います。

まずGitからSpring bootアプリケーションをPullしてきて、ビルドすること。まず私はSpring Framework[^1]は扱ってみたことがありますが、Spring Bootは今回が初めてでした。そしてMavenは使ったことがあるものの、今回のようにGradleを使ったプロジェクトには触れたことがなかったです。Spring Bootが（そしてGradleが）以前と比べ初期設定が簡単とは言われていますが、Eclipeの上でしかアプリケーションを実行した経験しかないので、最初はどうやったらいいかイメージすらなかったです。

そしてJenkinsでどのようにJobを構成すればいいかもわからなかったので、まずJenkinsでどのようにJobを構成し自動化ができるかを調べることから始めました。

Jenkinsでは、Jobという名称のタスクを作り、とある行動を中に仕組み、それを一つの単位として実行することができます。全てのプログラムがそうでありますが、Linuxでのシェルスクリプトみたいに、繰り返す必要のある行為を自動化するようなものです。そして出来がったJobは手動実行するか、条件（トリガー）を指定して実行するようになります。

今回のタスクを振り返ってみましょう。JavaアプリケーションがGitにあります。それをPullし（Jenkinsが実行されているサーバにソースコードを持ってきて）、ビルド（実行可能なパッケージにする）する。これでJenkinsは最新のソースコードを確保しつつ、実行可能なアプリケーションをデプロイできる状態になります。それではまず必要な道具はJDKとGradleです。

## OpenJDKをインストールしましょう

まずJenkinsはJava８でも実行できますが、今回のアプリケーションはなんとJava11を使っていました。OpenJDK11[^2]をインストールします。`yum install java`をするとOracleのJavaがインストールされるので、OpenJDKをインストールしたい場合はまた少しの手順が必要となります。今回はwgetではなく、curlを使ってみます。

```bash
$ curl -O https://download.java.net/java/GA/jdk11/13/GPL/openjdk-11.0.1_linux-x64_bin.tar.gz
```

curl -OはURLからファイルをダウンロードして保存するということです。ダウンロードしたファイルは圧縮されているので解凍します。

```bash
$ tar zxvf openjdk-11.0.1_linux-x64_bin.tar.gz
```

tarは圧縮したり解凍するとき使うコマンド。zは.gzファイルを、xは圧縮ファイルの展開、vは処理したファイルを表示、fはこのファイルを指定するという意味らしいです。解凍が終わったらファイルを適切な場所に格納しましょう。

```bash
$ mv jdk-11.0.1 /usr/local/
```

次に簡単なシェルスクリプトを書きます。Linuxでの環境変数を設定するためです。まずviやvimでファイルを作りましょう。

```bash
$ vi /etc/profile.d/jdk11.sh
```

iを押して以下の内容を書きます。Javaを格納したパスを確認してください。

```bash
export JAVA_HOME=/usr/local/jdk-11.0.1
export PATH=$PATH:$JAVA_HOME/bin
```

esc⇨:wqで保存と終了。そしてすぐシェルスクリプトを今の状態に適用させます。sourceコマンドを使います。

```bash
$ source /etc/profile.d/jdk11.sh
```

これでOpenJDK11は準備されました。私はCentOS7を使っているのですが、ubuntuなどの違うLinuxではまた手順も色々あるようです。とにかくインストールがちゃんと終わっているか`java -version`で確認してみましょう。

![](/assets/images/jenkins_screenshot/jenkins_java8.png)

あれ？Javaのバージョンが8です。すでにインストールされていたんですね。

## 使いたいJavaのバージュヨンを切り替える

すでに違うバージョンのJDKがインストールされた場合は、次の手順で使いたいJavaのバージョンを選択できます。
まず新しくインストールしたJavaを選択できるよう登録します。

```bash
$ alternatives --install /usr/bin/java java $JAVA_HOME/bin/java 2
```

登録が終わったら、以下のコマンドで現在登録されているJavaを表示します。

```bash
$ alternatives --config java
```

CentOSインストール時にすでに二つのバージョンのJavaがインストールされてましたね。Java11が3番目になったので3を入力します。これでJavaは準備オッケーです。

![Jenkins Alter](/assets/images/jenkins_screenshot/jenkins_alter.png)

ちなみに、JenkinsもまたJavaで作られているので場合によってはJava11のJVMで起動できます。Java11のJVMをJenkinsに登録するには以下の手順になります。

```bash
$ vi /etc/init.d/jenkins
```

Jenkinsの設定ファイルです。candidatesという部分に様々なバージョンのJVMの経路が機材されていますので、ここにOpenJDK11のbinフォルダのパスを追加します。esc⇨:wqで終了！

![Jenkins JVM](/assets/images/jenkins_screenshot/jenkins_jvm.png)

## Gradleもインストールしよう

次にGradleをインストールします。macOSでは`brew install gradle`で簡単にインストールできましたが、LinuxではどうもまたOpenJDKと同じ手順が必要なようです。wgetをまた使ってみます。

```bash
$ wget https://services.gradle.org/distributions/gradle-5.4.1-bin.zip -P /tmp
```

-Pオプションをつけると指定したフォルダにファイルを保存します。

```bash
$ sudo unzip -d /opt/gradle /tmp/gradle-5.4.1-bin.zip
```

zipファイルンなので、unzipコマンドで解凍します。-dもまたフォルダ（ディレクトリ）を指定するオプションです。

```bash
$ sudo nano /etc/profile.d/gradle.sh
```

今回はnanoを使ってスクリプトを作ってみます。よりGUIに近い感じがしますね。nanoがなければ`yum install nano`でインストールしてもいいし、viを使っても良いです。

私はどこでも使えるということ（環境によってはsudoの権限があるとしても、勝手に色々インストールできない場合もあるので)からviを使うことが多いですが、nanoはより直観的で使いやすいと思います。

![Gradle SH](/assets/images/jenkins_screenshot/gradle_sh.png)

添付の画像のように、以下の内容を入力します。

```bash
export GRADLE_HOME=/opt/gradle/gradle-5.4.1
export PATH=${GRADLE_HOME}/bin:${PATH}
```

nanoではctrl+xを押すと編集した内容を保存するかを聞いてきます。Yのあとエンターを押すと保存。US標準のキーボードを使う私にとっては:wqより入力が簡単で好きです。

あとは作られたスクリプトに実行の権限を与え、`source`コマンドで環境変数として登録。

```bash
$ sudo chmod +x /etc/profile.d/gradle.sh
$ source /etc/profile.d/gradle.sh
```

chmod 755のようなコマンドを書いたことが多いですが、+xで実行の権限だけ与える方が習慣的には良さそうですね。

```bash
$ gradle -v
```

インストールに成功したかを確認するにはやはりバージョンの確認ですね。以下のような画面が表示されたらGradleのインストールも成功です。

![Jenkins Gradle Version](/assets/images/jenkins_screenshot/gradle_version.png)

最新のバージョンではSwift5をサポートするようですね。こちらもいつか扱いたいと思います。

それではいよいよJenkinsに戻り、JenkinsにJDKとGradleを環境として登録します。

## JenkinsにJDKとGradleをつなげる

やっとJenkinsに戻りました。もうこれがJenkinsのポストかLinuxのポストカわからないくらいになっていましたが、とにかく戻ってきました。

ウェブブラウザに`localhost:Jenkinsのポート番号`を入力してJenkinsのメイン画面に入ります。その後は、左のメニューから`Manage Jenkins`をクリックします。そうすると以下のような画面が現れます。

![Jenkins Manage](/assets/images/jenkins_screenshot/jenkins_manage.png)

`Global Tool Configuration`を押します。ここがJenkinsで使われる環境変数的なものを設定する画面です。

![Jenkins Global Tool Settings](/assets/images/jenkins_screenshot/jenkins_globaltoolsettings.png)

JDK項目の`ADD JDK`を押します。`install automatically`というオプジョンが基本的にチェックされていますが、これはOracleのJavaしかインストールできないオプションです。またバージョンに制限があるので（私の場合はJava9までしか設定できませんでした）、チェックを外して`JAVA_HOME`にインストールしたJava11のパスをいれます。以下のような感じです。Nameも必要なので適当な文句をいれます。

![Jenkins JDK](/assets/images/jenkins_screenshot/jenkins_jdk.png)

次にGradleです。こちらもJavaと同様、自動インストールのオプションがあります。でも我々がインストールしたのよりはバージョンが低いですね。なのでこちらも自動インストールのオプションを外し、パスをいれます。以下のようになります。

![Jenkins Gradle](/assets/images/jenkins_screenshot/jenkins_gradle.png)

`Save`を押して保存することを忘れずに！

これでJenkinsでのJDKとGradleの設定は終わりです。これからはJavaアプリケーションをビルドできるようになりました。次のポストで実際Spring Bootで作られたアプリケーションをGitでPullし、ビルドするタスクを作ってみたいと思います。

[^1]: 厳密には、Spring Frameworkというよりは旧バージョンと言えるでしょう。Spring BootはSpring Frameworkに含まれるものなんですからね。いつかSpring Bootに関しても勉強したいと思うので、機会があればポストしたいと思います。
[^2]: 現在はJava12まで出ていますが、JenkinsでJava11をサポートし始めたのも最近のことなので（2019年３月）、まずJava 11を選びました。
