---
title: Lock（二）AQS 源码分析以及 Lock 的实现
date: 2021-01-13 12:00:00 +0800
categories: [Android, Art]
tags: [Lock, AQS]
---

AQS 是 `Semaphore`、`ReentrantLock` 和 `ReentrantReadWriteLock` 的基础，它们紧密地结合在一块，分析 AQS 除了要明晰排队队列的操作，还要结合 `Semaphore`、`ReentrantLock` 和 `ReentrantReadWriteLock` 看看是怎么利用排队队列实现锁和信号量的

## AQS 是基于 CLH 锁修改而来的，它的排队队列也是双向链表

```java
public abstract class AbstractQueuedSynchronizer {
    private transient volatile Node head; // 队头节点
    private transient volatile Node tail; // 队尾节点
}
static final class Node {
    volatile Node prev;     // 前驱节点
    volatile Node next;     // 后驱节点
    volatile Thread thread; // 节点对应的线程
}
```

## 入队

```java
// tryAcquire 返回 true 表示获得锁，返回 false 表示未获得锁
// 由子类实现，比如公平锁、非公平锁等，这里先不讨论
public final void acquire(int arg) {
    if (!tryAcquire(arg) && // 如果不能获得锁，在队列里排队并阻塞
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 因为 futex 可能会因为中断而返回，acquireQueued 返回 true 表示发生了中断，这里主动调用中断
        selfInterrupt();
}

// 在队尾添加一个新节点
private Node addWaiter(Node mode) {
    Node node = new Node(mode);
    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            U.putObject(node, Node.PREV, oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            initializeSyncQueue();
        }
    }
}

// 刚开始 head 和 tail 都为 null，这里将 head 和 tail 都初始化为同一个空的 Node
private final void initializeSyncQueue() {
    Node h;
    if (U.compareAndSwapObject(this, HEAD, null, (h = new Node())))
        tail = h;
}

static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

## 阻塞线程（不是自旋）

```java
final boolean acquireQueued(final Node node, int arg) {
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 当线程没有阻塞且前驱是 head 时，既是轮到当前线程去尝试获得锁
            // 未获得锁，会进入下面的阻塞代码
            // 获得锁时，将 node 设为新的表头
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            // 节点入队后，如果不能获得锁，则阻塞线程；中断会打断阻塞
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}

// 一般情况下，waitStatus == 0，然后被置为 SIGNAL 并返回 false
// 然后在下一次的循环里，这个方法返回 true
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    // 阻塞当前线程直到 LockSupport.unpark 被调用
    LockSupport.park(this);
    // 中断会导致 park 返回，这里返回是不是由中断引起的返回
    return Thread.interrupted();
}

// 表头 head 其实是个空节点
// head.next 有机会去获得锁，后续的节点都是阻塞的
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

## `LockSupport.park` / `LockSupport.unpark`

它们是专为 `Lock` 设计的线程同步 API，`park` 可以阻塞线程，`unpark` 恢复线程，但它们又与 `wait`/`notify` 有所不同

- `park`，如果 `permit` 为真则把它置为假，否则阻塞（`unpark` 和中断会导致函数返回）
- `unpark`，如果线程阻塞中则恢复线程，否则将 `permit` 置为真；也就是说 `park` 之前的 `unpark` 会导致下一次的 `park` 无效，而且多次 `unpark` 不叠加效果

可以看到线程的阻塞和唤醒是通过 `futex` 系统调用实现的，`futex` 的原型是 `int futex (int *uaddr, int op, int val, const struct timespec *timeout,int *uaddr2, int val3)`

- op == `FUTEX_WAIT`，原子性的检查 `uaddr` 中计数器的值是否为 `val，`如果是则让进程休眠，直到 `FUTEX_WAKE` 或者超时，也就是把进程挂到 `uaddr` 相对应的等待队列上去
- op == `FUTEX_WAKE`，最多唤醒 `val` 个等待在 `uaddr` 上进程

而 `permit` 则是通过 `tls32_.park_state_` 实现，它是一个 `AtomicInteger`，取值范围为 `kPermitAvailable` = 0，`kNoPermit` = 1，`kNoPermitWaiterWaiting` = 2

