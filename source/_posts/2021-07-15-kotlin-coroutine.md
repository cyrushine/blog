---
title: 深入分析 Kotlin Coroutines 是如何实现的
date: 2021-07-15 12:00:00 +0800
categories: [Kotlin]
tags: [kotlin, coroutine, 协程]
---

# launch - 启动协程

从 kotlin coroutines 的 [Hello World!](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm/test/guide/example-basic-01.kt) 看起

```kotlin
// https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm/test/guide/example-basic-01.kt
fun main() = runBlocking { // this: CoroutineScope
    launch { // launch a new coroutine and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello") // main coroutine continues while a previous one is delayed
}
```

需要先了解的是 `launch` 的参数 `block: suspend CoroutineScope.() -> Unit` 被编译为继承自 `SuspendLamda` 和 `Function2<CoroutineScope, Continuation>`，如下面的代码所示（decompile by [bytecode-viewer](https://github.com/Konloch/bytecode-viewer)）

`SuspendLambda` 的继承关系如下：`SuspendLambda -> ContinuationImpl -> BaseContinuationImpl -> Continuation`，所以 `block` 在库代码里一般称为 `continuation`

```java
final class Example_basic_01Kt$main$1 extends SuspendLambda implements Function2 {
   int label;
   // $FF: synthetic field
   private Object L$0;

   Example_basic_01Kt$main$1(Continuation $completion) {
      super(2, $completion);
   }

   @Nullable
   public final Object invokeSuspend(@NotNull Object var1) {
      Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch(this.label) {
      case 0:
         ResultKt.throwOnFailure(var1);
         CoroutineScope $this$runBlocking = (CoroutineScope)this.L$0;
         BuildersKt.launch$default($this$runBlocking, (CoroutineContext)null, (CoroutineStart)null, (Function2)(new 1((Continuation)null)), 3, (Object)null);
         String var3 = "Hello";
         boolean var4 = false;
         System.out.println(var3);
         return Unit.INSTANCE;
      default:
         throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }
   }

   @NotNull
   public final Continuation create(@Nullable Object value, @NotNull Continuation $completion) {
      Example_basic_01Kt$main$1 var3 = new Example_basic_01Kt$main$1($completion);
      var3.L$0 = value;
      return (Continuation)var3;
   }

   @Nullable
   public final Object invoke(@NotNull CoroutineScope p1, @Nullable Continuation p2) {
      return ((Example_basic_01Kt$main$1)this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
   }
}

final class Example_basic_01Kt$main$1$1 extends SuspendLambda implements Function2 {
   int label;

   Example_basic_01Kt$main$1$1(Continuation $completion) {
      super(2, $completion);
   }

   @Nullable
   public final Object invokeSuspend(@NotNull Object $result) {
      Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch(this.label) {
      case 0:
         ResultKt.throwOnFailure($result);
         Continuation var10001 = (Continuation)this;
         this.label = 1;
         if (DelayKt.delay(1000L, var10001) == var4) {
            return var4;
         }
         break;
      case 1:
         ResultKt.throwOnFailure($result);
         break;
      default:
         throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }

      String var2 = "World!";
      boolean var3 = false;
      System.out.println(var2);
      return Unit.INSTANCE;
   }

   @NotNull
   public final Continuation create(@Nullable Object value, @NotNull Continuation $completion) {
      return (Continuation)(new Example_basic_01Kt$main$1$1($completion));
   }

   @Nullable
   public final Object invoke(@NotNull CoroutineScope p1, @Nullable Continuation p2) {
      return ((Example_basic_01Kt$main$1$1)this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
   }
}
```

然后一路跟踪下去看看 `launch` 做了什么

```kotlin
CoroutineScope.launch                              // return Job (实际上是 StandaloneCoroutine/LazyStandaloneCoroutine)
AbstractCoroutine.start                            // 创建 StandaloneCoroutine/LazyStandaloneCoroutine
CoroutineStart.invoke(block, receiver, completion) // CoroutineStart.DEFAULT
startCoroutineCancellable(receiver, completion, onCancellation)
    createCoroutineUnintercepted       // 此方法定义在 kotlin-stdlib 包的 /kotlin/coroutines/intrinsics/IntrinsicsJvm.kt 文件里
        block.create(value = null, completion = coroutine) // 如上面反编译出来的代码所示，重新创建一个 block 的实例
    ContinuationImpl.intercepted       // 包装为 DispatchedContinuation（dispatcher 是 BlockingEventLoop）
        BlockingEventLoop.interceptContinuation
    Continuation.resumeCancellableWith // 将 block 放入任务队列
        DispatchedContinuation.resumeCancellableWith
        BlockingEventLoop.dispatch
            BlockingEventLoop.enqueue

class EventLoopImplBase {
    public fun enqueue(task: Runnable) {
        if (enqueueImpl(task)) {
            unpark()
        } else {
            DefaultExecutor.enqueue(task)
        }
    }

    // _queue 是 Atomic<Any?>，当任务队列里只有一个任务时 _queue 持有此任务的引用，当任务队列里有多个任务时，_queue 是 Queue<Runnable>
    private fun enqueueImpl(task: Runnable): Boolean {
        _queue.loop { queue ->
            if (isCompleted) return false // fail fast if already completed, may still add, but queues will close
            when (queue) {
                null -> if (_queue.compareAndSet(null, task)) return true
                is Queue<*> -> {
                    when ((queue as Queue<Runnable>).addLast(task)) {
                        Queue.ADD_SUCCESS -> return true
                        Queue.ADD_CLOSED -> return false
                        Queue.ADD_FROZEN -> _queue.compareAndSet(queue, queue.next())
                    }
                }
                else -> when {
                    queue === CLOSED_EMPTY -> return false
                    else -> {
                        // update to full-blown queue to add one more
                        val newQueue = Queue<Runnable>(Queue.INITIAL_CAPACITY, singleConsumer = true)
                        newQueue.addLast(queue as Runnable)
                        newQueue.addLast(task)
                        if (_queue.compareAndSet(queue, newQueue)) return true
                    }
                }
            }
        }
    }
}    

// Infinite loop that reads this atomic variable and performs the specified action on its value.
public inline fun <T> AtomicRef<T>.loop(action: (T) -> Unit): Nothing {
    while (true) {
        action(value)
    }
}
```

至此知道了 `block` 是被放到了任务队列里，那是谁在执行任务队列里的任务呢？这就不得不说起 `runBlocking`

`runBlocking` 通过 `BlockingCoroutine.joinBlocking` 不断地执行任务队列里的任务（`EventLoopBase.processNextEvent`）直到队列为空，因此它是 `blocking method`；第一个加入到任务队列的是 runBlocking block，所以 `Hello World!` 示例里会先输出 `Hello` 然后执行第二个 coroutine 输出 `World!`

```kotlin
public fun <T> runBlocking(context: CoroutineContext = EmptyCoroutineContext, block: suspend CoroutineScope.() -> T): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    val currentThread = Thread.currentThread()
    val contextInterceptor = context[ContinuationInterceptor]
    val eventLoop: EventLoop?
    val newContext: CoroutineContext
    if (contextInterceptor == null) {
        // create or use private event loop if no dispatcher is specified
        eventLoop = ThreadLocalEventLoop.eventLoop
        newContext = GlobalScope.newCoroutineContext(context + eventLoop)
    } else {
        // See if context's interceptor is an event loop that we shall use (to support TestContext)
        // or take an existing thread-local event loop if present to avoid blocking it (but don't create one)
        eventLoop = (contextInterceptor as? EventLoop)?.takeIf { it.shouldBeProcessedFromContext() }
            ?: ThreadLocalEventLoop.currentOrNull()
        newContext = GlobalScope.newCoroutineContext(context)
    }
    val coroutine = BlockingCoroutine<T>(newContext, currentThread, eventLoop)
    coroutine.start(CoroutineStart.DEFAULT, coroutine, block)
    return coroutine.joinBlocking()
}

private class BlockingCoroutine {
    fun joinBlocking(): T {
        registerTimeLoopThread()
        try {
            eventLoop?.incrementUseCount()
            try {
                while (true) {
                    @Suppress("DEPRECATION")
                    if (Thread.interrupted()) throw InterruptedException().also { cancelCoroutine(it) }
                    val parkNanos = eventLoop?.processNextEvent() ?: Long.MAX_VALUE
                    // note: process next even may loose unpark flag, so check if completed before parking
                    if (isCompleted) break
                    parkNanos(this, parkNanos)
                }
            } finally { // paranoia
                eventLoop?.decrementUseCount()
            }
        } finally { // paranoia
            unregisterTimeLoopThread()
        }
        // now return result
        val state = this.state.unboxState()
        (state as? CompletedExceptionally)?.let { throw it.cause }
        return state as T
    }
}
```

## 总结

`launch` 把 block 放入任务队列等待执行，类似于 `ExecutorService.submit` 和 `Handler.post`，同时说明 `协程` 本质上就是对任务的调度，底层是线程/线程池 + 任务队列

# Dispatcher - 任务调度

首先要了解下 `CoroutineContext` 这个概念，跟 Android 上 `Context` 的意义是一样的，就是代表了一系列 coroutine API 的 `上下文`，本质上是一个 `Map`，通过 `CoroutineContext[key] = value` 存取

其中有一个非常重要的组件：`CoroutineContext[ContinuationInterceptor]`，负责分发/调度 coroutine，有 `BlockingEventLoop`, `Dispatchers.Default`, `Dispatchers.IO` 等等

```kotlin
public interface CoroutineContext {
    /**
     * Returns the element with the given [key] from this context or `null`.
     */
    public operator fun <E : Element> get(key: Key<E>): E?
}

public interface ContinuationInterceptor : CoroutineContext.Element {
    /**
     * The key that defines *the* context interceptor.
     */
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
}
```

通过 `CoroutineScope.launch` 启动 coroutine 时，新创建的 coroutine 会继承 scpoe context（`launch` 的参数 `context` 会覆盖相同 key 的 scope context element），而且当 context 不含 interceptor 时主动添加 `Dispatchers.Default` 作为 `Dispatcher`，也就是说 `Dispatcher`/`CoroutineContext[ContinuationInterceptor]` 作为任务调度器是一个 **必不可少** 的 coroutine 组件

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}

