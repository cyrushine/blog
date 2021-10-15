# overview

在 [阅读源码系列：ANR 是怎么产生的](../../../../2020/10/20/anr/) 聊过不及时消费 input event 会产生 `ANR`：

1. `InputReaderThread` 不断地从 `/dev/input` 读取 input event 并放入 `InputDispatcher.mInboundQueue` 等待分发
2. `InputDispatcher` 寻找 input event 对应的 window 并分发到它的待发送队列里（`outboundQueue`）
3. input event 通过 socket 发送给 app process 后转移到待消费队列（`waitQueue`）
4. app main thread 在 `Choreographer.doFrame` 渲染一帧时首先会响应 input event 并通过 socket 告诉 `InputDispatcher` 从待消费队列里移除
5. 在执行第二步的过程中，如果发现 window 存在有未消费的 input event 则产生 ANR

`产生 ANR - 输出 ANR 日志 - 弹出 ANR 对话框` 整个流程的方法栈如下：

```cpp
InputDispatcher::start
InputDispatcher::dispatchOnce
InputDispatcher::processAnrsLocked
InputDispatcher::onAnrLocked(const Connection& connection)
InputDispatcher::doNotifyAnrLockedInterruptible
NativeInputManager::notifyAnr
InputManagerService.notifyANR(InputApplicationHandle inputApplicationHandle, IBinder token, String reason)
InputManagerCallback.notifyANR
InputManagerCallback.notifyANRInner
ActivityManagerService.inputDispatchingTimedOut
AnrHelper.appNotResponding
AnrRecord.appNotResponding
ProcessErrorStateRecord.appNotResponding
ActivityManagerService.UiHandler.handleMessage(SHOW_NOT_RESPONDING_UI_MSG)
AppErrors.handleShowAnrUi
ProcessRecord.ErrorDialogController.showAnrDialogs
```

# 日志

其中 `ProcessErrorStateRecord.appNotResponding` 是输出 ANR 日志的关键方法

```java
class ProcessErrorStateRecord {
    void appNotResponding(String activityShortComponentName, ApplicationInfo aInfo,
            String parentShortComponentName, WindowProcessController parentProcess,
            boolean aboveSystem, String annotation, boolean onlyDumpSelf) {
        // ...
        // Log the ANR to the main log.
        StringBuilder info = new StringBuilder();
        info.setLength(0);
        info.append("ANR in ").append(mApp.processName);
        if (activityShortComponentName != null) {
            info.append(" (").append(activityShortComponentName).append(")");
        }
        info.append("\n");
        info.append("PID: ").append(pid).append("\n");
        if (annotation != null) {
            info.append("Reason: ").append(annotation).append("\n");
        }
        if (parentShortComponentName != null
                && parentShortComponentName.equals(activityShortComponentName)) {
            info.append("Parent: ").append(parentShortComponentName).append("\n");
        }
        if (errorId != null) {
            info.append("ErrorId: ").append(errorId.toString()).append("\n");
        }
        info.append("Frozen: ").append(mApp.mOptRecord.isFrozen()).append("\n");

        // ...

        StringBuilder report = new StringBuilder();
        report.append(MemoryPressureUtil.currentPsiState());
        ProcessCpuTracker processCpuTracker = new ProcessCpuTracker(true);

        // don't dump native PIDs for background ANRs unless it is the process of interest
        // ...

        // For background ANRs, don't pass the ProcessCpuTracker to
        // avoid spending 1/2 second collecting stats to rank lastPids.
        StringWriter tracesFileException = new StringWriter();
        // To hold the start and end offset to the ANR trace file respectively.
        final long[] offsets = new long[2];
        File tracesFile = ActivityManagerService.dumpStackTraces(firstPids,
                isSilentAnr ? null : processCpuTracker, isSilentAnr ? null : lastPids,
                nativePids, tracesFileException, offsets, annotation);

        if (isMonitorCpuUsage()) {
            mService.updateCpuStatsNow();
            mService.mAppProfiler.printCurrentCpuState(report, anrTime);
            info.append(processCpuTracker.printCurrentLoad());
            info.append(report);
        }
        report.append(tracesFileException.getBuffer());

        info.append(processCpuTracker.printCurrentState(anrTime));

        Slog.e(TAG, info.toString());
        if (tracesFile == null) {
            // There is no trace file, so dump (only) the alleged culprit's threads to the log
            Process.sendSignal(pid, Process.SIGNAL_QUIT);
        } else if (offsets[1] > 0) {
            // We've dumped into the trace file successfully
            mService.mProcessList.mAppExitInfoTracker.scheduleLogAnrTrace(
                    pid, mApp.uid, mApp.getPackageList(), tracesFile, offsets[0], offsets[1]);
        }

        // Check if package is still being loaded
        float loadingProgress = 1;
        IncrementalMetrics incrementalMetrics = null;
        final PackageManagerInternal packageManagerInternal = mService.getPackageManagerInternal();
        if (mApp.info != null && mApp.info.packageName != null) {
            IncrementalStatesInfo incrementalStatesInfo =
                    packageManagerInternal.getIncrementalStatesInfo(
                            mApp.info.packageName, mApp.uid, mApp.userId);
            if (incrementalStatesInfo != null) {
                loadingProgress = incrementalStatesInfo.getProgress();
            }
            final String codePath = mApp.info.getCodePath();
            if (codePath != null && !codePath.isEmpty()
                    && IncrementalManager.isIncrementalPath(codePath)) {
                // Report in the main log that the incremental package is still loading
                Slog.e(TAG, "App ANR on incremental package " + mApp.info.packageName
                        + " which is " + ((int) (loadingProgress * 100)) + "% loaded.");
                final IBinder incrementalService = ServiceManager.getService(
                        Context.INCREMENTAL_SERVICE);
                if (incrementalService != null) {
                    final IncrementalManager incrementalManager = new IncrementalManager(
                            IIncrementalService.Stub.asInterface(incrementalService));
                    incrementalMetrics = incrementalManager.getMetrics(codePath);
                }
            }
        }
        if (incrementalMetrics != null) {
            // Report in the main log about the incremental package
            info.append("Package is ").append((int) (loadingProgress * 100)).append("% loaded.\n");
        }

        FrameworkStatsLog.write(FrameworkStatsLog.ANR_OCCURRED, mApp.uid, mApp.processName,
                activityShortComponentName == null ? "unknown" : activityShortComponentName,
                annotation,
                (mApp.info != null) ? (mApp.info.isInstantApp()
                        ? FrameworkStatsLog.ANROCCURRED__IS_INSTANT_APP__TRUE
                        : FrameworkStatsLog.ANROCCURRED__IS_INSTANT_APP__FALSE)
                        : FrameworkStatsLog.ANROCCURRED__IS_INSTANT_APP__UNAVAILABLE,
                mApp.isInterestingToUserLocked()
                        ? FrameworkStatsLog.ANROCCURRED__FOREGROUND_STATE__FOREGROUND
                        : FrameworkStatsLog.ANROCCURRED__FOREGROUND_STATE__BACKGROUND,
                mApp.getProcessClassEnum(),
                (mApp.info != null) ? mApp.info.packageName : "",
                incrementalMetrics != null /* isIncremental */, loadingProgress,
                incrementalMetrics != null ? incrementalMetrics.getMillisSinceOldestPendingRead()
                        : -1,
                incrementalMetrics != null ? incrementalMetrics.getStorageHealthStatusCode()
                        : -1,
                incrementalMetrics != null ? incrementalMetrics.getDataLoaderStatusCode()
                        : -1,
                incrementalMetrics != null && incrementalMetrics.getReadLogsEnabled(),
                incrementalMetrics != null ? incrementalMetrics.getMillisSinceLastDataLoaderBind()
                        : -1,
                incrementalMetrics != null ? incrementalMetrics.getDataLoaderBindDelayMillis()
                        : -1,
                incrementalMetrics != null ? incrementalMetrics.getTotalDelayedReads()
                        : -1,
                incrementalMetrics != null ? incrementalMetrics.getTotalFailedReads()
                        : -1,
                incrementalMetrics != null ? incrementalMetrics.getLastReadErrorUid()
                        : -1,
                incrementalMetrics != null ? incrementalMetrics.getMillisSinceLastReadError()
                        : -1,
                incrementalMetrics != null ? incrementalMetrics.getLastReadErrorNumber()
                        : 0,
                incrementalMetrics != null ? incrementalMetrics.getTotalDelayedReadsDurationMillis()
                        : -1);
        final ProcessRecord parentPr = parentProcess != null
                ? (ProcessRecord) parentProcess.mOwner : null;
        mService.addErrorToDropBox("anr", mApp, mApp.processName, activityShortComponentName,
                parentShortComponentName, parentPr, null, report.toString(), tracesFile,
                null, new Float(loadingProgress), incrementalMetrics, errorId);

        if (mApp.getWindowProcessController().appNotResponding(info.toString(),
                () -> {
                    synchronized (mService) {
                        mApp.killLocked("anr", ApplicationExitInfo.REASON_ANR, true);
                    }
                },
                () -> {
                    synchronized (mService) {
                        mService.mServices.scheduleServiceTimeoutLocked(mApp);
                    }
                })) {
            return;
        }

        synchronized (mService) {
            // mBatteryStatsService can be null if the AMS is constructed with injector only. This
            // will only happen in tests.
            if (mService.mBatteryStatsService != null) {
                mService.mBatteryStatsService.noteProcessAnr(mApp.processName, mApp.uid);
            }

            if (isSilentAnr() && !mApp.isDebugging()) {
                mApp.killLocked("bg anr", ApplicationExitInfo.REASON_ANR, true);
                return;
            }

            synchronized (mProcLock) {
                // Set the app's notResponding state, and look up the errorReportReceiver
                makeAppNotRespondingLSP(activityShortComponentName,
                        annotation != null ? "ANR " + annotation : "ANR", info.toString());
                mDialogController.setAnrController(anrController);
            }

            // mUiHandler can be null if the AMS is constructed with injector only. This will only
            // happen in tests.
            if (mService.mUiHandler != null) {
                // Bring up the infamous App Not Responding dialog
                Message msg = Message.obtain();
                msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
                msg.obj = new AppNotRespondingDialog.Data(mApp, aInfo, aboveSystem);

                mService.mUiHandler.sendMessageDelayed(msg, anrDialogDelayMs);
            }
        }        
    }
}
```

