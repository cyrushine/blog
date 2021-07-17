
### 线程的状态

```cpp
// dalvik/vm/thread.h
THREAD_UNDEFINED    = -1, /* makes enum compatible with int32_t                                       */
THREAD_ZOMBIE       = 0,  /* TERMINATED (线程死亡，终止运行)                                            */
THREAD_RUNNING      = 1,  /* RUNNABLE or running now (线程可运行或正在运行)                             */
THREAD_TIMED_WAIT   = 2,  /* TIMED_WAITING in Object.wait() (执行了带有超时参数的wait、sleep或join函数) */
THREAD_MONITOR      = 3,  /* BLOCKED on a monitor (线程阻塞，等待获取对象锁)                            */
THREAD_WAIT         = 4,  /* WAITING in Object.wait() (执行了无超时参数的wait函数)                      */
THREAD_INITIALIZING = 5,  /* allocated, not yet running (新建，正在初始化，为其分配资源)                 */
THREAD_STARTING     = 6,  /* started, not yet on thread list (新建，正在启动)                          */
THREAD_NATIVE       = 7,  /* off in a JNI native method (正在执行JNI本地函数)                          */
THREAD_VMWAIT       = 8,  /* waiting on a VM resource (正在等待VM资源)                                 */
THREAD_SUSPENDED    = 9,  /* suspended, usually by GC or debugger (线程暂停，通常是由于GC或debug被暂停) */
```

### Binder 跨进程通讯超时导致的 ANR

主线程执行跨进程通讯（Binder 调用）超时导致的 ANR，解决办法是在子线程执行 Binder 调用（Binder 调用不更新 UI，没必要在主线程执行）

```text
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x73fd76e0 self=0xad33c000
  | sysTid=1085 nice=-10 cgrp=default sched=0/0 handle=0xb1f55494
  | state=S schedstat=( 395268730045 172870501110 604526 ) utm=35915 stm=3611 core=2 HZ=100
  | stack=0xbe74b000-0xbe74d000 stackSize=8MB
  | held mutexes=

  // 进入内核态，在 Binder 驱动里，看到 Binder 线程正阻塞在读取 server 端响应的地方
  kernel: binder_thread_read+0xcb5/0xf88
  kernel: binder_ioctl_write_read.constprop.23+0x1af/0x2e8
  kernel: binder_ioctl+0x2ed/0x580
  kernel: do_vfs_ioctl+0x375/0x518
  kernel: SyS_ioctl+0x47/0x54
  kernel: ret_fast_syscall+0x1/0x54

  // Binder 实现为一个驱动程序，挂载在 /dev/binder，通过系统调用 ioctl 进入 Binder 驱动
  native: #00 pc 000537ec  /system/lib/libc.so (__ioctl+8)
  native: #01 pc 000219d3  /system/lib/libc.so (ioctl+30)
  native: #02 pc 0003d3f5  /system/lib/libbinder.so (android::IPCThreadState::talkWithDriver(bool)+204)

  // 可以看到 isTopOfTask 是同步的方法，并不是异步的（oneway）
  // 所以主线程被阻塞住了，一直在等待 server 端的响应
  native: #03 pc 0003dde3  /system/lib/libbinder.so (android::IPCThreadState::waitForResponse(android::Parcel*, int*)+26)
  native: #04 pc 0003713d  /system/lib/libbinder.so (android::BpBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+36)
  native: #05 pc 000c495f  /system/lib/libandroid_runtime.so (android_os_BinderProxy_transact(_JNIEnv*, _jobject*, int, _jobject*, _jobject*, int)+82)
  at android.os.BinderProxy.transactNative(Native method) 
  at android.os.BinderProxy.transact(Binder.java:1129)

  // 主线程执行了 ActivityManager.isTopOfTask，ActivityManager 的 server 端在进程 system server
  // app 进程需要通过 Binder 与 ActivityManager 进行跨进程通讯
  at android.app.IActivityManager$Stub$Proxy.isTopOfTask(IActivityManager.java:7910)
  at java.lang.reflect.Method.invoke(Native method)
  at leakcanary.ServiceWatcher$install$4$2.invoke(ServiceWatcher.kt:85)
  at java.lang.reflect.Proxy.invoke(Proxy.java:1006)
  at android.app.IActivityManager.isTopOfTask(IActivityManager.java:-2)
  at android.app.Activity.isTopOfTask(Activity.java:6411)

  // Activity.onResume 需要在主线程里执行，到目前为止都算正常
  at android.app.Activity.onResume(Activity.java:1344) 
  at androidx.fragment.app.FragmentActivity.onResume(FragmentActivity.java:433)
  at com.viomi.fridge.vertical.common.base.BaseActivity.onResume(BaseActivity.java:122)
  at android.app.Instrumentation.callActivityOnResume(Instrumentation.java:1412)
  at android.app.Activity.performResume(Activity.java:7292)
  at android.app.ActivityThread.performResumeActivity(ActivityThread.java:3807)
  at android.app.ActivityThread.handleResumeActivity(ActivityThread.java:3847)
  at android.app.servertransaction.ResumeActivityItem.execute(ResumeActivityItem.java:51)
  at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:145)
  at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:70)

  // 可以看到主线程在处理消息队列的消息
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1836)
  at android.os.Handler.dispatchMessage(Handler.java:106)
  at android.os.Looper.loop(Looper.java:193)
  at android.app.ActivityThread.main(ActivityThread.java:6702)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:911)
```