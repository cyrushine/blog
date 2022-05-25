---
title: 深入 SharedPreferences：架构、缺点和优化
date: 2022-05-16 12:00:00 +0800
tags: [SharedPreferences, SP, MMKV, Jetpack-DataStore]
---

# 提问

* `apply` 和 `commit` 有何区别？
* SharedPreferences 是线程安全的吗？它是如何保证线程安全的？
* SharedPreferences 是进程安全的吗？
* 数据丢失时，其最终的屏障 —— 文件备份机制是如何实现的？
* 如何实现进程安全的 SharedPreferences
* SharedPreferences 导致 ANR 的原因

# 从磁盘加载

> 以下代码基于 Android 31

SharedPreferences 本质上是：
* 在磁盘上是一个 XML 文件：`/{app internal data dir}/shared_prefs/{name}.xml`
* 在内存中是一个 `Map<String, Object>`

构造 SharedPreferences 实例时会立即起一个线程来将磁盘上的 xml 文件加载并解析为内存中的 Map，在加载任务完成前，会通过 `mLock` 阻塞所有的 get/set 操作

所以在主线程上操作 SharedPreferences 是有可能阻塞主线程导致 ANR 的

```java
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    startLoadFromDisk();
}

private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}

private void loadFromDisk() {
    synchronized (mLock) {
        if (mLoaded) {
            return;
        }
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    // Debugging
    if (mFile.exists() && !mFile.canRead()) {
        Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
    }

    Map<String, Object> map = null;
    StructStat stat = null;
    Throwable thrown = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16 * 1024);
                map = (Map<String, Object>) XmlUtils.readMapXml(str);
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        // An errno exception means the stat failed. Treat as empty/non-existing by
        // ignoring.
    } catch (Throwable t) {
        thrown = t;
    }

    synchronized (mLock) {
        mLoaded = true;
        mThrowable = thrown;
        // It's important that we always signal waiters, even if we'll make
        // them fail with an exception. The try-finally is pretty wide, but
        // better safe than sorry.
        try {
            if (thrown == null) {
                if (map != null) {
                    mMap = map;
                    mStatTimestamp = stat.st_mtim;
                    mStatSize = stat.st_size;
                } else {
                    mMap = new HashMap<>();
                }
            }
            // In case of a thrown exception, we retain the old map. That allows
            // any open editors to commit and store updates.
        } catch (Throwable t) {
            mThrowable = t;
        } finally {
            mLock.notifyAll();
        }
    }
}

public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}

public Editor edit() {
    synchronized (mLock) {
        awaitLoadedLocked();
    }
    return new EditorImpl();
}    

private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) {
        throw new IllegalStateException(mThrowable);
    }
}
```

# 编辑器 Editor

* `SharedPreferences.Editor` 是线程安全的
* 编辑的过程其实是在记录编辑操作，在最终 `apply/commit` 之前都不会影响内存和磁盘里的数据

```java
public Editor putString(String key, @Nullable String value) {
    synchronized (mEditorLock) {
        mModified.put(key, value);
        return this;
    }
}

public Editor putInt(String key, int value) {
    synchronized (mEditorLock) {
        mModified.put(key, value);
        return this;
    }
}

public Editor remove(String key) {
    synchronized (mEditorLock) {
        mModified.put(key, this);
        return this;
    }
}

public Editor clear() {
    synchronized (mEditorLock) {
        mClear = true;
        return this;
    }
}
```

注意 `Editor.clear()` 操作是清空 SharedPreferences 里的数据，具体来说是清空已提交的数据，在 Editor 里尚未提交的数据并不会被清空

```java
private MemoryCommitResult commitToMemory() {
    // ...
    synchronized (mEditorLock) {
        boolean changesMade = false;
        if (mClear) {
            if (!mapToWriteToDisk.isEmpty()) {
                changesMade = true;
                mapToWriteToDisk.clear();
            }
            keysCleared = true;
            mClear = false;
        }
        
        for (Map.Entry<String, Object> e : mModified.entrySet()) {
            // ...
        }
    }
}
```
```kotlin
@Test
fun useAppContext() {
    val appContext = InstrumentationRegistry.getInstrumentation().targetContext
    val sp = appContext.getSharedPreferences("test", Context.MODE_PRIVATE)
    sp.edit { clear() }

    sp.edit {
        putBoolean("b", true)
        putFloat("f", 0.3f)
        putString("s", "apple")
    }
    sp.edit {
        putInt("i", 99)
        putLong("l", 999L)
        clear()
    }
    Log.d("cyrus", sp.all.map { "${it.key} - ${it.value}" }.joinToString())
    // l - 999, i - 99
    // 虽然 clear 操作排在最后面但它并不会清空 i 和 l
}
```

