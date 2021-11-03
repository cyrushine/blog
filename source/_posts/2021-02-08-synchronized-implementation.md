---
title: Lock（四）synchronized 的语言实现
date: 2021-02-08 12:00:00 +0800
tags: [synchronized]
---

## monitor 指令

在 [Lock（二）AQS 源码分析以及 Lock 的实现](../../../../2021/01/13/aqs-lock-implementation/) 这篇文章里介绍了基于 AQS 的 `Lock`，它是双向链表的排队队列和系统调用 `futex` 实现的

其实 java 语言规范里自带了 Lock 的实现：`synchronized` 关键字，下面看看 ART 是怎么实现它的

先写一个使用了 `synchronized` 的测试方法

```java
package com.example.myapplication;

public class Hello {

    private final Object lock = new Object();

    public void say(String msg) {
        synchronized(lock) {
            System.out.println(msg != null ? msg : "null");
        }
    }

    public static void main() {
        Hello instance = new Hello();
        instance.say("Hello World");
    }
}
```

编译打包出 apk 文件，解压出其中的 classes.dex，并用 `baksmali` 转换成 smali 指令

```bash
java -jar baksmali-2.4.0.jar disassemble classes.dex
```

`Hello.say(String)` 对应的 smali 代码是这样的

`synchronized` 代码块被两条指令包裹：`monitor-enter` 和 `monitor-exit`

```bash
.method public say(Ljava/lang/String;)V
    .registers 5
    .param p1, "msg"    # Ljava/lang/String;

    .line 8
		# 本地变量寄存器 v0 被赋予 Hello.lock
    iget-object v0, p0, Lcom/example/myapplication/Hello;->lock:Ljava/lang/Object;

		# 重点
    monitor-enter v0

    .line 9
    :try_start_3
    sget-object v1, Ljava/lang/System;->out:Ljava/io/PrintStream;

    if-eqz p1, :cond_9

    move-object v2, p1

    goto :goto_b

    :cond_9
    const-string v2, "null"

    :goto_b
    invoke-virtual {v1, v2}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V

    .line 10
		# 重点
    monitor-exit v0

    .line 11
    return-void

    .line 10
    :catchall_10
    move-exception v1

    monitor-exit v0
    :try_end_12
    .catchall {:try_start_3 .. :try_end_12} :catchall_10

    throw v1
.end method
```

## 对象锁的概念

