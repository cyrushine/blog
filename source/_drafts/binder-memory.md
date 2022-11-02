为什么 Binder IPC 时只需要复制一份数据？一般的 IPC 需要复制几次

# client request

## get server binder




## send request

binder client native 层与 binder driver 通讯用的是 `BpBinder::transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)`，它又调用 `IPCThreadState::transact(int32_t handle, uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)`

`IPCThreadState` 是线程私有的，它依靠全局单例 `ProcessState` 打开 binder driver 得到 binder fd

> pthread_getspecific, pthread_setspecific — thread-specific data management
> 
> The pthread_getspecific() function shall return the value currently bound to the specified key on behalf of the calling thread.
> 
> The pthread_setspecific() function shall associate a thread-specific value with a key obtained via a previous call to pthread_key_create().  Different threads may bind different values to the same key. 

```cpp
// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
        // ...
        status_t status;
        if (CC_UNLIKELY(isRpcBinder())) {
            status = rpcSession()->transact(sp<IBinder>::fromExisting(this), code, data, reply,
                                            flags);
        } else {
            if constexpr (!kEnableKernelIpc) {
                LOG_ALWAYS_FATAL("Binder kernel driver disabled at build time");
                return INVALID_OPERATION;
            }
            // binderHandle() 是 server 句柄，比如 servicemanager 就是 0
            // IPCThreadState::self() 返回线程本地/线程私有的 IPCThreadState 实例
            status = IPCThreadState::self()->transact(binderHandle(), code, data, reply, flags);
        }
        // ...
}

// IPCThreadState 是线程私有的
// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS.load(std::memory_order_acquire)) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState;
    }
    // ...
}

// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp
IPCThreadState::IPCThreadState()
      : mProcess(ProcessState::self()),
        mServingStackPointer(nullptr),
        mServingStackPointerGuard(nullptr),
        mWorkSource(kUnsetWorkSource),
        mPropagateWorkSource(false),
        mIsLooper(false),
        mIsFlushing(false),
        mStrictModePolicy(0),
        mLastTransactionBinderFlags(0),
        mCallRestriction(mProcess->mCallRestriction) {
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
}

// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp
sp<ProcessState> ProcessState::self()
{
    return init(kDefaultDriver /* /dev/binder */, false /*requireDefault*/);
}

// ProcessState 是全局单例模式（gProcess）
sp<ProcessState> ProcessState::init(const char *driver, bool requireDefault)
{
    if (driver == nullptr) {
        std::lock_guard<std::mutex> l(gProcessMutex);
        if (gProcess) {
            verifyNotForked(gProcess->mForked);
        }
        return gProcess;
    }

    [[clang::no_destroy]] static std::once_flag gProcessOnce;
    std::call_once(gProcessOnce, [&](){
        if (access(driver, R_OK) == -1) {
            ALOGE("Binder driver %s is unavailable. Using /dev/binder instead.", driver);
            driver = "/dev/binder";
        }

        // we must install these before instantiating the gProcess object,
        // otherwise this would race with creating it, and there could be the
        // possibility of an invalid gProcess object forked by another thread
        // before these are installed
        int ret = pthread_atfork(ProcessState::onFork, ProcessState::parentPostFork,
                                 ProcessState::childPostFork);
        LOG_ALWAYS_FATAL_IF(ret != 0, "pthread_atfork error %s", strerror(ret));

        std::lock_guard<std::mutex> l(gProcessMutex);
        gProcess = sp<ProcessState>::make(driver);
    });

    if (requireDefault) {
        // Detect if we are trying to initialize with a different driver, and
        // consider that an error. ProcessState will only be initialized once above.
        LOG_ALWAYS_FATAL_IF(gProcess->getDriverName() != driver,
                            "ProcessState was already initialized with %s,"
                            " can't initialize with %s.",
                            gProcess->getDriverName().c_str(), driver);
    }

    verifyNotForked(gProcess->mForked);
    return gProcess;
}

// 构造 ProcessState 的时候就会打开 binder：open("/dev/binder") && mmap
// http://www.aospxref.com/android-13.0.0_r3/xref/frameworks/native/libs/binder/ProcessState.cpp#ProcessState
ProcessState::ProcessState(const char* driver)
      : mDriverName(String8(driver)),
        mDriverFD(-1),
        mVMStart(MAP_FAILED),
        mThreadCountLock(PTHREAD_MUTEX_INITIALIZER),
        mThreadCountDecrement(PTHREAD_COND_INITIALIZER),
        mExecutingThreadsCount(0),
        mWaitingForThreads(0),
        mMaxThreads(DEFAULT_MAX_BINDER_THREADS),
        mStarvationStartTimeMs(0),
        mForked(false),
        mThreadPoolStarted(false),
        mThreadPoolSeq(1),
        mCallRestriction(CallRestriction::NONE) {
    base::Result<int> opened = open_driver(driver);

    if (opened.ok()) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE,
                        opened.value(), 0);
        if (mVMStart == MAP_FAILED) {
            close(opened.value());
            // *sigh*
            opened = base::Error()
                    << "Using " << driver << " failed: unable to mmap transaction memory.";
            mDriverName.clear();
        }
    }

    if (opened.ok()) {
        mDriverFD = opened.value();
    }
}

static base::Result<int> open_driver(const char* driver) {
    int fd = open(driver, O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        return base::ErrnoError() << "Opening '" << driver << "' failed";
    }
    int vers = 0;
    status_t result = ioctl(fd, BINDER_VERSION, &vers);
    if (result == -1) {
        close(fd);
        return base::ErrnoError() << "Binder ioctl to obtain version failed";
    }
    if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
        close(fd);
        return base::Error() << "Binder driver protocol(" << vers
                             << ") does not match user space protocol("
                             << BINDER_CURRENT_PROTOCOL_VERSION
                             << ")! ioctl() return value: " << result;
    }
    size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
    result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
    if (result == -1) {
        ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
    }
    uint32_t enable = DEFAULT_ENABLE_ONEWAY_SPAM_DETECTION;
    result = ioctl(fd, BINDER_ENABLE_ONEWAY_SPAM_DETECTION, &enable);
    if (result == -1) {
        ALOGE_IF(ProcessState::isDriverFeatureEnabled(
                     ProcessState::DriverFeature::ONEWAY_SPAM_DETECTION),
                 "Binder ioctl to enable oneway spam detection failed: %s", strerror(errno));
    }
    return fd;
}

// http://www.aospxref.com/android-13.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    // 将 [BC_TRANSACTION][binder_transaction_data] 写入 mOut
    // mOut 是 client 进程内的一块内存区域
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);
    // 将 request 发送给 binder driver 并等待 response
    err = waitForResponse(reply);
    // ...
    return err;
}

status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}

// 处理 response
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();
        switch (cmd) {
        case BR_FAILED_REPLY:
        case BR_FROZEN_REPLY:
        case BR_ACQUIRE_RESULT:
        case BR_REPLY:
        // ...
}

// 将 request 发送给 binder driver 并接收 response
// ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, binder_write_read)
// [write_buffer, write_size] 是 request
// [read_buffer, read_size] 是 response
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

    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
        if (mProcess->mDriverFD < 0) {
            err = -EBADF;
        }
    } while (err == -EINTR);
    // ...
}
```

