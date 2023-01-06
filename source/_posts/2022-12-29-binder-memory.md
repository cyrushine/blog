---
title: Binder IPC 内存模型
date: 2022-12-29 12:00:00 +0800
---

# 指导思想

进程 A 通过 `binder_transaction` 往进程 B 传输数据的过程，其中步骤 4、5、6 和 7 是一页一页地循环执行:

1. 准备需要发送的数据, 数据存放在进程 A 的用户空间中

2. 进程 A 通过系统调用进入 binder driver 的代码逻辑中

3. 在 binder driver 代码中, 找到与所发送数据大小匹配的进程 B 的 `binder_buffer`, 该 buffer 此时只有虚拟地址尚未映射物理页（指向进程 B 的用户空间）

4. 为此次数据传输分配物理页

5. 建立进程 B 的 binder buffer 里虚拟页与刚分配物理页的映射关系

6. 通过 `kmap` 为物理页映射一个内核空间的虚拟页

7. `copy_ from_user` 将进程 A 用户空间的数据写入刚刚映射的内核空间虚拟页中

8. 根据映射关系, 刚刚写入的数据现在可以通过进程 B 的 binder buffer 的虚拟地址读出

## 一次拷贝

传统 IPC 一般是进程 A 执行系统调用陷入内核，将数据从进程 A 内存空间拷贝至内核，进程 B 为了读取数据需要执行系统调用陷入内核，内核将数据拷贝至进程 B 内存空间，那么一共是两次内存拷贝和两次用户态内核态切换

Binder IPC 的第一次内存拷贝同传统 IPC，也是将数据从从进程 A 内存空间拷贝至内核，但 Binder IPC 不需要第二次内存拷贝，因为它将进程 B 内存空间里指向数据的虚存映射至内核里保存数据的内核物理内存，这里进程 B 读取到的数据其实是第一次拷贝至内核的那份数据，所以是一次内存拷贝和两次用户态内核态切换

注意这里说的只需 *一次拷贝* 指的是一次 Binder IPC，而一次 Binder 方法调用也即 Binder RPC 需要两次 IPC：发送请求和接收响应，所以一次 Binder RPC 其实包含了两次 IPC 也即两次内存拷贝（四次用户态内核态的切换）

# client write request

