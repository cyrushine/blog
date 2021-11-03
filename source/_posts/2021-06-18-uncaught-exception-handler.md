---
title: Uncaught Exception Handling
date: 2021-06-18 12:00:00 +0800
tags: [uncaught exception, exception, 崩溃, 崩溃日志, crash]
---

## 测试

根据 `Thread.setDefaultUncaughtExceptionHandler` 的方法文档

> Uncaught exception handling is controlled first by the thread, then by the thread's ThreadGroup object and finally by the default uncaught exception handler. If the thread does not have an explicit uncaught exception handler set, and the thread's thread group (including parent thread groups) does not specialize its uncaughtException method, then the default handler's uncaughtException method will be invoked.

当发生 `Uncaught Exception` 时，将会按照 `Thread.uncaughtExceptionHandler -> ThreadGroup.uncaughtException -> Thread.defaultUncaughtExceptionHandler` 的优先级次序去寻找异常处理器

而 `ThreadGroup.uncaughtException` 的默认实现仅仅是像事件冒泡那样把异常往上传递，跑到 root ThreadGroup 后中止冒泡并交由 `DefaultUncaughtExceptionHandler` 处理 or 打印至标准错误流，所以可以把 `ThreadGroup.uncaughtException` 当作透明的层忽略之

```java
public void uncaughtException(Thread t, Throwable e) {
    if (parent != null) {
        parent.uncaughtException(t, e);
    } else {
        Thread.UncaughtExceptionHandler ueh =
            Thread.getDefaultUncaughtExceptionHandler();
        if (ueh != null) {
            ueh.uncaughtException(t, e);
        } else if (!(e instanceof ThreadDeath)) {
            System.err.print("Exception in thread \""
                             + t.getName() + "\" ");
            e.printStackTrace(System.err);
        }
    }
}
```

下面仅考虑 `Thread.uncaughtExceptionHandler`、`DefaultUncaughtExceptionHandler` 和 `主线程` 三个因素


### 主线程

Thread.uncaughtExceptionHandler/DefaultUncaughtExceptionHandler 可以捕获异常，但无法改变 app 被 blocked 住，然后出现 ANR（即使点击 `等待` 依然被 blocked 住），点击 `确定` 后被 kill 的命运

