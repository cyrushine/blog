---
title: Kotlin Flow - 初识
date: 2022-04-09 12:00:00 +0800
tags: [kotlin, coroutine, flow, sequence, 协程, 热流, 冷流]
---

# for 操作符

具有以下特征的对象可以被 `for` 操作符遍历：

1. 具有 `operator` 函数 `iterator()`
2. 其返回 `Iterator` 具有 `operator` 函数 `hasNext()` 和 `next()`

```kotlin
public interface Sequence<out T> {
    public operator fun iterator(): Iterator<T>
}

public interface Iterator<out T> {
    public operator fun next(): T
    public operator fun hasNext(): Boolean
}
```

`Sequence` 就是可以被 `for` 操作符遍历的接口

```kotlin
val fruits = sequence {
    yield("Apple")
    yield("Banana")
    yield("Guava")
}
for (fruit in fruits) { println("we got $fruit") }
```

# 热流（hot stream）和冷流（cold stream）

* 已计算出结果的序列叫 `热流`，平时常用的 `Collection`、`List`、`Set` 和 `Map` 等容器都属于热流
* 尚未计算出结果的序列叫 `冷流`，`sequence`、`flow` 和 `RxJava` 都可以创建冷流
* 冷流的概念类似于 ORM Entity 的懒加载以及其关联关系的懒加载，就是等用到成员时才执行 DB 查询

```kotlin
val log = { msg: String ->
    println("${SimpleDateFormat("HH:mm:ss").format(Date())} $msg")
}

val collect: suspend SequenceScope<String>.(String) -> Unit = {
    log("collecting ${it}...")
    Thread.sleep(2 * 1000L)
    yield(it)
}

// 热流已计算出结果
var fruits = arrayOf("watermelon", "peach", "orange").asSequence()
for (fruit in fruits) { log("we got $fruit") }
// 16:22:14 we got watermelon
// 16:22:14 we got peach
// 16:22:14 we got orange

// 冷流的每个结果都是实时计算出来的
fruits = sequence {
    collect("Apple")
    collect("Banana")
    collect("Guava")
}
for (fruit in fruits) { log("we got $fruit") }
// 16:22:14 collecting Apple...
// 16:22:16 we got Apple
// 16:22:16 collecting Banana...
// 16:22:18 we got Banana
// 16:22:18 collecting Guava...
// 16:22:20 we got Guava
for (fruit in fruits) { log("we got $fruit") }
// 16:22:14 collecting Apple...
// 16:22:16 we got Apple
// 16:22:16 collecting Banana...
// 16:22:18 we got Banana
// 16:22:18 collecting Guava...
// 16:22:20 we got Guava

// 像这些操作符是实时计算的，不会存储计算结果，不会改变热流的本质
// The operation is _intermediate_ and _stateless_.
fruits
    .filter { it.startsWith("B") }
    .map { it.last() }

// 这些操作符的计算结果将不再是热流
// The operation is _terminal_.
fruits.toList()
fruits.groupBy { it.first() }    
```

# flow

协程版热流（Asynchronous Flow）：`flow` = `sequence` + `coroutine`，上面 `sequence` 版热流可以用 `flow` 改造为协程版本

```kotlin
val log = { msg: String ->
    println("${SimpleDateFormat("HH:mm:ss").format(Date())} $msg")
}

val collect: suspend FlowCollector<String>.(String) -> Unit = {
    log("collecting ${it}...")
    delay(2 * 1000L)
    emit(it)
}

val fruits = flow {
    collect("Apple")
    collect("Banana")
    collect("Guava")
}
runBlocking { fruits.collect { log("we got $it") } }
// 16:46:25 collecting Apple...
// 16:46:27 we got Apple
// 16:46:27 collecting Banana...
// 16:46:29 we got Banana
// 16:46:29 collecting Guava...
// 16:46:31 we got Guava
```

## 线程切换

* `collect` 在当前 `CoroutineContext` 上执行
* `flowOn` 使 upstream 在指定的 `CoroutineContext` 上执行
* 下游的 `flowOn` 不会影响上游的 `flowOn` 的效果

