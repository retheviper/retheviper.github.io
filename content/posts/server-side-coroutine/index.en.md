---
title: "Use Coroutine in Backend"
date: 2022-06-05
translationKey: "posts/server-side-coroutine"
categories: 
  - languages
image: "../../images/magic.webp"
tags:
  - kotlin
  - go
  - coroutine
  - goroutine
  - concurrency
---

When using a GUI such as an Android app, it is common knowledge to use multithreading. If it is single-threaded, the screen will freeze while something is being processed. In addition, there are many cases where it is necessary to update the state of components that change in real time, such as a progress bar, or to handle processing in separate threads for various situations such as chat and displaying notifications.

However, things are a little different when it comes to back-end processing. There may be no need to consider the GUI in the first place, but since servers often process a single request "sequentially", it is difficult to take advantage of the benefits of processing distribution using multithreading. There are things like [Reactive Streams](https://www.reactive-streams.org), but this is not so much suited for distributing a single request as it is for processing many requests with fewer resources, so it is not suitable if you want to increase efficiency by distributing a single process.

Of course, this does not mean that there is no need for distributed processing on the back end. It is true that even when processing a single request, there are times when it is possible to improve performance by dividing the threads depending on the processing. For example, if there is a process in between that is unrelated to the subsequent process, you may want to process it in a separate thread.

So this time I would like to introduce an example of distributing backend processing using [coroutines](https://en.wikipedia.org/wiki/Coroutine). [^Note]

## Parallelize API calls

First, consider batch processing as a case where parallelization can improve efficiency. Batch processing often involves extracting multiple pieces of data that meet a condition and repeating the same process on each piece of data. In this way, if the processing for each piece of data is executed independently and there is no particular problem with running it in parallel, the processing can be sufficiently distributed.

At work, I regularly extract data to be processed from a DB based on date from a server created in Go, and while looping that data into an array, I call the API of the server created in Kotlin. In addition, there are cases where the Kotlin server also calls the Backend API, which is also a state in which data is referenced in a loop. As the service grew, we started experiencing performance issues such as API calls timing out due to the processing time, so we decided to parallelize the API calls within this loop to reduce the processing time.

## Try implementing it

First, let's use Coroutine to call the API within a loop. As mentioned above, this code was written to verify whether it can be used in actual work, so the processing on each server is not much different. First, we write a process that calls each other's API in a loop, and the called side waits for 5 seconds before sending a response. We will parallelize this using Coroutine.

## Go

First, suppose we have the following process.

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

        r, err := client.Post(i) // Send a POST request to the Kotlin server
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

The above code is a sample server using [Gin](https://gin-gonic.com), and is the handler part. Inside this function, when the API is called, it sends a request to the Kotlin server in a loop of 10 times. Then, a response struct is created with the returned API call results, and the final result is a structure in which the results of the 10 executions are returned as JSON.

Here, the response returned by Kotlin takes 5 seconds, so the more times the loop is repeated, the slower the response will be returned. Since the log is being output, if you check the server log, you can see that it takes 50 seconds from request to response.

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

### Parallelize with Goroutine (1)

Now, let's parallelize the above processing. Go basically includes [Goroutine](https://go-tour-jp.appspot.com/concurrency/1). It is simple to use, just add the `go` keyword before the function you want to execute. However, the response needs to wait for the results of 10 executions before returning it, but if you call the API with a goroutine, the main thread may end first.

So, we will use a goroutine to call the API in the loop, and then return the result after all the goroutines have finished. Go has [WaitGroup](https://pkg.go.dev/sync#WaitGroup) in a package called `sync`, which allows you to wait for the goroutine to finish. Also, when executing a goroutine in a loop, the order will be random, so make sure to sort it once when returning the response. The results of implementation with the above considerations in mind are as follows.

```go
func CallKotlinServerAsync(c *gin.Context) {
    log.Print("[CallKotlinServerAsync] start")

    results := &callResults{}
    tries := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    group := &sync.WaitGroup{} // Define a WaitGroup

    for _, i := range tries {
        group.Add(1) // Add one goroutine per loop iteration

        go func(i int) { // Call the API in a goroutine
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

            group.Done() // Mark the goroutine as finished in the WaitGroup
        }(i)
    }

    group.Wait() // Wait for all goroutines to finish
    sort.Slice(results.Response, func(i, j int) bool {
        return results.Response[i].ID < results.Response[j].ID
    })

    log.Print("[CallKotlinServerAsync] done")

    c.JSON(http.StatusOK, results)
}
```

The resulting log after making this modification is as follows:

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

You can see that it takes about 5 seconds to respond because the 10 loops were executed almost simultaneously. And you can see that the goroutines are not executed sequentially. Therefore, even if the order of execution is not important, it is clear that sorting is still necessary when results need to be returned in order.

### Parallelize with Goroutine (2)

In some cases, it may be dangerous to run all processes at the same time, even if they can be parallelized. In the above code, the number of requests is 10, but what if you need more requests or call an API that requires more heavy processing? Since the Go side only makes requests, the processing load will not change much, but it will be a considerable load on the side where the API is being called.

If so, it is necessary to limit the number of parallel operations. For example, if you set the number of parallel requests to 2, requests will be sent two at a time, so the load will remain constant no matter how much the total number of requests increases. Although it is slower than sending all requests at the same time, you can respond flexibly by simply increasing the number of parallel requests while monitoring the resource situation, so being able to specify the number of parallel requests using an external configuration file has the advantage of allowing you to respond flexibly without having to build the app.

If you use threads, you will probably have to write a fairly complex process to perform this kind of processing. For example, you need to define threads according to the number of parallel operations, and then separate the processing assigned to each thread. Since we are currently assuming 10 requests, we can handle this by simply dividing the requests into 5 requests per thread, but we first need to add processing to divide the number of loop elements accordingly, taking into account the result of dividing the number of requests by the number of threads.

However, in reality, if you use goroutine, there is no need to add such complicated processing. With goroutines, you can use [Channel](https://go-tour-jp.appspot.com/concurrency/2) to specify the number of goroutines that will be executed at the same time. It is as below.

```go
func CallKotlinServerAsyncDual(c *gin.Context) {
    log.Print("[CallKotlinServerAsyncDual] start")

    results := &callResults{}
    tries := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    concurrency := 2 // Set the goroutine concurrency limit
    group := &sync.WaitGroup{}
    guard := make(chan struct{}, concurrency) // Define a channel with the concurrency limit

    for _, i := range tries {

        group.Add(1)
        guard <- struct{}{} // Add one running slot to the channel

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
            <-guard // Release the slot in the channel
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

If you send a specified number of minutes to the Channel, new goroutines will be blocked from running until they receive a value from the Channel. So, when you actually execute it, you can see that up to two requests are being sent as intended.

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

## Kotlin

First, let's look at the code for sequential processing. Basically, the same processing as in Go is available on the Kotlin side, so there is nothing particularly different. Below is the code.

```kotlin
fun callGoServer(): List<CallGoServerDto> {
    logger.info("[CallGoServer] start")
    return tries.map {
        logger.info("[CallGoServer] before request with id: $it")
        goServerClient.call(it) ?: CallGoServerDto(it, "failed") // Call the Go API
        .also { result ->
            logger.info("[CallGoServer] after request with id: ${result.id}")
        }
    }.also {
        logger.info("[CallGoServer] done")
    }
}
```

The results of running it with Curl are as follows.

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

As with Go, you can see that it takes about 50 seconds from request to response. Let's parallelize this using Coroutine.

### Parallelize with Coroutine (1)

Unlike Go, Coroutines in Kotlin are not a fundamental specification of the language. So we need to add dependencies first. However, according to the official explanation, it seems like it can be handled by adding only `coroutine-core`, but if you need Reactive Stream like Spring, you need to add `coroutine-reactor` to the dependencies.

After adding the dependencies, we will modify the code. Since we are using Spring Boot here and the Controller function can be marked `suspend`, we will also suspend the function called from the Controller. Also, because coroutine processing needs an explicit scope, we wrap the loop in [coroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html). After that, we call the API as [async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) in the `map` function, and since the result of `map` is returned as [Deferred](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html), we wait for completion with [awaitAll](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/await-all.html). The explanation is complicated, but I think it will be easier to understand if you look at the code below.

```kotlin
suspend fun callGoServerAsync(): List<CallGoServerDto> {
    logger.info("[CallGoServerAsync] start")
    return coroutineScope { // Run this as a coroutine
        tries.map {
            async { // Run in parallel
                logger.info("[CallGoServerAsync] before request with id: $it")
                goServerClient.call(it) ?: CallGoServerDto(it, "failed")
            }
        }.awaitAll() // Wait for the API call results
            .also {
                it.forEach { result ->
                    logger.info("[CallGoServerAsyncDual] after request with id: ${result.id}")
                }
                logger.info("[CallGoServerAsyncDual] done")
            }
    }
}
```

Also, the function (`goServerClient.call()`) that calls the API must be suspended. Here, we used Spring's [RestTemplate](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html) and defined the following functions.

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

If you modify the code as above and run it, you will see that 10 requests are being sent in parallel, just like in Go. However, the difference is that unlike goroutines, the order of execution is guaranteed. Due to this feature, response sorting is not necessary in Kotlin.

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

### Parallelize with Coroutine (2)

As with Go, in the above code, sending all requests at the same time may cause problems, so we will limit the number of requests sent at the same time. Just like in Go, Kotlin has a mechanism to limit the number of coroutines that can run concurrently. It's called [Semaphore](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-semaphore/index.html).

It is a form that limits the number of parallel executions by specifying a numerical value for Sempaphore and limiting the number of executions by the number specified for Semaphore in async. Below is the code.

```kotlin
suspend fun callGoServerAsyncDual(): List<CallGoServerDto> {
    logger.info("[CallGoServerAsyncDual] start")
    val semaphore = Semaphore(2) // Define a semaphore to limit concurrency
    return coroutineScope {
        tries.map {
            async {
                semaphore.withPermit { // Limit async concurrency to the semaphore count
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

I have created a code that can restrict async processing in almost the same way as Go, just written in a slightly different way. I don't think there are any compilation errors, so it's easy to get confused. Please note that it is necessary to follow the order of `async{ semaphore.withPermit{ } }` properly. The execution results are as follows.

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

The location where the logs were sent was a little strange, but if you look at the time the requests were being sent, you'd see that two requests were being sent every 5 seconds.

## lastly

I don't know much about coroutines, so I think there could have been a better way to write them (determining the order of execution of goroutines, defining `WaitGroup.Done()` in `defer`, adjusting the log output location in Kotlin, etc.), but I'm personally quite satisfied with how I found that I could easily parallelize API calls. Although I had briefly touched on coroutine while using Jetpack Compose, this was the first time I had to research and verify it as needed for work, so I can say that I gained a lot from it. Also, the impressions in each language are as follows.

- Go
  - The advantage is that it can be used without adding dependencies.
  - Convenient because you don't have to be aware of suspend and scope like in Kotlin
- Kotlin
  - The advantage is that the order of execution is guaranteed even with async.
  - There are more precautions than goroutine.

When I compare the two languages, I feel that each has its own advantages and disadvantages, but neither is difficult to apply, so I got the impression that it is definitely a good thing to be able to write something that can go straight into production code. I would like to continue experimenting with coroutines to see if there are any areas where performance can be improved. By the way, the entire source code from this article can be viewed in [this repository](https://github.com/retheviper/AsyncServerExample).

See you soon!

[^Note]: Strictly speaking, processing using coroutines is conceptually different from parallelization using multi-threading, but in terms of implementation and obtaining results, the feeling is not much different, so we have omitted theoretical discussions such as the differences between concurrency and parallelism.
