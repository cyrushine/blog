集成友盟 umeng 主要是想用它的 ANR 统计功能，Bugly 的 ANR 统计下线了

前置条件：移除 Bugly 和 xCrash，因为它们有可能与友盟 APM 冲突（猜测都是通过注册 SIGQUIT 处理器来捕获 ANR，友盟和 Bugly 不开源）

因为友盟 APM 不开源也没有详细的教程描述日志，所以只能从大堆大堆的日志中手动观察并筛选可能相关的日志，经过观察与友盟相关的 tag 有：UMConfigure, UMLog, UMCrash, crashsdk

手机 MI9 Android 11，可以看到 APM 成功初始化了，但没能捕获到 ANR

绿联 6810 Android 5.1 能捕获到 ANR，但后台没有记录（大概过了一个下午），相关日志如下：

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