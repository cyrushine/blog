
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
2021-07-19 11:13:29.464 28055-28055/com.example.myapplication D/AndroidRuntime: Shutting down VM

    --------- beginning of crash
2021-07-19 11:13:29.465 28055-28055/com.example.myapplication E/AndroidRuntime: FATAL EXCEPTION: main
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
2021-07-19 11:13:29.471 28055-28055/com.example.myapplication E/cyrus: main-2 UncaughtExceptionHandler: Test Exception
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
2021-07-19 11:17:10.449 1764-28190/? I/ActivityManager: Collecting stacks for pid 28055
2021-07-19 11:17:10.449 1764-28190/? I/system_server: libdebuggerd_client: started dumping process 28055
2021-07-19 11:17:10.450 697-697/? I/tombstoned: registered intercept for pid 28055 and type kDebuggerdJavaBacktrace
2021-07-19 11:17:10.450 28055-28065/com.example.myapplication I/e.myapplicatio: Thread[6,tid=28065,WaitingInMainSignalCatcherLoop,Thread*=0xb400007271841000,peer=0x13780260,"Signal Catcher"]: reacting to signal 3
2021-07-19 11:17:10.528 697-697/? I/tombstoned: received crash request for pid 28055
2021-07-19 11:17:10.528 697-697/? I/tombstoned: found intercept fd 512 for pid 28055 and type kDebuggerdJavaBacktrace
2021-07-19 11:17:10.528 28055-28065/com.example.myapplication I/e.myapplicatio: Wrote stack traces to tombstoned
2021-07-19 11:17:10.529 1764-28190/? I/system_server: libdebuggerd_client: done dumping process 28055

// 需要一段时间来 dump ANR 信息
2021-07-19 11:17:15.839 1764-28190/? E/ActivityManager: ANR in com.example.myapplication (com.example.myapplication/.MainActivity)
    PID: 28055
    Reason: Input dispatching timed out (com.example.myapplication/com.example.myapplication.MainActivity, 7fc1f1 com.example.myapplication/com.example.myapplication.MainActivity (server) is not responding. Waited 8001ms for MotionEvent(action=DOWN))
    Parent: com.example.myapplication/.MainActivity
    Load: 0.13 / 0.47 / 0.83
    ----- Output from /proc/pressure/memory -----
    some avg10=0.00 avg60=0.00 avg300=0.00 total=15206960
    full avg10=0.00 avg60=0.00 avg300=0.00 total=5127529
    ----- End output from /proc/pressure/memory -----
    
    CPU usage from 0ms to 5796ms later (2021-07-19 11:17:09.982 to 2021-07-19 11:17:15.778):
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
2021-07-19 11:17:15.839 1764-28190/? E/ActivityManager:   0.1% 807/android.hardware.health@2.1-service: 0% user + 0.1% kernel / faults: 9 minor
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
    CPU usage from 41ms to 439ms later (2021-07-19 11:17:10.023 to 2021-07-19 11:17:10.422) with 99% awake:
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
2021-07-19 11:18:58.935 1764-1789/? I/ActivityManager: Killing 28055:com.example.myapplication/u0a161 (adj 0): user request after error
2021-07-19 11:18:58.937 1764-1789/? I/Process: PerfMonitor : current process sending signal quiet. PID: 28055 SIG: 9
2021-07-19 11:18:58.938 1764-1802/? I/Process: PerfMonitor : current process killing process group. PID: 28055
2021-07-19 11:18:58.965 706-706/? I/Zygote: Process 28055 exited due to signal 9 (Killed)
2021-07-19 11:18:58.966 1764-1802/? I/libprocessgroup: Successfully killed process cgroup uid 10161 pid 28055 in 28ms
2021-07-19 11:18:58.970 882-882/? I/vendor.qti.hardware.servicetracker@1.2-service: killProcess is called for pid : 28055
2021-07-19 11:18:58.970 1764-5448/? W/ANRStateManager: clear state, but process isn't exist. hash=92035998 uid=10161 pid=28055 state=16
```


### 子线程且 Thread.uncaughtExceptionHandler != null

app 没有发生 ANR 也没有崩溃，且无论 `DefaultUncaughtExceptionHandler` 是否为 null，`Thread.uncaughtExceptionHandler` 都能够有限捕获异常

```log
2021-07-19 17:20:41.503 20615-22015/com.example.myapplication E/AndroidRuntime: FATAL EXCEPTION: Thread-10
    Process: com.example.myapplication, PID: 20615
    java.lang.RuntimeException: Test Exception
        at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:75)
        at com.example.myapplication.MainActivity$onCreate$4$1.invoke(MainActivity.kt:38)
        at com.example.myapplication.MainActivity$onCreate$4$1.invoke(MainActivity.kt:36)
        at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)

