### 进程的 `priority` 和 `nice`

`priority` 是比较好理解的，即进程的 **优先级**，或者通俗点说就是程序被 CPU 执行的先后顺序，此值越小进程的优先级别越高

`nice` 表示进程可被执行的优先级的 **修正数值**。如前面所说，`priority` 越小越快被执行，那么加入 `nice` 值后，将会变为 `priority = priority + nice`。由此看出 `nice` 越小优先权更大，即其优先级会变高，则其越快被执行

在 LINUX 系统中，`nice` 的范围从 `-20` 到 `+19`（不同系统的值范围是不一样的），正值表示低优先级，负值表示高优先级，值为零则表示不会调整该进程的优先级。`nice` 为 `-20` 使得一项任务变得非常重要；与之相反，如果任务的 `nice` 为 `+19`，则表示它是一个高尚的、无私的任务，允许所有其他任务比自己享有宝贵的 CPU 时间的更大使用份额，这也就是 `nice` 的名称的来意。

进程在创建时被赋予不同的 `priority`，而如前面所说 `nice` 是表示进程优先级值可被修正数据值，因此每个进程都在其计划执行时被赋予一个 `nice` 值，这样系统就可以根据系统的资源以及具体进程的各类资源消耗情况，主动干预进程的优先级值

### `/proc/stat`

CPU 使用时间的统计（从系统启动开始累计到当前时刻），单位为 `jiffies`

> `jiffies` 是内核中的一个全局变量，用来记录自系统启动以来产生的 **节拍数**，在 linux 中一个节拍大致可理解为操作系统进程调度的最小时间片，不同 linux 内核可能值有不同，通常在 1ms 到 10ms 之间

| 指标 | 含义 |
|------------|------|
| user       | 处于用户态的运行时间（nice <= 0 的进程）                                |
| nice       | 处于用户态的运行时间（nice > 0 的进程）                                |
| system     | 处于核心态的运行时间                                                   |
| idle       | 除 IO 等待时间以外的其它等待时间                                        |
| iowait     | IO等待时间                                                            |
| irq        | 硬中断时间                                                            |	
| softirq    | 软中断时间                                                            |	
| steal      | 被盗时间，虚拟化环境中运行其他操作系统上花费的时间（since Linux 2.6.11）  |
| guest      | 来宾时间，操作系统运行虚拟CPU花费的时间(since Linux 2.6.24)              |
| guest_nice | nice 来宾时间，运行一个带 nice 值的 guest 花费的时间(since Linux 2.6.33) |

那么总的 CPU 时间就是上述所有时间的总和

```shell
cpu  912913 130268 732403 2159286 7587 128906 41192 0 0 0
cpu0 175242 29825 159630 1304196 6682 46130 15201 0 0 0
cpu1 179914 28612 161382 100430 165 27380 7497 0 0 0
cpu2 181285 28765 158788 102001 160 28297 7772 0 0 0
cpu3 117670 14751 130352 104336 106 20836 8165 0 0 0
cpu4 73493 7615 33335 136160 164 1756 712 0 0 0
cpu5 80026 7980 38607 134985 128 1983 734 0 0 0
cpu6 81584 8231 38008 132643 130 1972 737 0 0 0
cpu7 23695 4485 12299 144532 50 547 370 0 0 0
...
```

### `/proc/{pid}/stat`

该文件包含了某一进程所有的活动的信息，该文件中的所有值都是从系统启动开始累计到当前时刻

进程的总 cpu 时间 = utime + stime + cutime + cstime，单位为 `jiffies`

