---
title: Matrix - ResourcesCanary 浅析
date: 2021-04-23 12:00:00 +0800
categories: [android, 内存]
tags: [内存优化，OOM]
---

## `Activity` 的泄漏检测

跟 [LeakCanary](https://github.com/square/leakcanary) 一样，使用 `ActivityLifecycleCallbacks` 监听 `Activity.onDestroy()` 事件，从而收集到已销毁的 `Activity` 对象

```java
public class ResourcePlugin extends Plugin {
    @Override
    public void start() {
        super.start();
        if (!isSupported()) {
            MatrixLog.e(TAG, "ResourcePlugin start, ResourcePlugin is not supported, just return");
            return;
        }
        mWatcher.start();
    }    
}

public class ActivityRefWatcher extends FilePublisher implements Watcher {

    private final Application.ActivityLifecycleCallbacks mRemovedActivityMonitor = new ActivityLifeCycleCallbacksAdapter() {
        @Override
        public void onActivityDestroyed(Activity activity) {
            pushDestroyedActivityInfo(activity);
            mHandler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    triggerGc();
                }
            }, 2000);
        }
    };

    @Override
    public void start() {
        stopDetect();
        final Application app = mResourcePlugin.getApplication();
        if (app != null) {
            app.registerActivityLifecycleCallbacks(mRemovedActivityMonitor);
            scheduleDetectProcedure();
            MatrixLog.i(TAG, "watcher is started.");
        }
    }    
}
```

跟 `LeakCanary` 不同的是，`ResourceCanary` 并没有采用 `WeakReference` + `ReferenceQueue` 的组合来监控 GC，而是用 `WeakReference` 软引用 `Activity` 并放在 `ConcurrentLinkedQueue` 里，然后专门开一个子线程监控并处理队列，开发者的解释是：

> 可见其对 `Activity` 是否泄漏的判断依赖 VM 会将可回收的对象加入 `WeakReference` 关联的 `ReferenceQueue` 这一特性，在 Demo 的测试过程中我们发现这中做法在个别系统上可能存在误报，原因大致如下：
> 
> 1. VM 并没有提供强制触发 GC 的 API，通过 `System.gc()` 或 `Runtime.getRuntime().gc()` 只能“建议”系统进行 GC，如果系统忽略了我们的 GC 请求，可回收的对象就不会被加入 `ReferenceQueue`
> 2. 将可回收对象加入 `ReferenceQueue` 需要等待一段时间，`LeakCanary`采用延时 100ms 的做法加以规避，但似乎并不绝对管用
> 3. 监测逻辑是异步的，如果判断 `Activity` 是否可回收时某个 `Activity` 正好还被某个方法的局部变量持有，就会引起误判
> 4. 若反复进入泄漏的 `Activity`，`LeakCanary` 会重复提示该Activity已泄漏
> 
> 对此我们做了以下改进：
> 
> 1. 增加一个一定能被回收的“哨兵”对象，用来确认系统确实进行了 GC
> 2. 直接通过 `WeakReference.get()` 来判断对象是否已被回收，避免因延迟导致误判
> 3. 若发现某个 `Activity` 无法被回收，再重复判断 3 次，且要求从该 `Activity` 被记录起有2个以上的 `Activity` 被创建才认为是泄漏，以防在判断时该 `Activity` 被局部变量持有导致误判
> 4. 对已判断为泄漏的 `Activity`，记录其类名，避免重复提示该 `Activity` 已泄漏

```java
public class ActivityRefWatcher extends FilePublisher implements Watcher {

    private final ConcurrentLinkedQueue<DestroyedActivityInfo> mDestroyedActivityInfos;

    // 处理上面的队列
    private final RetryableTask mScanDestroyedActivitiesTask = new RetryableTask() {
        @Override
        public Status execute() {
            // ...
        }
    };    

    // 销毁的 Activity 被放到上面的队列里
    private void pushDestroyedActivityInfo(Activity activity) {
        final String activityName = activity.getClass().getName();
        if ((mDumpHprofMode == ResourceConfig.DumpMode.NO_DUMP || mDumpHprofMode == ResourceConfig.DumpMode.AUTO_DUMP)
                && !mResourcePlugin.getConfig().getDetectDebugger()
                && isPublished(activityName)) {
            MatrixLog.i(TAG, "activity leak with name %s had published, just ignore", activityName);
            return;
        }
        final UUID uuid = UUID.randomUUID();
        final StringBuilder keyBuilder = new StringBuilder();
        keyBuilder.append(ACTIVITY_REFKEY_PREFIX).append(activityName)
                .append('_').append(Long.toHexString(uuid.getMostSignificantBits())).append(Long.toHexString(uuid.getLeastSignificantBits()));
        final String key = keyBuilder.toString();
        final DestroyedActivityInfo destroyedActivityInfo
                = new DestroyedActivityInfo(key, activity, activityName);
        mDestroyedActivityInfos.add(destroyedActivityInfo);
        synchronized (mDestroyedActivityInfos) {
            mDestroyedActivityInfos.notifyAll();
        }
        MatrixLog.d(TAG, "mDestroyedActivityInfos add %s", activityName);
    }    

    // 在子线程里处理 Activity 队列
    public void start() {
        stopDetect();
        final Application app = mResourcePlugin.getApplication();
        if (app != null) {
            app.registerActivityLifecycleCallbacks(mRemovedActivityMonitor);
            scheduleDetectProcedure();
            MatrixLog.i(TAG, "watcher is started.");
        }
    }
    private void scheduleDetectProcedure() {
        mDetectExecutor.executeInBackground(mScanDestroyedActivitiesTask);
    }
}
```

`LeakCanary` 每当一个 `Activity` 销毁时就放一个延时 5s 的检查任务，而 `ResourceCanary` 检测规则较为宽松：

1. 默认每隔 1min 检查一次 `mDestroyedActivityInfos`
2. 规定在检查次数 `mMaxRedetectTimes` 以下不判定为泄漏

```java
public Status execute() {
    // If destroyed activity list is empty, just wait to save power.
    if (mDestroyedActivityInfos.isEmpty()) {
        MatrixLog.i(TAG, "DestroyedActivityInfo is empty! wait...");
        synchronized (mDestroyedActivityInfos) {
            try {
                mDestroyedActivityInfos.wait();
            } catch (Throwable ignored) {
                // Ignored.
            }
        }
        MatrixLog.i(TAG, "DestroyedActivityInfo is NOT empty! resume check");
        return Status.RETRY;
    }
    // Fake leaks will be generated when debugger is attached.
    if (Debug.isDebuggerConnected() && !mResourcePlugin.getConfig().getDetectDebugger()) {
        MatrixLog.w(TAG, "debugger is connected, to avoid fake result, detection was delayed.");
        return Status.RETRY;
    }
//          final WeakReference<Object[]> sentinelRef = new WeakReference<>(new Object[1024 * 1024]); // alloc big object
    triggerGc();
    triggerGc();
    triggerGc();
//          if (sentinelRef.get() != null) {
//              // System ignored our gc request, we will retry later.
//              MatrixLog.d(TAG, "system ignore our gc request, wait for next detection.");
//              return Status.RETRY;
//          }
    final Iterator<DestroyedActivityInfo> infoIt = mDestroyedActivityInfos.iterator();
    while (infoIt.hasNext()) {
        final DestroyedActivityInfo destroyedActivityInfo = infoIt.next();
        if ((mDumpHprofMode == ResourceConfig.DumpMode.NO_DUMP || mDumpHprofMode == ResourceConfig.DumpMode.AUTO_DUMP)
                && !mResourcePlugin.getConfig().getDetectDebugger()
                && isPublished(destroyedActivityInfo.mActivityName)) {
            MatrixLog.v(TAG, "activity with key [%s] was already published.", destroyedActivityInfo.mActivityName);
            infoIt.remove();
            continue;
        }
        triggerGc();
        if (destroyedActivityInfo.mActivityRef.get() == null) {
            // The activity was recycled by a gc triggered outside.
            MatrixLog.v(TAG, "activity with key [%s] was already recycled.", destroyedActivityInfo.mKey);
            infoIt.remove();
            continue;
        }
        ++destroyedActivityInfo.mDetectedCount;
        if (destroyedActivityInfo.mDetectedCount < mMaxRedetectTimes
                && !mResourcePlugin.getConfig().getDetectDebugger()) {
            // Although the sentinel tell us the activity should have been recycled,
            // system may still ignore it, so try again until we reach max retry times.
            MatrixLog.i(TAG, "activity with key [%s] should be recycled but actually still exists in %s times, wait for next detectiontoconfirm.",
                    destroyedActivityInfo.mKey, destroyedActivityInfo.mDetectedCount);
            triggerGc();
            continue;
        }
        MatrixLog.i(TAG, "activity with key [%s] was suspected to be a leaked instance. mode[%s]", destroyedActivityInfo.mKey,mDumpHprofMode;
        if (mLeakProcessor == null) {
            throw new NullPointerException("LeakProcessor not found!!!");
        }
        triggerGc();
        if (mLeakProcessor.process(destroyedActivityInfo)) {
            MatrixLog.i(TAG, "the leaked activity [%s] with key [%s] has been processed. stop polling", destroyedActivityInfomActivityName,destroyedActivityInfo.mKey);
            infoIt.remove();
        }
    }
    triggerGc();
    return Status.RETRY;
}
```

## heap dump 并分析 hprof 文件

这个阶段 `ResourcesCanary` 的逻辑跟 `LeakCanary` 几乎是一模一样的，估计是 copy 代码过来的，只不过 `ResourcesCanary` 用的还是 [haha](https://github.com/square/haha) 而最新的 `LeakCanary` 已经使用 `Shark` 了


## 总结

对比 `LeakCanary`，`ResourcesCanary` 的优点是提出了子线程检查泄漏对象的思路，缺点是检测对象太单一，只有 `Activity`（重复 `Bitmap` 检测感觉并不可靠），而且没有抽象出通用的泄漏检测逻辑（`LeakCanary` 抽象出 `ObjectWatcher`）