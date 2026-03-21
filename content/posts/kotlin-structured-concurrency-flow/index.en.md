---
title: "Beyond a Coroutine Introduction: Structured Concurrency and Flow"
date: 2025-10-22
categories:
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - coroutine
  - flow
  - structured-concurrency
---
Coroutines in Kotlin are now a common choice in Android and server-side settings, but once you go beyond the "just launch and await" stage, the degree of freedom in design expands at once, but at the same time, the pitfalls also increase. In this article, we will take the introduction one step further and explain how to assemble a coroutine scope based on the idea of ​​Structured Concurrency, and how to balance performance and observability with reactive processing using Flow.

This topic is especially useful when refactoring complex concurrency, or when a team is transitioning from an asynchronous foundation in another language and wants to solidify its design principles.

## Revisit Structured Concurrency

Structured Concurrency is a design philosophy that ensures that a coroutine that starts hangs onto a parent somewhere and never outlives its parent. By clearly indicating parent-child relationships, cancellation and exception propagation become easier to read, and resource leaks can be prevented. This is also a rule that official libraries such as Gradle and Ktor strictly follow.

A common anti-pattern I see is code that keeps throwing work away at `GlobalScope.launch`.

```kotlin
fun refreshUserData() {
    GlobalScope.launch {
        val profile = api.fetchProfile()
        cache.store(profile)
    }
}
```

This way of writing separates the lifespan of the caller and the coroutine, the exception will be suppressed even if the API call fails, and the test cannot wait for convergence. The basic strategy is to use `coroutineScope` within the `suspend` function to clearly mark completion and cancellation boundaries.

```kotlin
suspend fun refreshUserData(): Unit = coroutineScope {
    val profile = async { api.fetchProfile() }
    val posts = async { api.fetchPosts() }
    cache.store(profile.await(), posts.await())
}
```

`coroutineScope` will not return until all child coroutines started within it have completed. If any one fails, the rest are canceled and the exception is passed directly to the caller. This is the basic form of Structured Concurrency.

### The Difference Between `coroutineScope` and `supervisorScope`

On the other hand, `supervisorScope` differs in that the failure of a child Coroutine is not propagated to other children. Select this if you want to tolerate partial failures.

```kotlin
suspend fun loadDashboard(userId: String): Dashboard = supervisorScope {
    val profile = async { profileRepository.load(userId) }
    val timeline = async { timelineRepository.load(userId) }

    val safeTimeline = try {
        timeline.await()
    } catch (e: Throwable) {
        emptyList() // Keep the UI running while surfacing the failure in logs
    }

    Dashboard(profile.await(), safeTimeline)
}
```

In `supervisorScope`, instead of completely ignoring exceptions, the code is responsible for catching and logging/falling back to them. The scope using `SupervisorJob` has the same behavior, so it doesn't matter which one you choose, but if you want to complete it within the `suspend` function, `supervisorScope` is easier to write.

## Practical Scope Design Patterns

When implementing structured concurrency in a product, we will divide scope design into three categories.

### Per-Request Scope

In asynchronous servers such as Ktor and Spring WebFlux, the basic pattern is to close the `coroutineScope` for each request. Use `withTimeout` to stay within your SLO while parallelizing multiple I/O operations.

```kotlin
suspend fun fetchUserSummary(userId: String): UserSummary = withTimeout(1200) {
    supervisorScope {
        val profileDeferred = async { profileClient.fetch(userId) }
        val ordersDeferred = async { orderClient.fetchRecent(userId) }
        val metricsDeferred = async { metricsClient.fetch(userId) }
        val metrics = runCatching { metricsDeferred.await() }.getOrNull()

        UserSummary(
            profile = profileDeferred.await(),
            orders = ordersDeferred.await(),
            metrics = metrics // Fall back here to prioritize availability
        )
    }
}
```

Having an extension like `awaitOrNull()` helps absorb partial failures. Since it is set to `supervisorScope`, the upper limit can be controlled with the outer `withTimeout` while preventing everything from rolling back due to one failure.

### UI/Presentation Layer Scope

In Android, libraries such as `viewModelScope` and `lifecycleScope` provide scopes, but if you are not aware of Structured Concurrency, it is easy to miss a cancellation. Synchronizing the lifecycle and coroutine with `repeatOnLifecycle` avoids running extra processing while the screen is hidden.

```kotlin
class DashboardViewModel(
    private val repository: DashboardRepository,
    private val scope: CoroutineScope = viewModelScope
) : ViewModel() {

    val uiState: StateFlow<DashboardUiState> =
        repository.observeDashboard()
            .stateIn(scope, SharingStarted.WhileSubscribed(5_000), DashboardUiState.Loading)

    fun refresh() {
        scope.launch {
            coroutineScope {
                launch { repository.refreshProfile() }
                launch { repository.refreshTimeline() }
            }
        }
    }
}
```

The key is to remember that `stateIn` internally uses `viewModelScope` and set it to stop at `SharingStarted.WhileSubscribed` when it is not needed.

### Background Workers and Concurrency Limits

In synchronous processing or batch processing that runs for a long time, there will be situations where you will need to prepare `CoroutineScope` yourself. Adding `SupervisorJob` and `CoroutineName` will make it easier to identify them in log observation.

