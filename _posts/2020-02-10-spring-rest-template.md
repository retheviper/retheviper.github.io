---
title: "Rest APIからRest APIにファイルを送る"
date: 2020-02-10
categories: 
  - spring
photos:
  - /assets/images/sideimage/spring_logo.jpg
tags:
  - rest api
  - java
  - spring
---

ウェブアプリケーションを開発していると、一つの`Rest API`だけで全ての機能を自己完結させる必要はない時もあります。例えば組み込みたい機能がすでに実装されているサーバー(API)が存在している場合もありますね。そういう場合は、素直にそのAPIをコールするだけで簡単に目的を達成できます。実際、仕事で他のRest APIサーバーとの通信が必要となって調べてみましたが、Springではすでにそのような場合に対応できるようなクラスを用意していました。今回のポストの主人公であるRestTemplateです。

RestTemplateを使うと、簡単にget・post・deleteと言ったHttpメソッドのAPIをコールできます。また、リクエストやレスポンスをカスタムして状況に合わせて使うこともできます。例えばカスタムヘッダーを作ったり、サイズの大きいファイルを転送するリクエストを作ることも可能ですね。なので今回はRestTemplateを利用し、ファイルをアップロードするとそのファイルになんらかの処理をして返してくれるAPIがすでに存在している場合で、そのAPIをコールする部品を作る方法を紹介します。

## サーバー側の例

すでにファイルを処理するサーバーが存在している場合のことですので、まずはコールしたいAPIで利用するリクエストとレスポンスの形を把握する必要がありますね。ファイルをアップロードされたら、ヘッダーからファイル情報を読み込むようになっていて、ファイルデータが書かれているボディを読み込むようなメソッドがあるとしましょう。ヘッダーの情報に問題がなかったらローカルストレージにファイルを書き込み、処理を行います。そして処理の終わったファイルはレスポンスとして返すサーバーです。

```java
@PostMapping("/upload")
public ResponseEntity<StreamingResponseBody> fileupload(HttpServletRequest request) {
    // リクエストヘッダーからファイルサイズを取得
    long fileLength = request.getContentLengthLong();
    // ファイルサイズが0だとIOException
    if (fileLength == 0) {
       throw new IOException("data size is 0");
    }

    // ファイルを臨時ファイルとして保存
    String fileName = request.getHeader("Content-File-Name");
    Path tempFile = Files.createTempFile("process_target_", fileName);
    try (InputStream is = request.getInputStream()) {
        Files.copy(is, tempFile);
    } catch (IOException e) {
        e.printStackTrace();
    }

    // ...ファイルを持ってなんらかの処理を行う

    // レスポンス用のヘッダーを作る
    HttpHeaders headers = new Headers();
    // 処理が終わったファイルを書き込むボディを作る
    StreamingResponseBody body = new StreamingResponseBody() {
        @Override
        public void writeTo(OutputStream outputStream) throws IOException {
            byte[] bytes = Files.readAllBytes(path);
            outputStream.write(bytes);
        }
    };
    // ヘッダーとボディ、HttpStatusをセットしてレスポンスを返す
    return new ResponseEntity<StreamingResponseBody>(body, headers, HttpStatus.OK);
}
```

サーバーがこういう形になっている場合、APIをコールする側としてはRequestのヘッダーにはファイル情報を書き、ボディにはファイルのデータを書いて転送する必要がありますね。そして処理結果のResponseにもまたファイルデータが入ってあるので、それを受け止める処理が必要となります。そのためにリクエストもレスポンスもカスタムのものを作って、RestTemplateに載せることにしましょう。

## RestTemplateを使う

今回はRestTemplateのexecute()というメソッドを使いますが、このメソッドの引数は以下のようになります。

1. APIのURL(StringもしくはURI)
2. Httpメソッド(ENUM)
3. リクエスト(RequestCallbackの実装クラス)
4. レスポンス(ResponseExtractorの実装クラス)

get()・post()・delete()などのメソッドの時と違ってリクエストとレスポンスの方を両方指定する理由は、上に述べた通りリクエストとレスポンスの両方でファイルの転送が必要からです。また、execute()でも引数にURI変数を指定することもできますが、現在はURIが固定なので使いません。では、リクエストとレスポンスのインタフェースをどう実装するかをみていきましょう。

## リクエスト

リクエストで使うRequestCallbackの実装クラスを作成します。このインタフェースにはコンストラクターの引数としてファイルを渡すとヘッダーとボディを作るようにしてみましょう。RequestCallbackをimplementsすると、doWithRequest()というメソッドをオーバーライドするようになります。このメソッドの引数であるClientHttpRequestにヘッダーとボディを設定することでリクエスト時のファイルアップロードができます。以下のコードを参照してください。