## logcat system 

> `adb logcat -v threadtime > logcat`
> 
> `-b` 指定 ring buffer，默认是 `main`, `system` 和 `crash`

从上面的代码可以看到 logcat system 里会输出一段 ANR 日志如下，包含以下几部分的信息（它们的次序可能会有所变化）：

1. `ANR` 关键字
2. 发生 ANR 的 app process name 及 android component name
3. app process id
4. 原因/描述（看得出来这是由于没有及时消费 input event 而产生的 ANR）
5. parent component (?)
6. `Load: 0.17 / 0.44 / 0.71` 读取自 `/proc/loadavg`，表示 1, 5 和 15 分钟内的系统平均负载
7. 内存压力统计信息（Pressure Stall Information），读取自 `/proc/pressure/memory`，表示任务阻塞在内存资源上的总时长


``` log
# from mi 9
09-29 16:03:03.457  1763 29602 E ActivityManager: ANR in com.example.myapplication (com.example.myapplication/.MainActivity)
09-29 16:03:03.457  1763 29602 E ActivityManager: PID: 27750
09-29 16:03:03.457  1763 29602 E ActivityManager: Reason: Input dispatching timed out (com.example.myapplication/com.example.myapplication.MainActivity, 23ec514 com.example.myapplication/com.example.myapplication.MainActivity (server) is not responding. Waited 8008ms for MotionEvent(action=DOWN))
09-29 16:03:03.457  1763 29602 E ActivityManager: Parent: com.example.myapplication/.MainActivity
09-29 16:03:03.457  1763 29602 E ActivityManager: Load: 0.17 / 0.44 / 0.71
09-29 16:03:03.457  1763 29602 E ActivityManager: ----- Output from /proc/pressure/memory -----
09-29 16:03:03.457  1763 29602 E ActivityManager: some avg10=0.00 avg60=0.00 avg300=0.02 total=32995625
09-29 16:03:03.457  1763 29602 E ActivityManager: full avg10=0.00 avg60=0.00 avg300=0.00 total=11591183
09-29 16:03:03.457  1763 29602 E ActivityManager: ----- End output from /proc/pressure/memory -----
09-29 16:03:03.457  1763 29602 E ActivityManager:
09-29 16:03:03.457  1763 29602 E ActivityManager: CPU usage from 0ms to 14680ms later (2021-09-29 16:02:48.726 to 2021-09-29 16:03:03.406):
09-29 16:03:03.457  1763 29602 E ActivityManager:   32% 8356/com.taobao.taobao: 17% user + 15% kernel / faults: 9334 minor 85 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   27% 19687/com.tencent.mm: 12% user + 15% kernel / faults: 14028 minor 129 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   13% 1763/system_server: 5.8% user + 7.5% kernel / faults: 7732 minor 29 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   12% 1464/media.codec: 8.5% user + 4.2% kernel / faults: 39180 minor 6 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   5.1% 969/surfaceflinger: 1.1% user + 3.9% kernel / faults: 582 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   4% 26354/com.xiaomi.mi_connect_service: 2.8% user + 1.1% kernel / faults: 2438 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1569/media.swcodec: 0% user + 0% kernel / faults: 21304 minor 8 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 876/media.hwcodec: 0% user + 0% kernel / faults: 6726 minor 20 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   2.3% 23059/kworker/u16:10: 0% user + 2.3% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   2.3% 5410/com.sohu.inputmethod.sogou: 1.3% user + 0.9% kernel / faults: 2961 minor 53 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   2% 2847/com.android.phone: 1.2% user + 0.7% kernel / faults: 2180 minor 12 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   1.7% 1283/adbd: 0.5% user + 1.1% kernel / faults: 3 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   1.7% 1362/cnss_diag: 1.3% user + 0.3% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   1.7% 22809/kworker/u16:2: 0% user + 1.7% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   1.2% 22965/kworker/u16:7: 0% user + 1.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1410/media.extractor: 0% user + 0% kernel / faults: 4053 minor 8 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   1% 2239/cds_ol_rx_threa: 0% user + 1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   1% 21236/kworker/u16:3: 0% user + 1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.9% 22805/kworker/u16:1: 0% user + 0.9% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.8% 150/kswapd0: 0% user + 0.8% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.8% 582/logd: 0.4% user + 0.3% kernel / faults: 1 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.8% 596/android.hardware.keymaster@4.0-service-qti: 0% user + 0.8% kernel / faults: 15 minor 4 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.8% 795/android.hardware.camera.provider@2.4-service_64: 0% user + 0.8% kernel / faults: 74 minor 29 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.8% 835/android.hardware.sensors@1.0-service: 0.4% user + 0.3% kernel / faults: 298 minor 15 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.8% 4647/com.android.nfc: 0.4% user + 0.3% kernel / faults: 1060 minor 4 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.7% 26974/com.miui.player: 0.2% user + 0.5% kernel / faults: 6 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.6% 530/irq/303-fts: 0% user + 0.6% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.6% 5337/com.miui.analytics: 0.2% user + 0.4% kernel / faults: 363 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.6% 23277/com.smile.gifmaker: 0.4% user + 0.2% kernel / faults: 119 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.5% 492/crtc_commit:131: 0% user + 0.5% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.5% 693/netd: 0.1% user + 0.4% kernel / faults: 208 minor 10 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.4% 8387/com.taobao.taobao:channel: 0.3% user + 0.1% kernel / faults: 53 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.4% 27750/com.example.myapplication: 0.4% user + 0% kernel / faults: 1733 minor 17 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.4% 861/vendor.qti.hardware.perf@2.2-service: 0.1% user + 0.2% kernel / faults: 54 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.4% 27850/com.android.browser: 0.1% user + 0.2% kernel / faults: 117 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.3% 9/rcu_preempt: 0% user + 0.3% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.3% 21238/kworker/u16:13: 0% user + 0.3% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.3% 26155/mdnsd: 0.2% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 10/rcu_sched: 0% user + 0.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 842/android.hardware.wifi@1.0-service: 0.1% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 2457/com.android.systemui: 0.2% user + 0% kernel / faults: 80 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 25386/com.tencent.mm:appbrand0: 0.2% user + 0% kernel / faults: 20 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 26203/com.smile.gifmaker:messagesdk: 0.2% user + 0% kernel / faults: 11 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 26730/logcat: 0% user + 0.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 1/init: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 12/rcuop/0: 0% user + 0.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 13/rcuos/0: 0% user + 0.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 584/servicemanager: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 664/jbd2/sda31-8: 0% user + 0.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 685/tombstoned: 0% user + 0% kernel / faults: 26 minor 52 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 958/audioserver: 0% user + 0.1% kernel / faults: 89 minor 3 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 3509/irq/33-90cd000.: 0% user + 0.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.2% 18880/kworker/u16:4: 0% user + 0.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 22/rcuop/1: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 30/rcuop/2: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 46/rcuop/4: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 652/ipacm: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 653/android.system.suspend@1.0-service: 0.1% user + 0% kernel / faults: 70 minor 7 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 692/statsd: 0% user + 0.1% kernel / faults: 73 minor 2 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 794/android.hardware.bluetooth@1.0-service-qti: 0% user + 0% kernel / faults: 15 minor 13 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 815/android.hardware.gnss@2.1-service-qti: 0% user + 0% kernel / faults: 97 minor 17 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 821/android.hardware.health@2.1-service: 0% user + 0.1% kernel / faults: 33 minor 4 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 833/android.hardware.neuralnetworks@1.3-service-qti: 0% user + 0% kernel / faults: 108 minor 34 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 862/qrtr_rx: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 1093/mi_thermald: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1371/cameraserver: 0% user + 0% kernel / faults: 46 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 1388/keystore: 0% user + 0.1% kernel / faults: 252 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1436/mediaserver: 0% user + 0% kernel / faults: 42 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 1651/msm_irqbalance: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 3449/com.google.android.gms.persistent: 0.1% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 3507/irq/32-90b6400.: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 4583/tcpdump: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 4834/com.xiaomi.mircs: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 5105/com.tencent.wework: 0% user + 0% kernel / faults: 5 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 25408/com.tencent.mm:appbrand1: 0.1% user + 0% kernel / faults: 15 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0.1% 29363/kworker/0:2: 0% user + 0.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 8/ksoftirqd/0: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 15/migration/0: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 31/rcuos/2: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 38/rcuop/3: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 39/rcuos/3: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 54/rcuop/5: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 70/rcuop/7: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 278/qseecom-unload-: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 370/irq/573-dma-gra: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 542/kworker/2:1H: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 556/ueventd: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 585/hwservicemanager: 0% user + 0% kernel / faults: 56 minor 8 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 626/vold: 0% user + 0% kernel / faults: 54 minor 1 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 740/kworker/3:1H: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 783/android.hardware.audio.service: 0% user + 0% kernel / faults: 65 minor 8 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 818/android.hardware.graphics.composer@2.4-service: 0% user + 0% kernel / faults: 215 minor 1 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1095/batteryd: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1272/wlan_logging_th: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1433/media.metrics: 0% user + 0% kernel / faults: 35 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1457/wificond: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1480/ipacm-diag: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1574/cnss-daemon: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1617/android.hardware.biometrics.fingerprint@2.1-service: 0% user + 0% kernel / faults: 24 minor 3 major
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1632/qcrild: 0% user + 0% kernel / faults: 3 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1644/hvdcp_opti: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1665/qcrild: 0% user + 0% kernel / faults: 3 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 1853/psimon: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 5514/com.miui.securitycenter.remote: 0% user + 0% kernel / faults: 21 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 8782/com.miui.powerkeeper: 0% user + 0% kernel / faults: 23 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 8829/com.taobao.taobao:sandboxed_privilege_process0: 0% user + 0% kernel / faults: 3 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 8985/com.taobao.taobao:remote: 0% user + 0% kernel / faults: 1 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 9034/cn.ticktick.task: 0% user + 0% kernel / faults: 1 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 13678/com.tencent.mm:toolsmp: 0% user + 0% kernel / faults: 17 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 13906/tv.danmaku.bili:download: 0% user + 0% kernel / faults: 2 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 14396/iptables-restore: 0% user + 0% kernel / faults: 10 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 14408/ip6tables-restore: 0% user + 0% kernel / faults: 1 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 26888/com.miui.player:remote: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 28950/kworker/1:0: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 29260/com.miui.aod:settings: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 29378/kworker/3:3: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 29558/logcat: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   0% 29564/kworker/2:0: 0% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager: 19% TOTAL: 8% user + 9.2% kernel + 0.6% iowait + 0.9% irq + 0.4% softirq
09-29 16:03:03.457  1763 29602 E ActivityManager: CPU usage from 57ms to 615ms later (2021-09-29 16:02:48.783 to 2021-09-29 16:02:49.341):
09-29 16:03:03.457  1763 29602 E ActivityManager:   75% 1763/system_server: 27% user + 47% kernel / faults: 1442 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:     54% 29602/AnrConsumer: 15% user + 38% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:     15% 1772/HeapTaskDaemon: 13% user + 2.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:     2.2% 1781/android.ui: 0% user + 2.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:     2.2% 3385/Binder:1763_F: 0% user + 2.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   4% 835/android.hardware.sensors@1.0-service: 2% user + 2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   4.2% 969/surfaceflinger: 0% user + 4.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:     2.1% 1040/Binder:969_1: 0% user + 2.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:     2.1% 1214/app: 0% user + 2.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   4.2% 1362/cnss_diag: 4.2% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   5.7% 8356/com.taobao.taobao: 2.8% user + 2.8% kernel / faults: 7 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:     2.8% 8356/m.taobao.taobao: 0% user + 2.8% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   1.8% 46/rcuop/4: 0% user + 1.8% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   1.8% 70/rcuop/7: 0% user + 1.8% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   1.9% 492/crtc_commit:131: 0% user + 1.9% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   1.9% 542/kworker/2:1H: 0% user + 1.9% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   1.9% 584/servicemanager: 0% user + 1.9% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   2.1% 1093/mi_thermald: 0% user + 2.1% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   2.2% 1665/qcrild: 0% user + 2.2% kernel / faults: 1 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:   2.3% 2239/cds_ol_rx_threa: 0% user + 2.3% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   2.3% 2847/com.android.phone: 2.3% user + 0% kernel / faults: 19 minor
09-29 16:03:03.457  1763 29602 E ActivityManager:     2.3% 2847/m.android.phone: 2.3% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:     2.3% 3437/Binder:2847_A: 2.3% user + 0% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   2.4% 3509/irq/33-90cd000.: 0% user + 2.4% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   3.2% 19687/com.tencent.mm: 0% user + 3.2% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   3.3% 22965/kworker/u16:7: 0% user + 3.3% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager:   3.5% 26155/mdnsd: 0% user + 3.5% kernel
09-29 16:03:03.457  1763 29602 E ActivityManager: 15% TOTAL: 6.1% user + 7.5% kernel + 0.9% irq + 0.4% softirq
```