```cpp
// https://cs.android.com/android/platform/superproject/+/master:system/libhwbinder/include/hwbinder/IPCThreadState.h
class IPCThreadState
{
           private:
            Parcel              mIn;   // binder driver 写入，应用进程读取并处理的内存区域，对应 binder_thread_read
            Parcel              mOut;  // 应用进程写入，binder driver 读取并处理的内存区域，对应 binder_thread_write
};

// http://www.aospxref.com/android-13.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data /* 参数被打包进这里 */,
                                  Parcel* reply /* 用以接收响应数据 */, uint32_t flags)
{
    // 将 [BC_TRANSACTION][binder_transaction_data] 写入 mOut
    // mOut 是 client 进程内的一块内存区域
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);
    // 将 request 发送给 binder driver 并等待 response
    err = waitForResponse(reply);
    // ...
    return err;
}

status_t IPCThreadState::writeTransactionData(int32_t cmd /* BC_TRANSACTION */, uint32_t binderFlags,
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
        tr.data_size = data.ipcDataSize();      // 存放参数的内存区域大小
        tr.data.ptr.buffer = data.ipcData();    // 其实就是 Parcel 存放数据的内存块的起始地址
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

    // mOut 写入 [BC_TRANSACTION][binder_transaction_data]
	// 对应章节【指导思想】里的步骤一
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}

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

    bwr.write_size = outAvail;                    // mOut 内存区域大小，也即参数包大小
    bwr.write_buffer = (uintptr_t)mOut.data();    // mOut 地址，存放打包好的请求参数

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
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)  // 对应章节【指导思想】里的步骤二
            err = NO_ERROR;
        else
            err = -errno;
        if (mProcess->mDriverFD < 0) {
            err = -EBADF;
        }
    } while (err == -EINTR);
    // ...
}

// https://cs.android.com/android/kernel/superproject/+/common-android-mainline:common/drivers/android/binder.c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {...}

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

    // 将 binder_write_read 从用户空间拷贝一份至内核空间
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}

    // 先处理应用进程的请求 write_buffer
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

            // 将 binder_transaction_data 从用户空间拷贝至内核空间
			if (copy_from_user(&tr, ptr, sizeof(tr)))
				return -EFAULT;
			ptr += sizeof(tr);
			binder_transaction(proc, thread, &tr,
					   cmd == BC_REPLY, 0);
			break;
		}
        // ...
}

static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply,
			       binder_size_t extra_buffers_size)
{
	struct binder_transaction *t;
	const void __user *user_buffer = (const void __user *)
				(uintptr_t)tr->data.ptr.buffer;    // 用户空间的数据块地址
	// ...

    // 从进程 B 的用以 binder 数据传输的虚存里找一块满足大小的虚存块 binder_buffer.user_data 用以存放 request
	// 这里就会发生物理页的分配了，binder 在内核分配物理页，并将进程 B 的虚存会映射至内核物理页
	// 这样进程 B 就可以直接读取 request 而无需进行一次内核态至用户态的内存拷贝
	// 参看章节【映射进程虚存与内核内存】
	// 这里对应章节【指导思想】里的步骤三、四、五和六
	t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size,
		tr->offsets_size, extra_buffers_size,
		!reply && (t->flags & TF_ONE_WAY), current->tgid);
	// ...

    // 将 request 从进程 A （tr->data.ptr.offsets）拷贝至进程 B（t->buffer->user_data）
	// 这里就是 binder ipc 中发生的唯一一次数据拷贝
	// 因为进程 B 的虚存地址实际上是映射至内核物理内存，所以 request 实际上是从进程 A 拷贝至内核，进程 B 读取的是内核物理内存空间
	// 对应章节【指导思想】里的步骤七
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
	// ...
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
			binder_txn_error("%d:%d copy offset from buffer failed\n",
				thread->pid, proc->pid);
			return_error = BR_FAILED_REPLY;
			return_error_param = -EINVAL;
			return_error_line = __LINE__;
			goto err_bad_offset;
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
			binder_user_error("%d:%d got transaction with invalid data ptr\n",
					proc->pid, thread->pid);
			return_error = BR_FAILED_REPLY;
			return_error_param = -EFAULT;
			return_error_line = __LINE__;
			goto err_copy_data_failed;
		}
		object_size = binder_get_object(target_proc, user_buffer,
				t->buffer, object_offset, &object);
		if (object_size == 0 || object_offset < off_min) {
			binder_user_error("%d:%d got transaction with invalid offset (%lld, min %lld max %lld) or object.\n",
					  proc->pid, thread->pid,
					  (u64)object_offset,
					  (u64)off_min,
					  (u64)t->buffer->data_size);
			return_error = BR_FAILED_REPLY;
			return_error_param = -EINVAL;
			return_error_line = __LINE__;
			goto err_bad_offset;
		}
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
		}
	}
	/* Done processing objects, copy the rest of the buffer */
	if (binder_alloc_copy_user_to_buffer(
				&target_proc->alloc,
				t->buffer, user_offset,
				user_buffer + user_offset,
				tr->data_size - user_offset)) {
		binder_user_error("%d:%d got transaction with invalid data ptr\n",
				proc->pid, thread->pid);
		return_error = BR_FAILED_REPLY;
		return_error_param = -EFAULT;
		return_error_line = __LINE__;
		goto err_copy_data_failed;
	}
	// ...
}
```

# server read request

