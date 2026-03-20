---
title: "백엔드에서 Coroutine 사용"
date: 2022-06-05
categories: 
  - languages
image: "../../images/magic.webp"
tags:
  - kotlin
  - go
  - coroutine
  - goroutine
  - concurrency
translationKey: "posts/server-side-coroutine"
---

Android 앱처럼 GUI를 다룰 때는 멀티스레드 처리가 거의 기본처럼 여겨집니다. 싱글 스레드로 무거운 작업을 돌리면 화면이 쉽게 멈추기 때문입니다. 진행바처럼 실시간으로 바뀌는 컴포넌트의 상태를 갱신하거나, 채팅과 알림 표시를 처리할 때도 스레드를 나누는 경우가 많습니다.
반면 백엔드에서는 사정이 조금 다릅니다. 애초에 GUI를 신경 쓸 필요가 없고, 서버는 한 요청을 보통 순차적으로 처리하기 때문에 멀티스레드로 일을 쪼갠다고 해서 항상 이점이 생기지는 않습니다. [Reactive Streams](https://www.reactive-streams.org) 같은 방식도 있지만, 이건 하나의 요청을 분산하기보다 적은 리소스로 많은 요청을 흘려보내는 데 더 가깝습니다. 그래서 한 요청 안의 작업을 나눠 효율을 높이고 싶을 때는 딱 맞지 않을 수 있습니다.

물론 백엔드에서 분산 처리가 전혀 필요 없다는 뜻은 아닙니다. 한 요청을 처리하는 도중에도, 뒤의 처리와 무관한 작업이 있다면 그 부분은 따로 돌려 성능을 높일 수 있습니다.

그래서 이번에는 [Coroutine](https://ja.wikipedia.org/wiki/%E3%82%B3%E3%83%AB%E3%83%BC%E3%83%81%E3%83%B3)을 이용해 백엔드의 처리를 분산해 본 예를 소개하려고 합니다. [^참고]
## API 호출 병렬화

병렬화로 효과를 보기 좋은 사례로는 배치 처리가 있습니다. 조건에 맞는 데이터를 여러 개 꺼내서, 각 항목에 같은 작업을 반복하는 경우가 많기 때문입니다. 이렇게 각 데이터의 처리가 서로 독립적이면, 동시에 돌려도 문제가 없는 경우가 많습니다.

실무에서는 Go로 만든 서버가 날짜 기준으로 DB에서 처리 대상을 가져와, 그 데이터를 루프 돌며 Kotlin으로 만든 서버의 API를 호출하고 있었습니다. 반대로 Kotlin 서버에서도 다른 백엔드 API를 호출하는 일이 있었고, 이쪽도 같은 식으로 데이터를 하나씩 순차 처리하고 있었습니다. 서비스가 커지면서 처리 시간이 길어졌고, 결국 API 호출이 타임아웃 나는 문제가 생겼습니다. 그래서 이 루프 안의 호출을 병렬화해 전체 처리 시간을 줄이기로 했습니다.
## 구현해보기

우선은 Coroutine으로 루프 안의 API 호출을 병렬화해 보겠습니다. 앞서 말했듯이 실제 업무에 적용 가능한지 검증하려고 만든 코드라서, 각 서버의 로직 자체는 거의 바꾸지 않았습니다. 호출받는 쪽은 일부러 5초를 기다렸다가 응답을 돌려주도록 해 두고, 이를 Coroutine으로 어떻게 분산할 수 있는지 확인합니다.
### Go

우선, 이하와 같은 처리가 있다고 합니다.
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

        r, err := client.Post(i) // Kotlin 서버에 POST 요청을 보낸다
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

위 코드는 [Gin](https://gin-gonic.com)을 사용하는 서버의 handler 예시입니다. 이 함수가 호출되면 10번 루프를 돌면서 Kotlin 서버에 요청을 보내고, 돌아온 결과를 모아 응답용 struct를 만든 뒤 최종적으로 JSON으로 반환합니다.

문제는 Kotlin 쪽 응답마다 5초가 걸린다는 점입니다. 그래서 루프 횟수가 늘어날수록 전체 응답 시간도 그대로 늘어납니다. 로그를 보면 요청부터 응답까지 50초가 걸린다는 사실을 확인할 수 있습니다.
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

#### Goroutine으로 병렬화 (1)

그럼 이 처리를 병렬화해 보겠습니다. Go에는 [Goroutine](https://go-tour-jp.appspot.com/concurrency/1)이 기본적으로 들어 있어, 실행하고 싶은 함수 앞에 `go`만 붙이면 됩니다. 다만 응답은 10번의 결과를 모두 모아서 반환해야 하므로, goroutine을 그대로 쓰면 메인 흐름이 먼저 끝나 버릴 수 있습니다.

그래서 루프 안의 API 호출을 goroutine으로 돌리고, 모든 goroutine이 끝난 뒤 결과를 반환하도록 했습니다. Go의 `sync` 패키지에는 [WaitGroup](https://pkg.go.dev/sync#WaitGroup)이 있어서 종료를 기다릴 수 있습니다. 또 루프 안에서 goroutine을 돌리면 실행 순서가 뒤섞이므로, 응답을 돌려주기 전에 한 번 정렬해 줍니다. 구현은 다음과 같습니다.
```go
func CallKotlinServerAsync(c *gin.Context) {
    log.Print("[CallKotlinServerAsync] start")

    results := &callResults{}
    tries := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    group := &sync.WaitGroup{} // WaitGroup을 정의한다

    for _, i := range tries {
        group.Add(1) // 루프마다 실행할 goroutine 수를 늘린다

        go func(i int) { // goroutine으로 API를 호출한다
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

            group.Done() // waitGroup에 goroutine 종료를 기록한다
        }(i)
    }

    group.Wait() // 모든 goroutine이 끝날 때까지 기다린다
    sort.Slice(results.Response, func(i, j int) bool {
        return results.Response[i].ID < results.Response[j].ID
    })

    log.Print("[CallKotlinServerAsync] done")

    c.JSON(http.StatusOK, results)
}
```

수정한 뒤 실행하면 로그는 다음과 같이 나옵니다.
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

10번의 루프가 거의 동시에 시작되었기 때문에 응답까지는 5초 정도밖에 걸리지 않습니다. 대신 goroutine은 실행 순서가 보장되지 않으니, 결과를 순서대로 반환해야 할 때는 별도로 정렬이 필요하다는 점도 확인할 수 있습니다.
#### Goroutine으로 병렬화 (2)

상황에 따라서는 모든 처리를 한꺼번에 돌리는 것이 오히려 위험할 때도 있습니다. 위 코드처럼 요청 수가 10개일 때는 괜찮아 보여도, 더 많은 요청을 보내거나 더 무거운 API를 호출하면 호출받는 쪽에 꽤 큰 부담이 됩니다.

이럴 때는 병렬 수를 제한해야 합니다. 예를 들어 동시에 2개씩만 보내면 요청 수가 늘어나도 서버 부담은 일정하게 유지할 수 있습니다. 물론 전체 처리 시간은 더 길어지지만, 리소스 상황에 따라 병렬 수를 조절할 수 있으니 외부 설정으로 빼 두면 유연하게 대응할 수 있습니다.

스레드로 이걸 처리하려면 꽤 복잡해질 수 있습니다. 병렬 수에 맞춰 스레드를 나누고, 각 스레드에 어떤 작업을 맡길지 직접 계산해야 하니까요. 지금은 10건 정도라 단순하게 나눌 수 있지만, 요청 수와 스레드 수를 고려해 루프를 적절히 쪼개는 로직이 따로 필요합니다.

하지만 goroutine을 쓰면 그런 복잡한 처리를 덜어낼 수 있습니다. [Channel](https://go-tour-jp.appspot.com/concurrency/2)로 동시에 실행할 goroutine 수를 제한하면 됩니다. 코드는 다음과 같습니다.
```go
func CallKotlinServerAsyncDual(c *gin.Context) {
    log.Print("[CallKotlinServerAsyncDual] start")

    results := &callResults{}
    tries := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    concurrency := 2 // goroutine 동시 실행 수를 지정한다
    group := &sync.WaitGroup{}
    guard := make(chan struct{}, concurrency) // 동시 실행 수에 맞는 Channel을 만든다

    for _, i := range tries {

        group.Add(1)
        guard <- struct{}{} // Channel에 실행 슬롯을 하나 추가한다

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
            <-guard // Channel 슬롯을 비워 다음 실행을 허용한다
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

채널에 정해 둔 수만큼만 값을 넣으면, 채널에서 다시 값을 받을 때까지 새로운 goroutine 실행이 막힙니다. 그래서 실제로 돌려 보면 의도한 대로 최대 2건씩만 요청이 전송되는 것을 확인할 수 있습니다.
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

이제 Kotlin 쪽도 보겠습니다. 먼저는 순차 처리 버전입니다. 기본 구조는 Go와 같고, 특별히 다른 점은 없습니다. 코드는 다음과 같습니다.
```kotlin
fun callGoServer(): List<CallGoServerDto> {
    logger.info("[CallGoServer] start")
    return tries.map {
        logger.info("[CallGoServer] before request with id: $it")
        goServerClient.call(it) ?: CallGoServerDto(it, "failed") // Go API를 호출한다
        .also { result ->
            logger.info("[CallGoServer] after request with id: ${result.id}")
        }
    }.also {
        logger.info("[CallGoServer] done")
    }
}
```

Curl로 실행해 본 결과는 다음과 같습니다.
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

여기도 Go와 마찬가지로 요청부터 응답까지 50초 정도 걸립니다. 이제 이걸 Coroutine으로 병렬화해 보겠습니다.
#### Coroutine으로 병렬화 (1)

Go와 달리 Kotlin의 Coroutine은 언어 기본 기능이 아닙니다. 그래서 먼저 의존성을 추가해야 합니다. 공식 설명만 보면 `coroutine-core`만 넣어도 될 것 같지만, Spring처럼 Reactive Stream이 필요한 경우에는 `coroutine-reactor`도 함께 넣어야 합니다.

의존성을 추가한 뒤 코드를 수정합니다. 여기서는 Spring Boot를 쓰고 있으므로 Controller 함수를 `suspend`로 만들 수 있고, 그에 맞춰 호출받는 함수도 `suspend`로 바꿉니다. 또 Coroutine은 스코프가 필요하니 루프를 [coroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html)로 감쌉니다. 그다음 `map` 안에서 [async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html)로 API를 호출하고, `map` 결과는 [Deferred](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html)로 돌아오므로 [awaitAll](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/await-all.html)로 완료를 기다립니다. 말로 풀면 길지만 코드를 보면 훨씬 간단합니다.
```kotlin
suspend fun callGoServerAsync(): List<CallGoServerDto> {
    logger.info("[CallGoServerAsync] start")
    return coroutineScope { // coroutine으로 처리한다
        tries.map {
            async { // 병렬로 실행한다
                logger.info("[CallGoServerAsync] before request with id: $it")
                goServerClient.call(it) ?: CallGoServerDto(it, "failed")
            }
        }.awaitAll() // API 호출 결과를 기다린다
            .also {
                it.forEach { result ->
                    logger.info("[CallGoServerAsyncDual] after request with id: ${result.id}")
                }
                logger.info("[CallGoServerAsyncDual] done")
            }
    }
}
```

또 API를 호출하는 함수(`goServerClient.call()`)도 `suspend`여야 합니다. 여기서는 Spring의 [RestTemplate](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)을 사용해 다음과 같이 정의했습니다.
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

코드를 이렇게 바꾸고 실행해 보면 Go와 마찬가지로 10건의 요청이 병렬로 나가는 것을 확인할 수 있습니다. 다만 goroutine과 달리 실행 순서가 보장된다는 점이 다릅니다. 그래서 Kotlin 쪽은 응답을 따로 정렬할 필요가 없습니다.
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

#### Coroutine으로 병렬화 (2)

이 코드도 Go와 마찬가지로 요청을 한꺼번에 모두 보내는 건 위험할 수 있으니, 동시 실행 수를 제한하겠습니다. Kotlin에서도 Coroutine의 동시 실행 수를 제한하는 메커니즘이 있는데, 바로 [Semaphore](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-semaphore/index.html)입니다.

Semaphore에 값을 넣고, `async` 안에서 그 값을 기준으로 실행 수를 제한하면 됩니다. 코드는 다음과 같습니다.
```kotlin
suspend fun callGoServerAsyncDual(): List<CallGoServerDto> {
    logger.info("[CallGoServerAsyncDual] start")
    val semaphore = Semaphore(2) // 동시 실행 수를 제한할 Semaphore를 정의한다
    return coroutineScope {
        tries.map {
            async {
                semaphore.withPermit { // async의 동시 실행 수를 Semaphore 값으로 제한한다
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

쓰기 방식만 조금 다를 뿐, Go와 거의 같은 감각으로 `async`의 실행 수를 제한할 수 있습니다. 컴파일 에러도 잘 안 나기 때문에 오히려 착각하기 쉬운 부분이니, `async { semaphore.withPermit { } }`의 순서를 잘 지켜야 합니다. 실행 결과는 다음과 같습니다.
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

로그를 찍는 위치가 조금 애매하긴 하지만, 요청이 나가는 시간을 보면 5초 간격으로 2개씩 보내고 있는 걸 확인할 수 있습니다.
## 마지막으로

Coroutine을 아주 깊이 이해하고 있는 편은 아니라서, 더 나은 방법은 충분히 있을 수 있습니다. 예를 들어 goroutine 실행 순서를 더 정리하거나, `WaitGroup.Done()`을 `defer`로 두거나, Kotlin 쪽 로그 위치를 더 다듬는 식의 개선은 가능했을 것입니다. 그래도 이 정도만으로도 API 호출을 비교적 간단하게 병렬화할 수 있다는 점을 확인한 것만으로 수확은 충분했습니다. Jetpack Compose를 만지며 coroutine을 접한 적은 있었지만, 이렇게 업무 맥락에서 직접 조사하고 검증한 것은 처음이라 더 의미가 있었습니다. 각 언어를 써 본 인상은 다음과 같습니다.
- Go
  - 의존성을 추가하지 않고 바로 쓸 수 있다는 점이 장점
  - Kotlin처럼 `suspend`나 scope를 의식하지 않아도 되어 편리
- Kotlin
  - `async`에서도 실행 순서가 보장되는 점이 장점
  - goroutine보다 신경 써야 할 부분이 많다

두 언어를 비교해 보면 장단점은 분명합니다. 그래도 둘 다 비교적 짧은 코드로 실전에 가까운 형태를 빠르게 만들 수 있다는 점은 큰 장점이라고 느꼈습니다. 앞으로도 Coroutine으로 성능을 더 개선할 수 있는 지점이 있는지 계속 찾아볼 생각입니다. 참고로 이 글에 들어간 전체 코드는 [이 저장소](https://github.com/retheviper/AsyncServerExample)에서 확인할 수 있습니다.
[^주]: 엄밀히 말하면, Coroutine에 의한 처리는 멀티스레드에 의한 병렬화와는 개념적으로는 다른 것입니다만, 구현과 결과의 취득이라고 하는 면에서는 감각이 크게 변하지 않기 때문에, Concurrency와 Parallelism에 의한 차이등의 이론적인 이야기는 생략하고 있습니다.