public actual fun CoroutineScope.newCoroutineContext(context: CoroutineContext): CoroutineContext {
    val combined = coroutineContext + context
    val debug = if (DEBUG) combined + CoroutineId(COROUTINE_ID.incrementAndGet()) else combined
    return if (combined !== Dispatchers.Default && combined[ContinuationInterceptor] == null)
        debug + Dispatchers.Default else debug
}
```

现在回顾下 `launch`

* `createCoroutineUnintercepted` - 创建 block 实例 `Continuation`，并使其获得 context
* `ContinuationImpl.intercepted` - 从 context 里取出 Dispatcher，由它负责把 `Continuation` 包装成 `DispatchedContinuation`（Dispatcher + Continuation），这样 `任务` 和 `调度器` 就齐活了
* `Continuation.resumeCancellableWith` - 让 Dispatcher 负责调度 Continuation

上一章节分析了 coroutine 本质上就是任务队列里的任务，并简单提了下 `BlockingEventLoop` 这个调度器，这一章节分析各种各样的调度器

```kotlin
CoroutineScope.launch
AbstractCoroutine.start
CoroutineStart.invoke(block, receiver, completion)
startCoroutineCancellable(receiver, completion, onCancellation)
    createCoroutineUnintercepted
        block.create(value = null, completion = coroutine)
    ContinuationImpl.intercepted
        ContinuationInterceptor.interceptContinuation
    Continuation.resumeCancellableWith
        DispatchedContinuation.resumeCancellableWith
        CoroutineDispatcher.dispatch

// kotlin-stdlib - kotlin/coroutines/intrinsics/IntrinsicsJvm.kt
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl) // block -> SuspendLambda -> ContinuationImpl -> BaseContinuationImpl
        create(receiver, probeCompletion)
    else {
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
        }
    }
}            

internal abstract class ContinuationImpl {
    public fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }
}