```log
2021-06-12 11:13:29.464 28055-28055/com.example.myapplication D/AndroidRuntime: Shutting down VM

    --------- beginning of crash
2021-06-12 11:13:29.465 28055-28055/com.example.myapplication E/AndroidRuntime: FATAL EXCEPTION: main
    Process: com.example.myapplication, PID: 28055
    java.lang.RuntimeException: Test Exception
        at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:58)
        at com.example.myapplication.MainActivity.onCreate$lambda-2(MainActivity.kt:30)
        at com.example.myapplication.MainActivity.lambda$hUPUYmntOngyO5ji3KzmjKQ19D4(Unknown Source:0)
        at com.example.myapplication.-$$Lambda$MainActivity$hUPUYmntOngyO5ji3KzmjKQ19D4.onClick(Unknown Source:0)
        at android.view.View.performClick(View.java:7509)
        at com.google.android.material.button.MaterialButton.performClick(MaterialButton.java:1119)
        at android.view.View.performClickInternal(View.java:7486)
        at android.view.View.access$3600(View.java:841)
        at android.view.View$PerformClick.run(View.java:28709)
        at android.os.Handler.handleCallback(Handler.java:938)
        at android.os.Handler.dispatchMessage(Handler.java:99)
        at android.os.Looper.loop(Looper.java:236)
        at android.app.ActivityThread.main(ActivityThread.java:8061)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:656)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:967)

// 如果 Thread.uncaughtExceptionHandler != null or DefaultUncaughtExceptionHandler != null 则能够在自己的 UncaughtExceptionHandler 里捕获异常
2021-06-12 11:13:29.471 28055-28055/com.example.myapplication E/cyrus: main-2 UncaughtExceptionHandler: Test Exception
    java.lang.RuntimeException: Test Exception
        at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:58)
        at com.example.myapplication.MainActivity.onCreate$lambda-2(MainActivity.kt:30)
        at com.example.myapplication.MainActivity.lambda$hUPUYmntOngyO5ji3KzmjKQ19D4(Unknown Source:0)
        at com.example.myapplication.-$$Lambda$MainActivity$hUPUYmntOngyO5ji3KzmjKQ19D4.onClick(Unknown Source:0)
        at android.view.View.performClick(View.java:7509)
        at com.google.android.material.button.MaterialButton.performClick(MaterialButton.java:1119)
        at android.view.View.performClickInternal(View.java:7486)
        at android.view.View.access$3600(View.java:841)
        at android.view.View$PerformClick.run(View.java:28709)
        at android.os.Handler.handleCallback(Handler.java:938)
        at android.os.Handler.dispatchMessage(Handler.java:99)
        at android.os.Looper.loop(Looper.java:236)
        at android.app.ActivityThread.main(ActivityThread.java:8061)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:656)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:967)        

// app 没有立刻崩溃，但是进入 blocked 状态，点击 app 没响应然后触发 ANR
2021-06-12 11:17:10.449 1764-28190/? I/ActivityManager: Collecting stacks for pid 28055
2021-06-12 11:17:10.449 1764-28190/? I/system_server: libdebuggerd_client: started dumping process 28055
2021-06-12 11:17:10.450 697-697/? I/tombstoned: registered intercept for pid 28055 and type kDebuggerdJavaBacktrace
2021-06-12 11:17:10.450 28055-28065/com.example.myapplication I/e.myapplicatio: Thread[6,tid=28065,WaitingInMainSignalCatcherLoop,Thread*=0xb400007271841000,peer=0x13780260,"Signal Catcher"]: reacting to signal 3
2021-06-12 11:17:10.528 697-697/? I/tombstoned: received crash request for pid 28055
2021-06-12 11:17:10.528 697-697/? I/tombstoned: found intercept fd 512 for pid 28055 and type kDebuggerdJavaBacktrace
2021-06-12 11:17:10.528 28055-28065/com.example.myapplication I/e.myapplicatio: Wrote stack traces to tombstoned
2021-06-12 11:17:10.529 1764-28190/? I/system_server: libdebuggerd_client: done dumping process 28055

// 需要一段时间来 dump ANR 信息
2021-06-12 11:17:15.839 1764-28190/? E/ActivityManager: ANR in com.example.myapplication (com.example.myapplication/.MainActivity)
    PID: 28055
    Reason: Input dispatching timed out (com.example.myapplication/com.example.myapplication.MainActivity, 7fc1f1 com.example.myapplication/com.example.myapplication.MainActivity (server) is not responding. Waited 8001ms for MotionEvent(action=DOWN))
    Parent: com.example.myapplication/.MainActivity
    Load: 0.13 / 0.47 / 0.83
    ----- Output from /proc/pressure/memory -----
    some avg10=0.00 avg60=0.00 avg300=0.00 total=15206960
    full avg10=0.00 avg60=0.00 avg300=0.00 total=5127529
    ----- End output from /proc/pressure/memory -----
    
    CPU usage from 0ms to 5796ms later (2021-06-12 11:17:09.982 to 2021-06-12 11:17:15.778):
      0.1% 1465/media.codec: 0% user + 0% kernel / faults: 38284 minor
      18% 1764/system_server: 8.2% user + 10% kernel / faults: 4408 minor
      0% 1509/media.swcodec: 0% user + 0% kernel / faults: 21804 minor
      0% 899/media.hwcodec: 0% user + 0% kernel / faults: 7313 minor
      0.1% 5899/com.sohu.inputmethod.sogou: 0.1% user + 0% kernel / faults: 1684 minor
      2% 895/kworker/u16:16: 0% user + 2% kernel
      2% 2930/com.android.phone: 1.2% user + 0.8% kernel / faults: 1835 minor
      1.8% 1284/adbd: 0.5% user + 1.3% kernel
      0% 1426/media.extractor: 0% user + 0% kernel / faults: 3197 minor
      1.8% 25102/kworker/u16:5: 0% user + 1.8% kernel
      1.5% 982/surfaceflinger: 0.1% user + 1.3% kernel / faults: 41 minor
      0% 7493/kworker/u16:14: 0% user + 0% kernel
      0% 28055/com.example.myapplication: 0% user + 0% kernel / faults: 1974 minor
      1.2% 5242/com.android.nfc: 1% user + 0.1% kernel / faults: 796 minor
      1% 18959/com.viomi.fridge.vertical: 0.8% user + 0.1% kernel / faults: 17 minor
      1% 28105/kworker/u16:0: 0% user + 1% kernel
      0.8% 584/logd: 0.3% user + 0.5% kernel
      0.8% 836/android.hardware.sensors@1.0-service: 0.3% user + 0.5% kernel / faults: 115 minor
      0.8% 1359/cnss_diag: 0.6% user + 0.1% kernel
      0.8% 25780/com.xiaomi.market: 0.6% user + 0.1% kernel / faults: 736 minor 2 major
      0.6% 806/android.hardware.graphics.composer@2.4-service: 0% user + 0.6% kernel / faults: 238 minor 2 major
      0.6% 873/vendor.qti.hardware.perf@2.2-service: 0.3% user + 0.3% kernel / faults: 33 minor
      0% 1443/mediaserver: 0% user + 0% kernel / faults: 70 minor
      0.6% 16518/com.miui.player: 0.1% user + 0.5% kernel
      0% 1/init: 0% user + 0% kernel
      0% 697/tombstoned: 0% user + 0% kernel
      0% 799/android.hardware.camera.provider@2.4-service_64: 0% user + 0% kernel / faults: 26 minor
      0% 949/audioserver: 0% user + 0% kernel / faults: 56 minor
      0.5% 20092/kworker/u17:0: 0% user + 0.5% kernel
      0.5% 28100/logcat: 0% user + 0.5% kernel
      0.3% 9/rcu_preempt: 0% user + 0.3% kernel
      0.3% 495/crtc_commit:131: 0% user + 0.3% kernel
      0.3% 534/irq/303-fts: 0% user + 0.3% kernel
      0.3% 704/statsd: 0.1% user + 0.1% kernel / faults: 27 minor
      0.3% 705/netd: 0.1% user + 0.1% kernel / faults: 62 minor
      0% 1367/drmserver: 0% user + 0% kernel / faults: 16 minor
      0.3% 2635/com.android.systemui: 0.3% user + 0% kernel / faults: 27 minor
      0.3% 3547/irq/32-90b6400.: 0% user + 0.3% kernel
      0.3% 6938/kworker/u17:2: 0% user + 0.3% kernel
      0.3% 8546/com.tencent.mm:toolsmp: 0.1% user + 0.1% kernel / faults: 5 minor
      0.3% 13715/com.tencent.mm: 0.1% user + 0.1% kernel / faults: 5 minor
      0.1% 10/rcu_sched: 0% user + 0.1% kernel
      0.1% 12/rcuop/0: 0% user + 0.1% kernel
      0% 13/rcuos/0: 0% user + 0% kernel
      0.1% 30/rcuop/2: 0% user + 0.1% kernel
      0% 31/rcuos/2: 0% user + 0% kernel
      0.1% 38/rcuop/3: 0% user + 0.1% kernel
      0% 66/migration/7: 0% user + 0% kernel
      0.1% 586/servicemanager: 0.1% user + 0% kernel
      0.1% 598/android.hardware.keymaster@4.0-service-qti: 0% user + 0.1% kernel / faults: 11 minor
      0.1% 628/vold: 0% user + 0.1% kernel / faults: 29 minor
      0.1% 664/ipacm: 0% user + 0.1% kernel
      0.1% 676/jbd2/sda31-8: 0% user + 0.1% kernel
      0% 793/android.hardware.audio.service: 0% user + 0% kernel / faults: 39 minor
      0% 798/android.hardware.bluetooth@1.0-service-qti: 0% user + 0% kernel / faults: 11 minor
2021-06-12 11:17:15.839 1764-28190/? E/ActivityManager:   0.1% 807/android.hardware.health@2.1-service: 0% user + 0.1% kernel / faults: 9 minor
      0% 828/android.hardware.neuralnetworks@1.3-service-qti: 0% user + 0% kernel / faults: 65 minor
      0.1% 851/android.hardware.wifi@1.0-service: 0.1% user + 0% kernel
      0% 1365/cameraserver: 0% user + 0% kernel / faults: 70 minor
      0% 1440/media.metrics: 0% user + 0% kernel / faults: 36 minor 1 major
      0% 1572/gatekeeperd: 0% user + 0% kernel / faults: 28 minor 7 major
      0% 1605/android.hardware.biometrics.fingerprint@2.1-service: 0% user + 0% kernel / faults: 16 minor
      0.1% 2251/cds_ol_rx_threa: 0% user + 0.1% kernel
      0.1% 2884/com.qualcomm.qti.devicestatisticsservice: 0.1% user + 0% kernel / faults: 1 minor
      0.1% 3549/irq/33-90cd000.: 0% user + 0.1% kernel
      0.1% 6156/com.xiaomi.xmsf: 0.1% user + 0% kernel / faults: 27 minor
      0.1% 8036/com.tencent.mm:appbrand0: 0.1% user + 0% kernel / faults: 6 minor
      0.1% 8047/com.tencent.mm:appbrand1: 0% user + 0.1% kernel / faults: 6 minor
      0.1% 8864/com.xiaomi.joyose: 0.1% user + 0% kernel
      0.1% 14440/com.tencent.mm:push: 0% user + 0.1% kernel / faults: 8 minor
      0.1% 15807/com.miui.personalassistant: 0% user + 0.1% kernel / faults: 9 minor
      0.1% 27963/kworker/2:3: 0% user + 0.1% kernel
      0.1% 28106/kworker/u16:3: 0% user + 0.1% kernel
      0.1% 28171/kworker/0:1: 0% user + 0.1% kernel
    14% TOTAL: 7.1% user + 6.5% kernel + 0.1% iowait + 0.5% irq + 0.2% softirq
    CPU usage from 41ms to 439ms later (2021-06-12 11:17:10.023 to 2021-06-12 11:17:10.422) with 99% awake:
      51% 1764/system_server: 18% user + 33% kernel / faults: 929 minor
        39% 28190/AnrConsumer: 9% user + 30% kernel
        6% 1789/android.ui: 3% user + 3% kernel
        3% 2347/InputDispatcher: 3% user + 0% kernel
      2.5% 66/migration/7: 0% user + 2.5% kernel
      2.6% 534/irq/303-fts: 0% user + 2.6% kernel
      2.6% 584/logd: 2.6% user + 0% kernel
      2.7% 873/vendor.qti.hardware.perf@2.2-service: 0% user + 2.7% kernel / faults: 6 minor
        2.7% 873/perf@2.2-servic: 0% user + 2.7% kernel
       +0% 28191/POSIX timer 269: 0% user + 0% kernel
       +0% 28192/POSIX timer 269: 0% user + 0% kernel
      2.7% 895/kworker/u16:16: 0% user + 2.7% kernel
      2.8% 982/surfaceflinger: 2.8% user + 0% kernel
      2.8% 1284/adbd: 0% user + 2.8% kernel
    8.9% TOTAL: 2.8% user + 5.7% kernel + 0.3% irq

// 点击 ANR 对话框的确定按钮杀死 app
2021-06-12 11:18:58.935 1764-1789/? I/ActivityManager: Killing 28055:com.example.myapplication/u0a161 (adj 0): user request after error
2021-06-12 11:18:58.937 1764-1789/? I/Process: PerfMonitor : current process sending signal quiet. PID: 28055 SIG: 9
2021-06-12 11:18:58.938 1764-1802/? I/Process: PerfMonitor : current process killing process group. PID: 28055
2021-06-12 11:18:58.965 706-706/? I/Zygote: Process 28055 exited due to signal 9 (Killed)
2021-06-12 11:18:58.966 1764-1802/? I/libprocessgroup: Successfully killed process cgroup uid 10161 pid 28055 in 28ms
2021-06-12 11:18:58.970 882-882/? I/vendor.qti.hardware.servicetracker@1.2-service: killProcess is called for pid : 28055
2021-06-12 11:18:58.970 1764-5448/? W/ANRStateManager: clear state, but process isn't exist. hash=92035998 uid=10161 pid=28055 state=16
```