在上一章节 [client write request](#client-write-request) 里，binder 已经把 request 从 client 进程拷贝至内核，并将 server 进程内的一块虚存映射至内核里的 request，下面看看 server 收到了什么、如何处理 request 数据

```cpp
// 在【Binder IPC 线程调度模型】里讲过：
// server binder 线程先往 binder 发送数据（binder_thread_write），如果有的话
// 然后读取 binder 发送过来的数据（binder_thread_read），如果没有数据需要处理则 binder 线程会阻塞在 binder_wait_for_work（让渡 CPU）
// 那么 binder 将 request（BINDER_WORK_TRANSACTION） 发送至 server 进程的 todo list，server binder 线程将被唤醒
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, /* 应用进程内存地址，实际上是 IPCThreadState->mIn 这块内存空间，看 talkWithDriver */
				  size_t size,                    /* IPCThreadState->mIn 的大小 */
			      binder_size_t *consumed         /* 输出，表示往 binder_buffer 里写入了多少内容 */, 
				  int non_block)
{
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;  // 应用进程的 IPCThreadState->mIn 这块内存空间
	void __user *ptr = buffer + *consumed;                          // 应用进程的 IPCThreadState->mIn 这块内存空间
	void __user *end = buffer + size;

	int ret = 0;
	int wait_for_proc_work;

	if (*consumed == 0) {
		if (put_user(BR_NOOP, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
	}

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

	if (non_block) {
		if (!binder_has_work(thread, wait_for_proc_work))
			ret = -EAGAIN;
	} else {
		ret = binder_wait_for_work(thread, wait_for_proc_work);  // binder thread 将在这里被唤醒并继续
	}

	thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

	if (ret)
		return ret;

	while (1) {
		// ...
		struct binder_transaction_data_secctx tr;
		struct binder_transaction_data *trd = &tr.transaction_data;
		struct binder_transaction *t = NULL;
		struct binder_work *w = NULL;
		size_t trsize = sizeof(*trd);

		w = binder_dequeue_work_head_ilocked(list);
		switch (w->type) {
		case BINDER_WORK_TRANSACTION: {    // 收到 binder 给的 request
			binder_inner_proc_unlock(proc);
			t = container_of(w, struct binder_transaction, work);
		} break;
		// ...

		trd->data_size = t->buffer->data_size;
		trd->offsets_size = t->buffer->offsets_size;
		// 在章节【client write request】里的 binder_transaction 里说过
		// binder_transaction->binder_buffer 是从进程 B 的空闲虚存空间里 binder_alloc->free_buffers 挑选的满足大小的一块虚存
		// 也即 binder_transaction->binder_buffer->user_data 指向 server 进程内的虚存
		// 而这块虚存映射到内核内存空间里的物理内存（request 拷贝了一次到内核）
		trd->data.ptr.buffer = (uintptr_t)t->buffer->user_data;
		trd->data.ptr.offsets = trd->data.ptr.buffer +
					ALIGN(t->buffer->data_size,
					    sizeof(void *));

		tr.secctx = t->security_ctx;
		if (t->security_ctx) {
			cmd = BR_TRANSACTION_SEC_CTX;
			trsize = sizeof(tr);
		}

		// 往应用进程内存地址 buffer 写入 BR_TRANSACTION，实际上是应用进程的 IPCThreadState->mIn 这块内存空间
		if (put_user(cmd, (uint32_t __user *)ptr)) {
			if (t_from)
				binder_thread_dec_tmpref(t_from);

			binder_cleanup_transaction(t, "put_user failed",
						   BR_FAILED_REPLY);

			return -EFAULT;
		}
		ptr += sizeof(uint32_t);

		// 往应用进程内存地址 buffer 写入 binder_transaction_data
		if (copy_to_user(ptr, &tr, trsize)) {
			if (t_from)
				binder_thread_dec_tmpref(t_from);

			binder_cleanup_transaction(t, "copy_to_user failed",
						   BR_FAILED_REPLY);

			return -EFAULT;
		}
		// ...
}

status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;  // binder 给过来的数据被写入至 IPCThreadState->mIn
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        switch (cmd) {
        // ...
        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
        logExtendedError();
    }

    return err;
}

status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
    // ...
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

            // tr.data.ptr.buffer 是此应用进程的虚存地址，实际映射至内核物理内存里保存的 request 拷贝
			// 用 Parcel 包装 tr.data.ptr.buffer 这块地址，然后交由 binder server impl 按照 aidl 协议逐个读取参数
            Parcel buffer;
            buffer.ipcSetDataReference(
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(binder_size_t), freeBuffer);
            // ...

			// 还记得 BBinder 吗？赶紧回顾下【Binder IPC 过程中常用的对象和类】
			Parcel reply;
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

            // 如果不是 one way 还需要将 response/reply 发送给 client
            if ((tr.flags & TF_ONE_WAY) == 0) {
                LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                if (error < NO_ERROR) reply.setError(error);

                // b/238777741: clear buffer before we send the reply.
                // Otherwise, there is a race where the client may
                // receive the reply and send another transaction
                // here and the space used by this transaction won't
                // be freed for the client.
                buffer.setDataSize(0);

                constexpr uint32_t kForwardReplyFlags = TF_CLEAR_BUF;
                sendReply(reply, (tr.flags & kForwardReplyFlags));
            } else { //...
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

    // binder_write_read.read_buffer 实际上指向 mIn
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

# 映射进程虚存与内核内存

`binder_alloc_new_buf` 寻找一块满足大小的进程虚存 `binder_buffer.user_data`，这块虚存是在进程在 mmap binder fd 时分配的，但当时并未给这块虚存映射物理页，参看 [binder_mmap](#binder_mmap)

同时会将这块虚存[映射至内核物理内存](#分配内核物理页)，将 request 拷贝至这里，然后 server 可以直接读取而不用将 request 拷贝至进程 B

```cpp
/**
 * https://cs.android.com/android/kernel/superproject/+/common-android-mainline:common/drivers/android/binder_alloc.c
 * 
 * binder_alloc_new_buf() - Allocate a new binder buffer
 * @alloc:              binder_alloc for this proc
 * @data_size:          size of user data buffer
 * @offsets_size:       user specified buffer offset
 * @extra_buffers_size: size of extra space for meta-data (eg, security context)
 * @is_async:           buffer for async transaction
 * @pid:				pid to attribute allocation to (used for debugging)
 *
 * Allocate a new buffer given the requested sizes. Returns
 * the kernel version of the buffer pointer. The size allocated
 * is the sum of the three given sizes (each rounded up to
 * pointer-sized boundary)
 *
 * Return:	The allocated buffer or %NULL if error
 */
struct binder_buffer *binder_alloc_new_buf(struct binder_alloc *alloc,
					   size_t data_size,
					   size_t offsets_size,
					   size_t extra_buffers_size,
					   int is_async,
					   int pid)
{
	struct binder_buffer *buffer;

	mutex_lock(&alloc->mutex);
	buffer = binder_alloc_new_buf_locked(alloc, data_size, offsets_size,
					     extra_buffers_size, is_async, pid);
	mutex_unlock(&alloc->mutex);
	return buffer;
}

static struct binder_buffer *binder_alloc_new_buf_locked(
				struct binder_alloc *alloc,
				size_t data_size,
				size_t offsets_size,
				size_t extra_buffers_size,
				int is_async,
				int pid)
{
	struct rb_node *n = alloc->free_buffers.rb_node;
	struct binder_buffer *buffer;
	size_t buffer_size;
	struct rb_node *best_fit = NULL;
	void __user *has_page_addr;
	void __user *end_page_addr;
	size_t size, data_offsets_size;
	int ret;

	mmap_read_lock(alloc->mm);
	if (!binder_alloc_get_vma(alloc)) {
		mmap_read_unlock(alloc->mm);
		binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
				   "%d: binder_alloc_buf, no vma\n",
				   alloc->pid);
		return ERR_PTR(-ESRCH);
	}
	mmap_read_unlock(alloc->mm);

	data_offsets_size = ALIGN(data_size, sizeof(void *)) +
		ALIGN(offsets_size, sizeof(void *));

	if (data_offsets_size < data_size || data_offsets_size < offsets_size) {
		binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
				"%d: got transaction with invalid size %zd-%zd\n",
				alloc->pid, data_size, offsets_size);
		return ERR_PTR(-EINVAL);
	}
	size = data_offsets_size + ALIGN(extra_buffers_size, sizeof(void *));
	if (size < data_offsets_size || size < extra_buffers_size) {
		binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
				"%d: got transaction with invalid extra_buffers_size %zd\n",
				alloc->pid, extra_buffers_size);
		return ERR_PTR(-EINVAL);
	}
	if (is_async &&
	    alloc->free_async_space < size + sizeof(struct binder_buffer)) {
		binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
			     "%d: binder_alloc_buf size %zd failed, no async space left\n",
			      alloc->pid, size);
		return ERR_PTR(-ENOSPC);
	}

	/* Pad 0-size buffers so they get assigned unique addresses */
	size = max(size, sizeof(void *));

    // 对应章节【指导思想】里的步骤三
    // 从目标进程（server）的空闲虚拟地址区间集合里（binder_alloc->free_buffers）找一块满足大小的（>= size）binder_buffer 出来
	// 
	// 空闲虚拟地址集合 binder_alloc->free_buffers 是一棵红黑树（平衡的二叉树），左孩子的地址空间比父亲小，右孩子比父亲大
    // struct rb_node {
    // 	unsigned long  __rb_parent_color;
    // 	struct rb_node *rb_right;
    // 	struct rb_node *rb_left;
    // } __attribute__((aligned(sizeof(long))));
	// 
	// 搜索二叉树，找到最匹配 size 大小的 buffer
	// 从章节【binder_mmap】可知 binder_alloc->free_buffers 里默认会初始化一个 1M 大小的应用进程可用虚存
	while (n) {
		buffer = rb_entry(n, struct binder_buffer, rb_node);  // 从 rb_node 取出 binder_buffer 类型的 entry
		BUG_ON(!buffer->free);
		buffer_size = binder_alloc_buffer_size(alloc, buffer);

		if (size < buffer_size) {  // 如果 buffer 大小满足 size 要求则选中（best_fit）然后继续往左下走寻找更接近 size 的 buffer
			best_fit = n;
			n = n->rb_left;
		} else if (size > buffer_size)  // buffer 大小不满足 size 要求，得继续往右下走找更大的 buffer
			n = n->rb_right;
		else {                          // buffer 大小刚好匹配 size，完美，无需继续寻找了
			best_fit = n;
			break;
		}
	}

	// 没有满足 size 大小要求的空闲 binder_buffer 则返回错误码
	// 此时说明 request or reponse 大小超过为 binder transaction 分配的虚存大小，会报以下错误：
	// JavaBinder: !!! FAILED BINDER TRANSACTION !!! (parcel size = 2001452)
	if (best_fit == NULL) {
		size_t allocated_buffers = 0;
		size_t largest_alloc_size = 0;
		size_t total_alloc_size = 0;
		size_t free_buffers = 0;
		size_t largest_free_size = 0;
		size_t total_free_size = 0;

		for (n = rb_first(&alloc->allocated_buffers); n != NULL;
		     n = rb_next(n)) {
			buffer = rb_entry(n, struct binder_buffer, rb_node);
			buffer_size = binder_alloc_buffer_size(alloc, buffer);
			allocated_buffers++;
			total_alloc_size += buffer_size;
			if (buffer_size > largest_alloc_size)
				largest_alloc_size = buffer_size;
		}
		for (n = rb_first(&alloc->free_buffers); n != NULL;
		     n = rb_next(n)) {
			buffer = rb_entry(n, struct binder_buffer, rb_node);
			buffer_size = binder_alloc_buffer_size(alloc, buffer);
			free_buffers++;
			total_free_size += buffer_size;
			if (buffer_size > largest_free_size)
				largest_free_size = buffer_size;
		}
		binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
				   "%d: binder_alloc_buf size %zd failed, no address space\n",
				   alloc->pid, size);
		binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
				   "allocated: %zd (num: %zd largest: %zd), free: %zd (num: %zd largest: %zd)\n",
				   total_alloc_size, allocated_buffers,
				   largest_alloc_size, total_free_size,
				   free_buffers, largest_free_size);
		return ERR_PTR(-ENOSPC);
	}
	if (n == NULL) {
		buffer = rb_entry(best_fit, struct binder_buffer, rb_node);
		buffer_size = binder_alloc_buffer_size(alloc, buffer);
	}

	binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
		     "%d: binder_alloc_buf size %zd got buffer %pK size %zd\n",
		      alloc->pid, size, buffer, buffer_size);

    // 将应用进程虚存地址空间 [buffer->user_data, end_page_addr] 映射内核物理内存地址空间
	// 这样应用进程写入此虚存空间的数据就有物理内存来存放了，并且是直接写入至 binder，无需从用户态拷贝至内核态
	// 这里会真正进行内核物理页的分配，参看章节【分配内核物理页】
	// 对应章节【指导思想】里的步骤四、五和六
	has_page_addr = (void __user *)
		(((uintptr_t)buffer->user_data + buffer_size) & PAGE_MASK);
	WARN_ON(n && buffer_size != size);
	end_page_addr =
		(void __user *)PAGE_ALIGN((uintptr_t)buffer->user_data + size);
	if (end_page_addr > has_page_addr)
		end_page_addr = has_page_addr;
	ret = binder_update_page_range(alloc, 1, (void __user *)
		PAGE_ALIGN((uintptr_t)buffer->user_data), end_page_addr);
	if (ret)
		return ERR_PTR(ret);

	if (buffer_size != size) {
		struct binder_buffer *new_buffer;

		new_buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
		if (!new_buffer) {
			pr_err("%s: %d failed to alloc new buffer struct\n",
			       __func__, alloc->pid);
			goto err_alloc_buf_struct_failed;
		}
		new_buffer->user_data = (u8 __user *)buffer->user_data + size;
		list_add(&new_buffer->entry, &buffer->entry);
		new_buffer->free = 1;
		binder_insert_free_buffer(alloc, new_buffer);
	}

	rb_erase(best_fit, &alloc->free_buffers);
	buffer->free = 0;
	buffer->allow_user_free = 0;
	binder_insert_allocated_buffer_locked(alloc, buffer);
	binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
		     "%d: binder_alloc_buf size %zd got %pK\n",
		      alloc->pid, size, buffer);
	buffer->data_size = data_size;
	buffer->offsets_size = offsets_size;
	buffer->async_transaction = is_async;
	buffer->extra_buffers_size = extra_buffers_size;
	buffer->pid = pid;
	buffer->oneway_spam_suspect = false;
	if (is_async) {
		alloc->free_async_space -= size + sizeof(struct binder_buffer);
		binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC_ASYNC,
			     "%d: binder_alloc_buf size %zd async free %zd\n",
			      alloc->pid, size, alloc->free_async_space);
		if (alloc->free_async_space < alloc->buffer_size / 10) {
			/*
			 * Start detecting spammers once we have less than 20%
			 * of async space left (which is less than 10% of total
			 * buffer size).
			 */
			buffer->oneway_spam_suspect = debug_low_async_space_locked(alloc, pid);
		} else {
			alloc->oneway_spam_detected = false;
		}
	}
	return buffer;