在进一步分析代码之前，先要了解下 java 对象锁的一些背景知识（from [JAVA锁的膨胀过程](https://blog.csdn.net/fan1865221/article/details/96338419)）

java 对象锁会有一个膨胀加码的过程：无锁 → 偏向锁 → 轻量级锁 → 重量级锁

- **无锁**
- **偏向锁，**为了在无多线程竞争的情况下尽量减少不必须要的轻量级锁执行路径。当一个线程访问同步代码块并获取锁时，会在 Mark Word 里存储锁偏向的线程 ID。在线程进入和退出同步块时不再通过 CAS 操作来加锁和解锁，而是检测 Mark Word 里是否存储着指向当前线程的偏向锁。轻量级锁的获取及释放依赖多次 CAS 原子指令，而偏向锁只需要在置换 ThreadID 的时候依赖一次 CAS 原子指令即可。
- **轻量级锁，**在多线程竞争不激烈的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。偏向锁是认为环境中不存在竞争情况，而轻量级锁则是认为环境中不存在竞争或者竞争不激烈，所以轻量级锁一般都只会有少数几个线程竞争锁对象，其他线程只需要稍微等待（自旋）下就可以获取锁，但是自旋次数有限制，如果自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁膨胀为重量级锁，重量级锁使除了拥有锁的线程以外的线程都阻塞，防止CPU空转。
- **重量级锁，**当有一个线程获取锁之后，其余所有等待获取该锁的线程都会处于阻塞状态。重量级锁通过操作系统的 Mutex Lock 实现，操作系统实现线程之间的切换需要从用户态切换到内核态，切换成本非常高。简言之，就是所有的控制权都交给了操作系统，由操作系统来负责线程间的调度和线程的状态变更。而这样会出现频繁地对线程运行状态的切换，线程的挂起和唤醒，从而消耗大量的系统资源，导致性能低下。

## LockWord

先了解一个结构`LockWord` ，它其实是一个 `uint32_t`，低 16 bits 保存持有锁的 thread id，后续 12 bits 保存锁的个数

它是 `Object` 的成员变量 `monitor_`，所以每个 java 对象都可以作为锁使用

```cpp
static LockWord FromThinLockId(uint32_t thread_id, uint32_t count, uint32_t gc_state) {
    CHECK_LE(thread_id, static_cast<uint32_t>(kThinLockMaxOwner));
    CHECK_LE(count, static_cast<uint32_t>(kThinLockMaxCount));
    // DCHECK_EQ(gc_bits & kGCStateMaskToggled, 0U);
    return LockWord((thread_id << kThinLockOwnerShift) |
                    (count << kThinLockCountShift) |
                    (gc_state << kGCStateShift) |
                    (kStateThinOrUnlocked << kStateShift));
}

// C++ mirror of java.lang.Object
class MANAGED LOCKABLE Object {
	// Monitor and hash code information.
	uint32_t monitor_;
}
```

## Mutex

ART 使用 `Mutex` 作为互斥量的实现（lock & unlock），它根据宏 `ART_USE_FUTEXES` 决定是使用 `futex` 还是 `mutex`

```cpp
// art/runtime/base/mutex.cc

// 获取排它锁
void Mutex::ExclusiveLock(Thread* self) {
  if (!recursive_ || !IsExclusiveHeld(self)) {
#if ART_USE_FUTEXES
    bool done = false;
    do {
      // Mutex::state_and_contenders_ 是 AtomicInteger
      // 最低 1 bit 表示互斥量是否被持有（1 - 持有，0 - 未持有），其余高位的 bits 表示在此互斥量上挂起的线程数量
      int32_t cur_state = state_and_contenders_.load(std::memory_order_relaxed);
      // 锁没有被取走，立刻获得锁（cas）
      if (LIKELY((cur_state & kHeldMask) == 0) /* lock not held */) {
        done = state_and_contenders_.CompareAndSetWeakAcquire(cur_state, cur_state | kHeldMask);
      } else {

        // 否则将挂起线程的数量加一，并用 futex 挂起当前线程
        ScopedContentionRecorder scr(this, SafeGetTid(self), GetExclusiveOwnerTid());
        if (!WaitBrieflyFor(&state_and_contenders_, self, [](int32_t v) { return (v & kHeldMask) == 0; })) {
          // Increment contender count. We can't create enough threads for this to overflow.
          increment_contenders();
          // Make cur_state again reflect the expected value of state_and_contenders.
          cur_state += kContenderIncrement;
          if (UNLIKELY(should_respond_to_empty_checkpoint_request_)) {
            self->CheckEmptyCheckpointFromMutex();
          }
          do {
            if (futex(state_and_contenders_.Address(), FUTEX_WAIT_PRIVATE, cur_state, nullptr, nullptr, 0) != 0) {
              if ((errno != EAGAIN) && (errno != EINTR)) {
                PLOG(FATAL) << "futex wait failed for " << name_;
              }
            }
            SleepIfRuntimeDeleted(self);
            // Retry until not held. In heavy contention situations we otherwise get redundant
            // futex wakeups as a result of repeatedly decrementing and incrementing contenders.
            cur_state = state_and_contenders_.load(std::memory_order_relaxed);
          } while ((cur_state & kHeldMask) != 0);
          decrement_contenders();
        }
      }
    } while (!done);
    // Confirm that lock is now held.
    DCHECK_NE(state_and_contenders_.load(std::memory_order_relaxed) & kHeldMask, 0);
#else

    // 使用 pthread_mutex_lock 加锁
    CHECK_MUTEX_CALL(pthread_mutex_lock, (&mutex_));
#endif

    // exclusive_owner_ 记下获得排他锁的 thread id
    exclusive_owner_.store(SafeGetTid(self), std::memory_order_relaxed);
    RegisterAsLocked(self);
  }
  recursion_count_++;
}

// tryLock 方法
bool Mutex::ExclusiveTryLock(Thread* self) {
  if (!recursive_ || !IsExclusiveHeld(self)) {
#if ART_USE_FUTEXES
    bool done = false;
    do {

      // 使用 futex 的情况下，利用一个 AtomicInteger 的最低 1 bit 表示锁有没被借出，一个 cas 操作即可
      int32_t cur_state = state_and_contenders_.load(std::memory_order_relaxed);
      if ((cur_state & kHeldMask) == 0) {
        // Change state to held and impose load/store ordering appropriate for lock acquisition.
        done = state_and_contenders_.CompareAndSetWeakAcquire(cur_state, cur_state | kHeldMask);
      } else {
        return false;
      }
    } while (!done);
    DCHECK_NE(state_and_contenders_.load(std::memory_order_relaxed) & kHeldMask, 0);
#else

    // mutex
    int result = pthread_mutex_trylock(&mutex_);
    // ...
#endif

    // exclusive_owner_ 记下获得锁的 thread id
    DCHECK_EQ(GetExclusiveOwnerTid(), 0);
    exclusive_owner_.store(SafeGetTid(self), std::memory_order_relaxed);
    RegisterAsLocked(self);
  }
  recursion_count_++;
  return true;
}

// 也是 tryLock 方法，特别的是它会自旋一小会
bool Mutex::ExclusiveTryLockWithSpinning(Thread* self) {
  // Spin a small number of times, since this affects our ability to respond to suspension
  // requests. We spin repeatedly only if the mutex repeatedly becomes available and unavailable
  // in rapid succession, and then we will typically not spin for the maximal period.
  const int kMaxSpins = 5;
  for (int i = 0; i < kMaxSpins; ++i) {
    if (ExclusiveTryLock(self)) {
      return true;
    }
#if ART_USE_FUTEXES
    if (!WaitBrieflyFor(&state_and_contenders_, self,
            [](int32_t v) { return (v & kHeldMask) == 0; })) {
      return false;
    }
#endif
  }
  return ExclusiveTryLock(self);
}

// 释放排它锁
void Mutex::ExclusiveUnlock(Thread* self) {
  recursion_count_--;
  if (!recursive_ || recursion_count_ == 0) {
    RegisterAsUnlocked(self);
#if ART_USE_FUTEXES
    bool done = false;
    do {

      // 使用 cas 将 state_and_contenders_ 最低 1 bit 置零（表示锁没被借出）
      int32_t cur_state = state_and_contenders_.load(std::memory_order_relaxed);
      if (LIKELY((cur_state & kHeldMask) != 0)) {
        // We're no longer the owner.
        exclusive_owner_.store(0 /* pid */, std::memory_order_relaxed);
        // Change state to not held and impose load/store ordering appropriate for lock release.
        uint32_t new_state = cur_state & ~kHeldMask;  // Same number of contenders.
        done = state_and_contenders_.CompareAndSetWeakRelease(cur_state, new_state);

        // state_and_contenders_ 不为零表示仍有线程在锁上挂起，用 futex 让系统唤醒其中一个
        if (LIKELY(done)) {  // Spurious fail or waiters changed ?
          if (UNLIKELY(new_state != 0) /* have contenders */) {
            futex(state_and_contenders_.Address(), FUTEX_WAKE_PRIVATE, kWakeOne,
                  nullptr, nullptr, 0);
          }
          // We only do a futex wait after incrementing contenders and verifying the lock was
          // still held. If we didn't see waiters, then there couldn't have been any futexes
          // waiting on this lock when we did the CAS. New arrivals after that cannot wait for us,
          // since the futex wait call would see the lock available and immediately return.
        }
      } else {
        // 异常情况...
      }
    } while (!done);
#else

    // mutex
    exclusive_owner_.store(0 /* pid */, std::memory_order_relaxed);
    CHECK_MUTEX_CALL(pthread_mutex_unlock, (&mutex_));
#endif
  }
}

// Unlock 等同于 ExclusiveUnlock
void Unlock(Thread* self) RELEASE() {  ExclusiveUnlock(self); }
```

## `monitor-enter` 指令

### 无锁、偏向锁和轻量级锁

进入临界区，尝试获得锁

```cpp
// 在 cs.android.com 找到的相关性很高的方法，可能不是指令 monitor-enter 直接调用的方法，但最终应该会走到这里来
// platform/superproject/art/runtime/mirror/object-inl.h
inline ObjPtr<mirror::Object> Object::MonitorEnter(Thread* self) {
  return Monitor::MonitorEnter(self, this, /*trylock=*/false);
}

ObjPtr<mirror::Object> Monitor::MonitorEnter(Thread* self, ObjPtr<mirror::Object> obj, bool trylock) {
  DCHECK(self != nullptr);
  DCHECK(obj != nullptr);
  self->AssertThreadSuspensionIsAllowable();
  obj = FakeLock(obj);
  uint32_t thread_id = self->GetThreadId();
  size_t contention_count = 0;
  constexpr size_t kExtraSpinIters = 100;
  StackHandleScope<1> hs(self);
  Handle<mirror::Object> h_obj(hs.NewHandle(obj));
#if !ART_USE_FUTEXES
  // In this case we cannot inflate an unowned monitor, so we sometimes defer inflation.
  bool should_inflate = false;
#endif
  while (true) {
    LockWord lock_word = h_obj->GetLockWord(false);
    switch (lock_word.GetState()) {

      // 无锁的情况下，升级为偏向锁（thin lock）
      // 偏向锁记录下 thread id 和锁的个数 0，通过 cas 记录在 Object.monitor_
      case LockWord::kUnlocked: {
        LockWord thin_locked(LockWord::FromThinLockId(thread_id, 0, lock_word.GCState()));
        if (h_obj->CasLockWord(lock_word, thin_locked, CASMode::kWeak, std::memory_order_acquire)) {
#if !ART_USE_FUTEXES
          if (should_inflate) {
            InflateThinLocked(self, h_obj, lock_word, 0);
          }
#endif
          AtraceMonitorLock(self, h_obj.Get(), /* is_wait= */ false);
          return h_obj.Get();  // Success!
        }
        continue;  // Go again.
      }

      // 偏向锁，而且当前线程跟偏向锁里记录的线程是同一个线程
      // 那么只需把偏向锁里的锁个数加一即可，依然使用 cas 保存在 Object.monitor_
      case LockWord::kThinLocked: {
        uint32_t owner_thread_id = lock_word.ThinLockOwner();
        if (owner_thread_id == thread_id) {
          // No ordering required for initial lockword read.
          // We own the lock, increase the recursion count.
          uint32_t new_count = lock_word.ThinLockCount() + 1;
          if (LIKELY(new_count <= LockWord::kThinLockMaxCount)) {
            LockWord thin_locked(LockWord::FromThinLockId(thread_id,
                                                          new_count,
                                                          lock_word.GCState()));
            // 重新设置偏向锁，一个选择用 cas 原子操作符，另一个选择没用，不明这样区分的意义
            // Only this thread pays attention to the count. Thus there is no need for stronger
            // than relaxed memory ordering.
            if (!kUseReadBarrier) {
              h_obj->SetLockWord(thin_locked, /* as_volatile= */ false);
              AtraceMonitorLock(self, h_obj.Get(), /* is_wait= */ false);
              return h_obj.Get();  // Success!
            } else {
              // Use CAS to preserve the read barrier state.
              if (h_obj->CasLockWord(lock_word,
                                     thin_locked,
                                     CASMode::kWeak,
                                     std::memory_order_relaxed)) {
                AtraceMonitorLock(self, h_obj.Get(), /* is_wait= */ false);
                return h_obj.Get();  // Success!
              }
            }
            continue;  // Go again.
          } else {

            // 当前线程持有偏向锁，但锁的个数超过阈值 kThinLockMaxCount
            // 那么将偏向锁（thin lock）升级为重量级锁（fat lock）
            // We'd overflow the recursion count, so inflate the monitor.
            InflateThinLocked(self, h_obj, lock_word, 0);
          }
        } else {

          // 持有偏向锁的线程不是当前线程，此时的 thin lock 对应上文的轻量级锁
          // 也就是说轻量级锁是这么一种情况：一个线程持有偏向锁，遇到了另一个线程的争抢
          // 争抢的线程在这里自旋（spin），contention_count 表示自旋的次数
          // 1. 如果自旋次数 <= kExtraSpinIters，那么继续在外一层的 while 循环里自旋
          // 2. 如果自旋次数 < kExtraSpinIters，争抢线程让渡 CPU 给优先级更高的线程，并将自己排到 CPU 调度队列的队尾（sched_yield），相当于优化的自旋
          // 3. 最后将轻量级锁膨胀为重量级锁
          if (trylock) {
            return nullptr;
          }
          contention_count++;
          Runtime* runtime = Runtime::Current();
          if (contention_count
              <= kExtraSpinIters + runtime->GetMaxSpinsBeforeThinLockInflation()) {
            if (contention_count > kExtraSpinIters) {
              sched_yield();
            }
          } else {
#if ART_USE_FUTEXES
            contention_count = 0;
            // No ordering required for initial lockword read. Install rereads it anyway.
            InflateThinLocked(self, h_obj, lock_word, 0);
#else
            // Can't inflate from non-owning thread. Keep waiting. Bad for power, but this code
            // isn't used on-device.
            should_inflate = true;
            usleep(10);
#endif
          }
        }
        continue;  // Start from the beginning.
      }

      // 重量级锁的情况下会挂起当前线程，在下一节分析
      case LockWord::kFatLocked: {
        std::atomic_thread_fence(std::memory_order_acquire);
        Monitor* mon = lock_word.FatLockMonitor();
        if (trylock) {
          return mon->TryLock(self) ? h_obj.Get() : nullptr;
        } else {
          mon->Lock(self);
          return h_obj.Get();  // Success!
        }
      }

      // 不清楚这个条件
      case LockWord::kHashCode:
        Inflate(self, nullptr, h_obj.Get(), lock_word.GetHashCode());
        continue;  // Start from the beginning.
      default: {
        LOG(FATAL) << "Invalid monitor state " << lock_word.GetState();
        UNREACHABLE();
      }
    }
  }
}
```

### 锁膨胀的过程（inflate）

```cpp
// thin lock 膨胀至重量级锁（fat lock）的过程
void Monitor::InflateThinLocked(Thread* self, Handle<mirror::Object> obj, LockWord lock_word, uint32_t hash_code) {
  // 当前线程持有此偏向锁的情况（由于锁个数超过阈值导致膨胀）
  // 升级到重量级锁（fat lock）
  DCHECK_EQ(lock_word.GetState(), LockWord::kThinLocked);
  uint32_t owner_thread_id = lock_word.ThinLockOwner();
  if (owner_thread_id == self->GetThreadId()) {
    // We own the monitor, we can easily inflate it.
    Inflate(self, self, obj.Get(), hash_code);
  } else {

    // 当前线程不持有此偏向锁，出现争抢（此时对应轻量级锁）
    // 挂起持有偏向锁的线程，将轻量级锁膨胀为重量级锁（fat lock），然后恢复线程
    ThreadList* thread_list = Runtime::Current()->GetThreadList();
    // Suspend the owner, inflate. First change to blocked and give up mutator_lock_.
    self->SetMonitorEnterObject(obj.Get());
    bool timed_out;
    Thread* owner;
    {
      ScopedThreadSuspension sts(self, kWaitingForLockInflation);
      owner = thread_list->SuspendThreadByThreadId(owner_thread_id, SuspendReason::kInternal, &timed_out);
    }
    if (owner != nullptr) {
      // We succeeded in suspending the thread, check the lock's status didn't change.
      lock_word = obj->GetLockWord(true);
      if (lock_word.GetState() == LockWord::kThinLocked &&
          lock_word.ThinLockOwner() == owner_thread_id) {
        // Go ahead and inflate the lock.
        Inflate(self, owner, obj.Get(), hash_code);
      }
      bool resumed = thread_list->Resume(owner, SuspendReason::kInternal);
      DCHECK(resumed);
    }
    self->SetMonitorEnterObject(nullptr);
  }
}

// 具体的膨胀过程
// 膨胀到 fat lock 后多个一个概念 Monitor
void Monitor::Inflate(Thread* self, Thread* owner, ObjPtr<mirror::Object> obj, int32_t hash_code) {
  // Allocate and acquire a new monitor.
  Monitor* m = MonitorPool::CreateMonitor(self, owner, obj, hash_code);
  if (m->Install(self)) {
    Runtime::Current()->GetMonitorList()->Add(m);
    CHECK_EQ(obj->GetLockWord(true).GetState(), LockWord::kFatLocked);
  } else {
    MonitorPool::ReleaseMonitor(self, m);
  }
}

// fat lock 在这里被设置到 Object.monitor
bool Monitor::Install(Thread* self) NO_THREAD_SAFETY_ANALYSIS {
  Thread* owner = owner_.load(std::memory_order_relaxed);
  CHECK(owner == nullptr || owner == self || (ART_USE_FUTEXES && owner->IsSuspended()));
  LockWord lw(GetObject()->GetLockWord(false));
  switch (lw.GetState()) {
    case LockWord::kThinLocked: {
      lock_count_ = lw.ThinLockCount();
#if ART_USE_FUTEXES
      monitor_lock_.ExclusiveLockUncontendedFor(owner);
#else
      monitor_lock_.ExclusiveLock(owner);
#endif
      LockWord fat(this, lw.GCState());
      // Publish the updated lock word, which may race with other threads.
      bool success = GetObject()->CasLockWord(lw, fat, CASMode::kWeak, std::memory_order_release);
      if (success) {
        if (ATraceEnabled()) {
          SetLockingMethod(owner);
        }
        return true;
      } else {
#if ART_USE_FUTEXES
        monitor_lock_.ExclusiveUnlockUncontended();
#else
        for (uint32_t i = 0; i <= lockCount; ++i) {
          monitor_lock_.ExclusiveUnlock(owner);
        }
#endif
        return false;
      }
    }
    // ...
}

// 上面说过，thin lock 的 LockWord 低 16 bits 是 thread id，然后是 12 bits 的锁个数
// 对于 fat lock，低 28 bits 是 monitor id
inline LockWord::LockWord(Monitor* mon, uint32_t gc_state)
    : value_(mon->GetMonitorId() | (gc_state << kGCStateShift) | (kStateFat << kStateShift)) {
#ifndef __LP64__
  DCHECK_ALIGNED(mon, kMonitorIdAlignment);
#endif
  DCHECK_EQ(FatLockMonitor(), mon);
  DCHECK_LE(mon->GetMonitorId(), static_cast<uint32_t>(kMaxMonitorId));
  CheckReadBarrierState();
}
```

### `Monitor::Lock` 挂起线程 

上面介绍的是对象锁，也就是把 `Object` 作为 `Lock` 使用，具体来说是 `Object.monitor` 的四种状态：无锁、偏向锁、轻量级锁和重量级锁

而 `Monitor::Lock` 实现的是在重量级锁状态下，「挂起」线程的过程，它包含了自旋、futex/mutex 系统调用

```cpp
template <LockReason reason>
void Monitor::Lock(Thread* self) {
  bool called_monitors_callback = false;
  // 一小会的自旋
  if (TryLock(self, /*spin=*/ true)) {
  //... 挂起
  // Acquire monitor_lock_ without mutator_lock_, expecting to block this time.
  // We already tried spinning above. The shutdown procedure currently assumes we stop
  // touching monitors shortly after we suspend, so don't spin again here.
  monitor_lock_.ExclusiveLock(self);
  //...
}

// 自己持有锁，锁加一；否则自旋一小会尝试加锁
bool Monitor::TryLock(Thread* self, bool spin) {
  Thread *owner = owner_.load(std::memory_order_relaxed);
  if (owner == self) {
    lock_count_++;
    CHECK_NE(lock_count_, 0u);  // Abort on overflow.
  } else {
    bool success = spin ? monitor_lock_.ExclusiveTryLockWithSpinning(self)
        : monitor_lock_.ExclusiveTryLock(self);
    //...
}
```

## `monitor-exit` 指令

### 偏向锁和轻量级锁

退出临界区，释放锁

```cpp
// platform/superproject/art/runtime/mirror/object-inl.h
inline bool Object::MonitorExit(Thread* self) {
  return Monitor::MonitorExit(self, this);
}
bool Monitor::MonitorExit(Thread* self, ObjPtr<mirror::Object> obj) {
  //...
  while (true) {
    LockWord lock_word = obj->GetLockWord(true);
    switch (lock_word.GetState()) {
      case LockWord::kHashCode:
        // Fall-through.
			
      // 对象的锁并没有借出，抛出 java 异常
      case LockWord::kUnlocked:
        FailedUnlock(h_obj.Get(), self->GetThreadId(), 0u, nullptr);
        return false;  // Failure.

      // 当前线程并不拥有锁，抛出 java 异常
      case LockWord::kThinLocked: {
        uint32_t thread_id = self->GetThreadId();
        uint32_t owner_thread_id = lock_word.ThinLockOwner();
        if (owner_thread_id != thread_id) {
          FailedUnlock(h_obj.Get(), thread_id, owner_thread_id, nullptr);
          return false;  // Failure.
        } else {

          // 偏向锁，锁减一，如果锁为零则释放锁，最后写回 Object.monitor
          // We own the lock, decrease the recursion count.
          LockWord new_lw = LockWord::Default();
          if (lock_word.ThinLockCount() != 0) {
            uint32_t new_count = lock_word.ThinLockCount() - 1;
            new_lw = LockWord::FromThinLockId(thread_id, new_count, lock_word.GCState());
          } else {
            new_lw = LockWord::FromDefault(lock_word.GCState());
          }
          if (!kUseReadBarrier) {
            DCHECK_EQ(new_lw.ReadBarrierState(), 0U);
            h_obj->SetLockWord(new_lw, true);
            AtraceMonitorUnlock();
            return true;
          } else {
            if (h_obj->CasLockWord(lock_word, new_lw, CASMode::kWeak, std::memory_order_release)) {
              AtraceMonitorUnlock();
              return true;
            }
          }
          continue;  // Go again.
        }
      }
			
      // 释放重量级锁
      case LockWord::kFatLocked: {
        Monitor* mon = lock_word.FatLockMonitor();
        return mon->Unlock(self);
      }

      default: {
        LOG(FATAL) << "Invalid monitor state " << lock_word.GetState();
        UNREACHABLE();
      }
    }
  }
}
```

### 重量级锁

```cpp
// 当前线程持有此重量级锁，且锁为零，退出临界区导致释放锁并唤醒等待线程
bool Monitor::Unlock(Thread* self) {
  DCHECK(self != nullptr);
  Thread* owner = owner_.load(std::memory_order_relaxed);
  if (owner == self) {
    // We own the monitor, so nobody else can be in here.
    CheckLockOwnerRequest(self);
    AtraceMonitorUnlock();
    if (lock_count_ == 0) {
      owner_.store(nullptr, std::memory_order_relaxed);
      SignalWaiterAndReleaseMonitorLock(self);
    } else {

      // 当前线程持有此重量级锁，且锁不为零（重入）
      // 退出临界区导致锁减一，但不释放锁
      --lock_count_;
      DCHECK(monitor_lock_.IsExclusiveHeld(self));
      DCHECK_EQ(owner_.load(std::memory_order_relaxed), self);
      // Keep monitor_lock_, but pretend we released it.
      FakeUnlockMonitorLock();
    }
    return true;
  }

  // 当前线程不持有此重量级锁，抛出 java 异常
  // We don't own this, so we're not allowed to unlock it.
  // The JNI spec says that we should throw IllegalMonitorStateException in this case.
  uint32_t owner_thread_id = 0u;
  {
    MutexLock mu(self, *Locks::thread_list_lock_);
    owner = owner_.load(std::memory_order_relaxed);
    if (owner != nullptr) {
      owner_thread_id = owner->GetThreadId();
    }
  }
  FailedUnlock(GetObject(), self->GetThreadId(), owner_thread_id, this);
  // Pretend to release monitor_lock_, which we should not.
  FakeUnlockMonitorLock();
  return false;
}

// 释放重量级锁
void Monitor::SignalWaiterAndReleaseMonitorLock(Thread* self) {
  // ...
  monitor_lock_.Unlock(self);
}
```

## 总结

根据上面的代码总结下 `synchronized` 加锁的流程：

- 初始为无锁
- 线程 A 进入临界区获得锁，升级为偏向锁（thin lock），偏向锁记下线程 A 的 thread id 和初始锁个数 0
- 如果线程 A 重入临界区，锁个数加一；当锁个数超过阈值时，膨胀为重量级锁
- 如果在线程 A 持有偏向锁的情况下，线程 B 尝试进入临界区；那么线程 B 首先自旋一小会等待线程 A 释放锁，失败后将偏向锁膨胀为重量级锁，并在锁上挂起；这一过程称为轻量级锁
- 线程 A 持有重量级锁的情况下，其他线程尝试进入临界区，会在锁上挂起（futex/mutex）

总结下 `synchronized` 和 `Lock` 的区别：

- `synchronized` 使用 `Object` 作为锁，也即所有的 `Object` 都可以当做锁使用；但具体的 lock/unlock 逻辑是在 `Monitor` 实现的，严谨地说是 `Object` + `Monitor` = Lock
- 偏向锁和轻量级锁并没有使用 `Monitor`，而是用 cas，`Object::monitor` 和自旋实现排他性；直到重量级锁时才构造 `Monitor`；`Monitor` 除了扮演 Lock 的角色外，[还扮演了 Condition 的角色](../../../../2021/02/11/wait-notify/)，所以一旦调用 Object.wait/Object.notify，就会立刻升级为重量级锁
- Lock 用排队队列来组织挂起的线程，而且以 FIFO 的优先级排队；`synchronized` 没有组织挂起的线程，完全由 CPU 决定谁能获得锁，可能会发生「饥饿」问题
- Lock 全靠 futex/mutex 阻塞线程，而 `synchronized` 先让线程自旋一会在陷入阻塞