class CoroutineDispatcher {
   public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        DispatchedContinuation(this, continuation)   
}
```

## BlockingEventLoop / runBlocking

`runBlocking` 会往 context 添加 `BlockingEventLoop`

* 上一章节介绍过，它是基于 `Queue` 的任务队列；只有一个任务时 `_queue` 直接引用这个任务节省空间，多个任务时 `_queue` 才引用 `Queue`
* 它是 `ThreadLocal` 的，跟 `Looper` 一样每个线程单独绑定一个 `BlockingEventLoop`
* `dispatch` 只是简单地将任务入队，然后唤醒对应的线程

```kotlin
public fun <T> runBlocking(context: CoroutineContext = EmptyCoroutineContext, block: suspend CoroutineScope.() -> T): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    val currentThread = Thread.currentThread()
    val contextInterceptor = context[ContinuationInterceptor]
    val eventLoop: EventLoop?
    val newContext: CoroutineContext
    if (contextInterceptor == null) {
        // create or use private event loop if no dispatcher is specified
        eventLoop = ThreadLocalEventLoop.eventLoop
        newContext = GlobalScope.newCoroutineContext(context + eventLoop)
    } else {
        // See if context's interceptor is an event loop that we shall use (to support TestContext)
        // or take an existing thread-local event loop if present to avoid blocking it (but don't create one)
        eventLoop = (contextInterceptor as? EventLoop)?.takeIf { it.shouldBeProcessedFromContext() }
            ?: ThreadLocalEventLoop.currentOrNull()
        newContext = GlobalScope.newCoroutineContext(context)
    }
    val coroutine = BlockingCoroutine<T>(newContext, currentThread, eventLoop)
    coroutine.start(CoroutineStart.DEFAULT, coroutine, block)
    return coroutine.joinBlocking()
}

internal object ThreadLocalEventLoop {
    private val ref = CommonThreadLocal<EventLoop?>()

    internal val eventLoop: EventLoop
        get() = ref.get() ?: createEventLoop().also { ref.set(it) }
}

// https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm/src/EventLoop.kt
internal class BlockingEventLoop(
    override val thread: Thread
) : EventLoopImplBase()

internal actual fun createEventLoop(): EventLoop = BlockingEventLoop(Thread.currentThread())

class EventLoopImplBase {
    private val _queue = atomic<Any?>(null)

    public final override fun dispatch(context: CoroutineContext, block: Runnable) = enqueue(block)

    public fun enqueue(task: Runnable) {
        if (enqueueImpl(task)) {
            // todo: we should unpark only when this delayed task became first in the queue
            unpark()
        } else {
            DefaultExecutor.enqueue(task)
        }
    }   

    private fun enqueueImpl(task: Runnable): Boolean {
        _queue.loop { queue ->
            if (isCompleted) return false // fail fast if already completed, may still add, but queues will close
            when (queue) {
                null -> if (_queue.compareAndSet(null, task)) return true
                is Queue<*> -> {
                    when ((queue as Queue<Runnable>).addLast(task)) {
                        Queue.ADD_SUCCESS -> return true
                        Queue.ADD_CLOSED -> return false
                        Queue.ADD_FROZEN -> _queue.compareAndSet(queue, queue.next())
                    }
                }
                else -> when {
                    queue === CLOSED_EMPTY -> return false
                    else -> {
                        // update to full-blown queue to add one more
                        val newQueue = Queue<Runnable>(Queue.INITIAL_CAPACITY, singleConsumer = true)
                        newQueue.addLast(queue as Runnable)
                        newQueue.addLast(task)
                        if (_queue.compareAndSet(queue, newQueue)) return true
                    }
                }
            }
        }
    }    
}
```

## Dispatchers.Default

上一章节说过 `Coroutine.launch` 开启协程时，如果 context 不包含 dispatcher，会自动添加默认的 dispatcher: `Dispatchers.Default`，如下示例代码

```kotlin
fun main() = runBlocking {
    val scope = CoroutineScope(EmptyCoroutineContext)
    val job = scope.launch {
        println("Thread name: ${Thread.currentThread().name}")
    }
    job.join()
}
```

从下面的代码可以看到 `Dispatchers.Default` 最终是通过 `CoroutineScheduler` 进行任务调度的，`CoroutineScheduler` 的代码量比较大这里就不深入分析了，但从它的构造函数参数 `corePoolSize`、`maxPoolSize` 和 `idleWorkerKeepAliveNs` 这些熟悉的概念可以看出它是个 [线程池](../../../../2021/02/19/threadpool/)

```kotlin
public actual object Dispatchers {
    /**
     * The default [CoroutineDispatcher] that is used by all standard builders like
     * [launch][CoroutineScope.launch], [async][CoroutineScope.async], etc
     * if no dispatcher nor any other [ContinuationInterceptor] is specified in their context.
     *
     * It is backed by a shared pool of threads on JVM. By default, the maximal level of parallelism used
     * by this dispatcher is equal to the number of CPU cores, but is at least two.
     * Level of parallelism X guarantees that no more than X tasks can be executed in this dispatcher in parallel.
     */
    @JvmStatic
    public actual val Default: CoroutineDispatcher = createDefaultDispatcher()
}

internal actual fun createDefaultDispatcher(): CoroutineDispatcher =
    if (useCoroutinesScheduler) DefaultScheduler else CommonPool

internal object DefaultScheduler : ExperimentalCoroutineDispatcher()

public open class ExperimentalCoroutineDispatcher(
    private val corePoolSize: Int,
    private val maxPoolSize: Int,
    private val idleWorkerKeepAliveNs: Long,
    private val schedulerName: String = "CoroutineScheduler"
) : ExecutorCoroutineDispatcher() {

    public constructor(
        corePoolSize: Int = CORE_POOL_SIZE,
        maxPoolSize: Int = MAX_POOL_SIZE,
        schedulerName: String = DEFAULT_SCHEDULER_NAME
    ) : this(corePoolSize, maxPoolSize, IDLE_WORKER_KEEP_ALIVE_NS, schedulerName)

    @Deprecated(message = "Binary compatibility for Ktor 1.0-beta", level = DeprecationLevel.HIDDEN)
    public constructor(
        corePoolSize: Int = CORE_POOL_SIZE,
        maxPoolSize: Int = MAX_POOL_SIZE
    ) : this(corePoolSize, maxPoolSize, IDLE_WORKER_KEEP_ALIVE_NS)

    override val executor: Executor
        get() = coroutineScheduler

    // This is variable for test purposes, so that we can reinitialize from clean state
    private var coroutineScheduler = createScheduler()

    override fun dispatch(context: CoroutineContext, block: Runnable): Unit =
        try {
            coroutineScheduler.dispatch(block)
        } catch (e: RejectedExecutionException) {
            // CoroutineScheduler only rejects execution when it is being closed and this behavior is reserved
            // for testing purposes, so we don't have to worry about cancelling the affected Job here.
            DefaultExecutor.dispatch(context, block)
        }

    private fun createScheduler() = CoroutineScheduler(corePoolSize, maxPoolSize, idleWorkerKeepAliveNs, schedulerName)        
}

