---
title: ANR 设计思路：埋雷和除雷
date: 2022-02-14 12:00:00 +0800
tags: [ANR]
---

`埋雷` 和 `除雷` 是 Android 设计 ANR 时的一个重要思路，ANR 的实质是超时，那么只需要在执行前埋下延迟爆炸的雷，如果在规定时间内执行完毕则把雷移除，否则到点雷爆炸抛出 ANR

下面以 `startService` 为例：

1. `startService` 时记录下当前时间，并埋下延时任务 `sendMessageDelayed`
2. `bindService` 表示启动完毕，移除上面埋下的雷 `SERVICE_TIMEOUT_MSG`
3. 否则到点雷爆炸 `serviceTimeout`，根据 Service 的启动时间和超时时间（`SERVICE_TIMEOUT` / `SERVICE_BACKGROUND_TIMEOUT`）找到执行超时的 Service

```java
// 埋雷，放入 ANR Message: SERVICE_TIMEOUT_MSG

// How long we wait for a service to finish executing.
static final int SERVICE_TIMEOUT = 20 * 1000 * Build.HW_TIMEOUT_MULTIPLIER; // 前台 Service 超时时间
// How long we wait for a service to finish executing.
static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;         // 后台 Service 超时时间

// AMS.startService
// ActiveServices.startServiceLocked
// ActiveServices.startServiceInnerLocked(ServiceRecord r, Intent service, ...)
// ActiveServices.startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r, boolean callerFg, boolean addToStarting)
// ActiveServices.bringUpServiceLocked
// ActiveServices.realStartServiceLocked
// ActiveServices.bumpServiceExecutingLocked
void scheduleServiceTimeoutLocked(ProcessRecord proc) {
    if (proc.mServices.numberOfExecutingServices() == 0 || proc.getThread() == null) {
        return;
    }
    Message msg = mAm.mHandler.obtainMessage(
            ActivityManagerService.SERVICE_TIMEOUT_MSG);
    msg.obj = proc;
    mAm.mHandler.sendMessageDelayed(msg, proc.mServices.shouldExecServicesFg()
            ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
}

// 看看这颗雷爆炸了会发生什么

class ActivityManagerService {
    final class MainHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case GC_BACKGROUND_PROCESSES_MSG: {
                synchronized (ActivityManagerService.this) {
                    mAppProfiler.performAppGcsIfAppropriateLocked();
                }
            } break;
            case SERVICE_TIMEOUT_MSG: {
                mServices.serviceTimeout((ProcessRecord) msg.obj);
            } break;
            // case ...
            }
        }
    }
}

class ActiveServices {
    void serviceTimeout(ProcessRecord proc) {
        String anrMessage = null;
        synchronized(mAm) {
            if (proc.isDebugging()) {
                // The app's being debugged, ignore timeout.
                return;
            }
            final ProcessServiceRecord psr = proc.mServices;
            if (psr.numberOfExecutingServices() == 0 || proc.getThread() == null) {
                return;
            }

            // sr.executingStart 是 startService 的时间
            // maxTime 是理论上推算出的、如果 Service 没有 timeout 的 startService 时间
            // 如果 sr.executingStart < maxTime 说明 Service 执行超时了
            final long now = SystemClock.uptimeMillis();
            final long maxTime =  now -
                    (psr.shouldExecServicesFg() ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
            ServiceRecord timeout = null;
            long nextTime = 0;
            for (int i = psr.numberOfExecutingServices() - 1; i >= 0; i--) {
                ServiceRecord sr = psr.getExecutingServiceAt(i);
                if (sr.executingStart < maxTime) {
                    timeout = sr;
                    break;
                }
                if (sr.executingStart > nextTime) {
                    nextTime = sr.executingStart;
                }
            }

            if (timeout != null && mAm.mProcessList.isInLruListLOSP(proc)) {
                Slog.w(TAG, "Timeout executing service: " + timeout);
                StringWriter sw = new StringWriter();
                PrintWriter pw = new FastPrintWriter(sw, false, 1024);
                pw.println(timeout);
                timeout.dump(pw, "    ");
                pw.close();
                mLastAnrDump = sw.toString();
                mAm.mHandler.removeCallbacks(mLastAnrDumpClearer);
                mAm.mHandler.postDelayed(mLastAnrDumpClearer, LAST_ANR_LIFETIME_DURATION_MSECS);
                anrMessage = "executing service " + timeout.shortInstanceName;
            } else {
                Message msg = mAm.mHandler.obtainMessage(
                        ActivityManagerService.SERVICE_TIMEOUT_MSG);
                msg.obj = proc;
                mAm.mHandler.sendMessageAtTime(msg, psr.shouldExecServicesFg()
                        ? (nextTime+SERVICE_TIMEOUT) : (nextTime + SERVICE_BACKGROUND_TIMEOUT));
            }
        }

        // 真正的 ANR 处理逻辑，具体可以看这篇 【深入 ANR：产生的根源、处理流程和日志文件】
        if (anrMessage != null) {
            mAm.mAnrHelper.appNotResponding(proc, anrMessage);
        }
    }    
}

// 除雷，Service 在规定时间内启动完毕，则需要移除 SERVICE_TIMEOUT_MSG 消息

// AMS.bindService
// AMS.bindIsolatedService
// ActiveServices.bindServiceLocked
// ActiveServices.requestServiceBindingLocked
// ActivityThread.scheduleBindService
// ActivityThread.handleBindService
// AMS.publishService
// ActiveServices.publishServiceLocked
// ActiveServices.serviceDoneExecutingLocked
class ActiveServices {
    private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing, boolean enqueueOomAdj) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "<<< DONE EXECUTING " + r
                + ": nesting=" + r.executeNesting
                + ", inDestroying=" + inDestroying + ", app=" + r.app);
        else if (DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                "<<< DONE EXECUTING " + r.shortInstanceName);
        r.executeNesting--;
        if (r.executeNesting <= 0) {
            if (r.app != null) {
                final ProcessServiceRecord psr = r.app.mServices;
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                        "Nesting at 0 of " + r.shortInstanceName);
                psr.setExecServicesFg(false);
                psr.stopExecutingService(r);
                if (psr.numberOfExecutingServices() == 0) {
                    if (DEBUG_SERVICE || DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                            "No more executingServices of " + r.shortInstanceName);

                    // Service 成功启动，移除上面埋下的雷 SERVICE_TIMEOUT_MSG
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
                } else if (r.executeFg) {
                    // Need to re-evaluate whether the app still needs to be in the foreground.
                    for (int i = psr.numberOfExecutingServices() - 1; i >= 0; i--) {
                        if (psr.getExecutingServiceAt(i).executeFg) {
                            psr.setExecServicesFg(true);
                            break;
                        }
                    }
                }
                if (inDestroying) {
                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                            "doneExecuting remove destroying " + r);
                    mDestroyingServices.remove(r);
                    r.bindings.clear();
                }
                if (enqueueOomAdj) {
                    mAm.enqueueOomAdjTargetLocked(r.app);
                } else {
                    mAm.updateOomAdjLocked(r.app, OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
                }
            }
            r.executeFg = false;
            if (r.tracker != null) {
                synchronized (mAm.mProcessStats.mLock) {
                    final int memFactor = mAm.mProcessStats.getMemFactorLocked();
                    final long now = SystemClock.uptimeMillis();
                    r.tracker.setExecuting(false, memFactor, now);
                    r.tracker.setForeground(false, memFactor, now);
                    if (finishing) {
                        r.tracker.clearCurrentOwner(r, false);
                        r.tracker = null;
                    }
                }
            }
            if (finishing) {
                if (r.app != null && !r.app.isPersistent()) {
                    stopServiceAndUpdateAllowlistManagerLocked(r);
                }
                r.setProcess(null, null, 0, null);
            }
        }
    }    
}
```
