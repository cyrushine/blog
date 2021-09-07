
[Logan](https://github.com/Meituan-Dianping/Logan) 是 [美团点评技术团队](https://github.com/meituan) 开源的包含前端 SDK 和后端 Server 的一整套日志系统，也是公司日志库 VLog 的基础

Logan Android SDK 提供了这么几个 API：

| API | Description |
|-----|-------------|
| Logan#w(log, type)             | 写日志（严谨地说应该是发送日志请求，因为日志是放在消息队列里等待被处理的） |
| Logan#init                     | 初始化 |
| Logan#f                        | Logan 内部有个内存缓存（memory/mmap），日志首先被写入到缓存里，只有到达一定大小时（1/3）才写入文件，这里请求立刻写入到文件里去 |
| Logan#s                        | 根据日期 获取/发送 日志文件 |
| Logan#getAllFilesInfo          | 获取所有的日志文件，key 是日期，value 是日志文件大小 |
| Logan#setDebug                 | 设置为 debug 模式后，会有更加详细的 native 日志，但是默认实现只是输出到 stdout 没有写入 android log |
| Logan#setOnLoganProtocolStatus | 可以拿到一些 Java 的关键日志 |

LoganConfig 是初始化配置参数：

| Field | Description |
|-------|-------------|
| path         | 存放日志文件的目录，日志是按日期（天）存放的，文件名是当天零时零分零秒的时间戳 |
| cachePath    | 内存缓存对应的 mmap 文件所在的目录 |
| maxFile      | 当日志文件超过此大小时，就不能再继续往 buffer 里写入日志 |
| day          | 只保留 n 天内的日志文件，旧的都删掉 |
| minSDCard    | 当可用的存储容量超过此阈值时才写入日志 |
| encryptKey16 | AES 加密参数 KEY |
| encryptIv16  | AES 加密参数 IV |


### 日志队列与 生产者-消费者 模型

调用 `Logan.w` 的线程是日志的生产者，日志写入请求被放入日志队列（`Queue`）里等待处理，`LoganThread` 线程作为消费者不断地执行日志队列里的任务

```java
public class Logan {

    /**
     * @param log  表示日志内容
     * @param type 表示日志类型
     * @brief Logan写入日志
     */
    public static void w(String log, int type) {
        if (sLoganControlCenter == null) {
            throw new RuntimeException("Please initialize Logan first");
        }
        sLoganControlCenter.write(log, type);
    }
}

class LoganControlCenter {

    private ConcurrentLinkedQueue<LoganModel> mCacheLogQueue = new ConcurrentLinkedQueue<>();
    private LoganThread mLoganThread;

    void write(String log, int flag) {
        if (TextUtils.isEmpty(log)) {
            return;
        }
        LoganModel model = new LoganModel();
        model.action = LoganModel.Action.WRITE;
        WriteAction action = new WriteAction();
        String threadName = Thread.currentThread().getName();
        long threadLog = Thread.currentThread().getId();
        boolean isMain = false;
        if (Looper.getMainLooper() == Looper.myLooper()) {
            isMain = true;
        }
        action.log = log;
        action.localTime = System.currentTimeMillis();
        action.flag = flag;
        action.isMainThread = isMain;
        action.threadId = threadLog;
        action.threadName = threadName;
        model.writeAction = action;
        if (mCacheLogQueue.size() < mMaxQueue) {
            mCacheLogQueue.add(model);
            if (mLoganThread != null) {
                mLoganThread.notifyRun();
            }
        }
    }

    private void init() {
        if (mLoganThread == null) {
            mLoganThread = new LoganThread(mCacheLogQueue, mCachePath, mPath, mSaveTime,
                    mMaxLogFile, mMinSDCard, mEncryptKey16, mEncryptIv16);
            mLoganThread.setName("logan-thread");
            mLoganThread.start();
        }
    }    
}

class LoganThread extends Thread {

    @Override
    public void run() {
        super.run();
        while (mIsRun) {
            synchronized (sync) {
                mIsWorking = true;
                try {
                    LoganModel model = mCacheLogQueue.poll();
                    if (model == null) {
                        mIsWorking = false;
                        sync.wait();
                        mIsWorking = true;
                    } else {
                        action(model);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    mIsWorking = false;
                }
            }
        }
    }
}
```

### 内存缓存 Buffer

并不是每次日志请求都立刻写入到日志文件里，而是在内存中开辟一段缓存（默认为 150K）作为 buffer，当 buffer 里的数据积累得足够多时（1/3 buffer 大小）才写入文件

```cpp
LoganThread.doWriteLog2File
LoganProtocol.logan_write
CLoganProtocol.logan_write
CLoganProtocol.clogan_write
Java_com_dianping_logan_CLoganProtocol_clogan_1write
clogan_write
clogan_write_section

void clogan_write2(char *data, int length) {
    if (NULL != logan_model && logan_model->is_ok) {
        clogan_zlib_compress(logan_model, data, length);    // 压缩和加密后的数据放在内存 buffer 里
        update_length_clogan(logan_model);
        int is_gzip_end = 0;

        } else if (buffer_type == LOGAN_MMAP_MMAP &&
                   logan_model->total_len >=
                   buffer_length / LOGAN_WRITEPROTOCOL_DEVIDE_VALUE) {  // 只有当数据大小到达阈值（1/3 buffer 容量）时才写入文件
            isWrite = 1;
            printf_clogan("clogan_write2 > write type MMAP \n");
        }

        if (isWrite) {  // 写入文件
            write_flush_clogan();
        
    }
}
```

#### mmap