### 子线程且 UEH != null

app 没有发生 ANR 也没有崩溃，且无论 `DefaultUncaughtExceptionHandler` 是否为 null，`Thread.uncaughtExceptionHandler` 都能够有限捕获异常，说明线程的 UncaughtExceptionHandler 比默认的 UncaughtExceptionHandler 优先级要高

```log
2021-06-12 17:20:41.503 20615-22015/com.example.myapplication E/AndroidRuntime: FATAL EXCEPTION: Thread-10
    Process: com.example.myapplication, PID: 20615
    java.lang.RuntimeException: Test Exception
        at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:75)
        at com.example.myapplication.MainActivity$onCreate$4$1.invoke(MainActivity.kt:38)
        at com.example.myapplication.MainActivity$onCreate$4$1.invoke(MainActivity.kt:36)
        at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)

2021-06-12 17:20:41.509 20615-22015/com.example.myapplication E/cyrus: Thread-10-3890 UncaughtExceptionHandler: Test Exception
    java.lang.RuntimeException: Test Exception
        at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:75)
        at com.example.myapplication.MainActivity$onCreate$4$1.invoke(MainActivity.kt:38)
        at com.example.myapplication.MainActivity$onCreate$4$1.invoke(MainActivity.kt:36)
        at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)
```


### 两个UEH都置空 or DUEH != null

app 没有发生 ANR 也没有崩溃