```java
public class LockSupport {
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        U.park(false, 0L); // U 是 sun.misc.Unsafe
        setBlocker(t, null);
    }
    public static void unpark(Thread thread) {
        if (thread != null)
            U.unpark(thread);
    }
}

public final class Unsafe {
    public native void park(boolean var1, long var2);
    public native void unpark(Object var1);
}
```

```cpp
// Unsafe.park 在 art/runtime/native/sun_misc_Unsafe.cc 里注册为 Unsafe_park
static void Unsafe_park(JNIEnv* env, jobject, jboolean isAbsolute, jlong time) {
    ScopedObjectAccess soa(env);
    Thread::Current()->Park(isAbsolute, time);
}

enum {
    kPermitAvailable = 0,  // Incrementing consumes the permit
    kNoPermit = 1,         // Incrementing marks as waiter waiting
    kNoPermitWaiterWaiting = 2
};

// 初始值为 kNoPermit，自增为 kNoPermitWaiterWaiting 并阻塞；恢复后复原为 kNoPermit
// 如果执行过 unpark，那么为 kPermitAvailable，自增为 kNoPermit 但不会阻塞
void Thread::Park(bool is_absolute, int64_t time) {
    // ...
    int old_state = tls32_.park_state_.fetch_add(1, std::memory_order_relaxed);
    if (old_state == kNoPermit) {
        // ...
        int result = futex(tls32_.park_state_.Address(), FUTEX_WAIT_PRIVATE,
            /* sleep if val = */ kNoPermitWaiterWaiting,
            /* timeout */ nullptr,
            nullptr, 0);

    // Mark as no longer waiting, and consume permit if there is one.
    tls32_.park_state_.store(kNoPermit, std::memory_order_relaxed);
    // ...
}

// 置为 kPermitAvailable，如果原值为 kNoPermitWaiterWaiting 表示线程被阻塞，需要执行系统调用 futex 唤醒
void Thread::Unpark() {
    // Set permit available; will be consumed either by fetch_add (when the thread
    // tries to park) or store (when the parked thread is woken up)
    if (tls32_.park_state_.exchange(kPermitAvailable, std::memory_order_relaxed) == kNoPermitWaiterWaiting) {
        int result = futex(tls32_.park_state_.Address(), FUTEX_WAKE_PRIVATE,
                           /* number of waiters = */ 1, nullptr, nullptr, 0);
        // ...
    }
}
```

## 可重入的公平锁（`ReentrantLock.FairSync`）

公平锁按照 FIFO 的优先级顺序，从排队队列的头部开始依次传递锁的所有权

```java
static final class FairSync extends Sync {
    final void lock() {
        acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) { // 锁没有被取走，把排队队列想象成在 ATM 钱排队取钱的人们，只有当前面没有人的时候才轮到自己取钱
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current); // 标识锁在谁手上
                return true;
            }
        }
    	// 可重入，如果锁在自己手上，递增 state
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}

class AbstractQueuedSynchronizer {
	// 返回 true 表示在前面有人在排队取钱，还没轮到自己；返回 false 表示前面没人了，轮到自己取钱了
	// h == t 和 h.next == null 是刚初始化 head 和 tail 为空 node 且没有线程入队的情况
	// h.next 是第一个等待取钱的人，如果它不是当前线程，说明还没轮到自己
	public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
}
```

## 可重入的非公平锁（`ReentrantLock.NonfairSync`）

```java
static final class NonfairSync extends Sync {
    final void lock() { // 只要锁没有被取走，自己就可以获得锁
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

abstract static class Sync extends AbstractQueuedSynchronizer {
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) { // 锁没有被取走，那么自己可以直接获得锁
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 可重入，锁已经在自己手上，递增 state
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

## 释放锁

排队取锁的线程都被阻塞了，释放锁的同时需要唤醒下一个排队的线程

```java
class AbstractQueuedSynchronizer {
    // tryRelease 由子类实现，返回 true 表示当前线程持有锁并成功释放锁（可重入的情况下，未必能够释放锁）
    public final boolean release(int arg) {
        if (tryRelease(arg)) { // head 是持有锁的线程
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }   

    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            node.compareAndSetWaitStatus(ws, 0);
        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         * head.next 一般是排队等待锁里的第一个，但它可能被取消了或者其他原因从队伍里删除了，那么我们从队尾开始遍历找可以唤醒的线程
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        // 唤醒下一个线程，让他去尝试获取锁
        if (s != null)
            LockSupport.unpark(s.thread);
    }
}
```

## 可重入锁的释放过程（`ReentrantLock.Sync`）

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    protected final boolean tryRelease(int releases) {
        // releases 恒为一，而 state 为重入得次数，也即重入次数减一
        int c = getState() - releases;
        // 当前线程不持有锁，抛出异常
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        // 如果重入次数为零，那么可以释放锁
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
}
```