err_alloc_buf_struct_failed:
	binder_update_page_range(alloc, 0, (void __user *)
				 PAGE_ALIGN((uintptr_t)buffer->user_data),
				 end_page_addr);
	return ERR_PTR(-ENOMEM);
}
```

# binder_mmap

`open_driver(driver)` 得到 binder fd 后会执行 `binder_mmap` 分配所需的内存空间：

1. 应用进程分配 1M 的虚存至 `ProcessState.mVMStart`，这块虚存是还没有映射任何物理内存的

2. binder driver 初始化 `binder_alloc->free_buffers`，它代表空闲可用的应用进程虚存块，也即上面第一步分配的这块虚存；它是一棵红黑树，`binder_buffer->user_data` 记录了起始地址，节点间的起始地址差表示此节点的空间大小，最后一个节点表示应用进程开辟的虚存空间还剩余的大小

3. 此时 binder driver 并没有相应地为此应用进程分配虚存

```cpp
// ProcessState 是进程内单例，负责初始化应用进程与 binder driver 的连接（fd）
// https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp
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
		// 应用进程开辟了 1M 的虚拟地址空间，用以 binder 间传输数据
		// 注意此时只是在应用进程开辟了 1M 的虚拟内存，并没有映射物理内存
        mVMStart = mmap(nullptr, BINDER_VM_SIZE /* 1M */, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }
}