```log
    --------- beginning of crash
2021-06-12 11:06:58.817 27816-27929/com.example.myapplication E/AndroidRuntime: FATAL EXCEPTION: Thread-3
    Process: com.example.myapplication, PID: 27816
    java.lang.RuntimeException: Test Exception
        at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:58)
        at com.example.myapplication.MainActivity$onCreate$2$1.invoke(MainActivity.kt:24)
        at com.example.myapplication.MainActivity$onCreate$2$1.invoke(MainActivity.kt:22)
        at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)

// DefaultUncaughtExceptionHandler != null 则能够捕获异常
2021-06-12 17:03:51.701 20615-20884/com.example.myapplication E/cyrus: Thread-2-3882 DefaultUncaughtExceptionHandler: Test Exception
    java.lang.RuntimeException: Test Exception
        at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:75)
        at com.example.myapplication.MainActivity$onCreate$2$1.invoke(MainActivity.kt:24)
        at com.example.myapplication.MainActivity$onCreate$2$1.invoke(MainActivity.kt:22)
        at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)        

// 当 DefaultUncaughtExceptionHandler == null 时异常被输出到标准异常流
2021-06-12 17:13:54.170 20615-21405/com.example.myapplication W/System.err: Exception in thread "Thread-8" java.lang.RuntimeException: Test Exception
2021-06-12 17:13:54.171 20615-21405/com.example.myapplication W/System.err:     at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:75)
2021-06-12 17:13:54.171 20615-21405/com.example.myapplication W/System.err:     at com.example.myapplication.MainActivity$onCreate$1$1.invoke(MainActivity.kt:17)
2021-06-12 17:13:54.171 20615-21405/com.example.myapplication W/System.err:     at com.example.myapplication.MainActivity$onCreate$1$1.invoke(MainActivity.kt:15)
2021-06-12 17:13:54.171 20615-21405/com.example.myapplication W/System.err:     at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)   
```


### 子线程且为默认 DUEH 

默认的 DefaultUncaughtExceptionHandler 是 KillApplicationHandler，它会杀死 app

```log
// 除了有上一节的日志外，还会有以下日志并且 app 被 kill
// app process 收到信号 SIGKILL(9) 被迫退出，app 立刻崩溃掉
2021-06-12 11:06:58.870 27816-27929/com.example.myapplication I/Process: Sending signal. PID: 27816 SIG: 9
2021-06-12 11:06:58.910 1764-1802/? I/Process: PerfMonitor : current process killing process group. PID: 27816
2021-06-12 11:06:58.911 1764-2936/? I/ActivityManager: Process com.example.myapplication (pid 27816) has died: prcp CRE 
2021-06-12 11:06:58.911 706-706/? I/Zygote: Process 27816 exited due to signal 9 (Killed)
2021-06-12 11:06:58.912 1764-1802/? I/libprocessgroup: Successfully killed process cgroup uid 10161 pid 27816 in 0ms
2021-06-12 11:06:58.917 1764-2936/? W/ANRStateManager: clear state, but process isn't exist. hash=224220282 uid=10161 pid=27816 state=16
2021-06-12 11:06:58.919 882-8554/? I/vendor.qti.hardware.servicetracker@1.2-service: killProcess is called for pid : 27816
```


### 总结

| 所在线程 | 子条件 | 结果 |
|---------|--------|------|
| 主线程 | | 始终会被 blocked 住，然后发生 ANR，最后被杀死 |
| 子线程 | 两个 UncaughtExceptionHandler 都置空 <br> 有任意一个自定义的 UncaughtExceptionHandler | app 没事 |
| | 默认 | KillApplicationHandler 捕获到异常并杀死 app |


## 代码跟踪

### 抛出 UE 时发生了什么

有一个 API 可以抛出异常：`JNIEnv->Throw`，所以我猜当 java 层发生 uncaught exception 时相当于调用了它

这个方法的实现很简单，就是把 exception 记录在 Thread::tlsPtr_::exception

```cpp
// art/runtime/jni/jni_internal.cc
static jint Throw(JNIEnv* env, jthrowable java_exception) {
  ScopedObjectAccess soa(env);
  ObjPtr<mirror::Throwable> exception = soa.Decode<mirror::Throwable>(java_exception);
  if (exception == nullptr) {
    return JNI_ERR;
  }
  soa.Self()->SetException(exception);
  return JNI_OK;
}

// art/runtime/thread.cc
void Thread::SetException(ObjPtr<mirror::Throwable> new_exception) {
  CHECK(new_exception != nullptr);
  // TODO: DCHECK(!IsExceptionPending());
  tlsPtr_.exception = new_exception.Ptr();
}

// art/runtime/thread.h
mirror::Throwable* exception;   // The pending exception or null.
```

接下来我猜想埋点在代码里的异常检查流程在发现 pending exception != null 后，会中断字节码的执行（`Thread.run()`）从而回到 native 代码

让我们从开启一个线程 `Thread.start()` 看看这个流程

