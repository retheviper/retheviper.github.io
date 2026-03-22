---
title: "Coroutines入門を越えて──Structured ConcurrencyとFlow"
date: 2025-10-22
translationKey: "posts/kotlin-structured-concurrency-flow"
categories:
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - coroutine
  - flow
  - structured-concurrency
---

KotlinのCoroutineは今やAndroidやサーバーサイドの現場で当たり前の選択肢ですが、「launchしてawaitするだけ」の段階を越えると設計の自由度が一気に広がり、同時に落とし穴も増えます。本稿では入門を一歩進め、Structured Concurrencyの考え方を軸にCoroutineスコープをどう組み立てるか、さらにFlowを使ったリアクティブ処理で性能と可観測性をどう両立させるかを整理します。

ここで扱う内容は、複雑な並行処理をリファクタリングするときや、他言語の非同期基盤から移行してきたチームが設計指針を固めたいときに特に役立ちます。

## Structured Concurrencyを改めて整理する

Structured Concurrencyは「開始したCoroutineは必ずどこかの親にぶら下がり、その寿命が親より長くならない」ことを保証する設計哲学です。親子関係を明示しておくことで、キャンセルや例外伝播が読みやすくなり、リソースリークを防げます。GradleやKtorなどの公式ライブラリが徹底して守っているルールでもあります。

アンチパターンとしてよく見かけるのが、`GlobalScope.launch`で作業を投げっぱなしにするコードです。

```kotlin
fun refreshUserData() {
    GlobalScope.launch {
        val profile = api.fetchProfile()
        cache.store(profile)
    }
}
```

この書き方だと呼び出し元とCoroutineの寿命が切り離され、API呼び出しが失敗しても例外は握りつぶされますし、テストでも収束を待てません。基本戦略は、`suspend`関数の中で`coroutineScope`を使い、完了やキャンセルの境界を明示することです。

```kotlin
suspend fun refreshUserData(): Unit = coroutineScope {
    val profile = async { api.fetchProfile() }
    val posts = async { api.fetchPosts() }
    cache.store(profile.await(), posts.await())
}
```

`coroutineScope`は内部で起動した全ての子Coroutineが完了するまで戻りません。どれか一つでも失敗すると残りはキャンセルされ、例外は呼び出し元にそのまま伝わります。これがStructured Concurrencyの基本形です。

### coroutineScopeとsupervisorScopeの違い

一方、`supervisorScope`は子Coroutineの失敗が他の子に伝播しない点が異なります。部分的な失敗を許容したい場合に選択します。

```kotlin
suspend fun loadDashboard(userId: String): Dashboard = supervisorScope {
    val profile = async { profileRepository.load(userId) }
    val timeline = async { timelineRepository.load(userId) }

    val safeTimeline = try {
        timeline.await()
    } catch (e: Throwable) {
        emptyList() // 表示は継続しつつログで可視化
    }

    Dashboard(profile.await(), safeTimeline)
}
```

`supervisorScope`でも例外を完全に無視するのではなく、キャッチしてログに残す／フォールバックする責務をコードに明示します。`SupervisorJob`を用いたスコープも同じ振る舞いになるため、どちらを選んでも構いませんが、`suspend`関数内で完結させたい場合は`supervisorScope`が書きやすいです。

## スコープ設計の実践パターン

プロダクトでStructured Concurrencyを徹底する際に頻出するスコープ設計を三つに分けて考えます。

### リクエスト単位のスコープ

