---
title: Binder IPC 线程调度模型
date: 2022-12-07 12:00:00 +0800
---

# client send request

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
				struct binder_thread *thread)
{
	// ... first write request
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
    // then wait & read response
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
	// ...
}

static int binder_thread_write(struct binder_proc *proc,
			struct binder_thread *thread,
			binder_uintptr_t binder_buffer, size_t size,
			binder_size_t *consumed)

static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply,
			       binder_size_t extra_buffers_size)

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
	bool oneway = !!(t->flags & TF_ONE_WAY);
	bool pending_async = false;
	struct binder_transaction *t_outdated = NULL;

	BUG_ON(!node);
	binder_node_lock(node);

	if (oneway) {
		BUG_ON(thread);
		if (node->has_async_transaction)
			pending_async = true;
		else
			node->has_async_transaction = true;
	}

	binder_inner_proc_lock(proc);
	if (proc->is_frozen) {
		proc->sync_recv |= !oneway;
		proc->async_recv |= oneway;
	}

	if ((proc->is_frozen && !oneway) || proc->is_dead ||
			(thread && thread->is_dead)) {
		binder_inner_proc_unlock(proc);
		binder_node_unlock(node);
		return proc->is_frozen ? BR_FROZEN_REPLY : BR_DEAD_REPLY;
	}

    // 如果没有指定线程，则从目标进程里找一个
	if (!thread && !pending_async)
		thread = binder_select_thread_ilocked(proc);

    // 如果目标进程中有可用线程，则将任务加到线程的 todo list
    // struct binder_thread {
    //     struct list_head todo; # 任务链表
	//     bool process_todo;     # 标识是否处理任务
    // }
	if (thread) {
		binder_transaction_priority(thread, t, node);
		binder_enqueue_thread_work_ilocked(thread, &t->work);

    // 否则就把任务加到目标进程的 todo list
    // struct binder_proc {
    //     struct list_head todo; # 任务链表
    // }
	} else if (!pending_async) {
		binder_enqueue_work_ilocked(&t->work, &proc->todo);
	} else {
		if ((t->flags & TF_UPDATE_TXN) && proc->is_frozen) {
			t_outdated = binder_find_outdated_transaction_ilocked(t,
									      &node->async_todo);
			if (t_outdated) {
				binder_debug(BINDER_DEBUG_TRANSACTION,
					     "txn %d supersedes %d\n",
					     t->debug_id, t_outdated->debug_id);
				list_del_init(&t_outdated->work.entry);
				proc->outstanding_txns--;
			}
		}
		binder_enqueue_work_ilocked(&t->work, &node->async_todo);
	}

    // 唤醒目标进程里的线程去处理任务
	if (!pending_async)
		binder_wakeup_thread_ilocked(proc, thread, !oneway /* sync */);

	proc->outstanding_txns++;
	binder_inner_proc_unlock(proc);
	binder_node_unlock(node);

	/*
	 * To reduce potential contention, free the outdated transaction and
	 * buffer after releasing the locks.
	 */
	if (t_outdated) {
		struct binder_buffer *buffer = t_outdated->buffer;

		t_outdated->buffer = NULL;
		buffer->transaction = NULL;
		trace_binder_transaction_update_buffer_release(buffer);
		binder_transaction_buffer_release(proc, NULL, buffer, 0, 0);
		binder_alloc_free_buf(&proc->alloc, buffer);
		kfree(t_outdated);
		binder_stats_deleted(BINDER_STAT_TRANSACTION);
	}

	return 0;
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
		ret = binder_wait_for_work(thread, wait_for_proc_work);
	}
    // ...
}

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

# binder schedule