```cpp
Thread.start()
Thread.nativeCreate()

// java_lang_Thread.cc
static void Thread_nativeCreate(JNIEnv* env, jclass, jobject java_thread, jlong stack_size,
                                jboolean daemon) {
  // There are sections in the zygote that forbid thread creation.
  Runtime* runtime = Runtime::Current();
  if (runtime->IsZygote() && runtime->IsZygoteNoThreadSection()) {
    jclass internal_error = env->FindClass("java/lang/InternalError");
    CHECK(internal_error != nullptr);
    env->ThrowNew(internal_error, "Cannot create threads in zygote");
    return;
  }

  Thread::CreateNativeThread(env, java_thread, stack_size, daemon == JNI_TRUE);
}

// art/runtime/thread.cc
// 最终调用 pthread_create 创建线程，新线程的入口是 Thread::CreateCallback
void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
  CHECK(java_peer != nullptr);
  Thread* self = static_cast<JNIEnvExt*>(env)->GetSelf();

  if (VLOG_IS_ON(threads)) {
    ScopedObjectAccess soa(env);

    ArtField* f = jni::DecodeArtField(WellKnownClasses::java_lang_Thread_name);
    ObjPtr<mirror::String> java_name =
        f->GetObject(soa.Decode<mirror::Object>(java_peer))->AsString();
    std::string thread_name;
    if (java_name != nullptr) {
      thread_name = java_name->ToModifiedUtf8();
    } else {
      thread_name = "(Unnamed)";
    }

    VLOG(threads) << "Creating native thread for " << thread_name;
    self->Dump(LOG_STREAM(INFO));
  }

  Runtime* runtime = Runtime::Current();

  // Atomically start the birth of the thread ensuring the runtime isn't shutting down.
  bool thread_start_during_shutdown = false;
  {
    MutexLock mu(self, *Locks::runtime_shutdown_lock_);
    if (runtime->IsShuttingDownLocked()) {
      thread_start_during_shutdown = true;
    } else {
      runtime->StartThreadBirth();
    }
  }
  if (thread_start_during_shutdown) {
    ScopedLocalRef<jclass> error_class(env, env->FindClass("java/lang/InternalError"));
    env->ThrowNew(error_class.get(), "Thread starting during runtime shutdown");
    return;
  }

  Thread* child_thread = new Thread(is_daemon);
  // Use global JNI ref to hold peer live while child thread starts.
  child_thread->tlsPtr_.jpeer = env->NewGlobalRef(java_peer);
  stack_size = FixStackSize(stack_size);

  // Thread.start is synchronized, so we know that nativePeer is 0, and know that we're not racing
  // to assign it.
  env->SetLongField(java_peer, WellKnownClasses::java_lang_Thread_nativePeer,
                    reinterpret_cast<jlong>(child_thread));

  // Try to allocate a JNIEnvExt for the thread. We do this here as we might be out of memory and
  // do not have a good way to report this on the child's side.
  std::string error_msg;
  std::unique_ptr<JNIEnvExt> child_jni_env_ext(
      JNIEnvExt::Create(child_thread, Runtime::Current()->GetJavaVM(), &error_msg));

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

  // Either JNIEnvExt::Create or pthread_create(3) failed, so clean up.
  {
    MutexLock mu(self, *Locks::runtime_shutdown_lock_);
    runtime->EndThreadBirth();
  }
  // Manually delete the global reference since Thread::Init will not have been run. Make sure
  // nothing can observe both opeer and jpeer set at the same time.
  child_thread->DeleteJPeer(env);
  delete child_thread;
  child_thread = nullptr;
  // TODO: remove from thread group?
  env->SetLongField(java_peer, WellKnownClasses::java_lang_Thread_nativePeer, 0);
  {
    std::string msg(child_jni_env_ext.get() == nullptr ?
        StringPrintf("Could not allocate JNI Env: %s", error_msg.c_str()) :
        StringPrintf("pthread_create (%s stack) failed: %s",
                                 PrettySize(stack_size).c_str(), strerror(pthread_create_result)));
    ScopedObjectAccess soa(env);
    soa.Self()->ThrowOutOfMemoryError(msg.c_str());
  }
}

// art/runtime/thread.cc
void* Thread::CreateCallback(void* arg) {
  Thread* self = reinterpret_cast<Thread*>(arg);
  Runtime* runtime = Runtime::Current();
  if (runtime == nullptr) {
    LOG(ERROR) << "Thread attaching to non-existent runtime: " << *self;
    return nullptr;
  }
  {
    // TODO: pass self to MutexLock - requires self to equal Thread::Current(), which is only true
    //       after self->Init().
    MutexLock mu(nullptr, *Locks::runtime_shutdown_lock_);
    // Check that if we got here we cannot be shutting down (as shutdown should never have started
    // while threads are being born).
    CHECK(!runtime->IsShuttingDownLocked());
    // Note: given that the JNIEnv is created in the parent thread, the only failure point here is
    //       a mess in InitStackHwm. We do not have a reasonable way to recover from that, so abort
    //       the runtime in such a case. In case this ever changes, we need to make sure here to
    //       delete the tmp_jni_env, as we own it at this point.
    CHECK(self->Init(runtime->GetThreadList(), runtime->GetJavaVM(), self->tlsPtr_.tmp_jni_env));
    self->tlsPtr_.tmp_jni_env = nullptr;
    Runtime::Current()->EndThreadBirth();
  }
  {
    ScopedObjectAccess soa(self);
    self->InitStringEntryPoints();

    // Copy peer into self, deleting global reference when done.
    CHECK(self->tlsPtr_.jpeer != nullptr);
    self->tlsPtr_.opeer = soa.Decode<mirror::Object>(self->tlsPtr_.jpeer).Ptr();
    // Make sure nothing can observe both opeer and jpeer set at the same time.
    self->DeleteJPeer(self->GetJniEnv());
    self->SetThreadName(self->GetThreadName()->ToModifiedUtf8().c_str());

    ArtField* priorityField = jni::DecodeArtField(WellKnownClasses::java_lang_Thread_priority);
    self->SetNativePriority(priorityField->GetInt(self->tlsPtr_.opeer));

    runtime->GetRuntimeCallbacks()->ThreadStart(self);

    // Unpark ourselves if the java peer was unparked before it started (see
    // b/28845097#comment49 for more information)

    ArtField* unparkedField = jni::DecodeArtField(
        WellKnownClasses::java_lang_Thread_unparkedBeforeStart);
    bool should_unpark = false;
    {
      // Hold the lock here, so that if another thread calls unpark before the thread starts
      // we don't observe the unparkedBeforeStart field before the unparker writes to it,
      // which could cause a lost unpark.
      art::MutexLock mu(soa.Self(), *art::Locks::thread_list_lock_);
      should_unpark = unparkedField->GetBoolean(self->tlsPtr_.opeer) == JNI_TRUE;
    }
    if (should_unpark) {
      self->Unpark();
    }

    // 重点在这里，执行 Thread.run()
    // Invoke the 'run' method of our java.lang.Thread.
    ObjPtr<mirror::Object> receiver = self->tlsPtr_.opeer;
    jmethodID mid = WellKnownClasses::java_lang_Thread_run;
    ScopedLocalRef<jobject> ref(soa.Env(), soa.AddLocalReference<jobject>(receiver));
    InvokeVirtualOrInterfaceWithJValues(soa, ref.get(), mid, nullptr);
  }

  // Thread.run() 返回后就销毁此线程
  // Detach and delete self.
  Runtime::Current()->GetThreadList()->Unregister(self);

  return nullptr;
}

// art/runtime/thread_list.cc
void ThreadList::Unregister(Thread* self) {
  DCHECK_EQ(self, Thread::Current());
  CHECK_NE(self->GetState(), kRunnable);
  Locks::mutator_lock_->AssertNotHeld(self);

  VLOG(threads) << "ThreadList::Unregister() " << *self;

  {
    MutexLock mu(self, *Locks::thread_list_lock_);
    ++unregistering_count_;
  }

  // Any time-consuming destruction, plus anything that can call back into managed code or
  // suspend and so on, must happen at this point, and not in ~Thread. The self->Destroy is what
  // causes the threads to join. It is important to do this after incrementing unregistering_count_
  // since we want the runtime to wait for the daemon threads to exit before deleting the thread
  // list.
  self->Destroy();

  // If tracing, remember thread id and name before thread exits.
  Trace::StoreExitingThreadInfo(self);

  uint32_t thin_lock_id = self->GetThreadId();
  while (true) {
    // Remove and delete the Thread* while holding the thread_list_lock_ and
    // thread_suspend_count_lock_ so that the unregistering thread cannot be suspended.
    // Note: deliberately not using MutexLock that could hold a stale self pointer.
    {
      MutexLock mu(self, *Locks::thread_list_lock_);
      if (!Contains(self)) {
        std::string thread_name;
        self->GetThreadName(thread_name);
        std::ostringstream os;
        DumpNativeStack(os, GetTid(), nullptr, "  native: ", nullptr);
        LOG(ERROR) << "Request to unregister unattached thread " << thread_name << "\n" << os.str();
        break;
      } else {
        MutexLock mu2(self, *Locks::thread_suspend_count_lock_);
        if (!self->IsSuspended()) {
          list_.remove(self);
          break;
        }
      }
    }
    // In the case where we are not suspended yet, sleep to leave other threads time to execute.
    // This is important if there are realtime threads. b/111277984
    usleep(1);
    // We failed to remove the thread due to a suspend request, loop and try again.
  }
  delete self;

  // Release the thread ID after the thread is finished and deleted to avoid cases where we can
  // temporarily have multiple threads with the same thread id. When this occurs, it causes
  // problems in FindThreadByThreadId / SuspendThreadByThreadId.
  ReleaseThreadId(nullptr, thin_lock_id);

  // Clear the TLS data, so that the underlying native thread is recognizably detached.
  // (It may wish to reattach later.)
#ifdef __BIONIC__
  __get_tls()[TLS_SLOT_ART_THREAD_SELF] = nullptr;
#else
  CHECK_PTHREAD_CALL(pthread_setspecific, (Thread::pthread_key_self_, nullptr), "detach self");
  Thread::self_tls_ = nullptr;
#endif

  // Signal that a thread just detached.
  MutexLock mu(nullptr, *Locks::thread_list_lock_);
  --unregistering_count_;
  Locks::thread_exit_cond_->Broadcast(nullptr);
}

// art/runtime/thread.cc
void Thread::Destroy() {
  Thread* self = this;
  DCHECK_EQ(self, Thread::Current());

  if (tlsPtr_.jni_env != nullptr) {
    {
      ScopedObjectAccess soa(self);
      MonitorExitVisitor visitor(self);
      // On thread detach, all monitors entered with JNI MonitorEnter are automatically exited.
      tlsPtr_.jni_env->monitors_.VisitRoots(&visitor, RootInfo(kRootVMInternal));
    }
    // Release locally held global references which releasing may require the mutator lock.
    if (tlsPtr_.jpeer != nullptr) {
      // If pthread_create fails we don't have a jni env here.
      tlsPtr_.jni_env->DeleteGlobalRef(tlsPtr_.jpeer);
      tlsPtr_.jpeer = nullptr;
    }
    if (tlsPtr_.class_loader_override != nullptr) {
      tlsPtr_.jni_env->DeleteGlobalRef(tlsPtr_.class_loader_override);
      tlsPtr_.class_loader_override = nullptr;
    }
  }

  if (tlsPtr_.opeer != nullptr) {
    ScopedObjectAccess soa(self);

    // 销毁线程的时候会检查一下有没 pending exception，也就是此线程在执行代码过程中发生的 uncaught exception
    // We may need to call user-supplied managed code, do this before final clean-up.
    HandleUncaughtExceptions(soa);
    RemoveFromThreadGroup(soa);
    Runtime* runtime = Runtime::Current();
    if (runtime != nullptr) {
      runtime->GetRuntimeCallbacks()->ThreadDeath(self);
    }

    // this.nativePeer = 0;
    if (Runtime::Current()->IsActiveTransaction()) {
      jni::DecodeArtField(WellKnownClasses::java_lang_Thread_nativePeer)
          ->SetLong<true>(tlsPtr_.opeer, 0);
    } else {
      jni::DecodeArtField(WellKnownClasses::java_lang_Thread_nativePeer)
          ->SetLong<false>(tlsPtr_.opeer, 0);
    }

    // Thread.join() is implemented as an Object.wait() on the Thread.lock object. Signal anyone
    // who is waiting.
    ObjPtr<mirror::Object> lock =
        jni::DecodeArtField(WellKnownClasses::java_lang_Thread_lock)->GetObject(tlsPtr_.opeer);
    // (This conditional is only needed for tests, where Thread.lock won't have been set.)
    if (lock != nullptr) {
      StackHandleScope<1> hs(self);
      Handle<mirror::Object> h_obj(hs.NewHandle(lock));
      ObjectLock<mirror::Object> locker(self, h_obj);
      locker.NotifyAll();
    }
    tlsPtr_.opeer = nullptr;
  }

  {
    ScopedObjectAccess soa(self);
    Runtime::Current()->GetHeap()->RevokeThreadLocalBuffers(this);
  }
  // Mark-stack revocation must be performed at the very end. No
  // checkpoint/flip-function or read-barrier should be called after this.
  if (kUseReadBarrier) {
    Runtime::Current()->GetHeap()->ConcurrentCopyingCollector()->RevokeThreadLocalMarkStack(this);
  }
}

// art/runtime/thread.cc
void Thread::HandleUncaughtExceptions(ScopedObjectAccessAlreadyRunnable& soa) {
  if (!IsExceptionPending()) {
    return;
  }
  ScopedLocalRef<jobject> peer(tlsPtr_.jni_env, soa.AddLocalReference<jobject>(tlsPtr_.opeer));
  ScopedThreadStateChange tsc(this, kNative);

  // Get and clear the exception.
  ScopedLocalRef<jthrowable> exception(tlsPtr_.jni_env, tlsPtr_.jni_env->ExceptionOccurred());
  tlsPtr_.jni_env->ExceptionClear();

  // 如果存在 pending exception/uncaught exception，则执行 Thread.dispatchUncaughtException()
  // Call the Thread instance's dispatchUncaughtException(Throwable)
  tlsPtr_.jni_env->CallVoidMethod(peer.get(),
      WellKnownClasses::java_lang_Thread_dispatchUncaughtException,
      exception.get());

  // If the dispatchUncaughtException threw, clear that exception too.
  tlsPtr_.jni_env->ExceptionClear();
}
```

