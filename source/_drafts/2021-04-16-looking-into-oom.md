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

## 