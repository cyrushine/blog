---
title: xCrash ANR 兼容性测试
date: 2021-11-29 12:00:00 +0800
tags: [xCrash, ANR, 崩溃]
---

| Target | Steps |
|--------|-------|
| 目的             | 测试 `xCrash` 捕获 `ANR` 的能力在各个 Android 版本上的兼容性 |
| 平台             | [WeTest](https://wetest.qq.com/) - 兼容性测试 - 选取 Android 5.1 至 Android 12 共 8 个机型 |
| Apk              | Viomi Fridge Launcher |
| xCrash Version   | 3.0.0 |
| ANR 捕获机制      | 注册 `SIGQUIT` 信号处理器，`Runtime::DumpForSigQuit` 打印调用栈，一段时间内轮询 `ActivityManager.getProcessesInErrorState` 是否为 `ProcessErrorStateInfo.NOT_RESPONDING` 来判断当前 APP 是否发生 ANR |
| 测试流程          | APP 启动后 5s 初始化 xCrash ANR，3s 后启动 `AnrService`，通过 `startService` 超时来触发 ANR |
| 日志分析          | 将日志文件逐个下载，放到 [LogPad](https://cloudvyzor.com/) 上过滤出 tag 为 `xcrash_dumper` 的条目 |

# 详细的日志埋点

```kotlin
fun initXCrash() {
    mainHandler.postDelayed({
        XCrash.init(app, XCrash.InitParameters().apply {
            setAnrCallback { logPath, emergency ->
                Log.d("xcrash_dumper", "ANR Callback, $emergency, $logPath")    // 自定义 ANR 回调
            }
        })

        mainHandler.postDelayed({
            try {
                val nativeHandlerClz = Class.forName("xcrash.NativeHandler")
                val instanceField = nativeHandlerClz.getDeclaredField("instance")
                instanceField.isAccessible = true
                val nativeHandler = instanceField.get(nativeHandlerClz)
                val anrTimeoutField = nativeHandlerClz.getDeclaredField("anrTimeoutMs")
                anrTimeoutField.isAccessible = true
                anrTimeoutField.set(nativeHandler, 20 * 1000L)
                // 判断 ANR 的阈值校正为 20s，默认 15s 太短有时 ANR 对话框还没弹出，APP 不处于 NOT_RESPONDING 状态
                Log.d("xcrash_dumper", "NativeHandler.anrTimeoutMs = 20s")    
            } catch (e: Exception) {
                Log.e("xcrash_dumper", "fail to change NativeHandler.anrTimeoutMs", e)
            }

            app.startService(Intent(app, AnrService::class.java))
            Log.d("xcrash_dumper", "AnrService was started")    // 启动 Service 超时
        }, 3 * 1000L)
    }, 5 * 1000L)    
}

class AnrService {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val tombstones = File(applicationContext.filesDir, "tombstones")
        var anrs = tombstones.listFiles { _, name -> name.endsWith("anr.xcrash") }
        log("AnrService, delete ${anrs?.size} anr logs before anr")    // 清理 tombstones 目录
        anrs.forEach { it.deleteSafely() }

        Thread.sleep(30 * 1000L)    // 触发 ANR

        anrs = tombstones.listFiles { _, name -> name.endsWith("anr.xcrash") }
        // 看看有没 ANR 日志，没有的话说明没能捕获到 ANR
        log("AnrService, now ${System.currentTimeMillis() / 1000L}, found anr logs, ${anrs?.joinToString { it.name }}")    

        return super.onStartCommand(intent, flags, startId)
    }    
}
```

```java
class XCrash {
    public static synchronized int init(Context ctx, InitParameters params) {
        if (XCrash.initialized) {
            return Errno.OK;
        }

        XCrash.initialized = true;

        if (ctx == null) {
            return Errno.CONTEXT_IS_NULL;
        }

        Log.d("xcrash_dumper", "xcrash start");    // xCrash 初始化
        // ...
        //init native crash handler / ANR handler (API level >= 21)
        int r = Errno.OK;
        if (params.enableNativeCrashHandler || (params.enableAnrHandler && Build.VERSION.SDK_INT >= 21)) {
            r = NativeHandler.getInstance().initialize(
                ctx,
                params.libLoader,
                appId,
                params.appVersion,
                params.logDir,
                params.enableNativeCrashHandler,
                params.nativeRethrow,
                params.nativeLogcatSystemLines,
                params.nativeLogcatEventsLines,
                params.nativeLogcatMainLines,
                params.nativeDumpElfHash,
                params.nativeDumpMap,
                params.nativeDumpFds,
                params.nativeDumpNetworkInfo,
                params.nativeDumpAllThreads,
                params.nativeDumpAllThreadsCountMax,
                params.nativeDumpAllThreadsWhiteList,
                params.nativeCallback,
                params.enableAnrHandler && Build.VERSION.SDK_INT >= 21,
                params.anrRethrow,
                params.anrCheckProcessState,
                params.anrLogcatSystemLines,
                params.anrLogcatEventsLines,
                params.anrLogcatMainLines,
                params.anrDumpFds,
                params.anrDumpNetworkInfo,
                params.anrCallback);
        }

        //maintain tombstone and placeholder files in a background thread with some delay
        FileManager.getInstance().maintain();

        return r;
    }    
}

class NativeHandler {
    int initialize(...) {
        //load lib
        if (libLoader == null) {
            try {
                System.loadLibrary("xcrash");
            } catch (Throwable e) {
                XCrash.getLogger().e(Util.TAG, "NativeHandler System.loadLibrary failed", e);
                return Errno.LOAD_LIBRARY_FAILED;
            }
        } else {
            try {
                libLoader.loadLibrary("xcrash");
            } catch (Throwable e) {
                XCrash.getLogger().e(Util.TAG, "NativeHandler ILibLoader.loadLibrary failed", e);
                return Errno.LOAD_LIBRARY_FAILED;
            }
        }

        this.ctx = ctx;
        this.crashRethrow = crashRethrow;
        this.crashCallback = crashCallback;
        this.anrEnable = anrEnable;
        this.anrCheckProcessState = anrCheckProcessState;
        this.anrCallback = anrCallback;
        this.anrTimeoutMs = anrRethrow ? 15 * 1000 : 30 * 1000; //setting rethrow to "false" is NOT recommended

        Log.d("xcrash_dumper", "xcrash native start");    // xCrash native 初始化 
        //init native lib
        try {
            int r = nativeInit(
                Build.VERSION.SDK_INT,
                Build.VERSION.RELEASE,
                Util.getAbiList(),
                Build.MANUFACTURER,
                Build.BRAND,
                Util.getMobileModel(),
                Build.FINGERPRINT,
                appId,
                appVersion,
                ctx.getApplicationInfo().nativeLibraryDir,
                logDir,
                crashEnable,
                crashRethrow,
                crashLogcatSystemLines,
                crashLogcatEventsLines,
                crashLogcatMainLines,
                crashDumpElfHash,
                crashDumpMap,
                crashDumpFds,
                crashDumpNetworkInfo,
                crashDumpAllThreads,
                crashDumpAllThreadsCountMax,
                crashDumpAllThreadsWhiteList,
                anrEnable,
                anrRethrow,
                anrLogcatSystemLines,
                anrLogcatEventsLines,
                anrLogcatMainLines,
                anrDumpFds,
                anrDumpNetworkInfo);
            if (r != 0) {
                XCrash.getLogger().e(Util.TAG, "NativeHandler init failed");
                return Errno.INIT_LIBRARY_FAILED;
            }
            initNativeLibOk = true;
            Log.d("xcrash_dumper", "xcrash native end");    // xCrash native 初始化结束
            return 0; //OK
        } catch (Throwable e) {
            XCrash.getLogger().e(Util.TAG, "NativeHandler init failed", e);
            return Errno.INIT_LIBRARY_FAILED;
        }
    }    
}
```

```cpp
static jint xc_jni_init(...)
{
    //...
    if(trace_enable)
    {
        //trace init
        XCD_LOG_DEBUG("ANR native start");    // xcrash anr 初始化
        r_trace = xc_trace_init(env,
                            trace_rethrow ? 1 : 0,
                            (unsigned int)trace_logcat_system_lines,
                            (unsigned int)trace_logcat_events_lines,
                            (unsigned int)trace_logcat_main_lines,
                            trace_dump_fds ? 1 : 0,
                            trace_dump_network_info ? 1 : 0);
    }
    //...
}

int xc_trace_init(JNIEnv *env,
                  int rethrow,
                  unsigned int logcat_system_lines,
                  unsigned int logcat_events_lines,
                  unsigned int logcat_main_lines,
                  int dump_fds,
                  int dump_network_info)
{
    int r;
    pthread_t thd;

    //capture SIGQUIT only for ART
    if(xc_common_api_level < 21) return 0;

    //is Android Lollipop (5.x)?
    xc_trace_is_lollipop = ((21 == xc_common_api_level || 22 == xc_common_api_level) ? 1 : 0);

    xc_trace_dump_status = XC_TRACE_DUMP_NOT_START;
    xc_trace_rethrow = rethrow;
    xc_trace_logcat_system_lines = logcat_system_lines;
    xc_trace_logcat_events_lines = logcat_events_lines;
    xc_trace_logcat_main_lines = logcat_main_lines;
    xc_trace_dump_fds = dump_fds;
    xc_trace_dump_network_info = dump_network_info;

    //init for JNI callback
    xc_trace_init_callback(env);

    //create event FD
    if(0 > (xc_trace_notifier = eventfd(0, EFD_CLOEXEC))) return XCC_ERRNO_SYS;

    //register signal handler
    if(0 != (r = xcc_signal_trace_register(xc_trace_handler))) goto err2;
    XCD_LOG_DEBUG("ANR native register SIGQUIT");    // 为捕获 ANR 注册 SIGQUIT 信号处理器并启动 xc_trace_dumper 线程

    //create thread for dump trace
    if(0 != (r = pthread_create(&thd, NULL, xc_trace_dumper, NULL))) goto err1;

    return 0;

 err1:
    xcc_signal_trace_unregister();
 err2:
    close(xc_trace_notifier);
    xc_trace_notifier = -1;
    
    return r;
}

static void xc_trace_handler(int sig, siginfo_t *si, void *uc)
{
    // 信号处理程序收到 SIGQUIT(3)，唤醒 xc_trace_dumper 线程
    XCD_LOG_DEBUG("SIGQUIT handler get %d, %d", sig, xc_trace_notifier);    

    uint64_t data;
    
    (void)sig;
    (void)si;
    (void)uc;

    if(xc_trace_notifier >= 0)
    {
        data = 1;
        XCC_UTIL_TEMP_FAILURE_RETRY(write(xc_trace_notifier, &data, sizeof(data)));
    }
}

static void *xc_trace_dumper(void *arg)
{
    JNIEnv         *env = NULL;
    uint64_t        data;
    uint64_t        trace_time;
    int             fd;
    struct timeval  tv;
    char            pathname[1024];
    jstring         j_pathname;
    
    (void)arg;
    
    pthread_detach(pthread_self());

    JavaVMAttachArgs attach_args = {
        .version = XC_JNI_VERSION,
        .name    = "xcrash_trace_dp",
        .group   = NULL
    };
    if(JNI_OK != (*xc_common_vm)->AttachCurrentThread(xc_common_vm, &env, &attach_args)) goto exit;

    while(1)
    {
        //block here, waiting for sigquit
        XCC_UTIL_TEMP_FAILURE_RETRY(read(xc_trace_notifier, &data, sizeof(data)));

        XCD_LOG_DEBUG("ANR native dumper awaken");    // xc_trace_dumper 线程被唤醒，说明收到了 SIGQUIT

        //check if process already crashed
        if(xc_common_native_crashed || xc_common_java_crashed) break;

        //trace time
        if(0 != gettimeofday(&tv, NULL)) break;
        trace_time = (uint64_t)(tv.tv_sec) * 1000 * 1000 + (uint64_t)tv.tv_usec;

        //Keep only one current trace.
        if(0 != xc_trace_logs_clean()) continue;

        //create and open log file
        if((fd = xc_common_open_trace_log(pathname, sizeof(pathname), trace_time)) < 0) continue;

        //write header info
        if(0 != xc_trace_write_header(fd, trace_time)) goto end;

        //write trace info from ART runtime
        if(0 != xcc_util_write_format(fd, XCC_UTIL_THREAD_SEP"Cmd line: %s\n", xc_common_process_name)) goto end;
        if(0 != xcc_util_write_str(fd, "Mode: ART DumpForSigQuit\n")) goto end;
        if(0 != xc_trace_load_symbols())
        {
            if(0 != xcc_util_write_str(fd, "Failed to load symbols.\n")) goto end;
            goto skip;
        }
        if(0 != xc_trace_check_address_valid())
        {
            if(0 != xcc_util_write_str(fd, "Failed to check runtime address.\n")) goto end;
            goto skip;
        }
        if(dup2(fd, STDERR_FILENO) < 0)
        {
            if(0 != xcc_util_write_str(fd, "Failed to duplicate FD.\n")) goto end;
            goto skip;
        }

        XCD_LOG_DEBUG("read header from %s", pathname);    // 首先写入 header，这里从日志文件里打印出来
        print_file(pathname);

        xc_trace_dump_status = XC_TRACE_DUMP_ON_GOING;
        if(sigsetjmp(jmpenv, 1) == 0) 
        {
            if(xc_trace_is_lollipop)
                xc_trace_libart_dbg_suspend();
            // 利用 Runtime::DumpForSigQuit 打印调用栈
            XCD_LOG_DEBUG("before dump");
            xc_trace_libart_runtime_dump(*xc_trace_libart_runtime_instance, xc_trace_libcpp_cerr);    
            XCD_LOG_DEBUG("after dump, print log file");
            if(xc_trace_is_lollipop)
                xc_trace_libart_dbg_resume();
        } 
        else 
        {
            fflush(NULL);
            XCD_LOG_WARN("longjmp to skip dumping trace\n");
        }

        dup2(xc_common_fd_null, STDERR_FILENO);
                            
    skip:
        if(0 != xcc_util_write_str(fd, "\n"XCC_UTIL_THREAD_END"\n")) goto end;

        //write other info
        if(0 != xcc_util_record_logcat(fd, xc_common_process_id, xc_common_api_level, xc_trace_logcat_system_lines, xc_trace_logcat_events_lines, xc_trace_logcat_main_lines)) goto end;
        XCD_LOG_DEBUG("logcat added");    // 收集 logcat 日志
        if(xc_trace_dump_fds)
            if(0 != xcc_util_record_fds(fd, xc_common_process_id)) goto end;
        if(xc_trace_dump_network_info)
            if(0 != xcc_util_record_network_info(fd, xc_common_process_id, xc_common_api_level)) goto end;
        if(0 != xcc_meminfo_record(fd, xc_common_process_id)) goto end;
        XCD_LOG_DEBUG("meminfo added");    // 收集内存相关信息

    end:
        //close log file
        xc_common_close_trace_log(fd);
        XCD_LOG_DEBUG("close log fd");    // 日志文件写入完毕

        //rethrow SIGQUIT to ART Signal Catcher
        if(xc_trace_rethrow && (XC_TRACE_DUMP_ART_CRASH != xc_trace_dump_status)) xc_trace_send_sigquit();
        xc_trace_dump_status = XC_TRACE_DUMP_END;
        XCD_LOG_DEBUG("rethrow SIGQUIT");    // 重新抛出 SIGQUIT，不中断系统的 ANR 流程

        //JNI callback
        //Do we need to implement an emergency buffer for disk exhausted?
        if(NULL == xc_trace_cb_method) continue;
        if(NULL == (j_pathname = (*env)->NewStringUTF(env, pathname))) continue;
        (*env)->CallStaticVoidMethod(env, xc_common_cb_class, xc_trace_cb_method, j_pathname, NULL);
        XCD_LOG_DEBUG("JNI callback %s", pathname);    // 执行 NativeHandler.traceCallback
        XC_JNI_IGNORE_PENDING_EXCEPTION();
        (*env)->DeleteLocalRef(env, j_pathname);
    }
    
    (*xc_common_vm)->DetachCurrentThread(xc_common_vm);

 exit:
    xc_trace_notifier = -1;
    close(xc_trace_notifier);
    return NULL;
}
```

```java
class NativeHandler {
    private static void traceCallback(String logPath, String emergency) {
        String tag = "xcrash_dumper";
        Log.d(tag, String.format("traceCallback %s %s", emergency, logPath));    // 记录 callback 调用
        if (TextUtils.isEmpty(logPath)) {
            return;
        }

        //append memory info
        TombstoneManager.appendSection(logPath, "memory info", Util.getProcessMemoryInfo());
        Log.d(tag, "memory info appended");    // 添加额外信息

        //append background / foreground
        TombstoneManager.appendSection(logPath, "foreground", ActivityMonitor.getInstance().isApplicationForeground() ? "yes" : "no");
        Log.d(tag, "background / foreground appended");    // 添加额外信息

        //check process ANR state
        // 一段时间内轮询 ActivityManager.getProcessesInErrorState 是否为 ProcessErrorStateInfo#NOT_RESPONDING 来判断当前 APP 是否发生 ANR
        if (NativeHandler.getInstance().anrCheckProcessState) {
            if (!Util.checkProcessAnrState(NativeHandler.getInstance().ctx, NativeHandler.getInstance().anrTimeoutMs)) {
                FileManager.getInstance().recycleLogFile(new File(logPath));
                Log.d(tag, "not an ANR");    
                return; //not an ANR
            }
        }

        //delete extra ANR log files
        if (!FileManager.getInstance().maintainAnr()) {
            return;
        }

        //rename trace log file to ANR log file
        String anrLogPath = logPath.substring(0, logPath.length() - Util.traceLogSuffix.length()) + Util.anrLogSuffix;
        File traceFile = new File(logPath);
        File anrFile = new File(anrLogPath);
        if (!traceFile.renameTo(anrFile)) {
            FileManager.getInstance().recycleLogFile(traceFile);
            return;
        }
        // trace.xcrash 和 anr.xcrash 的流程是一样的，通过上面的检查逻辑判断是否为 anr，是的话将后缀改为 anr.xcrash
        Log.d(tag, String.format("%s rename to %s", logPath, anrLogPath));    

        ICrashCallback callback = NativeHandler.getInstance().anrCallback;
        if (callback != null) {
            try {
                callback.onCrash(anrLogPath, emergency);
                Log.d(tag, "anr callback invoked");    // 执行用户自定义的回调
            } catch (Exception e) {
                Log.e(tag, e.getMessage(), e);
                XCrash.getLogger().w(Util.TAG, "NativeHandler ANR callback.onCrash failed", e);
            }
        }
    }    
}

class Util {
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
                        // 轮询检查当前进程是否处于 NOT_RESPONDING 状态，这个轮询时间很关键
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
}
```

# PIXEL 4 - Android 12

主线程阻塞 20s 后进入 ANR 处理流程，收到 SIGQUIT，0.5s 后完成 ANR 日志重启抛出 SIGQUIT，4s 后 APP 进入 NOT_RESPONDING 状态

```logcat
11-25 11:27:26.760  3215  3215 D xcrash_dumper: xcrash start
11-25 11:27:26.770  3215  3215 D xcrash_dumper: xcrash native start
11-25 11:27:26.771  3215  3215 D xcrash_dumper: ANR native start
11-25 11:27:26.771  3215  3215 D xcrash_dumper: ANR native register SIGQUIT
11-25 11:27:26.772  3215  3215 D xcrash_dumper: xcrash native end

11-25 11:27:29.775  3215  3215 D xcrash_dumper: NativeHandler.anrTimeoutMs = 20s
11-25 11:27:29.779  3215  3215 D xcrash_dumper: AnrService was started
11-25 11:27:29.781  3215  3215 D xcrash_dumper: AnrService, delete 0 anr logs before anr

11-25 11:27:50.178  3215  3215 D xcrash_dumper: SIGQUIT handler get 3, 235
11-25 11:27:50.178  3215  4153 D xcrash_dumper: ANR native dumper awaken
11-25 11:27:50.205  3215  4153 D xcrash_dumper: read header from /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637810870178747_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:27:50.205  3215  4153 D xcrash_dumper: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Tombstone maker: 'xCrash 3.0.0'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Crash type: 'anr'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Start time: '2021-11-25T11:27:26.770+0800'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Crash time: '2021-11-25T11:27:50.178+0800'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: App ID: 'com.viomi.fridge.vertical'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: App version: '2.1.2.11.17'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Rooted: 'No'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: API level: '30'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: OS version: '11'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Kernel version: 'Linux version 4.14.204-g074de30eab1a-ab7083077 #1 SMP PREEMPT Thu Jan 14 21:14:19 UTC 2021 (armv8l)'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: ABI list: 'arm64-v8a,armeabi-v7a,armeabi'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Manufacturer: 'Google'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Brand: 'google'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Model: 'Pixel 4'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Build fingerprint: 'google/flame/flame:S/SPP1.210122.020.A3/7145137:user/release-keys'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: ABI: 'arm'
11-25 11:27:50.205  3215  4153 D xcrash_dumper: pid: 3215  >>> com.viomi.fridge.vertical <<<
11-25 11:27:50.205  3215  4153 D xcrash_dumper:
11-25 11:27:50.205  3215  4153 D xcrash_dumper: --- --- --- --- --- --- --- --- --- --- --- --- --- --- --- ---
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Cmd line: com.viomi.fridge.vertical
11-25 11:27:50.205  3215  4153 D xcrash_dumper: Mode: ART DumpForSigQuit

11-25 11:27:50.205  3215  4153 D xcrash_dumper: before dump
11-25 11:27:50.563  3215  4153 D xcrash_dumper: after dump, print log file
11-25 11:27:50.630  3215  4153 D xcrash_dumper: logcat added
11-25 11:27:50.683  3215  4153 D xcrash_dumper: meminfo added
11-25 11:27:50.683  3215  4153 D xcrash_dumper: close log fd
11-25 11:27:50.684  3215  4153 D xcrash_dumper: rethrow SIGQUIT

11-25 11:27:50.684  3215  4153 D xcrash_dumper: traceCallback null /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637810870178747_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:27:50.849  3215  4153 D xcrash_dumper: memory info appended
11-25 11:27:50.849  3215  4153 D xcrash_dumper: background / foreground appended
11-25 11:27:54.360  3215  4153 D xcrash_dumper: process state: 2
11-25 11:27:54.361  3215  4153 D xcrash_dumper: delete anr log file until 10
11-25 11:27:54.362  3215  4153 D xcrash_dumper: /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637810870178747_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash rename to /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637810870178747_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash

11-25 11:27:54.362  3215  4153 D xcrash_dumper: ANR Callback, null, /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637810870178747_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
11-25 11:27:54.362  3215  4153 D xcrash_dumper: anr callback invoked
11-25 11:27:54.362  3215  4153 D xcrash_dumper: JNI callback /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637810870178747_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:27:59.784  3215  3215 D xcrash_dumper: AnrService, now 1637810879, found anr logs, tombstone_00001637810870178747_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
```

# PIXEL 5 - Android 11

主线程阻塞 12s 后进入 ANR 处理流程，收到 SIGQUIT，0.5s 后完成 ANR 日志重启抛出 SIGQUIT，6s 后 APP 进入 NOT_RESPONDING 状态

```logcat
11-25 15:43:01.820  2668  2668 D xcrash_dumper: xcrash start
11-25 15:43:01.844  2668  2668 D xcrash_dumper: xcrash native start
11-25 15:43:01.845  2668  2668 D xcrash_dumper: ANR native start
11-25 15:43:01.845  2668  2668 D xcrash_dumper: ANR native register SIGQUIT
11-25 15:43:01.846  2668  2668 D xcrash_dumper: xcrash native end

11-25 15:43:04.849  2668  2668 D xcrash_dumper: NativeHandler.anrTimeoutMs = 20s
11-25 15:43:04.852  2668  2668 D xcrash_dumper: AnrService was started
11-25 15:43:04.857  2668  2668 D xcrash_dumper: AnrService, delete 0 anr logs before anr

11-25 15:43:16.061  2668  2668 D xcrash_dumper: SIGQUIT handler get 3, 168
11-25 15:43:16.061  2668  3318 D xcrash_dumper: ANR native dumper awaken
11-25 15:43:16.079  2668  3318 D xcrash_dumper: read header from /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637826196061961_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 15:43:16.079  2668  3318 D xcrash_dumper: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Tombstone maker: 'xCrash 3.0.0'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Crash type: 'anr'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Start time: '2021-11-25T15:43:01.844+0800'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Crash time: '2021-11-25T15:43:16.061+0800'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: App ID: 'com.viomi.fridge.vertical'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: App version: '2.1.2.11.17'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Rooted: 'No'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: API level: '30'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: OS version: '11'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Kernel version: 'Linux version 4.19.110-g9ceb3bf92e0a-ab6790968 #2 SMP PREEMPT Wed Aug 26 04:14:37 UTC 2020 (armv8l)'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: ABI list: 'arm64-v8a,armeabi-v7a,armeabi'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Manufacturer: 'Google'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Brand: 'google'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Model: 'Pixel 5'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Build fingerprint: 'google/redfin/redfin:11/RD1A.200810.022.A4/6835977:user/release-keys'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: ABI: 'arm'
11-25 15:43:16.079  2668  3318 D xcrash_dumper: pid: 2668  >>> com.viomi.fridge.vertical <<<
11-25 15:43:16.079  2668  3318 D xcrash_dumper:
11-25 15:43:16.079  2668  3318 D xcrash_dumper: --- --- --- --- --- --- --- --- --- --- --- --- --- --- --- ---
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Cmd line: com.viomi.fridge.vertical
11-25 15:43:16.079  2668  3318 D xcrash_dumper: Mode: ART DumpForSigQuit

11-25 15:43:16.079  2668  3318 D xcrash_dumper: before dump
11-25 15:43:16.372  2668  3318 D xcrash_dumper: after dump, print log file
11-25 15:43:16.470  2668  3318 D xcrash_dumper: logcat added
11-25 15:43:16.519  2668  3318 D xcrash_dumper: meminfo added
11-25 15:43:16.519  2668  3318 D xcrash_dumper: close log fd
11-25 15:43:16.519  2668  3318 D xcrash_dumper: rethrow SIGQUIT

11-25 15:43:16.519  2668  3318 D xcrash_dumper: traceCallback null /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637826196061961_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 15:43:16.687  2668  3318 D xcrash_dumper: memory info appended
11-25 15:43:16.688  2668  3318 D xcrash_dumper: background / foreground appended
11-25 15:43:22.715  2668  3318 D xcrash_dumper: process state: 2
11-25 15:43:22.716  2668  3318 D xcrash_dumper: delete anr log file until 10
11-25 15:43:22.716  2668  3318 D xcrash_dumper: /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637826196061961_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash rename to /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637826196061961_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash

11-25 15:43:22.716  2668  3318 D xcrash_dumper: ANR Callback, null, /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637826196061961_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
11-25 15:43:22.717  2668  3318 D xcrash_dumper: anr callback invoked
11-25 15:43:22.717  2668  3318 D xcrash_dumper: JNI callback /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637826196061961_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 15:43:34.860  2668  2668 D xcrash_dumper: AnrService, now 1637826214, found anr logs, tombstone_00001637826196061961_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
```

# MI 10 - Android 10

主线程阻塞 20s 后进入 ANR 处理流程，收到 SIGQUIT，0.5s 后完成 ANR 日志重启抛出 SIGQUIT，5s 后 APP 进入 NOT_RESPONDING 状态

```logcat
11-25 11:30:52.898  8554  8554 D xcrash_dumper: xcrash start
11-25 11:30:52.906  8554  8554 D xcrash_dumper: xcrash native start
11-25 11:30:52.907  8554  8554 D xcrash_dumper: ANR native start
11-25 11:30:52.907  8554  8554 D xcrash_dumper: ANR native register SIGQUIT
11-25 11:30:52.907  8554  8554 D xcrash_dumper: xcrash native end

11-25 11:30:55.910  8554  8554 D xcrash_dumper: NativeHandler.anrTimeoutMs = 20s
11-25 11:30:55.915  8554  8554 D xcrash_dumper: AnrService was started
11-25 11:30:55.918  8554  8554 D xcrash_dumper: AnrService, delete 0 anr logs before anr

11-25 11:31:16.430  8554  8554 D xcrash_dumper: SIGQUIT handler get 3, 151
11-25 11:31:16.430  8554  8867 D xcrash_dumper: ANR native dumper awaken
11-25 11:31:16.449  8554  8867 D xcrash_dumper: read header from /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811076430808_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:31:16.449  8554  8867 D xcrash_dumper: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Tombstone maker: 'xCrash 3.0.0'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Crash type: 'anr'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Start time: '2021-11-25T11:30:52.906+0800'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Crash time: '2021-11-25T11:31:16.430+0800'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: App ID: 'com.viomi.fridge.vertical'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: App version: '2.1.2.11.17'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Rooted: 'No'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: API level: '29'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: OS version: '10'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Kernel version: 'Linux version 4.19.81-perf-g453ac5d #1 SMP PREEMPT Tue May 19 03:00:27 CST 2020 (armv8l)'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: ABI list: 'arm64-v8a,armeabi-v7a,armeabi'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Manufacturer: 'Xiaomi'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Brand: 'Xiaomi'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Model: 'Mi 10'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Build fingerprint: 'Xiaomi/umi/umi:10/QKQ1.191117.002/V11.0.25.0.QJBCNXM:user/release-keys'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: ABI: 'arm'
11-25 11:31:16.449  8554  8867 D xcrash_dumper: pid: 8554  >>> com.viomi.fridge.vertical <<<
11-25 11:31:16.449  8554  8867 D xcrash_dumper:
11-25 11:31:16.449  8554  8867 D xcrash_dumper: --- --- --- --- --- --- --- --- --- --- --- --- --- --- --- ---
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Cmd line: com.viomi.fridge.vertical
11-25 11:31:16.449  8554  8867 D xcrash_dumper: Mode: ART DumpForSigQuit

11-25 11:31:16.449  8554  8867 D xcrash_dumper: before dump
11-25 11:31:16.624  8554  8867 D xcrash_dumper: after dump, print log file
11-25 11:31:16.758  8554  8867 D xcrash_dumper: logcat added
11-25 11:31:16.804  8554  8867 D xcrash_dumper: meminfo added
11-25 11:31:16.804  8554  8867 D xcrash_dumper: close log fd
11-25 11:31:16.804  8554  8867 D xcrash_dumper: rethrow SIGQUIT

11-25 11:31:16.804  8554  8867 D xcrash_dumper: traceCallback null /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811076430808_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:31:16.923  8554  8867 D xcrash_dumper: memory info appended
11-25 11:31:16.923  8554  8867 D xcrash_dumper: background / foreground appended
11-25 11:31:21.932  8554  8867 D xcrash_dumper: process state: 2
11-25 11:31:21.932  8554  8867 D xcrash_dumper: delete anr log file until 10
11-25 11:31:21.933  8554  8867 D xcrash_dumper: /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811076430808_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash rename to /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811076430808_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash

11-25 11:31:21.933  8554  8867 D xcrash_dumper: ANR Callback, null, /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811076430808_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
11-25 11:31:21.933  8554  8867 D xcrash_dumper: anr callback invoked
11-25 11:31:21.933  8554  8867 D xcrash_dumper: JNI callback /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811076430808_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:31:25.920  8554  8554 D xcrash_dumper: AnrService, now 1637811085, found anr logs, tombstone_00001637811076430808_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
```

# HUAWEI Mate 30 - Android 10

主线程阻塞 20s 后进入 ANR 处理流程，收到 SIGQUIT，0.7s 后完成 ANR 日志重启抛出 SIGQUIT，0.8s 后 APP 进入 NOT_RESPONDING 状态

```logcat
11-29 16:01:45.634 22368 22368 D xcrash_dumper: xcrash start
11-29 16:01:45.657 22368 22368 D xcrash_dumper: xcrash native start
11-29 16:01:45.658 22368 22368 D xcrash_dumper: ANR native start
11-29 16:01:45.658 22368 22368 D xcrash_dumper: ANR native register SIGQUIT
11-29 16:01:45.658 22368 22368 D xcrash_dumper: xcrash native end

11-29 16:01:48.660 22368 22368 D xcrash_dumper: NativeHandler.anrTimeoutMs = 20s
11-29 16:01:48.667 22368 22368 D xcrash_dumper: AnrService was started
11-29 16:01:48.674 22368 22368 D xcrash_dumper: AnrService, delete 1 anr logs before anr

11-29 16:02:09.330 22368 22368 D xcrash_dumper: SIGQUIT handler get 3, 259
11-29 16:02:09.330 22368 22783 D xcrash_dumper: ANR native dumper awaken
11-29 16:02:09.404 22368 22783 D xcrash_dumper: read header from /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001638172929330558_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-29 16:02:09.404 22368 22783 D xcrash_dumper: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Tombstone maker: 'xCrash 3.0.0'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Crash type: 'anr'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Start time: '2021-11-29T16:01:45.658+0800'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Crash time: '2021-11-29T16:02:09.330+0800'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: App ID: 'com.viomi.fridge.vertical'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: App version: '2.1.2.11.17'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Rooted: 'No'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: API level: '29'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: OS version: '10'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Kernel version: 'Linux version 4.14.116 #1 SMP PREEMPT Mon Sep 27 15:35:43 CST 2021 (armv8l)'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: ABI list: 'arm64-v8a,armeabi-v7a,armeabi'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Manufacturer: 'HUAWEI'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Brand: 'HUAWEI'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Model: 'TAS-AL00'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Build fingerprint: 'HUAWEI/TAS-AL00/HWTAS:10/HUAWEITAS-AL00/102.0.0.209C00:user/release-keys'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: ABI: 'arm'
11-29 16:02:09.404 22368 22783 D xcrash_dumper: pid: 22368  >>> com.viomi.fridge.vertical <<<
11-29 16:02:09.404 22368 22783 D xcrash_dumper:
11-29 16:02:09.404 22368 22783 D xcrash_dumper: --- --- --- --- --- --- --- --- --- --- --- --- --- --- --- ---
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Cmd line: com.viomi.fridge.vertical
11-29 16:02:09.404 22368 22783 D xcrash_dumper: Mode: ART DumpForSigQuit

11-29 16:02:09.404 22368 22783 D xcrash_dumper: before dump
11-29 16:02:09.766 22368 22783 D xcrash_dumper: after dump, print log file
11-29 16:02:09.882 22368 22783 D xcrash_dumper: logcat added
11-29 16:02:10.002 22368 22783 D xcrash_dumper: meminfo added
11-29 16:02:10.002 22368 22783 D xcrash_dumper: close log fd
11-29 16:02:10.002 22368 22783 D xcrash_dumper: rethrow SIGQUIT

11-29 16:02:10.003 22368 22783 D xcrash_dumper: traceCallback null /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001638172929330558_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-29 16:02:10.232 22368 22783 D xcrash_dumper: memory info appended
11-29 16:02:10.232 22368 22783 D xcrash_dumper: background / foreground appended
11-29 16:02:10.735 22368 22783 D xcrash_dumper: process state: 2
11-29 16:02:10.735 22368 22783 D xcrash_dumper: delete anr log file until 10
11-29 16:02:10.736 22368 22783 D xcrash_dumper: /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001638172929330558_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash rename to /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001638172929330558_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash

11-29 16:02:10.736 22368 22783 D xcrash_dumper: ANR Callback, null, /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001638172929330558_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
11-29 16:02:10.736 22368 22783 D xcrash_dumper: anr callback invoked
11-29 16:02:10.736 22368 22783 D xcrash_dumper: JNI callback /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001638172929330558_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-29 16:02:18.678 22368 22368 D xcrash_dumper: AnrService, now 1638172938, found anr logs, tombstone_00001638172929330558_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
```

# Redmi K20 - Android 9

主线程阻塞 20s 后进入 ANR 处理流程，收到 SIGQUIT，0.5s 后完成 ANR 日志重启抛出 SIGQUIT，4s 后 APP 进入 NOT_RESPONDING 状态

```logcat
11-25 11:30:24.925 10449 10449 D xcrash_dumper: xcrash start
11-25 11:30:24.936 10449 10449 D xcrash_dumper: xcrash native start
11-25 11:30:24.938 10449 10449 D xcrash_dumper: ANR native start
11-25 11:30:24.939 10449 10449 D xcrash_dumper: ANR native register SIGQUIT
11-25 11:30:24.939 10449 10449 D xcrash_dumper: xcrash native end

11-25 11:30:27.942 10449 10449 D xcrash_dumper: NativeHandler.anrTimeoutMs = 20s
11-25 11:30:27.946 10449 10449 D xcrash_dumper: AnrService was started
11-25 11:30:27.949 10449 10449 D xcrash_dumper: AnrService, delete 0 anr logs before anr

11-25 11:30:48.467 10449 10449 D xcrash_dumper: SIGQUIT handler get 3, 159
11-25 11:30:48.467 10449 10750 D xcrash_dumper: ANR native dumper awaken
11-25 11:30:48.500 10449 10750 D xcrash_dumper: read header from /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811048467666_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:30:48.500 10449 10750 D xcrash_dumper: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Tombstone maker: 'xCrash 3.0.0'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Crash type: 'anr'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Start time: '2021-11-25T11:30:24.937+0800'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Crash time: '2021-11-25T11:30:48.467+0800'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: App ID: 'com.viomi.fridge.vertical'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: App version: '2.1.2.11.17'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Rooted: 'No'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: API level: '28'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: OS version: '9'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Kernel version: 'Linux version 4.14.83-perf+ #1 SMP PREEMPT Thu Sep 26 12:23:20 CST 2019 (armv8l)'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: ABI list: 'arm64-v8a,armeabi-v7a,armeabi'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Manufacturer: 'Xiaomi'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Brand: 'Xiaomi'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Model: 'Redmi K20'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Build fingerprint: 'Xiaomi/davinci/davinci:9/PKQ1.190302.001/V11.0.3.0.PFJCNXM:user/release-keys'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: ABI: 'arm'
11-25 11:30:48.500 10449 10750 D xcrash_dumper: pid: 10449  >>> com.viomi.fridge.vertical <<<
11-25 11:30:48.500 10449 10750 D xcrash_dumper:
11-25 11:30:48.500 10449 10750 D xcrash_dumper: --- --- --- --- --- --- --- --- --- --- --- --- --- --- --- ---
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Cmd line: com.viomi.fridge.vertical
11-25 11:30:48.500 10449 10750 D xcrash_dumper: Mode: ART DumpForSigQuit

11-25 11:30:48.500 10449 10750 D xcrash_dumper: before dump
11-25 11:30:48.658 10449 10750 D xcrash_dumper: after dump, print log file
11-25 11:30:48.975 10449 10750 D xcrash_dumper: logcat added
11-25 11:30:49.063 10449 10750 D xcrash_dumper: meminfo added
11-25 11:30:49.063 10449 10750 D xcrash_dumper: close log fd
11-25 11:30:49.065 10449 10750 D xcrash_dumper: rethrow SIGQUIT

11-25 11:30:49.066 10449 10750 D xcrash_dumper: traceCallback null /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811048467666_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:30:49.179 10449 10750 D xcrash_dumper: memory info appended
11-25 11:30:49.184 10449 10750 D xcrash_dumper: background / foreground appended
11-25 11:30:52.699 10449 10750 D xcrash_dumper: process state: 2
11-25 11:30:52.700 10449 10750 D xcrash_dumper: delete anr log file until 10
11-25 11:30:52.701 10449 10750 D xcrash_dumper: /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811048467666_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash rename to /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811048467666_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash

11-25 11:30:52.701 10449 10750 D xcrash_dumper: ANR Callback, null, /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811048467666_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
11-25 11:30:52.701 10449 10750 D xcrash_dumper: anr callback invoked
11-25 11:30:52.701 10449 10750 D xcrash_dumper: JNI callback /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811048467666_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:30:57.953 10449 10449 D xcrash_dumper: AnrService, now 1637811057, found anr logs, tombstone_00001637811048467666_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
```

# HUAWEI P20 Pro - Android 8.1.0

主线程阻塞 10s 后进入 ANR 处理流程，收到 SIGQUIT，0.5s 后完成 ANR 日志重启抛出 SIGQUIT，2s 后 APP 进入 NOT_RESPONDING 状态

```logcat
11-25 11:32:17.213 23690 23690 D xcrash_dumper: xcrash start
11-25 11:32:17.222 23690 23690 D xcrash_dumper: xcrash native start
11-25 11:32:17.223 23690 23690 D xcrash_dumper: ANR native start
11-25 11:32:17.223 23690 23690 D xcrash_dumper: ANR native register SIGQUIT
11-25 11:32:17.223 23690 23690 D xcrash_dumper: xcrash native end

11-25 11:32:20.226 23690 23690 D xcrash_dumper: NativeHandler.anrTimeoutMs = 20s
11-25 11:32:20.233 23690 23690 D xcrash_dumper: AnrService was started
11-25 11:32:20.238 23690 23690 D xcrash_dumper: AnrService, delete 0 anr logs before anr

11-25 11:32:30.826 23690 23690 D xcrash_dumper: SIGQUIT handler get 3, 165
11-25 11:32:30.826 23690 23972 D xcrash_dumper: ANR native dumper awaken
11-25 11:32:30.856 23690 23972 D xcrash_dumper: read header from /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811150826614_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:32:30.856 23690 23972 D xcrash_dumper: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Tombstone maker: 'xCrash 3.0.0'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Crash type: 'anr'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Start time: '2021-11-25T11:32:17.222+0800'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Crash time: '2021-11-25T11:32:30.826+0800'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: App ID: 'com.viomi.fridge.vertical'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: App version: '2.1.2.11.17'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Rooted: 'No'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: API level: '27'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: OS version: '8.1.0'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Kernel version: 'Linux version 4.4.103+ #1 SMP PREEMPT Sun Jul 8 16:11:30 CST 2018 (armv7l)'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: ABI list: 'arm64-v8a,armeabi-v7a,armeabi'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Manufacturer: 'HUAWEI'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Brand: 'HUAWEI'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Model: 'CLT-L29'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Build fingerprint: 'HUAWEI/CLT-L29/HWCLT:8.1.0/HUAWEICLT-L29/152(C432):user/release-keys'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: ABI: 'arm'
11-25 11:32:30.856 23690 23972 D xcrash_dumper: pid: 23690  >>> com.viomi.fridge.vertical <<<
11-25 11:32:30.856 23690 23972 D xcrash_dumper:
11-25 11:32:30.856 23690 23972 D xcrash_dumper: --- --- --- --- --- --- --- --- --- --- --- --- --- --- --- ---
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Cmd line: com.viomi.fridge.vertical
11-25 11:32:30.856 23690 23972 D xcrash_dumper: Mode: ART DumpForSigQuit

11-25 11:32:30.856 23690 23972 D xcrash_dumper: before dump
11-25 11:32:31.013 23690 23972 D xcrash_dumper: after dump, print log file
11-25 11:32:31.187 23690 23972 D xcrash_dumper: logcat added
11-25 11:32:31.311 23690 23972 D xcrash_dumper: meminfo added
11-25 11:32:31.311 23690 23972 D xcrash_dumper: close log fd
11-25 11:32:31.311 23690 23972 D xcrash_dumper: rethrow SIGQUIT

11-25 11:32:31.316 23690 23972 D xcrash_dumper: traceCallback null /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811150826614_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:32:31.474 23690 23972 D xcrash_dumper: memory info appended
11-25 11:32:31.475 23690 23972 D xcrash_dumper: background / foreground appended
11-25 11:32:33.487 23690 23972 D xcrash_dumper: process state: 2
11-25 11:32:33.488 23690 23972 D xcrash_dumper: delete anr log file until 10
11-25 11:32:33.491 23690 23972 D xcrash_dumper: /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811150826614_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash rename to /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811150826614_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash

11-25 11:32:33.491 23690 23972 D xcrash_dumper: ANR Callback, null, /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811150826614_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
11-25 11:32:33.491 23690 23972 D xcrash_dumper: anr callback invoked
11-25 11:32:33.491 23690 23972 D xcrash_dumper: JNI callback /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811150826614_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
```

# vivo X9 Plus - Android 7.1.2

AnrService 运行 30s 未返回并没有触发 ANR，而后通过手动点击阻塞主线程 60s 才触发 ANR

没有继续深入测试触发 ANR 的阈值，总之 SIGQUIT 处理器还是能捕获到 ANR 的，从收到 SIGQUIT 到写入调用栈重新抛出 SIGQUIT 用时 0.8s，0.3s 后进入 ANR 状态

系统的 ANR 处理流程还会将日志写入到 /data/anr/traces.txt

```logcat
11-25 16:38:01.561 12462 12462 D xcrash_dumper: xcrash start
11-25 16:38:01.574 12462 12462 D xcrash_dumper: xcrash native start
11-25 16:38:01.576 12462 12462 D xcrash_dumper: ANR native start
11-25 16:38:01.576 12462 12462 D xcrash_dumper: ANR native register SIGQUIT
11-25 16:38:01.576 12462 12462 D xcrash_dumper: xcrash native end

11-25 16:38:04.584 12462 12462 D xcrash_dumper: NativeHandler.anrTimeoutMs = 20s
11-25 16:38:04.598 12462 12462 D xcrash_dumper: AnrService was started
11-25 16:38:04.608 12462 12462 D xcrash_dumper: AnrService, delete 0 anr logs before anr
11-25 16:38:34.620 12462 12462 D xcrash_dumper: AnrService, now 1637829514, found anr logs, 

11-25 16:41:10.125 12462 12462 D xcrash_dumper: SIGQUIT handler get 3, 143
11-25 16:41:10.125 12462 12741 D xcrash_dumper: ANR native dumper awaken
11-25 16:41:10.296 12462 12741 D xcrash_dumper: read header from /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637829670125799_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 16:41:10.296 12462 12741 D xcrash_dumper: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Tombstone maker: 'xCrash 3.0.0'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Crash type: 'anr'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Start time: '2021-11-25T16:38:01.575+0800'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Crash time: '2021-11-25T16:41:10.125+0800'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: App ID: 'com.viomi.fridge.vertical'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: App version: '2.1.2.11.17'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Rooted: 'No'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: API level: '25'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: OS version: '7.1.2'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Kernel version: 'Linux version 3.10.84-perf-g955b276c #1 SMP PREEMPT Thu Dec 10 17:42:47 CST 2020 (armv8l)'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: ABI list: 'arm64-v8a,armeabi-v7a,armeabi'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Manufacturer: 'vivo'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Brand: 'vivo'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Model: 'vivo X9Plus'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Build fingerprint: 'vivo/PD1619/PD1619:7.1.2/N2G47H/compil12101707:user/release-keys'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: ABI: 'arm'
11-25 16:41:10.296 12462 12741 D xcrash_dumper: pid: 12462  >>> com.viomi.fridge.vertical <<<
11-25 16:41:10.296 12462 12741 D xcrash_dumper: 
11-25 16:41:10.296 12462 12741 D xcrash_dumper: --- --- --- --- --- --- --- --- --- --- --- --- --- --- --- ---
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Cmd line: com.viomi.fridge.vertical
11-25 16:41:10.296 12462 12741 D xcrash_dumper: Mode: ART DumpForSigQuit

11-25 16:41:10.296 12462 12741 D xcrash_dumper: before dump
11-25 16:41:10.581 12462 12741 D xcrash_dumper: after dump, print log file
11-25 16:41:10.700 12462 12741 D xcrash_dumper: logcat added
11-25 16:41:10.851 12462 12741 D xcrash_dumper: meminfo added
11-25 16:41:10.851 12462 12741 D xcrash_dumper: close log fd
11-25 16:41:10.851 12462 12741 D xcrash_dumper: rethrow SIGQUIT

11-25 16:41:10.851 12462 12468 I art     : Thread[3,tid=12468,WaitingInMainSignalCatcherLoop,Thread*=0xdcd2b000,peer=0x12c13d30,"Signal Catcher"]: reacting to signal 3
11-25 16:41:10.851 12462 12468 I art     : 
11-25 16:41:11.132 12462 12468 I art     : Wrote stack traces to '/data/anr/traces.txt'

11-25 16:41:11.133  3743  3778 E ActivityManager: ANR in com.viomi.fridge.vertical
11-25 16:41:11.133  3743  3778 E ActivityManager: PID: 12462
11-25 16:41:11.133  3743  3778 E ActivityManager: Reason: Broadcast of Intent { act=android.intent.action.TIME_TICK flg=0x58000014 (has extras) }

11-25 16:41:11.137 12462 12741 D xcrash_dumper: memory info appended
11-25 16:41:11.143 12462 12741 D xcrash_dumper: background / foreground appended
11-25 16:41:11.145 12462 12741 D xcrash_dumper: process state: 2
11-25 16:41:11.146 12462 12741 D xcrash_dumper: delete anr log file until 10
11-25 16:41:11.147 12462 12741 D xcrash_dumper: /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637829670125799_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash rename to /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637829670125799_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash

11-25 16:41:11.147 12462 12741 D xcrash_dumper: ANR Callback, null, /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637829670125799_2.1.2.11.17__com.viomi.fridge.vertical.anr.xcrash
11-25 16:41:11.147 12462 12741 D xcrash_dumper: anr callback invoked
11-25 16:41:11.147 12462 12741 D xcrash_dumper: JNI callback /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637829670125799_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
```

# Galaxy S7 edge - Android 6.0.1

从下面的日志可以看到 AnrService 运行 30s 未返回的确触发了 ANR 并收到了 SIGQUIT（具体是用时 22s 触发 ANR），但是输出日志文件用时 14s 太长了（Runtime::DumpForSigQuit 用了 14s），当重新抛出 SIGQUIT 时 AnrService 已返回，主线程随即恢复运行，我猜测此时 APP 就不再处于 NOT_RESPONDING 状态，也就无法判别 APP 发生了 ANR

HUAWEI Mate 30 时也出现过不能判别 ANR 的情况，当时是弹出通知权限的设置页，导致 APP 进入到后台（后台 ANR）

系统 ANR 流程会输出 /data/anr/traces.txt，从 Signal Catcher 线程收到 SIGQUIT 到输出日志文件耗时 6s

```logcat
11-25 11:31:44.883 17149 17149 D xcrash_dumper: xcrash start
11-25 11:31:44.913 17149 17149 D xcrash_dumper: xcrash native start
11-25 11:31:44.913 17149 17149 D xcrash_dumper: ANR native start
11-25 11:31:44.913 17149 17149 D xcrash_dumper: ANR native register SIGQUIT
11-25 11:31:44.923 17149 17149 D xcrash_dumper: xcrash native end

11-25 11:31:47.923 17149 17149 D xcrash_dumper: NativeHandler.anrTimeoutMs = 60s
11-25 11:31:47.933 17149 17149 D xcrash_dumper: AnrService was started
11-25 11:31:47.933 17149 17149 D xcrash_dumper: AnrService, delete 0 anr logs before anr

11-25 11:32:10.173 17149 17149 D xcrash_dumper: SIGQUIT handler get 3, 150
11-25 11:32:10.173 17149 17447 D xcrash_dumper: ANR native dumper awaken
11-25 11:32:10.543 17149 17447 D xcrash_dumper: read header from /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811130176482_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:32:10.543 17149 17447 D xcrash_dumper: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Tombstone maker: 'xCrash 3.0.0'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Crash type: 'anr'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Start time: '2021-11-25T11:31:44.917+0800'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Crash time: '2021-11-25T11:32:10.176+0800'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: App ID: 'com.viomi.fridge.vertical'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: App version: '2.1.2.11.17'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Rooted: 'No'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: API level: '23'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: OS version: '6.0.1'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Kernel version: 'Linux version 3.18.20-9032921 #1 SMP PREEMPT Fri Dec 23 15:24:17 KST 2016 (armv8l)'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: ABI list: 'arm64-v8a,armeabi-v7a,armeabi'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Manufacturer: 'samsung'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Brand: 'samsung'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Model: 'SM-G9350'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Build fingerprint: 'samsung/hero2qltezc/hero2qltechn:6.0.1/MMB29M/G9350ZCU2APL3:user/release-keys'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: ABI: 'arm'
11-25 11:32:10.543 17149 17447 D xcrash_dumper: pid: 17149  >>> com.viomi.fridge.vertical <<<
11-25 11:32:10.543 17149 17447 D xcrash_dumper:
11-25 11:32:10.543 17149 17447 D xcrash_dumper: --- --- --- --- --- --- --- --- --- --- --- --- --- --- --- ---
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Cmd line: com.viomi.fridge.vertical
11-25 11:32:10.543 17149 17447 D xcrash_dumper: Mode: ART DumpForSigQuit

11-25 11:32:10.543 17149 17447 D xcrash_dumper: before dump
11-25 11:32:17.933 17149 17149 D xcrash_dumper: AnrService, now 1637811137, found anr logs,
11-25 11:32:24.053 17149 17447 D xcrash_dumper: after dump, print log file
11-25 11:32:24.383 17149 17447 D xcrash_dumper: logcat added
11-25 11:32:24.863 17149 17447 D xcrash_dumper: meminfo added
11-25 11:32:24.863 17149 17447 D xcrash_dumper: close log fd
11-25 11:32:24.863 17149 17447 D xcrash_dumper: rethrow SIGQUIT
11-25 11:32:24.863 17149 17153 I art     : Thread[2,tid=17153,WaitingInMainSignalCatcherLoop,Thread*=0xee096000,peer=0x12c160a0,"Signal Catcher"]: reacting to signal 3

11-25 11:32:24.863 17149 17447 D xcrash_dumper: traceCallback null /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811130176482_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash
11-25 11:32:25.163 17149 17447 D xcrash_dumper: memory info appended
11-25 11:32:25.183 17149 17447 D xcrash_dumper: background / foreground appended
11-25 11:32:50.103 17149 17447 D xcrash_dumper: not an ANR
11-25 11:32:50.103 17149 17447 D xcrash_dumper: JNI callback /data/user/0/com.viomi.fridge.vertical/files/tombstones/tombstone_00001637811130176482_2.1.2.11.17__com.viomi.fridge.vertical.trace.xcrash

11-25 11:32:30.233 17149 17153 I art     : Wrote stack traces to '/data/anr/traces.txt'
```

## ANR 判别逻辑的缺陷

在 `NativeHandler.traceCallback` 里会检查当前进程的状态是否为 ANR，默认是在 15s 内轮询检查进程是否处于 `ActivityManager.ProcessErrorStateInfo.NOT_RESPONDING`

```java
class NativeHandler {
    private static void traceCallback(String logPath, String emergency) {
        if (TextUtils.isEmpty(logPath)) {
            return;
        }

        //append memory info
        TombstoneManager.appendSection(logPath, "memory info", Util.getProcessMemoryInfo());

        //append background / foreground
        TombstoneManager.appendSection(logPath, "foreground", ActivityMonitor.getInstance().isApplicationForeground() ? "yes" : "no");

        //check process ANR state
        if (NativeHandler.getInstance().anrCheckProcessState) {
            if (!Util.checkProcessAnrState(NativeHandler.getInstance().ctx, NativeHandler.getInstance().anrTimeoutMs)) {
                FileManager.getInstance().recycleLogFile(new File(logPath));
                return; //not an ANR
            }
        }

        //delete extra ANR log files
        if (!FileManager.getInstance().maintainAnr()) {
            return;
        }

        //rename trace log file to ANR log file
        String anrLogPath = logPath.substring(0, logPath.length() - Util.traceLogSuffix.length()) + Util.anrLogSuffix;
        File traceFile = new File(logPath);
        File anrFile = new File(anrLogPath);
        if (!traceFile.renameTo(anrFile)) {
            FileManager.getInstance().recycleLogFile(traceFile);
            return;
        }

        ICrashCallback callback = NativeHandler.getInstance().anrCallback;
        if (callback != null) {
            try {
                callback.onCrash(anrLogPath, emergency);
            } catch (Exception e) {
                XCrash.getLogger().w(Util.TAG, "NativeHandler ANR callback.onCrash failed", e);
            }
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
}
```

但是这个轮询时长很有问题，默认 15s 在 MI 9 Android 11 上不够长，ANR 对话框还没弹出来，此时进程并不是处于 ActivityManager.ProcessErrorStateInfo.NOT_RESPONDING，导致 xCrash 判定不是 ANR，然后把日志删了，我把这个时长增加到 60s 是能判定 ANR 的

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