线程进入 VM 的入口点是 `Thread.run()`，执行完毕（或者发生 uncaught exception 被中断字节码的执行）退出 VM 回到 native 代码后，就执行销毁线程的流程：`ThreadList::Unregister` -> `Thread::Destroy`，其中 `HandleUncaughtExceptions` 会检查是否有 uncaught exception/pending exception，有的话再次进入 VM 执行 `Thread.dispatchUncaughtException`


#### UEH 的入口点

如果有 `Thread.uncaughtExceptionHandler` 则直接给它处理，否则事件冒泡给到 ThreadGroup，ThreadGroup 会把异常一直冒泡到 root ThreadGroup，然后交由 `DefaultUncaughtExceptionHandler` 处理

```java
public class Thread {
    public final void dispatchUncaughtException(Throwable e) {
        // BEGIN Android-added: uncaughtExceptionPreHandler for use by platform.
        Thread.UncaughtExceptionHandler initialUeh =
                Thread.getUncaughtExceptionPreHandler();
        if (initialUeh != null) {
            try {
                initialUeh.uncaughtException(this, e);
            } catch (RuntimeException | Error ignored) {
                // Throwables thrown by the initial handler are ignored
            }
        
        // END Android-added: uncaughtExceptionPreHandler for use by platform.
        getUncaughtExceptionHandler().uncaughtException(this, e);
    }
    
    public UncaughtExceptionHandler getUncaughtExceptionHandler() {
        return uncaughtExceptionHandler != null ?
            uncaughtExceptionHandler : group;
    }        
}

public class ThreadGroup {
    public void uncaughtException(Thread t, Throwable e) {
        if (parent != null) {
            parent.uncaughtException(t, e);
        } else {
            Thread.UncaughtExceptionHandler ueh =
                Thread.getDefaultUncaughtExceptionHandler();
            if (ueh != null) {
                ueh.uncaughtException(t, e);
            } else if (!(e instanceof ThreadDeath)) {
                System.err.print("Exception in thread \""
                                 + t.getName() + "\" ");
                e.printStackTrace(System.err);
            }
        }
    }    
}
```