```java
public class FileTransferRequestCallback implements RequestCallback {

    // アップロードしたいファイル
    private Path path;
    
    // ヘッダーにファイル情報を載せるためのコンストラクター
    public PdfConvertRequestCallback(File file) {
        this.path = file.toPath();
    }

    @Override
    public void doWithRequest(ClientHttpRequest request) throws IOException {
        // ファイルからヘッダーを作る
        request.getHeaders().set("Content-Length", Files.size(path));
        request.getHeaders().set("Content-File-Name", path.getFileName().toString());
        // ボディにファイルを書き込む
        try (InputStream is = Files.newInputStream(file); OutputStream os = request.getBody()) {
            is.transferTo(os);
        }
    }
}
```

ヘッダーにはサーバーで要求するファイルサイズとファイル名を載せました。そしてアップロードするファイルをInputStreamとして取得して、OutputStreamであるボディに書き込みます。これでリクエストでのファイルアップロード設定は終わりです。次はレスポンスですね。

## レスポンス

レスポンスでは、ResponseExctractorをimplementsします。この場合はextractData()というメソッドをオーバーライドするようになります。このメソッドの引数であるClientHttpResponseからはリクエストの時と同じくHttpステータスコード、ヘッダー、ボディを取得できます。このレスポンスの結果からResponseEntityのインスタンスを作成し、レスポンスの結果を載せて返すとRestTemplateからは通信の結果としてResponseEntityを返すようになります。

ResponseEntityを返すためにはそのボディの型を指定する必要があります。InputStreamの型を指定して、レスポンスのボディがファイルであることを指定しましょう。また、ClientHttpResponseのボディをResponseEntityにそのまま載せると、InputSteamがCloseされるのでボディはコピーしておきます。私は一回byte[]に変えて、さらにByteArrayInputStreamを生成することにしました。

```java
public class FileTransferResponseExtractor implements ResponseExtractor<ResponseEntity<InputStream>> {

    @Override
    public ResponseEntity<InputStream> extractData(ClientHttpResponse response) throws IOException {
        // レスポンスのボディをコピー
        byte[] bytes = response.getBody().readAllBytes();
        // ステータスコード、ヘッダー、ボディのデータを載せてResponseEntityを返却
        return ResponseEntity.status(response.getStatusCode()).headers(response.getHeaders()).body(new ByteArrayInputStream(bytes));
    }
}
```

これでレスポンスのファイルを取得できるようになりました。次は、RestTemplateでAPIをコールするだけです。

## Rest APIのコール

先に述べましたが、RestTemplateのメソッドを実行するのは簡単です。まずはURLと、アップロードしたいファイルのインスタンスを作っておきましょう。そして、先ほど作成したRequestCallbackとResponseExtractorのインスタンスも作成します(ResponseExtractorは、状態を持たないのでBeanとして登録しても良いです)。

execute()の引数に、URL・Httpメソッドのタイプ・RequestCallback・ResponseExtractorを指定して実行すると、その結果をResponseEntityとして取得できて、そこからさらにステータスコード、ヘッダー、ボディを取得できます。これでアップロードしたファイルを処理してもらい、処理結果のファイルも即取得可能になりますね。

```java
// RestTemplateに渡す引数を準備
String url = "http://api/v1/file/upload";
File uploadFile = new File("path/to/upload_file.txt");
FileTransferRequestCallback requestCallback = new FileTransferRequestCallback(uploadFile);
FileTransferResponseExtractor responseExtractor = new FileTransferResponseExtractor();

// RestTemplateでAPIコールし、その結果を取得
ResponseEntity<InputStream> responseEntity = new RestTemplate().execute(url, HttpMethod.POST, requestCallback, responseExtractor);

// ResponseEntityからHttpステータスを取得
if (responseEntity.getStatusCode() != HttpStatus.OK) {
    throw new IOException();
}
// ResponseEntityからヘッダーを取得
HttpHeaders headers = responseEntity.getHeaders();

// ResponseEntityからボディを取得
try (InputStream is = responseEntity.getBody()) {
    File downloadedFile = new File("path/to/downloaded_file.txt");
    Files.copy(is, downloadedFile);
}
```

意外と簡単！これで他のRest APIとのファイルのやりとりができるようになりました。

## 最後に

クラスやメソッドを機能別に分けるだけでなく、Rest APIもまた機能によっては分離されることもありますね。今回の場合がまさにそのような例でした。もちろんネットを経由するのでこのようなやり方は一つのRest API内に機能をまとめて置くよりは安定性が劣るかもしれませんが、再使用性が確保できるという面では良い方法ではないかと思います。同じサーバー内だと通信失敗の確率も下がるだろうし、色々と活用できる余地はありそうですね。

最近はなかなかブログに載せられるようなコンテンツがなかったので(勉強は続けているつもりですが…)、次は何を書けばいいかなと悩んでいましたが、ちょうど面白い部品を作ることができてよかったです。Springの世界も本当に広くて奥深いものですね。では、また！