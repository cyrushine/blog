---
title: 深入 Binder 之 servicemanager 进程
date: 2022-06-08 12:00:00 +0800
---

`servicemanager` 进程是由 init 进程通过 [servicemanager.rc](https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/cmds/servicemanager/servicemanager.rc) 配置文件启动的，其所在的可执行文件在 `system/bin/servicemanager`，对应的源文件是 [/platform/frameworks/native/cmds/servicemanager/main.cpp](https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/cmds/servicemanager/main.cpp)

```rc
# /platform/frameworks/native/cmds/servicemanager/servicemanager.rc
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart apexd
    onrestart restart audioserver
    onrestart restart gatekeeperd
    onrestart class_restart main
    onrestart class_restart hal
    onrestart class_restart early_hal
    writepid /dev/cpuset/system-background/tasks
    shutdown critical
```

servicemanager 是 Binder IPC 过程中的守护进程，是一个具体的服务，其功能主要是 `查询` 和 `注册服务`

```cpp
// platform/frameworks/native/cmds/servicemanager/main.cpp
int main(int argc, char** argv) {
    if (argc > 2) {
        LOG(FATAL) << "usage: " << argv[0] << " [binder driver]";
    }
    const char* driver = argc == 2 ? argv[1] : "/dev/binder";    // binder IPC 的默认地址是 /dev/binder

    sp<ProcessState> ps = ProcessState::initWithDriver(driver);
    ps->setThreadPoolMaxThreadCount(0);
    ps->setCallRestriction(ProcessState::CallRestriction::FATAL_IF_NOT_ONEWAY);
    sp<ServiceManager> manager = sp<ServiceManager>::make(std::make_unique<Access>());
    if (!manager->addService("manager", manager, false /*allowIsolated*/, IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT).isOk()) {
        LOG(ERROR) << "Could not self register servicemanager";
    }
    IPCThreadState::self()->setTheContextObject(manager);
    ps->becomeContextManager();
    sp<Looper> looper = Looper::prepare(false /*allowNonCallbacks*/);
    BinderCallback::setupTo(looper);
    ClientCallbackCallback::setupTo(looper, manager);
    while(true) {
        looper->pollAll(-1);
    }
    // should not be reached
    return EXIT_FAILURE;
}

// platform/frameworks/native/libs/binder/ProcessState.cpp
sp<ProcessState> ProcessState::initWithDriver(const char* driver) { return init(driver, true /*requireDefault*/); }
sp<ProcessState> ProcessState::init(const char *driver, bool requireDefault) {...}
ProcessState::ProcessState(const char *driver)          // driver 参数是指 binder 的路径：/dev/binder
    : mDriverName(String8(driver))                      // 记录 binder 路径
    , mDriverFD(open_driver(driver))                    // 打开 binder 并记录下它的 fd
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mWaitingForThreads(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
    , mCallRestriction(CallRestriction::NONE)
{
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }
#ifdef __ANDROID__
    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver '%s' could not be opened.  Terminating.", driver);
#endif
}

static int open_driver(const char *driver)
{
    int fd = open(driver, O_RDWR | O_CLOEXEC);               // binder 用一段路径（/dev/binder）表示，可以被 open 打开
    if (fd >= 0) {
        int vers = 0;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);  // 使用 ioctl 校验 binder 版本
        if (result == -1) {
            ALOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
            close(fd);
            fd = -1;
        }
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
          ALOGE("Binder driver protocol(%d) does not match user space protocol(%d)! ioctl() return value: %d",
                vers, BINDER_CURRENT_PROTOCOL_VERSION, result);
            close(fd);
            fd = -1;
        }
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);  // 使用 ioctl 设置 binder 的线程池数量
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
        uint32_t enable = DEFAULT_ENABLE_ONEWAY_SPAM_DETECTION;
        result = ioctl(fd, BINDER_ENABLE_ONEWAY_SPAM_DETECTION, &enable);
        if (result == -1) {
            ALOGD("Binder ioctl to enable oneway spam detection failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '%s' failed: %s\n", driver, strerror(errno));
    }
    return fd;
}
```