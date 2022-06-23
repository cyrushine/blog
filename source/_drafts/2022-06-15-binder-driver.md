---
title: 深入 Binder 之驱动篇（Binder Driver）
date: 2022-06-15 12:00:00 +0800
---

# Linux 设备与驱动

![devices.jpg](../../../../2022-06-13-binder-driver/devices.jpg)

linux 系统将设备分为3类：字符设备、块设备和网络设备：

1. `字符设备` 是指只能一个字节一个字节读写的设备，不能随机读取设备内存中的某一数据，读取数据需要按照先后顺序。字符设备是面向流的设备，常见的字符设备有鼠标、键盘、串口、控制台和 LED 设备等

2. `块设备` 是指可以从设备的任意位置读取一定长度数据的设备。块设备包括硬盘、磁盘、U 盘和 SD 卡等

每个字符设备或块设备都在 `/dev` 目录下对应一个设备文件，linux 用户程序通过设备文件（或称设备节点）来使用驱动程序操作字符设备和块设备

## 字符设备

字符设备、字符设备驱动与用户空间访问该设备的程序三者之间的关系如下图：
* 使用 cdev 结构体来描述字符设备
* 通过其成员 dev_t 来定义设备号（分为主、次设备号）以确定字符设备的唯一性
* 通过其成员 file_operations 来定义字符设备驱动提供给 VFS 的接口函数，如常见的 open()、read()、write() 等

![device_driver.jpg](../../../../2022-06-13-binder-driver/device_driver.jpg)

在 Linux 字符设备驱动中：
* 模块加载函数通过 register_chrdev_region 或 alloc_chrdev_region 来静态或者动态获取设备号
* 通过 cdev_init 建立 cdev 与 file_operations 之间的连接，通过 cdev_add 向系统添加一个 cdev 以完成注册
* 模块卸载函数通过 cdev_del 来注销 cdev，通过 unregister_chrdev_region 来释放设备号

![chrdev.jpg](../../../../2022-06-13-binder-driver/chrdev.jpg)

## 杂项设备（misc device）

杂项设备也是在嵌入式系统中用得比较多的一种设备驱动，使用 `misc_register(struct miscdevice *misc)` 注册杂项设备，`misc_deregister(struct miscdevice *misc)` 释放杂项设备

在 Linux 内核的 include/linux 目录下有 Miscdevice.h 文件，要把自己定义的 misc device 设备定义在这里。其实是因为这些字符设备不符合预先确定的字符设备范畴，所有这些设备采用主编号 10 ，一起归于 misc device，其实 misc_register 就是用主标号 10 调用 register_chrdev() 的，也就是说 misc device 就是特殊的字符设备，可自动生成设备节点

misc device 是特殊字符设备，注册驱动程序时采用 misc_register 函数注册，此函数中会自动创建设备节点（即设备文件），无需 mknod 指令手动创建设备文件

杂项字符设备和一般字符设备的区别：
1. 一般字符设备首先申请设备号，但是杂项字符设备的主设备号为 10 次设备号通过结构体 miscdevice 中的 minor 来设置
2. 一般字符设备要创建设备文件，但是杂项字符设备在注册时会自动创建
3. 一般字符设备要分配一个 cdev（字符设备），但是杂项字符设备只要创建 miscdevice 结构体即可
4. 一般字符设备需要初始化 cdev，即给字符设备设置对应的操作函数集 file_operation，但是杂项字符设备在结构体 miscdevice 中定义
5. 一般字符设备使用注册函数 cdev_add 而杂项字符设备使用 misc_register 来注册
 

驱动调用的实质，就是通过设备文件找到与之对应设备号的设备，再通过设备初始化时绑定的操作函数对硬件进行控制

```cpp
struct miscdevice{
　　int minor;                          // 杂项设备的此设备号(如果设置为 MISC_DYNAMIC_MINOR，表示系统自动分配未使用的 minor)
　　const char *name;
　　const stuct file_operations *fops;  // 驱动主题函数入口指针
　　struct list_head list;
　　struct device *parent;
　　struct device *this device;
　　const char *nodename;              // 在 /dev 目录下面创建的设备驱动节点
　　mode_t mode;
};
```

## 常见的设备列表

| 主设备号 | 次设备号 | 文件名 | 设备类型 | 说明 |
|---------|----------|-------|---------|------|
| 1  | 3   | /dev/null  | char  | 空设备。任何写入都将被直接丢弃(但返回"成功")；任何读取都将得到 EOF (文件结束标志) |
| 4  | 0   | /dev/tty0  | char  | 当前虚拟控制台 |
|    | 1   | /dev/tty1  | char  | 第 1 个虚拟控制台 |
| 8  | 0   | /dev/sda   | block | 第 1 个磁盘 |
|    | 16  | /dev/sdb   | block | 第 2 个磁盘 |
|    | 32  | /dev/sdc   | block | 第 3 个磁盘 |
| 10 | 1   | /dev/psaux | char  | PS/2 鼠标 |
|    | 156 | /dev/lcd   | char  | 液晶(LCD)显示屏 |

# Binder 也是设备

Binder 以杂项设备（misc device，是特殊的字符设备）进行注册，作为虚拟字符设备没有直接操作硬件，只是对设备内存的处理，主要是驱动的初始化 `binder_init`、打开 `binder_open`、映射`binder_mmap` 和数据操作 `binder_ioctl`

* 通过 init() 创建 /dev/binder 设备节点
* 通过 open() 获取 Binder Driver 的文件描述符
* 通过 mmap() 在内核分配一块内存用于存放数据
* 通过 ioctl() 将 IPC 数据作为参数传递给 Binder Driver

Binder 驱动是 Android 专用的，但底层的驱动架构与 Linux 设备驱动一样，用户态的程序调用 Kernel 层驱动是需要陷入内核态进行系统调用（syscall），比如打开 Binder 驱动方法的调用链是：

```
open() -> __open() -> binder_open()
```

1. open() 为用户空间的方法
2. _open() 是系统调用中相应的处理方法
3. 通过查找，对应调用到内核 binder 驱动的 binder_open 方法

![systemcall.png](../../../../2022-06-13-binder-driver/systemcall.png)

Client 进程通过 RPC(Remote Procedure Call Protocol) 与 Server 通信的过程，可以简单的分为三层：驱动层、IPC 层和业务层
* demo() 是 client 和 server 共同协商好的统一方法
* RPC 数据、code、handle 和协议这四项组成了 IPC 的层的数据，通过 IPC 层进行数据传输
* 而真正在 Client 和 Server 两端建立通信的基础设施是 Binder Driver

