---
title: Lock（五）Condition 的语言实现：Object.wait 和 Object.notify
date: 2021-02-11 12:00:00 +0800
categories: [Android, Art]
tags: [wait, notify]
---

在 [Lock（三）利用 Lock 实现 Condition](../condition-by-lock/) 我们介绍了如何用 `Lock` 来实现 `Condition`，而 `Condition` 对标的是 `Object.wait` 和 `Object.notify`

我们来看看 ART 是怎么实现 wait/notify 的（最好先了解下 [synchronized 的基础知识](../synchronized-implementation/)）

## ConditionVariable

对 futex/mutex 的封装，宏 `ART_USE_FUTEXES` 决定底层是使用 futex 还是 mutex；它不是「条件变量」，`Monitor` 才是（而且它还包含 Lock 的角色）

```cpp
// art/runtime/base/mutex.cc
void ConditionVariable::Wait(Thread* self) {
  guard_.CheckSafeToWait(self);
  WaitHoldingLocks(self);
}

void ConditionVariable::WaitHoldingLocks(Thread* self) {
  DCHECK(self == nullptr || self == Thread::Current());
  guard_.AssertExclusiveHeld(self);
  unsigned int old_recursion_count = guard_.recursion_count_;
#if ART_USE_FUTEXES
  num_waiters_++;
  // Ensure the Mutex is contended so that requeued threads are awoken.
  guard_.increment_contenders();
  guard_.recursion_count_ = 1;
  int32_t cur_sequence = sequence_.load(std::memory_order_relaxed);
  guard_.ExclusiveUnlock(self);
  if (futex(sequence_.Address(), FUTEX_WAIT_PRIVATE, cur_sequence, nullptr, nullptr, 0) != 0) {
    // Futex failed, check it is an expected error.
    // EAGAIN == EWOULDBLK, so we let the caller try again.
    // EINTR implies a signal was sent to this thread.
    if ((errno != EINTR) && (errno != EAGAIN)) {
      PLOG(FATAL) << "futex wait failed for " << name_;
    }
  }
  SleepIfRuntimeDeleted(self);
  guard_.ExclusiveLock(self);
  CHECK_GT(num_waiters_, 0);
  num_waiters_--;
  // We awoke and so no longer require awakes from the guard_'s unlock.
  CHECK_GT(guard_.get_contenders(), 0);
  guard_.decrement_contenders();
#else
  pid_t old_owner = guard_.GetExclusiveOwnerTid();
  guard_.exclusive_owner_.store(0 /* pid */, std::memory_order_relaxed);
  guard_.recursion_count_ = 0;
  CHECK_MUTEX_CALL(pthread_cond_wait, (&cond_, &guard_.mutex_));
  guard_.exclusive_owner_.store(old_owner, std::memory_order_relaxed);
#endif
  guard_.recursion_count_ = old_recursion_count;
}

void ConditionVariable::Signal(Thread* self) {
  DCHECK(self == nullptr || self == Thread::Current());
  guard_.AssertExclusiveHeld(self);
#if ART_USE_FUTEXES
  RequeueWaiters(1);
#else
  CHECK_MUTEX_CALL(pthread_cond_signal, (&cond_));
#endif
}

void ConditionVariable::RequeueWaiters(int32_t count) {
  if (num_waiters_ > 0) {
    sequence_++;  // Indicate a signal occurred.
    // Move waiters from the condition variable's futex to the guard's futex,
    // so that they will be woken up when the mutex is released.
    bool done = futex(sequence_.Address(),
                      FUTEX_REQUEUE_PRIVATE,
                      /* Threads to wake */ 0,
                      /* Threads to requeue*/ reinterpret_cast<const timespec*>(count),
                      guard_.state_and_contenders_.Address(),
                      0) != -1;
    if (!done && errno != EAGAIN && errno != EINTR) {
      PLOG(FATAL) << "futex requeue failed for " << name_;
    }
  }
}
```

## Monitor

从逻辑上实现了「条件变量」，对应 Condition；它有两条单向链表的排队队列：

- 等待队列，被挂起的线程（wait）在这里排队，`Monitor::wait_set_` 是队头
- 唤醒队列，等待被唤醒的线程（notify）在这里排队，`Monitor::wake_set_` 是队头

`Thread` 有个成员变量充当 next 指针：`Thread::GetWaitNext()` 和 `Thread::SetWaitNext(Thread* next)`

