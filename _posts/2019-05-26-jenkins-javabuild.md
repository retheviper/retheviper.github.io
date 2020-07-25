---
title: "JenkinsでJavaプロジェクトをビルドする"
date: 2019-05-26
categories: 
  - jenkins
photos:
  - /assets/images/sideimage/jenkins_logo.jpg
tags:
  - ci/cd
  - jenkins
---

今回はJenkinsで一つのJobを生成することまでやりたいと思います。まず前回も書きましたが、私に要求されたタスクは次のようなことです。「GitにSpring Bootのアプリケーションがあるので、それをビルドするJobを作って欲しい」とのこと。リポジトリを覗き、JavaとGradleはインストールしたので早速Jobの生成に取り掛かることにしました。

今回は当時と似たような環境を作るためにあらかじめSpring Bootプロジェクトを作成してGitにアップロードしています。

## それでJobとは？

Jenkinsを使う理由は、このJobのためです。Jobは`自動化されたタスク`で、実行の条件から頻度、タスクの内容を組み込むことができます。例えばGitのリポジトリに誰かがPushしたらJobを実行するように設定できるし、実行したら自動的にGitをPullするように設定できるということです。

なのでJenkinsをサーバ上で起動しておいて、必要に応じてJobを組んでおけば大変手間が省けるという話は全てこのJobのおかげだと言えます。今回はこのJobの生成の手順から説明しましょう。

## Jobを生成する

Jenkinsのメインページに入ります。何もJobがない状態なので、メイン画面に`create new jobs`というリンクがあることを確認できます。Jobが無い場合はこちらでもJobを生成することができます。しかし何かしらのJobがある場合は、左のメニューで`New item`をクリックすれば新しいJobの生成画面に移行します。どちらかをクリックします。

![](/assets/images/jenkins_screenshot/jenkins_mainpage.png)

`Enter an item name`でJobの名前を指定します。また、下にはどんなJobを生成するかに対するいくつかのテンプレートがあります。ここでは`Freestyle project`を選びます。ちなみに、Job生成時に指定した名前と同じ名のフォルダがJenkinsの`Workspace`というフォルダの下に生成されるので、なるべく半角英文字・スペースなしで生成することをお勧めします。

![](/assets/images/jenkins_screenshot/jenkins_createjob.png)

Jobの名前とテンプレートを選んだなら、`Ok`を押してJobを生成します。

![](/assets/images/jenkins_screenshot/jenkins_jobmain.png)

ここがJob設定のメイン画面です。画面の最上段にある各タブの機能は以下のようになります。

- `General`: Job全般の設定。Jobの説明を記述したり（どんなタスクなのかなど）、ビルド[^1]時以前のビルドを削除するかなどの設定ができます。
- `Source Code Management`: バージョン管理に関する設定です。GitとSubversionに対応しています。今回はGitを使うことになりますね。
- `Build Triggers`: Jobをどう実行するかに対するトリガーの設定です。リモートからURLを通じた実行になるか、定期的に実行するかなどの設定ができます。
- `Build Environment`: Jobのビルド時に使われる環境に関しての設定です。基本的にJobはWorkspaceという空間を占め、ビルド時に使われたファイルなどは全てそのフォルダに保存されます。基本パスは`/var/lib/jenkins/workspace/[生成したJobの名前]`となります。
- `Build`: Jobのビルド時のアクション（タスク）を指定します。ここで主な処理が行われます。
- `Post-build Actions`: Jobのビルドが終わった後に遂行するアクションを指定します。他のJobをビルドするなどの行動が指定できます。

各タブはどんなプラグインをインストールしたかによって指定できる動作の項目が増えることがあります。また違うポストで扱うかもしれませんが、例えば`Publish over SSH`という

## Gitのレポジトリを設定

それでは本格的に今回のタスクを作ってみましょう。まずGitからJavaのコードを持ってくる必要があるので、`Source Code Management`からGitを選択します。

![](/assets/images/jenkins_screenshot/jenkins_gitconfigure.png)

`Repository URL`にはGitのリポジトリを入力します。最初何も入力されてない時は接続エラーが出ていますが、URLを入力すると自動的にリポジトリへの接続を試し、問題がないとエラーは消えます。また、`Credentials`から接続するIDなどの認証情報を選択します。最初は何もないので、`Add`をクリックし新しい認証情報を入力しましょう。

今の所、私の作ったリポジトリはブランチが一つしかないのですが、もし特定のブランチだけをPullしたい場合は`Branches to Build`のフィールドにブランチ名を入力することで指定できます。

![](/assets/images/jenkins_screenshot/jenkins_gitcredential.png)

他はデフォルト値で、GitのIDとパスワードだけを入力して`Add`を押せば終了。こちらで入力した認証情報は全域設定なので、他のJobでも使えます。

