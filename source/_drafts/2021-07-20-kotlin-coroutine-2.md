---
title: 深入分析 Kotlin Coroutines 是如何实现的（二）
date: 2021-07-20 12:00:00 +0800
tags: [kotlin, coroutine, 协程]
---

# async / await - 结构化并发编程 (Structured Concurrency)

`async` 和 `await` 是一对较为现代的 API 用以实现结构化并发编程，如下面代码所示，虽然 `runBlocking` 底层是单个线程，但是 `delay` 操作是非阻塞的，这两个操作的结合模拟了多线程环境下的阻塞 IO

`job1`、`job2` 和 `job3` 三个任务并发执行，不需要编写任何线程同步代码如 `Condition.await` 、`Condition.signal`、`CountDownLatch` 等即可获得任务结果并计算它们之和，从输出 `measureTimeMillis: 961` 可以确认三个任务是并发执行的，`result: 2453` 正确地输出了任务结果之和说明求和这一行代码是在三个任务都执行完毕并返回计算结果后执行的

```kotlin
fun main() = runBlocking {
    val values = (500L..1000L)

    val cost = measureTimeMillis {
        val job1 = async {
            val value = values.random()
            println("job1 delay: $value")
            delay(value)
            value
        }

        val job2 = async {
            val value = values.random()
            println("job2 delay: $value")
            delay(value)
            value
        }

        val job3 = async {
            val value = values.random()
            println("job3 delay: $value")
            delay(value)
            value
        }

        println("result: ${job3.await() + job2.await() + job1.await()}")
    }
    println("measureTimeMillis: $cost")
}

// output:
// job1 delay: 758
// job2 delay: 822
// job3 delay: 873
// result: 2453
// measureTimeMillis: 961
```

从 `async` 的实现可以看到它跟 [上篇文章](../../../../2021/07/15/kotlin-coroutine/) 介绍到的 `launch` 基本相同，那也就说 `async` 同样是创建协程/任务 -> 放入任务队列 -> 等待调度

跟踪 `await` 调用栈：`DeferredCoroutine.await` -> `JobSupport.awaitInternal` -> `JobSupport.awaitSuspend` -> `CancellableContinuationImpl.getResult`，在这里分为两条路：

1. 如果协程已经执行完毕得到计算结果（比如在多线程环境下），计算结果存储在 `CancellableContinuationImpl.state`，`trySuspend` 返回 false，`getResult` 返回计算所得的结果
2. 协程尚未完成，比如上面的例子，被 `delay` 中断执行而放入任务队列重新调度，`trySuspend` 返回 true，`getResult` 返回 `COROUTINE_SUSPENDED`

```kotlin
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}

private open class DeferredCoroutine<T>(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<T>(parentContext, true, active = active), Deferred<T>, SelectClause1<T> {
    override suspend fun await(): T = awaitInternal() as T
}

public open class JobSupport constructor(active: Boolean) : Job, ChildJob, ParentJob, SelectClause0 {
    internal suspend fun awaitInternal(): Any? {
        // fast-path -- check state (avoid extra object creation)
        while (true) { // lock-free loop on state
            val state = this.state
            if (state !is Incomplete) {
                // already complete -- just return result
                if (state is CompletedExceptionally) { // Slow path to recover stacktrace
                    recoverAndThrow(state.cause)
                }
                return state.unboxState()

            }
            if (startInternal(state) >= 0) break // break unless needs to retry
        }
        return awaitSuspend() // slow-path
    }

    private suspend fun awaitSuspend(): Any? = suspendCoroutineUninterceptedOrReturn { uCont ->
        /*
         * Custom code here, so that parent coroutine that is using await
         * on its child deferred (async) coroutine would throw the exception that this child had
         * thrown and not a JobCancellationException.
         */
        val cont = AwaitContinuation(uCont.intercepted(), this)
        // we are mimicking suspendCancellableCoroutine here and call initCancellability, too.
        cont.initCancellability()
        cont.disposeOnCancellation(invokeOnCompletion(ResumeAwaitOnCompletion(cont).asHandler))
        cont.getResult()
    }    
}    

internal open class CancellableContinuationImpl<in T>(
    final override val delegate: Continuation<T>,
    resumeMode: Int
) : DispatchedTask<T>(resumeMode), CancellableContinuation<T>, CoroutineStackFrame {
    internal fun getResult(): Any? {
        val isReusable = isReusable()
        // trySuspend may fail either if 'block' has resumed/cancelled a continuation
        // or we got async cancellation from parent.
        if (trySuspend()) {
            /*
             * Invariant: parentHandle is `null` *only* for reusable continuations.
             * We were neither resumed nor cancelled, time to suspend.
             * But first we have to install parent cancellation handle (if we didn't yet),
             * so CC could be properly resumed on parent cancellation.
             *
             * This read has benign data-race with write of 'NonDisposableHandle'
             * in 'detachChildIfNotReusable'.
             */
            if (parentHandle == null) {
                installParentHandle()
            }
            /*
             * Release the continuation after installing the handle (if needed).
             * If we were successful, then do nothing, it's ok to reuse the instance now.
             * Otherwise, dispose the handle by ourselves.
            */
            if (isReusable) {
                releaseClaimedReusableContinuation()
            }
            return COROUTINE_SUSPENDED
        }
        // otherwise, onCompletionInternal was already invoked & invoked tryResume, and the result is in the state
        if (isReusable) {
            // release claimed reusable continuation for the future reuse
            releaseClaimedReusableContinuation()
        }
        val state = this.state
        if (state is CompletedExceptionally) throw recoverStackTrace(state.cause, this)
        // if the parent job was already cancelled, then throw the corresponding cancellation exception
        // otherwise, there is a race if suspendCancellableCoroutine { cont -> ... } does cont.resume(...)
        // before the block returns. This getResult would return a result as opposed to cancellation
        // exception that should have happened if the continuation is dispatched for execution later.
        if (resumeMode.isCancellableMode) {
            val job = context[Job]
            if (job != null && !job.isActive) {
                val cause = job.getCancellationException()
                cancelCompletedResult(state, cause)
                throw recoverStackTrace(cause, this)
            }
        }
        return getSuccessfulResult(state)
    }

    override fun <T> getSuccessfulResult(state: Any?): T =
        when (state) {
            is CompletedContinuation -> state.result as T
            else -> state as T
        }    
}
```