await 是把线程添加到 wait set 队尾，notify 是把 wait set 队头转移为 wake set 队头，然后在退出临界区（释放锁）时唤醒 wake set 队头

同时 `Thread::wait_monitor_` 标识线程在哪个 `Monitor` 上挂起

```cpp
class Monitor {
  // Threads currently waiting on this monitor.
  Thread* wait_set_ GUARDED_BY(monitor_lock_);
  // Threads that were waiting on this monitor, but are now contending on it.
  Thread* wake_set_ GUARDED_BY(monitor_lock_);
}

// 将新挂起的线程添加到 wait set 队尾
void Monitor::AppendToWaitSet(Thread* thread) {
  // Not checking that the owner is equal to this thread, since we've released
  // the monitor by the time this method is called.
  DCHECK(thread != nullptr);
  DCHECK(thread->GetWaitNext() == nullptr) << thread->GetWaitNext();
  if (wait_set_ == nullptr) {
    wait_set_ = thread;
    return;
  }

  // push_back.
  Thread* t = wait_set_;
  while (t->GetWaitNext() != nullptr) {
    t = t->GetWaitNext();
  }
  t->SetWaitNext(thread);
}

// notify 并没有唤醒线程，而是把 wait set 的队头转移到 wake set 队头
// 实际上是在释放锁时唤醒 wake set 队头
void Monitor::Notify(Thread* self) {
  DCHECK(self != nullptr);
  // Make sure that we hold the lock.
  if (owner_.load(std::memory_order_relaxed) != self) {
    ThrowIllegalMonitorStateExceptionF("object not locked by thread before notify()");
    return;
  }
  // Move one thread from waiters to wake set
  Thread* to_move = wait_set_;
  if (to_move != nullptr) {
    wait_set_ = to_move->GetWaitNext();
    to_move->SetWaitNext(wake_set_);
    wake_set_ = to_move;
  }
}
```

## Object.wait

```java
void Object.wait() throws InterruptedException {
    wait(0);
}
void Object.wait(long timeout) throws InterruptedException {
    wait(timeout, 0);
}
native void Object.wait(long timeout, int nanos) throws InterruptedException;
```