internal class CoroutineScheduler(
    @JvmField val corePoolSize: Int,
    @JvmField val maxPoolSize: Int,
    @JvmField val idleWorkerKeepAliveNs: Long = IDLE_WORKER_KEEP_ALIVE_NS,
    @JvmField val schedulerName: String = DEFAULT_SCHEDULER_NAME
) : Executor, Closeable
```

## Dispatchers.IO

`Dispatchers.IO` 如注释所说的，是一个用以承载阻塞型 IO 操作的线程池，线程池是由上面说的 `CoroutineScheduler` 实现的，而它本身之所以叫做 `LimitingDispatcher` 是因为它限制了并发任务数为 64：

* `inFlightTasks` 记录了并发任务数，如果小于阈值 `parallelism`（默认 64）则提交到线程池
* 否则将任务放到任务队列 `queue: ConcurrentLinkedQueue` 里，等提交到线程池的任务执行完自身逻辑，空出一个并发任务的位置后，再从 `queue` 里取一个等待中的任务提交到线程池

```kotlin
public actual object Dispatchers {
    /**
     * The [CoroutineDispatcher] that is designed for offloading blocking IO tasks to a shared pool of threads.
     *
     * Additional threads in this pool are created and are shutdown on demand.
     * The number of threads used by tasks in this dispatcher is limited by the value of
     * `kotlinx.coroutines.io.parallelism`" ([IO_PARALLELISM_PROPERTY_NAME]) system property.
     * It defaults to the limit of 64 threads or the number of cores (whichever is larger).
     *
     * Moreover, the maximum configurable number of threads is capped by the
     * `kotlinx.coroutines.scheduler.max.pool.size` system property.
     * If you need a higher number of parallel threads,
     * you should use a custom dispatcher backed by your own thread pool.
     *
     * ### Implementation note
     *
     * This dispatcher shares threads with the [Default][Dispatchers.Default] dispatcher, so using
     * `withContext(Dispatchers.IO) { ... }` when already running on the [Default][Dispatchers.Default]
     * dispatcher does not lead to an actual switching to another thread &mdash; typically execution
     * continues in the same thread.
     * As a result of thread sharing, more than 64 (default parallelism) threads can be created (but not used)
     * during operations over IO dispatcher.
     */   
    public val IO: CoroutineDispatcher = DefaultScheduler.IO
}

internal object DefaultScheduler : ExperimentalCoroutineDispatcher() {
    val IO: CoroutineDispatcher = LimitingDispatcher(
        this,
        systemProp(IO_PARALLELISM_PROPERTY_NAME, 64.coerceAtLeast(AVAILABLE_PROCESSORS)), // 64
        "Dispatchers.IO",
        TASK_PROBABLY_BLOCKING // 1
    )
}

private class LimitingDispatcher(
    private val dispatcher: ExperimentalCoroutineDispatcher,
    private val parallelism: Int,
    private val name: String?,
    override val taskMode: Int
) : ExecutorCoroutineDispatcher(), TaskContext, Executor {

    private val queue = ConcurrentLinkedQueue<Runnable>()
    private val inFlightTasks = atomic(0)

    override val executor: Executor
        get() = this

    override fun execute(command: Runnable) = dispatch(command, false)

    override fun dispatch(context: CoroutineContext, block: Runnable) = dispatch(block, false)

    private fun dispatch(block: Runnable, tailDispatch: Boolean) {
        var taskToSchedule = block
        while (true) {
            // Commit in-flight tasks slot
            val inFlight = inFlightTasks.incrementAndGet()

            // Fast path, if parallelism limit is not reached, dispatch task and return
            if (inFlight <= parallelism) {
                dispatcher.dispatchWithContext(taskToSchedule, this, tailDispatch)
                return
            }

            // Parallelism limit is reached, add task to the queue
            queue.add(taskToSchedule)

            /*
             * We're not actually scheduled anything, so rollback committed in-flight task slot:
             * If the amount of in-flight tasks is still above the limit, do nothing
             * If the amount of in-flight tasks is lesser than parallelism, then
             * it's a race with a thread which finished the task from the current context, we should resubmit the first task from the queue
             * to avoid starvation.
             *
             * Race example #1 (TN is N-th thread, R is current in-flight tasks number), execution is sequential:
             *
             * T1: submit task, start execution, R == 1
             * T2: commit slot for next task, R == 2
             * T1: finish T1, R == 1
             * T2: submit next task to local queue, decrement R, R == 0
             * Without retries, task from T2 will be stuck in the local queue
             */
            if (inFlightTasks.decrementAndGet() >= parallelism) {
                return
            }

            taskToSchedule = queue.poll() ?: return
        }
    }

    override fun dispatchYield(context: CoroutineContext, block: Runnable) {
        dispatch(block, tailDispatch = true)
    }

    override fun toString(): String {
        return name ?: "${super.toString()}[dispatcher = $dispatcher]"
    }

    /**
     * Tries to dispatch tasks which were blocked due to reaching parallelism limit if there is any.
     *
     * Implementation note: blocking tasks are scheduled in a fair manner (to local queue tail) to avoid
     * non-blocking continuations starvation.
     * E.g. for
     * ```
     * foo()
     * blocking()
     * bar()
     * ```
     * it's more profitable to execute bar at the end of `blocking` rather than pending blocking task
     */
    override fun afterTask() {
        var next = queue.poll()
        // If we have pending tasks in current blocking context, dispatch first
        if (next != null) {
            dispatcher.dispatchWithContext(next, this, true)
            return
        }
        inFlightTasks.decrementAndGet()

        /*
         * Re-poll again and try to submit task if it's required otherwise tasks may be stuck in the local queue.
         * Race example #2 (TN is N-th thread, R is current in-flight tasks number), execution is sequential:
         * T1: submit task, start execution, R == 1
         * T2: commit slot for next task, R == 2
         * T1: finish T1, poll queue (it's still empty), R == 2
         * T2: submit next task to the local queue, decrement R, R == 1
         * T1: decrement R, finish. R == 0
         *
         * The task from T2 is stuck is the local queue
         */
        next = queue.poll() ?: return
        dispatch(next, true)
    }
}
```

## Dispatchers.Main

`Dispatchers.Main` 在 Android 代表主线程，也就说这个调度器会把任务放到主线程执行：`Handler.post`

值得注意的是这里创建的是 `Async Handler`，`Async Handler` 创建的 `Message` 都是异步的：`Message.isAsynchronous() == true`，也就是说通过 `Dispatchers.Main` 调度的任务不会受到 [同步栅栏](../../../../2020/09/27/handler-messagequeue-looper/) 的影响

```kotlin
public actual object Dispatchers {
    /**
     * A coroutine dispatcher that is confined to the Main thread operating with UI objects.
     * This dispatcher can be used either directly or via [MainScope] factory.
     * Usually such dispatcher is single-threaded.
     *
     * Access to this property may throw [IllegalStateException] if no main thread dispatchers are present in the classpath.
     *
     * Depending on platform and classpath it can be mapped to different dispatchers:
     * - On JS and Native it is equivalent of [Default] dispatcher.
     * - On JVM it is either Android main thread dispatcher, JavaFx or Swing EDT dispatcher. It is chosen by
     *   [`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html).
     *
     * In order to work with `Main` dispatcher, the following artifacts should be added to project runtime dependencies:
     *  - `kotlinx-coroutines-android` for Android Main thread dispatcher
     *  - `kotlinx-coroutines-javafx` for JavaFx Application thread dispatcher
     *  - `kotlinx-coroutines-swing` for Swing EDT dispatcher
     *
     * In order to set a custom `Main` dispatcher for testing purposes, add the `kotlinx-coroutines-test` artifact to 
     * project test dependencies.
     *
     * Implementation note: [MainCoroutineDispatcher.immediate] is not supported on Native and JS platforms.
     */
    @JvmStatic
    public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher
}