### load average

用进程数来描述 CPU 负载压力

```java
class ProcessErrorStateRecord {
    void appNotResponding(...) {
        // ...
        if (isMonitorCpuUsage()) {
            mService.updateCpuStatsNow();
            synchronized (mService.mProcessCpuTracker) {
                report.append(mService.mProcessCpuTracker.printCurrentState(anrTime));
            }
            info.append(processCpuTracker.printCurrentLoad());  // 这里输出系统平均负载
            info.append(report);
        }
        // ...
    }
}

class ProcessCpuTracker {

    // 三个值分别对应 mLoad1, mLoad5 和 mLoad15
    final public String printCurrentLoad() {
        StringWriter sw = new StringWriter();
        PrintWriter pw = new FastPrintWriter(sw, false, 128);
        pw.print("Load: ");
        pw.print(mLoad1);
        pw.print(" / ");
        pw.print(mLoad5);
        pw.print(" / ");
        pw.println(mLoad15);
        pw.flush();
        return sw.toString();
    }    

    // 这三个变量来自 /proc/loadavg
    public void update() {
        // ...
        final float[] loadAverages = mLoadAverageData;
        if (Process.readProcFile("/proc/loadavg", LOAD_AVERAGE_FORMAT,
                null, null, loadAverages)) {
            float load1 = loadAverages[0];
            float load5 = loadAverages[1];
            float load15 = loadAverages[2];
            if (load1 != mLoad1 || load5 != mLoad5 || load15 != mLoad15) {
                mLoad1 = load1;
                mLoad5 = load5;
                mLoad15 = load15;
                onLoadChanged(load1, load5, load15);
            }
        }
        // ...
    }    
}
```