![binderdriver_frame.png](../../../../2022-06-13-binder-driver/binderdriver_frame.png)

# 注册 Binder Driver

> Name  
> misc_register — register a miscellaneous device  
> 
> Synopsis  
> int misc_register (struct miscdevice *misc);
>  
> Arguments  
> struct miscdevice *misc  
> device structure
> 
> Description  
> Register a miscellaneous device with the kernel. If the minor number is set to MISC_DYNAMIC_MINOR a minor number is assigned and placed in the minor field of the structure. For other > cases the minor number requested is used.  
> 
> The structure passed is linked into the kernel and may not be destroyed until it has been unregistered. By default, an open syscall to the device sets file->private_data to point to > the structure. Drivers don't need open in fops for this.  
> 
> A zero is returned on success and a negative errno code for failure.

```cpp
// https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/drivers/android/binder.c
static int __init binder_init(void) {...}

static int __init init_binder_device(const char *name)
{
	int ret;
	struct binder_device *binder_device;
	binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
	if (!binder_device)
		return -ENOMEM;
	binder_device->miscdev.fops = &binder_fops;         // 系统调用与驱动内方法的映射
	binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;  // 让系统分配次设备号
	binder_device->miscdev.name = name;                 
	refcount_set(&binder_device->ref, 1);
	binder_device->context.binder_context_mgr_uid = INVALID_UID;
	binder_device->context.name = name;
	mutex_init(&binder_device->context.context_mgr_node_lock);
	ret = misc_register(&binder_device->miscdev);       // 将 Binder 注册为杂项设备
	if (ret < 0) {
		kfree(binder_device);
		return ret;
	}
	hlist_add_head(&binder_device->hlist, &binder_devices);
	return ret;
}

const struct file_operations binder_fops = {
	.owner = THIS_MODULE,
	.poll = binder_poll,
	.unlocked_ioctl = binder_ioctl,
	.compat_ioctl = compat_ptr_ioctl,
	.mmap = binder_mmap,
	.open = binder_open,
	.flush = binder_flush,
	.release = binder_release,
};
```

# binder_open

创建 binder_proc 对象，并把当前进程等信息保存到 binder_proc，该对象管理 IPC 所需要的各种信息并拥有其他结构体的根结构体

把 binder_proc 对象保存到文件指针中（`filp->private_data`）

把 binder_proc 加入到全局链表 binder_procs

```cpp
// https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/drivers/android/binder.c
static int binder_open(struct inode *nodp, struct file *filp)
{
	struct binder_proc *proc, *itr;
	struct binder_device *binder_dev;
	struct binderfs_info *info;
	struct dentry *binder_binderfs_dir_entry_proc = NULL;
	bool existing_pid = false;
	binder_debug(BINDER_DEBUG_OPEN_CLOSE, "%s: %d:%d\n", __func__,
		     current->group_leader->pid, current->pid);
	proc = kzalloc(sizeof(*proc), GFP_KERNEL);
	if (proc == NULL)
		return -ENOMEM;
	spin_lock_init(&proc->inner_lock);
	spin_lock_init(&proc->outer_lock);
	get_task_struct(current->group_leader);
	proc->tsk = current->group_leader;
	proc->cred = get_cred(filp->f_cred);
	INIT_LIST_HEAD(&proc->todo);
	init_waitqueue_head(&proc->freeze_wait);
	if (binder_supported_policy(current->policy)) {
		proc->default_priority.sched_policy = current->policy;
		proc->default_priority.prio = current->normal_prio;
	} else {
		proc->default_priority.sched_policy = SCHED_NORMAL;
		proc->default_priority.prio = NICE_TO_PRIO(0);
	}
	/* binderfs stashes devices in i_private */
	if (is_binderfs_device(nodp)) {           // binderfs 一般为 false
		binder_dev = nodp->i_private;
		info = nodp->i_sb->s_fs_info;
		binder_binderfs_dir_entry_proc = info->proc_log_dir;
	} else {
		binder_dev = container_of(filp->private_data,
					  struct binder_device, miscdev);
	}
	refcount_inc(&binder_dev->ref);
	proc->context = &binder_dev->context;
	binder_alloc_init(&proc->alloc);
	binder_stats_created(BINDER_STAT_PROC);
	proc->pid = current->group_leader->pid;
	INIT_LIST_HEAD(&proc->delivered_death);
	INIT_LIST_HEAD(&proc->waiting_threads);
	filp->private_data = proc;                // 把 binder_proc 对象保存到文件指针中
	mutex_lock(&binder_procs_lock);
	hlist_for_each_entry(itr, &binder_procs, proc_node) {
		if (itr->pid == proc->pid) {
			existing_pid = true;
			break;
		}
	}
	hlist_add_head(&proc->proc_node, &binder_procs);
	mutex_unlock(&binder_procs_lock);
	if (binder_debugfs_dir_entry_proc && !existing_pid) {
		char strbuf[11];
		snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
		/*
		 * proc debug entries are shared between contexts.
		 * Only create for the first PID to avoid debugfs log spamming
		 * The printing code will anyway print all contexts for a given
		 * PID so this is not a problem.
		 */
		proc->debugfs_entry = debugfs_create_file(strbuf, 0444,
			binder_debugfs_dir_entry_proc,
			(void *)(unsigned long)proc->pid,
			&proc_fops);
	}
	if (binder_binderfs_dir_entry_proc && !existing_pid) {
		char strbuf[11];
		struct dentry *binderfs_entry;
		snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
		/*
		 * Similar to debugfs, the process specific log file is shared
		 * between contexts. Only create for the first PID.
		 * This is ok since same as debugfs, the log file will contain
		 * information on all contexts of a given PID.
		 */
		binderfs_entry = binderfs_create_file(binder_binderfs_dir_entry_proc,
			strbuf, &proc_fops, (void *)(unsigned long)proc->pid);
		if (!IS_ERR(binderfs_entry)) {
			proc->binderfs_entry = binderfs_entry;
		} else {
			int error;
			error = PTR_ERR(binderfs_entry);
			pr_warn("Unable to create file %s in binderfs (error %d)\n",
				strbuf, error);
		}
	}
	return 0;
}
```

# binder_mmap

通过mmap映射内存，其中ServiceManager映射的空间大小为128K,其他Binder应用进程映射的内存大小为1M-8k；

Binder驱动基于这种映射的内存采用最佳匹配来动态分配和释放，通过bind_buffer结构体中的free字段来表示相应的buffer是空闲还是已分配状态。对于已分配的buffers加入到binder_proc中的allocated_buffers红黑树，对于空闲的buffer加入到free_buffers红黑树；