# binder get request

https://zhuanlan.zhihu.com/p/381310378

```cpp
// https://vscode.dev/github/cyrus-lin/msm-google
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;

	binder_selftest_alloc(&proc->alloc);

	trace_binder_ioctl(cmd, arg);

	ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	if (ret)
		goto err_unlocked;

	thread = binder_get_thread(proc);  // 取当前处理器核心对应的 binder_thread
	if (thread == NULL) {
		ret = -ENOMEM;
		goto err;
	}

	switch (cmd) {
	case BINDER_WRITE_READ:
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
	// ...
}

// binder_proc 有个红黑树结构的线程组 threads
// thread->pid 取创建 binder_thread 结构的处理器核心的 id 并按此成员属性排序，左小右大
// 那么最终 binder_proc->threads 就是一个大小等于处理器核心数的红黑树
// binder_get_thread 就是取当前处理器核心对应的那个 binder_thread 实例
struct binder_proc {
	struct rb_root threads;
    // ...
};
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
	struct binder_thread *thread;
	struct binder_thread *new_thread;

	binder_inner_proc_lock(proc);
	thread = binder_get_thread_ilocked(proc, NULL);
	binder_inner_proc_unlock(proc);
	if (!thread) {
		new_thread = kzalloc(sizeof(*thread), GFP_KERNEL);
		if (new_thread == NULL)
			return NULL;
		binder_inner_proc_lock(proc);
		thread = binder_get_thread_ilocked(proc, new_thread);
		binder_inner_proc_unlock(proc);
		if (thread != new_thread)
			kfree(new_thread);
	}
	return thread;
}
static struct binder_thread *binder_get_thread_ilocked(
		struct binder_proc *proc, struct binder_thread *new_thread)
{
	struct binder_thread *thread = NULL;
	struct rb_node *parent = NULL;
	struct rb_node **p = &proc->threads.rb_node;

	while (*p) {
		parent = *p;
		thread = rb_entry(parent, struct binder_thread, rb_node);

		if (current->pid < thread->pid)
			p = &(*p)->rb_left;
		else if (current->pid > thread->pid)
			p = &(*p)->rb_right;
		else
			return thread;
	}
	if (!new_thread)
		return NULL;
	thread = new_thread;
	binder_stats_created(BINDER_STAT_THREAD);
	thread->proc = proc;
	thread->pid = current->pid;
	get_task_struct(current);
	thread->task = current;
	atomic_set(&thread->tmp_ref, 0);
	init_waitqueue_head(&thread->wait);
	INIT_LIST_HEAD(&thread->todo);
	rb_link_node(&thread->rb_node, parent, p);
	rb_insert_color(&thread->rb_node, &proc->threads);
	thread->looper_need_return = true;
	thread->return_error.work.type = BINDER_WORK_RETURN_ERROR;
	thread->return_error.cmd = BR_OK;
	thread->reply_error.work.type = BINDER_WORK_RETURN_ERROR;
	thread->reply_error.cmd = BR_OK;
	spin_lock_init(&thread->prio_lock);
	thread->prio_state = BINDER_PRIO_SET;
	thread->ee.command = BR_OK;
	INIT_LIST_HEAD(&new_thread->waiting_thread_node);
	return thread;
}

static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread /* 当前处理器核心对应的 binder_thread */)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data; // 当前进程，调用 ioctl 的进程
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
	binder_debug(BINDER_DEBUG_READ_WRITE,
		     "%d:%d wrote %lld of %lld, read return %lld of %lld\n",
		     proc->pid, thread->pid,
		     (u64)bwr.write_consumed, (u64)bwr.write_size,
		     (u64)bwr.read_consumed, (u64)bwr.read_size);
	if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
out:
	return ret;
}

static int binder_thread_write(struct binder_proc *proc /* 当前进程 */,
			struct binder_thread *thread /* 当前处理器核心对应的 binder_thread */,
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
		trace_binder_command(cmd);
		if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {
			atomic_inc(&binder_stats.bc[_IOC_NR(cmd)]);
			atomic_inc(&proc->stats.bc[_IOC_NR(cmd)]);
			atomic_inc(&thread->stats.bc[_IOC_NR(cmd)]);
		}
		switch (cmd) {
		case BC_TRANSACTION:
		case BC_REPLY: {
			struct binder_transaction_data tr;

			if (copy_from_user(&tr, ptr, sizeof(tr)))
				return -EFAULT;
			ptr += sizeof(tr);
			binder_transaction(proc, thread, &tr,
					   cmd == BC_REPLY, 0);
			break;
		}
        // ...
		*consumed = ptr - buffer;
	}
	return 0;
}

static void binder_transaction(struct binder_proc *proc /* 当前进程 */,
			       struct binder_thread *thread /* 当前处理器核心对应的 binder_thread */,
			       struct binder_transaction_data *tr, int reply /* false，暂时只考虑 request 的情况 */,
			       binder_size_t extra_buffers_size)
{
	int ret;
	struct binder_transaction *t;
	struct binder_work *w;
	struct binder_work *tcomplete;
	binder_size_t buffer_offset = 0;
	binder_size_t off_start_offset, off_end_offset;
	binder_size_t off_min;
	binder_size_t sg_buf_offset, sg_buf_end_offset;
	binder_size_t user_offset = 0;
	struct binder_proc *target_proc = NULL;
	struct binder_thread *target_thread = NULL;
	struct binder_node *target_node = NULL;
	struct binder_transaction *in_reply_to = NULL;
	struct binder_transaction_log_entry *e;
	uint32_t return_error = 0;
	uint32_t return_error_param = 0;
	uint32_t return_error_line = 0;
	binder_size_t last_fixup_obj_off = 0;
	binder_size_t last_fixup_min_off = 0;
	struct binder_context *context = proc->context;
	int t_debug_id = atomic_inc_return(&binder_last_id);
	char *secctx = NULL;
	u32 secctx_sz = 0;
	struct list_head sgc_head;
	struct list_head pf_head;
	const void __user *user_buffer = (const void __user *)
				(uintptr_t)tr->data.ptr.buffer;
	bool is_nested = false;
	INIT_LIST_HEAD(&sgc_head);
	INIT_LIST_HEAD(&pf_head);

	if (reply) {
		// ...
	} else {
		if (tr->target.handle) {
			struct binder_ref *ref;
			binder_proc_lock(proc);
			ref = binder_get_ref_olocked(proc, tr->target.handle,
						     true);
			if (ref) {
				target_node = binder_get_node_refs_for_txn(
						ref->node, &target_proc,
						&return_error);
			} else {
				binder_user_error("%d:%d got transaction to invalid handle\n",
						  proc->pid, thread->pid);
				return_error = BR_FAILED_REPLY;
			}
			binder_proc_unlock(proc);
		} else {
			mutex_lock(&context->context_mgr_node_lock);
			target_node = context->binder_context_mgr_node;
			// handle == 0 表示 server 是 servicemanager，这个在【深入 Binder 之 servicemanager 进程】有讲过
            // ...
		}
		e->to_node = target_node->debug_id;
		if (security_binder_transaction(proc->cred,
						target_proc->cred) < 0) {
			return_error = BR_FAILED_REPLY;
			return_error_param = -EPERM;
			return_error_line = __LINE__;
			goto err_invalid_target_handle;
		}
		binder_inner_proc_lock(proc);
		if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
			struct binder_transaction *tmp;

			tmp = thread->transaction_stack;
			if (tmp->to_thread != thread) {
				spin_lock(&tmp->lock);
				binder_user_error("%d:%d got new transaction with bad transaction stack, transaction %d has target %d:%d\n",
					proc->pid, thread->pid, tmp->debug_id,
					tmp->to_proc ? tmp->to_proc->pid : 0,
					tmp->to_thread ?
					tmp->to_thread->pid : 0);
				spin_unlock(&tmp->lock);
				binder_inner_proc_unlock(proc);
				return_error = BR_FAILED_REPLY;
				return_error_param = -EPROTO;
				return_error_line = __LINE__;
				goto err_bad_call_stack;
			}
			while (tmp) {
				struct binder_thread *from;

				spin_lock(&tmp->lock);
				from = tmp->from;
				if (from && from->proc == target_proc) {
					atomic_inc(&from->tmp_ref);
					target_thread = from;
					spin_unlock(&tmp->lock);
					break;
				}
				spin_unlock(&tmp->lock);
				tmp = tmp->from_parent;
			}
		}
		binder_inner_proc_unlock(proc);
	}

	/* TODO: reuse incoming transaction for reply */
	t = kzalloc(sizeof(*t), GFP_KERNEL);
	INIT_LIST_HEAD(&t->fd_fixups);
	binder_stats_created(BINDER_STAT_TRANSACTION);
	spin_lock_init(&t->lock);
	trace_android_vh_binder_transaction_init(t);

	tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
	binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);

	if (!reply && !(tr->flags & TF_ONE_WAY))
		t->from = thread;
	else
		t->from = NULL;
	t->sender_euid = task_euid(proc->tsk);
	t->to_proc = target_proc;
	t->to_thread = target_thread;
	t->code = tr->code;
	t->flags = tr->flags;
	t->is_nested = is_nested;
	if (!(t->flags & TF_ONE_WAY) &&
	    binder_supported_policy(current->policy)) {
		/* Inherit supported policies for synchronous transactions */
		t->priority.sched_policy = current->policy;
		t->priority.prio = current->normal_prio;
	} else {
		/* Otherwise, fall back to the default priority */
		t->priority = target_proc->default_priority;
	}

	if (target_node && target_node->txn_security_ctx) {
		u32 secid;
		size_t added_size;

		security_cred_getsecid(proc->cred, &secid);
		ret = security_secid_to_secctx(secid, &secctx, &secctx_sz);
		if (ret) {
			binder_txn_error("%d:%d failed to get security context\n",
				thread->pid, proc->pid);
			return_error = BR_FAILED_REPLY;
			return_error_param = ret;
			return_error_line = __LINE__;
			goto err_get_secctx_failed;
		}
		added_size = ALIGN(secctx_sz, sizeof(u64));
		extra_buffers_size += added_size;
		if (extra_buffers_size < added_size) {
			binder_txn_error("%d:%d integer overflow of extra_buffers_size\n",
				thread->pid, proc->pid);
			return_error = BR_FAILED_REPLY;
			return_error_param = -EINVAL;
			return_error_line = __LINE__;
			goto err_bad_extra_size;
		}
	}

	trace_binder_transaction(reply, t, target_node);

	t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size,
		tr->offsets_size, extra_buffers_size,
		!reply && (t->flags & TF_ONE_WAY), current->tgid);

	if (secctx) {
		int err;
		size_t buf_offset = ALIGN(tr->data_size, sizeof(void *)) +
				    ALIGN(tr->offsets_size, sizeof(void *)) +
				    ALIGN(extra_buffers_size, sizeof(void *)) -
				    ALIGN(secctx_sz, sizeof(u64));

		t->security_ctx = (uintptr_t)t->buffer->user_data + buf_offset;
		err = binder_alloc_copy_to_buffer(&target_proc->alloc,
						  t->buffer, buf_offset,
						  secctx, secctx_sz);
		if (err) {
			t->security_ctx = 0;
			WARN_ON(1);
		}
		security_release_secctx(secctx, secctx_sz);
		secctx = NULL;
	}
	t->buffer->debug_id = t->debug_id;
	t->buffer->transaction = t;
	t->buffer->target_node = target_node;
	t->buffer->clear_on_free = !!(t->flags & TF_CLEAR_BUF);
	trace_binder_transaction_alloc_buf(t->buffer);

	if (binder_alloc_copy_user_to_buffer(
				&target_proc->alloc,
				t->buffer,
				ALIGN(tr->data_size, sizeof(void *)),
				(const void __user *)
					(uintptr_t)tr->data.ptr.offsets,
				tr->offsets_size)) {
		binder_user_error("%d:%d got transaction with invalid offsets ptr\n",
				proc->pid, thread->pid);
		return_error = BR_FAILED_REPLY;
		return_error_param = -EFAULT;
		return_error_line = __LINE__;
		goto err_copy_data_failed;
	}
	if (!IS_ALIGNED(tr->offsets_size, sizeof(binder_size_t))) {
		binder_user_error("%d:%d got transaction with invalid offsets size, %lld\n",
				proc->pid, thread->pid, (u64)tr->offsets_size);
		return_error = BR_FAILED_REPLY;
		return_error_param = -EINVAL;
		return_error_line = __LINE__;
		goto err_bad_offset;
	}
	if (!IS_ALIGNED(extra_buffers_size, sizeof(u64))) {
		binder_user_error("%d:%d got transaction with unaligned buffers size, %lld\n",
				  proc->pid, thread->pid,
				  (u64)extra_buffers_size);
		return_error = BR_FAILED_REPLY;
		return_error_param = -EINVAL;
		return_error_line = __LINE__;
		goto err_bad_offset;
	}
	off_start_offset = ALIGN(tr->data_size, sizeof(void *));
	buffer_offset = off_start_offset;
	off_end_offset = off_start_offset + tr->offsets_size;
	sg_buf_offset = ALIGN(off_end_offset, sizeof(void *));
	sg_buf_end_offset = sg_buf_offset + extra_buffers_size -
		ALIGN(secctx_sz, sizeof(u64));
	off_min = 0;
	for (buffer_offset = off_start_offset; buffer_offset < off_end_offset;
	     buffer_offset += sizeof(binder_size_t)) {
		struct binder_object_header *hdr;
		size_t object_size;
		struct binder_object object;
		binder_size_t object_offset;
		binder_size_t copy_size;

		if (binder_alloc_copy_from_buffer(&target_proc->alloc,
						  &object_offset,
						  t->buffer,
						  buffer_offset,
						  sizeof(object_offset))) {
			// error...
		}

		/*
		 * Copy the source user buffer up to the next object
		 * that will be processed.
		 */
		copy_size = object_offset - user_offset;
		if (copy_size && (user_offset > object_offset ||
				binder_alloc_copy_user_to_buffer(
					&target_proc->alloc,
					t->buffer, user_offset,
					user_buffer + user_offset,
					copy_size))) {
			// error...
		}
		object_size = binder_get_object(target_proc, user_buffer,
				t->buffer, object_offset, &object);
                
		/*
		 * Set offset to the next buffer fragment to be
		 * copied
		 */
		user_offset = object_offset + object_size;

		hdr = &object.hdr;
		off_min = object_offset + object_size;
		switch (hdr->type) {
		case BINDER_TYPE_BINDER:
		case BINDER_TYPE_HANDLE:
		case BINDER_TYPE_FD:
		case BINDER_TYPE_PTR:
        // ...
		default:
			binder_user_error("%d:%d got transaction with invalid object type, %x\n",
				proc->pid, thread->pid, hdr->type);
			return_error = BR_FAILED_REPLY;
			return_error_param = -EINVAL;
			return_error_line = __LINE__;
			goto err_bad_object_type;
		}
	}

	/* Done processing objects, copy the rest of the buffer */
	if (binder_alloc_copy_user_to_buffer(
				&target_proc->alloc,
				t->buffer, user_offset,
				user_buffer + user_offset,
				tr->data_size - user_offset)) {
		// error...
	}

	if (reply) {
		// ...
	} else if (!(t->flags & TF_ONE_WAY)) {
		BUG_ON(t->buffer->async_transaction != 0);
		binder_inner_proc_lock(proc);
		/*
		 * Defer the TRANSACTION_COMPLETE, so we don't return to
		 * userspace immediately; this allows the target process to
		 * immediately start processing this transaction, reducing
		 * latency. We will then return the TRANSACTION_COMPLETE when
		 * the target replies (or there is an error).
		 */
		binder_enqueue_deferred_thread_work_ilocked(thread, tcomplete);
		t->need_reply = 1;
		t->from_parent = thread->transaction_stack;
		thread->transaction_stack = t;
		binder_inner_proc_unlock(proc);
		return_error = binder_proc_transaction(t,
				target_proc, target_thread);
		}
	} // ...
	return;

err_dead_proc_or_thread:
err_translate_failed:
err_bad_object_type:
err_bad_offset:
err_bad_parent:
err_copy_data_failed:
// errors ...
}

/**
 * binder_proc_transaction() - sends a transaction to a process and wakes it up
 * @t:		transaction to send
 * @proc:	process to send the transaction to
 * @thread:	thread in @proc to send the transaction to (may be NULL)
 *
 * This function queues a transaction to the specified process. It will try
 * to find a thread in the target process to handle the transaction and
 * wake it up. If no thread is found, the work is queued to the proc
 * waitqueue.
 *
 * If the @thread parameter is not NULL, the transaction is always queued
 * to the waitlist of that specific thread.
 *
 * Return:	0 if the transaction was successfully queued
 *		BR_DEAD_REPLY if the target process or thread is dead
 *		BR_FROZEN_REPLY if the target process or thread is frozen
 */
static int binder_proc_transaction(struct binder_transaction *t,
				    struct binder_proc *proc,
				    struct binder_thread *thread)
{
	struct binder_node *node = t->buffer->target_node;
	struct binder_priority node_prio;
	bool oneway = !!(t->flags & TF_ONE_WAY);
	bool pending_async = false;

	BUG_ON(!node);
	binder_node_lock(node);
	node_prio.prio = node->min_priority;
	node_prio.sched_policy = node->sched_policy;

	if (oneway) {
		BUG_ON(thread);
		if (node->has_async_transaction) {
			pending_async = true;
		} else {
			node->has_async_transaction = true;
		}
	}

	binder_inner_proc_lock(proc);
	if (proc->is_frozen) {
		proc->sync_recv |= !oneway;
		proc->async_recv |= oneway;
	}

	if ((proc->is_frozen && !oneway) || proc->is_dead ||
			(thread && thread->is_dead)) {
		bool proc_is_dead = proc->is_dead
			|| (thread && thread->is_dead);
		binder_inner_proc_unlock(proc);
		binder_node_unlock(node);
		return proc_is_dead ? BR_DEAD_REPLY : BR_FROZEN_REPLY;
	}

	if (!thread && !pending_async)
		thread = binder_select_thread_ilocked(proc);

	if (thread) {
		binder_transaction_priority(thread->task, t, node_prio,
					    node->inherit_rt);
		binder_enqueue_thread_work_ilocked(thread, &t->work);
	} else if (!pending_async) {
		binder_enqueue_work_ilocked(&t->work, &proc->todo);
	} else {
		binder_enqueue_work_ilocked(&t->work, &node->async_todo);
	}

	if (!pending_async)
		binder_wakeup_thread_ilocked(proc, thread, !oneway /* sync */);

	proc->outstanding_txns++;
	binder_inner_proc_unlock(proc);
	binder_node_unlock(node);

	return 0;
}

/**
 * binder_wakeup_thread_ilocked() - wakes up a thread for doing proc work.
 * @proc:	process to wake up a thread in
 * @thread:	specific thread to wake-up (may be NULL)
 * @sync:	whether to do a synchronous wake-up
 *
 * This function wakes up a thread in the @proc process.
 * The caller may provide a specific thread to wake-up in
 * the @thread parameter. If @thread is NULL, this function
 * will wake up threads that have called poll().
 *
 * Note that for this function to work as expected, callers
 * should first call binder_select_thread() to find a thread
 * to handle the work (if they don't have a thread already),
 * and pass the result into the @thread parameter.
 */
static void binder_wakeup_thread_ilocked(struct binder_proc *proc,
					 struct binder_thread *thread,
					 bool sync)
{
	assert_spin_locked(&proc->inner_lock);

	if (thread) {
		if (sync)
			wake_up_interruptible_sync(&thread->wait); // server thread 在这里 epoll
		else
			wake_up_interruptible(&thread->wait);
		return;
	}

	/* Didn't find a thread waiting for proc work; this can happen
	 * in two scenarios:
	 * 1. All threads are busy handling transactions
	 *    In that case, one of those threads should call back into
	 *    the kernel driver soon and pick up this work.
	 * 2. Threads are using the (e)poll interface, in which case
	 *    they may be blocked on the waitqueue without having been
	 *    added to waiting_threads. For this case, we just iterate
	 *    over all threads not handling transaction work, and
	 *    wake them all up. We wake all because we don't know whether
	 *    a thread that called into (e)poll is handling non-binder
	 *    work currently.
	 */
	binder_wakeup_poll_threads_ilocked(proc, sync);
}
```

