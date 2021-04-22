---
title: LeakCanary 浅析
date: 2021-04-12 12:00:00 +0800
categories: [android, 内存]
tags: [LeakCanary, 内存优化，OOM]
---

## 检测内存泄漏

### 四种引用类型

| 引用类型                  | GC 时机                                                     |
|--------------------------|------------------------------------------------------------|
| 强引用                    | 平时写代码最常用的引用类型，对象只要被强引用就不会被 GC             |
| 软引用 `SoftReference`    | 只有当内存不足时才会被 GC                                      |
| 弱引用 `WeakReference`    | 会被正常 GC                                                  |
| 虚引用 `PhantomReference` | 会被正常 GC，因为 `get()` 总是返回 null，一般用来跟踪对象的生命周期 |

所有的引用类型都可以在构造时与一个 `ReferenceQueue` 关联，当引用的对象被 GC 后，这个 `Reference` 将被入队到关联的引用队列里

```java
public abstract class Reference<T> {
    /* -- Constructors -- */

    Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = queue;
    }
}
```

### 如何检测泄漏对象

`ObjectWatcher` 实现了 `LeakCanary` 的泄漏检测机制：监控 - 等待 - 检查

用 `WeakReference` + `ReferenceQueue` 监控对象的 GC 状态，并用 `watchedObjects` 持有它的弱引用，key 是 UUID

```kotlin
class ObjectWatcher constructor(
  private val clock: Clock,
  private val checkRetainedExecutor: Executor,
  private val isEnabled: () -> Boolean = { true }
) : ReachabilityWatcher {

  // 持有被监控对象的弱引用，以便后续的检查
  private val watchedObjects = mutableMapOf<String, KeyedWeakReference>()

  // 引用队列，被 GC 的对象的引用会进入此队列
  private val queue = ReferenceQueue<Any>()
}
```

将指定对象交由 `ObjectWatcher` 进行监控

```kotlin
// 监控检测对象
@Synchronized override fun expectWeaklyReachable(
  watchedObject: Any,
  description: String
) {
  if (!isEnabled()) {
    return
  }
  removeWeaklyReachableObjects()  // 从 watchedObjects 移除已被 GC 的对象

  // 弱引用监控对象并放入 watchedObjects
  val key = UUID.randomUUID().toString()
  val watchUptimeMillis = clock.uptimeMillis()
  val reference = KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
  SharkLog.d {
    "Watching " +
      (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
      (if (description.isNotEmpty()) " ($description)" else "") +
      " with key $key"
  }
  watchedObjects[key] = reference

  // 选机检查（默认 5s 后执行检查函数 moveToRetained）
  checkRetainedExecutor.execute {
    moveToRetained(key)
  }
}

// 出现在 queue 里的对象已被成功 GC 就不需要监控了
private fun removeWeaklyReachableObjects() {
  // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
  // reachable. This is before finalization or garbage collection has actually happened.
  var ref: KeyedWeakReference?
  do {
    ref = queue.poll() as KeyedWeakReference?
    if (ref != null) {
      watchedObjects.remove(ref.key)
    }
  } while (ref != null)
}
```

检查是否发生泄漏，默认情况是等待 5s，让 GC 线程有足够的机会去发现并回收这个对象，如果 5s 后仍然没有被 GC（没有出现在引用队列里），那么可以证明这个对象发生了内存泄漏，被强引用导致存活超过它的生命周期

```kotlin
// 上面说过检查操作将提交给 checkRetainedExecutor 执行
@Synchronized override fun expectWeaklyReachable(
  watchedObject: Any,
  description: String
) {
  // ...
  checkRetainedExecutor.execute {
    moveToRetained(key)
  }
}

// 而 checkRetainedExecutor 是通过构造函数传入的
class ObjectWatcher constructor(
  private val clock: Clock,
  private val checkRetainedExecutor: Executor,
  private val isEnabled: () -> Boolean = { true }
)

// 默认是等待 5s 并在主线程执行检查操作
object AppWatcher {
  private const val RETAINED_DELAY_NOT_SET = -1L
  @Volatile private var retainedDelayMillis = RETAINED_DELAY_NOT_SET

  val objectWatcher = ObjectWatcher(
    clock = { SystemClock.uptimeMillis() },
    checkRetainedExecutor = {
      check(isInstalled) {
        "AppWatcher not installed"
      }
      mainHandler.postDelayed(it, retainedDelayMillis)  // 在主线程执行检查
    },
    isEnabled = { true }
  )

  fun manualInstall(
    application: Application,
    retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),   // 默认等待 5s
    watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application)
  ) {
    checkMainThread()
    if (isInstalled) {
      throw IllegalStateException(
        "AppWatcher already installed, see exception cause for prior install call", installCause
      )
    }
    check(retainedDelayMillis >= 0) {
      "retainedDelayMillis $retainedDelayMillis must be at least 0 ms"
    }
    installCause = RuntimeException("manualInstall() first called here")
    this.retainedDelayMillis = retainedDelayMillis
    if (application.isDebuggableBuild) {
      LogcatSharkLog.install()
    }
    // Requires AppWatcher.objectWatcher to be set
    LeakCanaryDelegate.loadLeakCanary(application)

    watchersToInstall.forEach {
      it.install()
    }
  }
}

// 如果没有出现在引用队列里，说明此对象已发生泄漏，发出通知
@Synchronized private fun moveToRetained(key: String) {
  removeWeaklyReachableObjects()    // 出现在引用队列里说明对象已被 GC，可以从 watchedObjects 移除
  val retainedRef = watchedObjects[key]
  if (retainedRef != null) {
    retainedRef.retainedUptimeMillis = clock.uptimeMillis()
    onObjectRetainedListeners.forEach { it.onObjectRetained() }
  }
}
```

