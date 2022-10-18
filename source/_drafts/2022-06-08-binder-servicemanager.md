---
title: 深入 Binder 之 servicemanager 进程
date: 2022-06-08 12:00:00 +0800
---

# 主例程

`servicemanager` 进程是由 init 进程通过 [servicemanager.rc](https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/cmds/servicemanager/servicemanager.rc) 配置文件启动的，其所在的可执行文件在 `system/bin/servicemanager`，对应的源文件是 [/platform/frameworks/native/cmds/servicemanager/main.cpp](https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/cmds/servicemanager/main.cpp)

```rc
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
// https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/cmds/servicemanager/main.cpp
int main(int argc, char** argv) {
    if (argc > 2) {
        LOG(FATAL) << "usage: " << argv[0] << " [binder driver]";
    }
    const char* driver = argc == 2 ? argv[1] : "/dev/binder";    // binder IPC 的默认地址是 /dev/binder

    // 初始化 binder 连接
    sp<ProcessState> ps = ProcessState::initWithDriver(driver);
    ps->setThreadPoolMaxThreadCount(0);    // ioctl(mDriverFD, BINDER_SET_MAX_THREADS, &maxThreads)
    ps->setCallRestriction(ProcessState::CallRestriction::FATAL_IF_NOT_ONEWAY);

    // 注册成为 binder 的服务管理器（ContextManager），id == 0，handle == 0
    sp<ServiceManager> manager = sp<ServiceManager>::make(std::make_unique<Access>());
    if (!manager->addService("manager", manager, false /*allowIsolated*/, IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT).isOk()) {
        LOG(ERROR) << "Could not self register servicemanager";
    }
    // IPCThreadState 实例是线程本地变量（thread local），通过 IPCThreadState::self() 获得
    // cpp 通过 pthread_getspecific(key) 和 pthread_setspecific(key, value) 设置/获取线程本地变量
    IPCThreadState::self()->setTheContextObject(manager);
    ps->becomeContextManager();    // BINDER_SET_CONTEXT_MGR_EXT

    // 开始主循环：提供注册&查询服务
    sp<Looper> looper = Looper::prepare(false /*allowNonCallbacks*/);
    BinderCallback::setupTo(looper);
    ClientCallbackCallback::setupTo(looper, manager);
    while(true) {
        looper->pollAll(-1);
    }
    // should not be reached
    return EXIT_FAILURE;
}
```

# 初始化 binder 连接

1. 通过系统调用 `open` 打开 `/dev/binder` 得到 binder 驱动的文件描述符 fd
2. 通过 `ioctl` 与 binder 驱动交互进行一系列的初始化操作
    1. BINDER_VERSION 版本校验
    2. BINDER_SET_MAX_THREADS 设置 binder 驱动线程池的容量
    3. BINDER_ENABLE_ONEWAY_SPAM_DETECTION
    4. ...
3. 通过 `mmap` 在 servicemanager 进程分配一块虚存用以后续使用（对应 binder 驱动的 `binder_mmap`）