## 可重入的读写锁（`ReentrantReadWriteLock`）

一个资源能够被多个读线程访问（读锁有多把），或者被一个写线程访问（写锁只有一把），但是不能同时存在读写线程（读锁和写锁是互斥的）

## 获取读锁

```java
public static class ReadLock implements Lock, java.io.Serializable {
    public void lock() {
        sync.acquireShared(1);
    }
}

public abstract class AbstractQueuedSynchronizer {
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
}

// 能不能获得读锁
abstract static class Sync extends AbstractQueuedSynchronizer {
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        // 对于上面的可重入排它锁，state == 0 表示锁未被其他线程获得，
        // state == 1 表示锁已被某个线程获得，state > 1 表示重入得次数
        // 对于读写锁，state 高 16 位表示读锁的个数，state 低 16 位表示写锁重入得个数
        int c = getState();
        // 写锁被其他线程获得了，读写锁是互斥的，不能借出读锁
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return -1;
        // 获得读锁，state 读锁次数加一
        // 获得读锁的线程用 threadlocal count 记录获得的读锁的数量，这里也要加一（用来观察当前线程拿了几个读锁）
        // readerShouldBlock 的解释见下文
        int r = sharedCount(c);
        if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return 1;
        }
        // fullTryAcquireShared 其实是 tryAcquireShared 的自旋版本
        // 针对 compareAndSetState(c, c + SHARED_UNIT) 失败而自旋，也就是被别的线程抢先获得了一个读锁
        return fullTryAcquireShared(current);
    }
}

// 不能获得锁，排队并阻塞
public abstract class AbstractQueuedSynchronizer {
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED); // 添加 shared 节点至队尾
        try {
            boolean interrupted = false;
            for (;;) { // 循环取锁
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                    interrupted = true;
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
}

// tryAcquireShared 有可能是在排队等待的过程中线程被唤醒而执行
// 此时对于公平锁，只有当前面没有排队的前驱时才能去拿锁
// 对于非公平锁，见下面的注释，为了防止「写饥饿」，也就是认为写操作要比读操作更重要一点，不能完全地让所有取锁的线程去争抢
// 而是得让在对头等待的写线程优先获得锁
static final class NonfairSync extends Sync {
    final boolean readerShouldBlock() {
        /* As a heuristic to avoid indefinite writer starvation,
         * block if the thread that momentarily appears to be head
         * of queue, if one exists, is a waiting writer.  This is
         * only a probabilistic effect since a new reader will not
         * block if there is a waiting writer behind other enabled
         * readers that have not yet drained from the queue.
         */
        return apparentlyFirstQueuedIsExclusive();
    }
}
static final class FairSync extends Sync {
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}

public abstract class AbstractQueuedSynchronizer {
    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
}
```

## 获取写锁

