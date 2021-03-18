---
title: 面试官家常之Handler、MessageQueue 和 Looper
date: 2020-09-27 12:00:00 +0800
categories: [Android, Framework]
tags: [Handler, Looper]
---

`MessageQueue` 是个单向链表，按 `Message.when` 自然序排

它有类似于「生产者 - 消费者」模型的阻塞队列：没有 `Message` 时，阻塞直到新 `Message` 入队；否则阻塞到下一个 `Message.when`

```java
// 这里会阻塞
nativePollOnce(ptr, nextPollTimeoutMillis);

synchronized (this) {
    // ... ...
    if (msg != null) {
        if (now < msg.when) {
            // Next message is not ready.  Set a timeout to wake up when it is ready.
            // 阻塞一段时间，直到时间到达下一个 message.when
            nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
        } else {
            // Got a message.
            mBlocked = false;
            if (prevMsg != null) {
                prevMsg.next = msg.next;
            } else {
                mMessages = msg.next;
            }
            msg.next = null;
            if (DEBUG) Log.v(TAG, "Returning message: " + msg);
            msg.markInUse();
            return msg;
        }
    } else {
        // No more messages.
        // 没有 Message，阻塞直到 enqueueMessage 时被唤醒
        nextPollTimeoutMillis = -1;
    }
    // ... ...
}
```

```cpp
// 查看 c 层源码，可以看到 nativePollOnce 是通过 epoll 实现的

// 阻塞
MessageQueue.nativePollOnce ->
android_os_MessageQueue_nativePollOnce ->
NativeMessageQueue::pollOnce ->
Looper::pollOnce ->
Looper::pollInner ->
epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

// 唤醒
MessageQueue.enqueueMessage ->
MessageQueue.nativeWake ->
android_os_MessageQueue_nativeWake ->
NativeMessageQueue::wake ->
Looper::wake ->
write(mWakeEventFd.get(), &inc, sizeof(uint64_t));
```

与 `NativeMessageQueue`、`NativeLooper` 的关系

1. 每个 `MessageQueue` 持有一个 `NativeMessageQueue`，而 `NativeMessageQueue` 又持有当前线程的 `NativeLooper`
2. 关键的阻塞和唤醒函数都是由 `NativeLooper` 实现的，也就是 `epoll` 实现的阻塞和唤醒；只不过这个线程在 `native` 层进行 loop poll 操作
    - `MessageQueue.nativePollOnce` → `NativeMessageQueue.pollOnce` → `NativeLooper.pollOnce`
    - `MessageQueue.nativeWake` → `NativeMessageQueue.wake` → `NativeLooper.wake`

同步栅栏（`SyncBarrier`）

开启同步栅栏后，“栅栏”将会把 sync message 过滤掉，仅处理 async message，是一种提高 `Message` 优先级的方法

`postSyncBarrier` 开启同步栅栏，`removeSyncBarrier` 关闭同步栅栏，“栅栏”就是一个 target == null 的 `Message`

```java
Message msg = mMessages;    // head of linked list
if (msg != null && msg.target == null) {    // it is a barrier
    // Stalled by a barrier.  Find the next asynchronous message in the queue.
    do {
        prevMsg = msg;
        msg = msg.next;
    } while (msg != null && !msg.isAsynchronous());
}
```

主线程 `Looper` 与子线程 `Looper` 有什么不同？

主要区别在于 main looper `queue.mQuitAllowed == false`，即不允许 `looper.quit` 退出

`Looper.quit()` 和 `Looper.quitSafely()` 有什么区别？

`quit` 直接回收所有 message；而 `quitSafely` 则只回收 future message（还未到执行时间），继续执行完所有已到时间的 message 才结束

既然可以存在多个 `Handler` 往 `MessageQueue` 中添加数据（发消息时各个 `Handler` 可能处于不同线程），那它内部是如何确保线程安全的？

```java
// 加了锁
boolean enqueueMessage(Message msg, long when) {
    // ... ...
    synchronized (this) {
        // ... ...
    }
    return true;
}
```

`Looper.loop()` 为什么不会导致死循环？

queue 是个阻塞队列，（当前时间点）没有 message 时，会阻塞 thread 而不消耗 cpu；当有新 message 入队时，会唤醒 thread；参考上面阻塞和唤醒的代码片段

`Handler `造成泄露的原因

非静态内部类会持有一个外部类的隐式引用，比如下述写法：

```java
Handler handler = new Handler() {
    @Override
    public void handleMessage(@NonNull Message msg) {}
};

handler.post(new Runnable() {
    @Override
    public void run() {}
});
```

那么我们可以追溯出一条完整的 gc root path：

activity → handler/runnable → message(target/callback) → queue → looper → class looper(static field sThreadLocal) → thread

解决办法：使用 `WeakReference`

tip:

可以在 [https://cs.android.com/android](https://cs.android.com/android) 用类名、方法名等 symbol 搜索 aosp 代码，简单方便；各种 symbol 之间还有关联，可以点击跳转