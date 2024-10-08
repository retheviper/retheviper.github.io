---
title: "Tera Termを使う"
date: 2019-06-24
categories: 
  - linux
image: "../../images/linux_terminal.webp"
tags:
  - terminal
  - linux
---

PuttyやMacのターミナルは使ってみたことがありますが(CLIの範囲まで行くとMS-DOSも)、[Tera term](https://ttssh2.osdn.jp/)を使ったことはなかったです。でも仕事ではAWSでLinuxのサーバーを立てて使っているのでSSH接続が必要となります。ここで私が初めてしたことはそのSSH接続ができるようにTera termのマクロを作ることでした。それまでは主に文書作業をしていたので、やっとコーディングのようなことができて嬉しく思いましたね。

他にも色々良いツールはあるのではないかと思いますが、マクロで簡単にシェルのコマンドが発行できたり、画面のカスタマイズが簡単だということが便利ですね。最近はWindows 10でもターミナルが使えるようになったりWSLというサービスができましたが、まだ会社で支給されるパソコンがWindows 10ではない場合にはこちらの方を使うのが便利だろうと思います。

それでは、設計と実際のコードでどうマクロを作ったかを紹介します。

## マクロの設計

全てのAWS EC2がそうなのかわかりませんが、今回の案件ではいわゆる「踏み台サーバー」を経由してからEC2サーバーに接続することができています。最初はその概念すらなかったのでなぜこんな構造になるのか疑問でしたが、それぞれのEC2サーバーが直接インターネットに繋がっている訳ではないのでこういう手続きが必要となるらしいです。なのでまず踏み台サーバーに接続し、そこからSSHコマンドを発行しそれぞれのEC2サーバーに接続できるようなマクロを作ります。

そして今回はバッチサーバー、WebAPサーバー、CIサーバーという三つの構成となるので踏み台からそれぞれのサーバーに接続できるようにしたいです。また、同時に複数のサーバーで作業する場合を踏まえてサーバーごとのターミナル画面を少しカスタマイズしたいです。少し調べてみると、マクロには接続の手続きを書き、マクロからあらかじめ画面の設定をしておいたINIファイルを読み込むようにすればいいらしいです。またCIサーバーではJenkinsのようなサービスもポートフォワードで使いたいのですが、その設定もこのINIファイルに記録されるみたいなのでマクロからCIサーバーに接続する場合に限定してポートフォワード設定が入ったINIファイルを呼ぶようにしたいです。

以上の要件を整理すると、マクロの仕組みは以下となります。

1. リストから接続先を選択する(選択肢は3つ)
2. まず踏み台サーバーに接続し、そこからさらに選択したサーバーへSSH接続する
3. 接続が終わると選択したサーバーごとのINIファイルを読み込む

INIファイルの設定はUIからターミナルとウィンドウ、フォント、ポートフォワードを予め設定し、それぞれの接続先に合わせ3つを用意しておきました。あとはマクロから読み込むだけです。では肝心のマクロのコードはどうなかったかというと、以下のようになります。

## マクロのコード

```bash
;==============================================
;; 踏み台ユーザーID／PW
;==============================================
USERNAME = '踏み台ユーサーID'
PASSWORD = '踏み台ユーザーPW'

;==============================================
;; 踏み台サーバーIPアドレス
;==============================================
HOSTIP = 'ここにIP'

;==============================================
;; 接続先別作業用ユーザーID
;==============================================
strdim WORKUSERLIST 3
WORKUSERLIST[0] = 'バッチサーバーのユーザーID'
WORKUSERLIST[1] = 'WebAPサーバーのユーザーID'
WORKUSERLIST[2] = 'CIサーバーのユーザーID'

;==============================================
;; 接続先別作業用ユーザPW
;==============================================
strdim WORKPWLIST 3
WORKPWLIST[0] = 'バッチサーバーのユーザーPW'
WORKPWLIST[1] = 'WebAPサーバーのユーザーPW'
WORKPWLIST[2] = 'CIサーバーのユーザーPW'

;==============================================
;; サーバーIP
;==============================================
strdim SERVERipLIST 3
SERVERipLIST[0] = 'バッチサーバーのIP'
SERVERipLIST[1] = 'WebAPサーバーのIP'
SERVERipLIST[2] = 'CIサーバーのIP'
 
;==============================================
;; リストに表示されるサーバー名称設定
;==============================================
strdim SERVERnameLIST 3
SERVERnameLIST[0] = 'バッチサーバー'
SERVERnameLIST[1] = 'WebAPサーバー'
SERVERnameLIST[2] = 'CIサーバー'
 
;==============================================
;; サーバー別INIファイル
;==============================================
strdim INILIST 3
INILIST[0] = '/BatchServer.INI'
INILIST[1] = '/WebAPServer.INI'
INILIST[2] = '/CIServer.INI'

;==============================================
;; 接続先ホスト選択画面
;==============================================
listbox 'サーバーを選択して下さい' '決定' SERVERnameLIST
if result >= 0 then
    SERVERIP = SERVERipLIST[result]
    WORKUSER = WORKUSERLIST[result]
    WORKPASSWORD = WORKPWLIST[result]
    INIFILE = INILIST[result]
else
    end
endif
 
;==============================================
;; INIファイルのパスの読み込み
;==============================================
getdir INIPATH
strconcat INIPATH INIFILE

;==============================================
;; 踏み台サーバーへの接続用コマンド組立て + 接続コマンド実行
;==============================================
PROXY = '-proxy=http://proxy.server.com:6000'
COMMAND = PROXY
strconcat COMMAND ' '
strconcat COMMAND HOSTIP
strconcat COMMAND ':22 /ssh /auth=password /user='
strconcat COMMAND USERNAME
strconcat COMMAND ' /passwd='
strconcat COMMAND PASSWORD
connect COMMAND

wait '$'

;==============================================
;; 接続先別SSH接続処理
;==============================================
SSHCOMMAND = 'ssh '
strconcat SSHCOMMAND WORKUSER
strconcat SSHCOMMAND '@'
strconcat SSHCOMMAND SERVERIP
sendln SSHCOMMAND

;==============================================
;; 初SSHログイン処理
;==============================================
wait 'Are you sure you want to continue connecting (yes/no)?' "'s password: "
if result = 1 then
    sendln 'yes'
    wait "'s password: "
elseif result = 2 then
    goto INPUTPWD
endif

:INPUTPWD
sendln WORKPASSWORD
wait '$'

sendln 'sudo su -'
wait 'sudo'

sendln WORKPASSWORD
wait '#'

restoresetup INIPATH
;==============================================
;; マクロ終了
;==============================================
end
```

## コードの説明

最初は踏み台サーバーに指定しといたユーザーID、PW、IPからコマンドを組み立ててへSSH接続する構造となっています。具現化したものをみるとstconcatで文字列を繋ぎコマンドを叩いています。また仕事ではプロキシサーバーも使っているので踏み台サーバー接続のコマンドに入れていますが、ない場合はそこだけを取り除くことになります。ただ踏み台サーバー接続の後に違うサーバーに接続するのがメインなので、これの前にマクロを起動した時点ではまずリスト画面から接続先の3つのサーバーを選ぶようになっていますね。

まず接続先のそれぞれのユーザーID、PW、IP、画面に表示するサーバー名を配列(strdim)として用意します。そして画面にはlistboxで選択できるリストを表示します。listboxで接続先を選択して決定ボタンを押すと、その結果がresultという変数に数字として入力されるのでそれを配列のインデックスとして踏み台サーバー接続のコマンドのように文字列を繋ぎSSH接続を行います。また、同じく配列で接続先ごとのINIファイル名を用意し、getdirからマクロの相対パスを取得、INIファイル名と結合したあとrestoresetupでINIファイルを読み込むようにします。(マクロファイルとINIファイルは同じフォルダーにある前提です)

コマンドの発行では、waitを使ってターミナルの反応を待ちます。これはシェルでのexpectと同じ機能をします。いきなりコマンドを連続で発行してもサーバー側の反応とずれると想定通りにならないことを防ぐためです。waitでサーバーに次のコマンドを発行できるかどうかを判断してからsendlnでコマンドを発行するようになっています。このwaitとifを組み立てることで初接続時に「本当に接続しますか？」という文が出力されても対応できるようにマクロを作ることができます。他にはそんなに難しい部分はないと思うので説明はここまでにします。

このマクロを`.ttl`という拡張子で保存し、Tera termから読み込むかマクロから直接実行するか(Tera termのインストール時にオプションとしてマクロを直接実行できるように設定できます)で終わりです。思ったより簡単ですね！

## 最後に

ifとwaitを活用してだいたいなんでもコマンドを発行できるので、ある意味シェルのようなこともできるのがこのTera termのマクロの魅力かと思います。ここのコードではマクロを起動することでrootまで行くようにしていますが、他に作業パスを変えたりとあるシェルスクリプトを実行させたりもできますね。構造的にはターミナルでコマンドを発行するようになっているだけですので。

それでは今回のポストはここまで。楽なSSH生活のため、みなさんもぜひTera termを使ってください。