当应用程序需要内存时，根据所需内存大小从free_buffers中找到最合适的内存，并放入allocated_buffers树中，当应用程序处理完成后必须尽快使用BC_FREE_BUFFER命令来释放该buffer，从而添加回到free_buffers树。

用户空间调用 `void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);`

> mmap() creates a new mapping in the virtual address space of the
> calling process.  The starting address for the new mapping is
> specified in `addr`.  The `length` argument specifies the length of
> the mapping (which must be greater than 0).
> 
> If addr is NULL, then the kernel chooses the (page-aligned)
> address at which to create the mapping; this is the most portable
> method of creating a new mapping.  If addr is not NULL, then the
> kernel takes it as a hint about where to place the mapping; 
> 
> The contents of a file mapping (as opposed to an anonymous
> mapping; see MAP_ANONYMOUS below), are initialized using length
> bytes starting at `offset` offset in the file (or other object)
> referred to by the file descriptor `fd`. 	   	   

进入内核空间，分配虚存，并将相关信息保存至 `vm_area_struct`，从结构上看其成员属性能够与 `mmap()` 参数相匹配

`struct vm_area_struct` 描述的是一段连续的、具有相同访问属性的虚存空间，该虚存空间的大小为物理内存页面的整数倍；成员 `vm_start` 和 `vm_end` 表示该虚存空间的首地址和末地址后第一个字节的地址，以字节为单位，所以虚存空间范围可以用 `[vm_start, vm_end)` 表示

```cpp
struct vm_area_struct {
        struct mm_struct * vm_mm;       /* VM area parameters */
        unsigned long vm_start;
        unsigned long vm_end;
        pgprot_t vm_page_prot;
        unsigned short vm_flags;
/* AVL tree of VM areas per task, sorted by address */
        short vm_avl_height;
        struct vm_area_struct * vm_avl_left;
        struct vm_area_struct * vm_avl_right;
/* linked list of VM areas per task, sorted by address */
        struct vm_area_struct * vm_next;
/* for areas with inode, the circular list inode->i_mmap */
/* for shm areas, the circular list of attaches */
/* otherwise unused */
        struct vm_area_struct * vm_next_share;
        struct vm_area_struct * vm_prev_share;
/* more */
        struct vm_operations_struct * vm_ops;
        unsigned long vm_offset;
        struct inode * vm_inode;
        unsigned long vm_pte;                   /* shared mem */
};
```

回调 binder driver 的 `binder_mmap` -> `binder_alloc_mmap_handler`

```cpp
// https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/drivers/android/binder_alloc.h

struct binder_alloc {
	struct mutex mutex;
	struct vm_area_struct *vma;
	struct mm_struct *vma_vm_mm;
	void __user *buffer;  // base of per-proc address space mapped via mmap
	struct list_head buffers;
	struct rb_root free_buffers;
	struct rb_root allocated_buffers;
	size_t free_async_space;
	struct binder_lru_page *pages;  // array of binder_lru_page
	size_t buffer_size;  // size of address space specified via mmap
	uint32_t buffer_free;
	int pid;
	size_t pages_high;
	bool oneway_spam_detected;
};

/**
 * struct binder_lru_page - page object used for binder shrinker
 * @page_ptr: pointer to physical page in mmap'd space
 * @lru:      entry in binder_alloc_lru
 * @alloc:    binder_alloc for a proc
 */
struct binder_lru_page {
	struct list_head lru;
	struct page *page_ptr;
	struct binder_alloc *alloc;
};
```

> Name  
> kcalloc — allocate memory for an array. The memory is set to zero.
> 
> Synopsis  
> void* kcalloc (size_t n, size_t size, gfp_t flags);
>  
> Arguments  
> size_t n - number of elements.  
> size_t size - element size.  
> gfp_t flags - the type of memory to allocate (see kmalloc).  

```cpp
// https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/drivers/android/binder.c
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	struct binder_proc *proc = filp->private_data;  // 用户进程信息是在 binder_open 里获取的
	//...
	return binder_alloc_mmap_handler(&proc->alloc, vma);
}

// https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/drivers/android/binder_alloc.c

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
 */
int binder_alloc_mmap_handler(struct binder_alloc *alloc, struct vm_area_struct *vma)
{
	int ret;
	const char *failure_string;
	struct binder_buffer *buffer;
	mutex_lock(&binder_alloc_mmap_lock);
	if (alloc->buffer_size) {
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;
	}

	alloc->buffer_size = min_t(unsigned long, vma->vm_end - vma->vm_start, SZ_4M);
	mutex_unlock(&binder_alloc_mmap_lock);
	alloc->buffer = (void __user *)vma->vm_start;
	alloc->pages = kcalloc(alloc->buffer_size / PAGE_SIZE,
			       sizeof(alloc->pages[0]),
			       GFP_KERNEL);
	if (alloc->pages == NULL) {
		ret = -ENOMEM;
		failure_string = "alloc page array";
		goto err_alloc_pages_failed;
	}
	buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
	if (!buffer) {
		ret = -ENOMEM;
		failure_string = "alloc buffer struct";
		goto err_alloc_buf_struct_failed;
	}
	buffer->user_data = alloc->buffer;
	list_add(&buffer->entry, &alloc->buffers);
	buffer->free = 1;
	binder_insert_free_buffer(alloc, buffer);
	alloc->free_async_space = alloc->buffer_size / 2;
	binder_alloc_set_vma(alloc, vma);
	mmgrab(alloc->vma_vm_mm);
	return 0;

err_alloc_buf_struct_failed:
	kfree(alloc->pages);
	alloc->pages = NULL;
err_alloc_pages_failed:
	alloc->buffer = NULL;
	mutex_lock(&binder_alloc_mmap_lock);
	alloc->buffer_size = 0;
err_already_mapped:
	mutex_unlock(&binder_alloc_mmap_lock);
	binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
			   "%s: %d %lx-%lx %s failed %d\n", __func__,
			   alloc->pid, vma->vm_start, vma->vm_end,
			   failure_string, ret);
	return ret;
}
```

# binder 通讯模型/协议

一次完整的 Binder 通信过程如下（非 oneway）

Binder 协议包含在 IPC 数据中，分为两类：
1. BINDER_COMMAND_PROTOCOL：binder 请求码，以 BC_ 开头，简称 `BC` 码，用于从客户端/服务端传递到 Binder Driver
2. BINDER_RETURN_PROTOCOL：binder 响应码，以 BR_ 开头，简称 `BR` 码，用于从 Binder Driver 传递到客户端/服务端