// Lazy loader for the main dispatcher
internal object MainDispatcherLoader {

    private val FAST_SERVICE_LOADER_ENABLED = systemProp(FAST_SERVICE_LOADER_PROPERTY_NAME, true)

    @JvmField
    val dispatcher: MainCoroutineDispatcher = loadMainDispatcher()

    private fun loadMainDispatcher(): MainCoroutineDispatcher {
        return try {
            val factories = if (FAST_SERVICE_LOADER_ENABLED) {
                FastServiceLoader.loadMainDispatcherFactory()
            } else {
                // We are explicitly using the
                // `ServiceLoader.load(MyClass::class.java, MyClass::class.java.classLoader).iterator()`
                // form of the ServiceLoader call to enable R8 optimization when compiled on Android.
                ServiceLoader.load(
                        MainDispatcherFactory::class.java,
                        MainDispatcherFactory::class.java.classLoader
                ).iterator().asSequence().toList()
            }
            @Suppress("ConstantConditionIf")
            factories.maxByOrNull { it.loadPriority }?.tryCreateDispatcher(factories)
                ?: createMissingDispatcher()
        } catch (e: Throwable) {
            // Service loader can throw an exception as well
            createMissingDispatcher(e)
        }
    }
}

internal class AndroidDispatcherFactory : MainDispatcherFactory {
    override fun createDispatcher(allFactories: List<MainDispatcherFactory>) =
        HandlerContext(Looper.getMainLooper().asHandler(async = true))
}

internal fun Looper.asHandler(async: Boolean): Handler {
    // Async support was added in API 16.
    if (!async || Build.VERSION.SDK_INT < 16) {
        return Handler(this)
    }

    if (Build.VERSION.SDK_INT >= 28) {
        // TODO compile against API 28 so this can be invoked without reflection.
        val factoryMethod = Handler::class.java.getDeclaredMethod("createAsync", Looper::class.java)
        return factoryMethod.invoke(null, this) as Handler
    }

    val constructor: Constructor<Handler>
    try {
        constructor = Handler::class.java.getDeclaredConstructor(Looper::class.java,
            Handler.Callback::class.java, Boolean::class.javaPrimitiveType)
    } catch (ignored: NoSuchMethodException) {
        // Hidden constructor absent. Fall back to non-async constructor.
        return Handler(this)
    }
    return constructor.newInstance(this, null, true)
}

internal class HandlerContext private constructor(
    private val handler: Handler,
    private val name: String?,
    private val invokeImmediately: Boolean
) : HandlerDispatcher(), Delay {

    override fun dispatch(context: CoroutineContext, block: Runnable) {
        if (!handler.post(block)) {
            cancelOnRejection(context, block)
        }
    }
}
```

# delay - 非阻塞

> Delays coroutine for a given time without blocking a thread and resumes it after a specified time.

正如 `delay` 的函数文档所说，它是 `non-blocking` 的，那什么是非阻塞呢？如下代码所示，上面说过 `launch coroutine` 其实就是任务调度，下面的两个 `launch` 一共往任务队列放进两个任务，而 `runBlocking` 底层是单个线程，故两个任务是串行的：先执行 `delay(1000)` 再执行 `delay(500)`；然而 `500` 却先于 `1000` 输出，说明 `delay(1000)` 确实没有阻塞线程

```kotlin
fun main(): Unit = runBlocking {
    launch {
        delay(1000)
        println("1000")
    }
    launch {
        delay(500)
        println("500")
    }
}

// output:
// 500
// 1000
```

跟踪 `delay` 看看它是怎么实现的，但到 `suspendCoroutineUninterceptedOrReturn` 就走不下去了因为这个函数并没有什么实质的 body，那就直接看编译后的字节码吧

```kotlin
public suspend fun delay(timeMillis: Long) {
    if (timeMillis <= 0) return // don't delay
    return suspendCancellableCoroutine sc@ { cont: CancellableContinuation<Unit> ->
        // if timeMillis == Long.MAX_VALUE then just wait forever like awaitCancellation, don't schedule.
        if (timeMillis < Long.MAX_VALUE) {
            cont.context.delay.scheduleResumeAfterDelay(timeMillis, cont)
        }
    }
}

public suspend inline fun <T> suspendCancellableCoroutine(
    crossinline block: (CancellableContinuation<T>) -> Unit
): T =
    suspendCoroutineUninterceptedOrReturn { uCont ->
        val cancellable = CancellableContinuationImpl(uCont.intercepted(), resumeMode = MODE_CANCELLABLE)
        /*
         * For non-atomic cancellation we setup parent-child relationship immediately
         * in case when `block` blocks the current thread (e.g. Rx2 with trampoline scheduler), but
         * properly supports cancellation.
         */
        cancellable.initCancellability()
        block(cancellable)
        cancellable.getResult()
    }

