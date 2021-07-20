
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

下面仅考虑 `Thread.uncaughtExceptionHandler`、`DefaultUncaughtExceptionHandler` 和`主线程`三个因素


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


### 子线程且 Thread.uncaughtExceptionHandler != null

app 没有发生 ANR 也没有崩溃，且无论 `DefaultUncaughtExceptionHandler` 是否为 null，`Thread.uncaughtExceptionHandler` 都能够有限捕获异常

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


### 子线程且两个 UncaughtExceptionHandler 都置空 or DefaultUncaughtExceptionHandler != null

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


### 子线程且为默认 DefaultUncaughtExceptionHandler 

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


## Show Me The Code

### Throw Exception 时发生了什么

java 层发生 uncaught exception 相当于调用了 JNIEnv->Throw，这个方法的实现很简单，就是把 exception 记录在 Thread::tlsPtr_::exception

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


### 从 Thread 的生命周期开始

接下来我猜想埋点在代码里的异常检查流程在发现 pending exception != null 后，会中断字节码的执行并退出 Thread.run()，从而回到 native thread 代码

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


### 找到 Uncaught Exception 的入口点 dispatchUncaughtException

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


## 谁打印了 AndroidRuntime: FATAL EXCEPTION

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


## 为什么主线程发生 Uncaught Exception 会被 blocked 而不能恢复

```cpp
// frameworks/base/cmds/app_process/app_main.cpp
// zygote 进程的 native 层入口点，app 进程是由 zygote fork 出来的，所以这也算是 app 进程的入口点
int main(int argc, char* const argv[])
{
    if (!LOG_NDEBUG) {
      String8 argv_String;
      for (int i = 0; i < argc; ++i) {
        argv_String.append("\"");
        argv_String.append(argv[i]);
        argv_String.append("\" ");
      }
      ALOGV("app_process main with argv: %s", argv_String.string());
    }

    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;

    // Everything up to '--' or first non '-' arg goes to the vm.
    //
    // The first argument after the VM args is the "parent dir", which
    // is currently unused.
    //
    // After the parent dir, we expect one or more the following internal
    // arguments :
    //
    // --zygote : Start in zygote mode
    // --start-system-server : Start the system server.
    // --application : Start in application (stand alone, non zygote) mode.
    // --nice-name : The nice name for this process.
    //
    // For non zygote starts, these arguments will be followed by
    // the main class name. All remaining arguments are passed to
    // the main method of this class.
    //
    // For zygote starts, all remaining arguments are passed to the zygote.
    // main function.
    //
    // Note that we must copy argument string values since we will rewrite the
    // entire argument block when we apply the nice name to argv0.
    //
    // As an exception to the above rule, anything in "spaced commands"
    // goes to the vm even though it has a space in it.
    const char* spaced_commands[] = { "-cp", "-classpath" };
    // Allow "spaced commands" to be succeeded by exactly 1 argument (regardless of -s).
    bool known_command = false;

    int i;
    for (i = 0; i < argc; i++) {
        if (known_command == true) {
          runtime.addOption(strdup(argv[i]));
          // The static analyzer gets upset that we don't ever free the above
          // string. Since the allocation is from main, leaking it doesn't seem
          // problematic. NOLINTNEXTLINE
          ALOGV("app_process main add known option '%s'", argv[i]);
          known_command = false;
          continue;
        }

        for (int j = 0;
             j < static_cast<int>(sizeof(spaced_commands) / sizeof(spaced_commands[0]));
             ++j) {
          if (strcmp(argv[i], spaced_commands[j]) == 0) {
            known_command = true;
            ALOGV("app_process main found known command '%s'", argv[i]);
          }
        }

        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }

        runtime.addOption(strdup(argv[i]));
        // The static analyzer gets upset that we don't ever free the above
        // string. Since the allocation is from main, leaking it doesn't seem
        // problematic. NOLINTNEXTLINE
        ALOGV("app_process main add option '%s'", argv[i]);
    }

    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);

        if (!LOG_NDEBUG) {
          String8 restOfArgs;
          char* const* argv_new = argv + i;
          int argc_new = argc - i;
          for (int k = 0; k < argc_new; ++k) {
            restOfArgs.append("\"");
            restOfArgs.append(argv_new[k]);
            restOfArgs.append("\" ");
          }
          ALOGV("Class name = %s, args = %s", className.string(), restOfArgs.string());
        }
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}

/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 *
 * Passes the main function two arguments, the class name and the specified
 * options string.
 *
 * 执行 ZygoteInit.main(args)
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());

    static const String8 startSystemServer("start-system-server");
    // Whether this is the primary zygote, meaning the zygote which will fork system server.
    bool primary_zygote = false;

    /*
     * 'startSystemServer == true' means runtime is obsolete and not run from
     * init.rc anymore, so we print out the boot start event here.
     */
    for (size_t i = 0; i < options.size(); ++i) {
        if (options[i] == startSystemServer) {
            primary_zygote = true;
           /* track our progress through the boot sequence */
           const int LOG_BOOT_PROGRESS_START = 3000;
           LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
        }
    }

    const char* rootDir = getenv("ANDROID_ROOT");
    if (rootDir == NULL) {
        rootDir = "/system";
        if (!hasDir("/system")) {
            LOG_FATAL("No root directory specified, and /system does not exist.");
            return;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }

    const char* artRootDir = getenv("ANDROID_ART_ROOT");
    if (artRootDir == NULL) {
        LOG_FATAL("No ART directory specified with ANDROID_ART_ROOT environment variable.");
        return;
    }

    const char* i18nRootDir = getenv("ANDROID_I18N_ROOT");
    if (i18nRootDir == NULL) {
        LOG_FATAL("No runtime directory specified with ANDROID_I18N_ROOT environment variable.");
        return;
    }

    const char* tzdataRootDir = getenv("ANDROID_TZDATA_ROOT");
    if (tzdataRootDir == NULL) {
        LOG_FATAL("No tz data directory specified with ANDROID_TZDATA_ROOT environment variable.");
        return;
    }

    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);

    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    onVmCreated(env);

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```

