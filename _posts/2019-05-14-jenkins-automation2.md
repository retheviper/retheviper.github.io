---
title: "Jenkinsで何もかも楽にしたい(2)"
date: 2019-05-14
categories: 
  - jenkins
photos:
- /assets/images/sideimage/jenkins_logo.jpg
tags:
  - ci/cd
  - jenkins
---

前回から続きます。Jenkinのインストールが終わったら、初期設定の番です。どんなことでも始めが一番面倒なものですが、それだけ初期設定をちゃんとやっていると後の作業が楽になるものですね。なので今回はJenkinsの初期設定に関してまず述べたいと思います。

Jenkinsの初期設定はだいたい以下の順になります。他にも色々しておいたら便利な設定があるかも知れませんが、あくまで初心者の私の観点が基準なので参考までにしてください。

## ポートの設定

Jenkinsの基本ポートは`8080`です。このままJenkinsを起動すると、Webブラウザから`Jenkinsを起動中のシステムのIPアドレス:8080`でJenkinsに接続できます。

しかし、Webアプリケーションを開発してみた経験のある方には8080というポートはあまり良い選択肢ではないことがわかるはずです。同じポートに設定されている二つ以上のサービスが問題を起こす可能性がありますからね。[^1]

私はのちにTomcatを使うことがあるかもしれないと思い、Jenkinのポートは`8088`に変えました。`vi`や`vim`[^2]でJenkinsのシステムコンフィグファイルを開けます。
    
```bash
$ sudo vim /etc/sysconfig/jenkins
```

そうすると、以下のような画面が現れます。少し下にスクロールしたら`JENKINS_PORT`と、親切に書いてあるのが見えますね！`I`を押してインサートモードに切り替え、好きなポートに変えましょう。

![](/assets/images/jenkins_screenshot/jenkins_configport.png)

書き換えが終わったら`ESC` ⇨ `:wq`で保存後終了を忘れずにしましょう。


## 起動〜初期パスワードの設定

Jenkinsは初起動で初期パスワードを要求します。この初期パスワードが格納されてある位置は、基本的に`/var/lib/jenkins/secrets/initialAdminPassword`というパスに保存されるようですが、OSなどの環境によってパスが変わる場合もありますからね。[^3]でも、一度Jenkinsを起動したら初期設定のページからパスを確認できるので心配いりません。

ポート設定が終わったら（8080をそのまま使いたいならそのままでもいいです）、まずJenkinsを起動します。
```bash
$ service jenkins start
```
`[OK]`というメッセージが出力されるはずですが（ポート設定に問題がある場合もあるので）念の為起動状況を確認します。

```bash
$ service jenkins status
```
実は、仕事で１日前は元気だったJenkins先生が、いきなり接続できなくなっていたことがったのです。やはり人は何かよくないことを経験すると、慎重になるものです。`Active: active (running)`というメッセージを確認できたら、いよいよJenkinsのページに接続です。

![](/assets/images/jenkins_screenshot/jenkins_servicestatus.png)

`Jenkinsを起動中のシステムのIPアドレス:ポーと番号`をWebブラウザに入力してJenkinsのページへ接続します。もちろん、起動中のシステムからは`localhost:8080`などでも接続できます。

![](/assets/images/jenkins_screenshot/jenkins_initpass.png)

やはりパスは`/var/lib/jenkins/secrets/initialAdminPassword`でした。ポート設定と同じく、viやvimで中を覗き、そのパスワードを入力します。

```bash
$ sudo vim /var/lib/jenkins/secrets/initialAdminPassword
```

## プラグインと管理者アカウントの設定

パスワードの入力に成功するとしばらくして、プラグインの設定画面が現れます。自分でプラグインを選んでも良さそうですが、私は自信がないのでオススメのボタンを押します。当たり前なことですが、プラグインは後ででもインストールできます。


![](/assets/images/jenkins_screenshot/jenkins_initplugins.png)

オススメのボタンを押すと自動的にプラグインのインストールを進めてくれるので、待ちましょう。

![](/assets/images/jenkins_screenshot/jenkins_installingplugins.png)

プラグインのインストールが終わったら次は管理者アカウントの設定です。一つでも満たしてない項目があったら怒られるので全部書きます。

![](/assets/images/jenkins_screenshot/jenkins_setupadmin.png)

管理者アカウント設定の次は接続アドレスの設定。私は今のままでいいと思うので（仮想マシンでCentOSをインストールしてJenkinsを動かしています）そのままにします。

![](/assets/images/jenkins_screenshot/jenkins_addresssetting.png)
    
そして、Jenkinsが用意されたという画面が出ます。長かったですね…早速使うというボタンを押します。

![](/assets/images/jenkins_screenshot/jenkins_ready.png)
    
ジャジャーン。ようやくたどり着きました。Jenkinsのメイン画面です。ここまでの旅も本当に長かったですね。それでもJenkinsは強力なツールなので、ここまでする価値があると私は思います。

![](/assets/images/jenkins_screenshot/jenkins_mainpage.png)

それでは、次からは具体的にJenkins先生と共にどんなタスク（Job）を作り、実行したかを述べたいと思います。また会いましょう！

（続きます）




[^1]: 特にSpring FrameworkなどでWebアプリケーションを実装する場合にそうですね。Tomcatの基本ポートも`8080`になっていますから。

[^2]: viとvimのうち、どれを選ぶかはいつも悩ましいことです。vimの方がよりカラフルでコードを読みやすいという点では良いですが、システムによってはインストールされてないですからね。

[^3]: macOSでインストールしたら、初期パスワードのパスが`ユーザホームディレクトリ`の直下だった場合もありました。rootではないユーザでインストールしたからかもしれませんが。