---
title: 深入 OOM
date: 2021-04-16 12:00:00 +0800
categories: [Android, 内存]
tags: [OOM]
---

## 堆内存分配失败导致的 OOM

### 度量

采用下述 API 来度量 APP 或系统的内存使用情况

| 类别 | API                                     | 说明                                                                       |
|-----|-----------------------------------------|---------------------------------------------------------------------------|
| APP | `Runtime.maxMemory()`                   | JVM 可以从系统那申请到的内存的最大值，`ActivityManager.getMemoryClass()` 和 `ActivityManager.getLargeMemoryClass()` 其中之一，超过此阈值会发生 OOM |
|     | `Runtime.totalMemory()`                 | JVM 已申请到的内存大小，小于等于 `Runtime.maxMemory()` 且大于 `Runtime.freeMemory()`；当需要的内存（比如创建 1M 的字节数组）超过 `Runtime.freeMemory()` 时 JVM 会向系统申请内存，此时 `Runtime.totalMemory()` 会逐渐增大；直到等于 `Runtime.maxMemory()` 时，如果需要的内存超过 `Runtime.freeMemory()` 则抛出 OOM |
|     | `Runtime.freeMemory()`                  | JVM 已申请到但仍未使用的内存，当需要的内存超过此值时，JVM 会向系统申请内存                                           |
|     | `ActivityManager.getMemoryClass()`      | APP 可申请的内存上限                                                                                         |
|     | `ActivityManager.getLargeMemoryClass()` | 设置 `android:largeHeap="true"` 后 APP 可申请的内存上限，一般是 `ActivityManager.getLargeMemoryClass()` 的两倍  |
| 系统 | `ActivityManager.MemoryInfo.availMem`   | 当前系统可用内存大小，即设置里可用运存的值                                                                       |
|     | `ActivityManager.MemoryInfo.totalMem`   | 系统总内存大小，即设置里运存总空间的值                                                                           |
|     | `ActivityManager.MemoryInfo.lowMemory`  | 标识系统是否处于低内存状态                                                                                     |
|     | `ActivityManager.MemoryInfo.threshold`  | 当系统可用内存小于此阈值时，系统处于低内存状态                                                                    |

### 测试

使用下面的测试代码在我的测试机（iQOO 3）上，循环申请 30M/5M/500K 的内存，每次间隔 1s 并打印内存统计情况

可以看到 JVM 使用内存上限 `max` 是 256M（刚好是 `memoryClass`，如果配置了 `largeHeap="true"` 则是 `largeMemoryClass`），已申请内存 `total` 逐步上升直到 256M，`total` - `free` 等于申请内存的累计量；运行内存为 11.36GB，可用运存徘徊在 5GB 左右，当运存掉到 216M 时进入低内存状态，所以目前并不是低内存；APP 可申请内存上限为 256MB，设置 `largeHeap="true"` 后翻倍到 512MB

抛出 OOM 时的信息为：`Failed to allocate a 512016 byte allocation with 22760 free bytes and 22KB until OOM, target footprint 268435456, growth limit 268435456`

参考日志的最后一行，当时正在申请 500KB 的内存（`Failed to allocate a 512016 byte allocation`），还剩 14.95KB（`with 22760 free bytes and 22KB until OOM`），上限是 256MB（`target footprint 268435456, growth limit 268435456`）

```kotlin
// 循环申请内存直到 OOM
fun mallocJVM() {
    val steps = arrayOf(1024L * 1024 * 30, 1024L * 1024 * 5, 1024L * 500)
    val pool = mutableListOf<ByteArray>()
    while (true) {
        val runtime = Runtime.getRuntime()
        val free = runtime.maxMemory() - runtime.totalMemory() + runtime.freeMemory()
        val size = if (free >= steps[0]) steps[0] else if (free >= steps[1]) steps[1] else steps[2]
        pool += ByteArray(size = size.toInt())
        await()
        printMemoryUsages()
    }
}

// 间隔 1s
fun await(time: Long = 1000) {
    lock.lock()
    try {
        cond.await(time, TimeUnit.MILLISECONDS)
    } finally {
        lock.unlock()
    }
}

// 打印内存统计情况
fun printMemoryUsages() {
    val runtime = Runtime.getRuntime()
    am.getMemoryInfo(memInfo)
    d("max:${runtime.maxMemory} total:${runtime.totalMemory} free:${runtime.freeMemory} " +
            "availMem:${memInfo.availMemReadable} totalMem:${memInfo.totalMemReadable} " +
            "threshold:${memInfo.thresholdReadable} lowMemory:${memInfo.lowMemory} " +
            "memoryClass:${am.memoryClassReadable} largeMemoryClass:${am.largeMemoryClassReadable}")
}
```