認証情報まで無事入力ができたら、`Save`を押して一旦作業を保存しましょう。そうすると、以下のような画面に戻ります。

![](/assets/images/jenkins_screenshot/jenkins_jobsaved.png)

## ビルドしてみる(1)

これまででGitからPullするというタスクの設定は終わっています。ちゃんとタスクが意図通りに動作するかを検証するため、ビルドしてみましょう。左のメニューから`Build now`をクリックすると、しばらくして左下の`Build History`という領域に初めてのビルドが表示されることを確認できます。`#1`の所に青い丸が付いていたらビルドが成功したという意味です。不安定だったら黄色、失敗だと赤の丸がつくのでビルドの状態を確認できます。

ビルドに成功したら、`#1`をクリックしてビルドの詳細を確認します。以下のような画面が現れます。

![](/assets/images/jenkins_screenshot/jenkins_gitjob1.png)

実際のビルドでどんな処理が行われたかを確認するには、左のメニューから`Console Output`をクリックします。Linuxのコンソールのような画面で表示されます。

![](/assets/images/jenkins_screenshot/jenkins_gitjob2.png)

ちゃんと指定した通り、Gitのリポジトリ（ブランチはマスター）からPullしてきたことを確認できます。コミットメッセージまで出力してくれてますね。

Linuxのコンソールから`/var/lib/jenkins/workspace/`配下のフォルダを直接確認することもできますが、JenkinsのWebコンソールからもJobのワークスペースを確認することができます。左のメニューで`Back to Project`をクリックし、`Workspace`をクリックすると以下のような画面が現れます。

![](/assets/images/jenkins_screenshot/jenkins_gitjob3.png)

実際GitのPullが正しく行われ、フォルダとファイルが生成されたことをこちらで確認できます。

## Gradleの設定

それでは次に、Gradleによるビルドの設定です。歯車のアイコンの付いた`Configure`をクリックし、Jobの設定画面に戻ります。GradleでJarファイルをビルドするのが目的なので、GitでPullした後のタスクになりますね。

`Build`のタブに移動し、`Add build step`をクリック、ドロップダウンメニューから`Invoke Gradle script`を選択します。そして`Advanced...`をクリックしましょう。

![](/assets/images/jenkins_screenshot/jenkins_gradleconfigure1.png)

ここで`Invoke Gradle`には前回のポストで入れといたGraldeをドロップダウンメニューで選択します。

そして次に、`Use Gradle Wrapper`の下のTasksに`bootJar`を入力します。このコマンドで実行可能なJarファイルが生成されます。また、`bootJar`コマンドでは`build.gradle`のパスを確保する必要があるので（GraldeがWorkspaceのJob名のフォルダで実行されるので、その同じパスに`build.gradle`がある場合でないと必須です）下の`Build File`にちゃんとパスを書きます。

もちろんGradleのコマンドではビルドファイルのパスを指定するオプションもあるので、`bootJar`だけでなく`-b SpringBootDemo/build.gradle bootJar`のようにコマンドを書いても同じくビルドはできます。

![](/assets/images/jenkins_screenshot/jenkins_gradleconfigure2.png)

これでGradleでのビルド設定は終わり。次にGitの時と同じく、検証してみます。

## ビルドしてみる(2)

ビルドの手順もGitの時と変わりません。`Build Now` → `#2` → `Console Output`の手順でビルドのコンソール出力を確認します。

![](/assets/images/jenkins_screenshot/jenkins_gradlebuild.png)

無事Gradleでのビルドが終わりました。それではJarファイルがちゃんとできたか確認しましょう。

![](/assets/images/jenkins_screenshot/jenkins_gradlebuildjar.png)

ちゃんと`build/libs`フォルダにJarファイルができたことを確認できます。

## 最後に

Jenkinsでできることはこれだけではありません。ビルド終了後にデプロイのJobを作り、そのJobを実行するように連携もできれば、JUnitとの連携でテストを行えうように設定も可能です。なので今後のポストでは出来上がったJarファイルをテストしデプロイする方法について書きたいと思います。

また、Jobの設定で`Build Environment`タブにある`Delete workspace before build starts`というオプションをチェックしておくのも良いと思います。Jobの実行時にWorkspaceフォルダをまず掃除してからタスクを実行していきますが、これによって前回のビルドで生成されたファイルによる異常動作を防げられますので。

仕事で触れた部分が少ないのでまだ私も全部の機能を試した訳ではありませんが、Jenkinsは各種プラグインとその組み合わせによって可能性は無限大に違いのではないかと思うくらい良いツールです。また色々研究したいものですね。

[^1]: JenkinsのJobにおいてのビルドは、Jobのバージョンに近いイメージです。特定時点のJobを設定〜実行までの単位をビルドと言います。Javaのビルドとは少し違う概念なので混同しないようにしましょう。