```java
// 启动 zygote server，监听并处理 fork app 进程的请求
class ZygoteInit {
    public static void main(String[] argv) {
        ZygoteServer zygoteServer = null;

        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation();

        // Zygote goes into its own process group.
        try {
            Os.setpgid(0, 0);
        } catch (ErrnoException ex) {
            throw new RuntimeException("Failed to setpgid(0,0)", ex);
        }

        Runnable caller;
        try {
            // Store now for StatsLogging later.
            final long startTime = SystemClock.elapsedRealtime();
            final boolean isRuntimeRestarted = "1".equals(
                    SystemProperties.get("sys.boot_completed"));

            String bootTimeTag = Process.is64Bit() ? "Zygote64Timing" : "Zygote32Timing";
            TimingsTraceLog bootTimingsTraceLog = new TimingsTraceLog(bootTimeTag,
                    Trace.TRACE_TAG_DALVIK);
            bootTimingsTraceLog.traceBegin("ZygoteInit");
            RuntimeInit.preForkInit();

            boolean startSystemServer = false;
            String zygoteSocketName = "zygote";
            String abiList = null;
            boolean enableLazyPreload = false;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if ("--enable-lazy-preload".equals(argv[i])) {
                    enableLazyPreload = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);
            if (!isRuntimeRestarted) {
                if (isPrimaryZygote) {
                    FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
                            BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__ZYGOTE_INIT_START,
                            startTime);
                } else if (zygoteSocketName.equals(Zygote.SECONDARY_SOCKET_NAME)) {
                    FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
                            BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__SECONDARY_ZYGOTE_INIT_START,
                            startTime);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }

            // In some configurations, we avoid preloading resources and classes eagerly.
            // In such cases, we will preload things prior to our first fork.
            if (!enableLazyPreload) {
                bootTimingsTraceLog.traceBegin("ZygotePreload");
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                        SystemClock.uptimeMillis());
                preload(bootTimingsTraceLog);
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                        SystemClock.uptimeMillis());
                bootTimingsTraceLog.traceEnd(); // ZygotePreload
            }

            // Do an initial gc to clean up after startup
            bootTimingsTraceLog.traceBegin("PostZygoteInitGC");
            gcAndFinalize();
            bootTimingsTraceLog.traceEnd(); // PostZygoteInitGC

            bootTimingsTraceLog.traceEnd(); // ZygoteInit

            Zygote.initNativeState(isPrimaryZygote);

            ZygoteHooks.stopZygoteNoThreadCreation();

            zygoteServer = new ZygoteServer(isPrimaryZygote);

            if (startSystemServer) {
                Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

                // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
                // child (system_server) process.
                if (r != null) {
                    r.run();
                    return;
                }
            }

            Log.i(TAG, "Accepting command socket connections");

            // The select loop returns early in the child process after a fork and
            // loops forever in the zygote.
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with exception", ex);
            throw ex;
        } finally {
            if (zygoteServer != null) {
                zygoteServer.closeServerSocket();
            }
        }

        // We're in the child process and have exited the select loop. Proceed to execute the
        // command.
        if (caller != null) {
            caller.run();
        }
    }
}

class ZygoteServer {
    /**
     * Runs the zygote process's select loop. Accepts new connections as
     * they happen, and reads commands from connections one spawn-request's
     * worth at a time.
     * @param abiList list of ABIs supported by this zygote.
     */
    Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
        ArrayList<ZygoteConnection> peers = new ArrayList<>();

        socketFDs.add(mZygoteSocket.getFileDescriptor());
        peers.add(null);

        mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;

        while (true) {
            fetchUsapPoolPolicyPropsWithMinInterval();
            mUsapPoolRefillAction = UsapPoolRefillAction.NONE;

            int[] usapPipeFDs = null;
            StructPollfd[] pollFDs;

            // Allocate enough space for the poll structs, taking into account
            // the state of the USAP pool for this Zygote (could be a
            // regular Zygote, a WebView Zygote, or an AppZygote).
            if (mUsapPoolEnabled) {
                usapPipeFDs = Zygote.getUsapPipeFDs();
                pollFDs = new StructPollfd[socketFDs.size() + 1 + usapPipeFDs.length];
            } else {
                pollFDs = new StructPollfd[socketFDs.size()];
            }

            /*
             * For reasons of correctness the USAP pool pipe and event FDs
             * must be processed before the session and server sockets.  This
             * is to ensure that the USAP pool accounting information is
             * accurate when handling other requests like API deny list
             * exemptions.
             */

            int pollIndex = 0;
            for (FileDescriptor socketFD : socketFDs) {
                pollFDs[pollIndex] = new StructPollfd();
                pollFDs[pollIndex].fd = socketFD;
                pollFDs[pollIndex].events = (short) POLLIN;
                ++pollIndex;
            }

            final int usapPoolEventFDIndex = pollIndex;

            if (mUsapPoolEnabled) {
                pollFDs[pollIndex] = new StructPollfd();
                pollFDs[pollIndex].fd = mUsapPoolEventFD;
                pollFDs[pollIndex].events = (short) POLLIN;
                ++pollIndex;

                // The usapPipeFDs array will always be filled in if the USAP Pool is enabled.
                assert usapPipeFDs != null;
                for (int usapPipeFD : usapPipeFDs) {
                    FileDescriptor managedFd = new FileDescriptor();
                    managedFd.setInt$(usapPipeFD);

                    pollFDs[pollIndex] = new StructPollfd();
                    pollFDs[pollIndex].fd = managedFd;
                    pollFDs[pollIndex].events = (short) POLLIN;
                    ++pollIndex;
                }
            }

            int pollTimeoutMs;

            if (mUsapPoolRefillTriggerTimestamp == INVALID_TIMESTAMP) {
                pollTimeoutMs = -1;
            } else {
                long elapsedTimeMs = System.currentTimeMillis() - mUsapPoolRefillTriggerTimestamp;

                if (elapsedTimeMs >= mUsapPoolRefillDelayMs) {
                    // The refill delay has elapsed during the period between poll invocations.
                    // We will now check for any currently ready file descriptors before refilling
                    // the USAP pool.
                    pollTimeoutMs = 0;
                    mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
                    mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

                } else if (elapsedTimeMs <= 0) {
                    // This can occur if the clock used by currentTimeMillis is reset, which is
                    // possible because it is not guaranteed to be monotonic.  Because we can't tell
                    // how far back the clock was set the best way to recover is to simply re-start
                    // the respawn delay countdown.
                    pollTimeoutMs = mUsapPoolRefillDelayMs;

                } else {
                    pollTimeoutMs = (int) (mUsapPoolRefillDelayMs - elapsedTimeMs);
                }
            }

            int pollReturnValue;
            try {
                pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }

            if (pollReturnValue == 0) {
                // The poll returned zero results either when the timeout value has been exceeded
                // or when a non-blocking poll is issued and no FDs are ready.  In either case it
                // is time to refill the pool.  This will result in a duplicate assignment when
                // the non-blocking poll returns zero results, but it avoids an additional
                // conditional in the else branch.
                mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
                mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

            } else {
                boolean usapPoolFDRead = false;

                while (--pollIndex >= 0) {
                    if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                        continue;
                    }

                    if (pollIndex == 0) {
                        // Zygote server socket
                        ZygoteConnection newPeer = acceptCommandPeer(abiList);
                        peers.add(newPeer);
                        socketFDs.add(newPeer.getFileDescriptor());
                    } else if (pollIndex < usapPoolEventFDIndex) {
                        // Session socket accepted from the Zygote server socket

                        try {
                            ZygoteConnection connection = peers.get(pollIndex);
                            boolean multipleForksOK = !isUsapPoolEnabled()
                                    && ZygoteHooks.isIndefiniteThreadSuspensionSafe();
                            final Runnable command =
                                    connection.processCommand(this, multipleForksOK);

                            // TODO (chriswailes): Is this extra check necessary?
                            if (mIsForkChild) {
                                // We're in the child. We should always have a command to run at
                                // this stage if processCommand hasn't called "exec".
                                if (command == null) {
                                    throw new IllegalStateException("command == null");
                                }

                                return command;
                            } else {
                                // We're in the server - we should never have any commands to run.
                                if (command != null) {
                                    throw new IllegalStateException("command != null");
                                }

                                // We don't know whether the remote side of the socket was closed or
                                // not until we attempt to read from it from processCommand. This
                                // shows up as a regular POLLIN event in our regular processing
                                // loop.
                                if (connection.isClosedByPeer()) {
                                    connection.closeSocket();
                                    peers.remove(pollIndex);
                                    socketFDs.remove(pollIndex);
                                }
                            }
                        } catch (Exception e) {
                            if (!mIsForkChild) {
                                // We're in the server so any exception here is one that has taken
                                // place pre-fork while processing commands or reading / writing
                                // from the control socket. Make a loud noise about any such
                                // exceptions so that we know exactly what failed and why.

                                Slog.e(TAG, "Exception executing zygote command: ", e);

                                // Make sure the socket is closed so that the other end knows
                                // immediately that something has gone wrong and doesn't time out
                                // waiting for a response.
                                ZygoteConnection conn = peers.remove(pollIndex);
                                conn.closeSocket();

                                socketFDs.remove(pollIndex);
                            } else {
                                // We're in the child so any exception caught here has happened post
                                // fork and before we execute ActivityThread.main (or any other
                                // main() method). Log the details of the exception and bring down
                                // the process.
                                Log.e(TAG, "Caught post-fork exception in child process.", e);
                                throw e;
                            }
                        } finally {
                            // Reset the child flag, in the event that the child process is a child-
                            // zygote. The flag will not be consulted this loop pass after the
                            // Runnable is returned.
                            mIsForkChild = false;
                        }

                    } else {
                        // Either the USAP pool event FD or a USAP reporting pipe.

                        // If this is the event FD the payload will be the number of USAPs removed.
                        // If this is a reporting pipe FD the payload will be the PID of the USAP
                        // that was just specialized.  The `continue` statements below ensure that
                        // the messagePayload will always be valid if we complete the try block
                        // without an exception.
                        long messagePayload;

                        try {
                            byte[] buffer = new byte[Zygote.USAP_MANAGEMENT_MESSAGE_BYTES];
                            int readBytes =
                                    Os.read(pollFDs[pollIndex].fd, buffer, 0, buffer.length);

                            if (readBytes == Zygote.USAP_MANAGEMENT_MESSAGE_BYTES) {
                                DataInputStream inputStream =
                                        new DataInputStream(new ByteArrayInputStream(buffer));

                                messagePayload = inputStream.readLong();
                            } else {
                                Log.e(TAG, "Incomplete read from USAP management FD of size "
                                        + readBytes);
                                continue;
                            }
                        } catch (Exception ex) {
                            if (pollIndex == usapPoolEventFDIndex) {
                                Log.e(TAG, "Failed to read from USAP pool event FD: "
                                        + ex.getMessage());
                            } else {
                                Log.e(TAG, "Failed to read from USAP reporting pipe: "
                                        + ex.getMessage());
                            }

                            continue;
                        }

                        if (pollIndex > usapPoolEventFDIndex) {
                            Zygote.removeUsapTableEntry((int) messagePayload);
                        }

                        usapPoolFDRead = true;
                    }
                }

                if (usapPoolFDRead) {
                    int usapPoolCount = Zygote.getUsapPoolCount();

                    if (usapPoolCount < mUsapPoolSizeMin) {
                        // Immediate refill
                        mUsapPoolRefillAction = UsapPoolRefillAction.IMMEDIATE;
                    } else if (mUsapPoolSizeMax - usapPoolCount >= mUsapPoolRefillThreshold) {
                        // Delayed refill
                        mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
                    }
                }
            }

            if (mUsapPoolRefillAction != UsapPoolRefillAction.NONE) {
                int[] sessionSocketRawFDs =
                        socketFDs.subList(1, socketFDs.size())
                                .stream()
                                .mapToInt(FileDescriptor::getInt$)
                                .toArray();

                final boolean isPriorityRefill =
                        mUsapPoolRefillAction == UsapPoolRefillAction.IMMEDIATE;

                final Runnable command =
                        fillUsapPool(sessionSocketRawFDs, isPriorityRefill);

                if (command != null) {
                    return command;
                } else if (isPriorityRefill) {
                    // Schedule a delayed refill to finish refilling the pool.
                    mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
                }
            }
        }
    }
}
```