> man proc.5
> 
> /proc/loadavg
> 
> The  first  three  fields in this file are load average figures giving the number of jobs in the run queue (state R) or waiting for disk I/O (state D) averaged over 1, 5, and 15 minutes.  They are the same as the load average numbers given by uptime(1) and other programs.  
> 
> The fourth field consists of two numbers separated by a slash (/).  The first of these is the number of currently  runnable  kernel  scheduling entities (processes, threads).  The value after the slash is the number of kernel scheduling entities that currently exist on the system.  
> 
> The fifth field is the PID of the process that was most recently created on the system.

在绿联 6812（4 x A53@1.5G）上获取的值是 `13.31 13.74 13.81 2/1167 3888`

* 在过去的 1, 5, 15 分钟内 CPU 的平均负载是 13.31, 13.74, 13.81
* 当前有 1167 个进程，其中有 2 个正在运行
* 最近创建的一个进程是 3888

> 在 Linux 系统中，uptime、w、top 等命令都会有系统平均负载 load average 的输出，系统平均负载被定义为在特定时间间隔内运行队列中的平均进程数
> 
> 如果一个进程满足以下条件则其就会位于运行队列中：它没有在等待 I/O 操作的结果，它没有主动进入等待状态(也就是没有调用'wait')，没有被停止(例如：等待终止)
> 
> 例如：<br>
> [root@opendigest root]# uptime <br>
> 7:51pm up 2 days, 5:43, 2 users, load average: 8.13, 5.90, 4.94 <br>
> 命令输出的最后内容表示在过去的 1、5、15 分钟内运行队列中的平均进程数量
> 
> 一般来说只要每个 CPU 的当前活动进程数不大于 3 那么系统的性能就是良好的，如果每个 CPU 的任务数大于 5 那么就表示这台机器的性能有严重问题。对于上面的例子来说，假设系统有两个 CPU 那么其每个 CPU 的当前任务数为：8.13 / 2 = 4.065，这表示该系统的性能是可以接受的

这么看来 `13.81 / 4 = 3.45` 这块绿联屏的性能还是可以接受的

### PSI (Pressure Stall Information)

> Pressure Stall Information 提供了一种评估系统资源压力的方法。系统有三个基础资源：CPU、Memory 和 IO，无论这些资源配置如何增加，似乎永远无法满足软件的需求。一旦产生资源竞争，就有可能带来延迟增大，使用户体验到卡顿
> 
> 如果没有一种相对准确的方法检测系统的资源压力程度，有两种后果：一种是资源使用者过度克制，没有充分使用系统资源；另一种是经常产生资源竞争，过度使用资源导致等待延迟过大。准确的检测方法可以帮忙资源使用者确定合适的工作量，同时也可以帮助系统制定高效的资源调度策略，最大化利用系统资源，最大化改善用户体验
> 
> Facebook 在 2018 年开源了一套解决重要计算集群管理问题的 Linux 内核组件和相关工具，PSI 是其中重要的资源度量工具，它提供了一种实时检测系统资源竞争程度的方法，以竞争等待时间的方式呈现，简单而准确地供用户以及资源调度者进行决策
> 
> 在此之前，Linux 也有一些资源压力的评估方法，最具代表性的是 load average 和 vmpressure

每类资源的压力信息都通过 proc 文件系统的独立文件来提供，路径为：`/proc/pressure/memory`，`/proc/pressure/cpu` 和 `/proc/pressure/io`，其中 /proc/pressure/io 输出格式如下：

```log
some avg10=0.30 avg60=0.12 avg300=0.02 total=4170757
full avg10=0.12 avg60=0.05 avg300=0.01 total=1856503
```

* avg10、avg60、avg300 分别代表 10s、60s、300s 的时间周期内的阻塞时间百分比
* total 是总累计时间，以毫秒为单位
* some 这一行，代表至少有一个任务在某个资源上阻塞的时间占比
* full 这一行，代表所有的非 idle 任务同时被阻塞的时间占比，这期间 cpu 被完全浪费，会带来严重的性能问题

我们以 IO 的 some 和 full 来举例说明，假设在 60 秒的时间段内，系统有两个 task，在 60 秒的周期内的运行情况如下图所示：

![proc_pressure_io_1]((../../../../image/2021-07-10-deep-drive-into-anr/proc_pressure_io_1.jpg))

红色阴影部分表示任务由于等待 IO 资源而进入阻塞状态。Task A 和 Task B 同时阻塞的部分为 full，占比 16.66%；至少有一个任务阻塞（仅 Task B 阻塞的部分也计算入内）的部分为 some，占比 50%

some 和 full 都是在某一时间段内阻塞时间占比的总和，阻塞时间不一定连续，如下图所示：

![proc_pressure_io_2]((../../../../image/2021-07-10-deep-drive-into-anr/proc_pressure_io_2.jpg))

IO 和 memory 都有 some 和 full 两个维度，那是因为的确有可能系统中的所有任务都阻塞在 IO 或者 memory 资源，同时 CPU 进入 idle 状态

但是 CPU 资源不可能出现这个情况：不可能全部的 runnable 的任务都等待 CPU 资源，至少有一个 runnable 任务会被调度器选中占有 CPU 资源，因此 CPU 资源没有 full 维度的 PSI 信息呈现

通过这些阻塞占比数据，我们可以看到短期以及中长期一段时间内各种资源的压力情况，可以较精确的确定时延抖动原因，并制定对应的负载管理策略

```log
09-29 16:03:03.457  1763 29602 E ActivityManager: ----- Output from /proc/pressure/memory -----
09-29 16:03:03.457  1763 29602 E ActivityManager: some avg10=0.00 avg60=0.00 avg300=0.02 total=32995625
09-29 16:03:03.457  1763 29602 E ActivityManager: full avg10=0.00 avg60=0.00 avg300=0.00 total=11591183
09-29 16:03:03.457  1763 29602 E ActivityManager: ----- End output from /proc/pressure/memory -----
```

现在我们来看看 anr logcat 里输出的 /proc/pressure/memory 内容，full 都为零说明任务同时阻塞在内存资源上的情况没有出现，some - avg300=0.02 表明在有任务阻塞在内存资源上 6s（300s * 0.02，注意这 6s 是总和，不一定是连续的）

```java
public final class MemoryPressureUtil {
    private static final String FILE = "/proc/pressure/memory";
    private static final String TAG = "MemoryPressure";

    /**
     * @return a stanza about memory PSI to add to a report.
     */
    public static String currentPsiState() {
        final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskReads();
        StringWriter contents = new StringWriter();
        try {
            if (new File(FILE).exists()) {
                contents.append("----- Output from /proc/pressure/memory -----\n");
                contents.append(IoUtils.readFileAsString(FILE));
                contents.append("----- End output from /proc/pressure/memory -----\n\n");
            }
        } catch (IOException e) {
            Slog.e(TAG, "Could not read " + FILE, e);
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
        return contents.toString();
    }

    private MemoryPressureUtil(){}
}
```

### CPU usage

在 MI 9 上通过 `cat /proc/8538/stat` 打印出的内容如下（8538 是通过 `ps -A | grep com.tencent.mm` 查找出的微信的 pid）：

```shell
cepheus:/proc $ cat /proc/8538/stat
8538 (com.tencent.mm) S 690 690 0 0 -1 1077952832 2355958 1597 22562 6 39676 24386 2 1 20 0 209 0 464604 73654394880 26729 18446744073709551615 1 1 0 0 0 0 4608 4097 1073775868 0 0 0 17 0 0 0 2401 0 0 0 0 0 0 0 0 0 0
```

一些重要字段的含义如下（来自 `man proc.5` 里的 `/proc/[pid]/stat` 章节）：

