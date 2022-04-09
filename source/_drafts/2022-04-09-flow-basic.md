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

# 参考

1. [Kotlin Docs - Asynchronous Flow](https://kotlinlang.org/docs/flow.html)
2. [Android Developers - Kotlin Flow](https://developer.android.com/kotlin/flow)
3. [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)