# 服务端



```cpp
// system_server 进程是由 zygote 进程 fork 出来的（其实所有的 app 进程都是由 zygote 进程 fork 出来的）
// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/cmds/app_process/app_main.cpp
int main(int argc, char* const argv[])
{
    // ...
    bool zygote = false;
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
    // ...
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);  // 启动虚拟机，并调用 ZygoteInit.main(String argv[])
    } else if // ...
}

/**
 * 主要做四件事：
 * 
 * 1、注册 socket
 * 2、预加载资源
 * 3、启动 system_server
 * 4、启动消息循环
 * 
 * This is the entry point for a Zygote process.  It creates the Zygote server, loads resources,
 * and handles other tasks related to preparing the process for forking into applications
 */
// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public static void main(String[] argv) {
    // ...
    boolean startSystemServer = false;
    for (int i = 1; i < argv.length; i++) {
        if ("start-system-server".equals(argv[i])) {
            startSystemServer = true;
        }
    // ...
    if (startSystemServer) {
        Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
        // ...
}
// fork 出 system_server 进程并进入其逻辑
private static Runnable forkSystemServer(String abiList, String socketName,
        ZygoteServer zygoteServer) {
    // ...
    /* Request to fork the system server process */
    pid = Zygote.forkSystemServer(
            parsedArgs.mUid, parsedArgs.mGid,
            parsedArgs.mGids,
            parsedArgs.mRuntimeFlags,
            null,
            parsedArgs.mPermittedCapabilities,
            parsedArgs.mEffectiveCapabilities);

    /* For child process */
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }
    return null;
}
private static Runnable handleSystemServerProcess(ZygoteArguments parsedArgs) {
        // ...
        /*
         * Pass the remaining arguments to SystemServer.
         */
        return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                parsedArgs.mDisabledCompatChanges,
                parsedArgs.mRemainingArgs, cl);
    }
    /* should never reach here */
}
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

// 开启一个子线程执行 binder 消息通讯的 loop：IPCThreadState::self()->joinThreadPool = talkWithDriver + executeCommand
// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        if (mMaxThreads == 0) {
            ALOGW("Extra binder thread started, but 0 threads requested. Do not use "
                  "*startThreadPool when zero threads are requested.");
        }
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}
void ProcessState::spawnPooledThread(bool isMain /* true */)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
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
        IPCThreadState::self()->joinThreadPool(mIsMain /* true */);
        return false;
    }
};
void IPCThreadState::joinThreadPool(bool isMain)
{
    // mOut 是给 binder driver 读取的区域，此时是 BC_ENTER_LOOPER
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    mIsLooper = true;
    status_t result;
    do {
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();
        // ...
    } while (result != -ECONNREFUSED && result != -EBADF);
    // ...
}
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;
    result = talkWithDriver();
    if (result >= NO_ERROR) {
        // ...
        cmd = mIn.readInt32();
        result = executeCommand(cmd);
        // ...
}
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    switch ((uint32_t)cmd) {
    case BR_ERROR:
    case BR_OK:
    case BR_TRANSACTION_SEC_CTX:
    case BR_TRANSACTION:
        {
            binder_transaction_data_secctx tr_secctx;
            binder_transaction_data& tr = tr_secctx.transaction_data;

            if (cmd == BR_TRANSACTION_SEC_CTX) {
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
                tr.offsets_size/sizeof(binder_size_t), freeBuffer, this);

            const void* origServingStackPointer = mServingStackPointer;
            mServingStackPointer = &origServingStackPointer; // anything on the stack

            const pid_t origPid = mCallingPid;
            const char* origSid = mCallingSid;
            const uid_t origUid = mCallingUid;
            const int32_t origStrictModePolicy = mStrictModePolicy;
            const int32_t origTransactionBinderFlags = mLastTransactionBinderFlags;

            mCallingPid = tr.sender_pid;
            mCallingSid = reinterpret_cast<const char*>(tr_secctx.secctx);
            mCallingUid = tr.sender_euid;
            mLastTransactionBinderFlags = tr.flags;

            Parcel reply;
            status_t error;
            bool reply_sent = false;
            constexpr size_t kForwardReplyFlags = TF_CLEAR_BUF;
            auto reply_callback = [&] (auto &replyParcel) {
                if (reply_sent) {
                    // Reply was sent earlier, ignore it.
                    ALOGE("Dropping binder reply, it was sent already.");
                    return;
                }
                reply_sent = true;
                if ((tr.flags & TF_ONE_WAY) == 0) {
                    replyParcel.setError(NO_ERROR);
                    sendReply(replyParcel, (tr.flags & kForwardReplyFlags));
                } else {
                    ALOGE("Not sending reply in one-way transaction");
                }
            };

            if (tr.target.ptr) {
                // We only have a weak reference on the target object, so we must first try to
                // safely acquire a strong reference before doing anything else with it.
                if (reinterpret_cast<RefBase::weakref_type*>(
                        tr.target.ptr)->attemptIncStrong(this)) {
                    error = reinterpret_cast<BHwBinder*>(tr.cookie)->transact(tr.code, buffer,
                            &reply, tr.flags, reply_callback);
                    reinterpret_cast<BHwBinder*>(tr.cookie)->decStrong(this);
                } else {
                    error = UNKNOWN_TRANSACTION;
                }

            } else {
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags, reply_callback);
            }

            if ((tr.flags & TF_ONE_WAY) == 0) {
                if (!reply_sent) {
                    // Should have been a reply but there wasn't, so there
                    // must have been an error instead.
                    reply.setError(error);
                    sendReply(reply, (tr.flags & kForwardReplyFlags));
                } else {
                    if (error != NO_ERROR) {
                        ALOGE("transact() returned error after sending reply.");
                    } else {
                        // Ok, reply sent and transact didn't return an error.
                    }
                }
            } else {
                // One-way transaction, don't care about return value or reply.
            }
            // ...
}
```

