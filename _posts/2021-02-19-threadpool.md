---
title: 线程池 ThreadPool 的实现
date: 2021-02-19 12:00:00 +0800
categories: [Android, Art]
tags: [threadpool]
---

## 基础知识

线程池有几个重要的参数：

- `maximumPoolSize` 最大线程数量，如果新提交的任务因为 `workQueue` 的容量限制而无法入队，则会尝试新开一个线程执行任务，而如果此时总线程数超过 `maximumPoolSize` 的限制，那么不再新开一个线程而是提交失败
- `corePoolSize` 核心线程数量，当任务执行完毕，线程也将结束它的生命周期，但最少不会低于 `corePoolSize`
- `keepAliveTime` 空闲线程的存活时间，执行完任务的线程会存活至少 `keepAliveTime`，再根据当前线程数量和 `corePoolSize` 决定要不要结束生命
- `workQueue` 任务队列，当提交的任务不能被立刻执行时（线程数 > `corePoolSize`），会放在 `workQueue` 排队等待执行

线程池的内部状态：

- `RUNNING`，可以提交新任务
- `SHUTDOWN`，不能提交新任务，但可以继续把 workQueue 里的任务执行完
- `STOP`，不能提交新任务，不执行 workQueue 里的任务，且中断正在执行的任务
- `TIDYING`，所有任务都已结束，此时线程数为零，准备执行 terminated()
- `TERMINATED`，`terminated()` 执行完毕

## 生产者 - 消费者模式的 worker

线程（worker）作为消费者，不断地从任务队列（workQueue）里获取任务并执行

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    public void run() {
        runWorker(this);
    }	
}

// worker 的执行流程
// 用一个 while 循环不断地从 workQueue 里获取任务（blocked & timeouted）
// getTask 返回 null 导致当前线程结束生命
void ThreadPoolExecutor.runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}

// 从 workQueue 获取任务（blocked & timeouted）
Runnable ThreadPoolExecutor.getTask() {
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        
        // 1. 线程池已 shutdown（只需把 workQueue 执行完毕），但 workQueue 已清空
        // 2. 线程池已 stop，无需执行 workQueue 剩下的任务
        // 此时返回 null 结束线程 
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        // 当线程数 > corePoolSize，如果阻塞 keepAliveTime 时间段都没有新任务进来，则返回 null 结束当前线程
        // 当线程数 > maximumPoolSize 且 workQueue 为空，也要结束当前线程，从而降低线程数
        int wc = workerCountOf(c);
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

## `corePoolSize`，`workQueue` 和 `maximumPoolSize` 之间的关系

```java
void ThreadPoolExecutor.execute(Runnable command) {
    // 如果线程数 < corePoolSize，则新开一个线程执行任务
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // 否则放入 workQueue 等待执行
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }

    // 如果超过 workQueue 容量限制，则尝试新开一个线程执行任务
    // 从下面的 addWorker 可以知道，如果新开线程的时候发现当前线程总数 >= maximumPoolSize，那么任务提交失败
    else if (!addWorker(command, false))
        reject(command);
}

boolean ThreadPoolExecutor.addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

## 定时任务（`ScheduledExecutorService`）

上文里的任务队列用的是 `BlockingQueue`，它是按照 FIFO 的优先级给任务排队的；要实现定时，就要按照执行时间点的优先级给任务排序，只有到达执行时间点的任务才能出队

### 堆

要排序，每次取最小值（到达执行时间点的任务），而且会插入新元素，典型的数据结构是「小顶堆」；`DelayedWorkQueue` 就是用小顶堆实现的阻塞队列

堆有几个特性：

- 堆在逻辑上是二叉树，存储为数组；节点从上到下，从左到右按顺序摊平在数组上
- 既然节点是按顺序排的，那么可以从索引计算出父/子节点：
    - `parent(i) = floor((i - 1)/2)`
    - `left(i) = 2i + 1`
    - `right(i) = 2i + 2`
- 父节点比子节点要小的是小顶堆，根节点最小；父节点比子节点大的是大顶堆，根节点最大；左右节点之间没有大小要求
- `shiftDown()`，出队最小/大值（也即根节点）后，将数组最后一个元素转移到根节点，然后从根节点开始递归地重排：如果一个节点比它的子节点小（最大堆）或者大（最小堆），那么需要将它向下移动，这样是这个节点在数组的位置下降
- `shiftUp()`，入队一个新元素到数组尾部，那么从这个元素开始从下往上重排：如果一个节点比它的父节点大（最大堆）或者小（最小堆），那么需要将它同父节点交换位置，这样是这个节点在数组的位置上升

### `DelayedWorkQueue` 按执行时间优先级排序的阻塞队列

提交一个新任务到 `DelayedWorkQueue`，重排 workQueue

workQueue 是按执行时间点排序的，leader 是在队头任务上挂起的线程，leader 未唤醒时新来的出队请求将在 available 上挂起

队列为空时线程也在 available 上挂起

```java
ScheduledFuture<?> ScheduledThreadPoolExecutor.schedule(Runnable command, long delay, TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    RunnableScheduledFuture<Void> t = decorateTask(command,
        new ScheduledFutureTask<Void>(command, null, triggerTime(delay, unit), sequencer.getAndIncrement()));
    delayedExecute(t);
    return t;
}

void ScheduledThreadPoolExecutor.delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        reject(task);
    else {
        super.getQueue().add(task);
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
}

boolean DelayedWorkQueue.add(Runnable e) {
    return offer(e);
}

// 插入小顶堆
boolean DelayedWorkQueue.offer(Runnable x) {
    if (x == null)
        throw new NullPointerException();
    RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 小顶堆底层是数组，数组容量不足需要扩容（扩容 50%）
        int i = size;
        if (i >= queue.length)
            grow();
        // 小顶堆的插入操作 siftUp
        size = i + 1;
        if (i == 0) {
            queue[0] = e;
            setIndex(e, 0);
        } else {
            siftUp(i, e);
        }
        // 新加入的任务排在队头，它的执行时间点最近
        // 此时 leader 是挂起在上一个队头上的，要置空（有更近的执行时间点进来了）并唤醒 available 上的线程来争抢新的队头任务
        if (queue[0] == e) {
            leader = null;
            available.signal();
        }
    } finally {
        lock.unlock();
    }
    return true;
}

