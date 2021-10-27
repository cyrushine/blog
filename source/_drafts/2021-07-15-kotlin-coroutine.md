# launch

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
        BaseContinuationImpl.create(value = null, completion = coroutine) // 如上面反编译出来的代码所示，重新创建一个 block 的实例
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

# Dispatcher

首先要了解下 `CoroutineContext` 这个概念，跟 Android 上 `Context` 的意义是一样的，就是代表了一系列 coroutine API 的 `上下文`，本质上是一个 `Map`，通过 `CoroutineContext[key] = value` 存取

其中有一个非常重要的组件：`CoroutineContext[ContinuationInterceptor]`，负责分发/调度 coroutine

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

AbstractCoroutine.start                            // 创建 StandaloneCoroutine/LazyStandaloneCoroutine
CoroutineStart.invoke(block, receiver, completion) // CoroutineStart.DEFAULT
startCoroutineCancellable(receiver, completion, onCancellation)
    createCoroutineUnintercepted       // 此方法定义在 kotlin-stdlib 包的 /kotlin/coroutines/intrinsics/IntrinsicsJvm.kt 文件里
        BaseContinuationImpl.create(value = null, completion = coroutine) // 如上面反编译出来的代码所示，重新创建一个 block 的实例
    ContinuationImpl.intercepted       // 包装为 DispatchedContinuation（dispatcher 是 BlockingEventLoop）
        BlockingEventLoop.interceptContinuation
    Continuation.resumeCancellableWith // 将 block 放入任务队列
        DispatchedContinuation.resumeCancellableWith
        BlockingEventLoop.dispatch
            BlockingEventLoop.enqueue
```
context[ContinuationInterceptor]