
# 主线程进行 IO 操作导致的 ANR

在 layout xml 里配置了 `fontFamily`，`LayoutInflator` 会在解析过程中调用 `ResourcesCompat.getFont` 去加载字体文件从而导致主线程 ANR，这个案例其实比较简单，观察主线程 main 的调用栈时就能发现问题

观察主线程的调用栈，发现有个全局的字体缓存 `TypefaceCompat.sTypefaceCompatImpl`，`ResourcesCompat.getFont` 首先会从缓存里找字体，如果没有才会执行加载字体的逻辑 `TypefaceCompat.createFromResourcesFontFile`，所以解决办法就是在子线程预先加载需要用到的字体，刚好有个方法 `ResourcesCompat.getCachedFont`

```xml
<TextView
    android:id="@+id/instruction_title"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@drawable/bg_white_bottom_half_corner_12"
    android:fontFamily="@font/oppo_sans_b"
    android:gravity="center"
    android:includeFontPadding="false"
    android:paddingTop="@dimen/px_y_28"
    android:paddingBottom="@dimen/px_y_27"
    android:text="@string/management_instruction"
    android:textColor="@color/color_2c"
    android:textSize="@dimen/px_x_32"
    app:layout_constraintTop_toTopOf="parent" />
```

```text
ANR in com.viomi.fridge.vertical (com.viomi.fridge.vertical/.home.activity.HomeActivity)0
PID: 5853
Reason: Input dispatching timed out (Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.)
Load: 13.08 / 10.3 / 7.53
CPU usage from 0ms to 6237ms later (2021-08-25 10:33:51.670 to 2021-08-25 10:33:57.907) with 99% awake:
  263% 5853/com.viomi.fridge.vertical: 236% user + 27% kernel / faults: 46832 minor
  40% 409/system_server: 29% user + 11% kernel / faults: 1683 minor 1 major
  6.5% 2991/adbd: 1.6% user + 4.9% kernel / faults: 1251 minor
  4.6% 4767/com.android.systemui: 3.3% user + 1.2% kernel / faults: 235 minor 1 major
  4% 234/surfaceflinger: 1.2% user + 2.7% kernel / faults: 653 minor
  3.6% 990/com.tencent.qqpinyin: 2.8% user + 0.8% kernel / faults: 178 minor
  3.6% 5222/com.android.phone: 3.2% user + 0.4% kernel / faults: 804 minor 2 major
  3.3% 185/logd: 1.1% user + 2.2% kernel / faults: 25 minor
  2.7% 259/media.codec: 0.6% user + 2% kernel / faults: 1055 minor
  2.4% 225/android.hardware.graphics.allocator@2.0-service: 0.1% user + 2.2% kernel / faults: 34 minor
  2.2% 3020/logcat: 0.9% user + 1.2% kernel / faults: 92 minor
  2% 142/mmcqd/0: 0% user + 2% kernel
  2% 226/android.hardware.graphics.composer@2.1-service: 1.1% user + 0.9% kernel / faults: 234 minor
  0% 5208/com.cghs.stresstest: 0% user + 0% kernel / faults: 798 minor
  1.2% 1469/ksdioirqd/mmc1: 0% user + 1.2% kernel
  1.1% 5548/com.android.commands.monkey: 0.8% user + 0.3% kernel / faults: 183 minor
  0.8% 1347/com.viomi.fridge.vertical:miot: 0.6% user + 0.1% kernel / faults: 440 minor
  0.6% 7/rcu_preempt: 0% user + 0.6% kernel
  0.6% 1428/dhd_rxf: 0% user + 0.6% kernel
  0.4% 186/servicemanager: 0.3% user + 0.1% kernel / faults: 64 minor
  0.4% 211/netd: 0.1% user + 0.3% kernel / faults: 170 minor
  0% 274/tombstoned: 0% user + 0% kernel / faults: 97 minor
  0.3% 183/jbd2/mmcblk0p16: 0% user + 0.3% kernel
  0.3% 1426/dhd_watchdog_th: 0% user + 0.3% kernel
  0.3% 1427/dhd_dpc: 0% user + 0.3% kernel
  0.3% 5514/kworker/u9:1: 0% user + 0.3% kernel
  0.3% 5540/kworker/u8:1: 0% user + 0.3% kernel
  0.1% 1//init: 0% user + 0.1% kernel
  0.1% 8/rcu_sched: 0% user + 0.1% kernel
  0.1% 19/ksoftirqd/2: 0% user + 0.1% kernel
  0.1% 30/kconsole: 0% user + 0.1% kernel
  0.1% 228/android.hardware.power@1.0-service: 0.1% user + 0% kernel
  0.1% 239/audioserver: 0% user + 0.1% kernel / faults: 43 minor
  0% 240/cameraserver: 0% user + 0% kernel / faults: 48 minor
  0.1% 251/mediaserver: 0% user + 0.1% kernel / faults: 36 minor
  0.1% 254/wificond: 0.1% user + 0% kernel / faults: 60 minor
  0.1% 353/kworker/0:1H: 0% user + 0.1% kernel
  0.1% 402/kworker/2:1H: 0% user + 0.1% kernel
  0.1% 453/kworker/u9:2: 0% user + 0.1% kernel
  0.1% 2216/kworker/u8:5: 0% user + 0.1% kernel
  0.1% 4166/kworker/u9:3: 0% user + 0.1% kernel
  0.1% 5534/kworker/0:1: 0% user + 0.1% kernel
 +0% 6075/kbase_event: 0% user + 0% kernel
95% TOTAL: 76% user + 18% kernel + 0% iowait + 0.5% softirq
CPU usage from 106ms to 757ms later (2021-08-25 10:33:51.775 to 2021-08-25 10:33:52.426):
  314% 5853/com.viomi.fridge.vertical: 290% user + 24% kernel / faults: 3246 minor
    86% 5853/fridge.vertical: 79% user + 6.9% kernel
    83% 5979/pool-20-thread-: 83% user + 0% kernel
    55% 5930/RxCachedThreadS: 55% user + 0% kernel
    31% 6045/RxCachedThreadS: 24% user + 6.9% kernel
    13% 6038/RxCachedThreadS: 13% user + 0% kernel
    10% 5858/Jit thread pool: 10% user + 0% kernel
    6.9% 6003/ot.viomi.com.cn: 6.9% user + 0% kernel
    3.4% 5929/RxCachedThreadS: 3.4% user + 0% kernel
    3.4% 6040/RxCachedThreadS: 3.4% user + 0% kernel
   +0% 6063/Binder:5853_5: 0% user + 0% kernel
  75% 409/system_server: 49% user + 25% kernel / faults: 532 minor
    45% 423/ActivityManager: 21% user + 23% kernel
    25% 418/HeapTaskDaemon: 25% user + 0% kernel
    2.1% 605/WifiService: 0% user + 2.1% kernel
  8.8% 2991/adbd: 2.9% user + 5.8% kernel / faults: 66 minor
    2.9% 2991/adbd: 2.9% user + 0% kernel
  3.6% 185/logd: 1.8% user + 1.8% kernel / faults: 3 minor
    1.8% 191/logd.writer: 1.8% user + 0% kernel
    1.8% 3022/logd.reader.per: 0% user + 1.8% kernel
  1.5% 19/ksoftirqd/2: 0% user + 1.5% kernel
98% TOTAL: 77% user + 19% kernel + 0.3% iowait + 0.7% softirq

procrank:
// Exception from procrank:
java.io.IOException: Cannot run program "procrank": error=13, Permission denied
anr traces:

----- pid 5853 at 2021-08-25 10:33:52 -----
Cmd line: com.viomi.fridge.vertical
Build fingerprint: 'rockchip/rk3326_32bit/rk3326_32bit:8.1.0/OPM8.181005.003/142729:user/release-keys'
ABI: 'arm'
Build type: optimized
Zygote loaded classes=5030 post zygote classes=4128
Intern table: 71548 strong; 146 weak
JNI: CheckJNI is off; globals=838 (plus 38 weak)
Libraries: /system/lib/libandroid.so /system/lib/libcompiler_rt.so /system/lib/libjavacrypto.so /system/lib/libjnigraphics.so /system/lib/libmedia_jni.so /system/lib/librs_jni.so /system/lib/libsoundpool.so /system/lib/libwebviewchromium_loader.so /system/priv-app/viomi/viomi.apk!/lib/armeabi-v7a/libBugly.so /system/priv-app/viomi/viomi.apk!/lib/armeabi-v7a/libImSDK.so /system/priv-app/viomi/viomi.apk!/lib/armeabi-v7a/libliteavsdk.so /system/priv-app/viomi/viomi.apk!/lib/armeabi-v7a/liblogan.so /system/priv-app/viomi/viomi.apk!/lib/armeabi-v7a/libobjectbox-jni.so /system/priv-app/viomi/viomi.apk!/lib/armeabi-v7a/libtnet-3.1.14.so /system/priv-app/viomi/viomi.apk!/lib/armeabi-v7a/libtraeimp-rtmp.so /system/priv-app/viomi/viomi.apk!/lib/armeabi-v7a/libtxffmpeg.so /system/priv-app/viomi/viomi.apk!/lib/armeabi-v7a/libumeng-spy.so libjavacore.so libopenjdk.so (19)
Heap: 19% free, 32MB/40MB; 325141 objects
Dumping cumulative Gc timings
Start Dumping histograms for 5 iterations for concurrent copying
ProcessMarkStack:       Sum: 1.031s 99% C.I. 15.014ms-353.311ms Avg: 206.303ms Max: 353.311ms
VisitConcurrentRoots:   Sum: 135.988ms 99% C.I. 10.740ms-59.207ms Avg: 27.197ms Max: 59.207ms
ScanImmuneSpaces:       Sum: 74.474ms 99% C.I. 8.787ms-23.286ms Avg: 14.894ms Max: 23.286ms
ClearFromSpace: Sum: 57.288ms 99% C.I. 0.312ms-29.150ms Avg: 11.457ms Max: 29.228ms
FlipOtherThreads:       Sum: 54.005ms 99% C.I. 0.331ms-43.800ms Avg: 10.801ms Max: 44.150ms
ReclaimPhase:   Sum: 48.588ms 99% C.I. 0.023ms-45.300ms Avg: 9.717ms Max: 46.352ms
MarkingPhase:   Sum: 42.677ms 99% C.I. 0.034ms-32.489ms Avg: 8.535ms Max: 33.154ms
SweepLargeObjects:      Sum: 42.497ms 99% C.I. 0.567ms-34.820ms Avg: 8.499ms Max: 35.332ms
EnqueueFinalizerReferences:     Sum: 35.719ms 99% C.I. 0.199ms-25.960ms Avg: 7.143ms Max: 26.295ms
VisitNonThreadRoots:    Sum: 31.779ms 99% C.I. 0.131ms-20.060ms Avg: 6.355ms Max: 20.360ms
SweepSystemWeaks:       Sum: 26.217ms 99% C.I. 0.175ms-24.970ms Avg: 5.243ms Max: 25.340ms
ForwardSoftReferences:  Sum: 13.387ms 99% C.I. 0.012ms-6.301ms Avg: 2.677ms Max: 6.301ms
InitializePhase:        Sum: 9.071ms 99% C.I. 0.254ms-6.005ms Avg: 1.814ms Max: 6.038ms
EmptyRBMarkBitStack:    Sum: 8.452ms 99% C.I. 0.046ms-4.346ms Avg: 1.690ms Max: 4.390ms
GrayAllDirtyImmuneObjects:      Sum: 6.552ms 99% C.I. 0.250ms-3.274ms Avg: 1.310ms Max: 3.274ms
FlipThreadRoots:        Sum: 6.229ms 99% C.I. 0.006ms-5.382ms Avg: 1.245ms Max: 5.466ms
ClearRegionSpaceCards:  Sum: 3.586ms 99% C.I. 32us-1858us Avg: 717.200us Max: 1858us
ThreadListFlip: Sum: 3.573ms 99% C.I. 77us-3026.250us Avg: 714.600us Max: 3093us
MarkStackAsLive:        Sum: 2.017ms 99% C.I. 74us-1602us Avg: 403.400us Max: 1602us
MarkZygoteLargeObjects: Sum: 1.718ms 99% C.I. 94us-1223us Avg: 343.600us Max: 1223us
SwapBitmaps:    Sum: 1.296ms 99% C.I. 20us-1209us Avg: 259.200us Max: 1209us
ProcessReferences:      Sum: 729us 99% C.I. 5us-289us Avg: 72.900us Max: 289us
RecordFree:     Sum: 600us 99% C.I. 97us-172us Avg: 120us Max: 172us
ResumeRunnableThreads:  Sum: 447us 99% C.I. 45us-228us Avg: 89.400us Max: 228us
(Paused)GrayAllNewlyDirtyImmuneObjects: Sum: 295us 99% C.I. 53us-65us Avg: 59us Max: 65us
ResumeOtherThreads:     Sum: 183us 99% C.I. 5us-67us Avg: 36.600us Max: 67us
(Paused)ClearCards:     Sum: 130us 99% C.I. 0.250us-25us Avg: 1.625us Max: 25us
SweepAllocSpace:        Sum: 122us 99% C.I. 18us-34us Avg: 24.400us Max: 34us
(Paused)SetFromSpace:   Sum: 70us 99% C.I. 4us-32us Avg: 14us Max: 32us
Sweep:  Sum: 52us 99% C.I. 10us-11us Avg: 10.400us Max: 11us
(Paused)FlipCallback:   Sum: 38us 99% C.I. 6us-8us Avg: 7.600us Max: 8us
UnBindBitmaps:  Sum: 16us 99% C.I. 2us-7us Avg: 3.200us Max: 7us
Done Dumping histograms
concurrent copying paused:      Sum: 4.033ms 99% C.I. 177us-3195us Avg: 806.600us Max: 3195us
concurrent copying total time: 1.639s mean time: 327.862ms
concurrent copying freed: 197528 objects with total size 15MB
concurrent copying throughput: 120517/s / 9MB/s
Cumulative bytes moved 5720512
Cumulative objects moved 110207
Total time spent in GC: 1.639s
Mean GC size throughput: 8MB/s
Mean GC object throughput: 120433 objects/s
Total number of allocations 522568
Total bytes allocated 45MB
Total bytes freed 13MB
Free memory 8MB
Free memory until GC 8MB
Free memory until OOME 223MB
Total memory 40MB
Max memory 256MB
Zygote space size 636KB
Total mutator paused time: 4.033ms
Total time waiting for GC to complete: 22.748us
Total GC count: 5
Total GC time: 1.639s
Total blocking GC count: 0
Total blocking GC time: 0
Registered native bytes allocated: 21626500
/data/dalvik-cache/arm/system@priv-app@viomi@viomi.apk@classes.dex: quicken
Current JIT code cache size: 8KB
Current JIT data cache size: 5KB
Current JIT capacity: 256KB
Current number of JIT code cache entries: 4
Total number of JIT compilations: 40
Total number of JIT compilations for on stack replacement: 1
Total number of JIT code cache collections: 3
Memory used for stack maps: Avg: 743B Max: 5KB Min: 36B
Memory used for compiled code: Avg: 1561B Max: 10KB Min: 74B
Memory used for profiling info: Avg: 170B Max: 2104B Min: 16B
Start Dumping histograms for 42 iterations for JIT timings
Compiling:      Sum: 4.970s 99% C.I. 1.271ms-852.928ms Avg: 127.450ms Max: 868.921ms
Code cache collection:  Sum: 170.797ms 99% C.I. 45.038ms-77.168ms Avg: 56.932ms Max: 77.357ms
TrimMaps:       Sum: 102.737ms 99% C.I. 0.041ms-26.342ms Avg: 2.634ms Max: 27.049ms
Done Dumping histograms
Memory used for compilation: Avg: 290KB Max: 1483KB Min: 32KB
ProfileSaver total_bytes_written=0
ProfileSaver total_number_of_writes=0
ProfileSaver total_number_of_code_cache_queries=0
ProfileSaver total_number_of_skipped_writes=0
ProfileSaver total_number_of_failed_writes=0
ProfileSaver total_ms_of_sleep=5000
ProfileSaver total_ms_of_work=0
ProfileSaver max_number_profile_entries_cached=0
ProfileSaver total_number_of_hot_spikes=0
ProfileSaver total_number_of_wake_ups=0
Number of JIT inline cache deoptimizations: 1

suspend all histogram:  Sum: 4.151ms 99% C.I. 10us-2897.279us Avg: 319.307us Max: 3039us
DALVIK THREADS (95):
"RxCachedThreadScheduler-3" daemon prio=5 tid=42 Runnable
  | group="main" sCount=0 dsCount=0 flags=0 obj=0x13742fa8 self=0xcd19ac00
  | sysTid=5930 nice=0 cgrp=default sched=0/0 handle=0xcc6d5970
  | state=R schedstat=( 1644751450 2033895533 2133 ) utm=145 stm=18 core=1 HZ=100
  | stack=0xcc5d3000-0xcc5d5000 stackSize=1038KB
  | held mutexes= "mutator lock"(shared held)
  at com.google.gson.stream.JsonReader.nextNonWhitespace(JsonReader.java:1323)
  at com.google.gson.stream.JsonReader.doPeek(JsonReader.java:483)
  at com.google.gson.stream.JsonReader.hasNext(JsonReader.java:415)
  at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$Adapter.read(ReflectiveTypeAdapterFactory.java:216)
  at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$1.read(ReflectiveTypeAdapterFactory.java:131)
  at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$Adapter.read(ReflectiveTypeAdapterFactory.java:222)
  at com.google.gson.Gson.fromJson(Gson.java:932)
  at com.google.gson.Gson.fromJson(Gson.java:897)
  at com.google.gson.Gson.fromJson(Gson.java:846)
  at com.google.gson.Gson.fromJson(Gson.java:817)
  at com.viomi.devicelib.utils.VGsonUtils.fromJson(VGsonUtils.java:45)
  at com.viomi.devicelib.manager.DeviceCardManager.parseDeviceCard(DeviceCardManager.java:281)
  at com.viomi.devicelib.manager.DeviceCardManager.downLoad(DeviceCardManager.java:264)
  at com.viomi.devicelib.manager.DeviceCardManager.lambda$refreshDevicesCard$0(DeviceCardManager.java:182)
  at com.viomi.devicelib.manager.-$$Lambda$DeviceCardManager$TfD0batl1y7eFSSa4qdhYSpsQ8Y.apply(lambda:-1)
  at io.reactivex.internal.operators.observable.ObservableMap$MapObserver.onNext(ObservableMap.java:57)
  at io.reactivex.internal.operators.observable.ObservableSubscribeOn$SubscribeOnObserver.onNext(ObservableSubscribeOn.java:58)
  at io.reactivex.internal.operators.observable.ObservableObserveOn$ObserveOnObserver.drainNormal(ObservableObserveOn.java:201)
  at io.reactivex.internal.operators.observable.ObservableObserveOn$ObserveOnObserver.run(ObservableObserveOn.java:255)
  at io.reactivex.internal.schedulers.ScheduledRunnable.run(ScheduledRunnable.java:66)
  at io.reactivex.internal.schedulers.ScheduledRunnable.call(ScheduledRunnable.java:57)
  at java.util.concurrent.FutureTask.run(FutureTask.java:266)
  at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:301)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1162)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:636)
  at java.lang.Thread.run(Thread.java:764)

"Signal Catcher" daemon prio=5 tid=3 Runnable
  | group="system" sCount=0 dsCount=0 flags=0 obj=0x135000b0 self=0xeadaa200
  | sysTid=5859 nice=0 cgrp=default sched=0/0 handle=0xe3d87970
  | state=R schedstat=( 63395792 35312378 123 ) utm=3 stm=2 core=3 HZ=100
  | stack=0xe3c8d000-0xe3c8f000 stackSize=1006KB
  | held mutexes= "mutator lock"(shared held)
  native: #00 pc 002e8367  /system/lib/libart.so (art::DumpNativeStack(std::__1::basic_ostream<char, std::__1::char_traits<char>>&, int, BacktraceMap*, char const*, art::ArtMethod*, void*)+130)
  native: #01 pc 00379111  /system/lib/libart.so (art::Thread::DumpStack(std::__1::basic_ostream<char, std::__1::char_traits<char>>&, bool, BacktraceMap*, bool) const+204)
  native: #02 pc 00375847  /system/lib/libart.so (art::Thread::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char>>&, bool, BacktraceMap*, bool) const+34)
  native: #03 pc 0038d1db  /system/lib/libart.so (art::DumpCheckpoint::Run(art::Thread*)+698)
  native: #04 pc 00386d55  /system/lib/libart.so (art::ThreadList::RunCheckpoint(art::Closure*, art::Closure*)+320)
  native: #05 pc 00386853  /system/lib/libart.so (art::ThreadList::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char>>&, bool)+530)
  native: #06 pc 00386573  /system/lib/libart.so (art::ThreadList::DumpForSigQuit(std::__1::basic_ostream<char, std::__1::char_traits<char>>&)+626)
  native: #07 pc 00364453  /system/lib/libart.so (art::Runtime::DumpForSigQuit(std::__1::basic_ostream<char, std::__1::char_traits<char>>&)+122)
  native: #08 pc 0036b54f  /system/lib/libart.so (art::SignalCatcher::HandleSigQuit()+1282)
  native: #09 pc 0036a52d  /system/lib/libart.so (art::SignalCatcher::Run(void*)+240)
  native: #10 pc 0004751f  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #11 pc 0001af8d  /system/lib/libc.so (__start_thread+32)
  (no managed stack frames)

"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x70e93ec0 self=0xeada9000
  | sysTid=5853 nice=-10 cgrp=default sched=0/0 handle=0xeec154a4
  | state=R schedstat=( 3703998364 448965126 3683 ) utm=322 stm=47 core=0 HZ=100
  | stack=0xff601000-0xff603000 stackSize=8MB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/5853/stack)
  native: #00 pc 0000b748  /system/lib/libz.so (inflate+2148)
  native: #01 pc 0002e93d  /system/lib/libandroidfw.so (_Z15inflateToBufferI12BufferReaderEbRT_Pvll+132)
  native: #02 pc 0002e897  /system/lib/libandroidfw.so (android::ZipUtils::inflateToBuffer(void*, void*, long, long)+34)
  native: #03 pc 000188b5  /system/lib/libandroidfw.so (android::_CompressedAsset::getBuffer(bool)+36)
  native: #04 pc 000c66b5  /system/lib/libandroid_runtime.so (???)
  native: #05 pc 001a0635  /system/framework/arm/boot-framework.oat (Java_android_graphics_FontFamily_nAddFontFromAssetManager__JLandroid_content_res_AssetManager_2Ljava_lang_String_2IZIII+196)
  at android.graphics.FontFamily.nAddFontFromAssetManager(Native method)
  at android.graphics.FontFamily.addFontFromAssetManager(FontFamily.java:149)
  at java.lang.reflect.Method.invoke(Native method)
  at androidx.core.graphics.TypefaceCompatApi26Impl.addFontFromAssetManager(TypefaceCompatApi26Impl.java:140)
  at androidx.core.graphics.TypefaceCompatApi26Impl.createFromResourcesFontFile(TypefaceCompatApi26Impl.java:298)
  at androidx.core.graphics.TypefaceCompat.createFromResourcesFontFile(TypefaceCompat.java:176)
  at androidx.core.content.res.ResourcesCompat.loadFont(ResourcesCompat.java:472)
  at androidx.core.content.res.ResourcesCompat.loadFont(ResourcesCompat.java:403)
  at androidx.core.content.res.ResourcesCompat.getFont(ResourcesCompat.java:381)
  at androidx.appcompat.widget.TintTypedArray.getFont(TintTypedArray.java:126)
  at androidx.appcompat.widget.AppCompatTextHelper.updateTypefaceAndStyle(AppCompatTextHelper.java:381)
  at androidx.appcompat.widget.AppCompatTextHelper.loadFromAttributes(AppCompatTextHelper.java:212)
  at androidx.appcompat.widget.AppCompatTextView.<init>(AppCompatTextView.java:110)
  at androidx.appcompat.widget.AppCompatTextView.<init>(AppCompatTextView.java:97)
  at androidx.appcompat.app.AppCompatViewInflater.createTextView(AppCompatViewInflater.java:194)
  at androidx.appcompat.app.AppCompatViewInflater.createView(AppCompatViewInflater.java:115)
  at androidx.appcompat.app.AppCompatDelegateImpl.createView(AppCompatDelegateImpl.java:1563)
  at androidx.appcompat.app.AppCompatDelegateImpl.onCreateView(AppCompatDelegateImpl.java:1614)
  at android.view.LayoutInflater$FactoryMerger.onCreateView(LayoutInflater.java:189)
  at android.view.LayoutInflater.createViewFromTag(LayoutInflater.java:772)
  at android.view.LayoutInflater.createViewFromTag(LayoutInflater.java:730)
  at android.view.LayoutInflater.rInflate(LayoutInflater.java:863)
  at android.view.LayoutInflater.rInflateChildren(LayoutInflater.java:824)
  at android.view.LayoutInflater.rInflate(LayoutInflater.java:866)
  at android.view.LayoutInflater.rInflateChildren(LayoutInflater.java:824)
  at android.view.LayoutInflater.rInflate(LayoutInflater.java:866)
  at android.view.LayoutInflater.rInflateChildren(LayoutInflater.java:824)
  at android.view.LayoutInflater.rInflate(LayoutInflater.java:866)
  at android.view.LayoutInflater.rInflateChildren(LayoutInflater.java:824)
  at android.view.LayoutInflater.rInflate(LayoutInflater.java:866)
  at android.view.LayoutInflater.rInflateChildren(LayoutInflater.java:824)
  at android.view.LayoutInflater.inflate(LayoutInflater.java:515)
  - locked <0x0cc880b7> (a java.lang.Object[])
  at android.view.LayoutInflater.inflate(LayoutInflater.java:423)
  at com.viomi.fridge.vertical.common.base.BaseFragment.onCreateView(BaseFragment.java:39)
  at androidx.fragment.app.Fragment.performCreateView(Fragment.java:2963)
  at androidx.fragment.app.FragmentStateManager.createView(FragmentStateManager.java:518)
  at androidx.fragment.app.FragmentStateManager.moveToExpectedState(FragmentStateManager.java:282)
  at androidx.fragment.app.FragmentManager.executeOpsTogether(FragmentManager.java:2189)
  at androidx.fragment.app.FragmentManager.removeRedundantOperationsAndExecute(FragmentManager.java:2100)
  at androidx.fragment.app.FragmentManager.execSingleAction(FragmentManager.java:1971)
  at androidx.fragment.app.BackStackRecord.commitNowAllowingStateLoss(BackStackRecord.java:311)
  at androidx.fragment.app.FragmentStatePagerAdapter.finishUpdate(FragmentStatePagerAdapter.java:274)
  at androidx.viewpager.widget.ViewPager.populate(ViewPager.java:1244)
  at androidx.viewpager.widget.ViewPager.populate(ViewPager.java:1092)
  at androidx.viewpager.widget.ViewPager.onMeasure(ViewPager.java:1622)
  at android.view.View.measure(View.java:22071)
  at android.widget.RelativeLayout.measureChildHorizontal(RelativeLayout.java:715)
  at android.widget.RelativeLayout.onMeasure(RelativeLayout.java:461)
  at android.view.View.measure(View.java:22071)
  at android.widget.RelativeLayout.measureChildHorizontal(RelativeLayout.java:715)
  at android.widget.RelativeLayout.onMeasure(RelativeLayout.java:461)
  at android.view.View.measure(View.java:22071)
  at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:6602)
  at android.widget.FrameLayout.onMeasure(FrameLayout.java:185)
  at androidx.appcompat.widget.ContentFrameLayout.onMeasure(ContentFrameLayout.java:145)
  at android.view.View.measure(View.java:22071)
  at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:6602)
  at android.widget.LinearLayout.measureChildBeforeLayout(LinearLayout.java:1514)
  at android.widget.LinearLayout.measureVertical(LinearLayout.java:806)
  at android.widget.LinearLayout.onMeasure(LinearLayout.java:685)
  at android.view.View.measure(View.java:22071)
  at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:6602)
  at android.widget.FrameLayout.onMeasure(FrameLayout.java:185)
  at android.view.View.measure(View.java:22071)
  at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:6602)
  at android.widget.LinearLayout.measureChildBeforeLayout(LinearLayout.java:1514)
  at android.widget.LinearLayout.measureVertical(LinearLayout.java:806)
  at android.widget.LinearLayout.onMeasure(LinearLayout.java:685)
  at android.view.View.measure(View.java:22071)
  at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:6602)
  at android.widget.FrameLayout.onMeasure(FrameLayout.java:185)
  at com.android.internal.policy.DecorView.onMeasure(DecorView.java:724)
  at android.view.View.measure(View.java:22071)
  at android.view.ViewRootImpl.performMeasure(ViewRootImpl.java:2422)
  at android.view.ViewRootImpl.measureHierarchy(ViewRootImpl.java:1504)
  at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:1761)
  at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:1392)
  at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:6752)
  at android.view.Choreographer$CallbackRecord.run(Choreographer.java:911)
  at android.view.Choreographer.doCallbacks(Choreographer.java:723)
  at android.view.Choreographer.doFrame(Choreographer.java:658)
  at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:897)
  at android.os.Handler.handleCallback(Handler.java:790)
  at android.os.Handler.dispatchMessage(Handler.java:99)
  at android.os.Looper.loop(Looper.java:164)
  at android.app.ActivityThread.main(ActivityThread.java:6494)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)
```