```cpp
// https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/libs/binder/ProcessState.cpp
sp<ProcessState> ProcessState::initWithDriver(const char* driver) { return init(driver, true /*requireDefault*/); }
sp<ProcessState> ProcessState::init(const char *driver, bool requireDefault) {...}
ProcessState::ProcessState(const char *driver)          // driver 参数是指 binder 的路径：/dev/binder
    : mDriverName(String8(driver))                      // 记录 binder 路径
    , mDriverFD(open_driver(driver))                    // 打开 binder 并记录下它的 fd
    , mVMStart(MAP_FAILED)                              // mmap 分配的一块内存空间，后续会用到
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
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }
}

// 打开 binder driver 得到 fd，同时会通过 ioctl 设置一些初始参数
static int open_driver(const char *driver /* /dev/binder */)
{
    int fd = open(driver, O_RDWR | O_CLOEXEC);               // 打开 binder driver 得到其 fd，后续通过 ioctl 与 binder driver 通讯
    if (fd >= 0) {
        int vers = 0;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);  // 校验 binder driver 版本
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
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;      // 使用 ioctl 设置 binder 的线程池数量
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);  
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

# 注册为服务管理器

1. 进程内的 `ServiceManager` 注册为 `manager`
2. 通过 `ioctl(BINDER_SET_CONTEXT_MGR_EXT)` 在 binder driver 将自己注册为服务管理器 `binder_context_mgr_node`，handle == 0

```cpp
// ServiceManager 内部维护了一个 map：mNameToService，string -> IBinder
// 这里将自己添加进去：manager -> ServiceManager
// https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/cmds/servicemanager/ServiceManager.cpp
Status ServiceManager::addService(
    const std::string& name   /* manager */, 
    const sp<IBinder>& binder /* this */, 
    bool allowIsolated        /* false */, 
    int32_t dumpPriority      /* IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT */
) {
    auto ctx = mAccess->getCallingContext();
    // apps cannot add services
    if (multiuser_get_app_id(ctx.uid) >= AID_APP) {
        return Status::fromExceptionCode(Status::EX_SECURITY);
    }
    if (!mAccess->canAdd(ctx, name)) {
        return Status::fromExceptionCode(Status::EX_SECURITY);
    }
    if (binder == nullptr) {
        return Status::fromExceptionCode(Status::EX_ILLEGAL_ARGUMENT);
    }
    if (!isValidServiceName(name)) {
        LOG(ERROR) << "Invalid service name: " << name;
        return Status::fromExceptionCode(Status::EX_ILLEGAL_ARGUMENT);
    }

    // implicitly unlinked when the binder is removed
    if (binder->remoteBinder() != nullptr &&
        binder->linkToDeath(sp<ServiceManager>::fromExisting(this)) != OK) {
        LOG(ERROR) << "Could not linkToDeath when adding " << name;
        return Status::fromExceptionCode(Status::EX_ILLEGAL_STATE);
    }
    // Overwrite the old service if it exists
    mNameToService[name] = Service {
        .binder = binder,
        .allowIsolated = allowIsolated,
        .dumpPriority = dumpPriority,
        .debugPid = ctx.debugPid,
    };
    auto it = mNameToRegistrationCallback.find(name);
    if (it != mNameToRegistrationCallback.end()) {
        for (const sp<IServiceCallback>& cb : it->second) {
            mNameToService[name].guaranteeClient = true;
            // permission checked in registerForNotifications
            cb->onRegistration(name, binder);
        }
    }
    return Status::ok();
}

// 发送消息 BINDER_SET_CONTEXT_MGR_EXT 给 binder，使自己成为服务管理器
// https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/libs/binder/ProcessState.cpp
bool ProcessState::becomeContextManager()
{
    AutoMutex _l(mLock);

    flat_binder_object obj {
        .flags = FLAT_BINDER_FLAG_TXN_SECURITY_CTX,
    };

    int result = ioctl(mDriverFD, BINDER_SET_CONTEXT_MGR_EXT, &obj);

    // fallback to original method
    if (result != 0) {
        android_errorWriteLog(0x534e4554, "121035042");

        int unused = 0;
        result = ioctl(mDriverFD, BINDER_SET_CONTEXT_MGR, &unused);
    }

    if (result == -1) {
        ALOGE("Binder ioctl to become context manager failed: %s\n", strerror(errno));
    }

    return result == 0;
}

// https://android.googlesource.com/kernel/common/+/refs/tags/android-13.0.0_r0.20/drivers/android/binder.c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	switch (cmd) {
	// ...
	case BINDER_SET_CONTEXT_MGR_EXT: {
		struct flat_binder_object fbo;
		if (copy_from_user(&fbo, ubuf, sizeof(fbo))) {
			ret = -EINVAL;
			goto err;
		}
		ret = binder_ioctl_set_ctx_mgr(filp, &fbo);
		if (ret)
			goto err;
		break;
	}
    // ...
	}
}