```cpp
// art/runtime/native/java_lang_Object.cc

static void Object_waitJI(JNIEnv* env, jobject java_this, jlong ms, jint ns) {
  ScopedFastNativeObjectAccess soa(env);
  soa.Decode<mirror::Object>(java_this)->Wait(soa.Self(), ms, ns);
}
inline void Object::Wait(Thread* self, int64_t ms, int32_t ns) {
  Monitor::Wait(self, this, ms, ns, true, kTimedWaiting);
}

void Monitor::Wait(Thread* self,
                   ObjPtr<mirror::Object> obj,
                   int64_t ms,
                   int32_t ns,
                   bool interruptShouldThrow,
                   ThreadState why) {
  DCHECK(self != nullptr);
  DCHECK(obj != nullptr);
  StackHandleScope<1> hs(self);
  Handle<mirror::Object> h_obj(hs.NewHandle(obj));

  // 将锁膨胀为 fat lock
  LockWord lock_word = h_obj->GetLockWord(true);
  while (lock_word.GetState() != LockWord::kFatLocked) {
    switch (lock_word.GetState()) {
      case LockWord::kHashCode:

      // wait/notify 必须先用 synchronized 获取此对象上的锁
      // 否则抛出 java 异常
      case LockWord::kUnlocked:
        ThrowIllegalMonitorStateExceptionF("object not locked by thread before wait()");
        return;  // Failure.

      // 同上，必须获得此对象锁；此时对象锁被别的线程持有，抛出 java 异常
      case LockWord::kThinLocked: {
        uint32_t thread_id = self->GetThreadId();
        uint32_t owner_thread_id = lock_word.ThinLockOwner();
        if (owner_thread_id != thread_id) {
          ThrowIllegalMonitorStateExceptionF("object not locked by thread before wait()");
          return;  // Failure.
        } else {

          // 将 thin lock（偏向锁）膨胀为 fat lock（重量级锁），同时创建一个监视器 Monitor
          // We own the lock, inflate to enqueue ourself on the Monitor. May fail spuriously so
          // re-load.
          Inflate(self, self, h_obj.Get(), 0);
          lock_word = h_obj->GetLockWord(true);
        }
        break;
      }

      // 已经是 fat lock 了
      case LockWord::kFatLocked:  // Unreachable given the loop condition above. Fall-through.
      default: {
        LOG(FATAL) << "Invalid monitor state " << lock_word.GetState();
        UNREACHABLE();
      }
    }
  }

  // 必须膨胀为 fat lock，它才有 Monitor
  Monitor* mon = lock_word.FatLockMonitor();
  mon->Wait(self, ms, ns, interruptShouldThrow, why);
}

// 在监视器上挂起
void Monitor::Wait(Thread* self, int64_t ms, int32_t ns,
                   bool interruptShouldThrow, ThreadState why) {
  /*
   * Release our hold - we need to let it go even if we're a few levels
   * deep in a recursive lock, and we need to restore that later.
   */
  unsigned int prev_lock_count = lock_count_;
  lock_count_ = 0;

  // 挂起线程前需要释放锁
  // 将线程添加到 wait set 队尾，释放锁，wake set 不为空则唤醒第一个（队头开始）
  bool was_interrupted = false;
  bool timed_out = false;
  owner_.store(nullptr, std::memory_order_relaxed);
  num_waiters_.fetch_add(1, std::memory_order_relaxed);
  {
    ScopedThreadSuspension sts(self, why);
    MutexLock mu(self, *self->GetWaitMutex());
    AppendToWaitSet(self);
    self->SetWaitMonitor(this);
    SignalWaiterAndReleaseMonitorLock(self);
    // Handle the case where the thread was interrupted before we called wait().
    if (self->IsInterrupted()) {
      was_interrupted = true;
    } else {

      // 然后将线程在它的成员变量 Thread.wait_cond_ 上挂起
      // Wait for a notification or a timeout to occur.
      if (why == kWaiting) {
        self->GetWaitConditionVariable()->Wait(self);
      } else {
        DCHECK(why == kTimedWaiting || why == kSleeping) << why;
        timed_out = self->GetWaitConditionVariable()->TimedWait(self, ms, ns);
      }
      was_interrupted = self->IsInterrupted();
    }
  }

  // 线程被唤醒后，要将线程上的监视器置空，并重新获得锁
  {
    // We reset the thread's wait_monitor_ field after transitioning back to runnable so
    // that a thread in a waiting/sleeping state has a non-null wait_monitor_ for debugging
    // and diagnostic purposes. (If you reset this earlier, stack dumps will claim that threads
    // are waiting on "null".)
    MutexLock mu(self, *self->GetWaitMutex());
    DCHECK(self->GetWaitMonitor() != nullptr);
    self->SetWaitMonitor(nullptr);
  }
  Lock<LockReason::kForWait>(self);
  lock_count_ = prev_lock_count;
  DCHECK(monitor_lock_.IsExclusiveHeld(self));
  self->GetWaitMutex()->AssertNotHeld(self);
  num_waiters_.fetch_sub(1, std::memory_order_relaxed);
  RemoveFromWaitSet(self);
}
```

## Object.notify

把挂起的线程从 wait set 转移到 wake set

```cpp
static void Object_notify(JNIEnv* env, jobject java_this) {
  ScopedFastNativeObjectAccess soa(env);
  soa.Decode<mirror::Object>(java_this)->Notify(soa.Self());
}
inline void Object::Notify(Thread* self) {
  Monitor::Notify(self, this);
}
static void Notify(Thread* self, ObjPtr<mirror::Object> obj)
      REQUIRES_SHARED(Locks::mutator_lock_) {
    DoNotify(self, obj, false);
}

void Monitor::DoNotify(Thread* self, ObjPtr<mirror::Object> obj, bool notify_all) {
  DCHECK(self != nullptr);
  DCHECK(obj != nullptr);
  LockWord lock_word = obj->GetLockWord(true);
  switch (lock_word.GetState()) {
    case LockWord::kHashCode:
      // Fall-through.
    case LockWord::kUnlocked:
      ThrowIllegalMonitorStateExceptionF("object not locked by thread before notify()");
      return;  // Failure.
    case LockWord::kThinLocked: {
      uint32_t thread_id = self->GetThreadId();
      uint32_t owner_thread_id = lock_word.ThinLockOwner();
      if (owner_thread_id != thread_id) {
        ThrowIllegalMonitorStateExceptionF("object not locked by thread before notify()");
        return;  // Failure.
      } else {
        // We own the lock but there's no Monitor and therefore no waiters.
        return;  // Success.
      }
    }
    case LockWord::kFatLocked: {
      Monitor* mon = lock_word.FatLockMonitor();
      if (notify_all) {
        mon->NotifyAll(self);
      } else {
        mon->Notify(self);
      }
      return;  // Success.
    }
    default: {
      LOG(FATAL) << "Invalid monitor state " << lock_word.GetState();
      UNREACHABLE();
    }
  }
}

void Monitor::Notify(Thread* self) {
  DCHECK(self != nullptr);
  // Make sure that we hold the lock.
  if (owner_.load(std::memory_order_relaxed) != self) {
    ThrowIllegalMonitorStateExceptionF("object not locked by thread before notify()");
    return;
  }
  // Move one thread from waiters to wake set
  Thread* to_move = wait_set_;
  if (to_move != nullptr) {
    wait_set_ = to_move->GetWaitNext();
    to_move->SetWaitNext(wake_set_);
    wake_set_ = to_move;
  }
}
```

