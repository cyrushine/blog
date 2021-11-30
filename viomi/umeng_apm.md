# 背景

集成友盟 umeng 主要是想用它的 ANR 统计功能（Bugly 的 ANR 统计下线了）

前置条件：移除 Bugly 和 xCrash，因为它们有可能与友盟 APM 冲突（猜测都是通过注册 SIGQUIT 处理器来捕获 ANR，友盟和 Bugly 不开源），分支是 `local/umeng_anr`

因为友盟 APM 不开源也没有详细的教程描述日志，所以只能从大堆大堆的日志中手动观察并筛选可能相关的日志，经过观察与友盟相关的 tag 有：`UMConfigure`, `UMLog`, `UMCrash`, `crashsdk`，用 `adb logcat -s UMConfigure UMLog UMCrash crashsdk` 筛选出来

# 2021/11/29

按照 [接入与基础功能](https://developer.umeng.com/docs/193624/detail/194590) 和 [高级功能](https://developer.umeng.com/docs/193624/detail/291367) 这两个文档接入 APM

因为 crash 由 Bugly 负责，友盟 APM 只想用它的 ANR 功能，所以只开启 ANR

```java
// 目前只需用到 ANR 监控，崩溃监控用 bugly
UMCrash.initConfig(bundleOf(
    UMCrash.KEY_ENABLE_ANR to true,             // 开启 ANR 监控
    UMCrash.KEY_ENABLE_CRASH_JAVA to false,     // 关闭捕获 java crash
    UMCrash.KEY_ENABLE_CRASH_NATIVE to false,   // 关闭捕获 native crash
    UMCrash.KEY_ENABLE_PA to false,             // 关闭卡顿监控
    UMCrash.KEY_ENABLE_LAUNCH to false,         // 关闭启动监控
    UMCrash.KEY_ENABLE_MEM to false             // 关闭内存监控
))
```

## MI 9 Android 11

可以看到 APM 成功初始化了，但没能捕获到 ANR

## 绿联 6810 Android 5.1 

能捕获到 ANR，但后台没有记录（大概过了一个下午），相关日志如下：

```logcat
I/UMConfigure( 2142): common version is 9.3.8
I/UMConfigure( 2142): common type is 0
I/UMConfigure( 2142): current appkey is 5a7cf79cf43e481b2a00045c, last appkey is 5a7cf79cf43e481b2a00045c
I/UMConfigure( 2142): channel is Umeng
I/UMLog ( 2142): 统计SDK初始化成功
I/UMLog ( 2142): PUSH AppKey设置成功
I/UMLog ( 2142): PUSH Channel设置成功
I/UMConfigure( 2142): push secret is 6462c6bb0284e4a9ebb739dd6b80260e
I/UMLog ( 2142): PUSH Secret设置成功
E/UMCrash ( 2142): enablePaLog is false
E/UMCrash ( 2142): enableLaunchLog is false
E/UMCrash ( 2142): enableMemLog is false
I/UMCrash ( 2142): inner config : net rate is -1
I/UMLog ( 2142): APM SDK初始化成功                                        # 说明捕获 ANR 已准备好
I/crashsdk( 2142): version unique build id: 1e2addac
I/crashsdk( 1834): Unexp log not enabled, skip update unexp info!
...
E/crashsdk( 1834): ANR occurred in process: com.viomi.fridge.vertical    # 捕获到 ANR，但没有更多日志也不知道本地日志在哪
```

## 创能达 hmi3326a Android 8.1.0

能捕获到 ANR 但控制台无统计

```logcat
11-29 14:39:11.801   947  1087 I UMConfigure: common version is 9.3.8
11-29 14:39:11.803   947  1087 I UMConfigure: common type is 0
11-29 14:39:11.978   947  1087 I UMConfigure: current appkey is 5a7cf79cf43e481b2a00045c, last appkey is 5a7cf79cf43e481b2a00045c
11-29 14:39:11.983   947  1087 I UMConfigure: channel is Umeng
11-29 14:39:12.233   947  1087 D UMLog   : 统计SDK常见问题索引贴 详见链接 http://developer.umeng.com/docs/66650/cate/66650
11-29 14:39:12.235   947  1087 I UMLog   : 统计SDK初始化成功
11-29 14:39:12.285   947  1087 I UMLog   : PUSH AppKey设置成功
11-29 14:39:12.305   947  1087 I UMLog   : PUSH Channel设置成功
11-29 14:39:12.305   947  1087 I UMConfigure: push secret is 6462c6bb0284e4a9ebb739dd6b80260e
11-29 14:39:12.309   947  1087 I UMLog   : PUSH Secret设置成功
11-29 14:39:13.065   947  1180 I crashsdk: version unique build id: c4bceb1a
11-29 14:39:13.133   947  1087 E UMCrash : enablePaLog is false
11-29 14:39:13.133   947  1087 E UMCrash : enableLaunchLog is false
11-29 14:39:13.133   947  1087 E UMCrash : enableMemLog is false
11-29 14:39:13.367   947  1087 I UMCrash : inner config : net rate is 0
11-29 14:39:13.367   947  1087 I UMCrash : inner config : net close.
11-29 14:39:13.367   947  1087 I UMLog   : APM SDK初始化成功
11-29 14:39:15.062   947  1243 I UMLog   : 基础组件库完整性自检通过。
...
11-29 14:39:55.861   947  1154 E crashsdk: ANR occurred in process: com.viomi.fridge.vertical

// 发生 ANR 后重启能看到上传日志的 logcat
11-29 14:46:49.782  2353  2413 I crashsdk: crashsdk uploading logs
```

## 自研 Android 10

依然是能捕获到 ANR 但控制台无统计

# 2021/11/30

如上文，接入失败后提工单交流下，客服建议按照 Demo 排除问题

用官方的 [CrashDemo](https://gitee.com/umengplus/CrashDemo) 测试，Demo 很简单就是加入相关依赖并调用 `UMConfigure.init`

```gradle
dependencies {
    implementation files('libs/umeng-common-9.3.8.jar')
    implementation(name: 'umeng-asms-v1.2.1', ext: 'aar')
    implementation(name: 'umeng-apm-v1.5.0', ext: 'aar')
}
```

```java
public void onCreate() {
    super.onCreate();
    UMConfigure.setLogEnabled(true);
    UMConfigure.init(
            this,
            "5a7cf79cf43e481b2a00045c", // launcher app key
            "umeng",
            UMConfigure.DEVICE_TYPE_PHONE,
            "6462c6bb0284e4a9ebb739dd6b80260e" // launcher message secret
    );
}
```

## MI 9 - Android 11

官方 Demo 捕获 ANR 成功，控制台能记录到一次 ANR 记录

```logcat
11-30 13:52:03.250 27736 27736 I UMConfigure: common version is 9.3.8
11-30 13:52:03.250 27736 27736 I UMConfigure: common type is 0
11-30 13:52:03.267 27736 27736 I UMConfigure: current appkey is 5a7cf79cf43e481b2a00045c, last appkey is 5a7cf79cf43e481b2a00045c
11-30 13:52:03.268 27736 27736 I UMConfigure: channel is umeng
11-30 13:52:03.275 27736 27736 D UMLog   : 统计SDK常见问题索引贴 详见链接 http://developer.umeng.com/docs/66650/cate/66650
11-30 13:52:03.275 27736 27736 I UMLog   : 统计SDK初始化成功
11-30 13:52:03.298 27736 27763 I crashsdk: version unique build id: 3e56d57c
11-30 13:52:03.300 27736 27764 D crashsdk: Native log stat thread 27764 setup, waiting
11-30 13:52:03.302 27736 27736 I crashsdk: Catcher mask: 1000, 4096
11-30 13:52:03.302 27736 27736 I crashsdk: Catcher thread 27746
11-30 13:52:03.302 27736 27736 D crashsdk: Register ANR handler ...
11-30 13:52:03.302 27736 27736 I crashsdk: begin hack android.os.Process
11-30 13:52:03.303 27736 27736 I crashsdk: end hack android.os.Process
11-30 13:52:03.303 27736 27736 I crashsdk: LibcMalloc detail: disabled.
11-30 13:52:03.329 27736 27736 I UMCrash : inner config : net rate is 0
11-30 13:52:03.329 27736 27736 I UMCrash : inner config : net close.
11-30 13:52:03.329 27736 27736 I UMLog   : APM SDK初始化成功
11-30 13:52:03.407 27736 27777 I UMLog   : 基础组件库完整性自检通过。
11-30 13:52:03.457 27736 27777 I UMLog   : 统计SDK版本号: 9.3.8
11-30 13:52:03.457 27736 27777 I UMLog   : ZID SDK版本号: 1.2.1
11-30 13:52:03.459 27736 27777 I UMLog   : APM SDK版本号: 1.5.0
11-30 13:52:03.605 27736 27736 I crashsdk: Unexp log not enabled, skip update unexp info!
11-30 13:52:03.627 27736 27736 I crashsdk: Unexp log not enabled, skip update unexp info!
11-30 13:52:03.769 27736 27777 D UMLog   : 当前发送策略为：启动时发送。详见链接 https://developer.umeng.com/docs/66632/detail/66976?um_channel=sdk
11-30 13:52:03.843 27736 27777 D UMLog   : 当前发送策略为：启动时发送。详见链接 https://developer.umeng.com/docs/66632/detail/66976?um_channel=sdk
11-30 13:52:03.844 27736 27777 D UMLog   : 当前发送策略为：启动时发送。详见链接 https://developer.umeng.com/docs/66632/detail/66976?um_channel=sdk
--------- beginning of crash
11-30 13:52:11.305 27736 27762 D crashsdk: reportCrashStatImpl: processName: com.umeng.crashdemo
11-30 13:52:11.305 27736 27762 D crashsdk: name: all_all, key: 1, count: 1
11-30 13:52:11.305 27736 27762 D crashsdk: name: start_pv, key: 100, count: 1
11-30 13:52:11.305 27736 27762 D crashsdk: name: all_bg, key: 101, count: 1
11-30 13:52:18.377 27736 27763 I crashsdk: crashsdk uploading logs
11-30 13:53:07.179 27736 27736 I crashsdk: [DEBUG] begin generate traces: /data/user/10/com.umeng.crashdemo/crashsdk/tags/OMEDHSARC0GNEMU0MOC.anr (2000 ms)
11-30 13:53:07.217 27955 27955 I crashsdk: [DEBUG] forked traces process: 27955, gid: 704
11-30 13:53:07.217 27955 27955 I crashsdk: [DEBUG] dump art internal: 124
11-30 13:53:07.217 27955 27955 I crashsdk: [DEBUG] VMExt: 0xb400007a3f24a380, i: 62, str: 0
11-30 13:53:07.217 27955 27955 I crashsdk: [DEBUG] aborting: 0x0, 0
11-30 13:53:07.217 27955 27955 I crashsdk: [DEBUG] Dump: 0x0, State: 0x79b8cd82b4, JavaStack: 0x79b8cdc8f8
11-30 13:53:07.217 27955 27955 I crashsdk: [DEBUG] current: 0xb400007a3f2b4c00, pid: 27955
11-30 13:53:07.217 27955 27955 I crashsdk: [DEBUG] List: 0xb400007a3f305000
11-30 13:53:07.217 27955 27955 I crashsdk: [DEBUG] Each: 0x79b8cf4920
11-30 13:53:07.218 27955 27955 I crashsdk: [DEBUG] err: 0x7a3aa75c30
11-30 13:53:07.218 27955 27955 I crashsdk: [DEBUG] begin each
11-30 13:53:07.218 27955 27955 I crashsdk: [DEBUG] dumping 0xb400007a3f2b4c00 ...
11-30 13:53:07.239 27955 27955 I crashsdk: [DEBUG] end each
11-30 13:53:07.240 27955 27955 I crashsdk: [DEBUG] wrote traces into: /data/user/10/com.umeng.crashdemo/crashsdk/tags/OMEDHSARC0GNEMU0MOC.anr
11-30 13:53:07.246 27736 27736 I crashsdk: Raising signal 3 ...
11-30 13:53:07.352 27736 27956 I crashsdk: art.onDebugMessage: cmd: 'dumpAndClose lids=0,3,2 tail=1000'
11-30 13:53:07.389 27736 27956 I crashsdk: art.onDebugMessage: ret: 0, e: 2 (No such file or directory)
11-30 13:53:07.389 27736 27956 I crashsdk: art.onDebugMessage: recv end: 2 (No such file or directory)
11-30 13:53:07.392 27736 27956 I crashsdk: [DEBUG] Open dir '/storage' failed: Permission denied
11-30 13:53:07.393 27736 27956 I UMCrash : page json is {"source":0,"action_name":"page_view","action_parameter":[{"name":"MainActivity-1638251523462-onCreated"},{"name":"MainActivity-1638251523605-onStarted"},{"name":"MainActivity-1638251523608-onResumed"}]}
11-30 13:53:07.393 27736 27956 I crashsdk: [DEBUG] Check logs in directory: /data/user/10/com.umeng.crashdemo/crashsdk/logs/
11-30 13:53:07.394 27736 27956 I crashsdk: Unexp log not enabled, skip update unexp info!
11-30 13:53:17.492 28079 27959 I crashsdk: [DEBUG] process: 28079, gid: 704
11-30 13:53:17.492 28079 27959 I crashsdk: source_file: /data/user/10/com.umeng.crashdemo/crashsdk/logs/5a7cf79cf43e481b2a00045c_1.0_3e56d57c_MI-9_11_163825152329918673_20211130135307_fg_anr.log
11-30 13:53:17.492 28079 27959 I crashsdk: zipExt: .gz, zip: 1
11-30 13:53:17.534 27736 27959 I crashsdk: [DEBUG] zip_log, rtn: 2, timeout or died: 0
11-30 13:53:17.536 27736 27762 I crashsdk: crashsdk uploading logs
11-30 13:53:17.539 27736 27762 D crashsdk: Uploading to https://errlog.umeng.com/upload
11-30 13:53:18.434 27736 27762 I crashsdk: Response code: 200
11-30 13:53:18.438 27736 27762 I crashsdk: Log upload response:��������
��������succeed��
11-30 13:53:18.438 27736 27762 E crashsdk: Uploaded log: 5a7cf79cf43e481b2a00045c_1.0_3e56d57c_MI-9_11_163825152329918673_20211130135307_fg_anr.log.gz
```

Launcher 按照上一章节的步骤依然不能捕获 ANR，对照上面的日志发现没有 `Register ANR handler` 相关日志

```logcat
2021-11-30 14:59:19.073 14773-14807/com.viomi.fridge.vertical I/UMConfigure: common version is 9.3.8
2021-11-30 14:59:19.073 14773-14807/com.viomi.fridge.vertical I/UMConfigure: common type is 0
2021-11-30 14:59:19.110 14773-14807/com.viomi.fridge.vertical I/UMConfigure: current appkey is 5a7cf79cf43e481b2a00045c, last appkey is 5a7cf79cf43e481b2a00045c
2021-11-30 14:59:19.110 14773-14807/com.viomi.fridge.vertical I/UMConfigure: channel is Umeng
2021-11-30 14:59:19.130 14773-14807/com.viomi.fridge.vertical D/UMLog: 统计SDK常见问题索引贴 详见链接 http://developer.umeng.com/docs/66650/cate/66650
2021-11-30 14:59:19.130 14773-14807/com.viomi.fridge.vertical I/UMLog: 统计SDK初始化成功
2021-11-30 14:59:19.153 14773-14807/com.viomi.fridge.vertical I/UMLog: PUSH AppKey设置成功
2021-11-30 14:59:19.155 14773-14807/com.viomi.fridge.vertical I/UMLog: PUSH Channel设置成功
2021-11-30 14:59:19.155 14773-14807/com.viomi.fridge.vertical I/UMConfigure: push secret is 6462c6bb0284e4a9ebb739dd6b80260e
2021-11-30 14:59:19.155 14773-14807/com.viomi.fridge.vertical I/UMLog: PUSH Secret设置成功
2021-11-30 14:59:19.238 14773-14836/com.viomi.fridge.vertical I/crashsdk: version unique build id: fe872a95
2021-11-30 14:59:19.278 14773-14807/com.viomi.fridge.vertical E/UMCrash: enablePaLog is false
2021-11-30 14:59:19.278 14773-14807/com.viomi.fridge.vertical E/UMCrash: enableLaunchLog is false
2021-11-30 14:59:19.278 14773-14807/com.viomi.fridge.vertical E/UMCrash: enableMemLog is false
2021-11-30 14:59:19.306 14773-14807/com.viomi.fridge.vertical I/UMCrash: inner config : net rate is 0
2021-11-30 14:59:19.306 14773-14807/com.viomi.fridge.vertical I/UMCrash: inner config : net close.
2021-11-30 14:59:19.306 14773-14807/com.viomi.fridge.vertical I/UMLog: APM SDK初始化成功
2021-11-30 14:59:19.334 14773-14807/com.viomi.fridge.vertical I/UMLog_com.umeng.message.PushAgent: AndroidManifest配置正确、参数正确
2021-11-30 14:59:19.440 14773-14864/com.viomi.fridge.vertical I/UMLog_com.umeng.message.PushAgent: appkey:umeng:5a7cf79cf43e481b2a00045c,secret:6462c6bb0284e4a9ebb739dd6b80260e
2021-11-30 14:59:19.534 14773-14847/com.viomi.fridge.vertical I/UMLog: 基础组件库完整性自检通过。
2021-11-30 14:59:19.664 14773-14847/com.viomi.fridge.vertical I/UMLog: 统计SDK版本号: 9.3.8
2021-11-30 14:59:19.664 14773-14847/com.viomi.fridge.vertical I/UMLog: ZID SDK版本号: 1.2.2
2021-11-30 14:59:19.666 14773-14847/com.viomi.fridge.vertical I/UMLog: 推送SDK版本号: 6.3.3
2021-11-30 14:59:19.668 14773-14847/com.viomi.fridge.vertical I/UMLog: APM SDK版本号: 1.5.2
2021-11-30 14:59:20.341 14773-14773/com.viomi.fridge.vertical I/UMLog_com.umeng.message.UTrack: appCachedPushlog开始, 设置appLaunchSending标志位
2021-11-30 14:59:20.483 14773-14773/com.viomi.fridge.vertical I/crashsdk: Unexp log not enabled, skip update unexp info!
2021-11-30 14:59:20.535 14773-14773/com.viomi.fridge.vertical I/crashsdk: Unexp log not enabled, skip update unexp info!
2021-11-30 14:59:20.566 14773-14773/com.viomi.fridge.vertical I/UMLog_com.umeng.message.PushAgent: 注册成功:AtiraebTU4tda5HrzDIoRVIC6mhmlKv3eQVHiI7XdiY3
2021-11-30 14:59:20.715 14773-14847/com.viomi.fridge.vertical D/UMLog: 当前发送策略为：启动时发送。详见链接 https://developer.umeng.com/docs/66632/detail/66976?um_channel=sdk
2021-11-30 14:59:20.861 14773-14957/com.viomi.fridge.vertical I/UMLog_UmengMessageCallbackHandlerService: 进程名：com.viomi.fridge.vertical
2021-11-30 14:59:20.863 14773-14957/com.viomi.fridge.vertical I/UMLog_UmengMessageCallbackHandlerService: 注册：AtiraebTU4tda5HrzDIoRVIC6mhmlKv3eQVHiI7XdiY3，状态：true
2021-11-30 14:59:34.260 14773-14836/com.viomi.fridge.vertical I/crashsdk: crashsdk uploading logs

// 发生 ANR 后未能捕获到
```

## 找到问题了

经过对比和测试两边的配置代码，发现是 `UMCrash.initConfig` 的 bug

文档 [采集开关](https://developer.umeng.com/docs/193624/detail/291367#h3-jm0-fos-42f) 里介绍可通过 `UMCrash.initConfig` 开/关 java crash、native crash、anr 等功能，对于 Launcher 来说 crash 目前由 Bugly 收集，于是只想用 ANR，配置如下：

```kotlin
// 目前只需用到 ANR 监控，崩溃监控用 bugly
UMCrash.initConfig(bundleOf(
    UMCrash.KEY_ENABLE_ANR to true,             // 开启 ANR 监控
    UMCrash.KEY_ENABLE_CRASH_JAVA to false,     // 关闭捕获 java crash
    UMCrash.KEY_ENABLE_CRASH_NATIVE to false,   // 关闭捕获 native crash
    UMCrash.KEY_ENABLE_PA to false,             // 关闭卡顿监控
    UMCrash.KEY_ENABLE_LAUNCH to false,         // 关闭启动监控
    UMCrash.KEY_ENABLE_MEM to false             // 关闭内存监控
))
```

但发现有个 bug，必须开启 `KEY_ENABLE_CRASH_NATIVE` 才能使 `KEY_ENABLE_ANR` 生效：ANR 是通过添加 `SIGQUIT` 信号处理器来实现监听的，而只有开启 `KEY_ENABLE_CRASH_NATIVE` 后才有 `Register ANR handler ...` 以及 ANR 日志的出现

提交了一个 [工单](https://account.umeng.com/feedback/detail?id=5185009759545592) 给友盟，看那边后续的处理

## KEY_ENABLE_CRASH_NATIVE & KEY_ENABLE_ANR

经测试，开启 native crash & anr 后，在绿联 6810 Android 5.1 上也能捕获到 ANR

从日志上看，友盟也是通过 `SIGQUIT` 信号处理器来捕获 ANR 的，不同的是友盟是 fork 出一个子进程来进行 dump 操作

```logcat
2021-01-01 08:14:12.546 3741-3741/com.viomi.fridge.vertical I/crashsdk: Catcher mask: 1000, 4096
2021-01-01 08:14:12.548 3741-3741/com.viomi.fridge.vertical I/crashsdk: Catcher thread 3750
2021-01-01 08:14:12.548 3741-3741/com.viomi.fridge.vertical D/crashsdk: Register ANR handler ...
2021-01-01 08:14:12.989 2702-2702/com.viomi.fridge.vertical I/crashsdk: [DEBUG] begin generate traces: /data/data/com.viomi.fridge.vertical/crashsdk/tags/TOIM1LACITREV0EGDIRF0IMOIV0MOC.anr (2000 ms)
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] forked traces process: 3943, gid: 247
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] dump art internal: 101
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] VMExt: 0xf4898200, i: 58, str: 3
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] runtime trace: 33,20,/data/anr/traces.txt
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] aborting: 0xf47fdaf4, 0
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] Dump: 0x0, State: 0xf4730949, JavaStack: 0xf472f2f5
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] Thread spec key: 0xf47feffc
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] current: 0xf4827400, pid: 3943
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] List: 0xf48ac000
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] Each: 0xf473637d
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] err: 0xf5eb0e60
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] begin each
2021-01-01 08:14:12.995 3943-3943/? I/crashsdk: [DEBUG] dumping 0xf4827400 ...
2021-01-01 08:14:13.071 3943-3943/? I/crashsdk: [DEBUG] end each
2021-01-01 08:14:13.071 3943-3943/? I/crashsdk: [DEBUG] wrote traces into: /data/data/com.viomi.fridge.vertical/crashsdk/tags/TOIM1LACITREV0EGDIRF0IMOIV0MOC.anr
2021-01-01 08:14:13.077 2702-2702/com.viomi.fridge.vertical I/crashsdk: Raising signal 3 ...
```