static int binder_ioctl_set_ctx_mgr(struct file *filp,
				    struct flat_binder_object *fbo)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	struct binder_context *context = proc->context;
	struct binder_node *new_node;
	kuid_t curr_euid = current_euid();
	mutex_lock(&context->context_mgr_node_lock);
	if (context->binder_context_mgr_node) {
		pr_err("BINDER_SET_CONTEXT_MGR already set\n");
		ret = -EBUSY;
		goto out;
	}
	ret = security_binder_set_context_mgr(proc->cred);
	if (ret < 0)
		goto out;
	if (uid_valid(context->binder_context_mgr_uid)) {
		if (!uid_eq(context->binder_context_mgr_uid, curr_euid)) {
			pr_err("BINDER_SET_CONTEXT_MGR bad uid %d != %d\n",
			       from_kuid(&init_user_ns, curr_euid),
			       from_kuid(&init_user_ns,
					 context->binder_context_mgr_uid));
			ret = -EPERM;
			goto out;
		}
	} else {
		context->binder_context_mgr_uid = curr_euid;
	}
	new_node = binder_new_node(proc, fbo);
	if (!new_node) {
		ret = -ENOMEM;
		goto out;
	}
	binder_node_lock(new_node);
	new_node->local_weak_refs++;
	new_node->local_strong_refs++;
	new_node->has_strong_ref = 1;
	new_node->has_weak_ref = 1;
	context->binder_context_mgr_node = new_node;
	binder_node_unlock(new_node);
	binder_put_node(new_node);
out:
	mutex_unlock(&context->context_mgr_node_lock);
	return ret;
}

```

# 查询

## 暴露自己

看看其他进程（client）是如何使用 servicemanager 提供的查询服务的，随便找个 biner ipc 比如 `ActivityManager.getRunningAppProcesses()` 开始深入下去：

1. 内部是调用了 `IActivityManager.getRunningAppProcesses()`，`IActivityManager` 明显是个 binder ipc，它的 binder client proxy 是从 `ServiceManager.getService` 获取的

2. `ServiceManager` 内部调用了 `IServiceManager.getService`，看起来 `IServiceManager` 又是个 binder ipc，它的 proxy 是从 `BinderInternal.getContextObject()` 获得的

3. 往下看发现 `IServiceManager` 的 proxy 实际上是个 handle == 0 的 `BpBinder`（binder client 包括 java 层的 `BinderProxy` 和 native 层的 `BpBinder`）

4. binder ipc 过程中，client 通过 `ioctl(binder_fd, BC_TRANSACTION, binder_transaction_data)` 将 request 发送给 binder driver，再由 binder driver 根据 `binder_transaction_data.target.handle` 字段找到对应的 server 将 request 转发给它，具体流程是 `binder_ioctl -> binder_ioctl_write_read -> binder_thread_write -> binder_transaction`

5. 在 `binder_transaction` 可以看到如果发现 handle == 0 则取 `binder_context_mgr_node` 作为 target server，而在章节 [注册为服务管理器](#注册为服务管理器) 有说到 servicemanager 进程会通过 `BINDER_SET_CONTEXT_MGR_EXT/binder_ioctl_set_ctx_mgr` 把自己注册至 `binder_context_mgr_node`

整理下逻辑：app 进行 binder ipc 时需要得到 target server handle，可以通过 `ServiceManager.getService` 查询得到，这个 ipc 方法最终会调用 servicemanager 进程提供的查询功能；而 servicemanager 早已在 binder driver 里把自己注册为服务管理器（`binder_context_mgr_node`，handle == 0），这样 app 进程就无需再进行查找即可调用 `ServiceManager`


```java
class ActivityManager {
    public List<RunningAppProcessInfo> getRunningAppProcesses() {
        try {
            return getService().getRunningAppProcesses();
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    // IActivityManager 看起来像是 aidl 生成的 java interface
    // 一搜果然没有 IActivityManager.java 只有 frameworks/base/core/java/android/app/IActivityManager.aidl
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);  // 获得 client proxy，真正与 server 通讯的是 b
                                                                                       // 参考【深入 Binder 之 AIDL】
                    return am;
                }
            };        
}

