---
title: "PyQtで字幕翻訳ツールを作ってみた"
date: 2022-05-01
categories: 
  - python
image: "../../images/python.jpg"
tags:
  - python
  - qt
---

最近友人がYoutubeのチャンネルを始めました。主に海外のYoutubeのチャンネルにアップされてあるドラマのメイキングやインタビュー動画などに字幕をつけて配信するというチャンネルですが、そこで毎回字幕を作るのはかなりめんどくさい作業なので、ちょっとしたコマンドaliasとして作成して提供することにしました。元の動画には原語の字幕がついていて、編集用ツールでそれを読み込むことができるらしいので動画だけでなく字幕もダウンロードするコマンドを作成しています。これなら翻訳した字幕ファイルを編集ツールで読み込んで使えるので、かなり作業量が減りました。

ただ、チャンネルの登録者数が増えながら、中には色々な国からの人もいたようで、友人から「自動翻訳を使って多国語の字幕を追加できる手段が欲しい」という依頼が来ました。簡単なツールを作れば良さそうだったので、早速ツールを作って公開することになったので、今回はそのツールを作った話をしたいと思います。

## 設計

まずは友人の要望と、技術スタック、アプリの基本設計を行いました。といっても、複雑なことはあまりしたくなかったので、（友人の）やりたいことをベースに、自分がやってみたいことを混ぜたくらいのレベル感です。

### 字幕