Binder IPC 通信至少是两个进程的交互：
1. client 进程执行 binder_thread_write，根据 BC_xx 命令，生成相应的 binder_work
2. server 进程执行 binder_thread_read，根据 binder_work_type 类型，生成 BR_xx，发送到用户空间处理

![bindermodel.png](../../../../2022-06-13-binder-driver/bindermodel.jpg)

# BC 请求码（BC_PROTOCOL）

Binder 的请求码是在 `binder_driver_command_protocol` 中定义的，用于应用程序向 binder 驱动设备发送请求消息，应用程序包含 Client 和 Server 端，以 BC_ 开头，总共 19 条

| 请求码 | 参数类型 | 作用 |
|-------|---------|------|
| BC_TRANSACTION	              | binder_transaction_data     | Client 向 Binder 驱动发送的请求数据 |
| BC_REPLY	                      | binder_transaction_data	    |  Server 向 Binder 驱动发送的回复数据 |
| BC_ACQUIRE_RESULT	              | __s32	                    |  暂时不支持 |
| BC_FREE_BUFFER	              | binder_uintptr_t	        |  释放内存 |
| BC_INCREFS	                  | __u32	                    |  binder_ref 弱引用加1操作（这些请求码的作用是对 binder 的强/弱引用的计数操作，用于实现强/弱指针的功能） |
| BC_ACQUIRE	                  | __u32	                    |  binder_ref 弱引用减1操作 |
| BC_RELEASE	                  | __u32	                    |  binder_ref 强引用加1操作 |
| BC_DECREFS	                  | __u32	                    |  binder_ref 强引用减1操作 |
| BC_INCREFS_DONE	              | binder_ptr_cookie	        |  binder_node 强引用减1操作 |
| BC_ACQUIRE_DONE	              | binder_ptr_cookie	        |  binder_node 弱引用减1操作 |
| BC_ATTEMPT_ACQUIRE	          | binder_pri_desc	            |  暂时不支持 |
| BC_REGISTER_LOOPER	          | 无参数	                    |  创建新的 Looper 线程 |
| BC_ENTER_LOOPER	              | 无参数	                    |  应用线程进入 Looper |
| BC_EXIT_LOOPER	              | 无参数	                    |  应用线程退出 Looper |
| BC_REQUEST_DEATH_NOTIFICATION   | binder_handle_cookie	    | 注册死亡通知 |
| BC_CLEAR_DEATH_NOTIFICATION	  | binder_handle_cookie	    | 取消注册的死亡通知 |
| BC_DEAD_BINDER_DONE	          | binder_uintptr_t	        | 已经完成的死亡通知 |
| BC_TRANSACTION_SG	              | binder_transaction_data_sg  | Client 向 Binder 驱动发送的 Command |
| BC_REPLY_SG	                  | binder_transaction_data_sg  | Server 向 Binder 驱动发送的 Command |

# BR 响应码（BR_PROTOCOL）

Binder 响应码，在 `binder_driver_return_protocol` 中定义，是 binder 设备向应用程序回复的消息，应用程序包括 client 和 server 端，以 BR_ 开头，总共 18 条

| 响应码 | 参数类型 | 作用 |
|-------|----------|-----|
| BR_ERROR                         | __s32                   | 操作发送错误 | 
| BR_OK                            | 无参数                   | 操作完成 | 
| BR_TRANSACTION                   | binder_transaction_data | Binder 驱动向 Server 发送的请求数据 | 
| BR_REPLY                         | binder_transaction_data | Binder 驱动向 Client 发送的回复数据 | 
| BR_ACQUIRE_RESULT                | __s32                   | 暂时不支持 | 
| BR_DEAD_REPLY                    | 无参数                   | 回复失败，线程或节点为空 | 
| BR_TRANSACTION_COMPLETE          | 无参数                   | 对请求发送的成功反馈 | 
| BR_INCREFS                       | binder_ptr_cookie       | binder_ref 弱引用加1操作 | 
| BR_ACQUIRE                       | binder_ptr_cookie       | binder_ref 弱引用减1操作 | 
| BR_RELEASE                       | binder_ptr_cookie       | binder_ref 强引用加1操作 | 
| BR_DECREFS                       | binder_ptr_cookie       | binder_ref 强引用减1操作 | 
| BR_ATTEMPT_ACQUIRE               | binder_pri_ptr_cookie   | 暂时不支持 | 
| BR_NOOP                          | 无参数                   | 不做任何事情 | 
| BR_SPAWN_LOOPER                  | 无参数                   | 创建新的 Looper 线程 | 
| BR_FINISHED                      | 无参数                   | 暂时不支持 | 
| BR_DEAD_BINDER                   | binder_uintptr_t        | Binder 驱动向 client 发送死亡通知 | 
| BR_CLEAR_DEATH_NOTIFICATION_DONE | binder_uintptr_t        | 清除死亡通知 | 
| BR_FAILED_REPLY                  | 无参数                   | 回复失败，transaction 出错导致 | 

# binder_ioctl_write_read

> ioctl - control device
> 
> `int ioctl(int fd, unsigned long request, ...);`
> 
> The ioctl() system call manipulates the underlying device  
> parameters of special files.  In particular, many operating  
> characteristics of character special files (e.g., terminals) may  
> be controlled with ioctl() requests.  The argument `fd` must be an  
> open file descriptor.  
> 
> The second argument is a `device-dependent request code`.  The  
> third argument is `an untyped pointer to memory`.
> 
> Ioctl command values are 32-bit constants.  In principle these  
> constants are completely arbitrary, but people have tried to  
> build some structure into them.
> 
> Later (0.98p5) some more information was built into the number.  
> One has 2 direction bits (00: none, 01: write, 10: read, 11:  
> read/write) followed by 14 size bits (giving the size of the  
> argument), followed by an 8-bit type (collecting the ioctls in  
> groups for a common purpose or a common driver), and an 8-bit  
> serial number.
> 
> The macros describing this structure live in <asm/ioctl.h> and  
> are _IO(type,nr) and {_IOR,_IOW,_IOWR}(type,nr,size).  They use  
> sizeof(size) so that size is a misnomer here: this third argument  
> is a data type.

ioctl cmd（也就是第二个参数 request）是 32 bits 的：
1. 前 2 bits 表示方向：写入还是读出，可以用宏 `_IOC_DIR(cmd)` 解析出来：
    1. _IOC_NONE - 没有数据传输
	2. _IOC_READ - 从设备中读取数据，驱动程序必须向用户空间写入数据
	3. _IOC_WRITE - 向设备写入数据，驱动程序必须从用户空间读入数据
	4. _IOC_READ | _IOC_WRITE