// 看看如何获取 IActivityManager client binder
class ServiceManager {
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return Binder.allowBlocking(rawGetService(name));
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }

    private static IBinder rawGetService(String name) throws RemoteException {
        final long start = sStatLogger.getTime();
        final IBinder binder = getIServiceManager().getService(name);  // IServiceManager? 看起来又是一个 aidl，又是一个 binder ipc
        // ...                                                         // 即是 IActivityManager client binder 是从 IServiceManager server 获得的
        return binder;
    }

// 有了上面的经验就可以很快地知道 IServiceManager.Stub.asInterface(remote) 返回的是 IServiceManager client proxy
// 真正的通讯是由 BinderInternal.getContextObject() 得到的 client binder 实现的
    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }            
}

public static IServiceManager ServiceManagerNative.asInterface(IBinder obj) {
    return new ServiceManagerProxy(obj);
}

class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
        mServiceManager = IServiceManager.Stub.asInterface(remote);
    }
}

public static final native IBinder BinderInternal.getContextObject();
```

```cpp
// frameworks/base/core/jni/android_util_Binder.cpp
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}

// frameworks/native/libs/binder/ProcessState.cpp
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    sp<IBinder> context = getStrongProxyForHandle(0);
    return context;
}
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)  // handle == 0
{
    sp<IBinder> result;
    AutoMutex _l(mLock);
    if (handle == 0 && the_context_object != nullptr) return the_context_object;
    handle_entry* e = lookupHandleLocked(handle);  // 有个成员属性 Vector<handle_entry> mHandleToObject 用以缓存 IBinder
    if (e != nullptr) {                            // 当 handle 所在位置为空时会创建一个新的 entry 实例返回
        IBinder* b = e->binder;                    // 此时 b == null
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            //...
            sp<BpBinder> b = BpBinder::PrivateAccessor::create(handle);  // 这里的 IBinder 是 BpBinder，而且是 handle == 0 的 BpBinder
            e->binder = b.get();
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            // This little bit of nastyness is to allow us to add a primary
            // reference to the remote proxy when this team doesn't have one
            // but another team is sending the handle to us.
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}

// 从【深入 Binder 之架构篇】知道 BpBinder 是 native client，负责：
// 从 servicemanager 查找 service handle，然后将 request 发送给 binder driver，由 binder driver 转发 request 给目标 server
// 但 servicemanager handler 又得从哪里获取呢？答案很简单，它的 handle == 0，无需通过查找获得，只要 handle == 0 就表示 server 是 service manager
// 看看在 binder driver 里是如何处理 handle == 0 的情况得

// https://cs.android.com/android/kernel/superproject/+/common-android-mainline:common/drivers/android/binder.c
// common/drivers/android/binder.c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	// ...
	switch (cmd) {
	case BINDER_WRITE_READ:
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
	case BINDER_SET_MAX_THREADS:
	case BINDER_SET_CONTEXT_MGR_EXT: {
    // ...
    }
}

static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;

	if (size != sizeof(struct binder_write_read)) {
		ret = -EINVAL;
		goto out;
	}
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
	if (bwr.write_size > 0) {
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		trace_binder_write_done(ret);
		if (ret < 0) {
			bwr.read_consumed = 0;
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	if (bwr.read_size > 0)
        // ...
}

static int binder_thread_write(struct binder_proc *proc,
			struct binder_thread *thread,
			binder_uintptr_t binder_buffer, size_t size,
			binder_size_t *consumed)
{
	uint32_t cmd;
	struct binder_context *context = proc->context;
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	while (ptr < end && thread->return_error.cmd == BR_OK) {
		int ret;
		if (get_user(cmd, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);

		switch (cmd) {
		case BC_TRANSACTION:
		case BC_REPLY: {
			struct binder_transaction_data tr;
			if (copy_from_user(&tr, ptr, sizeof(tr)))
				return -EFAULT;
			ptr += sizeof(tr);
			binder_transaction(proc, thread, &tr, cmd == BC_REPLY, 0);
			break;
		}
        // ...
		}
		*consumed = ptr - buffer;
	}
	return 0;
}

static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply,
			       binder_size_t extra_buffers_size)
{
    // ...
	if (reply) {
		// server response -> client ...
	} else {
        // client request -> server
		if (tr->target.handle) {  // handle > 0 的情况下，根据 handle 查找出 server/target
			struct binder_ref *ref;
			binder_proc_lock(proc);
			ref = binder_get_ref_olocked(proc, tr->target.handle,
						     true);
			if (ref) {
				target_node = binder_get_node_refs_for_txn(
						ref->node, &target_proc,
						&return_error);
			} else {
				binder_user_error("%d:%d got transaction to invalid handle, %u\n",
						  proc->pid, thread->pid, tr->target.handle);
				return_error = BR_FAILED_REPLY;
			}
			binder_proc_unlock(proc);
		} else {

            // handle == 0 说明 server/target 是 service manager，它保存在 binder_context_mgr_node
			mutex_lock(&context->context_mgr_node_lock);
			target_node = context->binder_context_mgr_node;
			if (target_node)
				target_node = binder_get_node_refs_for_txn(
						target_node, &target_proc,
						&return_error);
			else
				return_error = BR_DEAD_REPLY;
			mutex_unlock(&context->context_mgr_node_lock);
			if (target_node && target_proc->pid == proc->pid) {
				binder_user_error("%d:%d got transaction to context manager from process owning it\n",
						  proc->pid, thread->pid);
				return_error = BR_FAILED_REPLY;
				return_error_param = -EINVAL;
				return_error_line = __LINE__;
				goto err_invalid_target_handle;
			}
		}
        // ...
    }
}
```

## 响应 binder 消息

```cpp
// https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/cmds/servicemanager/main.cpp
class BinderCallback : public LooperCallback {
public:
    static sp<BinderCallback> setupTo(const sp<Looper>& looper) {    // 监听 binder driver fd，当有数据时回调 handlePolledCommands
        sp<BinderCallback> cb = sp<BinderCallback>::make();
        int binder_fd = -1;
        IPCThreadState::self()->setupPolling(&binder_fd);
        LOG_ALWAYS_FATAL_IF(binder_fd < 0, "Failed to setupPolling: %d", binder_fd);
        int ret = looper->addFd(binder_fd,
                                Looper::POLL_CALLBACK,
                                Looper::EVENT_INPUT,
                                cb,
                                nullptr /*data*/);
        LOG_ALWAYS_FATAL_IF(ret != 1, "Failed to add binder FD to Looper");
        return cb;
    }

    int handleEvent(int /* fd */, int /* events */, void* /* data */) override {
        IPCThreadState::self()->handlePolledCommands();
        return 1;  // Continue receiving callbacks.
    }
};

// https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::setupPolling(int* fd)
{
    if (mProcess->mDriverFD < 0) {
        return -EBADF;
    }
    mOut.writeInt32(BC_ENTER_LOOPER);// mOut 是发送给 binder driver 的内存区域
                                     // 在【深入 Binder 之架构篇】介绍过 BC_ENTER_LOOPER：告知 binder driver 应用线程进入 Looper
    flushCommands();                 // talkWithDriver(false)，通过 ioctl 与 binder 通讯，mOut 是请求数据，mIn 保存响应数据，详见【深入 Binder 之客户端】
    *fd = mProcess->mDriverFD;       // 返回 binder fd
    return 0;
}

status_t IPCThreadState::handlePolledCommands()
{
    status_t result;
    do {
        result = getAndExecuteCommand();
    } while (mIn.dataPosition() < mIn.dataSize());
    processPendingDerefs();
    flushCommands();
    return result;
}

status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;
    result = talkWithDriver();
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        cmd = mIn.readInt32();
        IF_LOG_COMMANDS() {
            alog << "Processing top-level Command: "
                 << getReturnString(cmd) << endl;
        }
        pthread_mutex_lock(&mProcess->mThreadCountLock);
        mProcess->mExecutingThreadsCount++;
        if (mProcess->mExecutingThreadsCount >= mProcess->mMaxThreads &&
                mProcess->mStarvationStartTimeMs == 0) {
            mProcess->mStarvationStartTimeMs = uptimeMillis();
        }
        pthread_mutex_unlock(&mProcess->mThreadCountLock);
        result = executeCommand(cmd);
        pthread_mutex_lock(&mProcess->mThreadCountLock);
        mProcess->mExecutingThreadsCount--;
        if (mProcess->mExecutingThreadsCount < mProcess->mMaxThreads &&
                mProcess->mStarvationStartTimeMs != 0) {
            int64_t starvationTimeMs = uptimeMillis() - mProcess->mStarvationStartTimeMs;
            if (starvationTimeMs > 100) {
                ALOGE("binder thread pool (%zu threads) starved for %" PRId64 " ms",
                      mProcess->mMaxThreads, starvationTimeMs);
            }
            mProcess->mStarvationStartTimeMs = 0;
        }
        // Cond broadcast can be expensive, so don't send it every time a binder
        // call is processed. b/168806193
        if (mProcess->mWaitingForThreads > 0) {
            pthread_cond_broadcast(&mProcess->mThreadCountDecrement);
        }
        pthread_mutex_unlock(&mProcess->mThreadCountLock);
    }
    return result;
}

