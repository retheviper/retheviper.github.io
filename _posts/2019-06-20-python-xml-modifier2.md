---
title: "Pythonでxmlファイルを操作する(2)"
date: 2019-06-20
categories: 
  - python
photos:
- /assets/images/sideimage/python_logo.jpg
tags:
  - xml
  - python
---

[前回のポスト](../python-xml-modifier1)で、xmlファイルを操作するスクリプトを紹介しました。しかし、仕事でそのスクリプトをしばらく使わない方針となったため実際使うことがなく放置していましたが、方針が変わり作成しておいたスクリプトを実際適用してみるととある問題が出てきました。最初はスクリプトが問題だろうとは思わず、原因究明にだいぶ時間がかかりましたが、なんとか解決できた今はホッとしています。やはりコーディングという行為は設計通りの実装よりも変な挙動をしていないか確認するのが大事なのではないかと、今回も思いました。

それでは具体的にどんな問題があり、どうコードを改善したかを今回のポストで述べていきたいと思います。厳密にいうとバグというよりは、詳細設計の段階でミスを起こしたという表現が正しいのかもしれませんが、理由はどうあれ当初の計画通り動かないコードを書いたことは反省しないといけないなと思うきっかけとなりました。これでまた成長できたと言えたらいいですね。

## 旧スクリプトの問題

前回のポストで紹介したスクリプトは、一見思い通りに動いているように見えました。実際テストをしてみたときも、指定したエレメントのテキスト部はちゃんと変えてくれてましたね。しかし問題は、処理すべきxmlファイルのエレメント数が想定よりも多かったというところから発生しました。つまり、旧スクリプトではSELECT文を発行するDBのエレメントが一つ、INSERT文を発行するDBのエレメントが一つというシンプルな構成になっていましたが、今回は複数のエレメントを処理しなければならない状況となりました。

それを旧スクリプトで処理しようとすると、一つのエレメントを書き換えたあと、残りのエレメントは無視して次のファイルに処理が移行していったのです。それがわからないまま処理が終わった時点で設計通りに動作してくれていると信じ、処理の終わったファイルを使うとエラーが発生した、というのがこのスクリプトを改修するきっかけとなりました。

まず原因をわかったので、スクリプトの目標を修正します。今回の目標は「条件と一致する全エレメントの修正」です。

## コードを改修する

目標に合わせてコードを修正すると同時に、些細な問題も改善することにします。前回のスクリプトではフォルダー内のファイルをリストとして取得するために`glob`のモジュールを使いました。わずかのコードで再帰的に下位フォルダー内のファイルまで収集してくれるので便利だったのですが、globのrecursiveオプションはPython3.5以上でしか使えないという問題があります。普段からPython3を使っていたなら問題はあまりないはずですが、他に使っているPythonのスクリプトは全てPython2を基準に作成されています。なのでこれをPython2でも使えるオプションに変えることにします。

そしてメインとなる改善点としては、`find`を`findall`に変えることにします。まずDBコネクションのエレメントを全部取得し、ここで`if`文を使えばDBのコネクション名によって分岐処理ができるはずです。また、今回変えたいDBコネクション名は`From_PostgreSQL_01`のようにアンダースコアで連番が付いているものを`From_PostgreSQL`のように連番だけ外すということなので、その仕組みも考えておきます。`replace`を使う方法もありますが、これなら全てのケースに対して条件を書かなければならないですし、条件指定の例によっては重複の可能性もあります。なので`rsplit`[^1]を使い、アンダースコアを基準に元の文字列を分割した後元のテキストを代替することにします。

これらの要件定義から変わったことは以下となります。

```python
# -*- coding: UTF-8 -*-

import xml.etree.ElementTree as ET
import os

# 名前空間（prefix）をマップで宣言
ns = {'fb': 'http://builder',
      'fe': 'http://engine',
      'mp': 'http://mapper'}

# xmlファイル名を再帰的に取得(Python2向け)
fileList = []
base_dir = os.path.normpath('./baseFolder') # 検索するディレクトリの起点を設定

for (path, dir, files) in os.walk(base_dir):
    # xml2というファイルは消し、xmlファイルだけ書き換えしたいので分岐をかけ、xmlファイルだけをリスト化する
    for fname in files:
        if ('.xml2' in fname):
            fullfname = path + "/" + fname
            os.remove(fullfname)
        elif ('.xml' in fname and not '.xml2' in fname):
            fullfname = path + "/" + fname
            fileList.append(fullfname)

# 取得したファイルを巡回しながらコネクション名の書き換え処理
for fileName in fileList:
    # ファイルをパーシング開始
    tree = ET.parse(fileName)

    # INSERTのコンポーネントのコネクション名に'_01'などの文字がついていると取る
    PutConnections = tree.findall("fe:Flow/fe:Component[@type='RDB(Put)']/fe:Property[@name='Connection']", ns)
    for PutConnection in PutConnections:
        if ('_0' in PutConnection.text):
            PutConnection.text = PutConnection.text.rsplit('_', 1)[0] # rsplitで分割し、その結果物を元のテキストに入れる

    # SELECTのコンポーネントのコネクション名に'_01'などの文字がついていると取る
    GetConnections = tree.findall("fe:Flow/fe:Component[@type='RDB(Get)']/fe:Property[@name='Connection']", ns)
    for GetConnection in GetConnections:
        if ('_0' in GetConnection.text):
            GetConnection.text = GetConnection.text.rsplit('_', 1)[0] # rsplitで分割し、その結果物を元のテキストに入れる

    # prefixが変わることを防止
    ET.register_namespace('fb', 'http://builder')
    ET.register_namespace('fe', 'http://engine')
    ET.register_namespace('mp', 'http://mapper')

    # 書き換え処理
    tree.write(fileName, 'UTF-8', True)
```

## 最後に

ファイル取得部は`glob`オプションで簡単にできたことに比べ少し複雑になっています。`os.walk()`で起点のディレクトリを指定してパスとファイル名を取得します。ただ`os.walk()`だとファイル名とパスは分離されるのでそれをつなぐ作業が必要ですね。そこで処理するファイルのリストに入れたり消したりする処理を加えます。これで以前の`glob`と似たような挙動ができます。

そして`rsplit('_', 1)`で、まず`From_PostgreSQL_01`という文字列は`From_PostgreSQL`と`01`に1回だけ分割されます。そして分割された文字列は配列になるので[0]を指定すると意図通り`From_PostgreSQL_01`が`From_PostgreSQL`に代替されます。また`find`を`findall`に変えただけで条件に一致する全エレメントをファイル内で探してリストにしてくれます。その中でループ処理するだけですね。これがわからなかった時は以前のコードにさらにループをかけたりして失敗していましたが、意外と簡単な解決策があったものです。

これで完成されたコードは意図通りに動いてくれました。あとで変動があっても少しだけ変えればいいので個人的には満足しています。より綺麗な書き方はあるかも知れませんがね。そして教訓として、いつもテストは大事だなということを改めて覚えられました。常に確認と確認です。

[^1]: `rsplit`と`split`の違いは、方向です。前者が文字列の右側を基準に分割するなら、後者は左側からです。今回は文字列の末尾の連番を取りたいので、`rsplit`を選びました。