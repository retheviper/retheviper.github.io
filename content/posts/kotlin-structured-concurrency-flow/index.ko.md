---
title: "Coroutines 입문을 넘어: Structured Concurrency와 Flow"
date: 2025-10-22
categories:
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - coroutine
  - flow
  - structured-concurrency
translationKey: "posts/kotlin-structured-concurrency-flow"
---

Kotlin의 Coroutine은 이제 안드로이드와 서버 사이드에서 자연스러운 선택지가 됐지만, 단순히 `launch`하고 `await`하는 단계를 넘어서면 설계의 자유도만큼 함정도 빠르게 늘어납니다. 이 글에서는 Structured Concurrency를 중심축으로 Coroutine 스코프를 어떻게 구성할지, 그리고 Flow를 사용한 리액티브 처리에서 성능과 관측 가능성을 어떻게 함께 가져갈지 정리해 보겠습니다.

여기서 다루는 내용은 복잡한 병렬 처리를 리팩터링할 때나, 다른 언어의 비동기 기반에서 옮겨온 팀이 설계 기준을 정리하고 싶을 때 특히 도움이 됩니다.
## Structured Concurrency를 다시 정리하기

Structured Concurrency는 "시작한 Coroutine이 반드시 어딘가의 부모에게 매달려 있고 그 수명이 부모보다 길지 않을 것"을 보장하는 설계 철학입니다. 부모와 자식 관계를 명시하면 취소 및 예외 전파를 읽을 수 있으며 리소스 누수를 방지합니다. Gradle이나 Ktor 등의 공식 라이브러리가 철저히 지키고 있는 규칙이기도 합니다.
안티 패턴으로 자주 보이는 것이 `GlobalScope.launch`로 작업을 던져 버리는 코드입니다.
```kotlin
fun refreshUserData() {
    GlobalScope.launch {
        val profile = api.fetchProfile()
        cache.store(profile)
    }
}
```

이 방식은 호출자와 Coroutine의 수명을 분리해 버리고, API 호출이 실패해도 예외가 흘러가는 경로가 흐려지며, 테스트에서도 완료 시점을 제대로 기다리기 어렵습니다. 기본 전략은 `suspend` 함수 안에서 `coroutineScope`를 사용해 완료와 취소의 경계를 명시하는 것입니다.
```kotlin
suspend fun refreshUserData(): Unit = coroutineScope {
    val profile = async { api.fetchProfile() }
    val posts = async { api.fetchPosts() }
    cache.store(profile.await(), posts.await())
}
```

`coroutineScope`은 내부에서 기동한 모든 아이 Coroutine가 완료할 때까지 돌아오지 않습니다. 어느 하나라도 실패하면 나머지는 취소되고 예외는 호출자에게 그대로 전해집니다. 이것이 Structured Concurrency의 기본 형태입니다.
### coroutineScope와 supervisorScope의 차이

반면에 `supervisorScope`은 하위 Coroutine 실패가 다른 하위로 전파되지 않는 점이 다릅니다. 부분 실패를 허용하려면 선택합니다.
```kotlin
suspend fun loadDashboard(userId: String): Dashboard = supervisorScope {
    val profile = async { profileRepository.load(userId) }
    val timeline = async { timelineRepository.load(userId) }

    val safeTimeline = try {
        timeline.await()
    } catch (e: Throwable) {
        emptyList() // 표시는 계속 유지하면서 로그로 가시화
    }

    Dashboard(profile.await(), safeTimeline)
}
```

`supervisorScope`에서도 예외를 그냥 무시하는 것이 아니라, 잡아서 로그를 남기거나 폴백을 적용하겠다는 의도를 코드에 명시해야 합니다. `SupervisorJob`을 이용한 스코프도 같은 동작을 만들 수 있지만, `suspend` 함수 안에서 완결하고 싶다면 `supervisorScope` 쪽이 더 다루기 쉽습니다.
## 범위 설계 연습 패턴

제품에서 Structured Concurrency를 철저히 할 때 자주 하는 스코프 설계를 세 가지로 나누어 생각합니다.
### 요청 단위 범위