# 二阶段提交

上面说过 SharedPreferences 本质上是内存中的 `Map` 对象和磁盘上的 `XML 文件`，所以提交一个修改就会产生二阶段提交：提交到内存（`commitToMemory`）和提交到磁盘（`writeToFile`）

## commitToMemory

* 提交到内存也就是把修改操作应用到 `mMap`，由 `mLock` 上锁
* 对应地内存的版本号（`mCurrentMemoryStateGeneration`）会自增
* 在提交至内存后，尚未提交到磁盘前，`mDiskWritesInFlight` 会自增，也就说 mDiskWritesInFlight > 0 表示有修改正在等待提交到磁盘
* 提交至内存的过程中是直接修改 `mMap` 的，后续提交至磁盘的过程使用了 `MemoryCommitResult`，它也是直接引用 `mMap` 的，但是提交至磁盘的操作只对 `mWritingToDiskLock` 上锁，这就造成对象的多线程读写问题：一个线程拿到锁 `mLock` 读写 `mMap` 同时另一线程拿到 `mWritingToDiskLock` 读 `mMap`，所以当 mDiskWritesInFlight > 0 时得将 `mMap` 复制一份使用，也就是写时复制的思想（`copy on write`）如 `CopyOnWriteArrayList`


```java
private MemoryCommitResult commitToMemory() {
    long memoryStateGeneration;
    List<String> keysModified = null;
    Set<OnSharedPreferenceChangeListener> listeners = null;
    Map<String, Object> mapToWriteToDisk;
    synchronized (SharedPreferencesImpl.this.mLock) {

        // We optimistically don't make a deep copy until
        // a memory commit comes in when we're already
        // writing to disk.
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            mMap = new HashMap<String, Object>(mMap);
        }
        mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;

        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            keysModified = new ArrayList<String>();
            listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }

        synchronized (mEditorLock) {
            boolean changesMade = false;
            if (mClear) {
                if (!mapToWriteToDisk.isEmpty()) {
                    changesMade = true;
                    mapToWriteToDisk.clear();
                }
                mClear = false;
            }

            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                if (v == this || v == null) {
                    if (!mapToWriteToDisk.containsKey(k)) {
                        continue;
                    }
                    mapToWriteToDisk.remove(k);
                } else {
                    if (mapToWriteToDisk.containsKey(k)) {
                        Object existingValue = mapToWriteToDisk.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mapToWriteToDisk.put(k, v);
                }
                changesMade = true;
                if (hasListeners) {
                    keysModified.add(k);
                }
            }
            mModified.clear();
            if (changesMade) {
                mCurrentMemoryStateGeneration++;
            }
            memoryStateGeneration = mCurrentMemoryStateGeneration;
        }
    }
    return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners,
            mapToWriteToDisk);
}
```

## writeToFile

* `isFromSyncCommit`，true - 来自 `commit`，false - 来自 `apply`

* 使用 backup file 策略确保文件写操作的完整性、原子性：

    * 文件写之前将 XML 文件 `mFile` 剪切到 `mBackupFile`，写完将 mBackupFile 删除

    * 文件写之前发现有 mBackupFile 说明上一次的写操作没有正确结束，mFile 是不正确的，删掉 mFile 即可（当前提交至磁盘的内容是最新的，不用管之前的版本）

    * 第一次打开文件 `loadFromDisk` 时发现 mBackupFile 存在，说明上一次的写操作是不正确的（比如在写的过程中 killed），那么 mFile 即是不正确的（删掉），要以 mBackupFile 为准（将其 rename to mFile）