### 检测 `Activity` 泄漏

通过 `ActivityLifecycleCallbacks.onActivityDestroyed` 可以收集到 destoryed `Activity`，这些 `Activity` 已走完它的生命周期，应该被后续的 GC 回收掉

```kotlin
// 收集 destroyed Activity
class ActivityWatcher(
  private val application: Application,
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }
}
```

### 检测 `Fragment` 和 `View` 的泄漏

利用 `FragmentLifecycleCallbacks` 发现被 destroyed `Fragment` 和 `View`，然后用 `ObjectWatcher` 监控是否发生泄漏

```kotlin
internal class AndroidXFragmentDestroyWatcher(
  private val reachabilityWatcher: ReachabilityWatcher
) : (Activity) -> Unit {

  private val fragmentLifecycleCallbacks = object : FragmentManager.FragmentLifecycleCallbacks() {

    // 发现 View
    override fun onFragmentViewDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      val view = fragment.view
      if (view != null) {
        reachabilityWatcher.expectWeaklyReachable(
          view, "${fragment::class.java.name} received Fragment#onDestroyView() callback " +
          "(references to its views should be cleared to prevent leaks)"
        )
      }
    }

    // 发现 Fragment
    override fun onFragmentDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      reachabilityWatcher.expectWeaklyReachable(
        fragment, "${fragment::class.java.name} received Fragment#onDestroy() callback"
      )
    }
  }
}
```

### 检测 `ViewModel` 泄漏

`Fragment` 里的 `ViewModel` 则是在 `FragmentLifecycleCallbacks.onFragmentCreated` 时，注入一个 `ViewModel`，通过反射拿到 `ViewModelStore.mMap`，这里有所有的 `ViewModel`，在 `ViewModel.onCleared` 时把它们加入 `ObjectWatcher` 进行泄漏检查

```kotlin
internal class ViewModelClearedWatcher(
  storeOwner: ViewModelStoreOwner,
  private val reachabilityWatcher: ReachabilityWatcher
) : ViewModel() {

  private val viewModelMap: Map<String, ViewModel>?

  init {
    // We could call ViewModelStore#keys with a package spy in androidx.lifecycle instead,
    // however that was added in 2.1.0 and we support AndroidX first stable release. viewmodel-2.0.0
    // does not have ViewModelStore#keys. All versions currently have the mMap field.
    viewModelMap = try {
      val mMapField = ViewModelStore::class.java.getDeclaredField("mMap")
      mMapField.isAccessible = true
      @Suppress("UNCHECKED_CAST")
      mMapField[storeOwner.viewModelStore] as Map<String, ViewModel>
    } catch (ignored: Exception) {
      null
    }
  }

  override fun onCleared() {
    viewModelMap?.values?.forEach { viewModel ->
      reachabilityWatcher.expectWeaklyReachable(
        viewModel, "${viewModel::class.java.name} received ViewModel#onCleared() callback"
      )
    }
  }

  companion object {
    fun install(
      storeOwner: ViewModelStoreOwner,
      reachabilityWatcher: ReachabilityWatcher
    ) {
      val provider = ViewModelProvider(storeOwner, object : Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel?> create(modelClass: Class<T>): T =
          ViewModelClearedWatcher(storeOwner, reachabilityWatcher) as T
      })
      provider.get(ViewModelClearedWatcher::class.java)
    }
  }
}
```

### 检测更多类型的泄漏

对 `Service` 的检查就比较 hack 了，通过反射替换 `ActivityThread.mH.mCallback`，通过 `Message.waht == H.STOP_SERVICE` 定位到 `ActivityThread.handleStopService` 的调用时机，然后把这个被 stop 的 `Service` 记录下来；用动态代理实现 `IActivityManager` 并替换掉 `ActivityManager.IActivityManagerSinglteon.mInstance`，从而拦截方法 `serviceDoneExecuting`，此方法的调用表示 `Service` 生命周期已完结，可以把它交由 `ObjectWatcher` 进行监控

这给我启示，对于我们感兴趣的对象（需要警惕泄漏的对象，比如 `Bitmap`），都可以通过 `ObjectWatcher` 去检测泄漏问题

```kotlin
class MyService : Service {
  // ...
  override fun onDestroy() {
    super.onDestroy()
    AppWatcher.objectWatcher.watch(
      watchedObject = this,
      description = "MyService received Service#onDestroy() callback"
    )
  }
}
```

## Heap Dump

发现对象泄漏后触发 `OnObjectRetainedListener.onObjectRetained()`，最终调用 `Debug.dumpHprofData` 生成 hprof 文件