status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;
    switch ((uint32_t)cmd) {
    case BR_ERROR:
        result = mIn.readInt32();
        break;
    case BR_OK:
        break;
    case BR_ACQUIRE:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();
        ALOG_ASSERT(refs->refBase() == obj,
                   "BR_ACQUIRE: object %p does not match cookie %p (expected %p)",
                   refs, obj, refs->refBase());
        obj->incStrong(mProcess.get());
        IF_LOG_REMOTEREFS() {
            LOG_REMOTEREFS("BR_ACQUIRE from driver on %p", obj);
            obj->printRefs();
        }
        mOut.writeInt32(BC_ACQUIRE_DONE);
        mOut.writePointer((uintptr_t)refs);
        mOut.writePointer((uintptr_t)obj);
        break;
    case BR_RELEASE:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();
        ALOG_ASSERT(refs->refBase() == obj,
                   "BR_RELEASE: object %p does not match cookie %p (expected %p)",
                   refs, obj, refs->refBase());
        IF_LOG_REMOTEREFS() {
            LOG_REMOTEREFS("BR_RELEASE from driver on %p", obj);
            obj->printRefs();
        }
        mPendingStrongDerefs.push(obj);
        break;
    case BR_INCREFS:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();
        refs->incWeak(mProcess.get());
        mOut.writeInt32(BC_INCREFS_DONE);
        mOut.writePointer((uintptr_t)refs);
        mOut.writePointer((uintptr_t)obj);
        break;
    case BR_DECREFS:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();
        // NOTE: This assertion is not valid, because the object may no
        // longer exist (thus the (BBinder*)cast above resulting in a different
        // memory address).
        //ALOG_ASSERT(refs->refBase() == obj,
        //           "BR_DECREFS: object %p does not match cookie %p (expected %p)",
        //           refs, obj, refs->refBase());
        mPendingWeakDerefs.push(refs);
        break;
    case BR_ATTEMPT_ACQUIRE:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();
        {
            const bool success = refs->attemptIncStrong(mProcess.get());
            ALOG_ASSERT(success && refs->refBase() == obj,
                       "BR_ATTEMPT_ACQUIRE: object %p does not match cookie %p (expected %p)",
                       refs, obj, refs->refBase());
            mOut.writeInt32(BC_ACQUIRE_RESULT);
            mOut.writeInt32((int32_t)success);
        }
        break;
    case BR_TRANSACTION_SEC_CTX:
    case BR_TRANSACTION:
        {
            binder_transaction_data_secctx tr_secctx;
            binder_transaction_data& tr = tr_secctx.transaction_data;
            if (cmd == (int) BR_TRANSACTION_SEC_CTX) {
                result = mIn.read(&tr_secctx, sizeof(tr_secctx));
            } else {
                result = mIn.read(&tr, sizeof(tr));
                tr_secctx.secctx = 0;
            }
            ALOG_ASSERT(result == NO_ERROR,
                "Not enough command data for brTRANSACTION");
            if (result != NO_ERROR) break;
            Parcel buffer;
            buffer.ipcSetDataReference(
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(binder_size_t), freeBuffer);
            const void* origServingStackPointer = mServingStackPointer;
            mServingStackPointer = &origServingStackPointer; // anything on the stack
            const pid_t origPid = mCallingPid;
            const char* origSid = mCallingSid;
            const uid_t origUid = mCallingUid;
            const int32_t origStrictModePolicy = mStrictModePolicy;
            const int32_t origTransactionBinderFlags = mLastTransactionBinderFlags;
            const int32_t origWorkSource = mWorkSource;
            const bool origPropagateWorkSet = mPropagateWorkSource;
            // Calling work source will be set by Parcel#enforceInterface. Parcel#enforceInterface
            // is only guaranteed to be called for AIDL-generated stubs so we reset the work source
            // here to never propagate it.
            clearCallingWorkSource();
            clearPropagateWorkSource();
            mCallingPid = tr.sender_pid;
            mCallingSid = reinterpret_cast<const char*>(tr_secctx.secctx);
            mCallingUid = tr.sender_euid;
            mLastTransactionBinderFlags = tr.flags;
            // ALOGI(">>>> TRANSACT from pid %d sid %s uid %d\n", mCallingPid,
            //    (mCallingSid ? mCallingSid : "<N/A>"), mCallingUid);
            Parcel reply;
            status_t error;
            IF_LOG_TRANSACTIONS() {
                TextOutput::Bundle _b(alog);
                alog << "BR_TRANSACTION thr " << (void*)pthread_self()
                    << " / obj " << tr.target.ptr << " / code "
                    << TypeCode(tr.code) << ": " << indent << buffer
                    << dedent << endl
                    << "Data addr = "
                    << reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer)
                    << ", offsets addr="
                    << reinterpret_cast<const size_t*>(tr.data.ptr.offsets) << endl;
            }
            if (tr.target.ptr) {
                // We only have a weak reference on the target object, so we must first try to
                // safely acquire a strong reference before doing anything else with it.
                if (reinterpret_cast<RefBase::weakref_type*>(
                        tr.target.ptr)->attemptIncStrong(this)) {
                    error = reinterpret_cast<BBinder*>(tr.cookie)->transact(tr.code, buffer,
                            &reply, tr.flags);
                    reinterpret_cast<BBinder*>(tr.cookie)->decStrong(this);
                } else {
                    error = UNKNOWN_TRANSACTION;
                }
            } else {
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
            }
            //ALOGI("<<<< TRANSACT from pid %d restore pid %d sid %s uid %d\n",
            //     mCallingPid, origPid, (origSid ? origSid : "<N/A>"), origUid);
            if ((tr.flags & TF_ONE_WAY) == 0) {
                LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                if (error < NO_ERROR) reply.setError(error);
                constexpr uint32_t kForwardReplyFlags = TF_CLEAR_BUF;
                sendReply(reply, (tr.flags & kForwardReplyFlags));
            } else {
                if (error != OK) {
                    alog << "oneway function results for code " << tr.code
                         << " on binder at "
                         << reinterpret_cast<void*>(tr.target.ptr)
                         << " will be dropped but finished with status "
                         << statusToString(error);
                    // ideally we could log this even when error == OK, but it
                    // causes too much logspam because some manually-written
                    // interfaces have clients that call methods which always
                    // write results, sometimes as oneway methods.
                    if (reply.dataSize() != 0) {
                         alog << " and reply parcel size " << reply.dataSize();
                    }
                    alog << endl;
                }
                LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
            }
            mServingStackPointer = origServingStackPointer;
            mCallingPid = origPid;
            mCallingSid = origSid;
            mCallingUid = origUid;
            mStrictModePolicy = origStrictModePolicy;
            mLastTransactionBinderFlags = origTransactionBinderFlags;
            mWorkSource = origWorkSource;
            mPropagateWorkSource = origPropagateWorkSet;
            IF_LOG_TRANSACTIONS() {
                TextOutput::Bundle _b(alog);
                alog << "BC_REPLY thr " << (void*)pthread_self() << " / obj "
                    << tr.target.ptr << ": " << indent << reply << dedent << endl;
            }
        }
        break;
    case BR_DEAD_BINDER:
        {
            BpBinder *proxy = (BpBinder*)mIn.readPointer();
            proxy->sendObituary();
            mOut.writeInt32(BC_DEAD_BINDER_DONE);
            mOut.writePointer((uintptr_t)proxy);
        } break;
    case BR_CLEAR_DEATH_NOTIFICATION_DONE:
        {
            BpBinder *proxy = (BpBinder*)mIn.readPointer();
            proxy->getWeakRefs()->decWeak(proxy);
        } break;
    case BR_FINISHED:
        result = TIMED_OUT;
        break;
    case BR_NOOP:
        break;
    case BR_SPAWN_LOOPER:
        mProcess->spawnPooledThread(false);
        break;
    default:
        ALOGE("*** BAD COMMAND %d received from Binder driver\n", cmd);
        result = UNKNOWN_ERROR;
        break;
    }
    if (result != NO_ERROR) {
        mLastError = result;
    }
    return result;
}