```kotlin
println("current thread: ${Thread.currentThread().name}")
// current thread: Test worker @coroutine#1

val fruits = arrayOf("apple", "banana", "watermelon", "blackcurrant")
val produce: suspend FlowCollector<String>.(String) -> Unit = {
    delay(500L)
    println("produce $it @${Thread.currentThread().name}")
    emit(it)
}
val consume: FlowCollector<String> = FlowCollector {
    println("consume $it @${Thread.currentThread().name}")
}

val collectFruits = flow { fruits.forEach { produce(it) } }
collectFruits.collect(consume)
// produce apple        @Test worker @coroutine#1
// consume apple        @Test worker @coroutine#1
// produce banana       @Test worker @coroutine#1
// consume banana       @Test worker @coroutine#1
// produce watermelon   @Test worker @coroutine#1
// consume watermelon   @Test worker @coroutine#1
// produce blackcurrant @Test worker @coroutine#1
// consume blackcurrant @Test worker @coroutine#1

val worker1 = Executors.newSingleThreadExecutor { Thread(it, "worker_1") }.asCoroutineDispatcher()
val worker2 = Executors.newSingleThreadExecutor { Thread(it, "worker_2") }.asCoroutineDispatcher()
collectFruits
    .flowOn(worker1)
    .map {
        println("map $it @${Thread.currentThread().name}")
        it
    }
    .flowOn(worker2)
    .collect(consume)
// produce apple        @worker_1    @coroutine#3
// map apple            @worker_2    @coroutine#2
// consume apple        @Test worker @coroutine#1
// produce banana       @worker_1    @coroutine#3
// map banana           @worker_2    @coroutine#2
// consume banana       @Test worker @coroutine#1
// produce watermelon   @worker_1    @coroutine#3
// map watermelon       @worker_2    @coroutine#2
// consume watermelon   @Test worker @coroutine#1
// produce blackcurrant @worker_1    @coroutine#3
// map blackcurrant     @worker_2    @coroutine#2
// consume blackcurrant @Test worker @coroutine#1    
```

## 异常处理

* exception 会中断 `flow`
* 未被 try-catch 的 exception，无论是在 `emitter`、`operator` 还是 `collector` 都会导致 flow 抛出 exception

```kotlin
val fruits = arrayOf("apple", "banana", "watermelon", "blackcurrant")
try {
    flow {
        fruits.forEach {
            println("emit $it")
            check(it != "watermelon") { "emitter exception: $it" }
            emit(it)
        }
    }.collect { println("collect $it") }
} catch (e: Exception) {
    println("caught ${e.message}")
}
// emit apple
// collect apple
// emit banana
// collect banana
// emit watermelon
// caught emitter exception: watermelon

try {
    flow {
        fruits.forEach {
            println("emit $it")
            emit(it)
        }
    }.map {
        check(it != "watermelon") { "operator exception: $it" }
        it
    }.collect { println("collect $it") }
} catch (e: Exception) {
    println("caught ${e.message}")
}
// emit apple
// collect apple
// emit banana
// collect banana
// emit watermelon
// caught operator exception: watermelon

try {
    flow {
        fruits.forEach {
            println("emit $it")
            emit(it)
        }
    }.collect {
        check(it != "watermelon") { "collector exception: $it" }
        println("collect $it")
    }
} catch (e: Exception) {
    println("caught ${e.message}")
}
// emit apple
// collect apple
// emit banana
// collect banana
// emit watermelon
// caught collector exception: watermelon    
```

* `catch` 操作符可以捕获 `flow` 中抛出的异常
* catch 里可以忽略异常、打印日志、重新抛出异常以及 emit 新数据来替代异常
* catch 不能恢复因异常导致中断的 flow
* catch 不能捕获 downstream 里发生的异常

```kotlin
val fruits = arrayOf("apple", "banana", "watermelon", "blackcurrant")
val flow = flow {
    fruits.forEach {
        println("emit $it")
        check(it != "watermelon") { "emitter exception: $it" }
        emit(it)
    }
}
flow.catch { emit("orange") }.collect { println("collect $it") }
println()
// emit apple
// collect apple
// emit banana
// collect banana
// emit watermelon
// collect orange

flow.catch { println("ignore ${it.message}") }.collect { println("collect $it") }
println()
// emit apple
// collect apple
// emit banana
// collect banana
// emit watermelon
// ignore emitter exception: watermelon

try {
    flow.catch { throw it }.collect { println("collect $it") }
} catch (e: Exception) {
    println("caught ${e.message}")
}
// emit apple
// collect apple
// emit banana
// collect banana
// emit watermelon
// caught emitter exception: watermelon

try {
    flow.catch { println("ignore ${it.message}") }
        .map {
            check(it != "banana") { "map throw $it" }
            it
        }
        .collect { println("collect $it") }
} catch (e: Exception) {
    println("caught ${e.message}")
}
// emit apple
// collect apple
// emit banana
// caught map throw banana

try {
    flow {
        fruits.forEach {
            println("emmit $it")
            emit(it)
        }
    }.catch {
        println("ignore ${it.message}")    
    }.map {
        check(it != "watermelon") { "map throw $it" }
        it
    }.collect { println("collect $it") }
} catch (e: Exception) {
    println("caught ${e.message}")
}
// emmit apple
// collect apple
// emmit banana
// collect banana
// emmit watermelon
// caught map throw watermelon
```