```java
public static class ReadLock implements Lock, java.io.Serializable {
    public void lock() {
        sync.acquire(1);
    }
}

public abstract class AbstractQueuedSynchronizer {
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
}

// 能不能获得写锁；失败的话跟排它锁一样排队阻塞
abstract static class Sync extends AbstractQueuedSynchronizer {
    protected final boolean tryAcquire(int acquires) {
        Thread current = Thread.currentThread();
        int c = getState();
        int w = exclusiveCount(c);
        if (c != 0) {
            // (Note: if c != 0 and w == 0 then shared count != 0)
            // 读锁不为零，读写锁互斥，不能获得写锁
            // 写锁不为零，但是被别的线程获得，当前线程也不能获得写锁
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            // 不能超过 16 位的长度（因为写锁的数量存储在 state 的低 16 位）
            if (w + exclusiveCount(acquires) > MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            // 当前线程已持有写锁，重入导致写锁数量加一
            setState(c + acquires);
            return true;
        }
        // 读锁和写锁都为零，当然可以获得写锁；state 里的写锁数量加一，记下谁拿了写锁
        // 因为 tryAcquire 有可能是在排队过程中被唤醒而触发的，所以在非公平锁的情况下，能获得锁直接拿就好了
        // 而在公平锁的情况下，需要等前面排队的先拿锁
        if (writerShouldBlock() || !compareAndSetState(c, c + acquires))
            return false;
        setExclusiveOwnerThread(current);
        return true;
    }
}

static final class NonfairSync extends Sync {
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
}
static final class FairSync extends Sync {
    final boolean writerShouldBlock() {
        // 上面介绍过，判断自己前面还有没有前驱
        // 公平锁的情况下，只有轮到自己（没有前驱，或者说前面没有排队的）的情况下，才去获取锁
        return hasQueuedPredecessors();
    }
}
```

## 释放读锁

```java
public static class ReadLock implements Lock, java.io.Serializable {
    public void unlock() {
        sync.releaseShared(1);
    }
}
public abstract class AbstractQueuedSynchronizer {
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
}
abstract static class Sync extends AbstractQueuedSynchronizer {
    protected final boolean tryReleaseShared(int unused) {
        // threadlocal count（线程的读锁计数器）减一
        Thread current = Thread.currentThread();
        if (firstReader == current) {
            if (firstReaderHoldCount == 1)
                firstReader = null;
            else
                firstReaderHoldCount--;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                rh = readHolds.get();
            int count = rh.count;
            if (count <= 1) {
                readHolds.remove();
                if (count <= 0)
                    throw unmatchedUnlockException();
            }
            --rh.count;
        }
        // 总的读锁计数器减一
        // 读锁是共享的，释放一个读锁不影响其他的读锁；但如果读锁为零，需要唤醒阻塞在写锁上的线程
        for (;;) {
            int c = getState();
            int nextc = c - SHARED_UNIT;
            if (compareAndSetState(c, nextc))
                // Releasing the read lock has no effect on readers,
                // but it may allow waiting writers to proceed if
                // both read and write locks are now free.
                return nextc == 0;
        }
    }
}
```

## 释放写锁

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 写锁个数减一，当写锁个数为零时，返回 true 导致 AQS 移除当前节点并唤醒下一个排队的线程
    protected final boolean tryRelease(int releases) {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        int nextc = getState() - releases;
        boolean free = exclusiveCount(nextc) == 0;
        if (free)
            setExclusiveOwnerThread(null);
        setState(nextc);
        return free;
    }
}
```

## Semaphore

信号量相当于一个保存着多个锁的保险箱，它可以向外借出锁（`acquire`）和回收借出的锁（`release`），当锁用完的时候 `acquire` 会阻塞

```java
// 获得锁
void Semaphore.acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
void AbstractQueuedSynchronizer.acquireSharedInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
// 看下排队队列，跟上面的 Lock 操作是一样的
// 添加节点到队尾，循环判断是否轮到自己获得锁，否则陷入阻塞直到被唤醒
void AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}

// 非公平锁
// state 表示保险箱内锁的数量
// 如果已经没有锁可以借出，则返回负数导致线程进入排队队列，排队并阻塞
// 如果可以借出锁，则更新 state 并返回
// 因为是非公平锁，所以无需考虑前面是否有排队的线程
int NonfairSync.tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}
int Sync.nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 || compareAndSetState(available, remaining))
            return remaining;
    }
}

// 公平锁，跟非公平锁一样的，只不过当前面有排队线程时，要让它先获得锁
int FairSync.tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}

// 释放锁
// state 加一，唤醒排队线程
void Semaphore.release() {
    sync.releaseShared(1);
}
boolean AbstractQueuedSynchronizer.releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

## 参考

- [AQS与CLH相关论文学习系列（四）- AQS的设计思路](https://blog.csdn.net/lengxiao1993/article/details/108449850)