[youtube-dl](https://youtube-dl.org/)や[yt-dlp](https://github.com/yt-dlp/yt-dlp)を使ってYouTubeの動画をダウンロードする際、元の動画に字幕がついてある場合はオプションを追加することで字幕も落とせます。Youtubeで字幕をダウンロードすると、多くの場合ファイル形式が[WebVTT](https://developer.mozilla.org/ja/docs/Web/API/WebVTT_API)になるので、とりあえず`.vtt`形式のファイルに対応する必要があると思いました。他のフォーマットまで対応するには時間がかかりそうだったので、まずはこれのみにします。

幸い、実際の字幕ファイルを落としてみたところ、フォーマットはシンプルで([SubRip](https://ja.wikipedia.org/wiki/SubRip)とほぼ一緒)、翻訳は字幕が表示される時間の次の行の文字列と、その行のインデックスとともに取ればなんとか入れ替えができそうな気がしました。実際のファイルは以下のようなフォーマットで作成されています。

```vtt
WEBVTT

00:01.000 --> 00:04.000
液体窒素を絶対に飲まないでください。

00:05.000 --> 00:09.000
- それはあなたの胃に穴をあけます。
- あなたは死ぬ可能性があります。
```

### 言語翻訳API

翻訳についてはGoogle翻訳もありますが、友人の要請によって[Papago](https://papago.naver.com/)を使うことにしました。幸いAPIを無料で利用できて、PythonのサンプルコードやAPIの詳細も提供されていたのであまり利用は難しくはい感じでした。事前にPostmanでAPIを実行してみてからこちらを採用することにしました。

### 技術

まずPythonで作ることにしました。この規模の小さいアプリは、Kotlinがメインの私でもPythonで書きたいと思っています。JVMだと起動が遅くなるのもあり、テキストを扱うだけでCPUの処理がクリティカルな作業でもないためパフォーマンスを考慮する必要もあまりありません。そして既にPythonでファイルを読み込んだり、ファイルを編集するなど色々とCLI用のツールを作っていたので「ファイルを読み込んで、編集して、保存する」という最重要の機能がすぐに実装できそうと思いました。

ただ、自分が使うものならCLI上でも十分使えますが、友人が使うものなのGUIを実装することにしました。Pythonは`.py`ファイルで渡すと、Pythonとスクリプトのdependencyのインストールが必要となるなどエンジニアではない人が触るには不便なところがあるので、成果物は[PyInstaler](https://pyinstaller.org/en/stable/)などで実行できるバイナリにします。

GUIに関しては[PySimpleGUI](https://pysimplegui.readthedocs.io/en/latest/)を使ったことがありますが、今回は違うフレームワークを使ってみたいと思いました。PythonのGUIフレームワークといえば[tkinter](https://docs.python.org/ja/3/library/tkinter.html)や[Kivy](https://kivy.org)、[wxPython](https://www.wxpython.org/)、[Libavg](https://www.libavg.de/site/)など様々なものがあるのですが、中でも[Qt](https://www.qt.io/)がPythonだけでなく色々と使われているので、PythonのコードだけでなくC++のコードでも参考できる例が多いのではないかと思い[PyQt](https://wiki.python.org/moin/PyQt)の方を選びました。

プロダクションレベルのものを作るならまた色々と基準を持って検討してみたかも知れませんが、このような趣味レベルのコードを書く場合はなるべく手軽に書ける、サンプルの多いものを選ぶのがちょうど良いかもですね。効率というのも大事なので。

## コード

### ファイル読み込み

まずはファイルを読み込むところから始めます。`.vtt`ファイルはヘッダーに字幕の情報（言語コードなど）が書かれていて、その次からは字幕が表示される時間と字幕の文字列が繰り返されます。

素直に全行を読み込んでも良いかもですが、翻訳のAPIを呼び出すときに一日で利用できる文字数の制限というのがあったので、極力リクエストにのせるデータは減らしたかったです。なので、後で入れ替えするためのもとのファイルのデータと、翻訳のためAPIのリクエストパラメータにのせる二つのデータにわけで読み込むようにしました。いかがそのコードです。

```python
# get original contents from vtt file
def get_content(self, file_path: str):
    exported_contents: dict[int, str] = {}
    global source
    with open(file_path, 'r') as file:
        original_contents: list[str] = file.readlines()
        for index, line in enumerate(original_contents):
            content = line.strip()
            if source == '' and 'Language:' in content:
                source = content.split(':')[1].strip()
            if '-->' not in content and content != '':
                exported_contents[index] = content
    return original_contents, exported_contents
```

APIのパラメータには元の言語(source)と、翻訳したい言語(target)が必要となるので、Global変数として字幕のヘッダにある言語情報を`source`に格納、翻訳したい行はインデックスと文字列をdictionaryとして格納します。そして最終的に元のデータと、翻訳対象のデータを返します。

### 翻訳APIの呼び出し

抽出したデータのうち、翻訳対象のデータを渡してAPIを呼び出す部分です。APIには「1秒当たり10件のリクエストを許容」するという制約があり、翻訳した文字列を一行づつ送って翻訳してもらいながら、リクエストごとに120msを待つようにしました。基本的には[Requests](https://docs.python-requests.org/en/latest/)を使ってPOSTしているだけですが、もしエラーが返ってきたときはalertを表示してアプリを即終了するようにしています。

```python
# send translate request to papago and get translated message
def send_request(self, contents: dict[int, str]):
    translated_contents: dict[int, str] = {}

    headers = {
        'X-Naver-Client-Id': client_id,
        'X-Naver-Client-Secret': client_secret,
        'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
    }

    for index, content in contents.items():

        payload = {
            'text': content,
            'source': source,
            'target': target
        }

        response = requests.post(api_url, headers=headers, data=payload)

        if response.status_code == 200:
            body = response.json()
            translated_text: str = body['message']['result']['translatedText']
            translated_contents[index] = translated_text

            # wait for API's limitation(only 10 request per second allowed)
            sleep(0.12)
        else:
            msgBox = QMessageBox().critical(
                self,
                'Error',
                response.json()['errorMessage'],
                buttons=QMessageBox.StandardButton.Abort
            )
            if msgBox == QMessageBox.StandardButton.Abort:
                sys.exit(255)

    return translated_contents
```

### ファイルの保存

ロジックとしてはここが最後です。元のファイルのある場所に翻訳された字幕ファイルを保存するために、元のファイルのパス、翻訳したものに入れ替えるための元のデータ、そしてAPIで翻訳した結果のデータを渡します。

基本的に翻訳したデータにはインデックスがkeyとして入っているので、そのキーを持って元のデータを入れ替えていきます。ここでヘッダには元の言語のコードが入っているので、翻訳した言語のコードに入れ替えます。そしてファイル名にも同じく言語のコードが入っているので、これもまた変えておきます。

全てのデータが入れ替えられ、ファイル名を決めたら保存して終了です。

```python
# write translated contents to file
def write_result(self, file_path: str, original_contents: list[str], translated_contents: dict[int, str]):
    contents = original_contents.copy()

    for index, content in translated_contents.items():
        if 'Language:' in content:
            contents[index] = content.replace(source, target) + line_separator
        else:
            contents[index] = content + line_separator

    root = os.path.dirname(file_path)
    file_name = os.path.basename(file_path).replace(source, target)
    target_file_path = os.path.join(root, file_name)

    with open(target_file_path, 'w') as file:
        file.writelines(contents)
```

### ファイルドロップダウン

せっかくGUIを使っているので、[QListWidget](https://doc.qt.io/qtforpython-5/PySide2/QtWidgets/QListWidget.html)にファイルをDrag & Dropすると勝手にファイル名が画面に表示され、翻訳対象にもなるようにしたかったです。ここはあまり自分がQtに詳しくないので、ネットで検索したものを流用しました。

基本的にはファイルをDrag & Dropすると、拡張子が`.vtt`の場合にリストに追加されます。ここでファイルのフルパスを得ることができるのですが、画面上はファイル名だけを見えるようにして、フルパスは別のグローバル変数に格納するようにしています。

```python
# drop down file list
class FileListView(QListWidget):
    def __init__(self, parent=None):
        super(FileListView, self).__init__(parent)
        self.setAcceptDrops(True)

    def dragEnterEvent(self, event):
        if event.mimeData().hasUrls:
            event.accept()
        else:
            event.ignore()

    def dragMoveEvent(self, event):
        if event.mimeData().hasUrls:
            event.accept()
        else:
            event.ignore()

    def dropEvent(self, event):
        if event.mimeData().hasUrls:
            event.accept()
            for url in event.mimeData().urls():
                file_path = url.toLocalFile()
                if file_path.endswith('.vtt') and str(file_path) not in original_files:
                    # add file path to list
                    original_files.append(str(file_path))
                    # add file name to file list view
                    self.addItem(file_path.split('/')[-1])
        else:
            event.ignore()
```

### ファイル削除

リストにファイルを追加したら削除したい場合もあるかと思ったので、リストからファイルを選択してボタンを押すとリストから消えるようにしたかったです。これは以下のようなコードで実現しています。ボタンをクリックすると、リストで選択したものが消え、翻訳対象（フルパス）からも消えるようにしています。

```python
# create remove button
def create_remove_button(self):
    button = QPushButton()
    button.setText('Remove file')
    button.clicked.connect(self.remove_file)
    return button


# remove file from file list
def remove_file(self):
    global original_files
    for selected_item in self.list_view.selectedIndexes():
        index = selected_item.row()
        self.list_view.takeItem(index)
        original_files.pop(index)
```

### 言語選択

元の字幕ファイルの言語はヘッダから抽出できるのですが、どの言語に翻訳したいかは自分で選べるようにしました。ここは翻訳APIの使用を参考にして、翻訳可能な言語のコードをdictionaryとしています。あえてdictionaryとしている理由は、[QComboBox](https://doc.qt.io/qtforpython-5/PySide2/QtWidgets/QComboBox.html)のアイテムは画面に見えるものなので実際の言語名にしたかったからです。アイテムがdictionaryのキーとなっているので、選択したアイテムに対応するvalueが翻訳したい言語となるようにしています。

```python
# languages
supported_language_code: dict[str, str] = {
    'Korean': 'ko',
    'English': 'en',
    'Japanese': 'ja',
    'Chinese(China)': 'zh-CN',
    'Chinese(Taiwan)': 'zh-TW',
    'Vietnamese': 'vi',
    'Indonesian': 'id',
    'Thailand': 'th',
    'German': 'de',
    'Russian': 'ru',
    'Spanish': 'es',
    'French': 'fr'
}

# create drop down menu
def create_target_language_selector(self):
    label = QLabel()
    label.setText('Select target language')
    selector = QComboBox()
    selector.addItems(supported_language_code.keys())
    selector.textActivated.connect(self.set_target_language)
    return label, selector

# set translate target language when dropdown menu selected
def set_target_language(self, selectd: str):
    global supported_language_code
    global target
    target = supported_language_code[selectd]
```

### 翻訳ボタン

最後に翻訳ボタンを追加して、実際の翻訳はボタンを押下した時に行われるようにしました。ボタンを押すと、翻訳対象のファイルのパスをループしながらデータの読み込み、元の言語と翻訳したい言語が違うと翻訳APIの呼び出し、結果の保存という作業を行うようになっています。念の為処理が行われる間はこのボタンを非活性化するという処理も追加しています。

```python
# create translate button
def create_translate_button(self):
    self.translate_button = QPushButton()
    self.translate_button.setText('Translate')
    self.translate_button.clicked.connect(self.translate_files)
    self.translate_button.setDisabled(False)


# read vtt file and do translate
def translate_files(self):
    self.translate_button.setDisabled(True)
    for file_path in original_files:
        original_contents, exported_contents = self.get_content(file_path)
        if source == target:
            continue
        translated_contents = self.send_request(exported_contents)
        self.write_result(file_path, original_contents, translated_contents)
    self.translate_button.setDisabled(False)
```

実際作ったツールで重要な部分は以上で、全体のソースコードは[こちらから](https://github.com/retheviper/PythonTools/blob/master/PapagoVtt/papago_vtt.py)確認できます。

## 改善したい

こうやってとりわけ欲しい機能は実現できたのですが、まだ自分がPyQtに慣れてないのもあり、ロジック上でも少し改善したいところは残っていますので、いくつかを挙げてみました。

### テキストをまとめて翻訳する

翻訳APIの仕様上、リクエストは1秒で10回となっているため、`sleep`を入れて一つのリクエスト毎に120msを待つようになっています。I/Oが発生することを考慮するともっと間隔を短くしてよかったかもしれませんが、そもそもの問題は、読み込んだファイルのデータから一行づつ翻訳のリクエストを送っているところです。短い動画でも字幕は数百行となるケースがあるので、こうなった場合は全体の翻訳が終わるまで処理にかなり時間がかかってしまいますね。

なので、最初はリクエストパラメータに翻訳したいデータを全て送る（dictionaryのvalueを一つのstrにjoinして）方法も考えてみましたが、APIの仕様として明示されてないだけで、リクエストパラメータのサイズには制限があるようでした。だとすると、翻訳したいデータをいくつかのchunkに分けて送るという方法があると思いますが、こうなった場合は最後にファイルを保存するときに入れ替えるデータの行をどうやって判断するか、それをうまく処理するための方法が悩ましくなります。

この処理のためファイルを翻訳するまでの時間がかなり長くなるので、いつかは解消したいものですが、翻訳した結果をうまくまとめて保存できる方法が思いつくまでは少し時間がかかりそうです。

### プログレスバーを追加する

今の作りだと、シングルスレッドであるため、一つのファイルに対して処理が行われている間は画面が固まってしまうという問題があります。そもそも処理が遅いので、かなり長い時間を固まっていますが、これはUXの観点ではあまり良くないですね。また、処理がどこまで終わっているかもわからないので、プログレスバーを追加して、一つのファイルおよび全体のリストでどれぐらいの処理が行われているかを視覚化したいと思っています。

ただ、プログレスバーを追加する場合、スレッドをわけ、さらに何を基準にプログレスバーを動いていくかというロジックを考えなければならないので、そこもまた時間がかかりそうですね。

### ボタンの非活性化

翻訳対象のファイルが追加される前と処理の途中ではボタンを非活性化したいのですが、これもまだうまく実装されていません。処理中に画面が固まってしまうからか、処理の前後でボタンを非活性化するという処理も思い通りにならなかったのでこちらも直したいと思っています。

## 最後に

今回はGUIを中心に色々と新しいチャレンジができたので、かなり面白い作業となっています。そしてやはり元がサーバサイドだからか、UI/UXの観点では考慮しきれなかった問題が出てきたり、画面を実装するためにも色々と試行錯誤があったのでこれは良い勉強になっています。PythonとPyQtが優秀だったのでできたものなのですが、これをまたJetpack Composeなどで書き換えてみるとどうなるかなという好奇心もありますね。やはり、普段やってみてないことに挑戦してみるのは自分がエンジニアとして成長するための良い糧となるような気がします。

では、また！