```kotlin
internal object InternalLeakCanary : (Application) -> Unit, OnObjectRetainedListener {

  override fun onObjectRetained() = scheduleRetainedObjectCheck()

  fun scheduleRetainedObjectCheck() {
    if (this::heapDumpTrigger.isInitialized) {
      heapDumpTrigger.scheduleRetainedObjectCheck()
    }
  }  
}

internal class HeapDumpTrigger {
  fun scheduleRetainedObjectCheck(
    delayMillis: Long = 0L
  ) {
    val checkCurrentlyScheduledAt = checkScheduledAt
    if (checkCurrentlyScheduledAt > 0) {
      return
    }
    checkScheduledAt = SystemClock.uptimeMillis() + delayMillis
    backgroundHandler.postDelayed({
      checkScheduledAt = 0
      checkRetainedObjects()
    }, delayMillis)
  }

  private fun checkRetainedObjects() {
    // ...
    dumpHeap(
      retainedReferenceCount = retainedReferenceCount,
      retry = true,
      reason = "$retainedReferenceCount retained objects, app is $visibility"
    )
  }

  private fun dumpHeap(
    retainedReferenceCount: Int,
    retry: Boolean,
    reason: String
  ) {
    saveResourceIdNamesToMemory()
    val heapDumpUptimeMillis = SystemClock.uptimeMillis()
    KeyedWeakReference.heapDumpUptimeMillis = heapDumpUptimeMillis
    when (val heapDumpResult = heapDumper.dumpHeap()) {
      // ...
    }
  }    
}

internal class AndroidHeapDumper {
  override fun dumpHeap(): DumpHeapResult {
    // ...
    val durationMillis = measureDurationMillis {
      Debug.dumpHprofData(heapDumpFile.absolutePath)
    }
    // ...  
  }
}
```

## 解析 hprof 文件

### hprof 文件格式 

![Java Hprof 格式](../../../../image/2021-04-12-leakcanary/java_hprof.png)

![Android Hprof 格式](../../../../image/2021-04-12-leakcanary/android_hprof.png)

hprof 是结构紧凑的二进制文件，整体上分为 `Header` 和 `Record` 数组两大部分

`Header` 总共 18 + 4 + 4 + 4 = 32 字节，包括：

1. 格式名和版本号：JAVA PROFILE 1.0.3（18 字节）
2. 标识符大小（4 字节）
3. 高位时间戳（4 字节）
4. 地位时间戳（4 字节）

![Hprof Header 结构](../../../../image/2021-04-12-leakcanary/hprof_header.png)

`Record` 数组记录了内存中的各种数据

1. TAG，`Record` 的类型（1 字节）
2. TIME，时间戳（4 字节）
3. LENGTH，`Record` BODY 的长度（4 字节）
4. BODY，不同的 `Record` 类型有不同的 BODY

![Hprof Record 结构](../../../../image/2021-04-12-leakcanary/hprof_record.png)

支持的 `TAG` 类型主要有：

* STRING_IN_UTF8             = 0x01
* LOAD_CLASS                 = 0x02
* STACK_FRAME                = 0x04
* STACK_TRACE                = 0x05
* **HEAP_DUMP**              = 0x0c
* **HEAP_DUMP_SEGMENT**      = 0x1c
  * ROOT_UNKNOWN             = 0xff
  * ROOT_JNI_GLOBAL          = 0x01
  * ROOT_JNI_LOCAL           = 0x02
  * ROOT_JAVA_FRAME          = 0x03
  * **CLASS_DUMP**           = 0x20
  * **INSTANCE_DUMP**        = 0x21
  * **OBJECT_ARRAY_DUMP**    = 0x22
  * **PRIMITIVE_ARRAY_DUMP** = 0x23
* HEAP_DUMP_END              = 0x2c