// 大概 1M 左右
#define BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)

// 来到内核 binder driver 的 mmap 处理函数
// https://cs.android.com/android/kernel/superproject/+/common-android-mainline:common/drivers/android/binder.c
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	struct binder_proc *proc = filp->private_data;

	if (proc->tsk != current->group_leader)
		return -EINVAL;

	binder_debug(BINDER_DEBUG_OPEN_CLOSE,
		     "%s: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",
		     __func__, proc->pid, vma->vm_start, vma->vm_end,
		     (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,
		     (unsigned long)pgprot_val(vma->vm_page_prot));

	if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
		pr_err("%s: %d %lx-%lx %s failed %d\n", __func__,
		       proc->pid, vma->vm_start, vma->vm_end, "bad vm_flags", -EPERM);
		return -EPERM;
	}
	vma->vm_flags |= VM_DONTCOPY | VM_MIXEDMAP;
	vma->vm_flags &= ~VM_MAYWRITE;

	vma->vm_ops = &binder_vm_ops;
	vma->vm_private_data = proc;

	return binder_alloc_mmap_handler(&proc->alloc, vma);
}

/**
 * binder_alloc_mmap_handler() - map virtual address space for proc
 * @alloc:	alloc structure for this proc
 * @vma:	vma passed to mmap()
 *
 * Called by binder_mmap() to initialize the space specified in
 * vma for allocating binder buffers
 *
 * Return:
 *      0 = success
 *      -EBUSY = address space already mapped
 *      -ENOMEM = failed to map memory to given address space
 *
 * https://cs.android.com/android/kernel/superproject/+/common-android-mainline:common/drivers/android/binder_alloc.c
 */