// LooperCallback for IClientCallback
// https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/cmds/servicemanager/main.cpp
class ClientCallbackCallback : public LooperCallback {
public:
    static sp<ClientCallbackCallback> setupTo(const sp<Looper>& looper, const sp<ServiceManager>& manager) {
        sp<ClientCallbackCallback> cb = sp<ClientCallbackCallback>::make(manager);
        int fdTimer = timerfd_create(CLOCK_MONOTONIC, 0 /*flags*/);
        LOG_ALWAYS_FATAL_IF(fdTimer < 0, "Failed to timerfd_create: fd: %d err: %d", fdTimer, errno);
        itimerspec timespec {
            .it_interval = {
                .tv_sec = 5,
                .tv_nsec = 0,
            },
            .it_value = {
                .tv_sec = 5,
                .tv_nsec = 0,
            },
        };
        int timeRes = timerfd_settime(fdTimer, 0 /*flags*/, &timespec, nullptr);
        LOG_ALWAYS_FATAL_IF(timeRes < 0, "Failed to timerfd_settime: res: %d err: %d", timeRes, errno);
        int addRes = looper->addFd(fdTimer,
                                   Looper::POLL_CALLBACK,
                                   Looper::EVENT_INPUT,
                                   cb,
                                   nullptr);
        LOG_ALWAYS_FATAL_IF(addRes != 1, "Failed to add client callback FD to Looper");
        return cb;
    }
    int handleEvent(int fd, int /*events*/, void* /*data*/) override {
        uint64_t expirations;
        int ret = read(fd, &expirations, sizeof(expirations));
        if (ret != sizeof(expirations)) {
            ALOGE("Read failed to callback FD: ret: %d err: %d", ret, errno);
        }
        mManager->handleClientCallbacks();
        return 1;  // Continue receiving callbacks.
    }
private:
    friend sp<ClientCallbackCallback>;
    ClientCallbackCallback(const sp<ServiceManager>& manager) : mManager(manager) {}
    sp<ServiceManager> mManager;
};
```

## 实现查询