| index | name | value | desc |
|-------|------|-------|------|
| 0 | pid          | 8538             | pid |
| 1 | comm         | (com.tencent.mm) | command, The filename of the executable, 这里是包名 |
| 2 | state        | S                | process state，进程状态：R - Running, S - Sleeping(interruptible), D - Waiting(uninterruptible), Z - Zombie ... |
| 3 | ppid         | 690              | 父进程的 PID |
| 4 | pgrp         | 690              | group ID |
| 5 |              | 0                |  |
| 6 |              | 0                |  |
| 7 |              | -1               |  |
| 8 |              | 1077952832       |  |
| 9 |              | 2355958          |  |
| 10 | minflt      | 1597            | minor page faults (which have not required loading a memory page from disk), 加载 CPU 指令时发生缺页错误，但指令以及加载至物理内存，只需将虚存映射到物理内存即可 |
| 11 | cminflt     | 22562           |  |
| 12 | majflt      | 6               | major page faults (which have required loading a memory page from disk), 同样是加载 CPU 指令时发生缺页错误，但此时需要从磁盘读取指令，比上面的情况要严重 |
| 13 | cmajflt     | 39676           |  |
| 14 | utime       | 24386           | Amount of time that this process has been scheduled in user mode, 进程运行在用户态的 CPU 时间 |
| 15 | stime       | 2               | Amount of time that this process has been scheduled in kernel mode, 进程运行在内核态的 CPU 时间 |
| 16 | cutime      | 1 |  |
| 17 | cstime      | 20 |  |
| 18 | priority    | 0 | 优先级（在创建进程的时候就设置好后续不能更改） |
| 19 | nice        | 209 | 进程最终优先级 = priorty + nice，nice 值可以在运行时动态修改 |
| 20 | num_threads | 0 |  |
| 21 | | 464604 |  |
| 22 | starttime | 73654394880 | The time the process started after system boot, 进程的运行时间（是一个从系统启动时间开始算起的相对值） |
| 23 | 26729 |  |
| 24 | 18446744073709551615 |  |
| 25 | 1 |  |
| 26 | 1 |  |
| 27 | 0 |  |
| 28 | 0 |  |
| 29 | 0 |  |
| 30 | 0 |  |
| 31 | 4608 |  |
| 32 | 4097 |  |
| 33 | 1073775868 |  |
| 34 | 0 |  |
| 35 | 0 |  |
| 36 | 0 |  |
| 37 | 17 |  |
| 38 | 0 |  |
| 39 | 0 |  |
| 40 | 0 |  |
| 41 | 2401 |  |
| 42 | 0 |  |
| 43 | 0 |  |
| 44 | 0 |  |
| 45 | 0 |  |
| 46 | 0 |  |
| 47 | 0 |  |
| 48 | 0 |  |
| 49 | 0 |  |
| 50 | 0 |  |
| 51 | 0 |  |

> Linux 对于物理内存的管理方法
> 
> 由 MMU 把物理内存分割成众多个 page，每个 page 是 4KB. 然后把 page 映射到进程的虚拟内存空间，CPU 在执行进程中的指令时以虚拟内存地址为基础，通过 map 映射进而找到物理内存中实际存放指令的地址
> 
> 缺页错误 (page fault)
> 
> 严格说这里指的是 major page fault，名字听起来挺严重，实际上并不是什么"错误"
> 
> 大致是这样，一个程序可能占几 MB，但并不是所有的指令都要同时运行，有些是在初始化时运行，有些是在特定条件下才会去运行。因此 linux 并不会把所有的指令都从磁盘加载到物理内存，那么当 cpu 在执行指令时如果发现下一条要执行的指令不在物理内存时，就会 raise a page fault 通知 MMU 把下面要执行的指令从磁盘加载到物理内存中
> 
> 还有另一种就是 minor fault
> 
> minor page fault 指的是要执行的指令实际上已经在物理内存，只是这个 page 没有映射到当前进程的虚拟内存，这时就会 raise a minor page fault 让 MMU 把这个 page 映射进当前进程的虚拟内存，因此 minor page fault 并不需要去访问磁盘
> 
> What a Swap?
> 
> 当物理内存不够时，把一些物理内存 page 写入到磁盘以腾出一些空闲的 page 出来供进程使用，这就是 swap out；反过来说当 CPU 要执行的指令被发现已经 swap out 到了磁盘中，这时就需要从磁盘把这些指令再 swap in 到物理内存中让CPU去执行
> 
> swap in 和 swap out 的操作都是比较耗时的, 频繁的 swap in 和 swap out 操作很影响系统性能


`printProcessCPU(prefix, pid, label, totalTime, user, system, iowait, irg, softIrq, minFaults, majFaults)` 打印出一行的 CPU 使用率（一行对应一个进程），各个字段解释如下：

| 字段名 | 描述 |
|-------|------|
| prefix | 前缀一般是 2、4 或更多倍数的空格，用以格式化输出 |
| pid | 进程 ID 号 |
| label | 进程名称 |
|  |  |
|  |  |
|  |  |
|  |  |


```log
[prefix][(user + system + iowait + irq + softIrq) / totalTime]% [pid]/[label]: 
[user/totalTime]% user + [system/totalTime]% kernel + [iowait/totalTime]% iowait + [irq/totalTime]% irq + [softirq/totalTime]% softirq / faults: [minFaults] minor [majFaults] major

32% 8356/com.taobao.taobao: 17% user + 15% kernel / faults: 9334 minor 85 major
```


```java
class ProcessCpuTracker {
    private void printProcessCPU(PrintWriter pw, String prefix, int pid, String label,
            int totalTime, int user, int system, int iowait, int irq, int softIrq,
            int minFaults, int majFaults) {
        pw.print(prefix);
        if (totalTime == 0) totalTime = 1;
        printRatio(pw, user+system+iowait+irq+softIrq, totalTime);
        pw.print("% ");
        if (pid >= 0) {
            pw.print(pid);
            pw.print("/");
        }
        pw.print(label);
        pw.print(": ");
        printRatio(pw, user, totalTime);
        pw.print("% user + ");
        printRatio(pw, system, totalTime);
        pw.print("% kernel");
        if (iowait > 0) {
            pw.print(" + ");
            printRatio(pw, iowait, totalTime);
            pw.print("% iowait");
        }
        if (irq > 0) {
            pw.print(" + ");
            printRatio(pw, irq, totalTime);
            pw.print("% irq");
        }
        if (softIrq > 0) {
            pw.print(" + ");
            printRatio(pw, softIrq, totalTime);
            pw.print("% softirq");
        }
        if (minFaults > 0 || majFaults > 0) {
            pw.print(" / faults:");
            if (minFaults > 0) {
                pw.print(" ");
                pw.print(minFaults);
                pw.print(" minor");
            }
            if (majFaults > 0) {
                pw.print(" ");
                pw.print(majFaults);
                pw.print(" major");
            }
        }
        pw.println();
    }   

    /**
     * 打印 numerator / denominator 至 pw，最多保留一位小数位
     */
    private void printRatio(PrintWriter pw, long numerator, long denominator) {
        long thousands = (numerator*1000)/denominator;
        long hundreds = thousands/10;
        pw.print(hundreds);
        if (hundreds < 10) {
            long remainder = thousands - (hundreds*10);
            if (remainder != 0) {
                pw.print('.');
                pw.print(remainder);
            }
        }
    }
}
```