2. 接着 14 bits 表示第三个参数（指针）所指的内容的长度，可以用宏 `_IOC_SIZE(cmd)` 解析
3. 接着 8 bits 表示一个幻数（magic number），可以用宏 `_IOC_TYPE(cmd)` 解析
4. 最后 8 bits 表示命令（的序号），可以用宏 `_IOC_SIZE(cmd)` 解析

> `unsigned long copy_from_user (void * to, const void __user * from, unsigned long n);`
> 
> to - Destination address, in kernel space.  
> from - Source address, in user space.  
> n- Number of bytes to copy.  
> 
> Description
> Copy data from user space to kernel space.  
> Returns number of bytes that could not be copied. On success, this will be zero.  
> If some data could not be copied, this function will pad the copied data to the requested size using zero bytes.

用户空间程序和 Binder 驱动程序交互基本都是通过 `ioctl(binder_fd, BINDER_WRITE_READ, p_bwr)`，通讯协议/内存布局为 `struct binder_write_read` 如下：

1. write_buffer：用于发送IPC(或IPC reply)数据，即传递经由Binder Driver的数据时使用。

2. read_buffer 变量：用于接收来自Binder Driver的数据，即Binder Driver在接收IPC(或IPC reply)数据后，保存到read_buffer，再传递到用户空间；
write_buffer和read_buffer都是包含Binder协议命令和binder_transaction_data结构体。

copy_from_user()将用户空间IPC数据拷贝到内核态binder_write_read结构体；
copy_to_user()将用内核态binder_write_read结构体数据拷贝到用户空间；

|        类型       |    成员变量    |         解释         |
|------------------|----------------|---------------------|
| binder_uintptr_t |  write_buffer  | 写缓冲数据的指针       |
|  binder_size_t   |   write_size   | write_buffer的总字节数     |
|  binder_size_t   | write_consumed | write_buffer已消费的字节数  |
| binder_uintptr_t |  read_buffer   | 读缓存数据的指针       |
|  binder_size_t   |   read_size    | read_buffer的总字节数      |
|  binder_size_t   | read_consumed  | read_buffer已消费的字节数   |

```cpp
// ioctl handler 的三个参数分别对应 syscall ioctl 的三个参数
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {...}
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);  // 上面说过 _IOC_SIZE 用来解析 cmd 里包含的，arg 指向内容的长度
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;

    // 将 ioctl 参数 struct binder_write_read 从用户空间 ubuf 拷贝到内核空间 bwr
	if (size != sizeof(struct binder_write_read)) {  // arg 的长度必须是 struct binder_write_read 的长度
		ret = -EINVAL;
		goto out;
	}
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {  // 将用户空间的 ubuf 拷贝到内核空间 bwr
		ret = -EFAULT;
		goto out;
	}
	binder_debug(BINDER_DEBUG_READ_WRITE,
		     "%d:%d write %lld at %016llx, read %lld at %016llx\n",
		     proc->pid, thread->pid,
		     (u64)bwr.write_size, (u64)bwr.write_buffer,
		     (u64)bwr.read_size, (u64)bwr.read_buffer);
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
		case BC_INCREFS:
		case BC_ACQUIRE:
		case BC_RELEASE:
		case BC_DECREFS:
		case BC_TRANSACTION:
		case BC_REPLY:
		case BC_REGISTER_LOOPER:
		case BC_ENTER_LOOPER:
		case BC_EXIT_LOOPER:
		// ...
		}
		*consumed = ptr - buffer;
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
		switch (w->type) {
		case BINDER_WORK_TRANSACTION:
		case BINDER_WORK_RETURN_ERROR:
		case BINDER_WORK_TRANSACTION_COMPLETE:
		case BINDER_WORK_TRANSACTION_ONEWAY_SPAM_SUSPECT:
		case BINDER_WORK_NODE:
		case BINDER_WORK_DEAD_BINDER:
		case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
		case BINDER_WORK_CLEAR_DEATH_NOTIFICATION:
		// ...
		}
		if (!t)
			continue;
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
			     "%d:%d %s %d %d:%d, cmd %d size %zd-%zd ptr %016llx-%016llx\n",
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
	}
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
```

# 

```cpp
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply,
			       binder_size_t extra_buffers_size)
{
	for (buffer_offset = off_start_offset; buffer_offset < off_end_offset;
	     buffer_offset += sizeof(binder_size_t)) {
		struct binder_object_header *hdr;
		size_t object_size;
		struct binder_object object;
		binder_size_t object_offset;
		binder_size_t copy_size;

        // 将 [buffer + buffer_offset, buffer + buffer_offset + bytes] 拷贝到 dest
		// int binder_alloc_copy_from_buffer(struct binder_alloc *alloc,
		// 		  void *dest,
		// 		  struct binder_buffer *buffer,
		// 		  binder_size_t buffer_offset,
		// 		  size_t bytes)
		if (binder_alloc_copy_from_buffer(&target_proc->alloc,
						  &object_offset,        // dest
						  t->buffer,             // buffer
						  buffer_offset,         // buffer_offset
						  sizeof(object_offset)  // bytes
			)) {
			return_error = BR_FAILED_REPLY;
			return_error_param = -EINVAL;
			return_error_line = __LINE__;
			goto err_bad_offset;
		}

        // binder_get_object(struct binder_proc *proc,
        //     const void __user *u,
        //     struct binder_buffer *buffer,
        //     unsigned long offset,
        //     struct binder_object *object)
		object_size = binder_get_object(target_proc, user_buffer,
				t->buffer, object_offset, &object);

		hdr = &object.hdr;
		switch (hdr->type) {
		case BINDER_TYPE_PTR: {
			struct binder_buffer_object *bp =
				to_binder_buffer_object(hdr);
			size_t buf_left = sg_buf_end_offset - sg_buf_offset;
			size_t num_valid;
			if (bp->length > buf_left) {
				binder_user_error("%d:%d got transaction with too large buffer\n",
						  proc->pid, thread->pid);
				return_error = BR_FAILED_REPLY;
				return_error_param = -EINVAL;
				return_error_line = __LINE__;
				goto err_bad_offset;
			}
			ret = binder_defer_copy(&sgc_head, sg_buf_offset,
				(const void __user *)(uintptr_t)bp->buffer,
				bp->length);
			if (ret) {
				return_error = BR_FAILED_REPLY;
				return_error_param = ret;
				return_error_line = __LINE__;
				goto err_translate_failed;
			}
			/* Fixup buffer pointer to target proc address space */
			bp->buffer = (uintptr_t)
				t->buffer->user_data + sg_buf_offset;
			sg_buf_offset += ALIGN(bp->length, sizeof(u64));
			num_valid = (buffer_offset - off_start_offset) /
					sizeof(binder_size_t);
			ret = binder_fixup_parent(&pf_head, t,
						  thread, bp,
						  off_start_offset,
						  num_valid,
						  last_fixup_obj_off,
						  last_fixup_min_off);
			if (ret < 0 ||
			    binder_alloc_copy_to_buffer(&target_proc->alloc,
							t->buffer,
							object_offset,
							bp, sizeof(*bp))) {
				return_error = BR_FAILED_REPLY;
				return_error_param = ret;
				return_error_line = __LINE__;
				goto err_translate_failed;
			}
			last_fixup_obj_off = object_offset;
			last_fixup_min_off = 0;
		} break;
		case BINDER_TYPE_BINDER:
		case BINDER_TYPE_WEAK_BINDER:
		case BINDER_TYPE_HANDLE:
		case BINDER_TYPE_WEAK_HANDLE:
		case BINDER_TYPE_FD:
		case BINDER_TYPE_FDA:
		// ...
	    }
}
```

