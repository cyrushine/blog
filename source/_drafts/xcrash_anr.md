xCrash 具有捕获 ANR 的功能
注册 SIGQUIT 信号处理器
调用 Runtime::DumpForSigQuit 打印各进程和线程的调用栈

在 xcrash.NativeHandler#traceCallback 里会检查当前进程的状态是否为 ANR
默认是在 15s 内轮询检查进程是否处于 ActivityManager.ProcessErrorStateInfo.NOT_RESPONDING

        //check process ANR state
        if (NativeHandler.getInstance().anrCheckProcessState) {
            if (!Util.checkProcessAnrState(NativeHandler.getInstance().ctx, NativeHandler.getInstance().anrTimeoutMs)) {
                FileManager.getInstance().recycleLogFile(new File(logPath));
                Log.d(tag, "not an ANR");
                return; //not an ANR
            }
        }

    static boolean checkProcessAnrState(Context ctx, long timeoutMs) {
        ActivityManager am = (ActivityManager) ctx.getSystemService(Context.ACTIVITY_SERVICE);
        if (am == null) return false;

        int pid = android.os.Process.myPid();
        long poll = timeoutMs / 500;
        for (int i = 0; i < poll; i++) {
            List<ActivityManager.ProcessErrorStateInfo> processErrorList = am.getProcessesInErrorState();
            if (processErrorList != null) {
                for (ActivityManager.ProcessErrorStateInfo errorStateInfo : processErrorList) {
                    if (errorStateInfo.pid == pid) {
                        Log.d("xcrash_dumper", String.format("process state: %s", errorStateInfo.condition));
                    }
                    if (errorStateInfo.pid == pid && errorStateInfo.condition == ActivityManager.ProcessErrorStateInfo.NOT_RESPONDING) {
                        return true;
                    }
                }
            }

            try {
                Thread.sleep(500);
            } catch (Exception ignored) {
            }
        }

        return false;
    }
            
但是这个轮询时长很有问题，默认 15s 在 mi9 Android 11 上不够长，ANR 对话框还没弹出来，此时进程并不是处于 ActivityManager.ProcessErrorStateInfo.NOT_RESPONDING，导致 xCrash 判定不是 ANR，然后把日志删了
我把这个时长增加到 60s 是能判定 ANR 的

所以这里有几个解决方案：

1，增大 ANR 轮询时长到一个足够大的值，但此时 ANR 对话框已经弹出，用户可能直接 kill app，于是以下逻辑可能会无法执行（xcrash.NativeHandler#traceCallback check process ANR state 以下的代码）：
    1.清理 ANR 目录（维持最多 xcrash.XCrash.InitParameters#anrLogCountMax）
    2.日志文件后缀是 .trace.xcrash 而不是 .anr.xcrash，因为 anr 和 trace 的逻辑是一样的，最后通过上面的 ANR 判定逻辑来判断是不是 ANR，是的话把 .trace.xcrash 改名为 .anr.xcrash
    3.anr callback 无法执行（xcrash.XCrash.InitParameters#setAnrCallback）

2. 关闭 ANR 检查（xcrash.XCrash.InitParameters#setAnrCheckProcessState）    
这个看似最简单但是不行的，因为 SIGQUIT 对于 APP 来说是 Runtime::DumpForSigQuit，那么当其他 APP 发生 ANR 时系统是通过向所有其他进程发送 SIGQUIT 来收集 ANR 日志的（参考 ANR 日志的内容），当然我们的 APP 也会收到，但此时并不是我们的 APP 发生了 ANR

所以检查是否当前 APP 发生了 ANR 这个校验是一定要做得，那么怎么优化第一个解决方案呢？
1. 清理操作可以放到其他地方，比如初始化后
2. 日志默认是 .anr.xcrash，轮询后发现 app 并没有处于 NOT_RESPONDING 则把日志删掉
    如果是 ANR 则足够长的轮询时长能够保证在弹出 ANR 对话框（处于 NOT_RESPONDING）后判定为 ANR
    即使用户即刻 kill app，ANR 日志还是保留下来，判定为 ANR
3. anr callback 不要再 catch anr 后回调，因为此刻很有可能被 kill 而不能稳定调用，可以将 ANR 日志标为 unread，初始化后检查日志将 unread 的回调并置为 read


xCrash 在绿联 6810

捕获到 SIGQUIT 后，会卡在 Runtime::DumpForSigQuit，然后不断收到 SIGQUIT，APP 会一直卡顿一直弹出 ANR 对话框（即使一直点击等待），这里不确定是 xc_trace_libart_dbg_suspend 的问题还是 Runtime::DumpForSigQuit 的问题

但是 Runtime::DumpForSigQuit 确实是有被执行的，APP 重启后可以发现上一次的 ANR dump 被输出到日志文件里了，但文件名还没改过来 tombstone_00001637663901079557_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash，从日志来看应该是后续代码没有执行，卡在 Runtime::DumpForSigQuit 里了

    //is Android Lollipop (5.x)?
    xc_trace_is_lollipop = ((21 == xc_common_api_level || 22 == xc_common_api_level) ? 1 : 0);

        if(sigsetjmp(jmpenv, 1) == 0) 
        {
            if(xc_trace_is_lollipop)
                xc_trace_libart_dbg_suspend();
            XCD_LOG_DEBUG("before dump");
            xc_trace_libart_runtime_dump(*xc_trace_libart_runtime_instance, xc_trace_libcpp_cerr);
            XCD_LOG_DEBUG("after dump, print log file");
            if(xc_trace_is_lollipop)
                xc_trace_libart_dbg_resume();
        }

2021-11-23 18:20:10.580 4176-4477/com.viomi.fridge.vertical D/xcrash_dumper: before dump
2021-11-23 18:20:36.784 4176-4176/com.viomi.fridge.vertical D/xcrash_dumper: SIGQUIT handler get 3, 255
2021-11-23 18:21:10.108 4176-4176/com.viomi.fridge.vertical D/xcrash_dumper: SIGQUIT handler get 3, 255
2021-11-23 18:21:26.920 4176-4176/com.viomi.fridge.vertical D/xcrash_dumper: SIGQUIT handler get 3, 255
2021-11-23 18:22:10.100 4176-4176/com.viomi.fridge.vertical D/xcrash_dumper: SIGQUIT handler get 3, 255
2021-11-23 18:22:26.634 4176-4176/com.viomi.fridge.vertical D/xcrash_dumper: SIGQUIT handler get 3, 255
2021-11-23 18:23:10.109 4176-4176/com.viomi.fridge.vertical D/xcrash_dumper: SIGQUIT handler get 3, 255