# 用 Flow 替代 LiveData

Android 正在大力推广 coroutine，然而 `LiveData` 并不能很好地与 coroutine 结合使用，而 `StateFlow` 正是其继任者

flow 需要配合着 repeatOnLifecycle 一起使用才不会造成 Activity/Fragment 的泄漏

```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    // Backing property to avoid state updates from other classes
    private val _uiState = MutableStateFlow(LatestNewsUiState.Success(emptyList()))
    // The UI collects from this StateFlow to get its state updates
    val uiState: StateFlow<LatestNewsUiState> = _uiState

    init {
        viewModelScope.launch {
            newsRepository.favoriteLatestNews
                // Update View with the latest favorite news
                // Writes to the value property of MutableStateFlow,
                // adding a new element to the flow and updating all
                // of its collectors
                .collect { favoriteNews ->
                    _uiState.value = LatestNewsUiState.Success(favoriteNews)
                }
        }
    }
}

// Represents different states for the LatestNews screen
sealed class LatestNewsUiState {
    data class Success(news: List<ArticleHeadline>): LatestNewsUiState()
    data class Error(exception: Throwable): LatestNewsUiState()
}

class LatestNewsActivity : AppCompatActivity() {
    private val latestNewsViewModel = // getViewModel()

    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        // Start a coroutine in the lifecycle scope
        lifecycleScope.launch {
            // repeatOnLifecycle launches the block in a new coroutine every time the
            // lifecycle is in the STARTED state (or above) and cancels it when it's STOPPED.
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Trigger the flow and start listening for values.
                // Note that this happens when lifecycle is STARTED and stops
                // collecting when the lifecycle is STOPPED
                latestNewsViewModel.uiState.collect { uiState ->
                    // New value received
                    when (uiState) {
                        is LatestNewsUiState.Success -> showFavoriteNews(uiState.news)
                        is LatestNewsUiState.Error -> showError(uiState.exception)
                    }
                }
            }
        }
    }
}
```

# viewModelScope

* `viewModelScope` 将任务分发到主线程，而且是 `Dispatchers.Main.immediate`，也就说在分发任务时如果执行线程是主线程，则立即执行而不是插入到任务队列里
* `SupervisorJob()` 里各个子任务是相互独立的，一个子任务的取消和失败并不会影响其他子任务的执行
* `ViewModel.onCleared` 将会取消整个 viewModelScope，那么里面的子任务也都会被取消掉

```kotlin
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) {
            return scope
        }
        return setTagIfAbsent(
            JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
        )
    }

internal class CloseableCoroutineScope(context: CoroutineContext) : Closeable, CoroutineScope {
    override val coroutineContext: CoroutineContext = context

    override fun close() {
        coroutineContext.cancel()
    }
}
```

# lifecycleScope

* `lifecycleScope` 也是将任务分发到主线程
* 当状态变为 `Lifecycle.State.DESTROYED` 时整个 scope 将会被取消
* `repeatOnLifecycle` 当进入指定状态时执行一次 block，当退出这个状态时 Job 将会被取消

```kotlin
public val LifecycleOwner.lifecycleScope: LifecycleCoroutineScope
    get() = lifecycle.coroutineScope

public val Lifecycle.coroutineScope: LifecycleCoroutineScope
    get() {
        while (true) {
            val existing = mInternalScopeRef.get() as LifecycleCoroutineScopeImpl?
            if (existing != null) {
                return existing
            }
            val newScope = LifecycleCoroutineScopeImpl(
                this,
                SupervisorJob() + Dispatchers.Main.immediate
            )
            if (mInternalScopeRef.compareAndSet(null, newScope)) {
                newScope.register()
                return newScope
            }
        }
    }

internal class LifecycleCoroutineScopeImpl(
    override val lifecycle: Lifecycle,
    override val coroutineContext: CoroutineContext
) : LifecycleCoroutineScope(), LifecycleEventObserver {
    init {
        // in case we are initialized on a non-main thread, make a best effort check before
        // we return the scope. This is not sync but if developer is launching on a non-main
        // dispatcher, they cannot be 100% sure anyways.
        if (lifecycle.currentState == Lifecycle.State.DESTROYED) {
            coroutineContext.cancel()
        }
    }

    fun register() {
        launch(Dispatchers.Main.immediate) {
            if (lifecycle.currentState >= Lifecycle.State.INITIALIZED) {
                lifecycle.addObserver(this@LifecycleCoroutineScopeImpl)
            } else {
                coroutineContext.cancel()
            }
        }
    }

    override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
        if (lifecycle.currentState <= Lifecycle.State.DESTROYED) {
            lifecycle.removeObserver(this)
            coroutineContext.cancel()
        }
    }
}
```