## AMS

上接 system_server 的启动流程

```cpp

// The main function called when started through the zygote process. This could be unified with
// main(), if the native code in nativeFinishInit() were rationalized with Zygote startup.
// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public static Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    // ...
    ZygoteInit.nativeZygoteInit();
    // ZygoteInit.forkSystemServer() 函数里写死的启动 system_server 进程的参数，argv 应该是 com.android.server.SystemServer
    return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
            classLoader);
}

// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    // Remaining arguments are passed to the start class's static main
    // 执行 SystemServer.main(String[] args)
    return findStaticMain(args.startClass /* com.android.server.SystemServer */, args.startArgs, classLoader);
}

// The main entry point from zygote.
// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/java/com/android/server/SystemServer.java
public static void main(String[] args) {
    new SystemServer().run();
}    
private void run() {
    // ...
    // Start services.
    try {
        t.traceBegin("StartServices");
        startBootstrapServices(t);
        startCoreServices(t);
        startOtherServices(t);
        startApexServices(t);
    }
    // ...
    // Loop forever.
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}    
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
    // ...
    // Activity manager runs the show.
    t.traceBegin("StartActivityManager");
    // TODO: Might need to move after migration to WM.
    ActivityTaskManagerService atm = mSystemServiceManager.startService(
            ActivityTaskManagerService.Lifecycle.class).getService();
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
            mSystemServiceManager, atm);
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    mWindowManagerGlobalLock = atm.getGlobalLock();
    t.traceEnd();
    // ...
    // Set up the Application instance for the system process and get started.
    t.traceBegin("SetSystemProcess");
    mActivityManagerService.setSystemProcess();
    t.traceEnd();
    // ...
}

// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public void setSystemProcess() {
    ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                DUMP_FLAG_PRIORITY_HIGH);
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        ServiceManager.addService("dbinfo", new DbBinder(this));
        mAppProfiler.setCpuInfoService();
        ServiceManager.addService("permission", new PermissionController(this));
        ServiceManager.addService("processinfo", new ProcessInfoService(this));
        ServiceManager.addService("cacheinfo", new CacheBinder(this));
        // ...
}

// 此方法在【深入 Binder 之 servicemanager 进程】中介绍过
// getIServiceManager() 实际上是返回一个 handle == 0 的 BpBinder(native) / BinderProxy(java)
// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java
public static void addService(String name, IBinder service, boolean allowIsolated,
        int dumpPriority) {
    try {
        getIServiceManager().addService(name, service /* ActivityManagerService */, allowIsolated, dumpPriority);
    } catch (RemoteException e) {
        Log.e(TAG, "error in addService", e);
    }
}

// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java
public final void writeStrongBinder(IBinder val) {
    nativeWriteStrongBinder(mNativePtr, val);
}
// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_os_Parcel.cpp
static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        const status_t err = parcel->writeStrongBinder(ibinderForJavaObject(env, object));
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}

// ActivityManagerService 继承自 IActivityManager.Stub 它又继承自 Binder
// server 端继承自 Binder，client 端是 BinderProxy
// https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp
sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj == NULL) return NULL;

    // Instance of Binder?
    if (env->IsInstanceOf(obj, gBinderOffsets.mClass)) {
        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env->GetLongField(obj, gBinderOffsets.mObject);  // Binder.mObject
        return jbh->get(env, obj);
    }

    // Instance of BinderProxy?
    if (env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
        return getBPNativeData(env, obj)->mObject;
    }

    ALOGW("ibinderForJavaObject: %p is not a Binder object", obj);
    return NULL;
}

// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flattenBinder(val);
}

status_t Parcel::flattenBinder(const sp<IBinder>& binder) {
    BBinder* local = nullptr;
    if (binder) local = binder->localBinder();
    // ...
    if (!local) {  // 比如 BpBinder 它只有 server handle
        const int32_t handle = proxy ? proxy->getPrivateAccessor().binderHandle() : 0;
        obj.hdr.type = BINDER_TYPE_HANDLE;
        obj.binder = 0; /* Don't pass uninitialized stack data to a remote process */
        obj.flags = 0;
        obj.handle = handle;
        obj.cookie = 0;
    } else {

        // 比如 BBinder
        // java Binder.mObject 是 native JavaBBinderHolder，它持有 JavaBBinder，而 JavaBBinder 继承自 BBinder
        // ActivityManagerService 继承自 IActivityManager.Stub，它又继承自 Binder
        obj.hdr.type = BINDER_TYPE_BINDER;
        obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
        obj.cookie = reinterpret_cast<uintptr_t>(local);  // cookie 是 IBinder 在 server/client 进程内的内存地址
    }
    status_t status = writeObject(obj, false);
    // ...
}

// BpBinder 通过 IPCThreadState::self()->transact(...) 与 binder driver 通讯
// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // ...
    status_t status;
    if (CC_UNLIKELY(isRpcBinder())) {
        status = rpcSession()->transact(sp<IBinder>::fromExisting(this), code, data, reply, flags);
    } else {
        status = IPCThreadState::self()->transact(binderHandle(), code, data, reply, flags);
    }
    // ...
}

// binder driver 遇到 handle == 0 会将 request 交由 service_manager 进程处理，本质是一个 map：string -> IBinder
// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/ServiceManager.cpp
ServiceManager::addService(...) {...}
```

## server handle request



```cpp
// 开启一个子线程执行 binder 消息通讯的 loop：IPCThreadState::self()->joinThreadPool = talkWithDriver + executeCommand
// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp
void ProcessState::startThreadPool()

// 在当前线程进行 binder 消息通讯 loop
void IPCThreadState::joinThreadPool(bool isMain)


```


# 参考

- [掌握 binder 机制？驱动核心源码详解](https://zhuanlan.zhihu.com/p/381310378)
- [Binder（四）system_server中binder的初始化](https://blog.csdn.net/Mumuuuuuuu/article/details/122108510)