![oom_memory_log.png](../../../../image/2021-04-16-looking-into-oom/oom_memory_log.png)

![oom_memory.png](../../../../image/2021-04-16-looking-into-oom/oom_memory.png)

### 定位抛出 OOM 的位置

最终抛出 `OutOfMemoryError` 的代码 是 `Thread::ThrowOutOfMemoryError`

```C++
void Thread::ThrowOutOfMemoryError(const char* msg) {
  LOG(WARNING) << "Throwing OutOfMemoryError "
               << '"' << msg << '"'
               << " (VmSize " << GetProcessStatus("VmSize")
               << (tls32_.throwing_OutOfMemoryError ? ", recursive case)" : ")");
  if (!tls32_.throwing_OutOfMemoryError) {
    tls32_.throwing_OutOfMemoryError = true;
    ThrowNewException("Ljava/lang/OutOfMemoryError;", msg);
    tls32_.throwing_OutOfMemoryError = false;
  } else {
    Dump(LOG_STREAM(WARNING));  // The pre-allocated OOME has no stack, so help out and log one.
    SetException(Runtime::Current()->GetPreAllocatedOutOfMemoryErrorWhenThrowingOOME());
  }
}

void Thread::ThrowNewException(const char* exception_class_descriptor,
                               const char* msg) {
  // Callers should either clear or call ThrowNewWrappedException.
  AssertNoPendingExceptionForNewException(msg);
  ThrowNewWrappedException(exception_class_descriptor, msg);
}

void Thread::ThrowNewWrappedException(const char* exception_class_descriptor,
                                      const char* msg) {
  // ... 构造 Ljava/lang/OutOfMemoryError;
}
```

而因为堆内存分配失败抛出 OOM 的代码在 `Heap::ThrowOutOfMemoryError`，其中异常信息为：

`"Failed to allocate a (x) byte allocation with (x) free bytes and (x) until OOM, target footprint (x), growth limit (x)"`

```C++
void Heap::ThrowOutOfMemoryError(Thread* self, size_t byte_count, AllocatorType allocator_type) {
  // If we're in a stack overflow, do not create a new exception. It would require running the
  // constructor, which will of course still be in a stack overflow.
  if (self->IsHandlingStackOverflow()) {
    self->SetException(
        Runtime::Current()->GetPreAllocatedOutOfMemoryErrorWhenHandlingStackOverflow());
    return;
  }

  std::ostringstream oss;
  size_t total_bytes_free = GetFreeMemory();
  oss << "Failed to allocate a " << byte_count << " byte allocation with " << total_bytes_free
      << " free bytes and " << PrettySize(GetFreeMemoryUntilOOME()) << " until OOM,"
      << " target footprint " << target_footprint_.load(std::memory_order_relaxed)
      << ", growth limit "
      << growth_limit_;
  // If the allocation failed due to fragmentation, print out the largest continuous allocation.
  if (total_bytes_free >= byte_count) {
    space::AllocSpace* space = nullptr;
    if (allocator_type == kAllocatorTypeNonMoving) {
      space = non_moving_space_;
    } else if (allocator_type == kAllocatorTypeRosAlloc ||
               allocator_type == kAllocatorTypeDlMalloc) {
      space = main_space_;
    } else if (allocator_type == kAllocatorTypeBumpPointer ||
               allocator_type == kAllocatorTypeTLAB) {
      space = bump_pointer_space_;
    } else if (allocator_type == kAllocatorTypeRegion ||
               allocator_type == kAllocatorTypeRegionTLAB) {
      space = region_space_;
    }

    // There is no fragmentation info to log for large-object space.
    if (allocator_type != kAllocatorTypeLOS) {
      CHECK(space != nullptr) << "allocator_type:" << allocator_type
                              << " byte_count:" << byte_count
                              << " total_bytes_free:" << total_bytes_free;
      space->LogFragmentationAllocFailure(oss, byte_count);
    }
  }
  self->ThrowOutOfMemoryError(oss.str().c_str());
}
```