int binder_alloc_mmap_handler(struct binder_alloc *alloc,
			      struct vm_area_struct *vma)
{
	int ret;
	const char *failure_string;
	struct binder_buffer *buffer;

	if (unlikely(vma->vm_mm != alloc->mm)) {
		ret = -EINVAL;
		failure_string = "invalid vma->vm_mm";
		goto err_invalid_mm;
	}

	mutex_lock(&binder_alloc_mmap_lock);
	if (alloc->buffer_size) {
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;
	}

	// vm_start 是应用进程虚拟地址空间的起始地址，vm_end 是结束地址
	// vma->vm_end - vma->vm_start 就是应用进程内这块虚拟地址的大小，上面说过是 1M
	// alloc->buffer      是应用进程的虚拟地址
	// alloc->buffer_size 是这块内存空间的大小
	alloc->buffer_size = min_t(unsigned long, vma->vm_end - vma->vm_start,
				   SZ_4M);
	mutex_unlock(&binder_alloc_mmap_lock);
	alloc->buffer = (void __user *)vma->vm_start;

    // kcalloc — allocate memory for an array. The memory is set to zero.
    // void* kcalloc (size_t n /* number of elements */, size_t size /* element size */, gfp_t flags);
	// 
	// binder_alloc.pages 表示内核物理页，此时只是初始化了这个数组（索引表示第几页），并没有分配物理页
	// 应用进程与 binder 进行数据传输时，会分配物理页并将应用进程虚存映射至此物理页，参看章节【分配内核物理页】
	alloc->pages = kcalloc(alloc->buffer_size / PAGE_SIZE,
			       sizeof(alloc->pages[0]),
			       GFP_KERNEL);
	if (alloc->pages == NULL) {
		ret = -ENOMEM;
		failure_string = "alloc page array";
		goto err_alloc_pages_failed;
	}

    // 初始化 binder driver fd 时就已经放了一个 binder_buffer 至 alloc->free_buffers
	buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
	if (!buffer) {
		ret = -ENOMEM;
		failure_string = "alloc buffer struct";
		goto err_alloc_buf_struct_failed;
	}

