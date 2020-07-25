---
title: "Ansibleでサーバーを構築する"
date: 2019-07-05
categories: 
  - ansible
photos:
  - /assets/images/sideimage/ansible_logo.jpg
tags:
  - ansible
  - yaml
  - linux
---

この度は[Ansible](https://www.ansible.com/)を少し、使ってみる機会がありました。`Ansible`もまた一つの自動化ツールで、あらかじめタスクを指定することで複数の環境で適用できるという意味ではJenkinsと似たようなものでした。ただ違う点は、Jenkinsは主にデプロイとリリース、テストなどの自動化に特化しているのに対し、Ansibleはサーバーの自動構築にその目的があるということです。

つまり、Ansibleを使えば複数の環境を同じくセッティングできます。例えば私の経験した範囲では「開発」→「内結」→「外結」という流れになっていて、それぞれで使う環境が変わっていました。段階が変わるたびに同じくサーバーを構築するのは時間の無駄で、セッティングすべき項目が増えると人の手では様々な面でミスが起こり得るので、Ansibleで自動化したいというのが今回のタスク。

ちょっと調べてみると、Ansibleでは`Playbook`と呼ばれるYAMLファイル[^1]を使ってサーバーの設定を行います。深く入るとより複雑な動きもできるようですが、基本は接続先の情報(hostsファイル)とセッティングの値を保存したYAMLで起動します。できることはフォルダーやユーザーの作成、`yum`によるプログラムのインストール、シェルコマンド実行、ファイル転送などができて、うまく設定ができたら複数のサーバーを分散して使う時などに有効活用できそうな印象でした。

ただYAMLファイルとフォルダー構成などが少し複雑なので、ここでは一つ一つのファイルの設定を述べてみようと思います。以下のYAML設定は、AMAZON Linuxを基準に書かれています。

## Ansibleのインストール

Ansibleは`yum`、`brew`、`pip`、`apt-get`でインストールできます。簡単ですね！ただ場合によってはPython2やPIPが必要となるので、事前にインストールしておきましょう。

```bash
$ yum install ansible
$ brew install ansible
$ pip install ansible
$ sudo apt-get install ansible
```
使っている環境に適した方法でインストールしましょう。

## batchserver.yml

まずはサーバーごとのYAMLファイルを生成します。Ansible実行時に使われるのはこっちで、どのサーバーでどんな動作をするかを記述します。サーバーごとにしたいことに関しては別のYAMLファイルを用意し、共通、バッチサーバー、WEBAPサーバー用のYAMLファイルをそれぞれ作るというイメージです。

以下のコードは、バッチサーバーを想定した場合の例です。

```yaml
- hosts: batchserver
  become: true
  roles:
    - common
    - batch
```
`hosts`はhostsファイルに記述されたホストのことを意味します。ここで`batchserver`と書くと、Ansibleの実行時は自動的にhostsファイルからbatchserverというグルーブに属している全サーバーに接続し同じ動きをします。全グループに対して実行したい場合は`all`と書きましょう。

そして`become`を`true`に設定すると、接続先での全ての命令ががsudoとして実行されます。`roles`はサーバーで行う行動を指定したYAMLファイルのことを記述していて、私の場合はどんなサーバーでも共通的に実行したことを書いた`common`とバッチサーバーでだけ実行したい`batch`を区分しておいたので両方を書いています。

## hosts

このファイルは上述した通り、Ansibleで自動設定を行いたい接続先のことを書きます。

```text
[batchserver]
192.168.0.1 ansible_ssh_user=batchuser1
192.168.0.2 ansible_ssh_user=batchuser2

[webapserver]
192.168.10.1 ansible_ssh_user=webapuser1
192.168.10.2 ansible_ssh_user=webapuser2
```
基本はグルーブを指定して、ホストの情報を書いておくと自動的に分類されます。上に書いたYAMLファイルではまず`batchserver`だけを指定しているので実行時に`webapserver`のグループは無視されます。そして`ansible_ssh_user`はその名の通りAnsibleでSSH接続する時に使われるユーザー名を指定しています。もちろんこうしなくても実行時にユーザー名やパスワードを入れることもできます。

## roles / batch / tasks / main.yml

ここは`batchserver.yml`で指定した`roles`で、実際どんなことがしたいかを記述するYAMLファイルが入ります。`roles`は同盟のフォルダーの下に作っておいたものしか実行できませんので注意してください。

本格的にやりたいことを書くファイルなので、どんな行動が指定できるかまず参考にしてください。

```yaml
  - name: Create user groups
    group:
      name: "{ { item.group_name } }" # マークダウンの仕様のためスペースを入れているが、実際はスペースなし
      gid: "{ { item.group_id } }"
    with_items:
    - { group_name: 'group01', group_id: '101' }
    - { group_name: 'group02', group_id: '201' }

  - name: Create users
    user:
      name: "{ { item.user_name } }"
      password: "{ { item.user_passwd } }"
      uid: "{ { item.user_id } }"
      group:  "{ { item.user_group } }"
      shell: /bin/bash
    with_items:
    - { user_name: 'user01', user_group: 'group01', user_id: '101', user_passwd: 'user01' }
    - { user_name: 'user02', user_group: 'group02', user_id: '201', user_passwd: 'user02' }

  - name : Create folders
    file:
      path={ { item } }
      owner=user01
      group=user01
      mode=0755
      state=directory
    with_items:
    - /user01

  - name: Package install by yum
    yum:
      name: "{ { packages } }"
    vars:
      packages:
        - python2-pip
        - postgresql
        - postgresql-devel
 
  - name: Upgrade pip by shell command
    shell: bash -lc "pip install --upgrade pip"

  - name: Install python modules
    pip:
      name: "{ { item } }"
      executable: pip
    with_items:
      - cx_Oracle
      - psycopg2
      - boto3
      - paramiko

  - name: Copy files
    copy:
      src= { { item.source } }
      dest= { { item.dest } }
      owner=root
      group=root
      mode=0755
    with_items:
      - { source: etc/somefile.zip, dest: /etc/somefile.zip }
```

上から順番に、ユーザーグループ作成、ユーザー作成、フォルダー作成、`yum`でパッケージインストール、シェルコマンドの実行、ファイル転送となります。結局はSSH接続してシェルスクリプトを実行するようなものですね。でもYAMLファイルにより簡単に設定できるということが良いところではないかと思います。なんども繰り返して実行しても良いですしね。

ただSSH接続したあとはYAMLファイルの上から一行づつ読みコマンドを実行していくので、実行したいことの順番には気をつける必要があります。例えば当たり前なことなんですが、ユーザーグループを作成する前に特定のグループにユーザーを作成するとかはできないので注意してください。

## roles / batch / files

このフォルダーには転送に使いたいファイルをおきます。例えば上の`main.yml`に書いた`etc/somfile.zip`を転送したい場合は、このフォルダーの配下に同じパスのファイルを置きます。もちろん複数のファイルを転送することも、それぞれ違うフォルダーに分けておくことも可能です。

## roles / common / tasks / main.yml

このファイルはどんなサーバーでも共通的に実行したいコマンドを集めています。

```yaml
  - name: Upgrade all packages by yum
    yum: name=* state=latest

  - name: Install openjdk 11
    shell: bash -lc "amazon-linux-extras install java-openjdk11"

  - name: Correct java version selected
    alternatives:
      name: java
      path: /usr/lib/jvm/java-11-openjdk-11.0.2.7-0.amzn2.x86_64/bin/java
```
このファイルでやっていることは、`yum`による全パッケージのアップデートと、OpenJDKのインストールです。JDKをインストールするだけではサーバーでJava実行時の基本バージョンがOpenJDK11にならないのでAlternativeからJavaのバージョンを選択するところまで入れています。同じやり方でPython3をインストールしてAlternativeで基本実行のバージョンを指定するなどのこともできます。

ここまでくるとAnsibleによる基本設定は終わり。難しくないですね！(深く入れば難しくなりそうですが)

## 実行する

それではPlaybookが用意されたので実行します。以下のコマンドで実行できます。

```bash
# 一般的な実行
$ ansible-playbook server.yml -i hosts
# Dry runの場合(Playbookの文法チェック用)
$ ansible-playbook server.yml -i hosts -C
# SSH接続ユーザー名を入れる場合
$ ansible-playbook server.yml -i hosts -u hostuser
```
こちらも簡単ですね。実行時にSSH接続するユーザーのパスワードを要求される場合がありますが、これはあらかじめSSH接続するユーザーの公開鍵を登録しておくことで回避できます。`sudo`の場合は接続先で`visudo`から`NOPASSWD`を設定しておくと便利です。

## 最後に

どうでしたか。最近はなんでも自動化が進んでいて、JenkinsとAnsibleがあればサーバー構築から作成物のデプロイまで簡単にできる環境を構築できるので、ますます生産性が上がりそうな気がします。まだ手動でサーバーの構築をやっている方にはぜひ一度使ってみてくださいとオススメしたいですね。

それでは今回のポストはここで終わり。また会いましょう！

[^1]: データフォーマットの一種で、その構造がJSONとかなり似ています。ただJSONと比べてみるとよりマークアップ言語に近い感じですね。多少癖はあるものの、JSONよりは直観的な表現ができます。