# startActivity

1. `startActivity` 最终走到 `IActivityTaskManager.startActivity`，[AIDL](../../../../2022/06/08/binder-aidl/) 的相关知识让我们可以从名字上看出这是一个 binder ipc 调用，会有一个 `IActivityTaskManager.aidl` 文件，编译出对应的 Interface 和 Stub 类，通过 `IActivityTaskManager.Stub.asInterface()` 获得接口实现

2. 但在此之前需要通过 `ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE)` 获得 IActivityTaskManager 服务（Binder）

3.  ServiceManager 内部是通过 `IServiceManager.getService` 来查找各种服务的，那 IServiceManager 服务的 Binder 又从哪里获得呢？答案是 `BinderInternal.getContextObject()`
    - 它返回一个 `BinderProxy` 对象，所有的 `IBinder` 操作都由 `BinderProxy.mNativeData` 指向的 native 层 `BpBinder` 实例来实现
	- native 层也有 `IBinder` 接口对应 java 层的 `IBinder` 接口，BpBinder 实现了 IBinder 负责向 binder driver 传输数据
	- 特别之处在于 IServiceManager 对应的 BpBinder 它的 `mHandle` 是 0

4. 通过 `IServiceManager.Stub.asInterface(remote)` 将 BinderProxy(IBinder) 包装为 IServiceManager 实现，但所有接口实现（IPC 操作）最终都由 native 层的 BpBinder 实现：`boolean IBinder.transact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags)` 对应 `status_t BpBinder::transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)`

```java
ContextImpl.startActivity(Intent intent, Bundle options)
Instrumentation.execStartActivity(
	Context who, IBinder contextThread, IBinder token, Activity target, 
	Intent intent, int requestCode, Bundle options) {
	// ...
    int result = ActivityTaskManager.getService().startActivity(whoThread,
            who.getOpPackageName(), who.getAttributionTag(), intent,
            intent.resolveTypeIfNeeded(who.getContentResolver()), token,
            target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);	
}

public class ActivityTaskManager {
    public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }

    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);    // AIDL 那篇讲过同进程直接返回 Stub 实现类，跨进程是返回 Stub.Proxy，底层依靠 binder 进行 IPC
                }
            };	
}

public class ServiceManager {
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
        final IBinder binder = getIServiceManager().getService(name);
		// ...
        return binder;
    }	

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

public class BinderInternal {
    /**
     * Return the global "context object" of the system.  This is usually
     * an implementation of IServiceManager, which you can use to find
     * other services.
     */
    @UnsupportedAppUsage
    public static final native IBinder getContextObject();	
}

public final class ServiceManagerNative {
    private ServiceManagerNative() {}

    /**
     * Cast a Binder object into a service manager interface, generating
     * a proxy if needed.
     *
     * TODO: delete this method and have clients use
     *     IServiceManager.Stub.asInterface instead
     */
    @UnsupportedAppUsage
    public static IServiceManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }

        // ServiceManager is never local
        return new ServiceManagerProxy(obj);
    }
}

// This class should be deleted and replaced with IServiceManager.Stub whenever
// mRemote is no longer used
class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
        mServiceManager = IServiceManager.Stub.asInterface(remote);
    }

    @UnsupportedAppUsage
    public IBinder getService(String name) throws RemoteException {
        // Same as checkService (old versions of servicemanager had both methods).
        return mServiceManager.checkService(name);
    }
}
```

