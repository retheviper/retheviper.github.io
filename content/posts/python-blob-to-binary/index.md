---
title: "PythonでDBの処理がしたい"
date: 2019-06-30
categories: 
  - python
image: "../../images/python.jpg"
tags:
  - oracle
  - postgresql
  - blob
  - python
---

あまり詳しくない上に、何か新しいプログラムを作り出すこととは少し距離があるなと思ってDBにはあまり興味を持ってなかった私ですが、ITの業界で働きながらDBと接しないのは難しいことです。そしてそうやって接するDBはまた優しくない課題になってきますね。この度の仕事でも主にPythonを使ってスクリプトを書いたものの、そのタスクのメインとなるものはDBとの連携でした。なので今回のポストではどうやってPythonでDBに繋ぎ、SQL文を発行し、そのデータを扱ったかについて述べたいと思います。

今回私に与えられたタスクは、DBからバイナリーのデータを抽出して、それをAWSのS3[^1]にアップロードするスクリプトを作ることでした。また、アップロードが終わった時点でテーブルを更新する必要がありますが、抽出の対象となるDBと更新の対象となるDBがそれぞれ違うものでした。片方はOracleで片方はPostgreSQL。そしてロガーを入れ各DBやS3の接続に失敗するなどの時にはログで分かるようにしたり、DBの接続情報を外出しにして外部ファイルから読み込むようにするなどの条件がありました。ロガーについては[前回のポスト](../../../06/15/python-logger/で述べましたので、今回は省略とさせていただきます。他にバイナリーデータを抽出するには引数を受け、最終更新日を確認した上でそれより後のファイルを保存する、S3にアップロードする際はテーブルのカラムからパス名を決める、画像のアップロードの後には最終更新日を更新する、などの条件がありました。

要略すると、今回作るスクリプトの仕様は以下となります。

1. コマンドラインで引数を受ける
2. 引数からPostgreSQLのテーブル(1)を参照し、引数に当たる作業の最終更新日を取得する
3. OcaleDBのテーブルから最終更新日以後のバイナリーデータをSELECTしてJPGファイルとして保存する
4. PostgreSQLのテーブル(2)からファイルの保存先のパスとなるカラムを取得する。
5. S3へJPGファイルごとのフォルダーに画像をアップロードする
6. PostgreSQLのテーブル(3)にアップロードされたJPGファイルの情報を記録する
7. PostgreSQLのテーブル(1)最終更新日を更新する
8. 正常終了したらexitコード0となり、例外が発生するとexitコード9を出力して終了する

PostgreSQLの場合は参照するテーブルが多く、処理の順番があるので少し複雑にも感じられますが、まずはDBとの連動を試しみスクリプトを組むことにしました。

## PythonでDBを連動する

案外、PythonでDBに接続することはそんな難しいことではなかったです。それぞれ違うモジュールを使っていてコマンドにも違いがあるわけですが、検索してみると例題が多かったですね。Oracleの場合はホスト名、ポート、サービス名、ユーザー名、パスワードを用意します。使うモジュールは`cx_Oracle`です。

PostgreSQLの場合は、`psycopg2`を使います。接続に必要となる情報はホスト名、ポート、DB名、ユーザー名、パスワードです。ただし、モジュールは両方`pip`からインストールできますが、psycopg2の場合はライブラリー依存性があるのでAmazon Linuxを基準に`postgresql-develop`をyumからインストールする必要があります。他は触れたことがないのでよくわかりませんが、CentOSも同様かと。psycopg2のインストールに問題が生じたらエラーメッセージを確認して、必要なPosgreSQLのライブラリーをインストールしましょう。

二つのDBに接続するための手順には少し違いがありますが、接続した後の処理は同じです(本当はSQL文の文法も少し違うようですが…)。それではOracleとPostgreSQLの接続方法と接続以後の処理を分けて説明しましょう。

### 接続(Oracleの場合)

```python
import cx_Oracle

tns = cx_Oracle.makedsn('ホスト名', 'ポート', 'サービス名')
connect = cx_Oracle.connect('ユーザー名', 'パスワード', tns)
```

cx_Oracleの`makedsn`にホスト名、ポート、サービス名を入れます。そしてその情報を`connect`を使う際にユーザー名、パスワードと共に入れます。これでOracleの接続設定は終わりです。

### 接続(PostgreSQLの場合)

```python
import psycopg2

connect = psycopg2.connect('host=' + 'ホスト名' + ' port=' + 'ポート' + ' dbname=' + 'DB名' + ' user=' + 'ユーザー名' + ' password=' + 'パスワード')
```

PostgreSQLの場合はより簡単です。Linuxで使うコマンドラインツールの`psql`みたいに、接続に必要な情報を文字列として並び`connect`すれば終了です。

### DB処理の共通部分

```python
# 実際DBに接続してカーソルを取得
cursor = connect.cursor()
# カーソルからSQL文の実行
cursor.execute('発行したいSQL文')

# SQL文を実行した結果を取得
# 一件だけの結果が必要な時
result = cursor.fetchone()
# 結果を1000件づつフィッチして処理したい場合
result = cursor.fetchmany(1000)
# 結果を全部フェッチしたい場合
result = cursor.fetchall()

# カーソルをクローズする
cursor.close()
# SQL文をコミットする(INSERT/UPDATEなどの変動)
connect.commit()
# コネクションを切る
connect.close()
```

接続したあとはカーソルを取得し、そのカーソルでSQL文を発行します。SQL文の実行結果を取得するには`fecth`を使いますが、処理したい結果の規模によって三つの選択肢があります。あと、大量のデータを処理したい場合はなるべく`fetchall`よりは`fetchmany`でフェッチサイズを指定して使う方が良いらしいです。データが多すぎると性能に影響が出ますので。フェッチまで終わって取得した結果は、Pythonでは一つのレコードを配列にして、それらを集めたリストになります。コードで言いますと以下のような形となります。

```python
# 'SELECT column_1, column2 FROM TABLE'の場合

result = cursor.fetchmany(1000)

for row in result: # 結果は1000件のリストとなる
    print row[0] # column_1の内容が出力される
    print row # column_2の内容が出力される

```

これでDBの連動の基礎は終わりです。それでは早速、スクリプトのコードをいかに公開します。

## 画像ファイルをS3へアップロードするコード

```python
# -*- coding: UTF-8 -*-
import os, sys, cx_Oracle, psycopg2, json, boto3, datetime, shutil

function_code = '引数で受け取る'

# 以下、DB環境情報(DBConnection.confから読み込む)
HOST_POST = 'PostgreSQLの接続ホスト名'
PORT_POST = 'PostgreSQLの接続ポート'
DB_NAME_POST = 'PostgreSQLのDM名'
USER_POST = 'PostgreSQLのユーザ名'
PWD_POST = 'PostgreSQLのパスワード'
HOST_ORAC = 'Oracleの接続ホスト名'
PORT_ORAC = 'Oracleの接続ポート'
SERVICE_NAME = 'Oracleのサービス名'
USER_ORAC = 'Oracleのスキーマ名'
PWD_ORAC = 'Oracleのパスワード'

# コマンドライン引数
args = sys.argv

# 環境関連変数
imageFolder = '/tmp/images'

# 差分連携のための前処理
def GetProcdate(args):
    global function_code
    function_code = args[1]
 
    try:
        # PostgreSQL接続
        print ('>> Starting Job. The function Code is: ' + function_code)
        connect = psycopg2.connect('host=' + HOST_POST + ' port=' + PORT_POST + ' dbname=' + DB_NAME_POST + ' user=' + USER_POST + ' password=' + PWD_POST)
        cursor = connect.cursor()
    except:
        print ('>> Unable to connect PostgreSQL. quitting.')
        sys.exit(9)

    selectSQL = "SELECT last_date FROM date_table WHERE function_code='%s'" % (function_code,)
    cursor.execute(selectSQL)
    result = cursor.fetchone()

    # 結果があった場合最終更新日を変数として保存
    if (result is not None):
        last_date = result[0]
    else:
        print ('>> No data matches with ' + function_code + '. quitting.')
        sys.exit(1)

    cursor.close()
    connect.close()

    # 次の処理へ移行
    GetImageFromTable(last_date)

# OracleにSQL文を発行してファイルを読み込み保存
def GetImageFromTable(last_date):
    global imageFolder
    try:
        # Oracle接続開始
        tns = cx_Oracle.makedsn(HOST_ORAC, PORT_ORAC, SERVICE_NAME)
        connect = cx_Oracle.connect(USER_ORAC, PWD_ORAC, tns)
        cursor = connect.cursor()
    except:
        print ('>> Unable to connect Oracle DB. quitting.')
        sys.exit(9)

    # Oracleのテーブルから商品画像ファイル取得
    cursor.execute("SELECT image_data, image_name FROM WHERE >=(:last_date)", {'last_date': last_date})

    # フォルダがなかったら作成・あったら削除
    if os.path.exists(imageFolder):
        shutil.rmtree(imageFolder)
    else:
        os.mkdir(imageFolder)

    # 1000件づつ取り出し
    while True:
        rows = cursor.fetchmany(1000)
        
        # 結果がない場合はループから出る
        if len(rows) == 0:
            break

        # 画像ファイル置き場に画像ファイル保存(イメージネーム.jpg)
        for image in rows:
            fileNameS = imageFolder + '/' + str(image[1]) + '.jpg'
            imageFile = open(fileNameS, 'wb+')
            imageFile.write(image[0].read())
            imageFile.close()
            counter = counter + 1

    # 保存されたファイルの数をカウント
    fileCount = len([name for name in os.listdir(imageFolder) if os.path.isfile(os.path.join(imageFolder, name))])
    print ('>> As result: ' + str(fileCount) + ' files were written.')
    print ('>> Finished job. Closing connection.')
    cursor.close()
    connect.close()
    FileUploadToS3()

# S3へ画像ファイルを格納
def FileUploadToS3():
    # グローバル変数取得
    global imageFolder
    global function_code

    # ファイル名リスト(イメージネーム)取得
    files = os.listdir(imageFolder)
    item_cd_list = [os.path.splitext(f)[0] for f in files if os.path.isfile(os.path.join(imageFolder, f))]

    try:
        # PostgreSQL接続
        connect = psycopg2.connect('host=' + HOST_POST + ' port=' + PORT_POST + ' dbname=' + DB_NAME_POST + ' user=' + USER_POST + ' password=' + PWD_POST)
        cursor = connect.cursor()
    except:
        print ('>> Unable to connect PostgreSQL. quitting.')
        sys.exit(9)

    # 結果確認用のファイル数カウンター
    fileCount = 0

    for item_cd in item_cd_list:
        # アイテム管理用テーブルからコードと一致するレコード取得⇒ファイル格納パス用
        selectSQL1 = "SELECT file_path1, file_path2, file_path3 FROM item_table WHERE item_cd='%s'" % (str(item_cd),)
        cursor.execute(selectSQL1)
        resultMaster = cursor.fetchone()

        # 一致するレコードがなかった場合でも処理続行
        if resultMaster is None:
        print ('>> Result was 0. continue job.')
            continue

        # 結果から必要な情報を取得し変数に保存
        image_id = str(resultMaster[2])
        path = str(resultMaster[0]) + '/' + str(resultMaster[1]) + '/' + str(resultMaster[2]) + '/'

        # アイテムイメージ管理用テーブルからコードと一致するレコード取得⇒分岐処理
        selectSQL2 = "SELECT * FROM WHERE image_table image_id='%s';" % (image_id,)
        cursor.execute(selectSQL2)
        result = cursor.fetchone()
        uploadToS3(item_cd, path)
        fileCount = fileCount + 1
        print ('>> Processing file upload.')

    # 作業完了日時を更新
    updateProcSQL = "UPDATE date_table SET last_date = CURRENT_TIMESTAMP WHERE function_code='%s'" % (function_code,)
    cursor.execute(updateProcSQL)

    # 作業終了
    print ('>> As result: ' + str(fileCount) + ' files were uploaded.')
    print ('>> Finished job. Closing connection and quit.')
    cursor.close()
    connect.commit()
    connect.close()
    sys.exit(0)

# S3へ画像ファイルをアップロード
def uploadToS3(item_cd, path):
    global imageFolder
    print ('>> Upload image files to AWS S3.')
    bucket_name = 'image'
    s3 = boto3.resource('s3')

    try:
        # サーバーに保存されたファイルを指定したパスに保存(file_path1/file_path2/file_path3/1_1_YYYYMMDD.jpg)
        s3.Bucket(bucket_name).upload_file(imageFolder + '/' + item_cd + '.jpg', path + '/1_1_' + datetime.datetime.today().strftime('%Y%m%d%H%M%S') + '.jpg')
        print ('>> Image file uploaded. (Item code: ' + item_cd + ')')
    except:
        print ('>> Unable to upload files. quitting.')
        sys.exit(9)

# コネクション情報を入力ファイルから読み込む
def getDBConnection():

    # DBConnection接続情報ファイルのパスを取得
    pwd = os.path.dirname(os.path.abspath(__file__))
    dbConnection = pwd.rsplit("/", 1)[0] + '/env/DBConnection.conf'

    # ファイルがない場合異常終了
    if (not os.path.exists(dbConnection)):
        print ('>> Please check DBConnection.conf. quitting.')
        sys.exit(9)

    # ファイルを読み込む
    connectionInfo = open(dbConnection, 'r')
    lines = connectionInfo.readlines()

    # グローバル変数として保存するための宣言
    global HOST_POST
    global PORT_POST
    global DB_NAME_POST
    global USER_POST
    global PWD_POST
    global HOST_ORAC
    global PORT_ORAC
    global SERVICE_NAME
    global USER_ORAC
    global PWD_ORAC

    # 条件と一致するデータがある場合変数に保存
    for line in lines:
        if (len(line) > 1 and 'POSTGRESQL' not in line and 'ORACLE' not in line):
            result = line.split('=')
            if ('HOST_POST' in result[0]):
                HOST_POST = result[1].replace('\n','')
            elif ('PORT_POST' in result[0]):
                PORT_POST = result[1].replace('\n','')
            elif ('DB_NAME_POST' in result[0]):
                DB_NAME_POST = result[1].replace('\n','')
            elif ('USER_POST' in result[0]):
                USER_POST = result[1].replace('\n','')
            elif ('PWD_POST' in result[0]):
                PWD_POST = result[1].replace('\n','')
            elif ('HOST_ORAC' in result[0]):
                HOST_ORAC = result[1].replace('\n','')
            elif ('PORT_ORAC' in result[0]):
                PORT_ORAC = result[1].replace('\n','')
            elif ('SERVICE_NAME' in result[0]):
                SERVICE_NAME = result[1].replace('\n','')
            elif ('USER_ORAC' in result[0]):
                USER_ORAC = result[1].replace('\n','')
            elif ('PWD_ORAC' in result[0]):
                PWD_ORAC = result[1].replace('\n','')

# 起動部
if __name__ == '__main__':
    getDBConnection()

    # 引数がない場合は異常終了処理分岐
    if (len(args) < 2):
        print ('>> Please check arguement. quitting.')
        sys.exit(9)
    else:
        GetProcdate(args)
```

DBConnectionファイルを読み込む処理に関してはより良い書き方があるのではないかと思います…が、Python特有のGetter/Setterの書き方にあまり慣れてないゆえ、自分が理解できるコードとして書いた結果がこうです。グローバル変数がメソッド内で`global`宣言をしないと使えないというところがJavaに慣れていた自分にはかなり新鮮でした。逆にグローバル変数をそれぞれ違うメソッドではなるべく使うなという意味もあるような気もします。

あとはファイル書き込みがテキストだけでなく、バイナリーデータの場合も簡単にかけるということが魅力的ですね。Javaだったらストリームやバッファーをはじめとしてかなり複雑な書き方になっていただろうと思いますが(しかも私はテキストしか扱ったことがないので、同じやり方でバイナリーまでかけるかどうかわかりません)、Pythonではモジュールのインポートもなしで簡単にできますね。もちろん性能を考えるとJavaでかいた方がよかったかも知れませんが、こういう簡単さんがやっぱり生産性の向上に役立つのではないかと思います。

あと、やはりPythonはLinux親和的な構造が魅力的だと思います。処理の結果を判別するために`sys.exit()`を設定すると終了コードを簡単に確認できますし、`sys.args`を通じてコマンドラインから引数を受け入れることもできます。簡単かつLinux親和的なので、Linuxでのスクリプト処理が必要な場合はシェルスクリプトよりもPythonを用いる方が良いのでは、と思うようになりました。性能もシェルより良いところもあるという話もありますしね。

結果的にPythonを褒めるようなポストとなりましたが、私としてはプログラミングの初心者もしくはLinuxサーバーからスクリプトを組むことの多い開発者に魅力的な言語と思うので、皆さんにもぜひPythonを使って欲しいです。楽しく、楽に開発しましょう！

[^1]: Simple Storage Serviceの略で、その名の通りOSにマウントして普通のディスクのように使えるサービスだそうです。