// 堆是一个二叉树，广度优先、从左到右存储为数组
// 新元素添加到数组尾部（相当于二叉树中的叶子节点），k 是它的索引，为了继续满足小顶堆的要求，需要重新排序
void DelayedWorkQueue.siftUp(int k, RunnableScheduledFuture<?> key) {
		// 从下往上比较，如果它比父节点小，交换之
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        RunnableScheduledFuture<?> e = queue[parent];
        if (key.compareTo(e) >= 0)
            break;
        queue[k] = e;
        setIndex(e, k);
        k = parent;
    }
    // 直到满足小于父节点的要求（小顶堆），那么这就是 key 的合适位置
    queue[k] = key;
    setIndex(key, k);
}
```

上面说过 `worker` 是一个「生产者-消费者」模型，通过 `getTask` 不断地从 `workQueue` 获取任务

```java
// 从 workQueue 获取任务（blocked & timeouted）
Runnable ThreadPoolExecutor.getTask() {
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        // ...
        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}

// 任务出队（blocked）
RunnableScheduledFuture<?> DelayedWorkQueue.take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {

            // 任务队列为空，在 available 挂起
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null)
                available.await();
            else {

                // 到达执行时间点的才能出队
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0L)
                    return finishPoll(first);

                // leader 是挂起在 first 上的线程，它将在 first 执行时间点上恢复
                // 如果已有线程在 first 上挂起，则当前线程在 available 上挂起
                first = null;
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        // 出队后，唤醒在 available 上挂起的线程
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}

// 将小顶堆的根 queue[0] 出队后，需要重排堆：
// 1. 将最后一个数组元素转移到根节点
// 2. 从上往下重排：比较父节点和子节点，如果父节点大于子节点则交换之
RunnableScheduledFuture<?> DelayedWorkQueue.finishPoll(RunnableScheduledFuture<?> f) {
    int s = --size;
    RunnableScheduledFuture<?> x = queue[s];
    queue[s] = null;
    if (s != 0)
        siftDown(0, x);
    setIndex(f, -1);
    return f;
}
void DelayedWorkQueue.siftDown(int k, RunnableScheduledFuture<?> key) {
    int half = size >>> 1;
    while (k < half) {
        int child = (k << 1) + 1;
        RunnableScheduledFuture<?> c = queue[child];
        int right = child + 1;
        if (right < size && c.compareTo(queue[right]) > 0)
            c = queue[child = right];
        if (key.compareTo(c) <= 0)
            break;
        queue[k] = c;
        setIndex(c, k);
        k = child;
    }
    queue[k] = key;
    setIndex(key, k);
}

// 任务出队（timeouted）
RunnableScheduledFuture<?> DelayedWorkQueue.poll(long timeout, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {

            // 队列为空，挂起 timeout 后继续
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null) {
                if (nanos <= 0L)
                    return null;
                else
                    nanos = available.awaitNanos(nanos);
            } else {

                // first 到达执行时间点，立刻返回
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0L)
                    return finishPoll(first);
								
                // 否则挂起
                if (nanos <= 0L)
                    return null;
                first = null; // don't retain ref while waiting
                if (nanos < delay || leader != null)
                    nanos = available.awaitNanos(nanos);
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        long timeLeft = available.awaitNanos(delay);
                        nanos -= delay - timeLeft;
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```