public suspend inline fun <T> suspendCoroutineUninterceptedOrReturn(crossinline block: (Continuation<T>) -> Any?): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    throw NotImplementedError("Implementation of suspendCoroutineUninterceptedOrReturn is intrinsic")
}    
```

用 `bytecode-viewer` 打开如下，关键代码是这一行：`getDelay(cont.getContext()).scheduleResumeAfterDelay(timeMillis, cont)`

* 从 context 里取 `Delay`，key 是 `ContinuationInterceptor.Key`，咦这不就是上一章节讨论的 `Dispatcher` 的 key 嘛！也就是说 `delay` 有可能是由 Dispatcher 实现的
* 如果 Dispatcher 没有实现 `Delay` 接口则由 `DefaultDelay` 负责

那么就一个个 Dispatcher 地看吧

```java
final class Example_basic_01Kt$main$1$1 extends SuspendLambda implements Function2 {
   @Nullable
   public final Object invokeSuspend(@NotNull Object $result) {
      Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch(this.label) {
      case 0:
         ResultKt.throwOnFailure($result);
         Continuation var10001 = (Continuation)this;
         this.label = 1;
         if (DelayKt.delay(1000L, var10001) == var4) {
            return var4;
         }
         break;
      case 1:
         ResultKt.throwOnFailure($result);
         break;
      default:
         throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }

      String var2 = "1000";
      boolean var3 = false;
      System.out.println(var2);
      return Unit.INSTANCE;
   }
}

public final class DelayKt {
   public static final Object delay(long timeMillis, @NotNull Continuation $completion) {
      if (timeMillis <= 0L) {
         return Unit.INSTANCE;
      } else {
         int $i$f$suspendCancellableCoroutine = false;
         int var5 = false;
         CancellableContinuationImpl cancellable$iv = new CancellableContinuationImpl(IntrinsicsKt.intercepted($completion), 1);
         cancellable$iv.initCancellability();
         CancellableContinuation cont = (CancellableContinuation)cancellable$iv;
         int var8 = false;
         if (timeMillis < Long.MAX_VALUE) {
            getDelay(cont.getContext()).scheduleResumeAfterDelay(timeMillis, cont);
         }

         Object var10000 = cancellable$iv.getResult();
         if (var10000 == IntrinsicsKt.getCOROUTINE_SUSPENDED()) {
            DebugProbesKt.probeCoroutineSuspended($completion);
         }

         return var10000 == IntrinsicsKt.getCOROUTINE_SUSPENDED() ? var10000 : Unit.INSTANCE;
      }
   }

   @NotNull
   public static final Delay getDelay(@NotNull CoroutineContext $this$delay) {
      Element var2 = $this$delay.get((Key)ContinuationInterceptor.Key);
      Delay var1 = var2 instanceof Delay ? (Delay)var2 : null;
      return var1 == null ? DefaultExecutorKt.getDefaultDelay() : var1;
   }
   
}
```

## BlockingEventLoop

block 被层层包装为 `DelayedResumeTask` 并被放入 `_delayed: atomic<DelayedTaskQueue?>`，等待在 `delay` 毫秒后重新调度，类似于 `Handler.postDelayed(r, delayMillis)`，可重新调度任务后 block 不就重新被执行了一遍吗？

```kotlin
internal class BlockingEventLoop(
    override val thread: Thread
) : EventLoopImplBase()

internal abstract class EventLoopImplBase: EventLoopImplPlatform(), Delay {
    public override fun scheduleResumeAfterDelay(timeMillis: Long, continuation: CancellableContinuation<Unit>) {
        val timeNanos = delayToNanos(timeMillis)
        if (timeNanos < MAX_DELAY_NS) {
            val now = nanoTime()
            DelayedResumeTask(now + timeNanos, continuation).also { task ->
                continuation.disposeOnCancellation(task)
                schedule(now, task)
            }
        }
    }     

    public fun schedule(now: Long, delayedTask: DelayedTask) {
        when (scheduleImpl(now, delayedTask)) {
            SCHEDULE_OK -> if (shouldUnpark(delayedTask)) unpark()
            SCHEDULE_COMPLETED -> reschedule(now, delayedTask)
            SCHEDULE_DISPOSED -> {} // do nothing -- task was already disposed
            else -> error("unexpected result")
        }
    }

    private fun scheduleImpl(now: Long, delayedTask: DelayedTask): Int {
        if (isCompleted) return SCHEDULE_COMPLETED
        val delayedQueue = _delayed.value ?: run {
            _delayed.compareAndSet(null, DelayedTaskQueue(now))
            _delayed.value!!
        }
        return delayedTask.scheduleTask(now, delayedQueue, this)
    }        
}

class DelayedTask {
    @Synchronized
    fun scheduleTask(now: Long, delayed: DelayedTaskQueue, eventLoop: EventLoopImplBase): Int {
        if (_heap === DISPOSED_TASK) return SCHEDULE_DISPOSED // don't add -- was already disposed
        delayed.addLastIf(this) { firstTask ->
            if (eventLoop.isCompleted) return SCHEDULE_COMPLETED // non-local return from scheduleTask
            /**
             * We are about to add new task and we have to make sure that [DelayedTaskQueue]
             * invariant is maintained. The code in this lambda is additionally executed under
             * the lock of [DelayedTaskQueue] and working with [DelayedTaskQueue.timeNow] here is thread-safe.
             */
            if (firstTask == null) {
                /**
                 * When adding the first delayed task we simply update queue's [DelayedTaskQueue.timeNow] to
                 * the current now time even if that means "going backwards in time". This makes the structure
                 * self-correcting in spite of wild jumps in `nanoTime()` measurements once all delayed tasks
                 * are removed from the delayed queue for execution.
                 */
                delayed.timeNow = now
            } else {
                /**
                 * Carefully update [DelayedTaskQueue.timeNow] so that it does not sweep past first's tasks time
                 * and only goes forward in time. We cannot let it go backwards in time or invariant can be
                 * violated for tasks that were already scheduled.
                 */
                val firstTime = firstTask.nanoTime
                // compute min(now, firstTime) using a wrap-safe check
                val minTime = if (firstTime - now >= 0) now else firstTime
                // update timeNow only when going forward in time
                if (minTime - delayed.timeNow > 0) delayed.timeNow = minTime
            }
            /**
             * Here [DelayedTaskQueue.timeNow] was already modified and we have to double-check that newly added
             * task does not violate [DelayedTaskQueue] invariant because of that. Note also that this scheduleTask
             * function can be called to reschedule from one queue to another and this might be another reason
             * where new task's time might now violate invariant.
             * We correct invariant violation (if any) by simply changing this task's time to now.
             */
            if (nanoTime - delayed.timeNow < 0) nanoTime = delayed.timeNow
            true
        }
        return SCHEDULE_OK
    }    
}   
```

上面的例子一个 block 里只有一个 `delay`，规律不明显，添加多个 `delay` 就能看出端倪了：

* `delay` 前的代码段被逐个编号：1, 2, 3...
* 引入成员属性 `label`，执行 block 时根据 `label` 的指示通过 `switch` 跳转执行各个代码段
* 重新调度时执行完的代码段就不会被再次执行了

```java
// kotlin
fun main(): Unit = runBlocking {
    println(0)
    delay(100)
    println(100)
    delay(200)
    println(200)
    delay(300)
    println(300)
    delay(400)
    println(400)
}

