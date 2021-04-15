---
title: 浅析 LeakCanary
date: 2021-04-12 12:00:00 +0800
categories: [内存优化]
tags: [LeakCanary, 内存]
---

## 检测泄漏对象

### 四种引用类型

| 引用类型                  | 什么时候会被 GC                                              |
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

### 检测 `Activity` 泄漏

通过 `ActivityLifecycleCallbacks.onActivityDestroyed` 可以收集到 destoryed `Activity`，这些 `Activity` 已走完它的生命周期，应该被后续的 GC 回收掉

用 `WeakReference` + `ReferenceQueue` 监控它，并用 `watchedObjects = mutableMapOf<String, KeyedWeakReference>()` 把它记录起来，key 是 UUID，value 是对这个 `Activity` 的弱引用；等待 5s，让 GC 线程有足够的机会去发现并回收这个 `Activity` 对象，如果 5s 后 `Activity` 仍然没有被 GC（没有出现在引用队列里），那么可以证明这个对象发生了内存泄漏，被强引用导致存活超过它的生命周期

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

// 弱引用这个 Activity 并设置在 5s 后检查此对象有没被 GC
@Synchronized override fun expectWeaklyReachable(
  watchedObject: Any,
  description: String
) {
  if (!isEnabled()) {
    return
  }
  removeWeaklyReachableObjects()
  val key = UUID.randomUUID()
    .toString()
  val watchUptimeMillis = clock.uptimeMillis()
  val reference =
    KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
  SharkLog.d {
    "Watching " +
      (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
      (if (description.isNotEmpty()) " ($description)" else "") +
      " with key $key"
  }
  watchedObjects[key] = reference
  checkRetainedExecutor.execute {
    moveToRetained(key)
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

拓展一下，上面检测 `Activity` 泄漏的机制：监控 - 等待 - 检查，其实可以应用到任何对象，包括 `Fragment`、`View` 等，于是抽象出一个通用的对象泄漏检测工具：`ObjectWatcher`，上面的 `expectWeaklyReachable` 和 `moveToRetained` 都是在 `ObjectWatcher` 里实现的

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

## 堆转储

利用 `Debug#dumpHprofData(fileName)`（但是生成的 hprof 文件很大）

## 解析堆转储文件

利用 [Shark](https://square.github.io/leakcanary/shark/)（性能是个问题）

## 组织并输出泄漏问题

通过 [Shark](https://square.github.io/leakcanary/shark/) 找到泄漏对象到 GC Root 的路径

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