```java
class ProcessErrorStateRecord {
    void appNotResponding(String activityShortComponentName, ApplicationInfo aInfo,
            String parentShortComponentName, WindowProcessController parentProcess,
            boolean aboveSystem, String annotation, boolean onlyDumpSelf) {
        // ...
        StringBuilder report = new StringBuilder();
        report.append(MemoryPressureUtil.currentPsiState());
        // ...
        if (isMonitorCpuUsage()) {
            mService.updateCpuStatsNow();
            mService.mAppProfiler.printCurrentCpuState(report, anrTime);
            info.append(processCpuTracker.printCurrentLoad());
            info.append(report);
        }
        // ...
    }
}

class AppProfiler {
    void printCurrentCpuState(StringBuilder report, long time) {
        synchronized (mProcessCpuTracker) {
            report.append(mProcessCpuTracker.printCurrentState(time));
        }
    }    
}

class ProcessCpuTracker {
    final public String printCurrentState(long now) {
        final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");

        buildWorkingProcs();

        StringWriter sw = new StringWriter();
        PrintWriter pw = new FastPrintWriter(sw, false, 1024);

        pw.print("CPU usage from ");
        if (now > mLastSampleTime) {
            pw.print(now-mLastSampleTime);
            pw.print("ms to ");
            pw.print(now-mCurrentSampleTime);
            pw.print("ms ago");
        } else {
            pw.print(mLastSampleTime-now);
            pw.print("ms to ");
            pw.print(mCurrentSampleTime-now);
            pw.print("ms later");
        }
        pw.print(" (");
        pw.print(sdf.format(new Date(mLastSampleWallTime)));
        pw.print(" to ");
        pw.print(sdf.format(new Date(mCurrentSampleWallTime)));
        pw.print(")");

        long sampleTime = mCurrentSampleTime - mLastSampleTime;
        long sampleRealTime = mCurrentSampleRealTime - mLastSampleRealTime;
        long percAwake = sampleRealTime > 0 ? ((sampleTime*100) / sampleRealTime) : 0;
        if (percAwake != 100) {
            pw.print(" with ");
            pw.print(percAwake);
            pw.print("% awake");
        }
        pw.println(":");

        final int totalTime = mRelUserTime + mRelSystemTime + mRelIoWaitTime
                + mRelIrqTime + mRelSoftIrqTime + mRelIdleTime;

        if (DEBUG) Slog.i(TAG, "totalTime " + totalTime + " over sample time "
                + (mCurrentSampleTime-mLastSampleTime));

        int N = mWorkingProcs.size();
        for (int i=0; i<N; i++) {
            Stats st = mWorkingProcs.get(i);
            printProcessCPU(pw, st.added ? " +" : (st.removed ? " -": "  "),
                    st.pid, st.name, (int)st.rel_uptime,
                    st.rel_utime, st.rel_stime, 0, 0, 0, st.rel_minfaults, st.rel_majfaults);
            if (!st.removed && st.workingThreads != null) {
                int M = st.workingThreads.size();
                for (int j=0; j<M; j++) {
                    Stats tst = st.workingThreads.get(j);
                    printProcessCPU(pw,
                            tst.added ? "   +" : (tst.removed ? "   -": "    "),
                            tst.pid, tst.name, (int)st.rel_uptime,
                            tst.rel_utime, tst.rel_stime, 0, 0, 0, 0, 0);
                }
            }
        }

        printProcessCPU(pw, "", -1, "TOTAL", totalTime, mRelUserTime, mRelSystemTime,
                mRelIoWaitTime, mRelIrqTime, mRelSoftIrqTime, 0, 0);

        pw.flush();
        return sw.toString();
    }    
}
```

