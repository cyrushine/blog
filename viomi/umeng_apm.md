集成友盟 umeng 主要是想用它的 ANR 统计功能（Bugly 的 ANR 统计下线了）

前置条件：移除 Bugly 和 xCrash，因为它们有可能与友盟 APM 冲突（猜测都是通过注册 SIGQUIT 处理器来捕获 ANR，友盟和 Bugly 不开源），分支是 `local/umeng_anr`

因为友盟 APM 不开源也没有详细的教程描述日志，所以只能从大堆大堆的日志中手动观察并筛选可能相关的日志，经过观察与友盟相关的 tag 有：`UMConfigure`, `UMLog`, `UMCrash`, `crashsdk`，用 `adb logcat -s UMConfigure UMLog UMCrash crashsdk`

# MI 9 Android 11

可以看到 APM 成功初始化了，但没能捕获到 ANR

# 绿联 6810 Android 5.1 

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

# 创能达 hmi3326a Android 8.1.0

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

# 自研 Android 10

依然是能捕获到 ANR 但控制台无统计