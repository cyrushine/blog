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
ProcessRecord.appNotResponding
ActivityManagerService.UiHandler.handleMessage(SHOW_NOT_RESPONDING_UI_MSG)
AppErrors.handleShowAnrUi
ProcessRecord.ErrorDialogController.showAnrDialogs
```

# 日志

其中 `ProcessRecord.appNotResponding` 是输出 ANR 日志的关键方法

```java
class ProcessRecord {
    void appNotResponding(String activityShortComponentName, ApplicationInfo aInfo,
            String parentShortComponentName, WindowProcessController parentProcess,
            boolean aboveSystem, String annotation, boolean onlyDumpSelf) {
        // ...

        // Log the ANR to the main log.
        // 输出 logcat system 的 anr 日志
        StringBuilder info = new StringBuilder();
        info.setLength(0);
        info.append("ANR in ").append(processName);
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

        StringBuilder report = new StringBuilder();
        report.append(MemoryPressureUtil.currentPsiState());
        ProcessCpuTracker processCpuTracker = new ProcessCpuTracker(true);

        // don't dump native PIDs for background ANRs unless it is the process of interest
        String[] nativeProcs = null;
        if (isSilentAnr || onlyDumpSelf) {
            for (int i = 0; i < NATIVE_STACKS_OF_INTEREST.length; i++) {
                if (NATIVE_STACKS_OF_INTEREST[i].equals(processName)) {
                    nativeProcs = new String[] { processName };
                    break;
                }
            }
        } else {
            nativeProcs = NATIVE_STACKS_OF_INTEREST;
        }

        int[] pids = nativeProcs == null ? null : Process.getPidsForCommands(nativeProcs);
        ArrayList<Integer> nativePids = null;

        if (pids != null) {
            nativePids = new ArrayList<>(pids.length);
            for (int i : pids) {
                nativePids.add(i);
            }
        }

        // For background ANRs, don't pass the ProcessCpuTracker to
        // avoid spending 1/2 second collecting stats to rank lastPids.
        StringWriter tracesFileException = new StringWriter();
        // To hold the start and end offset to the ANR trace file respectively.
        final long[] offsets = new long[2];
        File tracesFile = ActivityManagerService.dumpStackTraces(firstPids,
                isSilentAnr ? null : processCpuTracker, isSilentAnr ? null : lastPids,
                nativePids, tracesFileException, offsets);

        if (isMonitorCpuUsage()) {
            mService.updateCpuStatsNow();
            synchronized (mService.mProcessCpuTracker) {
                report.append(mService.mProcessCpuTracker.printCurrentState(anrTime));
            }
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
                    pid, uid, getPackageList(), tracesFile, offsets[0], offsets[1]);
        }

        FrameworkStatsLog.write(FrameworkStatsLog.ANR_OCCURRED, uid, processName,
                activityShortComponentName == null ? "unknown": activityShortComponentName,
                annotation,
                (this.info != null) ? (this.info.isInstantApp()
                        ? FrameworkStatsLog.ANROCCURRED__IS_INSTANT_APP__TRUE
                        : FrameworkStatsLog.ANROCCURRED__IS_INSTANT_APP__FALSE)
                        : FrameworkStatsLog.ANROCCURRED__IS_INSTANT_APP__UNAVAILABLE,
                isInterestingToUserLocked()
                        ? FrameworkStatsLog.ANROCCURRED__FOREGROUND_STATE__FOREGROUND
                        : FrameworkStatsLog.ANROCCURRED__FOREGROUND_STATE__BACKGROUND,
                getProcessClassEnum(),
                (this.info != null) ? this.info.packageName : "");
        final ProcessRecord parentPr = parentProcess != null
                ? (ProcessRecord) parentProcess.mOwner : null;
        mService.addErrorToDropBox("anr", this, processName, activityShortComponentName,
                parentShortComponentName, parentPr, annotation, report.toString(), tracesFile,
                null);

        if (mWindowProcessController.appNotResponding(info.toString(), () -> kill("anr",
                ApplicationExitInfo.REASON_ANR, true),
                () -> {
                    synchronized (mService) {
                        mService.mServices.scheduleServiceTimeoutLocked(this);
                    }
                })) {
            return;
        }

        synchronized (mService) {
            // mBatteryStatsService can be null if the AMS is constructed with injector only. This
            // will only happen in tests.
            if (mService.mBatteryStatsService != null) {
                mService.mBatteryStatsService.noteProcessAnr(processName, uid);
            }

            if (isSilentAnr() && !isDebugging()) {
                kill("bg anr", ApplicationExitInfo.REASON_ANR, true);
                return;
            }

            // Set the app's notResponding state, and look up the errorReportReceiver
            makeAppNotRespondingLocked(activityShortComponentName,
                    annotation != null ? "ANR " + annotation : "ANR", info.toString());

            // mUiHandler can be null if the AMS is constructed with injector only. This will only
            // happen in tests.
            if (mService.mUiHandler != null) {
                // Bring up the infamous App Not Responding dialog
                Message msg = Message.obtain();
                msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
                msg.obj = new AppNotRespondingDialog.Data(this, aInfo, aboveSystem);

                mService.mUiHandler.sendMessage(msg);
            }
        }
    }
}
```

## logcat system 

从上面的代码可以看到 logcat system 里会输出一段 ANR 日志如下，前四行包含了一些基本信息：

* `ANR` 关键字
* 发生 ANR 的 app process name 及 android component name
* app process id
* 原因/描述（看得出来这是由于没有及时消费 input event 而产生的 ANR）
* parent component (?)



``` log
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