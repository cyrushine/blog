---
title: Matrix - ANR 原理及与 xCrash ANR 的对比
date: 2021-12-03 12:00:00 +0800
tags: [xCrash, ANR, 崩溃, matrix]
---

上两篇文章 [崩溃日志收集库 xCrash 浅析](../../../../2021/06/22/xcrash/) 和 [xCrash ANR 兼容性测试](../../../../2021/11/29/xcrash-anr-compatibility/) 介绍了 [xCrash](https://github.com/iqiyi/xCrash) 是如何处理 ANR 事件的：

* 注册 `SIGQUIT` 信号处理器捕获 ANR
* 查询是否处于 `NOT_RESPONDING` 状态以判别是否自己发生了 ANR
* 寻找符号 `Runtime::DumpForSigQuit` 并调用以打印 ANR 日志

过程中提出了几个问题：

1. `NOT_RESPONDING` 不够准确，后台 ANR 和主线程从阻塞恢复的情况下会漏掉 ANR
2. 自己调用 `Runtime::DumpForSigQuit` 打印了一份 ANR 日志，重新抛出 `SIGQUIT` 后 `Signal Catcher` 线程又打印了一份 ANR 日志，重复打印了

本文主要是分析 [Matrix](https://github.com/Tencent/matrix) 处理 ANR 的流程，顺带看看对于上述问题 Matrix 有没提出更好的解决方案

# SIGQUIT Signal Handler

Matrix 也是通过注册 SIGQUIT 信号处理器来捕获 ANR 事件的，看来这种方法是比较稳定且被大家认可的，调用栈如下：

```cpp
TracePlugin.start
SignalAnrTracer.onStartTrace
SignalAnrTracer.onAlive
SignalAnrTracer.nativeInitSignalAnrDetective

static std::optional<AnrDumper> sAnrDumper;
static void nativeInitSignalAnrDetective(JNIEnv *env, jclass, jstring anrTracePath, jstring printTracePath) {
    const char* anrTracePathChar = env->GetStringUTFChars(anrTracePath, nullptr);
    const char* printTracePathChar = env->GetStringUTFChars(printTracePath, nullptr);
    anrTracePathstring = std::string(anrTracePathChar);
    printTracePathstring = std::string(printTracePathChar);
    sAnrDumper.emplace(anrTracePathChar, printTracePathChar, anrDumpCallback);
}

class AnrDumper : public SignalHandler

SignalHandler::SignalHandler() {
    std::lock_guard<std::mutex> lock(sHandlerStackMutex);

    if (!sHandlerStack)
        sHandlerStack = new std::vector<SignalHandler*>;

    installAlternateStackLocked();
    installHandlersLocked();
    sHandlerStack->push_back(this);
}

// 安装 SIGQUIT 信号处理器
const int TARGET_SIG = SIGQUIT;
bool SignalHandler::installHandlersLocked() {
    if (sHandlerInstalled) {
        return false;
    }

    if (sigaction(TARGET_SIG, nullptr, &sOldHandlers) == -1) {
        return false;
    }

    struct sigaction sa{};
    sa.sa_sigaction = signalHandler;
    sa.sa_flags = SA_ONSTACK | SA_SIGINFO | SA_RESTART;

    if (sigaction(TARGET_SIG, &sa, nullptr) == -1) {
        ALOGV("Signal handler cannot be installed");
    }

    sHandlerInstalled = true;
    ALOGV("Signal handler installed.");
    return true;
}

// 收到 SIGQUIT 信号后的回调
void SignalHandler::signalHandler(int sig, siginfo_t* info, void* uc) {
    ALOGV("Entered signal handler.");

    std::unique_lock<std::mutex> lock(sHandlerStackMutex);

    for (auto it = sHandlerStack->rbegin(); it != sHandlerStack->rend(); ++it) {
        (*it)->handleSignal(sig, info, uc);
    }

    lock.unlock();
}

SignalHandler::Result AnrDumper::handleSignal(int sig, const siginfo_t *info, void *uc) {
    // Only process SIGQUIT, which indicates an ANR.
    if (sig != SIGQUIT) return NOT_HANDLED;
    int fromPid1 = info->_si_pad[3];
    int fromPid2 = info->_si_pad[4]; // sender pid
    int myPid = getpid();

    pthread_t thd;
    if (fromPid1 != myPid && fromPid2 != myPid) {
        pthread_create(&thd, nullptr, anrCallback, nullptr);
    } else {
        pthread_create(&thd, nullptr, siUserCallback, nullptr);
    }
    pthread_detach(thd);

    return HANDLED_NO_RETRIGGER;
}
```

## fromPid1 和 fromPid2 分别是哪些进程？

经测试，如果自己发生 ANR 则 `fromPid1:0 fromPid2:1719 myPid:7414`，自己给自己发送 SIGQUIT 的话是 `fromPid1:0 fromPid2:7414 myPid:7414`，也就是说 `fromPid2` 表示发送信号的进程，正常情况下是走 `anrCallback`，下面先分析这种情况

```text
system         1719    703 9940960 199048 0                   0 S system_server
u10_a369       7414    703 5513524 135784 0                   0 S sample.tencent.matrix
```

# 判别是否自己发生了 ANR

Matrix 在判别是否发生 ANR 事件时，在 `NOT_RESPONDING` 的基础上，增加了【主线程是否阻塞】这一条件，增大了 ANR 捕获率：

1. 首先检查主线程是否阻塞，如果阻塞则一定发生了 ANR
2. 20s 内每隔 500ms 轮询是否处于 `NOT_RESPONDING` 状态（`SignalAnrTracer.checkErrorState`），这跟 xCrash 的逻辑是一致的

`SignalAnrTracer.isMainThreadBlocked` 判断主线程是否阻塞了，可以先看看 [面试官家常之Handler、MessageQueue 和 Looper](../../../../2020/09/27/handler-messagequeue-looper/) 回顾下 `MessageQueue` 的基础知识：

1. MessageQueue 是个单向链表，按 `Message.when` 自然序排
2. 表头是 `MessageQueue.mMessages`

如果主线程阻塞了，那么表头一定是超时的（`when - SystemClock.uptimeMillis()` 为负），当超时时间大于阈值时（前台 2s，后台 10s，这个阈值是怎么来的呢？）就可以认定是发生阻塞了

增加了这一条件，后台 ANR 也不会成为漏网之鱼了；而且判断过程发生在 dump anr 之前，不会因为 dump anr 耗时过长、主线程从阻塞及时恢复而导致的漏判

```cpp
static void *anrCallback(void* arg) {
    anrDumpCallback();

    if (strlen(mAnrTraceFile) > 0) {
        hookAnrTraceWrite(false);
    }

    sendSigToSignalCatcher();
    return nullptr;
}

bool anrDumpCallback() {
    JNIEnv *env = JniInvocation::getEnv();
    if (!env) return false;
    env->CallStaticVoidMethod(gJ.AnrDetective, gJ.AnrDetector_onANRDumped);
    return true;
}
```

```java
public class SignalAnrTracer extends Tracer {
    private static final String CHECK_ANR_STATE_THREAD_NAME = "Check-ANR-State-Thread";
    private static final int CHECK_ERROR_STATE_INTERVAL = 500;
    private static final int ANR_DUMP_MAX_TIME = 20000;
    private static final int CHECK_ERROR_STATE_COUNT =
            ANR_DUMP_MAX_TIME / CHECK_ERROR_STATE_INTERVAL;
    private static final long FOREGROUND_MSG_THRESHOLD = -2000;
    private static final long BACKGROUND_MSG_THRESHOLD = -10000;                

    private static void onANRDumped() {
        currentForeground = AppForegroundUtil.isInterestingToUser();
        boolean needReport = isMainThreadBlocked();

        if (needReport) {
            report(false);
        } else {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    checkErrorStateCycle();
                }
            }, CHECK_ANR_STATE_THREAD_NAME).start();
        }
    }   

    private static void checkErrorStateCycle() {
        int checkErrorStateCount = 0;
        while (checkErrorStateCount < CHECK_ERROR_STATE_COUNT) {
            try {
                checkErrorStateCount++;
                boolean myAnr = checkErrorState();
                if (myAnr) {
                    report(true);
                    break;
                }

                Thread.sleep(CHECK_ERROR_STATE_INTERVAL);
            } catch (Throwable t) {
                MatrixLog.e(TAG, "checkErrorStateCycle error, e : " + t.getMessage());
                break;
            }
        }
    }    

    // 轮询 NOT_RESPONDING 状态
    private static boolean checkErrorState() {
        try {
            Application application =
                    sApplication == null ? Matrix.with().getApplication() : sApplication;
            ActivityManager am = (ActivityManager) application
                    .getSystemService(Context.ACTIVITY_SERVICE);

            List<ActivityManager.ProcessErrorStateInfo> procs = am.getProcessesInErrorState();
            if (procs == null) return false;

            for (ActivityManager.ProcessErrorStateInfo proc : procs) {
                MatrixLog.i(TAG, "[checkErrorState] found Error State proccessName = %s, proc.condition = %d", proc.processName, proc.condition);

                if (proc.uid != android.os.Process.myUid()
                        && proc.condition == ActivityManager.ProcessErrorStateInfo.NOT_RESPONDING) {
                    MatrixLog.i(TAG, "maybe received other apps ANR signal");
                }

                if (proc.pid != android.os.Process.myPid()) continue;

                if (proc.condition != ActivityManager.ProcessErrorStateInfo.NOT_RESPONDING) {
                    continue;
                }

                return true;
            }
            return false;
        } catch (Throwable t) {
            MatrixLog.e(TAG, "[checkErrorState] error : %s", t.getMessage());
        }
        return false;
    }    

    // 判断主线程是否发生了阻塞
    private static boolean isMainThreadBlocked() {
        try {
            MessageQueue mainQueue = Looper.getMainLooper().getQueue();
            Field field = mainQueue.getClass().getDeclaredField("mMessages");
            field.setAccessible(true);
            final Message mMessage = (Message) field.get(mainQueue);
            if (mMessage != null) {
                anrMessageString = mMessage.toString();
                long when = mMessage.getWhen();
                if (when == 0) {
                    return false;
                }
                long time = when - SystemClock.uptimeMillis();
                anrMessageWhen = time;
                long timeThreshold = BACKGROUND_MSG_THRESHOLD;
                if (currentForeground) {
                    timeThreshold = FOREGROUND_MSG_THRESHOLD;
                }
                return time < timeThreshold;
            }
        } catch (Exception e) {
            return false;
        }
        return false;
    }     
}
```

# hook Signal Catcher

如何 dump anr 日志，Matrix 也与 xCrash 有着不同的实现方式，Matrix 并没有自己去 dump anr 日志，而是通过 hook 系统调用后重新发送 SIGQUIT 给 Signal Catcher 线程，让它去执行系统的 ANR 处理流程，从而自己也能拿到一份 ANR 日志，避免了重复 dump anr

hook 用的是 [xHook](https://github.com/iqiyi/xHook)

`Signal Catcher` 线程 ID 则是遍历 `/proc/[pid]/task/[tid]/comm` 对比线程名，以及对比 `/proc/[tid]/status` 里的 `magic code` 来确定的

```cpp
static void *anrCallback(void* arg) {
    anrDumpCallback();

    if (strlen(mAnrTraceFile) > 0) {
        hookAnrTraceWrite(false);
    }

    sendSigToSignalCatcher();
    return nullptr;
}

void hookAnrTraceWrite(bool isSiUser) {
    int apiLevel = getApiLevel();
    if (apiLevel < 19) {
        return;
    }

    if (!fromMyPrintTrace && isSiUser) {
        return;
    }

    if (isHooking) {
        return;
    }

    isHooking = true;

    if (apiLevel >= 27) {
        void *libcutils_info = xhook_elf_open("/system/lib64/libcutils.so");
        if(!libcutils_info) {
            libcutils_info = xhook_elf_open("/system/lib/libcutils.so");
        }
        xhook_got_hook_symbol(libcutils_info, "connect", (void*) my_connect, (void**) (&original_connect));
    } else {
        void* libart_info = xhook_elf_open("libart.so");
        xhook_got_hook_symbol(libart_info, "open", (void*) my_open, (void**) (&original_open));
    }

    if (apiLevel >= 30 || apiLevel == 25 || apiLevel == 24) {
        void* libc_info = xhook_elf_open("libc.so");
        xhook_got_hook_symbol(libc_info, "write", (void*) my_write, (void**) (&original_write));
    } else if (apiLevel == 29) {
        void* libbase_info = xhook_elf_open("/system/lib64/libbase.so");
        if(!libbase_info) {
            libbase_info = xhook_elf_open("/system/lib/libbase.so");
        }
        xhook_got_hook_symbol(libbase_info, "write", (void*) my_write, (void**) (&original_write));
        xhook_elf_close(libbase_info);
    } else {
        void* libart_info = xhook_elf_open("libart.so");
        xhook_got_hook_symbol(libart_info, "write", (void*) my_write, (void**) (&original_write));
    }
}

static void sendSigToSignalCatcher() {
    int tid = getSignalCatcherThreadId();
    syscall(SYS_tgkill, getpid(), tid, SIGQUIT);
}

static int getSignalCatcherThreadId() {
    char taskDirPath[128];
    DIR *taskDir;
    long long sigblk;
    int signalCatcherTid = -1;
    int firstSignalCatcherTid = -1;

    snprintf(taskDirPath, sizeof(taskDirPath), "/proc/%d/task", getpid());
    if ((taskDir = opendir(taskDirPath)) == nullptr) {
        return -1;
    }
    struct dirent *dent;
    pid_t tid;
    while ((dent = readdir(taskDir)) != nullptr) {
        tid = atoi(dent->d_name);
        if (tid <= 0) {
            continue;
        }

        char threadName[1024];
        char commFilePath[1024];
        snprintf(commFilePath, sizeof(commFilePath), "/proc/%d/task/%d/comm", getpid(), tid);

        Support::readFileAsString(commFilePath, threadName, sizeof(threadName));

        if (strncmp(SIGNAL_CATCHER_THREAD_NAME, threadName , sizeof(SIGNAL_CATCHER_THREAD_NAME)-1) != 0) {
            continue;
        }

        if (firstSignalCatcherTid == -1) {
            firstSignalCatcherTid = tid;
        }

        sigblk = 0;
        char taskPath[128];
        snprintf(taskPath, sizeof(taskPath), "/proc/%d/status", tid);

        ScopedFileDescriptor fd(open(taskPath, O_RDONLY, 0));
        LineReader lr(fd.get());
        const char *line;
        size_t len;
        while (lr.getNextLine(&line, &len)) {
            if (1 == sscanf(line, "SigBlk: %" SCNx64, &sigblk)) {
                break;
            }
            lr.popLine(len);
        }
        if (SIGNAL_CATCHER_THREAD_SIGBLK != sigblk) {
            continue;
        }
        signalCatcherTid = tid;
        break;
    }
    closedir(taskDir);

    if (signalCatcherTid == -1) {
        signalCatcherTid = firstSignalCatcherTid;
    }
    return signalCatcherTid;
}
```

# 总结

对比 xCrash 的 ANR 方案，Matrix 做了以下优化：

1. 提前判断是否发生 ANR，并增加【主线程是否阻塞】这一判断条件，提高 ANR 捕获率
2. hook 系统调用拿到 `Signal Catcher` 输出的 ANR 日志，避免了重复 dump anr，属于性能优化

# 参考

* [微信Android客户端的ANR监控方案](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649288031&idx=1&sn=91c94e16460a4685a9c0c8e1b9c362a6)