Ktor와 Spring WebFlux와 같은 비동기 서버에서는 요청마다 `coroutineScope`을 닫는 패턴이 기본입니다. `withTimeout`에서 SLO를 보호하면서 여러 IO 처리를 병렬화합니다.
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
            metrics = metrics // 가용성 우선으로 폴백
        )
    }
}
```

`awaitOrNull()`과 같은 확장을 준비하면 부분적인 실패를 쉽게 흡수할 수 있습니다. `supervisorScope`로 하고 있기 때문에, 어느 쪽이든 하나의 실패로 모든 것이 되감기를 방지하면서, 외측의 `withTimeout`로 상한을 제어할 수 있습니다.
### UI/프레젠테이션 레이어의 범위

Android에서는 `viewModelScope`이나 `lifecycleScope` 등 라이브러리가 범위를 제공하지만, Structured Concurrency를 의식하지 않으면 취소 누설이 발생하기 쉽습니다. `repeatOnLifecycle`에서 라이프사이클과 Coroutine을 동기화하면 화면이 숨겨지는 동안 여분의 처리를 실행하지 않고 완료됩니다.
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

`stateIn`이 내부에서 `viewModelScope`을 사용하고 있음을 잊지 말고 `SharingStarted.WhileSubscribed`에서 원하지 않을 때 중지하도록 설정하는 것이 포인트입니다.
### 배경 작업자 및 한계 제어

장시간 움직이는 동기 처리나 배치 처리에서는, 자전으로 `CoroutineScope`을 준비하는 장면이 나옵니다. `SupervisorJob`과 `CoroutineName`을 더하면 로그 관측에서도 쉽게 식별할 수 있습니다.
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

`Semaphore` 또는 `Dispatchers.Default.limitedParallelism(n)`과 결합하여 Structured Concurrency 프레임워크에서 백 프레셔를 표현할 수 있습니다.
## Flow에서 리액티브 처리 최적화

Flow는 Coroutine과 같은 철학 위에서 설계되었기 때문에 Structured Concurrency의 관점을 거의 그대로 적용할 수 있습니다. 핵심은 "데이터가 흐르는 범위의 경계를 어디에 둘 것인가"를 코드에 분명히 남기는 일입니다.
### Flow 파이프라인 및 Structured Concurrency

`flow {}`의 블록 내에서는 암묵적으로 `coroutineScope`이 붙여집니다. 여러 IO를 병렬로 수행하고 싶을 때 명시적으로 `coroutineScope`을 넣으면 전체 파이프라인의 가독성이 향상됩니다.
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

`flowOn`에서 실행 스레드를 전환할 때는, `coroutineScope` 안쪽에서만 IO를 하고, 외부에서 UI용의 성형을 하는 등 책무를 분명히 나누어 두면 나중에 이유 붙이기 쉬워집니다.
`channelFlow`을 사용하면 여러 하위 Coroutine에서 이벤트를 동시에 전달할 수 있습니다. `awaitClose`에서 자원 해제를 잊지 않도록 합시다.
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

### Cold Flow와 Hot Flow의 구분

Cold Flow는 수집될 때마다 처리가 달리기 때문에, 무거운 처리의 경우는 `stateIn`나 `shareIn`로 핫화하는 것이 정석입니다. `SharingStarted.WhileSubscribed` 또는 `Eagerly`의 사용으로 캐시 수명과 리소스 소비를 조정합니다.
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

`WhileSubscribed`에 유예 시간을 설정하면 화면을 다시 표시할 때마다 여분의 새로 고침을 실행하지 않고 완료됩니다.
### 오류 처리 및 역압

Flow는 예외를 업스트림으로 전파하므로 `catch` 및 `retryWhen`에서 명시적으로 처리합니다. 또한 `buffer` 및 `conflate`에서 속도 차이를 흡수하면 다운 스트림 취소 빈도를 줄일 수 있습니다.
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

`retryWhen`에서 지수 백오프를 구현할 때 `delay`을 잊지 않고 넣어 CPU 스핀을 피하십시오. Structured Concurrency의 범위 내에서 재시도하고 있으므로 외부가 취소되면 즉시 중지됩니다.
## 관측과 테스트로 설계 강화

Structured Concurrency를 채택하면 테스트하기 쉬워지는 것도 장점입니다. `kotlinx-coroutines-test`의 `runTest`과 `TestScope`을 사용하면 하위 Coroutine의 완료를 기다리면서 시뮬레이션 할 수 있습니다.
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

테스트를 위한 스코프를 생성자에서 수신하도록 하면 취소 전파를 확실하게 검증할 수 있습니다. 또한 `CoroutineName`을 붙여두면 로그에서도 추적하기 쉬워집니다.
## 요약

- Structured Concurrency는 Coroutine의 수명을 제어하는 ​​기반이므로 `coroutineScope`과 `supervisorScope`의 구분을 먼저 굳힌다.
- Flow의 병렬화는 `coroutineScope`과 `channelFlow`에 의해 제어되고 `stateIn` 및 `shareIn`에서 캐시 전략을 명시한다.
- 테스트와 감시로 취소 전파나 재시도 거동을 가시화하여 실운용으로 무너지지 않도록 한다.

초보자 단계를 넘어서는 코루틴 설계에서는 여러 스코프가 교차하는 순간을 어떻게 설명하고 통제하느냐에 따라 품질이 크게 달라집니다. 구조적 동시성 개념을 팀의 공통 언어로 만들 수 있다면 코드 리뷰와 문제 해결 속도도 훨씬 빨라집니다.