KtorやSpring WebFluxなど非同期サーバーでは、リクエストごとに`coroutineScope`を閉じるパターンが基本です。`withTimeout`でSLOを守りつつ、複数のIO処理を並列化します。

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
            metrics = metrics // 可用性重視でフォールバック
        )
    }
}
```

`awaitOrNull()`のような拡張を用意しておくと、部分的な失敗を吸収しやすくなります。`supervisorScope`にしているため、どれか一つの失敗で全てが巻き戻るのを防ぎつつ、外側の`withTimeout`で上限を制御できます。

### UI／プレゼンテーション層のスコープ

Androidでは`viewModelScope`や`lifecycleScope`などライブラリがスコープを提供してくれますが、Structured Concurrencyを意識しないとキャンセル漏れが発生しがちです。`repeatOnLifecycle`でライフサイクルとCoroutineを同期させると、画面が非表示の間に余分な処理を走らせずに済みます。

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

`stateIn`が内部で`viewModelScope`を使っていることを忘れず、`SharingStarted.WhileSubscribed`で不要時に停止するよう設定するのがポイントです。

### バックグラウンドワーカーと限界制御

長時間動く同期処理やバッチ処理では、自前で`CoroutineScope`を用意する場面が出てきます。`SupervisorJob`と`CoroutineName`を足しておくと、ログ観測でも識別しやすくなります。

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

`Semaphore`や`Dispatchers.Default.limitedParallelism(n)`と組み合わせることで、Structured Concurrencyの枠組みの中でバックプレッシャーを表現できます。

## Flowでリアクティブ処理を最適化する

FlowはCoroutineと同じ文脈で設計されているため、Structured Concurrencyの考え方をそのまま活かせます。ポイントは「データを流すスコープの境界をどこに置くか」を明示することです。

### FlowパイプラインとStructured Concurrency

`flow {}`のブロック内では暗黙に`coroutineScope`が張られます。複数のIOを並列で行いたいときは、明示的に`coroutineScope`を入れるとパイプライン全体の可読性が上がります。

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

`flowOn`で実行スレッドを切り替えるときは、`coroutineScope`の内側でのみIOを行い、外でUI向けの整形をするなど責務をはっきり分けておくと後から理由付けしやすくなります。

`channelFlow`を使うと、複数の子Coroutineからイベントを同時に送出できます。`awaitClose`でリソース解放を忘れないようにしましょう。

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

### Cold FlowとHot Flowの使い分け

Cold Flowは収集されるたびに処理が走るため、重たい処理の場合は`stateIn`や`shareIn`でホット化するのが定石です。`SharingStarted.WhileSubscribed`や`Eagerly`の使い分けで、キャッシュの寿命とリソース消費を調整します。

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

`WhileSubscribed`に猶予時間を設定しておくと、画面再表示のたびに余分なリフレッシュを走らせずに済みます。

### エラー処理とバックプレッシャー

Flowは例外を上流に伝播させるため、`catch`や`retryWhen`で明示的に扱います。さらに`buffer`や`conflate`で速度差を吸収すると、下流のキャンセル頻度を抑えられます。

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

`retryWhen`で指数バックオフを実装するときは、`delay`を忘れずに入れてCPUスピンを避けます。Structured Concurrencyの範囲内でリトライしているので、外側がキャンセルされれば即座に停止します。

## 観測とテストで設計を固める

Structured Concurrencyを採用するとテストしやすくなるのも利点です。`kotlinx-coroutines-test`の`runTest`や`TestScope`を使えば、子Coroutineの完了を待ちながらシミュレーションできます。

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

テスト用のスコープをコンストラクタで受け取るようにしておくと、キャンセル伝播を確実に検証できます。さらに`CoroutineName`を付けておけばログでも追跡しやすくなります。

## まとめ

- Structured ConcurrencyはCoroutineの寿命をコントロールする基盤なので、`coroutineScope`と`supervisorScope`の使い分けを最初に固める。
- Flowの並列化は`coroutineScope`や`channelFlow`で制御し、`stateIn`や`shareIn`でキャッシュ戦略を明示する。
- テストと監視でキャンセル伝播やリトライ挙動を可視化し、実運用で崩れないようにする。

入門を越えたCoroutine設計では、複数のスコープが交差する瞬間をどう説明できるかが品質を左右します。Structured Concurrencyの考え方を共通言語にすると、コードレビューや障害対応のスピードが確実に上がります。

では、また！