# StateFlow

它们跟上面的 `flow` 是完全不同的概念，而是与 `LiveData` 类似，是一个可被观察的对象（`Observable`）

* 使用 `collect` 观察状态的变化，接收状态变化的通知
* collect 的返回值是 `Nothing`，表示此函数永远不会返回，后续的代码永远不可能被执行（因为它会抛出异常来结束代码的执行）
* 所以每个 collect 都需要在各自的 `launch` 里执行

```kotlin
suspend fun collect(collector: FlowCollector<T>): Nothing

launch {
    state.collect {  }
    // never reach...
}
launch { state.collect {  } }
launch { state.collect {  } }
```

LiveData 通过 LifecycleEventObserver 在 Lifecycle.State.DESTROYED 时与 Activity 断开联系，而 StateFlow 则是通过 lifecycleScope 的取消断开与 Activity 的联系，如下面代码所示：

* Activity 被匿名内部类实例 collector 持有引用
* collect 是个 infinite loop，会一直等待 state 的改变直到发生异常中断 loop
* `suspendCancellableCoroutine` 即将返回前，coroutine 进入 suspend 状态，等待 state 的改变，如果此时 coroutine 被 cancel 则 suspendCancellableCoroutine 会抛出 CancellationException，从而结束 collect 里的循环
* 一旦 collect 返回，state 将不再持有 collector，也就不再持有 Activity
* 当 state 改变时，`collectorJob?.ensureActive()` 确保 destroyed activity 不会再执行任何逻辑

```kotlin
private class StateFlowImpl {
    override suspend fun collect(collector: FlowCollector<T>): Nothing {
        val slot = allocateSlot()
        try {
            if (collector is SubscribedFlowCollector) collector.onSubscription()
            val collectorJob = currentCoroutineContext()[Job]
            var oldState: Any? = null // previously emitted T!! | NULL (null -- nothing emitted yet)
            // The loop is arranged so that it starts delivering current value without waiting first
            while (true) {
                // Here the coroutine could have waited for a while to be dispatched,
                // so we use the most recent state here to ensure the best possible conflation of stale values
                val newState = _state.value
                // always check for cancellation
                collectorJob?.ensureActive()
                // Conflate value emissions using equality
                if (oldState == null || oldState != newState) {
                    collector.emit(NULL.unbox(newState))
                    oldState = newState
                }
                // Note: if awaitPending is cancelled, then it bails out of this loop and calls freeSlot
                if (!slot.takePending()) { // try fast-path without suspending first
                    slot.awaitPending() // only suspend for new values when needed
                }
            }
        } finally {
            freeSlot(slot)
        }
    }    
}

public fun Job.ensureActive(): Unit {
    if (!isActive) throw getCancellationException()
}

private class StateFlowSlot {
    suspend fun awaitPending(): Unit = suspendCancellableCoroutine sc@ { cont ->
        assert { _state.value !is CancellableContinuationImpl<*> } // can be NONE or PENDING
        if (_state.compareAndSet(NONE, cont)) return@sc // installed continuation, waiting for pending
        // CAS failed -- the only possible reason is that it is already in pending state now
        assert { _state.value === PENDING }
        cont.resume(Unit)
    }
}

/**
 * Suspends the coroutine like [suspendCoroutine], but providing a [CancellableContinuation] to
 * the [block]. This function throws a [CancellationException] if the [Job] of the coroutine is
 * cancelled or completed while it is suspended.
 */
public suspend inline fun <T> suspendCancellableCoroutine(
    crossinline block: (CancellableContinuation<T>) -> Unit
): T
```

StateFlow.value 是线程安全的

```kotlin
public interface MutableStateFlow<T> : StateFlow<T>, MutableSharedFlow<T> {
    /**
     * The current value of this state flow.
     *
     * Setting a value that is [equal][Any.equals] to the previous one does nothing.
     *
     * This property is **thread-safe** and can be safely updated from concurrent coroutines without
     * external synchronization.
     */
    public override var value: T
}
```

# 参考

1. [Kotlin Docs - Asynchronous Flow](https://kotlinlang.org/docs/flow.html)
2. [Android Developers - Kotlin Flow](https://developer.android.com/kotlin/flow)
3. [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)