```cpp
// android_util_Binder.cpp
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);  // 得到一个 handle == 0 的 BpBinder
    return javaObjectForIBinder(env, b);                           // 创建并返回 java BinderProxy 对象
}

// ProcessState.cpp
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    sp<IBinder> context = getStrongProxyForHandle(0);

    if (context) {
        // The root object is special since we get it directly from the driver, it is never
        // written by Parcell::writeStrongBinder.
        internal::Stability::markCompilationUnit(context.get());
    } else {
        ALOGW("Not able to get context object on %s.", mDriverName.c_str());
    }

    return context;
}

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)  // handle = 0
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    if (handle == 0 && the_context_object != nullptr) return the_context_object;

    handle_entry* e = lookupHandleLocked(handle);

    if (e != nullptr) {
        // We need to create a new BpBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  The
        // attemptIncWeak() is safe because we know the BpBinder destructor will always
        // call expungeHandle(), which acquires the same lock we are holding now.
        // We need to do this because there is a race condition between someone
        // releasing a reference on this BpBinder, and a new reference on its handle
        // arriving from the driver.
        IBinder* b = e->binder;
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                // Special case for context manager...
                // The context manager is the only object for which we create
                // a BpBinder proxy without already holding a reference.
                // Perform a dummy transaction to ensure the context manager
                // is registered before we create the first local reference
                // to it (which will occur when creating the BpBinder).
                // If a local reference is created for the BpBinder when the
                // context manager is not present, the driver will fail to
                // provide a reference to the context manager, but the
                // driver API does not return status.
                //
                // Note that this is not race-free if the context manager
                // dies while this code runs.

                IPCThreadState* ipc = IPCThreadState::self();

                CallRestriction originalCallRestriction = ipc->getCallRestriction();
                ipc->setCallRestriction(CallRestriction::NONE);

                Parcel data;
                status_t status = ipc->transact(
                        0, IBinder::PING_TRANSACTION, data, nullptr, 0);

                ipc->setCallRestriction(originalCallRestriction);

                if (status == DEAD_OBJECT)
                   return nullptr;
            }

            sp<BpBinder> b = BpBinder::PrivateAccessor::create(handle);  // 创建 handle == 0 的 BpBinder
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

class PrivateAccessor {
    static sp<BpBinder> create(int32_t handle) { return BpBinder::create(handle); }
}

sp<BpBinder> BpBinder::create(int32_t handle) {
    int32_t trackedUid = -1;
    if (sCountByUidEnabled) {
        trackedUid = IPCThreadState::self()->getCallingUid();
        AutoMutex _l(sTrackingLock);
        uint32_t trackedValue = sTrackingMap[trackedUid];
        if (CC_UNLIKELY(trackedValue & LIMIT_REACHED_MASK)) {
            if (sBinderProxyThrottleCreate) {
                return nullptr;
            }
            trackedValue = trackedValue & COUNTING_VALUE_MASK;
            uint32_t lastLimitCallbackAt = sLastLimitCallbackMap[trackedUid];

            if (trackedValue > lastLimitCallbackAt &&
                (trackedValue - lastLimitCallbackAt > sBinderProxyCountHighWatermark)) {
                ALOGE("Still too many binder proxy objects sent to uid %d from uid %d (%d proxies "
                      "held)",
                      getuid(), trackedUid, trackedValue);
                if (sLimitCallback) sLimitCallback(trackedUid);
                sLastLimitCallbackMap[trackedUid] = trackedValue;
            }
        } else {
            if ((trackedValue & COUNTING_VALUE_MASK) >= sBinderProxyCountHighWatermark) {
                ALOGE("Too many binder proxy objects sent to uid %d from uid %d (%d proxies held)",
                      getuid(), trackedUid, trackedValue);
                sTrackingMap[trackedUid] |= LIMIT_REACHED_MASK;
                if (sLimitCallback) sLimitCallback(trackedUid);
                sLastLimitCallbackMap[trackedUid] = trackedValue & COUNTING_VALUE_MASK;
                if (sBinderProxyThrottleCreate) {
                    ALOGI("Throttling binder proxy creates from uid %d in uid %d until binder proxy"
                          " count drops below %d",
                          trackedUid, getuid(), sBinderProxyCountLowWatermark);
                    return nullptr;
                }
            }
        }
        sTrackingMap[trackedUid]++;
    }
    return sp<BpBinder>::make(BinderHandle{handle}, trackedUid);  // 创建 handle == 0 的 BpBinder
}

// If the argument is a JavaBBinder, return the Java object that was used to create it.
// Otherwise return a BinderProxy for the IBinder. If a previous call was passed the
// same IBinder, and the original BinderProxy is still alive, return the same BinderProxy.
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    // N.B. This function is called from a @FastNative JNI method, so don't take locks around
    // calls to Java code or block the calling thread for a long time for any reason.

    if (val == NULL) return NULL;

    if (val->checkSubclass(&gBinderOffsets)) {
        // It's a JavaBBinder created by ibinderForJavaObject. Already has Java object.
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        LOGDEATH("objectForBinder %p: it's our own %p!\n", val.get(), object);
        return object;
    }

    BinderProxyNativeData* nativeData = new BinderProxyNativeData();
    nativeData->mOrgue = new DeathRecipientList;
    nativeData->mObject = val;

	// 调用 java 层代码：android.os.BinderProxy#getInstance(long nativeData, long iBinder) 得到一个 BinderProxy 对象
    jobject object = env->CallStaticObjectMethod(gBinderProxyOffsets.mClass,
            gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get());
    if (env->ExceptionCheck()) {
        // In the exception case, getInstance still took ownership of nativeData.
        return NULL;
    }
    BinderProxyNativeData* actualNativeData = getBPNativeData(env, object);
    if (actualNativeData == nativeData) {
        // Created a new Proxy
        uint32_t numProxies = gNumProxies.fetch_add(1, std::memory_order_relaxed);
        uint32_t numLastWarned = gProxiesWarned.load(std::memory_order_relaxed);
        if (numProxies >= numLastWarned + PROXY_WARN_INTERVAL) {
            // Multiple threads can get here, make sure only one of them gets to
            // update the warn counter.
            if (gProxiesWarned.compare_exchange_strong(numLastWarned,
                        numLastWarned + PROXY_WARN_INTERVAL, std::memory_order_relaxed)) {
                ALOGW("Unexpectedly many live BinderProxies: %d\n", numProxies);
            }
        }
    } else {
        delete nativeData;
    }

    return object;  // 返回 java BinderProxy 对象
}
```

# BinderProxy

> std::variant 是 C++ 17 所提供的变体类型，`variant<X, Y, Z>` 是可存放 X, Y, Z 这三种类型数据的变体类型
> 
> 与C语言中传统的 union 类型相同的是，variant 也是联合(union)类型。即 variant 可以存放多种类型的数据，但任何时刻最多只能存放其中一种类型的数据  
> 与C语言中传统的 union 类型所不同的是，variant 是可辨识的类型安全的联合(union)类型。即 variant 无须借助外力只需要通过查询自身就可辨别实际所存放数据的类型  
> 
> `v = variant<int, double, std::string>`，则 v 是一个可存放 int, double, std::string 这三种类型数据的变体类型的对象
> 
> `v.index()`                    返回变体类型 v 实际所存放数据的类型的下标，变体中第 1 种类型下标为 0，第 2 种类型下标为 1，以此类推  
> `std::holds_alternative<T>(v)` 可查询变体类型 v 是否存放了 T 类型的数据  
> `std::get<I>(v)`               如果变体类型 v 存放的数据类型下标为 I，那么返回所存放的数据，否则报错  
> `std::get_if<I>(&v)`           如果变体类型 v 存放的数据类型下标为 I，那么返回所存放数据的指针，否则返回空指针  
> `std::get<T>(v)`               如果变体类型 v 存放的数据类型为 T，那么返回所存放的数据，否则报错  
> `std::get_if<T>(&v)`           如果变体类型 v 存放的数据类型为 T，那么返回所存放数据的指针，否则返回空指针