```java
class ProcessCpuTracker {
    final void buildWorkingProcs() {
        if (!mWorkingProcsSorted) {
            mWorkingProcs.clear();
            final int N = mProcStats.size();
            for (int i=0; i<N; i++) {
                Stats stats = mProcStats.get(i);
                if (stats.working) {
                    mWorkingProcs.add(stats);
                    if (stats.threadStats != null && stats.threadStats.size() > 1) {
                        stats.workingThreads.clear();
                        final int M = stats.threadStats.size();
                        for (int j=0; j<M; j++) {
                            Stats tstats = stats.threadStats.get(j);
                            if (tstats.working) {
                                stats.workingThreads.add(tstats);
                            }
                        }
                        Collections.sort(stats.workingThreads, sLoadComparator);
                    }
                }
            }
            Collections.sort(mWorkingProcs, sLoadComparator);
            mWorkingProcsSorted = true;
        }
    }

    public void update() {
        if (DEBUG) Slog.v(TAG, "Update: " + this);

        final long nowUptime = SystemClock.uptimeMillis();
        final long nowRealtime = SystemClock.elapsedRealtime();
        final long nowWallTime = System.currentTimeMillis();

        final long[] sysCpu = mSystemCpuData;
        if (Process.readProcFile("/proc/stat", SYSTEM_CPU_FORMAT,
                null, sysCpu, null)) {
            // Total user time is user + nice time.
            final long usertime = (sysCpu[0]+sysCpu[1]) * mJiffyMillis;
            // Total system time is simply system time.
            final long systemtime = sysCpu[2] * mJiffyMillis;
            // Total idle time is simply idle time.
            final long idletime = sysCpu[3] * mJiffyMillis;
            // Total irq time is iowait + irq + softirq time.
            final long iowaittime = sysCpu[4] * mJiffyMillis;
            final long irqtime = sysCpu[5] * mJiffyMillis;
            final long softirqtime = sysCpu[6] * mJiffyMillis;

            // This code is trying to avoid issues with idle time going backwards,
            // but currently it gets into situations where it triggers most of the time. :(
            if (true || (usertime >= mBaseUserTime && systemtime >= mBaseSystemTime
                    && iowaittime >= mBaseIoWaitTime && irqtime >= mBaseIrqTime
                    && softirqtime >= mBaseSoftIrqTime && idletime >= mBaseIdleTime)) {
                mRelUserTime = (int)(usertime - mBaseUserTime);
                mRelSystemTime = (int)(systemtime - mBaseSystemTime);
                mRelIoWaitTime = (int)(iowaittime - mBaseIoWaitTime);
                mRelIrqTime = (int)(irqtime - mBaseIrqTime);
                mRelSoftIrqTime = (int)(softirqtime - mBaseSoftIrqTime);
                mRelIdleTime = (int)(idletime - mBaseIdleTime);
                mRelStatsAreGood = true;

                if (DEBUG) {
                    Slog.i("Load", "Total U:" + (sysCpu[0]*mJiffyMillis)
                          + " N:" + (sysCpu[1]*mJiffyMillis)
                          + " S:" + (sysCpu[2]*mJiffyMillis) + " I:" + (sysCpu[3]*mJiffyMillis)
                          + " W:" + (sysCpu[4]*mJiffyMillis) + " Q:" + (sysCpu[5]*mJiffyMillis)
                          + " O:" + (sysCpu[6]*mJiffyMillis));
                    Slog.i("Load", "Rel U:" + mRelUserTime + " S:" + mRelSystemTime
                          + " I:" + mRelIdleTime + " Q:" + mRelIrqTime);
                }

                mBaseUserTime = usertime;
                mBaseSystemTime = systemtime;
                mBaseIoWaitTime = iowaittime;
                mBaseIrqTime = irqtime;
                mBaseSoftIrqTime = softirqtime;
                mBaseIdleTime = idletime;

            } else {
                mRelUserTime = 0;
                mRelSystemTime = 0;
                mRelIoWaitTime = 0;
                mRelIrqTime = 0;
                mRelSoftIrqTime = 0;
                mRelIdleTime = 0;
                mRelStatsAreGood = false;
                Slog.w(TAG, "/proc/stats has gone backwards; skipping CPU update");
                return;
            }
        }

        mLastSampleTime = mCurrentSampleTime;
        mCurrentSampleTime = nowUptime;
        mLastSampleRealTime = mCurrentSampleRealTime;
        mCurrentSampleRealTime = nowRealtime;
        mLastSampleWallTime = mCurrentSampleWallTime;
        mCurrentSampleWallTime = nowWallTime;

        final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskReads();
        try {
            mCurPids = collectStats("/proc", -1, mFirst, mCurPids, mProcStats);
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }

        final float[] loadAverages = mLoadAverageData;
        if (Process.readProcFile("/proc/loadavg", LOAD_AVERAGE_FORMAT,
                null, null, loadAverages)) {
            float load1 = loadAverages[0];
            float load5 = loadAverages[1];
            float load15 = loadAverages[2];
            if (load1 != mLoad1 || load5 != mLoad5 || load15 != mLoad15) {
                mLoad1 = load1;
                mLoad5 = load5;
                mLoad15 = load15;
                onLoadChanged(load1, load5, load15);
            }
        }

        if (DEBUG) Slog.i(TAG, "*** TIME TO COLLECT STATS: "
                + (SystemClock.uptimeMillis()-mCurrentSampleTime));

        mWorkingProcsSorted = false;
        mFirst = false;
    }

    private int[] collectStats(String statsFile, int parentPid, boolean first,
            int[] curPids, ArrayList<Stats> allProcs) {

        int[] pids = Process.getPids(statsFile, curPids);
        int NP = (pids == null) ? 0 : pids.length;
        int NS = allProcs.size();
        int curStatsIndex = 0;
        for (int i=0; i<NP; i++) {
            int pid = pids[i];
            if (pid < 0) {
                NP = pid;
                break;
            }
            Stats st = curStatsIndex < NS ? allProcs.get(curStatsIndex) : null;

            if (st != null && st.pid == pid) {
                // Update an existing process...
                st.added = false;
                st.working = false;
                curStatsIndex++;
                if (DEBUG) Slog.v(TAG, "Existing "
                        + (parentPid < 0 ? "process" : "thread")
                        + " pid " + pid + ": " + st);

                if (st.interesting) {
                    final long uptime = SystemClock.uptimeMillis();

                    final long[] procStats = mProcessStatsData;
                    if (!Process.readProcFile(st.statFile.toString(),
                            PROCESS_STATS_FORMAT, null, procStats, null)) {
                        continue;
                    }

                    final long minfaults = procStats[PROCESS_STAT_MINOR_FAULTS];
                    final long majfaults = procStats[PROCESS_STAT_MAJOR_FAULTS];
                    final long utime = procStats[PROCESS_STAT_UTIME] * mJiffyMillis;
                    final long stime = procStats[PROCESS_STAT_STIME] * mJiffyMillis;

                    if (utime == st.base_utime && stime == st.base_stime) {
                        st.rel_utime = 0;
                        st.rel_stime = 0;
                        st.rel_minfaults = 0;
                        st.rel_majfaults = 0;
                        if (st.active) {
                            st.active = false;
                        }
                        continue;
                    }

                    if (!st.active) {
                        st.active = true;
                    }

                    if (parentPid < 0) {
                        getName(st, st.cmdlineFile);
                        if (st.threadStats != null) {
                            mCurThreadPids = collectStats(st.threadsDir, pid, false,
                                    mCurThreadPids, st.threadStats);
                        }
                    }

                    if (DEBUG) Slog.v("Load", "Stats changed " + st.name + " pid=" + st.pid
                            + " utime=" + utime + "-" + st.base_utime
                            + " stime=" + stime + "-" + st.base_stime
                            + " minfaults=" + minfaults + "-" + st.base_minfaults
                            + " majfaults=" + majfaults + "-" + st.base_majfaults);

                    st.rel_uptime = uptime - st.base_uptime;
                    st.base_uptime = uptime;
                    st.rel_utime = (int)(utime - st.base_utime);
                    st.rel_stime = (int)(stime - st.base_stime);
                    st.base_utime = utime;
                    st.base_stime = stime;
                    st.rel_minfaults = (int)(minfaults - st.base_minfaults);
                    st.rel_majfaults = (int)(majfaults - st.base_majfaults);
                    st.base_minfaults = minfaults;
                    st.base_majfaults = majfaults;
                    st.working = true;
                }

                continue;
            }

            if (st == null || st.pid > pid) {
                // We have a new process!
                st = new Stats(pid, parentPid, mIncludeThreads);
                allProcs.add(curStatsIndex, st);
                curStatsIndex++;
                NS++;
                if (DEBUG) Slog.v(TAG, "New "
                        + (parentPid < 0 ? "process" : "thread")
                        + " pid " + pid + ": " + st);

                final String[] procStatsString = mProcessFullStatsStringData;
                final long[] procStats = mProcessFullStatsData;
                st.base_uptime = SystemClock.uptimeMillis();
                String path = st.statFile.toString();
                //Slog.d(TAG, "Reading proc file: " + path);
                if (Process.readProcFile(path, PROCESS_FULL_STATS_FORMAT, procStatsString,
                        procStats, null)) {
                    // This is a possible way to filter out processes that
                    // are actually kernel threads...  do we want to?  Some
                    // of them do use CPU, but there can be a *lot* that are
                    // not doing anything.
                    st.vsize = procStats[PROCESS_FULL_STAT_VSIZE];
                    if (true || procStats[PROCESS_FULL_STAT_VSIZE] != 0) {
                        st.interesting = true;
                        st.baseName = procStatsString[0];
                        st.base_minfaults = procStats[PROCESS_FULL_STAT_MINOR_FAULTS];
                        st.base_majfaults = procStats[PROCESS_FULL_STAT_MAJOR_FAULTS];
                        st.base_utime = procStats[PROCESS_FULL_STAT_UTIME] * mJiffyMillis;
                        st.base_stime = procStats[PROCESS_FULL_STAT_STIME] * mJiffyMillis;
                    } else {
                        Slog.i(TAG, "Skipping kernel process pid " + pid
                                + " name " + procStatsString[0]);
                        st.baseName = procStatsString[0];
                    }
                } else {
                    Slog.w(TAG, "Skipping unknown process pid " + pid);
                    st.baseName = "<unknown>";
                    st.base_utime = st.base_stime = 0;
                    st.base_minfaults = st.base_majfaults = 0;
                }

                if (parentPid < 0) {
                    getName(st, st.cmdlineFile);
                    if (st.threadStats != null) {
                        mCurThreadPids = collectStats(st.threadsDir, pid, true,
                                mCurThreadPids, st.threadStats);
                    }
                } else if (st.interesting) {
                    st.name = st.baseName;
                    st.nameWidth = onMeasureProcessName(st.name);
                }

                if (DEBUG) Slog.v("Load", "Stats added " + st.name + " pid=" + st.pid
                        + " utime=" + st.base_utime + " stime=" + st.base_stime
                        + " minfaults=" + st.base_minfaults + " majfaults=" + st.base_majfaults);

                st.rel_utime = 0;
                st.rel_stime = 0;
                st.rel_minfaults = 0;
                st.rel_majfaults = 0;
                st.added = true;
                if (!first && st.interesting) {
                    st.working = true;
                }
                continue;
            }

            // This process has gone away!
            st.rel_utime = 0;
            st.rel_stime = 0;
            st.rel_minfaults = 0;
            st.rel_majfaults = 0;
            st.removed = true;
            st.working = true;
            allProcs.remove(curStatsIndex);
            NS--;
            if (DEBUG) Slog.v(TAG, "Removed "
                    + (parentPid < 0 ? "process" : "thread")
                    + " pid " + pid + ": " + st);
            // Decrement the loop counter so that we process the current pid
            // again the next time through the loop.
            i--;
            continue;
        }

        while (curStatsIndex < NS) {
            // This process has gone away!
            final Stats st = allProcs.get(curStatsIndex);
            st.rel_utime = 0;
            st.rel_stime = 0;
            st.rel_minfaults = 0;
            st.rel_majfaults = 0;
            st.removed = true;
            st.working = true;
            allProcs.remove(curStatsIndex);
            NS--;
            if (localLOGV) Slog.v(TAG, "Removed pid " + st.pid + ": " + st);
        }

        return pids;
    }    
}
```