## 创建线程时 `Could not allocate JNI Env` 导致的 OOM

### 测试代码

线程数量可以通过 `/proc/{pid}/status` 里 `Threads` 那行拿到

```javascript
Name:	m.myapplication
State:	R (running)
Tgid:	26939
Ngid:	0
Pid:	26939
PPid:	1846
TracerPid:	0
Uid:	10100	10100	10100	10100
Gid:	10100	10100	10100	10100
FDSize:	64
Groups:	3003 9997 20100 50100 
...
Threads:	17                  // 线程数量
...
```

不断地创建新线程，新线程啥也不干就阻塞住，从而使 APP 进程的线程数量持续上涨，模拟 APP 随意创建线程/线程池而没有做统一的管理，导致创建大量线程的情况，最后抛出 `Could not allocate JNI Env: Failed anonymous mmap(0x0, 8192, 0x3, 0x2, 53, 0): Permission denied. See process maps in the log.`

```kotlin
class MemoryKillerService : IntentService("MemoryKillerService") {
    override fun onHandleIntent(intent: Intent?) {
        // 一个线程不断地创建测试线程（间隔 1ms）
        thread {
            while (true) {
                val thread = BlockingThread()
                thread.start()
                await(time = 1)
            }
        }

        // 一个线程不断地打印线程数
        while (true) {
            d("threadCount: ${getThreadCount()}")
            await(time = 100)
        }
    }
}

// 测试线程啥也不干，就阻塞住
class BlockingThread: Thread() {
    private val lock = ReentrantLock()
    private val cond = lock.newCondition()
    override fun run() {
        lock.lock()
        try {
            cond.await()
        } finally {
            lock.unlock()
        }
    }
}

// 查询进程的线程数量
fun getThreadCount() = runCatching {
    regexGroupInFile(pattern = ".*Threads:\\s+(\\d+).*", path = "/proc/${myPid()}/status")?.toInt()
}.getOrDefault(0)
```

```javascript
2021-04-20 13:58:10.067 29743-29769/com.myapplication D/memkiller: threadCount: 1633
2021-04-20 13:58:10.170 29743-29769/com.myapplication D/memkiller: threadCount: 1680
2021-04-20 13:58:10.272 29743-29769/com.myapplication D/memkiller: threadCount: 1731
2021-04-20 13:58:10.374 29743-29769/com.myapplication D/memkiller: threadCount: 1784
2021-04-20 13:58:10.475 29743-29769/com.myapplication D/memkiller: threadCount: 1829
2021-04-20 13:58:10.578 29743-29769/com.myapplication D/memkiller: threadCount: 1880
2021-04-20 13:58:10.680 29743-29769/com.myapplication D/memkiller: threadCount: 1929
2021-04-20 13:58:10.783 29743-29769/com.myapplication D/memkiller: threadCount: 1972
2021-04-20 13:58:10.886 29743-29769/com.myapplication D/memkiller: threadCount: 2017
2021-04-20 13:58:10.988 29743-29769/com.myapplication D/memkiller: threadCount: 2067
2021-04-20 13:58:11.090 29743-29769/com.myapplication D/memkiller: threadCount: 2113
2021-04-20 13:58:11.193 29743-29769/com.myapplication D/memkiller: threadCount: 2164
2021-04-20 13:58:11.294 29743-29769/com.myapplication D/memkiller: threadCount: 2208
2021-04-20 13:58:11.368 29743-29771/com.myapplication W/m.myapplicatio: Throwing OutOfMemoryError "Could not allocate JNI Env: Failed anonymous mmap(0x0, 8192, 0x3, 0x2, 53, 0): Permission denied. See process maps in the log."
    
    --------- beginning of crash
2021-04-20 13:58:11.368 29743-29771/com.myapplication E/AndroidRuntime: FATAL EXCEPTION: Thread-2
    Process: com.myapplication, PID: 29743
    java.lang.OutOfMemoryError: Could not allocate JNI Env: Failed anonymous mmap(0x0, 8192, 0x3, 0x2, 53, 0): Permission denied. See process maps in the log.
        at java.lang.Thread.nativeCreate(Native Method)
        at java.lang.Thread.start(Thread.java:733)
        at com.myapplication.MemoryKillerService$onHandleIntent$1.invoke(MemoryKillerService.kt:35)
        at com.myapplication.MemoryKillerService$onHandleIntent$1.invoke(MemoryKillerService.kt:22)
        at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)
```