	buffer->user_data = alloc->buffer;          // binder_buffer 也记录了应用进程的虚存起始地址
	list_add(&buffer->entry, &alloc->buffers);  // alloc->buffers, list of all buffers for this proc
	buffer->free = 1;
	binder_insert_free_buffer(alloc, buffer);   // alloc->free_buffers, rb tree of buffers available for allocation sorted by size
	alloc->free_async_space = alloc->buffer_size / 2;
	alloc->vma_addr = vma->vm_start;           // 应用进程的虚存起始地址
	// ...
}

// alloc->free_buffers 是一棵红黑树（平衡的二叉树）
// 左孩子内存空间较小，右孩子内存空间较大，这里的内存是用户进程的虚存
//
// 但 binder_buffer->user_data 只记录了虚存起始地址，并没有字段记录大小
// 这块地址的大小跟 binder_buffer 在红黑树上的位置有关，看 binder_alloc_buffer_size
// 
// 1，如果 binder_buffer 是最后一个节点，那么它表示应用进程开辟的虚存空间还剩余的大小
// 2，否则此节点与下一节点的起始地址差（binder_buffer->user_data）就是此节点的空间大小
static void binder_insert_free_buffer(struct binder_alloc *alloc,
				      struct binder_buffer *new_buffer)
{
	struct rb_node **p = &alloc->free_buffers.rb_node;
	struct rb_node *parent = NULL;
	struct binder_buffer *buffer;
	size_t buffer_size;
	size_t new_buffer_size;

	BUG_ON(!new_buffer->free);

	new_buffer_size = binder_alloc_buffer_size(alloc, new_buffer);

	binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
		     "%d: add free buffer, size %zd, at %pK\n",
		      alloc->pid, new_buffer_size, new_buffer);

	while (*p) {
		parent = *p;
		buffer = rb_entry(parent, struct binder_buffer, rb_node);
		BUG_ON(!buffer->free);

		buffer_size = binder_alloc_buffer_size(alloc, buffer);

		if (new_buffer_size < buffer_size)
			p = &parent->rb_left;
		else
			p = &parent->rb_right;
	}
	rb_link_node(&new_buffer->rb_node, parent, p);
	rb_insert_color(&new_buffer->rb_node, &alloc->free_buffers);
}