```kotlin
class SyncWorker(
    private val repository: SyncRepository
) {
    private val scope = CoroutineScope(
        SupervisorJob() + Dispatchers.Default + CoroutineName("SyncWorker")
    )
    private val semaphore = Semaphore(permits = 2)

    fun schedule(entityId: String) {
        scope.launch {
            semaphore.withPermit {
                repository.sync(entityId)
            }
        }
    }
}
```

By combining with `Semaphore` and `Dispatchers.Default.limitedParallelism(n)`, back pressure can be expressed within the framework of Structured Concurrency.

## Optimizing Reactive Processing with Flow

Flow is designed in the same context as Coroutine, so you can use the idea of Structured Concurrency as is. The key is to clearly indicate where the boundaries of the scope for data flow should be placed.

### Flow Pipelines and Structured Concurrency

`coroutineScope` is implicitly extended within the `flow {}` block. When you want to perform multiple IOs in parallel, explicitly inserting `coroutineScope` will improve the readability of the entire pipeline.

```kotlin
fun trendingFlow(): Flow<TrendingFeeds> = flow {
    val feeds = coroutineScope {
        val daily = async { api.fetchDaily() }
        val weekly = async { api.fetchWeekly() }
        transform(daily.await(), weekly.await())
    }
    emit(feeds)
}.flowOn(Dispatchers.IO)
```

When switching execution threads with `flowOn`, it is easier to reason about the code later if responsibilities are clearly separated, for example by doing I/O only inside `coroutineScope` and UI formatting outside it.

`channelFlow` lets you emit events from multiple child coroutines at the same time. Do not forget to release resources in `awaitClose`.

```kotlin
fun observeDashboard(userId: String): Flow<DashboardEvent> = channelFlow {
    val job = launch {
        repository.observeProfile(userId).collect { send(DashboardEvent.Profile(it)) }
    }
    launch {
        repository.observeTimeline(userId).collect { send(DashboardEvent.Timeline(it)) }
    }
    awaitClose { job.cancel() }
}
```

### Choosing Between Cold Flow and Hot Flow

Since Cold Flow is processed every time it is collected, it is standard practice to heat it up with `stateIn` or `shareIn` for heavy processing. Adjust cache lifespan and resource consumption by using `SharingStarted.WhileSubscribed` and `Eagerly`.

```kotlin
val sharedDashboard: StateFlow<DashboardUiState> =
    repository.observeDashboard()
        .retryWhen { cause, attempt ->
            cause is IOException && attempt < 3
        }
        .stateIn(
            scope = applicationScope,
            started = SharingStarted.WhileSubscribed(stopTimeoutMillis = 10_000),
            initialValue = DashboardUiState.Loading
        )
```

By setting a grace time in `WhileSubscribed`, you can avoid running an extra refresh every time the screen is redisplayed.

### Error Handling and Backpressure

Flow propagates exceptions upstream, so they are handled explicitly with `catch` and `retryWhen`. Furthermore, by absorbing the speed difference with `buffer` and `conflate`, the frequency of downstream cancellations can be suppressed.

```kotlin
fun updates(): Flow<UpdateEvent> =
    sourceFlow
        .buffer(capacity = Channel.BUFFERED / 2)
        .conflate()
        .retryWhen { cause, attempt ->
            delay(attempt * 200L)
            cause is RetryableException && attempt < 5
        }
        .catch { emit(UpdateEvent.Fallback) }
```

When implementing exponential backoff with `retryWhen`, remember to include `delay` to avoid CPU spins. Since we are retrying within the Structured Concurrency, if the outside is canceled, it will stop immediately.

## Solidify the Design with Observability and Tests

Another advantage of using Structured Concurrency is that it is easier to test. By using `runTest` and `TestScope` of `kotlinx-coroutines-test`, you can simulate while waiting for the child coroutine to complete.

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class DashboardViewModelTest {
    private val testDispatcher = StandardTestDispatcher()

    @Test
    fun refreshCancelsChildrenWhenScopeCancelled() = runTest(testDispatcher) {
        val repository = FakeDashboardRepository()
        val viewModel = DashboardViewModel(repository, scope = backgroundScope)

        val job = launch { viewModel.refresh() }
        advanceTimeBy(100)
        job.cancelAndJoin()

        assertTrue(repository.profileCancelled)
        assertTrue(repository.timelineCancelled)
    }
}
```

By having the constructor receive a test scope, you can reliably verify cancellation propagation. Furthermore, if you add `CoroutineName`, it will be easier to track it in the log.

## Summary

- Structured Concurrency is the basis for controlling the coroutine's lifespan, so first establish the proper use of `coroutineScope` and `supervisorScope`.
- Control the parallelization of Flow using `coroutineScope` and `channelFlow`, and specify the cache strategy using `stateIn` and `shareIn`.
- Visualize cancellation propagation and retry behavior through testing and monitoring to ensure that they do not break down in actual operation.

In coroutine design that goes beyond beginners, the quality depends on how you can explain the moment when multiple scopes intersect. Making the concept of Structured Concurrency a common language will definitely speed up code reviews and troubleshooting.

See you soon!