### 代码追踪

```cpp
Thread.start()
Thread.nativeCreate
Thread_nativeCreate

void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
  // ...
  // Try to allocate a JNIEnvExt for the thread. We do this here as we might be out of memory and
  // do not have a good way to report this on the child's side.
  std::string error_msg;
  std::unique_ptr<JNIEnvExt> child_jni_env_ext(
      JNIEnvExt::Create(child_thread, Runtime::Current()->GetJavaVM(), &error_msg));

  // 成功创建 JNIEnvExt 后才会真正创建线程 pthread_create ...

  // 找到了错误信息 Could not allocate JNI Env，它是由 child_jni_env_ext.get() == nullptr 触发的
  // 也就是 JNIEnvExt::Create 返回 nullptr，JNIEnvExt 创建失败
  {
    std::string msg(child_jni_env_ext.get() == nullptr ?
        StringPrintf("Could not allocate JNI Env: %s", error_msg.c_str()) :
        StringPrintf("pthread_create (%s stack) failed: %s",
                                 PrettySize(stack_size).c_str(), strerror(pthread_create_result)));
    ScopedObjectAccess soa(env);
    soa.Self()->ThrowOutOfMemoryError(msg.c_str());
  }
}

// 发现 JNIEnvExt.locals_.table_mem_map_.IsValid 返回 false 导致 JNIEnvExt 创建失败
JNIEnvExt::Create
JNIEnvExt::CheckLocalsValid
IndirectReferenceTable::IsValid
MemMap.IsValid

// locals_ 是在 JNIEnvExt 的构造函数里初始化的
JNIEnvExt::JNIEnvExt(Thread* self_in, JavaVMExt* vm_in, std::string* error_msg)
    : self_(self_in),
      vm_(vm_in),
      local_ref_cookie_(kIRTFirstSegment),
      locals_(kLocalsInitial, kLocal, IndirectReferenceTable::ResizableCapacity::kYes, error_msg),
      monitors_("monitors", kMonitorsInitial, kMonitorsMax),
      critical_(0),
      check_jni_(false),
      runtime_deleted_(false) {
  MutexLock mu(Thread::Current(), *Locks::jni_function_table_lock_);
  check_jni_ = vm_in->IsCheckJniEnabled();
  functions = GetFunctionTable(check_jni_);
  unchecked_functions_ = GetJniNativeInterface();
}

// table_mem_map_ 是由 MemMap::MapAnonymous 创建的
IndirectReferenceTable::IndirectReferenceTable(size_t max_count,
                                               IndirectRefKind desired_kind,
                                               ResizableCapacity resizable,
                                               std::string* error_msg)
    : segment_state_(kIRTFirstSegment),
      kind_(desired_kind),
      max_entries_(max_count),
      current_num_holes_(0),
      resizable_(resizable) {
  CHECK(error_msg != nullptr);
  CHECK_NE(desired_kind, kJniTransitionOrInvalid);

  // Overflow and maximum check.
  CHECK_LE(max_count, kMaxTableSizeInBytes / sizeof(IrtEntry));

  const size_t table_bytes = RoundUp(max_count * sizeof(IrtEntry), kPageSize);
  table_mem_map_ = MemMap::MapAnonymous("indirect ref table",
                                        table_bytes,
                                        PROT_READ | PROT_WRITE,
                                        /*low_4gb=*/ false,
                                        error_msg);
  if (!table_mem_map_.IsValid() && error_msg->empty()) {
    *error_msg = "Unable to map memory for indirect ref table";
  }

  if (table_mem_map_.IsValid()) {
    table_ = reinterpret_cast<IrtEntry*>(table_mem_map_.Begin());
  } else {
    table_ = nullptr;
  }
  segment_state_ = kIRTFirstSegment;
  last_known_previous_state_ = kIRTFirstSegment;
  // Take into account the actual length.
  max_entries_ = table_bytes / sizeof(IrtEntry);
}

// 找到错误信息 Failed anonymous mmap 的出处了
MemMap MemMap::MapAnonymous(const char* name,
                            uint8_t* addr,
                            size_t byte_count,
                            int prot,
                            bool low_4gb,
                            bool reuse,
                            /*inout*/MemMap* reservation,
                            /*out*/std::string* error_msg,
                            bool use_debug_name) {
  // ...
  // fd 没有指向任何有效文件，addr 也是 nullptr
  // 也就是说这里只是开辟了容量是 byte_count 的一段内存空间
  void* actual = MapInternal(addr,
                             page_aligned_byte_count,
                             prot,
                             flags,
                             fd.get(),
                             0,
                             low_4gb);
  saved_errno = errno;

  if (actual == MAP_FAILED) {
    if (error_msg != nullptr) {
      if (kIsDebugBuild || VLOG_IS_ON(oat)) {
        PrintFileToLog("/proc/self/maps", LogSeverity::WARNING);
      }

      *error_msg = StringPrintf("Failed anonymous mmap(%p, %zd, 0x%x, 0x%x, %d, 0): %s. "
                                    "See process maps in the log.",
                                addr,
                                page_aligned_byte_count,
                                prot,
                                flags,
                                fd.get(),
                                strerror(saved_errno));
    }
    return Invalid();
  }
  // ...
}

// 看来是将 fd 映射进内存地址时失败了
MemMap::MapInternal
MemMap::TargetMMap
mmap
```