## 谁打印了 FATAL EXCEPTION

在上面的代码块里 dispatchUncaughtException 还调用了 `Thread.uncaughtExceptionPreHandler`，这个 handler 是在 app 进程初始化时配置的，而且没有暴露给用户，就是它打印了 `AndroidRuntime: FATAL EXCEPTION` 的日志

```java
class RuntimeInit {
    protected static final void commonInit() {
        if (DEBUG) Slog.d(TAG, "Entered RuntimeInit!");

        /*
         * set handlers; these apply to all threads in the VM. Apps can replace
         * the default handler, but not the pre handler.
         */
        LoggingHandler loggingHandler = new LoggingHandler();
        RuntimeHooks.setUncaughtExceptionPreHandler(loggingHandler);
        Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler(loggingHandler));

        /*
         * Install a time zone supplier that uses the Android persistent time zone system property.
         */
        RuntimeHooks.setTimeZoneIdSupplier(() -> SystemProperties.get("persist.sys.timezone"));

        /*
         * Sets handler for java.util.logging to use Android log facilities.
         * The odd "new instance-and-then-throw-away" is a mirror of how
         * the "java.util.logging.config.class" system property works. We
         * can't use the system property here since the logger has almost
         * certainly already been initialized.
         */
        LogManager.getLogManager().reset();
        new AndroidConfig();

        /*
         * Sets the default HTTP User-Agent used by HttpURLConnection.
         */
        String userAgent = getDefaultUserAgent();
        System.setProperty("http.agent", userAgent);

        /*
         * Wire socket tagging to traffic stats.
         */
        NetworkManagementSocketTagger.install();

        initialized = true;
    }

    private static class LoggingHandler implements Thread.UncaughtExceptionHandler {
        public volatile boolean mTriggered = false;

        @Override
        public void uncaughtException(Thread t, Throwable e) {
            mTriggered = true;

            // Don't re-enter if KillApplicationHandler has already run
            if (mCrashing) return;

            // mApplicationObject is null for non-zygote java programs (e.g. "am")
            // There are also apps running with the system UID. We don't want the
            // first clause in either of these two cases, only for system_server.
            if (mApplicationObject == null && (Process.SYSTEM_UID == Process.myUid())) {
                Clog_e(TAG, "*** FATAL EXCEPTION IN SYSTEM PROCESS: " + t.getName(), e);
            } else {
                logUncaught(t.getName(), ActivityThread.currentProcessName(), Process.myPid(), e);
            }
        }
    }

    public static void logUncaught(String threadName, String processName, int pid, Throwable e) {
        StringBuilder message = new StringBuilder();
        // The "FATAL EXCEPTION" string is still used on Android even though
        // apps can set a custom UncaughtExceptionHandler that renders uncaught
        // exceptions non-fatal.
        message.append("FATAL EXCEPTION: ").append(threadName).append("\n");
        if (processName != null) {
            message.append("Process: ").append(processName).append(", ");
        }
        message.append("PID: ").append(pid);
        Clog_e(TAG, message.toString(), e);
    }    
}
```


## KillApplicationHandler

如上面的代码所示，app 进程初始化时 `DefaultUncaughtExceptionHandler` 被设置为 `KillApplicationHandler`，如果新线程没有设置 UncaughtExceptionHandler 或者没有替换 DefaultUncaughtExceptionHandler，那么子线程的 Uncaught Exception 也会导致 app 被 killed

KillApplicationHandler 主要干了两件事：
1. 弹出 `异常退出` 对话框，可以让用户选择重启 app
2. 退出 app 进程