```cpp
public final class BinderProxy implements IBinder {
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        // ...
        return transactNative(code, data, reply, flags);
    }

    public native boolean transactNative(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException;	
}

// frameworks/base/core/jni/android_util_Binder.cpp
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    if (dataObj == NULL) {
        jniThrowNullPointerException(env, NULL);
        return JNI_FALSE;
    }

    Parcel* data = parcelForJavaObject(env, dataObj);
    if (data == NULL) {
        return JNI_FALSE;
    }
    Parcel* reply = parcelForJavaObject(env, replyObj);
    if (reply == NULL && replyObj != NULL) {
        return JNI_FALSE;
    }

    // 取 BinderProxy.mNativeData 的值，它是一个指向 BinderProxyNativeData 的指针
	// BinderProxyNativeData.mObject 是上一章节介绍的、mHandle == 0 的 BpBinder
    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    if (target == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
        return JNI_FALSE;
    }

    ALOGV("Java code calling transact on %p in Java object %p with code %" PRId32 "\n",
            target, obj, code);


    bool time_binder_calls;
    int64_t start_millis;
    if (kEnableBinderSample) {
        // Only log the binder call duration for things on the Java-level main thread.
        // But if we don't
        time_binder_calls = should_time_binder_calls();

        if (time_binder_calls) {
            start_millis = uptimeMillis();
        }
    }

    //printf("Transact from Java code to %p sending: ", target); data->print();
    status_t err = target->transact(code, *data, reply, flags);
    //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();

    if (kEnableBinderSample) {
        if (time_binder_calls) {
            conditionally_log_binder_call(start_millis, target, code);
        }
    }

    if (err == NO_ERROR) {
        return JNI_TRUE;
    } else if (err == UNKNOWN_TRANSACTION) {
        return JNI_FALSE;
    }

    signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/, data->dataSize());
    return JNI_FALSE;
}

// frameworks/native/libs/binder/BpBinder.cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        bool privateVendor = flags & FLAG_PRIVATE_VENDOR;
        // don't send userspace flags to the kernel
        flags = flags & ~static_cast<uint32_t>(FLAG_PRIVATE_VENDOR);

        // user transactions require a given stability level
        if (code >= FIRST_CALL_TRANSACTION && code <= LAST_CALL_TRANSACTION) {
            using android::internal::Stability;

            int16_t stability = Stability::getRepr(this);
            Stability::Level required = privateVendor ? Stability::VENDOR
                : Stability::getLocalLevel();

            if (CC_UNLIKELY(!Stability::check(stability, required))) {
                ALOGE("Cannot do a user transaction on a %s binder (%s) in a %s context.",
                      Stability::levelString(stability).c_str(),
                      String8(getInterfaceDescriptor()).c_str(),
                      Stability::levelString(required).c_str());
                return BAD_TYPE;
            }
        }

        status_t status;
        if (CC_UNLIKELY(isRpcBinder())) {  // false
            status = rpcSession()->transact(sp<IBinder>::fromExisting(this), code, data, reply, flags);
        } else {  // ServiceManager 对应的 BpBinder.mHandle 是 BinderHandle 且其 handle == 0
            status = IPCThreadState::self()->transact(binderHandle(), code, data, reply, flags);
        }
        if (data.dataSize() > LOG_TRANSACTIONS_OVER_SIZE) {
            Mutex::Autolock _l(mLock);
            ALOGW("Large outgoing transaction of %zu bytes, interface descriptor %s, code %d",
                  data.dataSize(),
                  mDescriptorCache.size() ? String8(mDescriptorCache).c_str()
                                          : "<uncached descriptor>",
                  code);
        }

        if (status == DEAD_OBJECT) mAlive = 0;

        return status;
    }

    return DEAD_OBJECT;
}

// 判断 mHandle 是否是 RpcHandle
bool BpBinder::isRpcBinder() const {
    return std::holds_alternative<RpcHandle>(mHandle);
}

// frameworks/native/libs/binder/include/binder/BpBinder.h
class BpBinder : public IBinder
{
private:
    using Handle = std::variant<BinderHandle, RpcHandle>;

    BpBinder(BinderHandle&& handle, int32_t trackedUid);
    Handle mHandle;	
}

// frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    LOG_ALWAYS_FATAL_IF(data.isForRpc(), "Parcel constructed for RPC, but being used with binder.");

    status_t err;

    flags |= TF_ACCEPT_FDS;

    IF_LOG_TRANSACTIONS() {
        TextOutput::Bundle _b(alog);
        alog << "BC_TRANSACTION thr " << (void*)pthread_self() << " / hand "
            << handle << " / code " << TypeCode(code) << ": "
            << indent << data << dedent << endl;
    }

    LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
        (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }

    if ((flags & TF_ONE_WAY) == 0) {  // 一般为 false
        if (UNLIKELY(mCallRestriction != ProcessState::CallRestriction::NONE)) {
            if (mCallRestriction == ProcessState::CallRestriction::ERROR_IF_NOT_ONEWAY) {
                ALOGE("Process making non-oneway call (code: %u) but is restricted.", code);
                CallStack::logStack("non-oneway call", CallStack::getCurrent(10).get(),
                    ANDROID_LOG_ERROR);
            } else /* FATAL_IF_NOT_ONEWAY */ {
                LOG_ALWAYS_FATAL("Process may not make non-oneway calls (code: %u).", code);
            }
        }

        #if 0
        if (code == 4) { // relayout
            ALOGI(">>>>>> CALLING transaction 4");
        } else {
            ALOGI(">>>>>> CALLING transaction %d", code);
        }
        #endif
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        #if 0
        if (code == 4) { // relayout
            ALOGI("<<<<<< RETURNING transaction 4");
        } else {
            ALOGI("<<<<<< RETURNING transaction %d", code);
        }
        #endif

        IF_LOG_TRANSACTIONS() {
            TextOutput::Bundle _b(alog);
            alog << "BR_REPLY thr " << (void*)pthread_self() << " / hand "
                << handle << ": ";
            if (reply) alog << indent << *reply << dedent << endl;
            else alog << "(none requested)" << endl;
        }
    } else {
        err = waitForResponse(nullptr, nullptr);
    }

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

status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)  // reply = null, acquireResult == null
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        IF_LOG_COMMANDS() {
            alog << "Processing waitForResponse Command: "
                << getReturnString(cmd) << endl;
        }

        switch (cmd) {
        case BR_ONEWAY_SPAM_SUSPECT:
            ALOGE("Process seems to be sending too many oneway calls.");
            CallStack::logStack("oneway spamming", CallStack::getCurrent().get(),
                    ANDROID_LOG_ERROR);
            [[fallthrough]];
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;

        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;
            goto finish;

        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;

        case BR_FROZEN_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;

        case BR_ACQUIRE_RESULT:
            {
                ALOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");
                const int32_t result = mIn.readInt32();
                if (!acquireResult) continue;
                *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
            }
            goto finish;

        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer);
                    } else {
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(nullptr,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t));
                    }
                } else {
                    freeBuffer(nullptr,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t));
                    continue;
                }
            }
            goto finish;

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
```

# 参考

1. [深入理解 Binder 机制 5 - binder 驱动分析](https://skytoby.github.io/2020/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Binder%E6%9C%BA%E5%88%B65-binder%E9%A9%B1%E5%8A%A8%E5%88%86%E6%9E%90/)