static size_t binder_alloc_buffer_size(struct binder_alloc *alloc,
				       struct binder_buffer *buffer)
{
	if (list_is_last(&buffer->entry, &alloc->buffers))
		return alloc->buffer + alloc->buffer_size - buffer->user_data;
	return binder_buffer_next(buffer)->user_data - buffer->user_data;
}
```

# 分配内核物理页

> `struct vm_area_struct`
> 
> This struct describes a virtual memory area. There is one of these per VM-area/task. A VM area is any part of the process virtual memory space that has a special rule for the page-fault handlers (ie a shared library, the executable area etc).
> 
> 虚存管理的最基本的管理单元，描述了一段连续的、具有相同访问属性的虚存空间，该虚存空间的大小为物理内存页面的整数倍

> `int vm_insert_page(struct vm_area_struct *vma, unsigned long addr, struct page *page)`
> 
> vma - user vma to map to, addr - target user address of this page, page - source kernel page
>
> This allows drivers to insert individual pages they've allocated into a user vma.
> 
> 一般用在驱动的 mmap 操作里，建立内核物理内存与用户虚拟地址空间的映射关系；驱动的常见套路是不会立即为用户虚拟地址空间分配物理内存，而是当访问虚拟地址空间时产生缺页中断，此时才真正地为缺页分配物理内存

[binder_mmap](#binder_mmap) 应用进程已经开辟了一块虚存用以与 binder 进行数据传输，但并没有为它映射物理页，直到真正访问这块虚存时才会在内核分配物理页

```cpp
// 为 [start, end] 映射内核物理页（binder_proc.pages）
// [start, end] 是一块应用进程的虚拟地址空间，它是在初始化 binder driver fd 时分配的，参见【binder_mmap】章节
// https://cs.android.com/android/kernel/superproject/+/common-android-mainline:common/drivers/android/binder_alloc.c
static int binder_update_page_range(struct binder_alloc *alloc, int allocate /* true - 映射物理页，false - 取消映射 */,
				    void __user *start, void __user *end)
{
	void __user *page_addr;
	unsigned long user_page_addr;
	struct binder_lru_page *page;
	struct vm_area_struct *vma = NULL;
	struct mm_struct *mm = NULL;
	bool need_mm = false;

	binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
		     "%d: %s pages %pK-%pK\n", alloc->pid,
		     allocate ? "allocate" : "free", start, end);

	if (end <= start)
		return 0;

	trace_binder_update_page_range(alloc, allocate, start, end);

	if (allocate == 0)
		goto free_range;

    // binder_alloc.pages 是 binder_lru_page 类型的数组，表示内核物理内存空间，用以在应用进程和 binder 之间传输数据
	// binder 按页大小（PAGE_SIZE）管理这块物理内存空间，数组索引代表第几页，binder_lru_page.page_ptr 指向物理页的地址
	// 早在 binder_alloc_mmap_handler() 时就已经按应用进程 mmap 的大小（1M）初始化了这个数组，参看章节【binder_mmap】
	// 这里是看虚拟内存空间 [start, end] 有没映射至内核物理地址页，没有的话用 need_mm 标识需要映射
	for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
		page = &alloc->pages[(page_addr - alloc->buffer) / PAGE_SIZE];
		if (!page->page_ptr) {
			need_mm = true;
			break;
		}
	}

	if (need_mm && mmget_not_zero(alloc->mm))
		mm = alloc->mm;

    // 通过应用进程虚拟内存起始地址找到这块虚存的描述 vm_area_struct
	if (mm) {
		mmap_read_lock(mm);
		vma = vma_lookup(mm, alloc->vma_addr);
	}

	if (!vma && need_mm) {
		binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
				   "%d: binder_alloc_buf failed to map pages in userspace, no vma\n",
				   alloc->pid);
		goto err_no_vma;
	}

    // 一页页地映射应用进程虚存地址空间 [start, end]
	// 这块虚存是在 [alloc->buffer, alloc->buffer + alloc.buffer_size] 区间内的，参看章节【binder_mmap】
	for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
		int ret;
		bool on_lru;
		size_t index;

        // 算出这页虚存对应哪页物理内存
		index = (page_addr - alloc->buffer) / PAGE_SIZE;
		page = &alloc->pages[index];

		if (page->page_ptr) {  // 物理页已分配则跳过
			trace_binder_alloc_lru_start(alloc, index);

			on_lru = list_lru_del(&binder_alloc_lru, &page->lru);
			WARN_ON(!on_lru);

			trace_binder_alloc_lru_end(alloc, index);
			continue;
		}

		if (WARN_ON(!vma))
			goto err_page_ptr_cleared;

        // 真正在内核物理内存空间分配一页物理页
		// 对应章节【指导思想】里的步骤四
		trace_binder_alloc_page_start(alloc, index);
		page->page_ptr = alloc_page(GFP_KERNEL |
					    __GFP_HIGHMEM |
					    __GFP_ZERO);
		if (!page->page_ptr) {
			pr_err("%d: binder_alloc_buf failed for page at %pK\n",
				alloc->pid, page_addr);
			goto err_alloc_page_failed;
		}
		page->alloc = alloc;
		INIT_LIST_HEAD(&page->lru);

        // 然后将应用进程虚存页映射到内核物理页 
		// 对应章节【指导思想】里的步骤五
		user_page_addr = (uintptr_t)page_addr;
		ret = vm_insert_page(vma, user_page_addr, page[0].page_ptr);
		if (ret) {
			pr_err("%d: binder_alloc_buf failed to map page at %lx in userspace\n",
			       alloc->pid, user_page_addr);
			goto err_vm_insert_page_failed;
		}

		if (index + 1 > alloc->pages_high)
			alloc->pages_high = index + 1;

		trace_binder_alloc_page_end(alloc, index);
	}
	if (mm) {
		mmap_read_unlock(mm);
		mmput(mm);
	}
	return 0;

// 看下是如何取消映射的
// 依然是按页大小（PAGE_SIZE）遍历应用进程虚存空间 [start, end]，这里是从高地址开始
free_range:
	for (page_addr = end - PAGE_SIZE; 1; page_addr -= PAGE_SIZE) {
		bool ret;
		size_t index;

		index = (page_addr - alloc->buffer) / PAGE_SIZE;
		page = &alloc->pages[index];

		trace_binder_free_lru_start(alloc, index);

        // list_lru_add 和 list_lru_del 是一套帮助内存 shrink 的 API，目前还没搞清楚它是怎么运行的
		ret = list_lru_add(&binder_alloc_lru, &page->lru);
		WARN_ON(!ret);

		trace_binder_free_lru_end(alloc, index);
		if (page_addr == start)
			break;
		continue;

err_vm_insert_page_failed:
		__free_page(page->page_ptr);
		page->page_ptr = NULL;
err_alloc_page_failed:
err_page_ptr_cleared:
		if (page_addr == start)
			break;
	}
err_no_vma:
	if (mm) {
		mmap_read_unlock(mm);
		mmput(mm);
	}
	return vma ? -ENOMEM : -ESRCH;
}
```

# 参考

- [Binder | 内存拷贝的本质和变迁 - 芦半山 - 稀土掘金](https://github.com/cyrus-lin/bookmark/issues/46)