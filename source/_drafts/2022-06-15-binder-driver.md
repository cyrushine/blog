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

```cpp
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
	if (is_binderfs_device(nodp)) {
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
	filp->private_data = proc;
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