* 使用 `mDiskStateGeneration` 标识 XML 文件的版本，文件写操作完成后会将其置为 `MemoryCommitResult.memoryStateGeneration`，也即内存版本号和磁盘版本号最终会保持一致，那么：

    * 因为是先提交至内存再提交至磁盘，磁盘版本号 mDiskStateGeneration 肯定是小于等于内存版本号的 mcr.memoryStateGeneration，否则就是异常状态了

    * 如果是 apply，两个阶段的提交逻辑上是异步的，只有当即将提交到磁盘的 snapshot 的版本是当前内存的最新的版本时（`mCurrentMemoryStateGeneration == mcr.memoryStateGeneration`）才写入到文件，否则等待后续的文件写任务将最新内存版本写入文件即可

    * 如果是 commit，两个阶段的提交逻辑上是同步的，每个提交到内存的版本的 snapshot 都需要同步到磁盘

```java
final Runnable writeToDiskRunnable = new Runnable() {
        @Override
        public void run() {
            synchronized (mWritingToDiskLock) {
                writeToFile(mcr, isFromSyncCommit);
            }
            synchronized (mLock) {
                mDiskWritesInFlight--;
            }
            if (postWriteRunnable != null) {
                postWriteRunnable.run();
            }
        }
    };

private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
    long startTime = 0;
    long existsTime = 0;
    long backupExistsTime = 0;
    long outputStreamCreateTime = 0;
    long writeTime = 0;
    long fsyncTime = 0;
    long setPermTime = 0;
    long fstatTime = 0;
    long deleteTime = 0;
    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }
    boolean fileExists = mFile.exists();
    if (DEBUG) {
        existsTime = System.currentTimeMillis();
        // Might not be set, hence init them to a default value
        backupExistsTime = existsTime;
    }
    // Rename the current file so it may be used as a backup during the next read
    if (fileExists) {
        boolean needsWrite = false;
        // Only need to write if the disk state is older than this commit
        if (mDiskStateGeneration < mcr.memoryStateGeneration) {
            if (isFromSyncCommit) {
                needsWrite = true;
            } else {
                synchronized (mLock) {
                    // No need to persist intermediate states. Just wait for the latest state to
                    // be persisted.
                    if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
                        needsWrite = true;
                    }
                }
            }
        }
        if (!needsWrite) {
            mcr.setDiskWriteResult(false, true);
            return;
        }
        boolean backupFileExists = mBackupFile.exists();
        if (DEBUG) {
            backupExistsTime = System.currentTimeMillis();
        }
        if (!backupFileExists) {
            if (!mFile.renameTo(mBackupFile)) {
                Log.e(TAG, "Couldn't rename file " + mFile
                      + " to backup file " + mBackupFile);
                mcr.setDiskWriteResult(false, false);
                return;
            }
        } else {
            mFile.delete();
        }
    }
    // Attempt to write the file, delete the backup and return true as atomically as
    // possible.  If any exception occurs, delete the new file; next time we will restore
    // from the backup.
    try {
        FileOutputStream str = createFileOutputStream(mFile);
        if (DEBUG) {
            outputStreamCreateTime = System.currentTimeMillis();
        }
        if (str == null) {
            mcr.setDiskWriteResult(false, false);
            return;
        }
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);
        writeTime = System.currentTimeMillis();
        FileUtils.sync(str);
        fsyncTime = System.currentTimeMillis();
        str.close();
        ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);
        if (DEBUG) {
            setPermTime = System.currentTimeMillis();
        }
        try {
            final StructStat stat = Os.stat(mFile.getPath());
            synchronized (mLock) {
                mStatTimestamp = stat.st_mtim;
                mStatSize = stat.st_size;
            }
        } catch (ErrnoException e) {
            // Do nothing
        }
        if (DEBUG) {
            fstatTime = System.currentTimeMillis();
        }
        // Writing was successful, delete the backup file if there is one.
        mBackupFile.delete();
        if (DEBUG) {
            deleteTime = System.currentTimeMillis();
        }
        mDiskStateGeneration = mcr.memoryStateGeneration;
        mcr.setDiskWriteResult(true, true);
        if (DEBUG) {
            Log.d(TAG, "write: " + (existsTime - startTime) + "/"
                    + (backupExistsTime - startTime) + "/"
                    + (outputStreamCreateTime - startTime) + "/"
                    + (writeTime - startTime) + "/"
                    + (fsyncTime - startTime) + "/"
                    + (setPermTime - startTime) + "/"
                    + (fstatTime - startTime) + "/"
                    + (deleteTime - startTime));
        }
        long fsyncDuration = fsyncTime - writeTime;
        mSyncTimes.add((int) fsyncDuration);
        mNumSync++;
        if (DEBUG || mNumSync % 1024 == 0 || fsyncDuration > MAX_FSYNC_DURATION_MILLIS) {
            mSyncTimes.log(TAG, "Time required to fsync " + mFile + ": ");
        }
        return;
    } catch (XmlPullParserException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    } catch (IOException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    }
    // Clean up an unsuccessfully written file
    if (mFile.exists()) {
        if (!mFile.delete()) {
            Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
        }
    }
    mcr.setDiskWriteResult(false, false);
}
```