// decompile by CFR
static final class Example_basic_01Kt.main.1
extends SuspendLambda
implements Function2<CoroutineScope, Continuation<? super Unit>, Object> {
    
    int label;

    /*
     * Unable to fully structure code
     */
    @Nullable
    public final Object invokeSuspend(@NotNull Object var1_1) {
        var4_2 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
        switch (this.label) {
            case 0: {
                ResultKt.throwOnFailure((Object)var1_1);
                var2_3 = 0;
                var3_4 = false;
                System.out.println(var2_3);
                this.label = 1;
                v0 = DelayKt.delay((long)100L, (Continuation)((Continuation)this));
                if (v0 == var4_2) {
                    return var4_2;
                }
                ** GOTO lbl16
            }
            case 1: {
                ResultKt.throwOnFailure((Object)$result);
                v0 = $result;
lbl16:
                // 2 sources

                var2_3 = 100;
                var3_4 = false;
                System.out.println(var2_3);
                this.label = 2;
                v1 = DelayKt.delay((long)200L, (Continuation)((Continuation)this));
                if (v1 == var4_2) {
                    return var4_2;
                }
                ** GOTO lbl27
            }
            case 2: {
                ResultKt.throwOnFailure((Object)$result);
                v1 = $result;
lbl27:
                // 2 sources

                var2_3 = 200;
                var3_4 = false;
                System.out.println(var2_3);
                this.label = 3;
                v2 = DelayKt.delay((long)300L, (Continuation)((Continuation)this));
                if (v2 == var4_2) {
                    return var4_2;
                }
                ** GOTO lbl38
            }
            case 3: {
                ResultKt.throwOnFailure((Object)$result);
                v2 = $result;
lbl38:
                // 2 sources

                var2_3 = 300;
                var3_4 = false;
                System.out.println(var2_3);
                this.label = 4;
                v3 = DelayKt.delay((long)400L, (Continuation)((Continuation)this));
                if (v3 == var4_2) {
                    return var4_2;
                }
                ** GOTO lbl49
            }
            case 4: {
                ResultKt.throwOnFailure((Object)$result);
                v3 = $result;
lbl49:
                // 2 sources

                var2_3 = 400;
                var3_4 = false;
                System.out.println(var2_3);
                return Unit.INSTANCE;
            }
        }
        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
    }
}
```

### 总结

coroutine 本质是任务队列，而 `delay` 本质是重新调度（重新入队至 `EventLoopImplBase._delay`），通过引入成员属性 `label`，以及将各个代码段用 `switch` 分段，即可实现在重新调度时跳转到指定 `代码行` 而不重复执行前面的代码

## HandlerContext

`Dispatchers.Main` 在 Android 上是底层为单线程/主线程的 `HandlerContext(Looper.getMainLooper().asHandler(async = true))`，它实现了 `Delay`，通过 `Handler.postDelayed` 实现重新调度

```kotlin
internal class HandlerContext private constructor(
    private val handler: Handler,
    private val name: String?,
    private val invokeImmediately: Boolean
) : HandlerDispatcher(), Delay {

    override fun scheduleResumeAfterDelay(timeMillis: Long, continuation: CancellableContinuation<Unit>) {
        val block = Runnable {
            with(continuation) { resumeUndispatched(Unit) }
        }
        if (handler.postDelayed(block, timeMillis.coerceAtMost(MAX_DELAY))) {
            continuation.invokeOnCancellation { handler.removeCallbacks(block) }
        } else {
            cancelOnRejection(continuation.context, block)
        }
    }    
}
```

## DefaultDelay

`Dispatchers.Default` 和 `Dispatcher.IO` 并没有实现 `Delay` 接口，如上面反编译后的 `getDelay` 所示，是由 `DefaultDelay` 来实现的

`EventLoopImplBase` 在上一章节介绍过，它实现了 `Delay` 接口，`EventLoopImplBase.scheduleResumeAfterDelay` 会将任务放入 `EventLoopImplBase._delay` 任务队列，也就是说这两个调度器上的 delay task 实际上是被放入别人的（`DefaultDelay`） delay queue 里

```kotlin
internal actual val DefaultDelay: Delay = DefaultExecutor

internal actual object DefaultExecutor : EventLoopImplBase(), Runnable
```

那这些 delay task 不就被改变了 Dispatcher，跑到 `DefaultDelay/DefaultExecutor` 里执行了吗？我们从 `Delay.scheduleResumeAfterDelay` 看下去：

* task 被加入到 `_delay` 任务队列，然后唤醒对应的线程 `DefaultExecutor.run`
* `DefaultExecutor` 将 `_delay` 里第一个到执行时间点的 task 转移到任务队列 `_queue` 并执行

而任务队列里的 `Runnable` 实际上是 `DispatchedContinuation`

```kotlin
internal abstract class EventLoopImplBase: EventLoopImplPlatform(), Delay {
    public override fun scheduleResumeAfterDelay(timeMillis: Long, continuation: CancellableContinuation<Unit>) {
        val timeNanos = delayToNanos(timeMillis)
        if (timeNanos < MAX_DELAY_NS) {
            val now = nanoTime()
            DelayedResumeTask(now + timeNanos, continuation).also { task ->
                continuation.disposeOnCancellation(task)
                schedule(now, task)
            }
        }
    }

    public fun schedule(now: Long, delayedTask: DelayedTask) {
        when (scheduleImpl(now, delayedTask)) {
            SCHEDULE_OK -> if (shouldUnpark(delayedTask)) unpark()
            SCHEDULE_COMPLETED -> reschedule(now, delayedTask)
            SCHEDULE_DISPOSED -> {} // do nothing -- task was already disposed
            else -> error("unexpected result")
        }
    }

