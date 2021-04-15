---
title: Lock（三）利用 Lock 实现 Condition
date: 2021-01-24 12:00:00 +0800
categories: [Android, Art]
tags: [Lock, Condition]
---

## `Condition` 简介

`Condition` 主要有两类方法：

- await，释放锁并阻塞线程直到 signal 被调用，恢复后会重新获得锁
- signal，唤醒阻塞在这个 `Condition` 上的一个或全部线程

利用条件变量前需要先获得锁

所以 `Condition` 的所有方法都需要加锁

```java
lock.lock();
// ...
condition.await();
// ...
lock.unlock();
```

`Condition` 的 await/signal 对标 `Object` 的 wait/notify，wait/notify 使用对象监视器实现的，而 await/signal 使用 `Lock` 实现的（`Lock` 对标 `synchronized` 关键字）

`ConditionObject` 自己持有一个双向链表的排队队列 condition queue，所有阻塞在此条件变量上的线程都在此排队

被唤醒的线程会被转移到 AQS 队列尾部（又叫 sync queue）

```java
public class ConditionObject implements Condition, java.io.Serializable {
    private transient Node firstWaiter; /** First node of condition queue. */
    private transient Node lastWaiter;  /** Last node of condition queue. */
}
```

## 阻塞

整个 await 大体就是以是否在 sync queue 为标识的循环，当节点转移到 sync queue 时表示线程被唤醒，跳出阻塞循环

```java
void ConditionObject.await() throws InterruptedException {
    // ... 在 condition queue 队尾添加一个 Node.CONDITION 类型的新节点（所有阻塞在此条件变量上的线程都在此排队）
    Node node = addConditionWaiter();
    // 进入阻塞状态前需要释放锁，同时唤醒下一个等待锁的线程
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 进入阻塞
    // 被唤醒的线程会从 condition queue 转移到 AQS 排队队列（又叫同步队列，sync queue）
    // isOnSyncQueue 返回 true 表示此节点已被转移到 sync queue，跳出阻塞循环
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        // await() 是可被中断的，awaitUninterruptibly() 不会被中断
        // 线程恢复后，如果发生了中断，要跳出阻塞状态
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 线程恢复后需要重新获得锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

private Node ConditionObject.addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    Node node = new Node(Node.CONDITION)
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}

int AbstractQueuedSynchronizer.fullyRelease(Node node) {
    try {
        int savedState = getState();
        if (release(savedState))
            return savedState;
        throw new IllegalMonitorStateException();
    } catch (Throwable t) {
        node.waitStatus = Node.CANCELLED;
        throw t;
    }
}
boolean AbstractQueuedSynchronizer.release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
boolean ReentrantLock.Sync.tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

## 唤醒一个阻塞的线程（从队头节点开始）

将在 condition queue 上节点转移到 sync queue 上

```java
// 把 condition queue 第一个节点（等待最久的线程）从队列里移除，添加到 sync queue 并唤醒其线程
void ConditionObject.signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

// 从 condition queue 里移除
void ConditionObject.doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}

// 插入至 sync queue 队尾并唤醒其线程
boolean ConditionObject.transferForSignal(Node node) {
    if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
        return false;
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
Node AbstractQueuedSynchronizer.enq(Node node) {
    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            U.putObject(node, Node.PREV, oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return oldTail;
            }
        } else {
            initializeSyncQueue();
        }
    }
}
```

## 唤醒全部线程

```java
// 跟唤醒第一个线程时一样的：
// 将所有在 Condition 上排队的线程逐个从 Condition 排队队列里移除，添加到 AQS 队尾，并唤醒其线程
void ConditionObject.signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}

void ConditionObject.doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```