### 总结

创建线程时，需要构造 `JNIEnvExt` 这么一个对象，而 `JNIEnvExt` 需要 `mmap` 一块大小为 `RoundUp(max_count * sizeof(IrtEntry), kPageSize)` 的内存地址（内存页面大小的整数倍，比如上面 logcat 里，8192 = 4096 * 2），当虚拟内存地址空间耗尽时抛出 OOM

1. `Could not allocate JNI Env` 不能为 `JNIEnvExt` 分配内存
2. `Failed anonymous mmap(0x0, 8192, 0x3, 0x2, 53, 0)` 不能分配 8192 大小的内存地址
3. `Permission denied. See process maps in the log.` 我用的模拟器，可能是说超出内存限制后不能用其他方式获得内存？

## 创建线程时 `pthread_create failed` 导致的 OOM

用上面的测试代码还会出现另外一种 OOM 如下

```kotlin
2021-04-20 13:55:37.488 27404-27431/com.myapplication D/memkiller: threadCount: 1702
2021-04-20 13:55:37.590 27404-27431/com.myapplication D/memkiller: threadCount: 1747
2021-04-20 13:55:37.692 27404-27431/com.myapplication D/memkiller: threadCount: 1796
2021-04-20 13:55:37.794 27404-27431/com.myapplication D/memkiller: threadCount: 1848
2021-04-20 13:55:37.898 27404-27431/com.myapplication D/memkiller: threadCount: 1895
2021-04-20 13:55:38.000 27404-27431/com.myapplication D/memkiller: threadCount: 1946
2021-04-20 13:55:38.103 27404-27431/com.myapplication D/memkiller: threadCount: 1997
2021-04-20 13:55:38.205 27404-27431/com.myapplication D/memkiller: threadCount: 2045
2021-04-20 13:55:38.307 27404-27431/com.myapplication D/memkiller: threadCount: 2096
2021-04-20 13:55:38.410 27404-27431/com.myapplication D/memkiller: threadCount: 2139
2021-04-20 13:55:38.513 27404-27431/com.myapplication D/memkiller: threadCount: 2175
2021-04-20 13:55:38.615 27404-27431/com.myapplication D/memkiller: threadCount: 2220
2021-04-20 13:55:38.663 27404-27433/com.myapplication W/libc: pthread_create failed: couldn't allocate TLS: Permission denied
2021-04-20 13:55:38.663 27404-27433/com.myapplication W/m.myapplicatio: Throwing OutOfMemoryError "pthread_create (1040KB stack) failed: Try again"
    
    --------- beginning of crash
2021-04-20 13:55:38.664 27404-27433/com.myapplication E/AndroidRuntime: FATAL EXCEPTION: Thread-2
    Process: com.myapplication, PID: 27404
    java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Try again
        at java.lang.Thread.nativeCreate(Native Method)
        at java.lang.Thread.start(Thread.java:733)
        at com.myapplication.MemoryKillerService$onHandleIntent$1.invoke(MemoryKillerService.kt:35)
        at com.myapplication.MemoryKillerService$onHandleIntent$1.invoke(MemoryKillerService.kt:22)
        at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)
```