`await` 的返回值会直接影响代码的执行流程，为了使反编译后的代码清晰点，我将上面的例子删除掉多余的代码如下：

```kotlin
fun main() = runBlocking {
    val values = (500L..1000L)

    val job1 = async {
        val value = values.random()
        println("job1 delay: $value")
        delay(value)
        value
    }

    val job2 = async {
        val value = values.random()
        println("job2 delay: $value")
        delay(value)
        value
    }

    val job3 = async {
        val value = values.random()
        println("job3 delay: $value")
        delay(value)
        value
    }
    println("result: ${job3.await() + job2.await() + job1.await()}")
}
```

将其反编译（decompile by [bytecode-viewer](https://github.com/Konloch/bytecode-viewer) using CFR）可以看到：

* 初始代码段 `case 0` 创建了三个异步协程然后执行其中一个 `await`
* `case 1` 和 `case 2` 分别对应其余的两个 `await`，`case 3` 对应求和操作
* 从 `label` 的赋值操作看，四个 case 必须从 0 执行到 3，不能跳级
* 如果 `await` 返回计算结果则继续执行代码，进入下一个 case
* 如果 `await` 返回 `COROUTINE_SUSPENDED` 则中断代码执行直接返回（case 不变）

```java
static final class Example_basic_01Kt.main.1
extends SuspendLambda
implements Function2<CoroutineScope, Continuation<? super Unit>, Object> {
    Object L$1;
    Object L$2;
    long J$0;
    int label;
    private /* synthetic */ Object L$0;

    Example_basic_01Kt.main.1(Continuation<? super Example_basic_01Kt.main.1> $completion) {
    }

    /*
     * Unable to fully structure code
     */
    @Nullable
    public final Object invokeSuspend(@NotNull Object var1_1) {
        var13_2 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
        switch (this.label) {
            case 0: {
                ResultKt.throwOnFailure((Object)var1_1);
                $this$runBlocking = (CoroutineScope)this.L$0;
                values = new LongRange(500L, 1000L);
                job1 = BuildersKt.async$default((CoroutineScope)$this$runBlocking, null, null, (Function2)((Function2)new /* Unavailable Anonymous Inner Class!! */), (int)3, null);
                job2 = BuildersKt.async$default((CoroutineScope)$this$runBlocking, null, null, (Function2)((Function2)new /* Unavailable Anonymous Inner Class!! */), (int)3, null);
                job3 = BuildersKt.async$default((CoroutineScope)$this$runBlocking, null, null, (Function2)((Function2)new /* Unavailable Anonymous Inner Class!! */), (int)3, null);
                var9_8 = "result: ";
                this.L$0 = job1;
                this.L$1 = job2;
                this.L$2 = var9_8;
                this.label = 1;
                v0 = job3.await((Continuation)this);
                if (v0 == var13_2) {
                    return var13_2;
                }
                ** GOTO lbl25
            }
            case 1: {
                var9_8 = (String)this.L$2;
                job2 = (Deferred)this.L$1;
                job1 = (Deferred)this.L$0;
                ResultKt.throwOnFailure((Object)$result);
                v0 = $result;
lbl25:
                // 2 sources

                var10_9 = v0;
                var10_10 = ((Number)var10_9).longValue();
                this.L$0 = job1;
                this.L$1 = var9_8;
                this.L$2 = null;
                this.J$0 = var10_10;
                this.label = 2;
                v1 = job2.await((Continuation)this);
                if (v1 == var13_2) {
                    return var13_2;
                }
                ** GOTO lbl42
            }
            case 2: {
                var10_10 = this.J$0;
                var9_8 = (String)this.L$1;
                job1 = (Deferred)this.L$0;
                ResultKt.throwOnFailure((Object)$result);
                v1 = $result;
lbl42:
                // 2 sources

                var12_11 = v1;
                this.L$0 = var9_8;
                this.L$1 = null;
                this.J$0 = var10_10 += ((Number)var12_11).longValue();
                this.label = 3;
                v2 = job1.await((Continuation)this);
                if (v2 == var13_2) {
                    return var13_2;
                }
                ** GOTO lbl56
            }
            case 3: {
                var10_10 = this.J$0;
                var9_8 = (String)this.L$0;
                ResultKt.throwOnFailure((Object)$result);
                v2 = $result;
lbl56:
                // 2 sources

                var12_11 = v2;
                var7_12 = Intrinsics.stringPlus((String)var9_8, (Object)Boxing.boxLong((long)(var10_10 + ((Number)var12_11).longValue())));
                var8_13 = false;
                System.out.println((Object)var7_12);
                return Unit.INSTANCE;
            }
        }
        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
    }

    @NotNull
    public final Continuation<Unit> create(@Nullable Object value, @NotNull Continuation<?> $completion) {
        Function2<CoroutineScope, Continuation<? super Unit>, Object> function2 = new /* invalid duplicate definition of identical inner class */;
        function2.L$0 = value;
        return (Continuation)function2;
    }

    @Nullable
    public final Object invoke(@NotNull CoroutineScope p1, @Nullable Continuation<? super Unit> p2) {
        return (this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
    }
}
```

结合上面的内容，将 `async` / `await` 这对 API 的执行流程解释下：

1. 三个 `async` 将会创建三个协程：`job1`、`job2` 和 `job3` 并放入任务队列进行调度，假设当前的 Dispatcher 是线程池那么三个协程将创建三个线程并发执行
2. 假设在求和前 `job1` 和 `job3` 执行完毕（计算结果作为 `CompletedContinuation` 存储在 `CancellableContinuationImpl.state`）
3. `case 0` 从 `job3.await` 取得计算结果并进入 `case 1`，`job2.await` 返回 `COROUTINE_SUSPENDED` 导致代码执行中断并返回，此时 `label = 2`
4. 为啥不是 `case 1` 呢？不是得从 `job2.await` 获取计算结果吗？并不需要，下次 `Example_basic_01Kt.main.1` 被调度时 `job2` 的计算结果将会作为参数从 `invokeSuspend` 传入
5. 进入 `case 2`，`job3.await` 返回计算结果
6. 进入 `case 3`，执行求和并打印出来

现在只剩下一个问题：`await` 返回 `COROUTINE_SUSPENDED` 导致流程中断后 `Example_basic_01Kt.main.1` 啥时候被再次调度？如下代码所示

* `uCont` 实际上是 `CoroutineStart.DEFAULT.invoke(block, receiver, completion)` 参数里的 `completion`，也就是 `Example_basic_01Kt.main.1` 这个 `Continuation`
* `invokeOnCompletion` 给 `async` 创建的任务添加了一个回调，当任务执行完毕时调用（返回非 `COROUTINE_SUSPENDED` 值）
* `ResumeAwaitOnCompletion` 顾名思义就是恢复任务，重新调度被 `await` 中断的 `Continuation`
* 在这里就是 `Example_basic_01Kt.main.1` 由于 `job2` 没有完成而被中断，当 `job2` 完成后重新将 `Example_basic_01Kt.main.1` 放入任务队列进行调度，并且将 `job2` 的计算结果作为参数

```kotlin
// DeferredCoroutine.await
// JobSupport.awaitInternal
private suspend fun awaitSuspend(): Any? = suspendCoroutineUninterceptedOrReturn { uCont ->
    /*
     * Custom code here, so that parent coroutine that is using await
     * on its child deferred (async) coroutine would throw the exception that this child had
     * thrown and not a JobCancellationException.
     */
    val cont = AwaitContinuation(uCont.intercepted(), this)
    // we are mimicking suspendCancellableCoroutine here and call initCancellability, too.
    cont.initCancellability()
    cont.disposeOnCancellation(invokeOnCompletion(ResumeAwaitOnCompletion(cont).asHandler))
    cont.getResult()
}

private class ResumeAwaitOnCompletion<T>(
    private val continuation: CancellableContinuationImpl<T>
) : JobNode() {
    override fun invoke(cause: Throwable?) {
        val state = job.state
        assert { state !is Incomplete }
        if (state is CompletedExceptionally) {
            // Resume with with the corresponding exception to preserve it
            continuation.resumeWithException(state.cause)
        } else {
            // Resuming with value in a cancellable way (AwaitContinuation is configured for this mode).
            @Suppress("UNCHECKED_CAST")
            continuation.resume(state.unboxState() as T)
        }
    }
}

public enum class CoroutineStart {
    public operator fun <R, T> invoke(block: suspend R.() -> T, receiver: R, completion: Continuation<T>): Unit =
        when (this) {
            DEFAULT -> block.startCoroutineCancellable(receiver, completion)
            ATOMIC -> block.startCoroutine(receiver, completion)
            UNDISPATCHED -> block.startCoroutineUndispatched(receiver, completion)
            LAZY -> Unit // will start lazily
        }
}
```

# withContext - 线程切换

