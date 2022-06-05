---
title: "BackendでCoroutineを使う"
date: 2022-06-05
categories: 
  - languages
image: "../../images/magic.jpg"
tags:
  - kotlin
  - go
  - coroutine
  - goroutine
  - concurrency
---

Androidアプリのように、GUIを使う場合にはマルチスレッドで処理するのはもはや常識のようなものです。シングルスレッドだと何か思い処理が行われる間に画面が固まるからです。他にもプログレスバーのようにリアルタイムで変化されるコンポーネントの状態を更新したり、チャット、通知の表示などさまざまな場面でスレッドを分けて処理する必要がある場合が多いですね。

ただ、バックエンドの処理においては少し事情が違うものです。そもそもGUIを考慮する必要がないということもありあすが、サーバでは一つのリクエストに対しての処理を「順次的に」行う場合が多いため、マルチスレッドを利用した処理の分散の利点を活かすのはなかなか難しいものです。[Reactive Streams](https://www.reactive-streams.org)のようなものもありますが、これは一つのリクエストを分散するというより少ないリソースで多くのリクエストに対する処理を行うためのものなので、一つの処理を分散して効率を上げたいという場合にはあまりふさわしくないものですね。

もちろん、だからと言ってバックエンドにおいて分散処理が全く必要ないというわけではありません。確かに一つのリクエストに対しての処理を行う中でも、処理によってスレッドを分けて性能向上を期待できる場面があります。例えば後続の処理と関係のない処理を途中に挟んでい場合では、別スレッドで処理したくなりますね。

なので、今回は[Coroutine](https://ja.wikipedia.org/wiki/%E3%82%B3%E3%83%AB%E3%83%BC%E3%83%81%E3%83%B3)を使ったバックエンドでの処理の分散するという一例を紹介したいと思います。

## APIの呼び出しを並列化する

まず並列化で効率を上げられるケースとして、バッチ処理を考えられます。バッチ処理では、条件に当てはまるデータを複数抽出し、それぞれのデータに対して同じ処理を繰り返すことが多いですね。このように個別のデータに対しての処理が独立的に実行されるものであり、並行して走っても特に問題はないという場合は十分その処理を分散できるものです。

仕事ではGoで作成されたサーバから定期的に日付を基準にDBから処理対象のデータを抽出し、そのデータを配列にしてループしながらKotlinで作成されたサーバのAPIを呼び出すようになっています。また、KotlinサーバでもBackendのAPIを呼び出すケースがあり、これもまたループでデータの参照を行っている状態です。サービスが成長するにつれて、処理にかかる時間によりAPI呼び出しがタイムアウトになるなどパフォーマンスの問題が出てきたので、このループの中でのAPI呼び出しを並列化することで処理にかかる時間を減らすことにします。

## 実装してみる

まずはCoroutineにより、ループの中でのAPIの呼び出しを実現してみます。上述したとおり、実際の仕事で使えるかどうかを検証してみたく書いたコードなので、各サーバの処理は大して変わらないものとなっています。まずループの中で互いのAPIを呼び出すような処理を書き、呼び出される側では5秒を待ってレスポンスを送るようになっています。これをCoroutineを利用して並列化していきます。

### Go

まず、以下のような処理があるとします。

```go
type callResults struct {
    Response []*client.Response `json:"response"`
}

func CallKotlinServer(c *gin.Context) {
    log.Print("[CallKotlinServer] start")

    results := &callResults{}
    tries := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    for _, i := range tries {
        log.Print("[CallKotlinServer] before request with id: ", i)

        var result string

        r, err := client.Post(i) // KotlinサーバにPOSTでリクエストを送る
        if err != nil || r == nil {
            result = "failed"
        } else {
            result = r.Result
        }

        log.Print("[CallKotlinServer] after request with id: ", i)

        results.Response = append(results.Response, &client.Response{
            ID:     i,
            Result: result,
        })
    }

    log.Print("[CallKotlinServer] done")

    c.JSON(http.StatusOK, results)
}
```

上記のコードは[Gin](https://gin-gonic.com)を使ったサーバのサンプルで、handlerの部分です。この関数の中ではAPIが呼び出されると、10回のループの中でKotlinサーバにリクエストを送ります。そして帰ってきたAPI呼び出しの結果を持ってレスポンスのstructを作成して、最終的には10回の実行結果をまとめてJSONとして返す構造となっています。

ここでKotlin側が返すレスポンスは5秒かかるため、ループの回数が多くなれば多くなるほどレスポンスが帰ってくるのも遅くなります。ログを吐くようにしているので、サーバのログを確認するとリクエストからレスポンスに50秒がかかっているのを確認できます。

```bash
2022/06/05 18:49:31 [CallKotlinServer] start
2022/06/05 18:49:31 [CallKotlinServer] before request with id: 1
2022/06/05 18:49:36 [CallKotlinServer] after request with id: 1
2022/06/05 18:49:36 [CallKotlinServer] before request with id: 2
2022/06/05 18:49:41 [CallKotlinServer] after request with id: 2
2022/06/05 18:49:41 [CallKotlinServer] before request with id: 3
2022/06/05 18:49:46 [CallKotlinServer] after request with id: 3
2022/06/05 18:49:46 [CallKotlinServer] before request with id: 4
2022/06/05 18:49:51 [CallKotlinServer] after request with id: 4
2022/06/05 18:49:51 [CallKotlinServer] before request with id: 5
2022/06/05 18:49:56 [CallKotlinServer] after request with id: 5
2022/06/05 18:49:56 [CallKotlinServer] before request with id: 6
2022/06/05 18:50:01 [CallKotlinServer] after request with id: 6
2022/06/05 18:50:01 [CallKotlinServer] before request with id: 7
2022/06/05 18:50:06 [CallKotlinServer] after request with id: 7
2022/06/05 18:50:06 [CallKotlinServer] before request with id: 8
2022/06/05 18:50:11 [CallKotlinServer] after request with id: 8
2022/06/05 18:50:11 [CallKotlinServer] before request with id: 9
2022/06/05 18:50:16 [CallKotlinServer] after request with id: 9
2022/06/05 18:50:16 [CallKotlinServer] before request with id: 10
2022/06/05 18:50:21 [CallKotlinServer] after request with id: 10
2022/06/05 18:50:21 [CallKotlinServer] done
[GIN] 2022/06/05 - 18:50:21 | 200 | 50.250251292s |       127.0.0.1 | GET      "/api/v1/call-kotlin-server"
```

#### Goroutineで並列化する(1)

では、以上の処理を並列化することにします。Goには[Goroutine](https://go-tour-jp.appspot.com/concurrency/1)が基本的に含まれています。使い方は単純で、実行したい関数の前に`go`のキーワードをつけるだけですね。ただ、レスポンスでは10回の実行結果を待ってから返す必要があるのですが、goroutineでAPIの呼び出しをするとメインスレッドが先に終わってしまう可能性があります。

というわけで、ループの中でのAPIの呼び出しにgoroutineを使い、さらにそのgoroutineが全て終了してから結果を返すようにします。goには`sync`というパッケージに[waitGroup](https://pkg.go.dev/sync#WaitGroup)があり、goroutineの終了を待つことができるようになっています。また、goroutineをループの中で実行する場合、順番はランダムになるのでレスポンスを返す際は一度ソートをかけるようにします。以上を考慮して実装した結果は以下の通りです。

```go
func CallKotlinServerAsync(c *gin.Context) {
    log.Print("[CallKotlinServerAsync] start")

    results := &callResults{}
    tries := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    group := &sync.WaitGroup{} // waitGroupを定義

    for _, i := range tries {
        group.Add(1) // ループごとに実行するgoroutineの回数を追加

        go func(i int) { // goroutineでAPIの呼び出す
            log.Print("[CallKotlinServerAsync] before request with id: ", i)

            var result string

            r, err := client.Post(i)
            if err != nil || r == nil {
                result = "failed"
            } else {
                result = r.Result
            }

            log.Print("[CallKotlinServerAsync] after request with id: ", i)

            results.Response = append(results.Response, &client.Response{
                ID:     i,
                Result: result,
            })

            group.Done() // waitGroupにgoroutineの終了を設定
        }(i)
    }

    group.Wait() // 全てのgoroutineが終了するのを待つ
    sort.Slice(results.Response, func(i, j int) bool {
        return results.Response[i].ID < results.Response[j].ID
    })

    log.Print("[CallKotlinServerAsync] done")

    c.JSON(http.StatusOK, results)
}
```

このように修正して実行した結果のログは以下のとおりです。

```bash
2022/06/05 18:52:30 [CallKotlinServerAsync] start
2022/06/05 18:52:30 [CallKotlinServerAsync] before request with id: 10
2022/06/05 18:52:30 [CallKotlinServerAsync] before request with id: 5
2022/06/05 18:52:30 [CallKotlinServerAsync] before request with id: 7
2022/06/05 18:52:30 [CallKotlinServerAsync] before request with id: 8
2022/06/05 18:52:30 [CallKotlinServerAsync] before request with id: 6
2022/06/05 18:52:30 [CallKotlinServerAsync] before request with id: 1
2022/06/05 18:52:30 [CallKotlinServerAsync] before request with id: 9
2022/06/05 18:52:30 [CallKotlinServerAsync] before request with id: 3
2022/06/05 18:52:30 [CallKotlinServerAsync] before request with id: 4
2022/06/05 18:52:30 [CallKotlinServerAsync] before request with id: 2
2022/06/05 18:52:35 [CallKotlinServerAsync] after request with id: 6
2022/06/05 18:52:35 [CallKotlinServerAsync] after request with id: 1
2022/06/05 18:52:35 [CallKotlinServerAsync] after request with id: 8
2022/06/05 18:52:35 [CallKotlinServerAsync] after request with id: 10
2022/06/05 18:52:35 [CallKotlinServerAsync] after request with id: 4
2022/06/05 18:52:35 [CallKotlinServerAsync] after request with id: 7
2022/06/05 18:52:35 [CallKotlinServerAsync] after request with id: 3
2022/06/05 18:52:35 [CallKotlinServerAsync] after request with id: 9
2022/06/05 18:52:35 [CallKotlinServerAsync] after request with id: 5
2022/06/05 18:52:35 [CallKotlinServerAsync] after request with id: 2
2022/06/05 18:52:35 [CallKotlinServerAsync] done
[GIN] 2022/06/05 - 18:52:35 | 200 |  5.012657333s |       127.0.0.1 | GET      "/api/v1/call-kotlin-server-async"
```

10回のループがほぼ同時に実行されたため、レスポンスまで5秒ほどかかっているのがわかります。そしてやはりgoroutineの実行が順番に行われてないことがわかりますね。なので、実行の順番が重要でなくても、結果は順番を守って返す必要がある時はやはりソートが必要ということがわかります。

#### Goroutineで並列化する(2)

場合によっては並列化できるからって、全ての処理を同時に走らせるのは危険な時もあります。上記のコードの場合、リクエストの数は10となっていますが、もしそれより多くのリクエストが必要か、さらに重い処理のAPIを呼び出す場合はどうでしょうか。Go側はリクエストを投げるだけなので処理の負荷はあまり変わらないものですが、APIを呼び出されている側としてはかなりの負荷になるはずです。

だとすると、やはり並列の数を制限する必要があるはずです。例えば並列の数を2にすると、リクエストは2件づつ送られるのでリクエストの全体の数がいくら増えても負荷は一定に保てます。同時に全てのリクエストを送るよりは遅くなりますが、リソースの状況を見ながら並列数を増やすだけで柔軟に対応ができるので、外部設定ファイルなどで並列数を指定できるようにするとアプリのビルドなしでも柔軟に対応ができるというメリットもありますね。

スレッドを使う場合だと、このような処理をするためにはかなり複雑な処理を書くことになるはずです。例えば、並列数に合わせてスレッドを定義して、さらにスレッドごとに割り当てる処理を分けなければならないですね。今は10件のリクエストを想定しているので、スレッドごとに5件づつというふうにリクエストを分けるだけで対応ができますが、リクエスト数をスレッド数で割った結果を考慮してループする要素数を適宜分割するような処理をまず足す必要があります。

しかし、実はgoroutineを使うとそのような複雑な処理をまた足す必要はないです。goroutineでは[Channel](https://go-tour-jp.appspot.com/concurrency/2)を利用して、同時に実行されるgoroutineの数を指定できます。以下のようにです。

```go
func CallKotlinServerAsyncDual(c *gin.Context) {
    log.Print("[CallKotlinServerAsyncDual] start")

    results := &callResults{}
    tries := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    concurrency := 2 // goroutineの同時実行数を指定
    group := &sync.WaitGroup{}
    guard := make(chan struct{}, concurrency) // 同時実行数でChannelを定義

    for _, i := range tries {

        group.Add(1)
        guard <- struct{}{} // Channelに実行を一つたす

        go func(i int) {
            log.Print("[CallKotlinServerAsyncDual] before request with id: ", i)

            var result string

            r, err := client.Post(i)
            if err != nil || r == nil {
                result = "failed"
            } else {
                result = r.Result
            }

            log.Print("[CallKotlinServerAsyncDual] after request with id: ", i)

            results.Response = append(results.Response, &client.Response{
                ID:     i,
                Result: result,
            })

            group.Done()
            <-guard // Channelを準備させる
        }(i)
    }

    group.Wait()

    sort.Slice(results.Response, func(i, j int) bool {
        return results.Response[i].ID < results.Response[j].ID
    })

    log.Print("[CallKotlinServerAsyncDual] done")

    c.JSON(http.StatusOK, results)
}
```

Channelには指定した数分だけ送信すると、Channelから値を受信するまでは新しいgoroutineの実行はブロックされます。なので、実際に実行してみると、意図通り最大2件づつのリクエストが送信されているのを確認できます。

```bash
2022/06/05 19:56:10 [CallKotlinServerAsyncDual] start
2022/06/05 19:56:10 [CallKotlinServerAsyncDual] before request with id: 2
2022/06/05 19:56:10 [CallKotlinServerAsyncDual] before request with id: 1
2022/06/05 19:56:15 [CallKotlinServerAsyncDual] after request with id: 2
2022/06/05 19:56:15 [CallKotlinServerAsyncDual] before request with id: 3
2022/06/05 19:56:15 [CallKotlinServerAsyncDual] after request with id: 1
2022/06/05 19:56:15 [CallKotlinServerAsyncDual] before request with id: 4
2022/06/05 19:56:21 [CallKotlinServerAsyncDual] after request with id: 4
2022/06/05 19:56:21 [CallKotlinServerAsyncDual] before request with id: 5
2022/06/05 19:56:21 [CallKotlinServerAsyncDual] after request with id: 3
2022/06/05 19:56:21 [CallKotlinServerAsyncDual] before request with id: 6
2022/06/05 19:56:26 [CallKotlinServerAsyncDual] after request with id: 5
2022/06/05 19:56:26 [CallKotlinServerAsyncDual] before request with id: 7
2022/06/05 19:56:26 [CallKotlinServerAsyncDual] after request with id: 6
2022/06/05 19:56:26 [CallKotlinServerAsyncDual] before request with id: 8
2022/06/05 19:56:31 [CallKotlinServerAsyncDual] after request with id: 7
2022/06/05 19:56:31 [CallKotlinServerAsyncDual] before request with id: 9
2022/06/05 19:56:31 [CallKotlinServerAsyncDual] after request with id: 8
2022/06/05 19:56:31 [CallKotlinServerAsyncDual] before request with id: 10
2022/06/05 19:56:36 [CallKotlinServerAsyncDual] after request with id: 9
2022/06/05 19:56:36 [CallKotlinServerAsyncDual] after request with id: 10
2022/06/05 19:56:36 [CallKotlinServerAsyncDual] done
[GIN] 2022/06/05 - 19:56:36 | 200 | 25.194952625s |       127.0.0.1 | GET      "/api/v1/call-kotlin-server-async-dual"
```

### Kotlin

まずは順次処理する場合のコードから見ていきます。基本的にGoの場合と同じ処理をKotlin側にも用意していて、特に変わったものはありません。以下がそのコードです。

```kotlin
fun callGoServer(): List<CallGoServerDto> {
    logger.info("[CallGoServer] start")
    return tries.map {
        logger.info("[CallGoServer] before request with id: $it")
        goServerClient.call(it) ?: CallGoServerDto(it, "failed") // GoのAPIを呼び出す
        .also { result ->
            logger.info("[CallGoServer] after request with id: ${result.id}")
        }
    }.also {
        logger.info("[CallGoServer] done")
    }
}
```

Curlで実行してみた結果は以下の通りです。

```bash
2022-06-05 20:06:33.429  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] start
2022-06-05 20:06:33.430  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] before request with id: 1
2022-06-05 20:06:38.483  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] after request with id: 1
2022-06-05 20:06:38.483  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] before request with id: 2
2022-06-05 20:06:43.490  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] after request with id: 2
2022-06-05 20:06:43.491  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] before request with id: 3
2022-06-05 20:06:48.498  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] after request with id: 3
2022-06-05 20:06:48.499  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] before request with id: 4
2022-06-05 20:06:53.509  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] after request with id: 4
2022-06-05 20:06:53.510  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] before request with id: 5
2022-06-05 20:06:58.518  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] after request with id: 5
2022-06-05 20:06:58.518  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] before request with id: 6
2022-06-05 20:07:03.530  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] after request with id: 6
2022-06-05 20:07:03.531  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] before request with id: 7
2022-06-05 20:07:08.538  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] after request with id: 7
2022-06-05 20:07:08.539  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] before request with id: 8
2022-06-05 20:07:13.552  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] after request with id: 8
2022-06-05 20:07:13.553  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] before request with id: 9
2022-06-05 20:07:18.561  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] after request with id: 9
2022-06-05 20:07:18.562  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] before request with id: 10
2022-06-05 20:07:23.570  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] after request with id: 10
2022-06-05 20:07:23.570  INFO 60551 --- [nio-8900-exec-2] c.e.c.d.service.CallGoServerService      : [CallGoServer] done
```

こちらもGoの時と同じく、リクエストからレスポンスまで50秒ほどかかっているのがわかります。これをCoroutineを持って並列化していきましょう。

#### Coroutineで並列化する(1)

Goと違って、KotlinのCoroutineは言語の基本仕様ではありません。なので、依存関係をまず追加する必要があります。ただ、公式の説明では`coroutine-core`だけを追加すると対応できそうなイメージですが、SpringのようにReactive Streamが必要な場合は`coroutine-reactor`を依存関係に追加する必要があります。

依存関係を追加した上で、コードを直していきます。ここではSpring Bootを使っていて、Controllerの関数を`suspend`にすることができるので、Contollerから呼び出している関数にもsuspendにしていきます。また、coroutineでの処理はスコープの指定が必要なのでループの周りを[coroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html)で包むようにします。その後は`map`関数の中で[async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html)としてAPIの呼び出しを行い、`map`した結果は[Deferred](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html)として帰ってくるので[awaitAll](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/await-all.html)で終了を待ちます。説明では複雑ですが、以下のコードをみるとわかりやすいかなと思います。

```kotlin
suspend fun callGoServerAsync(): List<CallGoServerDto> {
    logger.info("[CallGoServerAsync] start")
    return coroutineScope { // coroutineとして処理する
        tries.map {
            async { // 並列に実行
                logger.info("[CallGoServerAsync] before request with id: $it")
                goServerClient.call(it) ?: CallGoServerDto(it, "failed")
            }
        }.awaitAll() // APIの呼び出し結果を待つ
            .also {
                it.forEach { result ->
                    logger.info("[CallGoServerAsyncDual] after request with id: ${result.id}")
                }
                logger.info("[CallGoServerAsyncDual] done")
            }
    }
}
```

また、APIを呼び出している関数(`goServerClient.call()`)もsuspendにしておく必要があります。ここではSpringの[RestTemplate](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)を使い、以下のような関数を定義しておきました。

```kotlin
private val client = RestTemplate()

private val header = HttpHeaders().apply {
    set(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
}

suspend fun call(id: Int): CallGoServerDto? {
    val request = HttpEntity(CallGoServerRequest(id), header)
    return withContext(Dispatchers.IO) {
        client.postForObject("http://localhost:8800/api/v1/some-process", request, CallGoServerDto::class.java)
    }
}
```

上記のようにコードを修正して実行してみると、Goの時と同じく並列で10件のリクエストが送られているのがわかります。ただ、違う点としてはgoroutineと違って実行の順番が保証されているというところですね。この特徴があるため、Kotlinの場合はレスポンスのソートが必要ないです。

```bash
2022-06-05 20:46:52.934  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] start
2022-06-05 20:46:52.939  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] before request with id: 1
2022-06-05 20:46:52.939  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] before request with id: 2
2022-06-05 20:46:52.939  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] before request with id: 3
2022-06-05 20:46:52.940  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] before request with id: 4
2022-06-05 20:46:52.940  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] before request with id: 5
2022-06-05 20:46:52.940  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] before request with id: 6
2022-06-05 20:46:52.941  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] before request with id: 7
2022-06-05 20:46:52.941  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] before request with id: 8
2022-06-05 20:46:52.941  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] before request with id: 9
2022-06-05 20:46:52.941  INFO 60551 --- [nio-8900-exec-1] c.e.c.d.service.CallGoServerService      : [CallGoServerAsync] before request with id: 10
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 1
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 2
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 3
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 4
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 5
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 6
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 7
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 8
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 9
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 10
2022-06-05 20:46:57.951  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] done
```

#### Coroutineで並列化する(2)

上記のコードもGoの時と同じく、リクエストを同時に全部送っているのは問題になる可能性があるので、同時に送信するリクエストの数を制限することにします。Goでもそうであったように、KotlinでもCoroutineの同時実行の数を制限する仕組みがあります。[Semaphore](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-semaphore/index.html)というものです。

Sempaphoreに数値を指定し、asyncの中でSemaphoreに指定した数で実行数を制限することで並行実行数を制限するような形です。以下がそのコードです。

```kotlin
suspend fun callGoServerAsyncDual(): List<CallGoServerDto> {
    logger.info("[CallGoServerAsyncDual] start")
    val semaphore = Semaphore(2) // 同時実行数を制限するためのSempahoreの定義
    return coroutineScope {
        tries.map {
            async {
                semaphore.withPermit { // asyncの同時実行数をSemaphoreに指定した数値に制限
                    logger.info("[CallGoServerAsyncDual] before request with id: $it")
                    goServerClient.call(it) ?: CallGoServerDto(it, "failed")
                }
            }
        }
    }.awaitAll()
        .also {
            it.forEach { result ->
                logger.info("[CallGoServerAsyncDual] after request with id: ${result.id}")
            }
            logger.info("[CallGoServerAsyncDual] done")
        }
}
```

書き方が少し違うだけで、Goとほぼ同じ感覚でasyncの処理を制限できるコードが出来ました。特にコンパイルエラーが出ることはないので勘違いしやすいところではないかと思います。`async{ semaphore.withPermit{ } }`の順番をちゃんと守る必要がありますので注意しましょう。実行結果は以下の通りです。

```bash
2022-06-05 20:50:50.361  INFO 60551 --- [nio-8900-exec-6] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] start
2022-06-05 20:50:50.365  INFO 60551 --- [nio-8900-exec-6] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] before request with id: 1
2022-06-05 20:50:50.366  INFO 60551 --- [nio-8900-exec-6] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] before request with id: 2
2022-06-05 20:50:55.369  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] before request with id: 3
2022-06-05 20:50:55.369  INFO 60551 --- [atcher-worker-2] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] before request with id: 4
2022-06-05 20:51:00.377  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] before request with id: 6
2022-06-05 20:51:00.379  INFO 60551 --- [atcher-worker-2] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] before request with id: 5
2022-06-05 20:51:05.386  INFO 60551 --- [atcher-worker-2] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] before request with id: 7
2022-06-05 20:51:05.386  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] before request with id: 8
2022-06-05 20:51:10.393  INFO 60551 --- [atcher-worker-2] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] before request with id: 10
2022-06-05 20:51:10.393  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] before request with id: 9
2022-06-05 20:51:15.404  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 1
2022-06-05 20:51:15.404  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 2
2022-06-05 20:51:15.405  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 3
2022-06-05 20:51:15.405  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 4
2022-06-05 20:51:15.405  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 5
2022-06-05 20:51:15.405  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 6
2022-06-05 20:51:15.405  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 7
2022-06-05 20:51:15.405  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 8
2022-06-05 20:51:15.405  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 9
2022-06-05 20:51:15.405  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] after request with id: 10
2022-06-05 20:51:15.405  INFO 60551 --- [atcher-worker-8] c.e.c.d.service.CallGoServerService      : [CallGoServerAsyncDual] done
```

ログを吐く場所が微妙だったのですが、リクエストを送っている時間をみると、5秒置きで2つづつを送信しているのがわかります。

## 最後に

あまりCoroutineに詳しくないゆえ、もっと良い書き方はあったかなと思いますが(goroutineの実行順を決めておく、Kotlinのログ出力箇所を調整するなど)、これで簡単にAPIの呼び出しを並列化することができるというのがわかったので個人的にはかなり満足しています。Jetpack Composeを少し触りながらcoroutineに触れたことはあったものの、こうやって仕事で必要となり調査と検証をしてみたのは初めてだったのでかなりの収穫を得たと言えますね。また、各言語においての感想は以下の通りです。

- Go
  - 依存関係の追加なしで使えるのはメリット
  - Kotlinのようにsuspendやscopeを意識しなくていいので便利
- Kotlin
  - asyncでも実行順が保証されているのはメリット
  - goroutineよりは注意点が多い

二つの言語を比べると一長一短があるという感覚ですが、どれも応用が難しいものではないので、すぐにプロダクションコードにも適用できそうなものが書けるのは確かに良いものという印象を受けました。これからもcoroutineを使って性能向上ができる箇所はないか、色々と試してみたくなるものです。ちなみに、この記事に載せてあるコードを全体のソースは[こちらのリポジトリ](https://github.com/retheviper/AsyncServerExample)から参照できます。

では、また！