创建线程最终会执行系统调用 `pthread_create`，如果失败会抛出 `pthread_create (1040KB stack) failed`，1040KB 是线程的栈大小

上面 `pthread_create failed: couldn't allocate TLS: Permission denied` 表示为线程分配 Thread Local（TLS，THREAD LOCAL STORAGE）相关的内存时因为内存不足失败了

```cpp
Thread.start()
Thread.nativeCreate
Thread_nativeCreate

void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
  // ...
  // 不同于上面的情况，此时已为 JNIEnvExt 分配内存，但在执行 pthread_create 时出错了
  int pthread_create_result = 0;
  if (child_jni_env_ext.get() != nullptr) {
    pthread_t new_pthread;
    pthread_attr_t attr;
    child_thread->tlsPtr_.tmp_jni_env = child_jni_env_ext.get();
    CHECK_PTHREAD_CALL(pthread_attr_init, (&attr), "new thread");
    CHECK_PTHREAD_CALL(pthread_attr_setdetachstate, (&attr, PTHREAD_CREATE_DETACHED),
                       "PTHREAD_CREATE_DETACHED");
    CHECK_PTHREAD_CALL(pthread_attr_setstacksize, (&attr, stack_size), stack_size);
    pthread_create_result = pthread_create(&new_pthread,
                                           &attr,
                                           Thread::CreateCallback,
                                           child_thread);
    CHECK_PTHREAD_CALL(pthread_attr_destroy, (&attr), "new thread");

    if (pthread_create_result == 0) {
      // pthread_create started the new thread. The child is now responsible for managing the
      // JNIEnvExt we created.
      // Note: we can't check for tmp_jni_env == nullptr, as that would require synchronization
      //       between the threads.
      child_jni_env_ext.release();  // NOLINT pthreads API.
      return;
    }
  }

  // pthread_create 成功会在上面返回，跑到这里是因为其执行失败了
  {
    std::string msg(child_jni_env_ext.get() == nullptr ?
        StringPrintf("Could not allocate JNI Env: %s", error_msg.c_str()) :
        StringPrintf("pthread_create (%s stack) failed: %s",
                                 PrettySize(stack_size).c_str(), strerror(pthread_create_result)));
    ScopedObjectAccess soa(env);
    soa.Self()->ThrowOutOfMemoryError(msg.c_str());
  }
}
```

### 更多的例子

线程数超出了限制

```kotlin
W/libc: pthread_create failed: clone failed: Out of memory
W/art: Throwing OutOfMemoryError "pthread_create (1040KB stack) failed: Out of memory"
```

内存不足

```kotlin
W/libc: pthread_create failed: couldn't allocate 1073152-bytes mapped space: Out of memory
W/art: Throwing OutOfMemoryError with VmSize  4191668 kB "pthread_create (1040KB stack) failed: Try again"
```

## 参考

1. [Probe：Android线上OOM问题定位组件](https://tech.meituan.com/2019/11/14/crash-oom-probe-practice.html)
2. [不可思议的OOM](https://www.jianshu.com/p/e574f0ffdb42)
3. [OOM问题分析](https://github.com/CharonChui/AndroidNote/blob/master/AdavancedPart/OOM%E9%97%AE%E9%A2%98%E5%88%86%E6%9E%90.md)