```cpp
jboolean android_os_Process_readProcFile(JNIEnv* env, jobject clazz,
        jstring file, jintArray format, jobjectArray outStrings,
        jlongArray outLongs, jfloatArray outFloats)
{
    if (file == NULL || format == NULL) {
        jniThrowNullPointerException(env, NULL);
        return JNI_FALSE;
    }

    const char* file8 = env->GetStringUTFChars(file, NULL);
    if (file8 == NULL) {
        jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
        return JNI_FALSE;
    }

    ::android::base::unique_fd fd(open(file8, O_RDONLY | O_CLOEXEC));
    if (!fd.ok()) {
        if (kDebugProc) {
            ALOGW("Unable to open process file: %s\n", file8);
        }
        env->ReleaseStringUTFChars(file, file8);
        return JNI_FALSE;
    }
    env->ReleaseStringUTFChars(file, file8);

    // Most proc files we read are small, so we only go through the
    // loop once and use the stack buffer.  We allocate a buffer big
    // enough for the whole file.

    char readBufferStack[kProcReadStackBufferSize];
    std::unique_ptr<char[]> readBufferHeap;
    char* readBuffer = &readBufferStack[0];
    ssize_t readBufferSize = kProcReadStackBufferSize;
    ssize_t numberBytesRead;
    for (;;) {
        // By using pread, we can avoid an lseek to rewind the FD
        // before retry, saving a system call.
        numberBytesRead = pread(fd, readBuffer, readBufferSize, 0);
        if (numberBytesRead < 0 && errno == EINTR) {
            continue;
        }
        if (numberBytesRead < 0) {
            if (kDebugProc) {
                ALOGW("Unable to open process file: %s fd=%d\n", file8, fd.get());
            }
            return JNI_FALSE;
        }
        if (numberBytesRead < readBufferSize) {
            break;
        }
        if (readBufferSize > std::numeric_limits<ssize_t>::max() / 2) {
            if (kDebugProc) {
                ALOGW("Proc file too big: %s fd=%d\n", file8, fd.get());
            }
            return JNI_FALSE;
        }
        readBufferSize = std::max(readBufferSize * 2,
                                  kProcReadMinHeapBufferSize);
        readBufferHeap.reset();  // Free address space before getting more.
        readBufferHeap = std::make_unique<char[]>(readBufferSize);
        if (!readBufferHeap) {
            jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
            return JNI_FALSE;
        }
        readBuffer = readBufferHeap.get();
    }

    // parseProcLineArray below modifies the buffer while parsing!
    return android_os_Process_parseProcLineArray(
        env, clazz, readBuffer, 0, numberBytesRead,
        format, outStrings, outLongs, outFloats);
}

jboolean android_os_Process_parseProcLineArray(JNIEnv* env, jobject clazz,
        char* buffer, jint startIndex, jint endIndex, jintArray format,
        jobjectArray outStrings, jlongArray outLongs, jfloatArray outFloats)
{

    const jsize NF = env->GetArrayLength(format);
    const jsize NS = outStrings ? env->GetArrayLength(outStrings) : 0;
    const jsize NL = outLongs ? env->GetArrayLength(outLongs) : 0;
    const jsize NR = outFloats ? env->GetArrayLength(outFloats) : 0;

    jint* formatData = env->GetIntArrayElements(format, 0);
    jlong* longsData = outLongs ?
        env->GetLongArrayElements(outLongs, 0) : NULL;
    jfloat* floatsData = outFloats ?
        env->GetFloatArrayElements(outFloats, 0) : NULL;
    if (formatData == NULL || (NL > 0 && longsData == NULL)
            || (NR > 0 && floatsData == NULL)) {
        if (formatData != NULL) {
            env->ReleaseIntArrayElements(format, formatData, 0);
        }
        if (longsData != NULL) {
            env->ReleaseLongArrayElements(outLongs, longsData, 0);
        }
        if (floatsData != NULL) {
            env->ReleaseFloatArrayElements(outFloats, floatsData, 0);
        }
        jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
        return JNI_FALSE;
    }

    jsize i = startIndex;
    jsize di = 0;

    jboolean res = JNI_TRUE;

    for (jsize fi=0; fi<NF; fi++) {
        jint mode = formatData[fi];
        if ((mode&PROC_PARENS) != 0) {
            i++;
        } else if ((mode&PROC_QUOTES) != 0) {
            if (buffer[i] == '"') {
                i++;
            } else {
                mode &= ~PROC_QUOTES;
            }
        }
        const char term = (char)(mode&PROC_TERM_MASK);
        const jsize start = i;
        if (i >= endIndex) {
            if (kDebugProc) {
                ALOGW("Ran off end of data @%d", i);
            }
            res = JNI_FALSE;
            break;
        }

        jsize end = -1;
        if ((mode&PROC_PARENS) != 0) {
            while (i < endIndex && buffer[i] != ')') {
                i++;
            }
            end = i;
            i++;
        } else if ((mode&PROC_QUOTES) != 0) {
            while (buffer[i] != '"' && i < endIndex) {
                i++;
            }
            end = i;
            i++;
        }
        while (i < endIndex && buffer[i] != term) {
            i++;
        }
        if (end < 0) {
            end = i;
        }

        if (i < endIndex) {
            i++;
            if ((mode&PROC_COMBINE) != 0) {
                while (i < endIndex && buffer[i] == term) {
                    i++;
                }
            }
        }

        //ALOGI("Field %" PRId32 ": %" PRId32 "-%" PRId32 " dest=%" PRId32 " mode=0x%" PRIx32 "\n", i, start, end, di, mode);

        if ((mode&(PROC_OUT_FLOAT|PROC_OUT_LONG|PROC_OUT_STRING)) != 0) {
            char c = buffer[end];
            buffer[end] = 0;
            if ((mode&PROC_OUT_FLOAT) != 0 && di < NR) {
                char* end;
                floatsData[di] = strtof(buffer+start, &end);
            }
            if ((mode&PROC_OUT_LONG) != 0 && di < NL) {
                if ((mode&PROC_CHAR) != 0) {
                    // Caller wants single first character returned as one long.
                    longsData[di] = buffer[start];
                } else {
                    char* end;
                    longsData[di] = strtoll(buffer+start, &end, 10);
                }
            }
            if ((mode&PROC_OUT_STRING) != 0 && di < NS) {
                jstring str = env->NewStringUTF(buffer+start);
                env->SetObjectArrayElement(outStrings, di, str);
            }
            buffer[end] = c;
            di++;
        }
    }

    env->ReleaseIntArrayElements(format, formatData, 0);
    if (longsData != NULL) {
        env->ReleaseLongArrayElements(outLongs, longsData, 0);
    }
    if (floatsData != NULL) {
        env->ReleaseFloatArrayElements(outFloats, floatsData, 0);
    }

    return res;
}
```

`Process.getPids(dir, array)` 遍历目录 `dir` （这里传入的是 `/proc`）下的条目，找到纯数字的条目（即为 `pid`）加入到 `array`（pid array），`array` 会复用，只有当 `pid` 的数量超过 `array` 容量时才分配新的数组

```cpp
// Process.getPids(dir, array)
// 从 /proc 获取 pid 列表到 array
jintArray android_os_Process_getPids(JNIEnv* env, jobject clazz,
                                     jstring file /* /proc */, jintArray lastArray)
{
    if (file == NULL) {
        jniThrowNullPointerException(env, NULL);
        return NULL;
    }

    const char* file8 = env->GetStringUTFChars(file, NULL);
    if (file8 == NULL) {
        jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
        return NULL;
    }

    DIR* dirp = opendir(file8);

    env->ReleaseStringUTFChars(file, file8);

    if(dirp == NULL) {
        return NULL;
    }

    jsize curCount = 0;
    jint* curData = NULL;
    if (lastArray != NULL) {
        curCount = env->GetArrayLength(lastArray);
        curData = env->GetIntArrayElements(lastArray, 0);
    }

    jint curPos = 0;

    struct dirent* entry;
    while ((entry=readdir(dirp)) != NULL) {
        const char* p = entry->d_name;
        while (*p) {
            if (*p < '0' || *p > '9') break;
            p++;
        }
        if (*p != 0) continue;

        char* end;
        int pid = strtol(entry->d_name, &end, 10);
        //ALOGI("File %s pid=%d\n", entry->d_name, pid);
        if (curPos >= curCount) {
            jsize newCount = (curCount == 0) ? 10 : (curCount*2);
            jintArray newArray = env->NewIntArray(newCount);
            if (newArray == NULL) {
                closedir(dirp);
                jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
                return NULL;
            }
            jint* newData = env->GetIntArrayElements(newArray, 0);
            if (curData != NULL) {
                memcpy(newData, curData, sizeof(jint)*curCount);
                env->ReleaseIntArrayElements(lastArray, curData, 0);
            }
            lastArray = newArray;
            curCount = newCount;
            curData = newData;
        }

        curData[curPos] = pid;
        curPos++;
    }

    closedir(dirp);

    if (curData != NULL && curPos > 0) {
        qsort(curData, curPos, sizeof(jint), pid_compare);
    }

    while (curPos < curCount) {
        curData[curPos] = -1;
        curPos++;
    }

    if (curData != NULL) {
        env->ReleaseIntArrayElements(lastArray, curData, 0);
    }

    return lastArray;
}
```

# 参考

1. [Linux对内存的管理, 以及page fault的概念](https://www.jianshu.com/p/f9b8c139c2ed)
2. [Understanding page faults and memory swap-in/outs: when should you worry?](https://scoutapm.com/blog/understanding-page-faults-and-memory-swap-in-outs-when-should-you-worry)