调用 notify 前需要先获得它的对象锁，notify 把线程转移到 wake set，释放锁时会唤醒线程（从而让线程能够重新获得锁）

```cpp
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
      --lock_count_;
      DCHECK(monitor_lock_.IsExclusiveHeld(self));
      DCHECK_EQ(owner_.load(std::memory_order_relaxed), self);
      // Keep monitor_lock_, but pretend we released it.
      FakeUnlockMonitorLock();
    }
    return true;
  }
  // ...
}

void Monitor::SignalWaiterAndReleaseMonitorLock(Thread* self) {
  // We want to release the monitor and signal up to one thread that was waiting
  // but has since been notified.
  DCHECK_EQ(lock_count_, 0u);
  DCHECK(monitor_lock_.IsExclusiveHeld(self));
  while (wake_set_ != nullptr) {
    // No risk of waking ourselves here; since monitor_lock_ is not released until we're ready to
    // return, notify can't move the current thread from wait_set_ to wake_set_ until this
    // method is done checking wake_set_.
    Thread* thread = wake_set_;
    wake_set_ = thread->GetWaitNext();
    thread->SetWaitNext(nullptr);
    DCHECK(owner_.load(std::memory_order_relaxed) == nullptr);

    // Check to see if the thread is still waiting.
    {
      // In the case of wait(), we'll be acquiring another thread's GetWaitMutex with
      // self's GetWaitMutex held. This does not risk deadlock, because we only acquire this lock
      // for threads in the wake_set_. A thread can only enter wake_set_ from Notify or NotifyAll,
      // and those hold monitor_lock_. Thus, the threads whose wait mutexes we acquire here must
      // have already been released from wait(), since we have not released monitor_lock_ until
      // after we've chosen our thread to wake, so there is no risk of the following lock ordering
      // leading to deadlock:
      // Thread 1 waits
      // Thread 2 waits
      // Thread 3 moves threads 1 and 2 from wait_set_ to wake_set_
      // Thread 1 enters this block, and attempts to acquire Thread 2's GetWaitMutex to wake it
      // Thread 2 enters this block, and attempts to acquire Thread 1's GetWaitMutex to wake it
      //
      // Since monitor_lock_ is not released until the thread-to-be-woken-up's GetWaitMutex is
      // acquired, two threads cannot attempt to acquire each other's GetWaitMutex while holding
      // their own and cause deadlock.
      MutexLock wait_mu(self, *thread->GetWaitMutex());
      if (thread->GetWaitMonitor() != nullptr) {
        // Release the lock, so that a potentially awakened thread will not
        // immediately contend on it. The lock ordering here is:
        // monitor_lock_, self->GetWaitMutex, thread->GetWaitMutex
        monitor_lock_.Unlock(self);  // Releases contenders.
        thread->GetWaitConditionVariable()->Signal(self);
        return;
      }
    }
  }
  monitor_lock_.Unlock(self);
  DCHECK(!monitor_lock_.IsExclusiveHeld(self));
}
```

## 总结下

- 与 JCU.Condition 不同，对象监视器 Monitor 并没有把「条件变量」这部分功能抽离出来，它既是 Lock 又是 Condition
- Condition 和 Monitor 都用排队队列来组织挂起的线程
- Condition 在 notify 后立刻唤醒线程，而 Monitor 因为 wait/notify 需要获得锁后才能执行，只能在 notify 线程释放锁时才唤醒 wait 线程