2021-07-19 17:20:41.509 20615-22015/com.example.myapplication E/cyrus: Thread-10-3890 UncaughtExceptionHandler: Test Exception
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
2021-07-19 11:06:58.817 27816-27929/com.example.myapplication E/AndroidRuntime: FATAL EXCEPTION: Thread-3
    Process: com.example.myapplication, PID: 27816
    java.lang.RuntimeException: Test Exception
        at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:58)
        at com.example.myapplication.MainActivity$onCreate$2$1.invoke(MainActivity.kt:24)
        at com.example.myapplication.MainActivity$onCreate$2$1.invoke(MainActivity.kt:22)
        at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)

// DefaultUncaughtExceptionHandler != null 则能够捕获异常
2021-07-19 17:03:51.701 20615-20884/com.example.myapplication E/cyrus: Thread-2-3882 DefaultUncaughtExceptionHandler: Test Exception
    java.lang.RuntimeException: Test Exception
        at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:75)
        at com.example.myapplication.MainActivity$onCreate$2$1.invoke(MainActivity.kt:24)
        at com.example.myapplication.MainActivity$onCreate$2$1.invoke(MainActivity.kt:22)
        at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)        

// 当 DefaultUncaughtExceptionHandler == null 时异常被输出到标准异常流
2021-07-19 17:13:54.170 20615-21405/com.example.myapplication W/System.err: Exception in thread "Thread-8" java.lang.RuntimeException: Test Exception
2021-07-19 17:13:54.171 20615-21405/com.example.myapplication W/System.err:     at com.example.myapplication.MainActivityKt.throwException(MainActivity.kt:75)
2021-07-19 17:13:54.171 20615-21405/com.example.myapplication W/System.err:     at com.example.myapplication.MainActivity$onCreate$1$1.invoke(MainActivity.kt:17)
2021-07-19 17:13:54.171 20615-21405/com.example.myapplication W/System.err:     at com.example.myapplication.MainActivity$onCreate$1$1.invoke(MainActivity.kt:15)
2021-07-19 17:13:54.171 20615-21405/com.example.myapplication W/System.err:     at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)   
```


### 子线程且为默认 DefaultUncaughtExceptionHandler 

默认的 DefaultUncaughtExceptionHandler 是 KillApplicationHandler，它会杀死 app

```log
// 除了有上一节的日志外，还会有以下日志并且 app 被 kill
// app process 收到信号 SIGKILL(9) 被迫退出，app 立刻崩溃掉
2021-07-19 11:06:58.870 27816-27929/com.example.myapplication I/Process: Sending signal. PID: 27816 SIG: 9
2021-07-19 11:06:58.910 1764-1802/? I/Process: PerfMonitor : current process killing process group. PID: 27816
2021-07-19 11:06:58.911 1764-2936/? I/ActivityManager: Process com.example.myapplication (pid 27816) has died: prcp CRE 
2021-07-19 11:06:58.911 706-706/? I/Zygote: Process 27816 exited due to signal 9 (Killed)
2021-07-19 11:06:58.912 1764-1802/? I/libprocessgroup: Successfully killed process cgroup uid 10161 pid 27816 in 0ms
2021-07-19 11:06:58.917 1764-2936/? W/ANRStateManager: clear state, but process isn't exist. hash=224220282 uid=10161 pid=27816 state=16
2021-07-19 11:06:58.919 882-8554/? I/vendor.qti.hardware.servicetracker@1.2-service: killProcess is called for pid : 27816
```


### 总结

| 所在线程 | 子条件 | 结果 |
|---------|--------|------|
| 主线程 | | 始终会被 blocked 住，然后发生 ANR，最后被杀死 |
| 子线程 | 两个 UncaughtExceptionHandler 都置空 <br> 有任意一个自定义的 UncaughtExceptionHandler | app 没事 |
| | 默认 | KillApplicationHandler 捕获到异常并杀死 app |