---
title: "Pythonでログを出したい"
date: 2019-06-15
categories: 
  - python
image: "../../images/python.webp"
tags:
  - log
  - python
---

コーディングにおいて、ロジックを組むこと異常に重要なことがあるとしたらそれは自分の書いているコードが正しいかどうかを確認することだと思います。まず仕様書などを書き、要件を満たしているかどうかをチェックしながら、コードを書いていきます。しかし、実行してみない限り思い通りにコードが動いてくれるかどうかは確認できませんね。そしてその動作が正しいかどうかを判別するためにはコードの中で扱っている変数やオブジェクトに正しいデータが入っているかどうかをみます。EclipseなどのIDEにはデバッグ機能があるので、チェックポイントを設定してコードの流れをモニタリングしながらそれを追うことができますが、それを利用できない場合もありますね。では標準出力という方法もありますが、実行後に記録を追うことができないという面で不安です。

そういう時にカスタムログを利用するのも一つの方法ですね。ちゃんとしたログを吐くように最初からロガーを作っておくと、自分の作ったプログラムが設計通りに動いているかどうかを判断できます。そしてプログラムが完成した後もネットワークの障害やデータの問題を確認するために有効活用できるという面もありますね。といっても、カスタムログを吐くロガーを書くということは難しいことのように思われます。最初ログを吐くようにしろという指示を受けて、私はどうしたらいいか？標準と言えるひな形はないのか？と悩みましたね。でもそんな難しく思うようなことはなくて、状況による分岐(メッセージやコードなど)を決め、それをテキストファイルとして出力するようにすれば良いだけの話でした。

それでは、実際のロガーをどう作ったかをこれからコードを通じて見せましょう。

## ロガーを作る

今回私た作りたいロガーの構造は簡単です。メインプログラムで呼び出された時に、引数としてログコード(分類番号的な)やジョブコード(作業の処理番号など)を受け、ログコードから出力するメッセージを決めます。ジョブコードは必須ではないが、ログコードは必須卯にしたいです。また、ログの書かれた時点を知りたいため日時もメッセージに入れます。この要件から出力されるログの形は以下となります。

> 2019/06/15 12:00:00 LOG0001: Job code 1234は正常終了しました。

また、ログファイルの最上段には`---- Job Log ----`を入れ、これがログファイルであることを示したいです。このように基本要件を決め、実装していきます。私が実装したコードの簡略化したバージョンは以下となります。

```python
# ログファイルの存在有無を確認するためのosと日付を残すためのdatetimeをインポート
import os
import datetime

# ログをかく関数(job_codeは与えられない場合があるのでデフォルト値を設定)
def writeLog(log_code, job_code=''):
    # ログファイルのパスを指定(絶対経路)
    log_path = '/log/program_log.log'

    # ログコード判定によりメッセージを決める
    if (log_code == 'LOGI001'):
        log_message = 'Job code ' + job_code + 'を実行します。'
    elif (log_code == 'LOGI002'):
        log_message = 'Job code ' + job_code + 'は正常終了しました。'
    elif (log_code == 'LOGW001'):
        log_message = 'Job codeがありません。プログラムの起動に失敗しました。'
    elif (log_code == 'LOGE001'):
        log_message = 'Job code ' + job_code + 'は異常終了しました。'
    else:
        log_message = '原因を特定できないエラーが発生しました。'

    # ログファイルが存在しない場合は先端にログであることを示すコードを含むファイルを生成
    if (not os.path.exists(log_path)):
        with open(log_path, mode='a') as log:
            log.write('----- Job Log -----\n')
    else 
    # ログファイルがすでに存在する場合は追記する
        with open(log_path, mode='a') as log:
            log.write(datetime.datetime.today().strftime('%Y/%m/%d %H:%M:%S') + ' ' + log_code + ': ' + log_message + '\n')
```

これでメインプログラムでロガーを呼び、そのタイミングでログコードやジョブコードを渡すだけでログは残されます。ログの種類を減らしたい、もしくは増やしたい場合はif文を修正するだけで対応できます。

## メインプログラムでロガーを呼び出す

メインプログラムでまずログを書く(ロガーを呼ぶ)タイミングを決めます。例えば今回はDBに接続するタスクがあったので、DBの接続に失敗した場合にロガーを書くようにしました。そのほかにも処理の開始や終了時のステータスなどに関してもログを残したいですね。

同じく簡略化したメインプログラムのコードは以下のような形です。先に作成して置いたロガーをインポートし、ログを出したい場合に呼び出すだけでいいです。ただロガーファイルの位置が実行するメインプログラムと違うパスにある場合はインポートの形が変わるので注意が必要です。ここではメインプログラムとロガーは同じパスから実行される場合を想定して実装しています。それでは下のコードをどうぞ。

```python
# 簡略化したメインプログラム
# 終了時のリターンコードと起動時の引数のためsysを、ログを書くためにロガーのwriteLog関数をインポート
import sys
from logger import writeLog

# 起動時にコマンドラインから引数を受けるための変数宣言
args = sys.argv

# 関数の例
def program(job_code):
    # 処理開始の時点でログを書く
    writeLog('LOGI001', job_code)
    # 正常処理時の挙動を書く
    try:
        print ('何かの処理を行う')
        # 正常終了のコードでログを書く
        writeLog('LOGI002')
        # 正常終了の場合は終了時のリターンコードが0
        sys.exit(0)
    # 処理に問題が起きた場合の処理を書く
    except:
        # 異常終了のコードでログを書く
        writeLog('LOGE001')
        # 異常終了の場合は終了時のリターンコードが9
        sys.exit(9)

# プログラムの起動部
if __name__ == '__main__':
    # 引数(ジョブコード)が入力されなかった場合のチェック
    if (args[1] in None):
        # 警告終了のコードでログを書く
        writeLog('LOGW001')
        # 警告終了の場合は終了時のリターンコードが1
        sys.exit(1)
    else:
        program(args[1])
```

## 最後に

どんなログを吐き出して欲しいかによってロガーを呼び出す位置を決めます。そして引数を渡すことでどんな場合にどんなログを出力するかを決められます。実装が難しい作業ではないですが、こうやってロガーを分離して必要な時だけ呼び出すことでメインプログラムのコードがいくら長くなっても関数を呼び出すだけで済むという効率的な構造となっています。そもそも関数というものは、反復を減らすために書くものでもありますしね。

あと少し調べたところ、Pythonではモジュールとしてloggerをすでに提供しているとのことです。標準出力としてログのレベルを指定して文字列を出力できるらしいですね。これを応用してファイルに描くようにコードを書くこともまた良い方法になるのではないかと思います。

仕事でコードを書きながら、大事なのはどんなコマンドと関数名を覚えているかではなく、どんな構造で実装していくかなのではないかと思うようになりました。最近は本でもインターネットでも基礎文法に関する講座やTipは簡単に得られますが、こういう設計の仕方に関しては、やはり実戦ではないとなかなか得られない分野の知識ではないかとも思います。またこれからどんなコードを書くようになるだろうか、色々楽しみです。
