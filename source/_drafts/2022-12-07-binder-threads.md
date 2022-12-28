---
title: Binder IPC 线程调度模型
date: 2022-12-07 12:00:00 +0800
---

![1.png](/image/2022-12-07-binder-threads/1.png)

# 主线程与普通线程

Binder 线程可以分为两类：`主线程` 和 `普通线程`，它们的区别是：

1. 主线程名称为 `Binder_1` or `Binder:<pid>_1`，普通线程则从 2 开始（`ProcessState::makeBinderThreadName`）

2. 主线程在应用进程起来后立刻启动，且不会退出；普通线程由 [BR_SPAWN_LOOPER](#创建普通线程) 启动且会被 binder driver 命令 `TIMED_OUT` 退出（`IPCThreadState::joinThreadPool`）

3. binder 线程上限（`BINDER_SET_MAX_THREADS`）只影响普通线程，更确切地说是只影响通过 `BR_SPAWN_LOOPER - BC_REGISTER_LOOPER` 创建的 binder 线程

Binder 主线程是在其所在进程的初始化流程里启动的，Java 层进程的创建都是通过 [Process.start()](/2021/03/02/how-application-being-created-and-init/) 方法，向 Zygote 进程发出创建进程的 socket 消息，Zygote 进程收到消息后会调用 `Zygote.forkAndSpecialize()` 来 fork 出新进程，在新进程中会调用到 `RuntimeInit.nativeZygoteInit()` 方法，该方法经过 jni 映射最终会调用到 `app_main.cpp` 中的 `onZygoteInit`，那么接下来从这个方法说起

```cpp
/**
 * The main function called when started through the zygote process. This could be unified with
 * main(), if the native code in nativeFinishInit() were rationalized with Zygote startup.
 */
public static Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    // ...
    ZygoteInit.nativeZygoteInit();
    return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
            classLoader);
}

// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/AndroidRuntime.cpp
static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();  // AppRuntime->onZygoteInit()
}

// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/cmds/app_process/app_main.cpp
virtual void onZygoteInit()
{
    // 上面介绍过，ProcessState 是单例模式，初始化实例时会：
    // 1，打开 binder driver：open("/dev/binder") && mmap
    // 2，版本查询和校验：BINDER_VERSION
    // 3，设置线程数：BINDER_SET_MAX_THREADS
    // ...
    sp<ProcessState> proc = ProcessState::self();
    ALOGV("App process: starting thread pool.\n");
    proc->startThreadPool();
}

// 开启 binder 主线程执行 binder 消息通讯的 loop：IPCThreadState::self()->joinThreadPool = talkWithDriver + executeCommand
// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}

// spawn - 产卵
// spawnPooledThread 是创建 binder 线程的核心函数，它创建新线程执行 IPCThreadState::joinThreadPool
// 新的 binder 线程的生命流程：
// 1，往 binder 发送 BC_ENTER_LOOPER（主线程） or BC_REGISTER_LOOPER（普通线程）
// 2. 主循环：getAndExecuteCommand()
// 3. 退出循环时发送 BC_EXIT_LOOPER 给 binder driver
void ProcessState::spawnPooledThread(bool isMain /* true 表示主线程 */)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();  // Binder_x or Binder:<pid>_x
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = sp<PoolThread>::make(isMain);
        t->run(name.string());
        pthread_mutex_lock(&mThreadCountLock);
        mKernelStartedThreads++;
        pthread_mutex_unlock(&mThreadCountLock);
    }
}
class PoolThread : public Thread
{
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }
};
void IPCThreadState::joinThreadPool(bool isMain)    // binder 线程的整个生命流程
{
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    status_t result;
    mIsLooper = true;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {  // 可以看到主线程与普通线程的区别之一：主线程永远不会主动退出
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER);
    mIsLooper = false;
    talkWithDriver(false);
}

// 主线程注册（BC_REGISTER_LOOPER）和普通线程注册（BC_ENTER_LOOPER）很类似
// 都是打开标志位 BINDER_LOOPER_STATE_ENTERED or BINDER_LOOPER_STATE_REGISTERED
static int binder_thread_write(struct binder_proc *proc,
			struct binder_thread *thread,
			binder_uintptr_t binder_buffer, size_t size,
			binder_size_t *consumed)
{
	case BC_REGISTER_LOOPER:
		binder_inner_proc_lock(proc);
		if (thread->looper & BINDER_LOOPER_STATE_ENTERED) {
			thread->looper |= BINDER_LOOPER_STATE_INVALID;
			binder_user_error("%d:%d ERROR: BC_REGISTER_LOOPER called after BC_ENTER_LOOPER\n",
				proc->pid, thread->pid);
		} else if (proc->requested_threads == 0) {
			thread->looper |= BINDER_LOOPER_STATE_INVALID;
			binder_user_error("%d:%d ERROR: BC_REGISTER_LOOPER called without request\n",
				proc->pid, thread->pid);
		} else {
			proc->requested_threads--;
			proc->requested_threads_started++;
		}
		thread->looper |= BINDER_LOOPER_STATE_REGISTERED;
		binder_inner_proc_unlock(proc);
		break;

	case BC_ENTER_LOOPER:
		if (thread->looper & BINDER_LOOPER_STATE_REGISTERED) {
			thread->looper |= BINDER_LOOPER_STATE_INVALID;
			binder_user_error("%d:%d ERROR: BC_ENTER_LOOPER called after BC_REGISTER_LOOPER\n",
				proc->pid, thread->pid);
		}
		thread->looper |= BINDER_LOOPER_STATE_ENTERED;
		break;	
}
```

## binder 线程命名

```cpp
// binder 线程命名规则：Binder_x（如果 >= Android N 则是 Binder:<pid>_x）
// 其中的 x 从 1 开始
// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp
String8 ProcessState::makeBinderThreadName() {
    int32_t s = android_atomic_add(1, &mThreadPoolSeq);
    pid_t pid = getpid();

    std::string_view driverName = mDriverName.c_str();
    android::base::ConsumePrefix(&driverName, "/dev/");

    String8 name;
    name.appendFormat("%.*s:%d_%X", static_cast<int>(driverName.length()), driverName.data(), pid,
                      s);
    return name;
}
ProcessState::ProcessState(const char* driver)
      : mDriverName(String8(driver)),
        mDriverFD(-1),
        mVMStart(MAP_FAILED),
        mThreadCountLock(PTHREAD_MUTEX_INITIALIZER),
        mThreadCountDecrement(PTHREAD_COND_INITIALIZER),
        mExecutingThreadsCount(0),
        mWaitingForThreads(0),
        mMaxThreads(DEFAULT_MAX_BINDER_THREADS),
        mCurrentThreads(0),
        mKernelStartedThreads(0),
        mStarvationStartTimeMs(0),
        mForked(false),
        mThreadPoolStarted(false),
        mThreadPoolSeq(1),    // binder 线程名的序号从 1 开始
        mCallRestriction(CallRestriction::NONE) {
	// ...
}
```

## 创建普通线程

流程概览：binder driver 在满足以下条件的情况下，发送 `BR_SPAWN_LOOPER` 给应用进程，应用进程开启新线程 `ProcessState::spawnPooledThread(false)` 后，发送 `BC_REGISTER_LOOPER` 告知 binder driver 线程已启动

有三种情况可以进入 `binder_thread_read` done 代码段：

1. 当 binder driver 处理类型为 `BINDER_WORK_TRANSACTION` 的任务，即 binder driver 收到的命令是 `BC_TRANSACTION`（client 发送 request 至 binder） or `BC_REPLY`（server 响应 response 至 binder）

2. 当前线程的 return_error 发生 error

3. Binder Driver 向 client 发送死亡通知 `BR_DEAD_BINDER`

按需启动普通 binder 线程的条件：

1. 进程没有请求创建 binder 线程 `proc->requested_threads == 0`

> @requested_threads: number of binder threads requested but not yet started. In current implementation, can only be 0 or 1.

binder 发送 `BR_SPAWN_LOOPER` 给应用进程后将 requested_threads 置真，直到应用进程将新线程启动并回复 `BC_REGISTER_LOOPER` 后才将 requested_threads 置假

2. 没有等待/阻塞在进程 todo list 的 client thread，`list_empty(&thread->proc->waiting_threads)`

> @waiting_threads: threads currently waiting for proc work

参考 [client 线程调度例子](#client-线程调度例子) 里的 `binder_wait_for_work`

3. 已启动的 binder 线程数小于上限，`proc->requested_threads_started < proc->max_threads`

binder 线程池上限默认 15，可通过 `BINDER_SET_MAX_THREADS` 修改

4. 当前线程已从应用进程收到 `BC_ENTER_LOOPER` or `BC_REGISTER_LOOPER`，`thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)`

参考 [主线程与普通线程](#主线程与普通线程)

```cpp
// https://cs.android.com/android/kernel/superproject/+/common-android-mainline:common/drivers/android/binder.c;l=4541?q=binder_thread_read
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
// ...
done:
	*consumed = ptr - buffer;
	binder_inner_proc_lock(proc);

	if (proc->requested_threads == 0 &&
	    list_empty(&thread->proc->waiting_threads) &&
	    proc->requested_threads_started < proc->max_threads &&
	    (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
	     BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
	     /*spawn a new thread if we leave this out */) {

		proc->requested_threads++;
		binder_inner_proc_unlock(proc);
		if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
			return -EFAULT;
		binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
	} else
		binder_inner_proc_unlock(proc);
	return 0;
}

// 应用进程收到 BR_SPAWN_LOOPER 后启动新的 binder 普通线程，参考【主线程与普通线程】对 spawnPooledThread(false) 的解析
// https://cs.android.com/android/platform/superproject/+/master:system/libhwbinder/IPCThreadState.cpp
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    case BR_SPAWN_LOOPER:
        mProcess->spawnPooledThread(false);
        break;
}


```

# client 线程调度例子

一次事务 `binder_thread.transaction_stack` 表示：client 发起请求 -> server 处理并返回响应 -> client 收到响应这么一整个流程，类似于一次完整的 HTTP 请求

client 是单线程即发起 Binder IPC 的那个线程，将 request 添加到目标进程 `binder_proc.todo` 或者目标线程 `binder_thread.todo` 的任务列表后，通过 `schedule()` 让渡出 cpu 资源从而实现 [线程休眠](https://github.com/cyrus-lin/bookmark/issues/45)，直到 server 返回响应唤醒它（`wake_up`）

`binder_thread_write` 将 request 添加到目标进程/目标线程的 todo 任务队列，`binder_thread_read` 将休眠此 client 线程直到被 server reponse 唤醒

```cpp
ActivityManager.getRunningAppProcesses                                                 // APP 进程 Java 层
IServiceManager.Stub.Proxy.getRunningAppProcesses
BinderProxy.transact(int code, Parcel data, Parcel reply, int flags)
BpBinder.transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)    // APP 进程 native 层
IPCThreadState::self()->transact(binderHandle(), code, data, reply, flags)
IPCThreadState::writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr)
IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
IPCThreadState::talkWithDriver(bool doReceive)
ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)                                     // 陷入内核来到 binder driver
binder_ioctl

// https://cs.android.com/android/kernel/superproject/+/common-android-mainline:common/drivers/android/binder.c
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread /* 当前线程，此时即是 client 执行 request 的线程 */ )
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    unsigned int size = _IOC_SIZE(cmd);    // sizeof(struct binder_write_read)
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;
    if (size != sizeof(struct binder_write_read)) {
        ret = -EINVAL;
        goto out;
    }
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {  // 从用户进程地址空间拷贝 struct binder_write_read 至内核
        ret = -EFAULT;
        goto out;
    }    
	// binder_write_read.write_size > 0 表示用户进程有数据发送到内核驱动
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
    // binder_write_read.read_size > 0 表示用户进程希望内核驱动返回 read_size 大小的数据给他
	if (bwr.read_size > 0) {
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
		trace_binder_read_done(ret);
		binder_inner_proc_lock(proc);
		if (!binder_worklist_empty_ilocked(&proc->todo))
			binder_wakeup_proc_ilocked(proc);
		binder_inner_proc_unlock(proc);
		if (ret < 0) {
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  // 处理成功，将 struct binder_write_read 从内核拷贝/覆盖至用户进程空间
        ret = -EFAULT;
        goto out;
    }    
	// ...
}

// 将数据 binder_buffer 从用户内存空间发送到内核地址空间
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer,  /* binder_write_read.write_buffer */ 
            size_t size,                     /* binder_write_read.write_size */
            binder_size_t *consumed)         /* 输出，从 write_buffer 读取了多少数据 */
{
    uint32_t cmd;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;  // write_buffer 起始地址
    void __user *ptr = buffer + *consumed;                          // 指示器，标识下一次读取的位置
    void __user *end = buffer + size;                               // write_buffer 结束地址
    while (ptr < end && thread->return_error == BR_OK) {
        if (get_user(cmd, (uint32_t __user *)ptr))                  // 从用户内存空间 write_buffer 拷贝 4 bytes 到 cmd
            return -EFAULT;
        ptr += sizeof(uint32_t);
        trace_binder_command(cmd);
        if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {
            binder_stats.bc[_IOC_NR(cmd)]++;
            proc->stats.bc[_IOC_NR(cmd)]++;
            thread->stats.bc[_IOC_NR(cmd)]++;
        }

        switch (cmd) {                          // write 是指将数据从用户内存空间发送到内核内存空间
        case BC_TRANSACTION:                    // 客户端的请求数据 client request
        case BC_REPLY: {                        // 服务端的响应数据 server response
            struct binder_transaction_data tr;  // 它们使用的数据结构都是 binder_transaction_data
            if (copy_from_user(&tr, ptr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);
            binder_transaction**(proc, thread, &tr, cmd == BC_REPLY);
            break;
            }
        // ...
        }
    }
}

static void binder_transaction(struct binder_proc *proc,
                struct binder_thread *thread,
                struct binder_transaction_data *tr, 
                int reply)    // false: client request, true: server response
{
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    binder_size_t *offp, *off_end;
    binder_size_t off_min;
    struct binder_proc *target_proc;
    struct binder_thread *target_thread = NULL;**
    struct binder_node *target_node = NULL;
    struct list_head *target_list;
    wait_queue_head_t *target_wait;
    struct binder_transaction *in_reply_to = NULL;
    struct binder_transaction_log_entry *e;
    uint32_t return_error;
    e = binder_transaction_log_add(&binder_transaction_log);
    e->call_type = reply ? 2 : !!(tr->flags & TF_ONE_WAY);
    e->from_proc = proc->pid;
    e->from_thread = thread->pid;
    e->target_handle = tr->target.handle;
    e->data_size = tr->data_size;
    e->offsets_size = tr->offsets_size;
    
    if (reply) {                    // BC_REPLY 说明这是一个 Server 发给 Client 的事务处理回复，在 server 端的线程上
        // ...
    } else {                        // BC_TRANSACTION 说明这是一个 Client 发给 Server 的请求事务，在 Client 端线程上
        if (tr->target.handle) {    // 确定目标进程 target_proc
            struct binder_ref *ref;
            ref = binder_get_ref(proc, tr->target.handle);
            if (ref == NULL) {
                binder_user_error("%d:%d got transaction to invalid handle\n",
                    proc->pid, thread->pid);
                return_error = BR_FAILED_REPLY;
                goto err_invalid_target_handle;
            }
            target_node = ref->node;
        } else {                     // handle == 0 表示是 service manager 进程
            target_node = binder_context_mgr_node;
            if (target_node == NULL) {
                return_error = BR_DEAD_REPLY;
                goto err_no_context_mgr_node;
            }
        }
        e->to_node = target_node->debug_id;
        target_proc = target_node->proc;
        if (target_proc == NULL) {
            return_error = BR_DEAD_REPLY;
            goto err_dead_binder;
        }
        if (security_binder_transaction(proc->tsk,
                        target_proc->tsk) < 0) {
            return_error = BR_FAILED_REPLY;
            goto err_invalid_target_handle;
        }

        // 确定目标线程 target_thread
        // 非 one_way（需要replay），则从栈顶向下搜索，从历史事务记录中过滤出与目标进程的通讯记录，复用以往使用过的线程
        if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
            struct binder_transaction *tmp;
            tmp = thread->transaction_stack;
            if (tmp->to_thread != thread) {
                binder_user_error("%d:%d got new transaction with bad transaction stack, transaction %d has target %d:%d\n",
                    proc->pid, thread->pid, tmp->debug_id,
                    tmp->to_proc ? tmp->to_proc->pid : 0,
                    tmp->to_thread ?
                    tmp->to_thread->pid : 0);
                return_error = BR_FAILED_REPLY;
                goto err_bad_call_stack;
            }
            while (tmp) {
                if (tmp->from && tmp->from->proc == target_proc)
                    target_thread = tmp->from;
                tmp = tmp->from_parent;
            }
        }
    }

    // 找到 target_thread, 则 target_list 和 target_wait 分别初始化为目标线程的 todo 和 wait
    // 这个意味着 client thread 与目标进程有过通讯记录
    if (target_thread) {
        e->to_thread = target_thread->pid;
        target_list = &target_thread->todo;
        target_wait = &target_thread->wait;
    } else {

        // 没有找到 target_thread, 那么 target_list 和 target_wait 分别初始化为目标进程的 todo 和 wait
        // 这个情况只有BC_TRANSACTION命令才有可能发生
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }
    e->to_proc = target_proc->pid;
    
    // 发送到目标进程 todo 列表的事务，由目标进程处理
    t = kzalloc(sizeof(*t), GFP_KERNEL);                 
    if (t == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_alloc_t_failed;
    }
    binder_stats_created(BINDER_STAT_TRANSACTION);

    // 发送到本线程 todo 列表的任务，该 binder_work 会发送 BR_WORK_TRANSCAION_COMPLETE 给目标进程，告知请求/回复已发送出去
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    if (tcomplete == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_alloc_tcomplete_failed;
    }
    binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);

    // 初始化 binder_transaction
    if (!reply && !(tr->flags & TF_ONE_WAY)) // BC_TRANSACTION 且不是 one way，即需要 replay，则发起线程（from）设为当前线程
      t->from = thread;
    else                                     // BC_REPLY，from 置为空
        t->from = NULL;
    t->sender_euid = task_euid(proc->tsk);
    t->to_proc = target_proc;
    t->to_thread = target_thread;
    t->code = tr->code;
    t->flags = tr->flags;
    t->priority = task_nice(current);
    trace_binder_transaction(reply, t, target_node);
    t->buffer = **binder_alloc_buf**(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
    if (t->buffer == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_binder_alloc_buf_failed;
    }
    t->buffer->allow_user_free = 0;
    t->buffer->debug_id = t->debug_id;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;
    trace_binder_transaction_alloc_buf(t->buffer);
    if (target_node)
       binder_inc_node(target_node, 1, 0, NULL);

    // 做一些内存拷贝工作，从用户内存空间 -> 内核空间
    // ...

    if (reply) {  // 事务处理完成，将本次 reply 对应的 transaction 从目标线程（Client）事务栈中移除，并释放其所占用的地址空间
        BUG_ON(t->buffer->async_transaction != 0);
        binder_pop_transaction(target_thread, in_reply_to);
    } else if (!(t->flags & TF_ONE_WAY)) {

        // 一个 client 到 server 的 transaction，且需要 reply，将本次事务加入到本线程(client)事务栈中
        BUG_ON(t->buffer->async_transaction != 0);
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        thread->transaction_stack = t;
    } else {
        BUG_ON(target_node == NULL);
        BUG_ON(t->buffer->async_transaction != 1);
        if (target_node->has_async_transaction) {
            target_list = &target_node->async_todo;
            *target_wait = NULL;
        } else
            target_node->has_async_transaction = 1;
    }

    // 将事务 binder_transaction 加入至目标进程 or 目标线程的 todo 列表
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);

    // 添加一个事务到本线程的 todo 队列中，稍后在处理 todo 队列时（binder_thread_write）
    // 该 binder_work 会发送 BR_WORK_TRANSCAION_COMPLETE 给目标进程，告知请求/回复已发送出去
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);

    // 唤醒休眠在 target_wait 上的线程去处理 todo 列表
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;

err_get_unused_fd_failed:
err_fget_failed:
err_fd_not_allowed:
err_binder_get_ref_for_node_failed:
err_binder_get_ref_failed:
// ... 异常处理
}

static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	int ret = 0;
	int wait_for_proc_work;

    // 当前的写入位置为 bwr.read_buffer 的起始位置，先写入一个 BR_NOOP 命令到 read_buffer 中
    // 该命令在用户态是一个空操作，什么也不做，主要意义应该是在输出日志等
	if (*consumed == 0) {
		if (put_user(BR_NOOP, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
	}

    // 	!thread->transaction_stack && 
    //      binder_worklist_empty_ilocked(&thread->todo) && 
    //      (thread->looper & (BINDER_LOOPER_STATE_ENTERED | BINDER_LOOPER_STATE_REGISTERED))
    // 
    // 如果线程的事务栈和 todo 列表都为空，表示线程没有任务要做，则去执行线程所在进程的 todo 列表上的任务
retry:
	binder_inner_proc_lock(proc);
	wait_for_proc_work = binder_available_for_proc_work_ilocked(thread);
	binder_inner_proc_unlock(proc);

	thread->looper |= BINDER_LOOPER_STATE_WAITING;

	trace_binder_wait_for_work(wait_for_proc_work,
				   !!thread->transaction_stack,
				   !binder_worklist_empty(proc, &thread->todo));
	if (wait_for_proc_work) {
		if (!(thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
					BINDER_LOOPER_STATE_ENTERED))) {
			binder_user_error("%d:%d ERROR: Thread waiting for process work before calling BC_REGISTER_LOOPER or BC_ENTER_LOOPER (state %x)\n",
				proc->pid, thread->pid, thread->looper);
			wait_event_interruptible(binder_user_error_wait,
						 binder_stop_on_user_error < 2);
		}
		trace_android_vh_binder_restore_priority(NULL, current);
		binder_restore_priority(thread, &proc->default_priority);
	}

    // thread->process_todo ||
	//     thread->looper_need_return ||
	// 	   (do_proc_work && !binder_worklist_empty_ilocked(&thread->proc->todo))
    //
    // 如上面所说，线程自己的事务栈和 todo 为空，且进程的 todo 也为空，则线程没有任务要执行
    // 非阻塞的情况下，返回 EAGAIN 让调用者重试
	if (non_block) {
		if (!binder_has_work(thread, wait_for_proc_work))
			ret = -EAGAIN;
	} else {  // 阻塞情况下，休眠当前线程直到别的线程 wake_up() 唤醒它
		ret = binder_wait_for_work(thread, wait_for_proc_work);
	}
	thread->looper &= ~BINDER_LOOPER_STATE_WAITING;
	if (ret)
		return ret;

	while (1) {
		uint32_t cmd;
		struct binder_transaction_data_secctx tr;
		struct binder_transaction_data *trd = &tr.transaction_data;
		struct binder_work *w = NULL;
		struct list_head *list = NULL;
		struct binder_transaction *t = NULL;
		struct binder_thread *t_from;
		size_t trsize = sizeof(*trd);

        // 找到 todo 列表：优先处理本线程的 todo 列表，同时也处理进程的 todo 列表
		binder_inner_proc_lock(proc);
		if (!binder_worklist_empty_ilocked(&thread->todo))
			list = &thread->todo;
		else if (!binder_worklist_empty_ilocked(&proc->todo) &&
			   wait_for_proc_work)
			list = &proc->todo;
		else {
			binder_inner_proc_unlock(proc);

			/* no data added */
			if (ptr - buffer == 4 && !thread->looper_need_return)
				goto retry;
			break;
		}

		if (end - ptr < sizeof(tr) + 4) {
			binder_inner_proc_unlock(proc);
			break;
		}
		w = binder_dequeue_work_head_ilocked(list);
		if (binder_worklist_empty_ilocked(&thread->todo))
			thread->process_todo = false;

        // 上面 binder_thread_write 那里塞了个 BINDER_WORK_TRANSACTION_COMPLETE 到当前线程的 todo 列表里
        // 这里回写个 BR_TRANSACTION_COMPLETE 到用户空间，然后跳转到 retry 休眠线程
		switch (w->type) {
		case BINDER_WORK_TRANSACTION_COMPLETE:
		case BINDER_WORK_TRANSACTION_ONEWAY_SPAM_SUSPECT: {
			if (proc->oneway_spam_detection_enabled &&
				   w->type == BINDER_WORK_TRANSACTION_ONEWAY_SPAM_SUSPECT)
				cmd = BR_ONEWAY_SPAM_SUSPECT;
			else
				cmd = BR_TRANSACTION_COMPLETE;
			binder_inner_proc_unlock(proc);
			kfree(w);
			binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
			if (put_user(cmd, (uint32_t __user *)ptr))
				return -EFAULT;
			ptr += sizeof(uint32_t);

			binder_stat_br(proc, thread, cmd);
			binder_debug(BINDER_DEBUG_TRANSACTION_COMPLETE,
				     "%d:%d BR_TRANSACTION_COMPLETE\n",
				     proc->pid, thread->pid);
		} break;

        // 在 retry 里休眠的线程，直到被 server 唤醒，当然不一定是刚才那个线程，可能是被别的线程抢到了
        // server 进程在 binder_thread_write -> binder_transaction 最后会通过 wake_up_interruptible
        // 唤醒目标线程，也就是 client 发送 request 的那个线程
        case BINDER_WORK_TRANSACTION: {
			binder_inner_proc_unlock(proc);
			t = container_of(w, struct binder_transaction, work);
		} break;
		// ... 
		} // switch-end
		if (!t)
			continue;

        // 拿到 binder_transaction 后继续流程
		BUG_ON(t->buffer == NULL);
		if (t->buffer->target_node) {
			struct binder_node *target_node = t->buffer->target_node;

			trd->target.ptr = target_node->ptr;
			trd->cookie =  target_node->cookie;
			binder_transaction_priority(thread, t, target_node);
			cmd = BR_TRANSACTION;
		} else {
			trd->target.ptr = 0;
			trd->cookie = 0;
			cmd = BR_REPLY;
		}
		trd->code = t->code;
		trd->flags = t->flags;
		trd->sender_euid = from_kuid(current_user_ns(), t->sender_euid);

		t_from = binder_get_txn_from(t);
		if (t_from) {
			struct task_struct *sender = t_from->proc->tsk;

			trd->sender_pid =
				task_tgid_nr_ns(sender,
						task_active_pid_ns(current));
			trace_android_vh_sync_txn_recvd(thread->task, t_from->task);
		} else {
			trd->sender_pid = 0;
		}

		ret = binder_apply_fd_fixups(proc, t);
		if (ret) {
			struct binder_buffer *buffer = t->buffer;
			bool oneway = !!(t->flags & TF_ONE_WAY);
			int tid = t->debug_id;

			if (t_from)
				binder_thread_dec_tmpref(t_from);
			buffer->transaction = NULL;
			binder_cleanup_transaction(t, "fd fixups failed",
						   BR_FAILED_REPLY);
			binder_free_buf(proc, thread, buffer, true);
			binder_debug(BINDER_DEBUG_FAILED_TRANSACTION,
				     "%d:%d %stransaction %d fd fixups failed %d/%d, line %d\n",
				     proc->pid, thread->pid,
				     oneway ? "async " :
					(cmd == BR_REPLY ? "reply " : ""),
				     tid, BR_FAILED_REPLY, ret, __LINE__);
			if (cmd == BR_REPLY) {
				cmd = BR_FAILED_REPLY;
				if (put_user(cmd, (uint32_t __user *)ptr))
					return -EFAULT;
				ptr += sizeof(uint32_t);
				binder_stat_br(proc, thread, cmd);
				break;
			}
			continue;
		}
		trd->data_size = t->buffer->data_size;
		trd->offsets_size = t->buffer->offsets_size;
		trd->data.ptr.buffer = (uintptr_t)t->buffer->user_data;
		trd->data.ptr.offsets = trd->data.ptr.buffer +
					ALIGN(t->buffer->data_size,
					    sizeof(void *));

		tr.secctx = t->security_ctx;
		if (t->security_ctx) {
			cmd = BR_TRANSACTION_SEC_CTX;
			trsize = sizeof(tr);
		}
		if (put_user(cmd, (uint32_t __user *)ptr)) {
			if (t_from)
				binder_thread_dec_tmpref(t_from);

			binder_cleanup_transaction(t, "put_user failed",
						   BR_FAILED_REPLY);

			return -EFAULT;
		}
		ptr += sizeof(uint32_t);
		if (copy_to_user(ptr, &tr, trsize)) {
			if (t_from)
				binder_thread_dec_tmpref(t_from);

			binder_cleanup_transaction(t, "copy_to_user failed",
						   BR_FAILED_REPLY);

			return -EFAULT;
		}
		ptr += trsize;

		trace_binder_transaction_received(t);
		binder_stat_br(proc, thread, cmd);
		binder_debug(BINDER_DEBUG_TRANSACTION,
			     "%d:%d %s %d %d:%d, cmd %u size %zd-%zd ptr %016llx-%016llx\n",
			     proc->pid, thread->pid,
			     (cmd == BR_TRANSACTION) ? "BR_TRANSACTION" :
				(cmd == BR_TRANSACTION_SEC_CTX) ?
				     "BR_TRANSACTION_SEC_CTX" : "BR_REPLY",
			     t->debug_id, t_from ? t_from->proc->pid : 0,
			     t_from ? t_from->pid : 0, cmd,
			     t->buffer->data_size, t->buffer->offsets_size,
			     (u64)trd->data.ptr.buffer,
			     (u64)trd->data.ptr.offsets);

		if (t_from)
			binder_thread_dec_tmpref(t_from);
		t->buffer->allow_user_free = 1;
		if (cmd != BR_REPLY && !(t->flags & TF_ONE_WAY)) {
			binder_inner_proc_lock(thread->proc);
			t->to_parent = thread->transaction_stack;
			t->to_thread = thread;
			thread->transaction_stack = t;
			binder_inner_proc_unlock(thread->proc);
		} else {
			binder_free_transaction(t);
		}
		break;
	} // while-end

done:

	*consumed = ptr - buffer;
	binder_inner_proc_lock(proc);
	if (proc->requested_threads == 0 &&
	    list_empty(&thread->proc->waiting_threads) &&
	    proc->requested_threads_started < proc->max_threads &&
	    (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
	     BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
	     /*spawn a new thread if we leave this out */) {
		proc->requested_threads++;
		binder_inner_proc_unlock(proc);
		binder_debug(BINDER_DEBUG_THREADS,
			     "%d:%d BR_SPAWN_LOOPER\n",
			     proc->pid, thread->pid);
		if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
			return -EFAULT;
		binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
	} else
		binder_inner_proc_unlock(proc);
	return 0;
}

// prepare_to_wait + schedule + finish_wait 是一整套线程调度模板（休眠线程）
// see https://github.com/cyrus-lin/bookmark/issues/45
// 它使线程休眠在 binder_thread.wait 上直到别的线程通过 wake_up(head) 唤醒它
static int binder_wait_for_work(struct binder_thread *thread,
				bool do_proc_work)
{
	DEFINE_WAIT(wait);
	struct binder_proc *proc = thread->proc;
	int ret = 0;

	binder_inner_proc_lock(proc);
	for (;;) {
		prepare_to_wait(&thread->wait, &wait, TASK_INTERRUPTIBLE|TASK_FREEZABLE);
		if (binder_has_work_ilocked(thread, do_proc_work))
			break;
		if (do_proc_work)
			list_add(&thread->waiting_thread_node,
				 &proc->waiting_threads);
		trace_android_vh_binder_wait_for_work(do_proc_work, thread, proc);
		binder_inner_proc_unlock(proc);
		schedule();
		binder_inner_proc_lock(proc);
		list_del_init(&thread->waiting_thread_node);
		if (signal_pending(current)) {
			ret = -EINTR;
			break;
		}
	}
	finish_wait(&thread->wait, &wait);
	binder_inner_proc_unlock(proc);
	return ret;
}
```

# binder 线程主例程

```cpp
// https://cs.android.com/android/platform/superproject/+/master:system/libhwbinder/IPCThreadState.cpp
void IPCThreadState::joinThreadPool(bool isMain /* true - 主线程*/ )
{
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    mIsLooper = true;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER);
    mIsLooper = false;
    talkWithDriver(false);
}

status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;

    result = talkWithDriver();  // doReceive 默认为 true
    // ...
}

status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD < 0) {
        return -EBADF;
    }

    binder_write_read bwr;

    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    IF_LOG_COMMANDS() {
        if (outAvail != 0) {
            alog << "Sending commands to driver: " << indent;
            const void* cmds = (const void*)bwr.write_buffer;
            const void* end = ((const uint8_t*)cmds)+bwr.write_size;
            alog << HexDump(cmds, bwr.write_size) << endl;
            while (cmds < end) cmds = printCommand(alog, cmds);
            alog << dedent;
        }
        alog << "Size of receive buffer: " << bwr.read_size
            << ", needRead: " << needRead << ", doReceive: " << doReceive << endl;
    }

    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        IF_LOG_COMMANDS() {
            alog << "About to read/write, write size = " << mOut.dataSize() << endl;
        }
#if defined(__ANDROID__)
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD < 0) {
            err = -EBADF;
        }
        IF_LOG_COMMANDS() {
            alog << "Finished read/write, write size = " << mOut.dataSize() << endl;
        }
    } while (err == -EINTR);

    IF_LOG_COMMANDS() {
        alog << "Our err: " << (void*)(intptr_t)err << ", write consumed: "
            << bwr.write_consumed << " (of " << mOut.dataSize()
                        << "), read consumed: " << bwr.read_consumed << endl;
    }

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                LOG_ALWAYS_FATAL("Driver did not consume write buffer. "
                                 "err: %s consumed: %zu of %zu",
                                 statusToString(err).c_str(),
                                 (size_t)bwr.write_consumed,
                                 mOut.dataSize());
            else {
                mOut.setDataSize(0);
                processPostWriteDerefs();
            }
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        IF_LOG_COMMANDS() {
            alog << "Remaining data size: " << mOut.dataSize() << endl;
            alog << "Received commands from driver: " << indent;
            const void* cmds = mIn.data();
            const void* end = mIn.data() + mIn.dataSize();
            alog << HexDump(cmds, mIn.dataSize()) << endl;
            while (cmds < end) cmds = printReturnCommand(alog, cmds);
            alog << dedent;
        }
        return NO_ERROR;
    }

    return err;
}
```

# server 线程调度

[上一章节](#client-线程调度例子)说过，binder 会将 request 添加到 server 进程的 todo list 并唤醒休眠在 wait 上的 server 线程

```cpp
static void binder_transaction(struct binder_proc *proc,
                struct binder_thread *thread,
                struct binder_transaction_data *tr, 
                int reply)    // false: client request, true: server response
{
	// ...
    wait_queue_head_t *target_wait;

	// ...
    // 找到 target_thread, 则 target_list 和 target_wait 分别初始化为目标线程的 todo 和 wait
    // 这个意味着 client thread 与目标进程有过通讯记录
    if (target_thread) {
        e->to_thread = target_thread->pid;
        target_list = &target_thread->todo;
        target_wait = &target_thread->wait;
    } else {

        // 没有找到 target_thread, 那么 target_list 和 target_wait 分别初始化为目标进程的 todo 和 wait
        // 这个情况只有BC_TRANSACTION命令才有可能发生
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }

    // ...
    // 唤醒休眠在 target_wait 上的线程去处理 todo 列表
    if (target_wait)
        wake_up_interruptible(target_wait);
    // ...
}
```

# 参考

- [Binder驱动之设备控制 binder_ioctl -- 一](https://www.jianshu.com/p/49830c3473b7)
- [Binder驱动之设备控制 binder_ioctl -- 二](https://www.jianshu.com/p/12bec1b16a5b)
- [Binder驱动之设备控制 binder_ioctl -- 三](https://www.jianshu.com/p/4be292a51388)
- [进程的Binder线程池工作过程 - Gityuan](http://gityuan.com/2016/10/29/binder-thread-pool/)