# apply & commit

了解了二阶段提交后，`apply` 和 `commit` 的逻辑与区别也清晰起来

* apply 立刻将修改提交至内存，然后将提交至磁盘的任务延迟个 100ms 交由工作队列 `QueuedWork` 执行，不阻塞当前线程；而且在工作队列里，多个 apply 产生的提交会被优化为只有最后那个（最新的修改，与内存版本一致的修改）才会真正执行文件写操作

* commit 也是立刻将修改提交至内存，然后提交至磁盘时分两种情况：
    * 如果此时只有一个提交磁盘的任务（`mDiskWritesInFlight == 1`），那么由当前线程来执行文件写操作
    * 否则将提交至磁盘的任务交由工作队列 QueuedWork 执行（无延迟），但当前线程会阻塞直到文件写操作完成

* 但无论哪种情况，commit 都会导致当前线程直到提交至磁盘后才返回，也即 commit 会确保当前修改写入至文件

```java
public void apply() {
    final long startTime = System.currentTimeMillis();
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {}
                if (DEBUG && mcr.wasWritten) {
                    Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                            + " applied after " + (System.currentTimeMillis() - startTime)
                            + " ms");
                }
            }
        };
    QueuedWork.addFinisher(awaitCommit);
    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}

public boolean commit() {
    long startTime = 0;
    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }
    MemoryCommitResult mcr = commitToMemory();
    SharedPreferencesImpl.this.enqueueDiskWrite(
        mcr, null /* sync write on this thread okay */);
    try {
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    } finally {
        if (DEBUG) {
            Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                    + " committed after " + (System.currentTimeMillis() - startTime)
                    + " ms");
        }
    }
    notifyListeners(mcr);
    return mcr.writeToDiskResult;
}
```

# 对于上面提问的回答

1. `apply` 和 `commit` 有何区别？

    * commit 会被阻塞 or 执行文件写操作直到修改提交至磁盘（确保完成二阶段提交），不适合在主线程执行
    * apply 只提交至内存，将提交至磁盘的任务交由其他线程执行，适合在主线程执行
    * apply 在频繁修改的情况下性能要比 commit 好，因为只有最新的 snapshot（具有最新的版本号）才会真正写入至文件，旧的 snapshot 会被忽略（即使写入文件后续还是会被更新的 snapshot 覆盖，不如省了一次文件写）
    * commit 确保修改能同时被提交至内存和磁盘，频繁的修改就会产生频繁的文件 IO
    * 由于 apply 只写入最新的 snapshot 至文件，极端情况下可能因为没有及时提交磁盘被 killed 而导致修改丢失

2. SharedPreferences 是线程安全的吗？它是如何保证线程安全的？

    是线程安全的，Editor 使用 `mLock` 确保内存层面的线程安全，使用 `mWritingToDiskLock` 确保磁盘文件的线程安全

3. SharedPreferences 是进程安全的吗？

    不是：
    1. 它没有使用文件锁等进程间同步工具来确保多进程对 XML 文件的安全读写
    2. 它只在构造时从文件读取内容，后续都是写操作，其他进程对 XML 文件的修改当前进程并不知道

4. 数据丢失时，其最终的屏障 —— 文件备份机制是如何实现的？

    见上面对 `mFile` 和 `mBackupFile` 的说明

5. 如何实现进程安全的 SharedPreferences

    1. 对于 XML 文件的读写操作需要加上文件锁
    2. 用 `epoll` 监控 XML 文件内容的改变 

6. SharedPreferences 导致 ANR 的原因

    1. 构造 SharedPreferences 实例时会解析 XML 文件，在解析操作完成前主线程调用 get 方法会阻塞，从而阻塞主线程
    2. 主线程执行 commit 发生文件 IO or 阻塞，从而导致主线程阻塞