```java
private static class KillApplicationHandler implements Thread.UncaughtExceptionHandler {
    private final LoggingHandler mLoggingHandler;
    /**
     * Create a new KillApplicationHandler that follows the given LoggingHandler.
     * If {@link #uncaughtException(Thread, Throwable) uncaughtException} is called
     * on the created instance without {@code loggingHandler} having been triggered,
     * {@link LoggingHandler#uncaughtException(Thread, Throwable)
     * loggingHandler.uncaughtException} will be called first.
     *
     * @param loggingHandler the {@link LoggingHandler} expected to have run before
     *     this instance's {@link #uncaughtException(Thread, Throwable) uncaughtException}
     *     is being called.
     */
    public KillApplicationHandler(LoggingHandler loggingHandler) {
        this.mLoggingHandler = Objects.requireNonNull(loggingHandler);
    }
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        try {
            ensureLogging(t, e);
            // Don't re-enter -- avoid infinite loops if crash-reporting crashes.
            if (mCrashing) return;
            mCrashing = true;
            // Try to end profiling. If a profiler is running at this point, and we kill the
            // process (below), the in-memory buffer will be lost. So try to stop, which will
            // flush the buffer. (This makes method trace profiling useful to debug crashes.)
            if (ActivityThread.currentActivityThread() != null) {
                ActivityThread.currentActivityThread().stopProfiling();
            }
            // Bring up crash dialog, wait for it to be dismissed
            ActivityManager.getService().handleApplicationCrash(
                    mApplicationObject, new ApplicationErrorReport.ParcelableCrashInfo(e));
        } catch (Throwable t2) {
            if (t2 instanceof DeadObjectException) {
                // System process is dead; ignore
            } else {
                try {
                    Clog_e(TAG, "Error reporting crash", t2);
                } catch (Throwable t3) {
                    // Even Clog_e() fails!  Oh well.
                }
            }
        } finally {
            // Try everything to make sure this process goes away.
            Process.killProcess(Process.myPid());
            System.exit(10);
        }
    }
    /**
     * Ensures that the logging handler has been triggered.
     *
     * See b/73380984. This reinstates the pre-O behavior of
     *
     *   {@code thread.getUncaughtExceptionHandler().uncaughtException(thread, e);}
     *
     * logging the exception (in addition to killing the app). This behavior
     * was never documented / guaranteed but helps in diagnostics of apps
     * using the pattern.
     *
     * If this KillApplicationHandler is invoked the "regular" way (by
     * {@link Thread#dispatchUncaughtException(Throwable)
     * Thread.dispatchUncaughtException} in case of an uncaught exception)
     * then the pre-handler (expected to be {@link #mLoggingHandler}) will already
     * have run. Otherwise, we manually invoke it here.
     */
    private void ensureLogging(Thread t, Throwable e) {
        if (!mLoggingHandler.mTriggered) {
            try {
                mLoggingHandler.uncaughtException(t, e);
            } catch (Throwable loggingThrowable) {
                // Ignored.
            }
        }
    }
}
```


## 主线程遇到 UE 时发生了什么

上面在研究子线程时已经发现：Uncaught Exception 会中断字节码的执行流程从而回到 native 代码，主线程在回到 native 代码后选择依次执行 `DetachCurrentThread` 和 `DestroyJavaVM`

```cpp
// zygote 进程的 native 层入口点，app 进程是由 zygote fork 出来的，所以这也算是 app 进程的入口点
// frameworks/base/cmds/app_process/app_main.cpp
int main(int argc, char* const argv[])

// 启动 VM，首次进入 java 层，入口点是 ZygoteInit.main(args)
// frameworks/base/core/jni/AndroidRuntime.cpp 
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote) {
    // ...
    // 启动 VM 后此线程就成为 VM 的主线程，直到 VM 退出后此线程才会结束生命
    // Start VM.  This thread becomes the main thread of the VM, and will not return until the VM exits.
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    // ...
    // ZygoteInit.main(args)
    env->CallStaticVoidMethod(startClass, startMeth, strArray);
    // ...
    // 还记得上面出现过的这行日志吗：D/AndroidRuntime: Shutting down VM
    // 就是在这里打印出来的，此时主线程已经退出了 VM 并准备销毁 VM
    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```

DetachCurrentThread 会调用 `HandleUncaughtExceptions`，这个方法也在上面介绍过了，它会检查是否有 uncaught exception/pending exception，有的话则再次进入 VM 执行 `Thread.dispatchUncaughtException()`，所以主线程的 uncaught exception 也是能够被捕获的

```cpp
// art/runtime/jni/java_vm_ext.cc
static jint DetachCurrentThread(JavaVM* vm) {
  if (vm == nullptr || Thread::Current() == nullptr) {
    return JNI_ERR;
  }
  JavaVMExt* raw_vm = reinterpret_cast<JavaVMExt*>(vm);
  Runtime* runtime = raw_vm->GetRuntime();
  runtime->DetachCurrentThread();
  return JNI_OK;
}

// art/runtime/runtime.cc
void Runtime::DetachCurrentThread() {
  ScopedTrace trace(__FUNCTION__);
  Thread* self = Thread::Current();
  if (self == nullptr) {
    LOG(FATAL) << "attempting to detach thread that is not attached";
  }
  if (self->HasManagedStack()) {
    LOG(FATAL) << *Thread::Current() << " attempting to detach while still running code";
  }
  thread_list_->Unregister(self);
}

ThreadList::Unregister
Thread::Destroy
HandleUncaughtExceptions
```

然后主线程就会把 VM 销毁掉并结束自己的生命周期，但 app 进程并没有结束，还有其他 native thread 的存在，从系统申请的资源如 Surface 也没有释放，所以 app 页面依然存在并没有出现 **崩溃/闪退** 的现象

归属于 app 的窗口没有被回收，那么 input 事件依然会分发给 app，input 事件是需要主线程来消费的，但此时主线程已退出，很明显会阻塞住，所以会触发 ANR

如果用户选择继续等待，app 就变成一个没有 VM 没有主线程的僵尸进程但还没退出，选择确定会发送 SIGKILL 信号杀死 app 进程


## 收集崩溃日志

* DefaultUncaughtExceptionHandler 可以收集到 app 的崩溃日志，也就是主线程的 Uncaught Exception
* 当然它也可以收集到子线程的 Uncaught Exception
* 它可以提高 app 的稳定性，防止 KillApplicationHandler 粗暴地把 app 杀死
* 理论上来说，把崩溃日志写入文件，甚至于即刻上传至服务器都是可以做到的，因为触发 ANR 需要 5s，然后弹出 ANR 对话框直到用户选择杀死 app 也需要几秒钟的时间
* [Shutdown Hook](../../../../2021/06/20/kill-exit/)