    private fun shouldUnpark(task: DelayedTask): Boolean = _delayed.value?.peek() === task    
}

internal actual abstract class EventLoopImplPlatform: EventLoop() {
    protected abstract val thread: Thread

    protected actual fun unpark() {
        val thread = thread // atomic read
        if (Thread.currentThread() !== thread)
            unpark(thread)
    }
}

internal actual object DefaultExecutor: EventLoopImplBase(), Delay {
    override fun run() {
        ThreadLocalEventLoop.setEventLoop(this)
        registerTimeLoopThread()
        try {
            var shutdownNanos = Long.MAX_VALUE
            if (!notifyStartup()) return
            while (true) {
                Thread.interrupted() // just reset interruption flag
                var parkNanos = processNextEvent()
                if (parkNanos == Long.MAX_VALUE) {
                    // nothing to do, initialize shutdown timeout
                    val now = nanoTime()
                    if (shutdownNanos == Long.MAX_VALUE) shutdownNanos = now + KEEP_ALIVE_NANOS
                    val tillShutdown = shutdownNanos - now
                    if (tillShutdown <= 0) return // shut thread down
                    parkNanos = parkNanos.coerceAtMost(tillShutdown)
                } else
                    shutdownNanos = Long.MAX_VALUE
                if (parkNanos > 0) {
                    // check if shutdown was requested and bail out in this case
                    if (isShutdownRequested) return
                    parkNanos(this, parkNanos)
                }
            }
        } finally {
            _thread = null // this thread is dead
            acknowledgeShutdownIfNeeded()
            unregisterTimeLoopThread()
            // recheck if queues are empty after _thread reference was set to null (!!!)
            if (!isEmpty) thread // recreate thread if it is needed
        }
    }    
}

internal abstract class EventLoopImplBase: EventLoopImplPlatform(), Delay {
    override fun processNextEvent(): Long {
        // unconfined events take priority
        if (processUnconfinedEvent()) return 0
        // queue all delayed tasks that are due to be executed
        val delayed = _delayed.value
        if (delayed != null && !delayed.isEmpty) {
            val now = nanoTime()
            while (true) {
                // make sure that moving from delayed to queue removes from delayed only after it is added to queue
                // to make sure that 'isEmpty' and `nextTime` that check both of them
                // do not transiently report that both delayed and queue are empty during move
                delayed.removeFirstIf {
                    if (it.timeToExecute(now)) {
                        enqueueImpl(it)
                    } else
                        false
                } ?: break // quit loop when nothing more to remove or enqueueImpl returns false on "isComplete"
            }
        }
        // then process one event from queue
        val task = dequeue()
        if (task != null) {
            task.run()
            return 0
        }
        return nextTime
    }     
}
```

上面曾经说过 block 会经过重重包装，其中一层就是 `DispatchedContinuation`，它也是一个 `Runnable`（被 `DefaultExecutor` 执行）：`SchedulerTask.run` ->  `DispatchedContinuation.resumeWith` -> `CoroutineDispatcher.dispatch`

也就说 `DispatchedContinuation` = `CoroutineDispatcher` + `Continuation`，它被执行时是将任务交由对应的 Dispatcher 执行而不是由当前的线程执行（正如它的名字描述的那样），同时也实现了一个很重要的概念：`线程切换`

```kotlin
internal class DispatchedContinuation<in T>(
    @JvmField val dispatcher: CoroutineDispatcher,
    @JvmField val continuation: Continuation<T>
) : DispatchedTask<T>(MODE_UNINITIALIZED), CoroutineStackFrame, Continuation<T> by continuation

internal abstract class DispatchedTask<in T>(
    @JvmField public var resumeMode: Int
) : SchedulerTask() {
    public final override fun run() {
        assert { resumeMode != MODE_UNINITIALIZED } // should have been set before dispatching
        val taskContext = this.taskContext
        var fatalException: Throwable? = null
        try {
            val delegate = delegate as DispatchedContinuation<T>
            val continuation = delegate.continuation
            withContinuationContext(continuation, delegate.countOrElement) {
                val context = continuation.context
                val state = takeState() // NOTE: Must take state in any case, even if cancelled
                val exception = getExceptionalResult(state)
                /*
                 * Check whether continuation was originally resumed with an exception.
                 * If so, it dominates cancellation, otherwise the original exception
                 * will be silently lost.
                 */
                val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null
                if (job != null && !job.isActive) {
                    val cause = job.getCancellationException()
                    cancelCompletedResult(state, cause)
                    continuation.resumeWithStackTrace(cause)
                } else {
                    if (exception != null) {
                        continuation.resumeWithException(exception)
                    } else {
                        continuation.resume(getSuccessfulResult(state))
                    }
                }
            }
        } catch (e: Throwable) {
            // This instead of runCatching to have nicer stacktrace and debug experience
            fatalException = e
        } finally {
            val result = runCatching { taskContext.afterTask() }
            handleFatalException(fatalException, result.exceptionOrNull())
        }
    }    
}

public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))

class DispatchedContinuation {
    override fun resumeWith(result: Result<T>) {
        val context = continuation.context
        val state = result.toState()
        if (dispatcher.isDispatchNeeded(context)) {
            _state = state
            resumeMode = MODE_ATOMIC
            dispatcher.dispatch(context, this)
        } else {
            executeUnconfined(state, MODE_ATOMIC) {
                withCoroutineContext(this.context, countOrElement) {
                    continuation.resumeWith(result)
                }
            }
        }
    }    
}    
```

# 其他

coroutine API 很多都需要上下文 context，但作为参数传来穿去总是麻烦，于是有了 `CoroutineScope` 用以承载上下文：

* `runBlocking` - 用当前线程作为调度器，会阻塞代码的执行直到所有协程都执行完毕
* `CoroutineScope(EmptyCoroutineContext|Dispatchers.Main|Dispatchers.IO|...)` - 构造 scope，使用指定的上下文/调度器，如果上下文里不包含调度器则使用默认的 `Dispatchers.Default`

```kotlin
public interface CoroutineScope {
    /**
     * The context of this scope.
     * Context is encapsulated by the scope and used for implementation of coroutine builders that are extensions on the scope.
     * Accessing this property in general code is not recommended for any purposes except accessing the [Job] instance for advanced usages.
     *
     * By convention, should contain an instance of a [job][Job] to enforce structured concurrency.
     */
    public val coroutineContext: CoroutineContext
}
```