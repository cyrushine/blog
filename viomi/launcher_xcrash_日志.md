冰箱 Launcher 并没有独自集成 [xCrash](https://github.com/iqiyi/xCrash) 来使用，而是用了公司日志库 [Vlog](https://gitlab.viomi.com.cn/app/Android/Libs/viomi_log_lib) 里带上的 xCrash

```kotlin
/**
 * 初始化日志抓取 SDK
 */
fun initLog() {
    Vlog.init(app, "Fridge", true)
    //...
}

public static void init(Context context, String globalTag, boolean withCrash) {
    if (logics == null) {
        logics = VlogLogics.getInstance();
    }
    logics.setContext(context);
    logics.setGlobalTag(globalTag);
    if (withCrash) {
        XCrash.init(context);
    }
}
```

它里面用的是默认配置：
1. `java crash`、`native crash` 和 `ANR` 都会被捕获
2. 日志目录在 `/data/data/[pkg]/files/tombstones`
3. `java crash` 日志文件为 `tombstone_[加载 xCrash 的时间，单位为秒的时间戳，宽度为 20]_[app version]__[process name].java.xcrash`
4. `native crash` 日志文件为 `tombstone_[加载 xCrash 的时间，单位为秒的时间戳，宽度为 20]_[app version]__[process name].native.xcrash`
5. `ANR` 日志文件为 `tombstone_[加载 xCrash 的时间，单位为秒的时间戳，宽度为 20]_[app version]__[process name].trace.xcrash`

通过 IOT MQTT 下发指令来收集线上用户的 crash & ANR 日志，操作面板在【SU - IOT - 智慧终端 - 自助工具 - 日志采集】，在【SU - IOT - 控制台 - 远程运维 - 设备信息查询】可用 `DID` 查询得到 `设备 ID`，它会下发这样的指令：

```json
{
    "log_type":"launcher",
    "command_type":"upload_history",
    "history_log_stime":1636992000,
    "history_log_etime":1637044207
}
```

server 下发的日志采集指令被交由 VLog 处理，Vlog 将上传以下日志：
1. logan 日志（不包含 crash 和 anr）
2. xcrash 日志
3. anr 日志
4. 系统日志

```java
VIotHostManager.instance.registerRemoteDebugCallback(object : OnRemoteDebugListener {
    override fun remoteMessage(message: String?) {
        message?.apply {
            Vlog.commandHandle(this)
        }
    }
})

class VlogLogics {
    public void commandHandle(String url, String data, boolean uploadSystem) {
        if (!TextUtils.isEmpty(url)) {
            this.logUrl = url;
        }
        Log.i(TAG_DEFAULT, "commandHandle：" + data);
        LogUplod.commandHandle(mAppCtx, data, this);            
        if (uploadSystem) {
            Log.i(TAG_DEFAULT, "commandHandle 上传系统日志");
            ApiManager.get().updateSystemLog();                 // 上传 xcrash 日志
        }
    }    
}

class ApiManager {
    public void updateSystemLog() {
        VlogLogics logics = VlogLogics.getInstance();
        String deviceId = logics.getDeviceId();
        Context app = logics.getmAppCtx();
        if (TextUtils.isEmpty(deviceId) || app == null) {
            return;
        }
        String mac = MacUtils.getMacAddress(app);
        try {
            uploadMemoryInfo(deviceId, mac);
        } catch (Exception e) {
            e.printStackTrace();
        }
        try {
            uploadSystemLog(deviceId, mac);     // 上传 xCrash 日志
        } catch (Exception e) {
            e.printStackTrace();
        }
    }    

    private void uploadSystemLog(final String deviceId, final String mac) {
        if (isHandle) {
            Log.i(VlogLogics.TAG_DEFAULT, "uploadSystemLog handle return");
            return;
        }
        Log.i(VlogLogics.TAG_DEFAULT, "uploadSystemLog deviceId=" + deviceId + " mac=" + mac);
        try {
            AsyncTask.THREAD_POOL_EXECUTOR.execute(new Runnable() {
                @Override
                public void run() {
                    isHandle = true;
                    try {
                        realUploadCrashLog(deviceId, mac);      // 上传 xCrash 日志
                        realUploadSystemLog(deviceId, mac);
                        realUploadSystemAnr(deviceId, mac);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                    isHandle = false;
                    Log.i(VlogLogics.TAG_DEFAULT, "uploadSystemLog end");
                }
            });
        } catch (Exception e) {
            e.printStackTrace();
            Log.i(VlogLogics.TAG_DEFAULT, "uploadSystemLog end");
        }
    }   

    // 在 xCrash 默认配置下的日志目录，将所有文件压缩上传为 [yyyyMMddHHmm]_crash_logs.zip
    private void realUploadCrashLog(String deviceId, String mac) throws IOException {
        Context context = VlogLogics.getInstance().getmAppCtx();
        if (context == null) {
            return;
        }
        String dir = context.getFilesDir() + "/tombstones";
        File vendorLogsDir = new File(dir);
        if (vendorLogsDir.canRead() && vendorLogsDir.exists() && vendorLogsDir.isDirectory()) {
            String date = TimeUtil.getCurrDateStr();
            String destDir = "/sdcard/viomisyscrash";
            File destDirFile = new File(destDir);
            if (destDirFile.exists() && destDirFile.isDirectory()) {
                File[] files = destDirFile.listFiles();
                if (files != null) {
                    for (File f : files) {
                        f.delete();
                    }
                }
            }
            if (!destDirFile.exists()) {
                destDirFile.mkdirs();
            }
            upSystemLogNotZiped(deviceId, mac, destDir, date, vendorLogsDir, "crash");
        }
    }     
}
```

xcrash 日志在 `IOT 后台 - 控制台 - 远程运维 - 设备日志 - 路由器日志` 通过 `mi did` 查询，查询条件里的 `设备 DID` 指的是 `mi did` 而不是云米 did，这跟其他地方是不同的

`[yyyyMMddHHmm]_crash_logs.zip` 是 xcrash 捕获的 crash 日志文件