| 参数 | 数值 | 解释 |
|------|------|-----|
| pid         | 15803             | 进程号                                                              |
| comm        | (ng.doraemondemo) | 应用程序或命令的名字                                                 |
| task_state  | S                 | 任务的状态: R(runnign), S(sleeping), T(stopped), Z(zombie), D(dead) |
| ppid        | 667               | 父进程 ID                                                           |
| pgid        | 667               | 进程组 ID                                                           |
| sid         | 0                 | 会话组 ID                                                           |
| tty_nr      | 0                 | 该任务的 tty 终端的设备号                                            |
| tty_pgrp    | -1                | 终端的进程组号，当前运行在该任务所在终端的前台任务的 PID                |
| task->flags | 1077952832        | 进程标志位，查看该任务的特性                                          |
| min_flt     | 953477            | 该任务不需要从硬盘拷数据而发生的缺页（次缺页）的次数                    |
| cmin_flt    | 857256            | 累计的该任务的所有的 waited-for 进程曾经发生的次缺页的次数目            |
| maj_flt     | 337               | 该任务需要从硬盘拷数据而发生的缺页（主缺页）的次数                      |
| cmaj_flt    | 1                 | 累计的该任务的所有的 waited-for 进程曾经发生的主缺页的次数目            |
| utime       | 59750             | 该任务在用户态运行的时间                                              |
| stime       | 19132             | 该任务在核心态运行的时间                                              |
| cutime      | 2915              | 所有已死线程在用户态运行的时间                                         |
| cstime      | 5748              | 所有已死在核心态运行的时间                                            |
| priority    | 20                | 动态优先级                                                           |
| nice        | 0                 | 静态优先级                                                           |

```shell
15803 (ng.doraemondemo) S 667 667 0 0 -1 1077952832 953477 857256 337 1 59750 19132 2915 5748 20 0 107 0 2996026 7388807168 40468 18446744073709551615 1 1 0 0 0 0 4612 1 1073779964 0 0 0 17 2 0 0 37 0 0 0 0 0 0 0 0 0 0
```

### 计算 CPU 使用率

1. 在足够短的时间间隔内对 `/proc/stat` 进行采样，记为 t1 和 t2
2. 计算总的 cpu 时间片：user + nice + system + idle + iowait + irq + softirq + stealstolen + guest
    1. 计算 t1 的总时间片 s1
    2. 计算 t2 的总时间片 s2
    3. 算得时间间隔内的 cpu 总时间片 total = s2 - s1
3. 计算时间间隔内的 cpu 空闲时间：idle = t1.idle - t2.idle
4. cpu 使用率：(total - idle) / total * 100

### 计算某进程的 CPU 使用率

1. 在足够短的时间间隔内对 `/proc/stat` 和 `/proc/<pid>/stat` 进行采用，记为 t1 和 t2
2. 参考上节计算总的 cpu 时间片：cpu = user + nice + system + idle + iowait + irq + softirq + stealstolen + guest
3. 计算进程的 cpu 时间片：proc = utime、stime、cutime、cstime
4. 计算进程的 cpu 使用率：(t2.proc - t1.proc) / (t2.cpu - t1.cpu) * 100

```java
/**
 * DoraemonKit 里 8.0 以下获取 cpu 使用率的方式
 *
 * @return
 */
private float getCPUData() {
    long cpuTime;
    long appTime;
    float value = 0.0f;
    try {
        if (mProcStatFile == null || mAppStatFile == null) {
            mProcStatFile = new RandomAccessFile("/proc/stat", "r");
            mAppStatFile = new RandomAccessFile("/proc/" + android.os.Process.myPid() + "/stat", "r");
        } else {
            mProcStatFile.seek(0L);
            mAppStatFile.seek(0L);
        }
        String procStatString = mProcStatFile.readLine();
        String appStatString = mAppStatFile.readLine();
        String procStats[] = procStatString.split(" ");
        String appStats[] = appStatString.split(" ");
        cpuTime = Long.parseLong(procStats[2]) + Long.parseLong(procStats[3])
                + Long.parseLong(procStats[4]) + Long.parseLong(procStats[5])
                + Long.parseLong(procStats[6]) + Long.parseLong(procStats[7])
                + Long.parseLong(procStats[8]);
        appTime = Long.parseLong(appStats[13]) + Long.parseLong(appStats[14]);
        if (mLastCpuTime == null && mLastAppCpuTime == null) {
            mLastCpuTime = cpuTime;
            mLastAppCpuTime = appTime;
            return value;
        }
        value = ((float) (appTime - mLastAppCpuTime) / (float) (cpuTime - mLastCpuTime)) * 100f;
        mLastCpuTime = cpuTime;
        mLastAppCpuTime = appTime;
    } catch (Exception e) {
        e.printStackTrace();
    }
    return value;
}
```

参考

1. [进程优先级，进程nice值和%nice的解释](https://www.cnblogs.com/yuanshuang/p/5573223.html)
2. [Linux平台Cpu使用率的计算](http://www.blogjava.net/fjzag/articles/317773.html)
3. [/proc文件系统（二）：/proc/<pid>/stat](https://www.cnblogs.com/Jimmy1988/p/10045601.html)