`CLASS_DUMP`、`INSTANCE_DUMP` 等重要结构可以看 [HPROF Agent](http://hg.openjdk.java.net/jdk8/jdk8/jdk/raw-file/tip/src/share/demo/jvmti/hprof/manual.html)


### 解析 Header

拿到 hprof 文件后，从 `HeapAnalyzerService.runAnalysis` 开始解析流程

`LeakCanary` 使用 `Shark` 解析 hprof 文件，首先解析出头部 `HprofHeader`

```kotlin
HeapAnalyzerService.runAnalysis
HeapAnalyzerService.onHandleIntentInForeground
HeapAnalyzerService.analyzeHeap
HeapAnalyzer.analyze

fun DualSourceProvider.openHeapGraph(
  proguardMapping: ProguardMapping? = null,
  indexedGcRootTypes: Set<HprofRecordTag> = HprofIndex.defaultIndexedGcRootTags()
): CloseableHeapGraph {
  val header = openStreamingSource().use { HprofHeader.parseHeaderOf(it) }
  // ...
}

// 解析出 HprofHeader
fun parseHeaderOf(source: BufferedSource): HprofHeader {
  require(!source.exhausted()) {
    throw IllegalArgumentException("Source has no available bytes")
  }

  // 开头是版本号 JAVA PROFILE 1.0.3，以 0 结尾
  val endOfVersionString = source.indexOf(0)
  val versionName = source.readUtf8(endOfVersionString)
  val version = supportedVersions[versionName]
  checkNotNull(version) {
    "Unsupported Hprof version [$versionName] not in supported list ${supportedVersions.keys}"
  }
  // Skip the 0 at the end of the version string.
  source.skip(1)

  // 然后是 ID 的长度
  val identifierByteSize = source.readInt()

  // 时间戳
  val heapDumpTimestamp = source.readLong()
  return HprofHeader(heapDumpTimestamp, version, identifierByteSize)
}
```

### 构造索引 `HprofIndex`

从 `openHeapGraph` 里构造完 `HprofHeader` 后，开始解析 `HprofIndex`

```kotlin
fun DualSourceProvider.openHeapGraph(
  proguardMapping: ProguardMapping? = null,
  indexedGcRootTypes: Set<HprofRecordTag> = HprofIndex.defaultIndexedGcRootTags()
): CloseableHeapGraph {
  val header = openStreamingSource().use { HprofHeader.parseHeaderOf(it) }
  val index = HprofIndex.indexRecordsOf(this, header, proguardMapping, indexedGcRootTypes)
  return index.openHeapGraph()
}

/**
 * Creates an in memory index of an hprof source provided by [hprofSourceProvider].
 */
fun indexRecordsOf(
  hprofSourceProvider: DualSourceProvider,
  hprofHeader: HprofHeader,
  proguardMapping: ProguardMapping? = null,
  indexedGcRootTags: Set<HprofRecordTag> = defaultIndexedGcRootTags()
): HprofIndex {
  val reader = StreamingHprofReader.readerFor(hprofSourceProvider, hprofHeader)
  val index = HprofInMemoryIndex.indexHprof(
    reader = reader,
    hprofHeader = hprofHeader,
    proguardMapping = proguardMapping,
    indexedGcRootTags = indexedGcRootTags
  )
  return HprofIndex(hprofSourceProvider, hprofHeader, index)
}
```

对象之所以会泄漏，是因为它被 GC ROOT 持有超过它的生命周期，所以分析 hprof 文件的首要目标是找出泄漏对象的 GC ROOT PATH；虽然 hprof 包含方方面面的信息，我们只关注需要的那几部分：`STRING_IN_UTF8`、`CLASS_DUMP`、`INSTANCE_DUMP`、`GC ROOT` 等等，其他的都不需要；而且 hprof 包含的数据非常多，全部加载到内存很容易发生 OOM

这个阶段的 `HprofInMemoryIndex` 主要包含以下信息

* `hprofStringCache` 字符串池，string id -> String，对应 TAG `STRING_IN_UTF8`，用来查找类名
* `classNames` 类名称池，class id -> string id，对应 TAG `LOAD_CLASS`，通过类名 `leakcanary.KeyedWeakReference` 找到泄漏对象
* `gcRoots` GC ROOT 对象 id 数组

```kotlin
internal class HprofInMemoryIndex private constructor(
  private val positionSize: Int,
  private val hprofStringCache: LongObjectScatterMap<String>,
  private val classNames: LongLongScatterMap,
  private val classIndex: SortedBytesMap,
  private val instanceIndex: SortedBytesMap,
  private val objectArrayIndex: SortedBytesMap,
  private val primitiveArrayIndex: SortedBytesMap,
  private val gcRoots: List<GcRoot>,
  private val proguardMapping: ProguardMapping?,
  private val bytesForClassSize: Int,
  private val bytesForInstanceSize: Int,
  private val bytesForObjectArraySize: Int,
  private val bytesForPrimitiveArraySize: Int,
  private val useForwardSlashClassPackageSeparator: Boolean,
  val classFieldsReader: ClassFieldsReader,
  private val classFieldsIndexSize: Int
)
```

参照 `Record` 的结构读取需要的内容

```kotlin
fun indexHprof(
  reader: StreamingHprofReader,
  hprofHeader: HprofHeader,
  proguardMapping: ProguardMapping?,
  indexedGcRootTags: Set<HprofRecordTag>
): HprofInMemoryIndex {

  // 首先过一遍 hprof，计算出 class，instance，object array 和 primitive array 的数量
  // First pass to count and correctly size arrays once and for all.
  var maxClassSize = 0L
  var maxInstanceSize = 0L
  var maxObjectArraySize = 0L
  var maxPrimitiveArraySize = 0L
  var classCount = 0
  var instanceCount = 0
  var objectArrayCount = 0
  var primitiveArrayCount = 0
  var classFieldsTotalBytes = 0
  val bytesRead = reader.readRecords(
    EnumSet.of(CLASS_DUMP, INSTANCE_DUMP, OBJECT_ARRAY_DUMP, PRIMITIVE_ARRAY_DUMP),
    OnHprofRecordTagListener { tag, _, reader ->
      val bytesReadStart = reader.bytesRead
      when (tag) {
        CLASS_DUMP -> {
          classCount++
          reader.skipClassDumpHeader()
          val bytesReadStaticFieldStart = reader.bytesRead
          reader.skipClassDumpStaticFields()
          reader.skipClassDumpFields()
          maxClassSize = max(maxClassSize, reader.bytesRead - bytesReadStart)
          classFieldsTotalBytes += (reader.bytesRead - bytesReadStaticFieldStart).toInt()
        }
        INSTANCE_DUMP -> {
          instanceCount++
          reader.skipInstanceDumpRecord()
          maxInstanceSize = max(maxInstanceSize, reader.bytesRead - bytesReadStart)
        }
        OBJECT_ARRAY_DUMP -> {
          objectArrayCount++
          reader.skipObjectArrayDumpRecord()
          maxObjectArraySize = max(maxObjectArraySize, reader.bytesRead - bytesReadStart)
        }
        PRIMITIVE_ARRAY_DUMP -> {
          primitiveArrayCount++
          reader.skipPrimitiveArrayDumpRecord()
          maxPrimitiveArraySize = max(maxPrimitiveArraySize, reader.bytesRead - bytesReadStart)
        }
      }
    })

  // 第二次才读取 string、class、instance 等结构信息  
  val bytesForClassSize = byteSizeForUnsigned(maxClassSize)
  val bytesForInstanceSize = byteSizeForUnsigned(maxInstanceSize)
  val bytesForObjectArraySize = byteSizeForUnsigned(maxObjectArraySize)
  val bytesForPrimitiveArraySize = byteSizeForUnsigned(maxPrimitiveArraySize)
  val indexBuilderListener = Builder(
    longIdentifiers = hprofHeader.identifierByteSize == 8,
    maxPosition = bytesRead,
    classCount = classCount,
    instanceCount = instanceCount,
    objectArrayCount = objectArrayCount,
    primitiveArrayCount = primitiveArrayCount,
    bytesForClassSize = bytesForClassSize,
    bytesForInstanceSize = bytesForInstanceSize,
    bytesForObjectArraySize = bytesForObjectArraySize,
    bytesForPrimitiveArraySize = bytesForPrimitiveArraySize,
    classFieldsTotalBytes = classFieldsTotalBytes
  )
  val recordTypes = EnumSet.of(
    STRING_IN_UTF8,
    LOAD_CLASS,
    CLASS_DUMP,
    INSTANCE_DUMP,
    OBJECT_ARRAY_DUMP,
    PRIMITIVE_ARRAY_DUMP
  ) + HprofRecordTag.rootTags.intersect(indexedGcRootTags)
  reader.readRecords(recordTypes, indexBuilderListener)
  return indexBuilderListener.buildIndex(proguardMapping, hprofHeader)
}

// 类似 SAX 地流式读取各个 Record 结构，然后回调给 listener 处理
fun readRecords(
  recordTags: Set<HprofRecordTag>,
  listener: OnHprofRecordTagListener
): Long {
  return sourceProvider.openStreamingSource().use { source ->
    val reader = HprofRecordReader(header, source)
    reader.skip(header.recordsPosition)
    // Local ref optimizations
    val intByteSize = INT.byteSize
    val identifierByteSize = reader.sizeOf(REFERENCE_HPROF_TYPE)
    while (!source.exhausted()) {
      // type of the record
      val tag = reader.readUnsignedByte()
      // number of microseconds since the time stamp in the header
      reader.skip(intByteSize)
      // number of bytes that follow and belong to this record
      val length = reader.readUnsignedInt()
      when (tag) {
        STRING_IN_UTF8.tag -> {
          if (STRING_IN_UTF8 in recordTags) {
            listener.onHprofRecord(STRING_IN_UTF8, length, reader)
          } else {
            reader.skip(length)
          }
        }
        LOAD_CLASS.tag -> {
          if (LOAD_CLASS in recordTags) {
            listener.onHprofRecord(LOAD_CLASS, length, reader)
          } else {
            reader.skip(length)
          }
        }
        STACK_FRAME.tag -> {
          if (STACK_FRAME in recordTags) {
            listener.onHprofRecord(STACK_FRAME, length, reader)
          } else {
            reader.skip(length)
          }
        }
        STACK_TRACE.tag -> {
          if (STACK_TRACE in recordTags) {
            listener.onHprofRecord(STACK_TRACE, length, reader)
          } else {
            reader.skip(length)
          }
        }
        HEAP_DUMP.tag, HEAP_DUMP_SEGMENT.tag -> {
          val heapDumpStart = reader.bytesRead
          var previousTag = 0
          var previousTagPosition = 0L
          while (reader.bytesRead - heapDumpStart < length) {
            val heapDumpTagPosition = reader.bytesRead
            val heapDumpTag = reader.readUnsignedByte()
            when (heapDumpTag) {
              ROOT_UNKNOWN.tag -> {
                if (ROOT_UNKNOWN in recordTags) {
                  listener.onHprofRecord(ROOT_UNKNOWN, -1, reader)
                } else {
                  reader.skip(identifierByteSize)
                }
              }
              ROOT_JNI_GLOBAL.tag -> {
                if (ROOT_JNI_GLOBAL in recordTags) {
                  listener.onHprofRecord(ROOT_JNI_GLOBAL, -1, reader)
                } else {
                  reader.skip(identifierByteSize + identifierByteSize)
                }
              }
              ROOT_JNI_LOCAL.tag -> {
                if (ROOT_JNI_LOCAL in recordTags) {
                  listener.onHprofRecord(ROOT_JNI_LOCAL, -1, reader)
                } else {
                  reader.skip(identifierByteSize + intByteSize + intByteSize)
                }
              }
              ROOT_JAVA_FRAME.tag -> {
                if (ROOT_JAVA_FRAME in recordTags) {
                  listener.onHprofRecord(ROOT_JAVA_FRAME, -1, reader)
                } else {
                  reader.skip(identifierByteSize + intByteSize + intByteSize)
                }
              }
              // ...
            }
            previousTag = heapDumpTag
            previousTagPosition = heapDumpTagPosition
          }
        }
        HEAP_DUMP_END.tag -> {
          if (HEAP_DUMP_END in recordTags) {
            listener.onHprofRecord(HEAP_DUMP_END, length, reader)
          }
        }
        else -> {
          reader.skip(length)
        }
      }
    }
    reader.bytesRead
  }
}
```

### 查找泄漏对象

泄漏对象被 `KeyedWeakReference` 弱引用并保存在 `ObjectWatcher.watchedObjects`，那么通过全限定类名 `leakcanary.KeyedWeakReference"`/ `com.squareup.leakcanary.KeyedWeakReference` 找到 class id，通过 class id 找到 instance Record，在 instance Record 里找到名为 `referent` 的成员变量值，这个值就是泄漏对象的 instance id，最终会产生一个泄漏对象 instance id 数组 `leakingObjectIds`

```kotlin
fun HeapAnalyzer.analyze(...): HeapAnalysis {
  // ...
  val sourceProvider = ConstantMemoryMetricsDualSourceProvider(FileSourceProvider(heapDumpFile))
  sourceProvider.openHeapGraph(proguardMapping).use { graph ->
    val helpers =
      FindLeakInput(graph, referenceMatchers, computeRetainedHeapSize, objectInspectors)
    val result = helpers.analyzeGraph(
      metadataExtractor, leakingObjectFinder, heapDumpFile, analysisStartNanoTime
    )
    // ...
  }
}

private fun FindLeakInput.analyzeGraph(
  metadataExtractor: MetadataExtractor,
  leakingObjectFinder: LeakingObjectFinder,
  heapDumpFile: File,
  analysisStartNanoTime: Long
): HeapAnalysisSuccess {
  // ...
  val leakingObjectIds = leakingObjectFinder.findLeakingObjectIds(graph)
  val (applicationLeaks, libraryLeaks, unreachableObjects) = findLeaks(leakingObjectIds)
  // ...
}

object KeyedWeakReferenceFinder : LeakingObjectFinder {

  override fun findLeakingObjectIds(graph: HeapGraph): Set<Long> =
    findKeyedWeakReferences(graph)
      .filter { it.hasReferent && it.isRetained }
      .map { it.referent.value }
      .toSet()

  internal fun findKeyedWeakReferences(graph: HeapGraph): List<KeyedWeakReferenceMirror> {
    return graph.context.getOrPut(KEYED_WEAK_REFERENCE.name) {
      val keyedWeakReferenceClass = graph.findClassByName("leakcanary.KeyedWeakReference")

      val keyedWeakReferenceClassId = keyedWeakReferenceClass?.objectId ?: 0
      val legacyKeyedWeakReferenceClassId =
        graph.findClassByName("com.squareup.leakcanary.KeyedWeakReference")?.objectId ?: 0

      val heapDumpUptimeMillis = heapDumpUptimeMillis(graph)

      val addedToContext: List<KeyedWeakReferenceMirror> = graph.instances
        .filter { instance ->
          instance.instanceClassId == keyedWeakReferenceClassId || instance.instanceClassId == legacyKeyedWeakReferenceClassId
        }
        .map {
          KeyedWeakReferenceMirror.fromInstance(
            it, heapDumpUptimeMillis
          )
        }
        .toList()
      graph.context[KEYED_WEAK_REFERENCE.name] = addedToContext
      addedToContext
    }
  }
}
```

从逻辑上看引用关系是个图，图中的节点有个指向父节点的指针 `Node.parent`，而 GC ROOT 就是图中的根节点，它们没有 `parent`，GC ROOT 包括以下几类：

1. `ROOT UNKNOWN`
2. `ROOT JNI GLOBAL`
3. `ROOT JNI LOCAL`
4. `ROOT JAVA FRAME`
5. `ROOT NATIVE STACK`
6. `ROOT STICKY CLASS`
7. `ROOT THREAD BLOCK`
8. `ROOT MONITOR USED`
9. `ROOT THREAD OBJECT`

GC ROOT 的成员变量作为子节点，`parent` 指向 GC ROOT，成员变量还有成员变量作为子节点，这样就形成了一个很大的图

我们使用 `Stack` 来遍历图里的节点，首先把 GC ROOT 都入栈，然后依次出栈执行：找到非空的成员变量并加入栈中，如果 instance id == leakingObjectId 则记录起来，直到栈空或者已找完所有的泄漏对象；这样就可以沿着 `Node.parent` 一直往上走到 GC ROOT，这样泄漏对象的 GC ROOT PATH 就出来了

```kotlin
private fun FindLeakInput.findLeaks(leakingObjectIds: Set<Long>): LeaksAndUnreachableObjects {
  val pathFinder = PathFinder(graph, listener, referenceMatchers)
  val pathFindingResults =
    pathFinder.findPathsFromGcRoots(leakingObjectIds, computeRetainedHeapSize)
  // ...
}

fun findPathsFromGcRoots(
  leakingObjectIds: Set<Long>,
  computeRetainedHeapSize: Boolean
): PathFindingResults {
  // ...
  val state = State(
    leakingObjectIds = leakingObjectIds.toLongScatterSet(),
    sizeOfObjectInstances = sizeOfObjectInstances,
    computeRetainedHeapSize = computeRetainedHeapSize,
    javaLangObjectId = javaLangObjectId,
    estimatedVisitedObjects = estimatedVisitedObjects
  )
  return state.findPathsFromGcRoots()
}

private fun State.findPathsFromGcRoots(): PathFindingResults {
  // 首先将 GC ROOT 入队
  enqueueGcRoots()

  val shortestPathsToLeakingObjects = mutableListOf<ReferencePathNode>()
  visitingQueue@ while (queuesNotEmpty) {
    val node = poll()
    if (leakingObjectIds.contains(node.objectId)) {
      shortestPathsToLeakingObjects.add(node)
      // Found all refs, stop searching (unless computing retained size)
      if (shortestPathsToLeakingObjects.size == leakingObjectIds.size()) {
        if (computeRetainedHeapSize) {
          listener.onAnalysisProgress(FINDING_DOMINATORS)
        } else {
          break@visitingQueue
        }
      }
    }

    // 找到子节点（类的静态成员变量、实例的成员变量等等）
    when (val heapObject = graph.findObjectById(node.objectId)) {
      is HeapClass -> visitClassRecord(heapObject, node)
      is HeapInstance -> visitInstance(heapObject, node)
      is HeapObjectArray -> visitObjectArray(heapObject, node)
    }
  }
  return PathFindingResults(
    shortestPathsToLeakingObjects,
    if (visitTracker is Dominated) visitTracker.dominatorTree else null
  )
}
```

上面找到了所有泄漏对象的 GC ROOT PATH，但可能出现重复，这里利用前缀树删除重复路径；前缀树节点 `Node` 用 `Map<Long, Node>` 表示它引用了某个对象，叶子节点就是泄漏对象，最后广度优先遍历前缀树，将每个叶子节点及其路径记下来

```kotlin
private fun FindLeakInput.findLeaks(leakingObjectIds: Set<Long>): LeaksAndUnreachableObjects {
  val pathFinder = PathFinder(graph, listener, referenceMatchers)
  val pathFindingResults =
    pathFinder.findPathsFromGcRoots(leakingObjectIds, computeRetainedHeapSize)
  val unreachableObjects = findUnreachableObjects(pathFindingResults, leakingObjectIds)
  val shortestPaths =
    deduplicateShortestPaths(pathFindingResults.pathsToLeakingObjects)
  // ...
}

// 利用前缀树删除重复路径
private fun deduplicateShortestPaths(
   inputPathResults: List<ReferencePathNode>
 ): List<ShortestPath> {
   val rootTrieNode = ParentNode(0)

   inputPathResults.forEach { pathNode ->
     // Go through the linked list of nodes and build the reverse list of instances from
     // root to leaking.
     val path = mutableListOf<Long>()
     var leakNode: ReferencePathNode = pathNode
     while (leakNode is ChildNode) {
       path.add(0, leakNode.objectId)
       leakNode = leakNode.parent
     }
     path.add(0, leakNode.objectId)
     updateTrie(pathNode, path, 0, rootTrieNode)
   }

   val outputPathResults = mutableListOf<ReferencePathNode>()
   findResultsInTrie(rootTrieNode, outputPathResults)

   if (outputPathResults.size != inputPathResults.size) {
     SharkLog.d {
       "Found ${inputPathResults.size} paths to retained objects," +
         " down to ${outputPathResults.size} after removing duplicated paths"
     }
   } else {
     SharkLog.d { "Found ${outputPathResults.size} paths to retained objects" }
   }

   return outputPathResults.map { retainedObjectNode ->
     val shortestChildPath = mutableListOf<ChildNode>()
     var node = retainedObjectNode
     while (node is ChildNode) {
       shortestChildPath.add(0, node)
       node = node.parent
     }
     val rootNode = node as RootNode
     ShortestPath(rootNode, shortestChildPath)
   }
 }

// 将一个 GC ROOT PATH 添加到前缀树
private fun updateTrie(
  pathNode: ReferencePathNode,
  path: List<Long>,
  pathIndex: Int,
  parentNode: ParentNode
) {
  val objectId = path[pathIndex]
  if (pathIndex == path.lastIndex) {
    parentNode.children[objectId] = LeafNode(objectId, pathNode)
  } else {
    val childNode = parentNode.children[objectId] ?: {
      val newChildNode = ParentNode(objectId)
      parentNode.children[objectId] = newChildNode
      newChildNode
    }()
    if (childNode is ParentNode) {
      updateTrie(pathNode, path, pathIndex + 1, childNode)
    }
  }
} 
```

到此 GC ROOT PATH 已找到，最后再封装为 `LeakTrace`

```kotlin
private fun FindLeakInput.findLeaks(leakingObjectIds: Set<Long>): LeaksAndUnreachableObjects {
  val pathFinder = PathFinder(graph, listener, referenceMatchers)
  val pathFindingResults =
    pathFinder.findPathsFromGcRoots(leakingObjectIds, computeRetainedHeapSize)
  val unreachableObjects = findUnreachableObjects(pathFindingResults, leakingObjectIds)
  val shortestPaths =
    deduplicateShortestPaths(pathFindingResults.pathsToLeakingObjects)
  val inspectedObjectsByPath = inspectObjects(shortestPaths)
  val retainedSizes =
    if (pathFindingResults.dominatorTree != null) {
      computeRetainedSizes(inspectedObjectsByPath, pathFindingResults.dominatorTree)
    } else {
      null
    }
  val (applicationLeaks, libraryLeaks) = buildLeakTraces(
    shortestPaths, inspectedObjectsByPath, retainedSizes
  )
  return LeaksAndUnreachableObjects(applicationLeaks, libraryLeaks, unreachableObjects)
}

private fun FindLeakInput.buildLeakTraces(
  shortestPaths: List<ShortestPath>,
  inspectedObjectsByPath: List<List<InspectedObject>>,
  retainedSizes: Map<Long, Pair<Int, Int>>?
): Pair<List<ApplicationLeak>, List<LibraryLeak>> {
  listener.onAnalysisProgress(BUILDING_LEAK_TRACES)
  val applicationLeaksMap = mutableMapOf<String, MutableList<LeakTrace>>()
  val libraryLeaksMap =
    mutableMapOf<String, Pair<LibraryLeakReferenceMatcher, MutableList<LeakTrace>>>()
  shortestPaths.forEachIndexed { pathIndex, shortestPath ->
    val inspectedObjects = inspectedObjectsByPath[pathIndex]
    val leakTraceObjects = buildLeakTraceObjects(inspectedObjects, retainedSizes)
    val referencePath = buildReferencePath(shortestPath.childPath, leakTraceObjects)
    val leakTrace = LeakTrace(
      gcRootType = GcRootType.fromGcRoot(shortestPath.root.gcRoot), // 第一个元素是 GC ROOT
      referencePath = referencePath,                                // GC ROOT PATH
      leakingObject = leakTraceObjects.last()                       // leaking object
    )
    val firstLibraryLeakNode = if (shortestPath.root is LibraryLeakNode) {
      shortestPath.root
    } else {
      shortestPath.childPath.firstOrNull { it is LibraryLeakNode } as LibraryLeakNode?
    }
    if (firstLibraryLeakNode != null) {
      val matcher = firstLibraryLeakNode.matcher
      val signature: String = matcher.pattern.toString()
        .createSHA1Hash()
      libraryLeaksMap.getOrPut(signature) { matcher to mutableListOf() }
        .second += leakTrace
    } else {
      applicationLeaksMap.getOrPut(leakTrace.signature) { mutableListOf() } += leakTrace
    }
  }
  val applicationLeaks = applicationLeaksMap.map { (_, leakTraces) ->
    ApplicationLeak(leakTraces)
  }
  val libraryLeaks = libraryLeaksMap.map { (_, pair) ->
    val (matcher, leakTraces) = pair
    LibraryLeak(leakTraces, matcher.pattern, matcher.description)
  }
  return applicationLeaks to libraryLeaks
}
```

## 思考

### 为什么不需要手动初始化？

`LeakCanary` 把初始化代码放在 `ContentProvider.onCreate()` 里（具体是 `AppWatcherInstaller`），而 `ContentProvider.onCreate()` 会早于 `Application.onCreate` 被调用

```java
// ContentProvider.onCreate 会早于 Application.onCreate
private void ActivityThread.handleBindApplication(AppBindData data) {
    // ...
    // If the app is being launched for full backup or restore, bring it up in
    // a restricted environment with the base application class.
    app = data.info.makeApplication(data.restrictedBackupMode, null);
    // Propagate autofill compat state
    app.setAutofillOptions(data.autofillOptions);
    // Propagate Content Capture options
    app.setContentCaptureOptions(data.contentCaptureOptions);
    mInitialApplication = app;
    // don't bring up providers in restricted mode; they may depend on the
    // app's custom Application class
    if (!data.restrictedBackupMode) {
        if (!ArrayUtils.isEmpty(data.providers)) {
            installContentProviders(app, data.providers);
        }
    }
    // Do this after providers, since instrumentation tests generally start their
    // test thread at this point, and we don't want that racing.
    try {
        mInstrumentation.onCreate(data.instrumentationArgs);
    }
    catch (Exception e) {
        throw new RuntimeException(
            "Exception thrown in onCreate() of "
            + data.instrumentationName + ": " + e.toString(), e);
    }
    try {
        mInstrumentation.callApplicationOnCreate(app);
    } catch (Exception e) {
        if (!mInstrumentation.onException(app, e)) {
            throw new RuntimeException(
              "Unable to create application " + app.getClass().getName()
              + ": " + e.toString(), e);
        }
    }
    // ...
}

private void installContentProviders(
        Context context, List<ProviderInfo> providers) {
    final ArrayList<ContentProviderHolder> results = new ArrayList<>();
    for (ProviderInfo cpi : providers) {
        ContentProviderHolder cph = installProvider(context, null, cpi,
                false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
        if (cph != null) {
            cph.noReleaseNeeded = true;
            results.add(cph);
        }
    }
}

// 实例化 ContentProvider 并调用生命周期函数 onCreate
private ContentProviderHolder ActivityThread.installProvider(Context context,
        ContentProviderHolder holder, ProviderInfo info,
        boolean noisy, boolean noReleaseNeeded, boolean stable) {
    // ...
    final java.lang.ClassLoader cl = c.getClassLoader();
    LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
    if (packageInfo == null) {
        // System startup case.
        packageInfo = getSystemContext().mPackageInfo;
    }
    localProvider = packageInfo.getAppFactory()
            .instantiateProvider(cl, info.name);
    provider = localProvider.getIContentProvider();
    if (provider == null) {
        Slog.e(TAG, "Failed to instantiate class " +
              info.name + " from sourceDir " +
              info.applicationInfo.sourceDir);
        return null;
    }
    if (DEBUG_PROVIDER) Slog.v(
        TAG, "Instantiating local provider " + info.name);
    // XXX Need to create the correct context for this provider.
    localProvider.attachInfo(c, info);
    // ...
}

public void ContentProvider.attachInfo(Context context, ProviderInfo info) {
    attachInfo(context, info, false);
}

private void ContentProvider.attachInfo(Context context, ProviderInfo info, boolean testing) {
    // ...
    ContentProvider.this.onCreate();
}
```

### 有什么缺点

1. `LeakCanary` 是在 app process 内 heap dump 的，期间进程内的其他线程会被挂起直到 heap dump 完成，这会导致 app 无响应，生产环境下是不可接受的
2. hprof 文件往往达到 400M / 500M 这个量级，客户端存储是个问题
3. hprof 文件要回传给服务器分析，但是文件太大网络消耗也有很大问题
4. 如果是在客户端分析 hprof 文件，由于文件太大导致分析进程在很多情况下自己也 OOM 了

## 参考

1. [Hprof文件解析](https://leo-wxy.github.io/2020/12/14/Hprof%E6%96%87%E4%BB%B6%E8%A7%A3%E6%9E%90/)
2. [HPROF Agent](http://hg.openjdk.java.net/jdk8/jdk8/jdk/raw-file/tip/src/share/demo/jvmti/hprof/manual.html)