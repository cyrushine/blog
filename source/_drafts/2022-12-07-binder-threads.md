---
title: Binder IPC 线程调度模型
date: 2022-12-07 12:00:00 +0800
---

# client 端的线程调度



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
                int reply)    // false: client request, false: server response
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
    
    if (reply) {    // BC_REPLY 说明这是一个 Server 发给 Client 的事务处理回复，在 server 端的线程上
        // ...
    } else {        // BC_TRANSACTION 说明这是一个 Client 发给 Server 的请求事务，在 Client 端线程上
        // 确定目标进程 target_proc
        if (tr->target.handle) {
            struct binder_ref *ref;
            ref = binder_get_ref(proc, tr->target.handle);
            if (ref == NULL) {
                binder_user_error("%d:%d got transaction to invalid handle\n",
                    proc->pid, thread->pid);
                return_error = BR_FAILED_REPLY;
                goto err_invalid_target_handle;
            }
            target_node = ref->node;
        } else {    // handle == 0 表示是 service manager 进程
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
        if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) { /*非one_way, 需要replay,且transaction栈不为空*/
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
            /* 从事务栈（transaction_stack）的栈顶向下搜索，
            * 找到最后（最早）一个目标进程中向当前进程发起事务请求的线程为本次请求的目标线程。
            */
            while (tmp) {
                if (tmp->from && tmp->from->proc == target_proc)
                    target_thread = tmp->from;
                tmp = tmp->from_parent;
            }
        }
    }

    // 
    if (target_thread) {
        /*找到target_thread, 则target_list和target_wait分别初始化为目标线程的todo和wait队列*/
        e->to_thread = target_thread->pid;
        target_list = &target_thread->todo;
        target_wait = &target_thread->wait;
    } else {
        /* 没有找到target_thread, target_list和target_wait分别初始化为目标进程的todo和wait队列
        * 这个情况只有BC_TRANSACTION命令才有可能发生
        */
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }
    e->to_proc = target_proc->pid;
    
    t = kzalloc(sizeof(*t), GFP_KERNEL);                 // 分配一个 binder_transaction
    if (t == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_alloc_t_failed;
    }
    binder_stats_created(BINDER_STAT_TRANSACTION);

    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);  // 分配一个 binder_work
    if (tcomplete == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_alloc_tcomplete_failed;
    }
    binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);

    if (!reply && !(tr->flags & TF_ONE_WAY)) /*BC_TRANSACTION，且不是one way，即需要replay，则发起线程（from）设为当前线程*/
      t->from = thread;
    else/*BC_REPLY，from置为空*/
        t->from = NULL;
    /*初始化binder_transaction各域*/
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
    if (target_node) /*该target_node被binder_buffer引用，所以增加引用计数*/
       binder_inc_node(target_node, 1, 0, NULL);
    /*计算offset区的起始地址*/
    offp = (binder_size_t *)(t->buffer->data +
                 ALIGN(tr->data_size, sizeof(void *)));
    /*将用户态binder_transaction_data中的数据拷贝到内核驱动的binder_buffer中，binder通信的一次拷贝就是发生在这里*/
    if (copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)
            **tr->data.ptr.buffer**, tr->data_size)) {
        binder_user_error("%d:%d got transaction with invalid data ptr\n",
                proc->pid, thread->pid);
        return_error = BR_FAILED_REPLY;
        goto err_copy_data_failed;
    }
    /* 拷贝binder_transaction_data的offset区到内核驱动
    */
    if (copy_from_user(offp, (const void __user *)(uintptr_t)
            tr->data.ptr.offsets, tr->offsets_size)) {
        binder_user_error("%d:%d got transaction with invalid offsets ptr\n",
                proc->pid, thread->pid);
        return_error = BR_FAILED_REPLY;
        goto err_copy_data_failed;
    }
    if (!IS_ALIGNED(tr->offsets_size, sizeof(binder_size_t))) {
        binder_user_error("%d:%d got transaction with invalid offsets size, %lld\n",
                proc->pid, thread->pid, (u64)tr->offsets_size);
        return_error = BR_FAILED_REPLY;
        goto err_bad_offset;
    }

    // 处理在前一步从 binder_transaction_data 中拷贝进来所有 flat_binder_object
    off_end = (void *)offp + tr->offsets_size;
    off_min = 0;
    for (; offp < off_end; offp++) {
        // ...
    }

    // 
    if (reply) {
        BUG_ON(t->buffer->async_transaction != 0);
        /*事务处理完成，将本次reply对应的transaction从目标线程（Client）事务栈中移除，并释放其所占用的地址空间*/
     binder_pop_transaction(target_thread, in_reply_to);
    } else if (!(t->flags & TF_ONE_WAY)) {/*一个client到server的transaction，且需要reply*/
        /*将本次事务的binder_transaction加入到本线程事务栈中*/
        BUG_ON(t->buffer->async_transaction != 0);
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        **thread->transaction_stack = t;**
    } else {
        BUG_ON(target_node == NULL);
        BUG_ON(t->buffer->async_transaction != 1);
        if (target_node->has_async_transaction) {
            target_list = &target_node->async_todo;
            *target_wait = NULL;
        } else
            target_node->has_async_transaction = 1;
    }


    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);

    // 添加一个本线程的todo队列中，稍后在线程处理todo队列的该binder_work时，会发送个BR_WORK_TRANSCAION_COMPLETE给进程，告知请求/回复已发送出去
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);

    // 
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
```

# binder schedule



# 进程调度 & 等待队列



# 参考

- [Binder驱动之设备控制 binder_ioctl -- 一](https://www.jianshu.com/p/49830c3473b7)
- [Binder驱动之设备控制 binder_ioctl -- 二](https://www.jianshu.com/p/12bec1b16a5b)
- [Binder驱动之设备控制 binder_ioctl -- 三](https://www.jianshu.com/p/4be292a51388)