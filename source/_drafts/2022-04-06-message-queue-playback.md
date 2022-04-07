---
title: 找到耗时任务：MessageQueue 的录制与回放
date: 2022-04-06 12:00:00 +0800
tags: [MessageQueue, Looper, Handler, 耗时任务, evil method]
---

# 背景

在 [今日头条 ANR 优化实践系列（3）- 实例剖析集锦](../../../..//2022/02/16/toutiao-anr-samples/) 里分析了很多 ANR 案例，很多情况下 ANR 日志并不能直观地反映出导致 ANR 的根本原因和罪魁祸首，如果没有合适的工具帮助我们录制和回放主线程消息队列 `MessageQueue`，我们根本没有线索去解决 ANR 问题

上面系列文章里字节/今日头条并没有开源相关工具，但 Tencent 开源的 APM 工具 [Matrix](https://github.com/Tencent/matrix/) 里有个模块 [TraceCanary](https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary)，它提供了一个思路

# 监控 Message 的开始和结束

## 原理

从下面 `Looper` 的逻辑可以看到，在处理 `MessageQueue` 里的每个 `Message` 之前和之后，都会通过一个 `Printer` 打印日志 `>>>>> Dispatching to` 和 `<<<<< Finished to`，那么我们可以通过 `setMessageLogging` 拦截这两条特殊的日志，从而知晓每个 `Message` 的执行开始和结束时间戳

```java
public finial class Looper {

    private Printer mLogging;

    public static void loop() {
        // ...
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
            
            // ...
            msg.target.dispatchMessage(msg);
            // ...

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
            // ...
        }
    }

    /**
     * Control logging of messages as they are processed by this Looper.  If
     * enabled, a log message will be written to <var>printer</var>
     * at the beginning and ending of each message dispatch, identifying the
     * target Handler and message contents.
     *
     * @param printer A Printer object that will receive log messages, or
     * null to disable message logging.
     */
    public void setMessageLogging(@Nullable Printer printer) {
        mLogging = printer;
    }    
}
```

## Matrix 的实现

给 `Looper` 设置自定义 `MessageLogging` 以实现拦截特殊日志的目的，并通过 `IdleHandler` 在空闲时刻检查自定义 MessageLogging 是否被替换，保障功能的完整性

```java
public class LooperMonitor {

    private static final long CHECK_TIME = 60 * 1000L;

    private LooperMonitor(Looper looper) {
        Objects.requireNonNull(looper);
        this.looper = looper;
        resetPrinter();
        addIdleHandler(looper);
    }    

    private synchronized void resetPrinter() {
        Printer originPrinter = null;
        try {
            if (!isReflectLoggingError) {
                originPrinter = ReflectUtils.get(looper.getClass(), "mLogging", looper);
                if (originPrinter == printer && null != printer) {
                    return;
                }
                // Fix issues that printer loaded by different classloader
                if (originPrinter != null && printer != null) {
                    if (originPrinter.getClass().getName().equals(printer.getClass().getName())) {
                        MatrixLog.w(TAG, "LooperPrinter might be loaded by different classloader"
                                + ", my = " + printer.getClass().getClassLoader()
                                + ", other = " + originPrinter.getClass().getClassLoader());
                        return;
                    }
                }
            }
        } catch (Exception e) {
            isReflectLoggingError = true;
            Log.e(TAG, "[resetPrinter] %s", e);
        }

        if (null != printer) {
            MatrixLog.w(TAG, "maybe thread:%s printer[%s] was replace other[%s]!",
                    looper.getThread().getName(), printer, originPrinter);
        }
        looper.setMessageLogging(printer = new LooperPrinter(originPrinter));
        if (null != originPrinter) {
            MatrixLog.i(TAG, "reset printer, originPrinter[%s] in %s", originPrinter, looper.getThread().getName());
        }
    }    

    private synchronized void addIdleHandler(Looper looper) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            looper.getQueue().addIdleHandler(this);
        } else {
            try {
                MessageQueue queue = ReflectUtils.get(looper.getClass(), "mQueue", looper);
                queue.addIdleHandler(this);
            } catch (Exception e) {
                Log.e(TAG, "[removeIdleHandler] %s", e);
            }
        }
    }

    @Override
    public boolean queueIdle() {
        if (SystemClock.uptimeMillis() - lastCheckPrinterTime >= CHECK_TIME) {
            resetPrinter();
            lastCheckPrinterTime = SystemClock.uptimeMillis();
        }
        return true;
    }        
}
```

自定义 MessageLogging 的逻辑也很简单，如果日志以 `>` 开头则是开始处理 Message，以 `<` 开头则是 Message 处理完毕

```java
class LooperPrinter implements Printer {

    public Printer origin;
    boolean isHasChecked = false;
    boolean isValid = false;

    LooperPrinter(Printer printer) {
        this.origin = printer;
    }

    @Override
    public void println(String x) {
        if (null != origin) {
            origin.println(x);
            if (origin == this) {
                throw new RuntimeException(TAG + " origin == this");
            }
        }

        if (!isHasChecked) {
            isValid = x.charAt(0) == '>' || x.charAt(0) == '<';
            isHasChecked = true;
            if (!isValid) {
                MatrixLog.e(TAG, "[println] Printer is inValid! x:%s", x);
            }
        }

        if (isValid) {
            dispatch(x.charAt(0) == '>', x);
        }
    }
}
```

整个 `LooperMonitor` 的 API 概览，通过继承 `LooperDispatchListener` 可以在 `dispatchStart` 得到消息处理开始的回调，在 `dispatchEnd` 得到消息处理结束的回调



```java
public class LooperMonitor {

    private final HashSet<LooperDispatchListener> listeners = new HashSet<>();

    public void addListener(LooperDispatchListener listener) {
        synchronized (listeners) {
            listeners.add(listener);
        }
    }

    public void removeListener(LooperDispatchListener listener) {
        synchronized (listeners) {
            listeners.remove(listener);
        }
    }    

    public abstract static class LooperDispatchListener {

        boolean isHasDispatchStart = false;
        boolean historyMsgRecorder = false;
        boolean denseMsgTracer = false;

        public LooperDispatchListener(boolean historyMsgRecorder, boolean denseMsgTracer) {
            this.historyMsgRecorder = historyMsgRecorder;
            this.denseMsgTracer = denseMsgTracer;
        }

        public LooperDispatchListener() {}

        public boolean isValid() {
            return false;
        }

        public void dispatchStart() {}

        @CallSuper
        public void onDispatchStart(String x) {
            this.isHasDispatchStart = true;
            dispatchStart();
        }

        @CallSuper
        public void onDispatchEnd(String x) {
            this.isHasDispatchStart = false;
            dispatchEnd();
        }

        public void dispatchEnd() {}
    }    

    private void dispatch(boolean isBegin, String log) {
        synchronized (listeners) {
            for (LooperDispatchListener listener : listeners) {
                if (listener.isValid()) {
                    if (isBegin) {
                        if (!listener.isHasDispatchStart) {
                            if (listener.historyMsgRecorder) {
                                messageStartTime = System.currentTimeMillis();
                                latestMsgLog = log;
                                recentMCount++;
                            }
                            listener.onDispatchStart(log);
                        }
                    } else {
                        if (listener.isHasDispatchStart) {
                            if (listener.historyMsgRecorder) {
                                recordMsg(log, System.currentTimeMillis() - messageStartTime, listener.denseMsgTracer);
                            }
                            listener.onDispatchEnd(log);
                        }
                    }
                } else if (!isBegin && listener.isHasDispatchStart) {
                    listener.dispatchEnd();
                }
            }
        }
    }    
}
```

# AppMethodBeat.i/o 插桩记录函数执行耗时

## 原理

为了在事后回放主线程 `Looper` 的历史消息的执行耗时，需要找到一个方法来记录所有函数的执行次序及其执行耗时，如下图所示：

* 用一个 long 类型（64 bits）记录一个函数的开始（`AppMethodBeat.i`）或者结束（`AppMethodBeat.o`）
* 第 63 位表示函数的开始（1）或者结束（0）
* \[43:62\] 共 20 bits 表示函数 ID（methodId），可以分配给 2^20 = 1,048,576 个函数
* \[0:42\] 共 43 bits 表示相对时间戳，单位毫秒，2^43 可以记录近 120W 小时的时间范围
* 为啥是相对时间戳？因为 43 bits 不足以保存 `SystemClock.uptimeMillis()`，且函数执行耗时 = 结束时间戳 - 开始时间戳，要的就是个差值，43 bits 足以保存这个差值了

![id.jpg](../../../../image/2022-04-06-message-queue-playback/id.jpg)

运行时用一个预先分配内存的长度为 100W 的 long 型数组记录函数调用的开始和结束（内存占用约 7.6M），如下图所示：

* 所有的函数调用呈现为线性的、一维的 long 型数组
* 一个函数调用的开始后续必定能找到对应的结束记录，函数执行耗时 = 开始时间戳 - 结束时间戳，类似于计算器中小括号的规则
* 用一个特殊的 method id 表示 Looper 从 MessageQueue 获得一个 Message 并开始处理（METHOD_ID_DISPATCH）
* 用 METHOD_ID_DISPATCH 将整个数组切分为多个小数组，每个小数组就是一个 Message 的调用栈
* 将每个 Message 调用栈数组展开为树，就可以很直观地剖析每个 Message 内的耗时函数调用

![method_tree.jpg](../../../../image/2022-04-06-message-queue-playback/method_tree.jpg)

## 实现 AppMethodBeat.i/o

* 数组 `sBuffer` 是个环形数组 ring array，超过容量（100W）后新加进来的数据会覆盖最旧的数据
* `SystemClock.uptimeMillis()` 只取低 43 位，是个不完整的时间戳，只能作为相对时间戳用来计算时间段
* 加载 Matrix 时将 `sDiffTime` 和 `sCurrentDiffTime` 初始化为当前时间戳，然后每次 Message 执行前将相对时间戳记录到 `sCurrentDiffTime = SystemClock.uptimeMillis() - sDiffTime`，其他的函数调用并不会更新 `sCurrentDiffTime`，这样可以降低频繁获取机器时间带来的额外时间损耗
* 新开一个线程 `sUpdateDiffTimeRunnable` 每隔 5ms 更新下 `sCurrentDiffTime`，执行耗时在 5ms 内的函数基本不会对 ANR 或者卡顿造成影响，可以忽略不计

```java
public class AppMethodBeat {

    private static long[] sBuffer = new long[Constants.BUFFER_SIZE];
    private volatile static long sCurrentDiffTime = SystemClock.uptimeMillis();
    private volatile static long sDiffTime = sCurrentDiffTime;

    /**
     * hook method when it's called in.
     *
     * @param methodId
     */
    public static void i(int methodId) {

        if (status <= STATUS_STOPPED) {
            return;
        }
        if (methodId >= METHOD_ID_MAX) {
            return;
        }

        if (status == STATUS_DEFAULT) {
            synchronized (statusLock) {
                if (status == STATUS_DEFAULT) {
                    realExecute();
                    status = STATUS_READY;
                }
            }
        }

        long threadId = Thread.currentThread().getId();
        if (sMethodEnterListener != null) {
            sMethodEnterListener.enter(methodId, threadId);
        }

        if (threadId == sMainThreadId) {
            if (assertIn) {
                android.util.Log.e(TAG, "ERROR!!! AppMethodBeat.i Recursive calls!!!");
                return;
            }
            assertIn = true;
            if (sIndex < Constants.BUFFER_SIZE) {
                mergeData(methodId, sIndex, true);
            } else {
                sIndex = 0;
                 mergeData(methodId, sIndex, true);
            }
            ++sIndex;
            assertIn = false;
        }
    }

    /**
     * hook method when it's called out.
     *
     * @param methodId
     */
    public static void o(int methodId) {
        if (status <= STATUS_STOPPED) {
            return;
        }
        if (methodId >= METHOD_ID_MAX) {
            return;
        }
        if (Thread.currentThread().getId() == sMainThreadId) {
            if (sIndex < Constants.BUFFER_SIZE) {
                mergeData(methodId, sIndex, false);
            } else {
                sIndex = 0;
                mergeData(methodId, sIndex, false);
            }
            ++sIndex;
        }
    }        

    /**
     * merge trace info as a long data
     *
     * @param methodId
     * @param index
     * @param isIn
     */
    private static void mergeData(int methodId, int index, boolean isIn) {
        if (methodId == AppMethodBeat.METHOD_ID_DISPATCH) {
            sCurrentDiffTime = SystemClock.uptimeMillis() - sDiffTime;
        }

        try {
            long trueId = 0L;
            if (isIn) {
                trueId |= 1L << 63;
            }
            trueId |= (long) methodId << 43;
            trueId |= sCurrentDiffTime & 0x7FFFFFFFFFFL;
            sBuffer[index] = trueId;
            checkPileup(index);
            sLastIndex = index;
        } catch (Throwable t) {
            MatrixLog.e(TAG, t.getMessage());
        }
    }    

    /**
     * update time runnable
     */
    private static Runnable sUpdateDiffTimeRunnable = new Runnable() {
        @Override
        public void run() {
            try {
                while (true) {
                    while (!isPauseUpdateTime && status > STATUS_STOPPED) {
                        sCurrentDiffTime = SystemClock.uptimeMillis() - sDiffTime;
                        SystemClock.sleep(Constants.TIME_UPDATE_CYCLE_MS);
                    }
                    synchronized (updateTimeLock) {
                        updateTimeLock.wait();
                    }
                }
            } catch (Exception e) {
                MatrixLog.e(TAG, "" + e.toString());
            }
        }
    };    
}

class Constants {
    public static final int BUFFER_SIZE = 100 * 10000; // 7.6M
    public static final int TIME_UPDATE_CYCLE_MS = 5;
}
```

## 字节码插桩

利用 [ASM](../../../../2022/04/07/asm/) 的 `AdviceAdapter` 在函数调用前后织入 `AppMethodBeat.i` 和 `AppMethodBeat.o`

```java
private class TraceMethodAdapter extends AdviceAdapter {
    
    protected void onMethodEnter() {
        TraceMethod traceMethod = collectedMethodMap.get(methodName);
        if (traceMethod != null) {
            traceMethodCount.incrementAndGet();
            mv.visitLdcInsn(traceMethod.id);
            mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "i", "(I)V", false);
            if (checkNeedTraceWindowFocusChangeMethod(traceMethod)) {
                traceWindowFocusChangeMethod(mv, className);
            }
        }
    }

    protected void onMethodExit(int opcode) {
        TraceMethod traceMethod = collectedMethodMap.get(methodName);
        if (traceMethod != null) {
            traceMethodCount.incrementAndGet();
            mv.visitLdcInsn(traceMethod.id);
            mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "o", "(I)V", false);
        }
    }
}
```

注册 Gradle Transform

```kotlin
// 注册 Gradle Plugin
// /META-INF/gradle-plugins/com.tencent.matrix-plugin.properties
// implementation-class=com.tencent.matrix.plugin.MatrixPlugin

class MatrixPlugin : Plugin<Project> {
    override fun apply(project: Project) {

        // 创建配置块
        val matrix = project.extensions.create("matrix", MatrixExtension::class.java)
        val traceExtension = (matrix as ExtensionAware).extensions.create("trace", MatrixTraceExtension::class.java)
        val removeUnusedResourcesExtension = matrix.extensions.create("removeUnusedResources", MatrixRemoveUnusedResExtension::class.java)

        if (!project.plugins.hasPlugin("com.android.application")) {
            throw GradleException("Matrix Plugin, Android Application plugin required.")
        }

        project.afterEvaluate {
            Log.setLogLevel(matrix.logLevel)
        }

        MatrixTasksManager().createMatrixTasks(
                project.extensions.getByName("android") as AppExtension,
                project,
                traceExtension,
                removeUnusedResourcesExtension
        )
    }
}

// 注册 Gradle Transform
class MatrixTraceInjection : ITraceSwitchListener {
    private fun injectTransparentTransform(appExtension: AppExtension,
                                           project: Project,
                                           extension: MatrixTraceExtension) {
        transparentTransform = MatrixTraceTransform(project, extension)
        appExtension.registerTransform(transparentTransform!!)
    }
}

class MatrixTraceTransform: Transform() {

    override fun getName(): String {
        return "MatrixTraceTransform"
    }

    override fun getInputTypes(): Set<QualifiedContent.ContentType> {
        return TransformManager.CONTENT_CLASS
    }

    override fun getScopes(): MutableSet<in QualifiedContent.Scope>? {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    override fun isIncremental(): Boolean {
        return true
    }

    override fun transform(transformInvocation: TransformInvocation) {
        super.transform(transformInvocation)

        if (transparent) {
            transparent(transformInvocation)
        } else {
            transforming(transformInvocation)
        }
    }

    private fun transforming(invocation: TransformInvocation) {

        val start = System.currentTimeMillis()

        val outputProvider = invocation.outputProvider!!
        val isIncremental = invocation.isIncremental && this.isIncremental

        if (!isIncremental) {
            outputProvider.deleteAll()
        }

        val config = configure(invocation)

        val changedFiles = ConcurrentHashMap<File, Status>()
        val inputToOutput = ConcurrentHashMap<File, File>()
        val inputFiles = ArrayList<File>()

        var transformDirectory: File? = null

        for (input in invocation.inputs) {
            for (directoryInput in input.directoryInputs) {
                changedFiles.putAll(directoryInput.changedFiles)
                val inputDir = directoryInput.file
                inputFiles.add(inputDir)
                val outputDirectory = outputProvider.getContentLocation(
                        directoryInput.name,
                        directoryInput.contentTypes,
                        directoryInput.scopes,
                        Format.DIRECTORY)

                inputToOutput[inputDir] = outputDirectory
                if (transformDirectory == null) transformDirectory = outputDirectory.parentFile
            }

            for (jarInput in input.jarInputs) {
                val inputFile = jarInput.file
                changedFiles[inputFile] = jarInput.status
                inputFiles.add(inputFile)
                val outputJar = outputProvider.getContentLocation(
                        jarInput.name,
                        jarInput.contentTypes,
                        jarInput.scopes,
                        Format.JAR)

                inputToOutput[inputFile] = outputJar
                if (transformDirectory == null) transformDirectory = outputJar.parentFile
            }
        }

        if (inputFiles.size == 0 || transformDirectory == null) {
            Log.i(TAG, "Matrix trace do not find any input files")
            return
        }

        // Get transform root dir.
        val outputDirectory = transformDirectory

        MatrixTrace(
                ignoreMethodMapFilePath = config.ignoreMethodMapFilePath,
                methodMapFilePath = config.methodMapFilePath,
                baseMethodMapPath = config.baseMethodMapPath,
                blockListFilePath = config.blockListFilePath,
                mappingDir = config.mappingDir,
                project = project
        ).doTransform(
                classInputs = inputFiles,
                changedFiles = changedFiles,
                isIncremental = isIncremental,
                skipCheckClass = config.skipCheckClass,
                traceClassDirectoryOutput = outputDirectory,
                inputToOutput = inputToOutput,
                legacyReplaceChangedFile = null,
                legacyReplaceFile = null,
                uniqueOutputName = true
        )

        val cost = System.currentTimeMillis() - start
        Log.i(TAG, " Insert matrix trace instrumentations cost time: %sms.", cost)
    }
}

// MatrixTrace.doTransform
// MethodTracer.trace
// MethodTracer.traceMethodFromSrc

public class MethodTracer {
    private void innerTraceMethodFromSrc(File input, File output, ClassLoader classLoader, boolean ignoreCheckClass) {

        ArrayList<File> classFileList = new ArrayList<>();
        if (input.isDirectory()) {
            listClassFiles(classFileList, input);
        } else {
            classFileList.add(input);
        }

        for (File classFile : classFileList) {
            InputStream is = null;
            FileOutputStream os = null;
            try {
                final String changedFileInputFullPath = classFile.getAbsolutePath();
                final File changedFileOutput = new File(changedFileInputFullPath.replace(input.getAbsolutePath(), output.getAbsolutePath()));

                if (changedFileOutput.getCanonicalPath().equals(classFile.getCanonicalPath())) {
                    throw new RuntimeException("Input file(" + classFile.getCanonicalPath() + ") should not be same with output!");
                }

                if (!changedFileOutput.exists()) {
                    changedFileOutput.getParentFile().mkdirs();
                }
                changedFileOutput.createNewFile();

                if (MethodCollector.isNeedTraceFile(classFile.getName())) {

                    // Reader - Adapter - Write 修改源字节码文件
                    is = new FileInputStream(classFile);
                    ClassReader classReader = new ClassReader(is);
                    ClassWriter classWriter = new TraceClassWriter(ClassWriter.COMPUTE_FRAMES, classLoader);
                    ClassVisitor classVisitor = new TraceClassAdapter(AgpCompat.getAsmApi(), classWriter);
                    classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);
                    is.close();

                    byte[] data = classWriter.toByteArray();

                    // 用 CheckClassAdapter 检查结果的正确性
                    if (!ignoreCheckClass) {
                        try {
                            ClassReader cr = new ClassReader(data);
                            ClassWriter cw = new ClassWriter(0);
                            ClassVisitor check = new CheckClassAdapter(cw);
                            cr.accept(check, ClassReader.EXPAND_FRAMES);
                        } catch (Throwable e) {
                            System.err.println("trace output ERROR : " + e.getMessage() + ", " + classFile);
                            traceError = true;
                        }
                    }

                    if (output.isDirectory()) {
                        os = new FileOutputStream(changedFileOutput);
                    } else {
                        os = new FileOutputStream(output);
                    }
                    os.write(data);
                    os.close();
                } else {
                    FileUtil.copyFileUsingStream(classFile, changedFileOutput);
                }
            } // ...
        }
    }    
}
```

