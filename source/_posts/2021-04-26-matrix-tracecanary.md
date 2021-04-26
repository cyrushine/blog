
## 无埋点插桩

为了在捕捉到卡顿堆栈后，获取各个函数的执行耗时，需要对所有函数进行无埋点插桩，在函数执行前调用 `MethodBeat.i`，在函数执行后调用 `MethodBeat.o`

> 实现细节
> 
> 编译期：
> 
> 通过代理编译期间的任务 `transformClassesWithDexTask`，将全局 `class` 文件作为输入，利用 `ASM` 工具，高效地对所有 `class` 文件进行扫描及插桩
> 
> 插桩过程有几个关键点：
> 
> 1. 选择在该编译任务执行时插桩，是因为 `proguard` 操作是在该任务之前就完成的，意味着插桩时的 `class` 文件已经被混淆过的。而选择 `proguard` 之后去插桩，是因为如果提前插桩会造成部分方法不符合内联规则，没法在 `proguard` 时进行优化，最终导致程序方法数无法减少，从而引发方法数过大问题
> 2. 为了减少插桩量及性能损耗，通过遍历 `class` 方法指令集，判断扫描的函数是否只含有 `PUT/READ FIELD` 等简单的指令，来过滤一些默认或匿名构造函数，以及 `get/set` 等简单不耗时函数
> 3. 针对界面启动耗时，因为要统计从 `Activity.onCreate` 到 `Activity.onWindowFocusChange` 间的耗时，所以在插桩过程中需要收集应用内所有 `Activity` 的实现类，并覆盖 `onWindowFocusChange` 函数进行打点
> 4. 为了方便及高效记录函数执行过程，我们为每个插桩的函数分配一个独立 ID，在插桩过程中，记录插桩的函数签名及分配的 ID，在插桩完成后输出一份 mapping，作为数据上报后的解析支持。
> 
> 归纳起来，编译期所做的工作如下图：
> 
> ![transform class](https://github.com/Tencent/matrix/wiki/images/trace/build.png)

### Gradle Transform

`ignoreMethodMapFilePath` 上面说过为了减少插桩量及性能损耗会忽略一些函数，这些被忽略的函数记录在此文件里（默认放在 `/app/build/outputs/mapping/{var}/ignoreMethodMapping.txt`），大概长这样：

```kotlin
ignore methods:
android.arch.core.executor.ArchTaskExecutor <clinit> ()V
android.arch.core.executor.ArchTaskExecutor <init> ()V
android.arch.core.executor.ArchTaskExecutor$1 execute (Ljava.lang.Runnable;)V
android.arch.core.executor.DefaultTaskExecutor executeOnDiskIO (Ljava.lang.Runnable;)V
android.arch.core.internal.FastSafeIterableMap <init> ()V
android.arch.core.internal.FastSafeIterableMap ceil (Ljava.lang.Object;)Ljava.util.Map$Entry;
android.arch.core.internal.SafeIterableMap size ()I
android.arch.core.internal.SafeIterableMap equals (Ljava.lang.Object;)Z
android.arch.core.internal.SafeIterableMap$AscendingIterator backward (Landroid.arch.core.internal.SafeIterableMap$Entry;)Landroid.arch.core.internal.SafeIterableMap$Entry;
android.arch.core.internal.SafeIterableMap$AscendingIterator forward (Landroid.arch.core.internal.SafeIterableMap$Entry;)Landroid.arch.core.internal.SafeIterableMap$Entry;
android.arch.core.internal.SafeIterableMap$AscendingIterator <init> (Landroid.arch.core.internal.SafeIterableMap$Entry;Landroid.arch.core.internal.SafeIterableMap$Entry;)V
```

`methodMapFilePath` 函数签名和函数 ID 的映射（默认放在 `/app/build/outputs/mapping/{var}/methodMapping.txt`），大概长这样：

```kotlin
1,1,sample.tencent.matrix.MainActivity$6 run ()V
2,10,sample.tencent.matrix.MatrixApplication initSQLiteLintConfig ()Lcom.tencent.sqlitelint.config.SQLiteLintConfig;
3,1,sample.tencent.matrix.MainActivity$5 onClick (Landroid.view.View;)V
4,1,sample.tencent.matrix.MainActivity$4 onClick (Landroid.view.View;)V
5,4,sample.tencent.matrix.MainActivity onResume ()V
6,1,sample.tencent.matrix.MatrixApplication onCreate ()V
7,1,sample.tencent.matrix.MainActivity$3 onClick (Landroid.view.View;)V
8,4,sample.tencent.matrix.MainActivity onCreate (Landroid.os.Bundle;)V
9,1,sample.tencent.matrix.MainActivity$2 onClick (Landroid.view.View;)V
10,1,org.apache.commons.io.comparator.DirectoryFileComparator compare (Ljava.io.File;Ljava.io.File;)I
```

`TraceCanary` 用 `Transform API` 处理并输出插桩后的 `class` 

```kotlin
class MatrixTraceTransform: Transform() {

    // Transform 的名称
    override fun getName(): String {
        return "MatrixTraceTransform"
    }

    // Transform 接收 Input 处理并输出 Output
    // 这里声明 MatrixTraceTransform 接收所有的 class，包括 class 文件和 jar 包中的 class
    override fun getInputTypes(): Set<QualifiedContent.ContentType> {
        return TransformManager.CONTENT_CLASS
    }

    // 指定 class 的范围，限定项目内的所有 class
    override fun getScopes(): MutableSet<in QualifiedContent.Scope>? {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    // 支持增量编译
    override fun isIncremental(): Boolean {
        return true
    }

    override fun transform(transformInvocation: TransformInvocation) {
        super.transform(transformInvocation)
        // ...
        transforming(transformInvocation)
    }

    // 核心逻辑
    private fun transforming(invocation: TransformInvocation) {
        val start = System.currentTimeMillis()
        val outputProvider = invocation.outputProvider!!
        val isIncremental = invocation.isIncremental && this.isIncremental
        if (!isIncremental) {
            outputProvider.deleteAll()
        }
        val config = configure(invocation)

        val changedFiles = ConcurrentHashMap<File, Status>()    // 需要进行插桩的 class/jar
        val inputToOutput = ConcurrentHashMap<File, File>()     // Input Dir -> Output Dir，Input Jar -> Output Jar
        val inputFiles = ArrayList<File>()                      // Input Dir && Input Jar
        var transformDirectory: File? = null
        for (input in invocation.inputs) {

            // 遍历并添加 class 文件
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

            // 遍历并添加 jar 包
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

        // 执行插桩
        val outputDirectory = transformDirectory
        MatrixTrace(
                ignoreMethodMapFilePath = config.ignoreMethodMapFilePath,
                methodMapFilePath = config.methodMapFilePath,
                baseMethodMapPath = config.baseMethodMapPath,   // 上一次插桩的 method mapping file，记录了已分配的 method id，插桩时要复用
                blockListFilePath = config.blockListFilePath,   // 黑名单机制，忽略匹配的函数
                mappingDir = config.mappingDir                  // Proguard mapping file
        ).doTransform(
                classInputs = inputFiles,
                changedFiles = changedFiles,
                isIncremental = isIncremental,
                traceClassDirectoryOutput = outputDirectory,
                inputToOutput = inputToOutput,
                legacyReplaceChangedFile = null,
                legacyReplaceFile = null)

        val cost = System.currentTimeMillis() - start
        Log.i(TAG, " Insert matrix trace instrumentations cost time: %sms.", cost)
    }

    private fun configure(transformInvocation: TransformInvocation): Configuration {
        val buildDir = project.buildDir.absolutePath
        val dirName = transformInvocation.context.variantName
        val mappingOut = Joiner.on(File.separatorChar).join(
                buildDir,
                FD_OUTPUTS,
                "mapping",
                dirName)

        return Configuration.Builder()
                .setBaseMethodMap(extension.baseMethodMapFile)
                .setBlockListFile(extension.blackListFile)
                .setMethodMapFilePath("$mappingOut/methodMapping.txt")
                .setIgnoreMethodMapFilePath("$mappingOut/ignoreMethodMapping.txt")
                .setMappingPath(mappingOut)
                .build()
    }
}
```

### 解析配置文件

```kotlin
fun MatrixTrace.doTransform(...) {
    val executor: ExecutorService = Executors.newFixedThreadPool(16)
    val config = Configuration.Builder()
            .setIgnoreMethodMapFilePath(ignoreMethodMapFilePath)
            .setMethodMapFilePath(methodMapFilePath)
            .setBaseMethodMap(baseMethodMapPath)
            .setBlockListFile(blockListFilePath)
            .setMappingPath(mappingDir)
            .build()

    /**
     * step 1
     */
    var start = System.currentTimeMillis()
    val futures = LinkedList<Future<*>>()
    val mappingCollector = MappingCollector()
    val methodId = AtomicInteger(0)
    val collectedMethodMap = ConcurrentHashMap<String, TraceMethod>()

    // 在线程池里解析各种 mapping file
    futures.add(executor.submit(ParseMappingTask(
            mappingCollector, collectedMethodMap, methodId, config)))

    // 在线程池里扫描 class dir 和 jar，将 Input 和 Output 映射好放在下面的两个 map 里
    val dirInputOutMap = ConcurrentHashMap<File, File>()
    val jarInputOutMap = ConcurrentHashMap<File, File>()
    for (file in classInputs) {
        if (file.isDirectory) {
            futures.add(executor.submit(CollectDirectoryInputTask(
                    directoryInput = file,
                    mapOfChangedFiles = changedFiles,
                    mapOfInputToOutput = inputToOutput,
                    isIncremental = isIncremental,
                    traceClassDirectoryOutput = traceClassDirectoryOutput,
                    legacyReplaceChangedFile = legacyReplaceChangedFile,
                    legacyReplaceFile = legacyReplaceFile,
                    resultOfDirInputToOut = dirInputOutMap
            )))
        } else {
            val status = Status.CHANGED
            futures.add(executor.submit(CollectJarInputTask(
                    inputJar = file,
                    inputJarStatus = status,
                    inputToOutput = inputToOutput,
                    isIncremental = isIncremental,
                    traceClassFileOutput = traceClassDirectoryOutput,
                    legacyReplaceFile = legacyReplaceFile,
                    resultOfDirInputToOut = dirInputOutMap,
                    resultOfJarInputToOut = jarInputOutMap
            )))
        }
    }
    for (future in futures) {
        future.get()
    }
    futures.clear()
    Log.i(TAG, "[doTransform] Step(1)[Parse]... cost:%sms", System.currentTimeMillis() - start)
    // ...
}

class ParseMappingTask(...) : Runnable {
    override fun run() {
        val start = System.currentTimeMillis()
        val mappingFile = File(config.mappingDir, "mapping.txt")    // 解析 Proguard mapping file
        if (mappingFile.isFile) {
            val mappingReader = MappingReader(mappingFile)
            mappingReader.read(mappingCollector)
        }
        val size = config.parseBlockFile(mappingCollector)          // 解析黑名单
        val baseMethodMapFile = File(config.baseMethodMapPath)      // 加载已分配 method id 的函数列表
        getMethodFromBaseMethod(baseMethodMapFile, collectedMethodMap)
        retraceMethodMap(mappingCollector, collectedMethodMap)
        Log.i(TAG, "[ParseMappingTask#run] cost:%sms, black size:%s, collect %s method from %s",
                System.currentTimeMillis() - start, size, collectedMethodMap.size, config.baseMethodMapPath)
    }
}
```

### 收集匹配的函数

```kotlin
fun MatrixTrace.doTransform(...) {
    // ... step 2 在线程池里用 ASM 解析 class 并收集匹配的函数
    start = System.currentTimeMillis()
    val methodCollector = MethodCollector(executor, mappingCollector, methodId, config, collectedMethodMap)
    methodCollector.collect(dirInputOutMap.keys, jarInputOutMap.keys)
    Log.i(TAG, "[doTransform] Step(2)[Collection]... cost:%sms", System.currentTimeMillis() - start)
    // ...
}
```

```java
// 搜集匹配的函数
public void MethodCollector.collect(Set<File> srcFolderList, Set<File> dependencyJarList) throws ExecutionException, InterruptedException {
    List<Future> futures = new LinkedList<>();
    for (File srcFile : srcFolderList) {                    // 在 class 里收集匹配的函数
        ArrayList<File> classFileList = new ArrayList<>();
        if (srcFile.isDirectory()) {
            listClassFiles(classFileList, srcFile);
        } else {
            classFileList.add(srcFile);
        }
        for (File classFile : classFileList) {
            futures.add(executor.submit(new CollectSrcTask(classFile)));
        }
    }
    for (File jarFile : dependencyJarList) {                // 在 jar 包里收集匹配的函数
        futures.add(executor.submit(new CollectJarTask(jarFile)));
    }
    for (Future future : futures) {
        future.get();
    }
    futures.clear();
    futures.add(executor.submit(new Runnable() {
        @Override
        public void run() {
            saveIgnoreCollectedMethod(mappingCollector);    // 写入 ignored methods 到文件里
        }
    }));
    futures.add(executor.submit(new Runnable() {
        @Override
        public void run() {
            saveCollectedMethod(mappingCollector);          // 将 method id -> method 映射写入文件里
        }
    }));
    for (Future future : futures) {
        future.get();
    }
    futures.clear();
}

// 用 ASM Core API 解析并找到匹配的函数
public void CollectMethodNode.visitEnd() {
    super.visitEnd();
    TraceMethod traceMethod = TraceMethod.create(0, access, className, name, desc);
    if ("<init>".equals(name)) {
        isConstructor = true;
    }

    // 过滤掉空函数、getter/setter 等，加入 ignored method file 里
    boolean isNeedTrace = isNeedTrace(configuration, traceMethod.className, mappingCollector);
    if ((isEmptyMethod() || isGetSetMethod() || isSingleMethod())
            && isNeedTrace) {
        ignoreCount.incrementAndGet();
        collectedIgnoreMethodMap.put(traceMethod.getMethodName(), traceMethod);
        return;
    }

    // 需要插桩的函数，如果没有 method id 则分配一个自增的 method id，加入 collectedMethodMap
    if (isNeedTrace && !collectedMethodMap.containsKey(traceMethod.getMethodName())) {
        traceMethod.id = methodId.incrementAndGet();
        collectedMethodMap.put(traceMethod.getMethodName(), traceMethod);
        incrementCount.incrementAndGet();
    } else if (!isNeedTrace && !collectedIgnoreMethodMap.containsKey(traceMethod.className)) {
        ignoreCount.incrementAndGet();
        collectedIgnoreMethodMap.put(traceMethod.getMethodName(), traceMethod);
    }
}

// Jar 包则用 ZipFile API 遍历，依然用 ASM 解析
class CollectJarTask implements Runnable {
    public void run() {
        ZipFile zipFile = null;
        try {
            zipFile = new ZipFile(fromJar);
            Enumeration<? extends ZipEntry> enumeration = zipFile.entries();
            while (enumeration.hasMoreElements()) {
                ZipEntry zipEntry = enumeration.nextElement();
                String zipEntryName = zipEntry.getName();
                if (isNeedTraceFile(zipEntryName)) {
                    InputStream inputStream = zipFile.getInputStream(zipEntry);
                    ClassReader classReader = new ClassReader(inputStream);
                    ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
                    ClassVisitor visitor = new TraceClassAdapter(Opcodes.ASM5, classWriter);
                    classReader.accept(visitor, 0);
                }
            }
        }
        // ...
    }
}
```

### 执行插桩操作

```kotlin
fun MatrixTrace.doTransform(...) {
    // ... step 3 执行插桩操作
    start = System.currentTimeMillis()
    val methodTracer = MethodTracer(executor, mappingCollector, config, methodCollector.collectedMethodMap, methodCollector.collectedClassExtendMap)
    methodTracer.trace(dirInputOutMap, jarInputOutMap)
    Log.i(TAG, "[doTransform] Step(3)[Trace]... cost:%sms", System.currentTimeMillis() - start)
}
```

```java
public void MethodTracer.trace(Map<File, File> srcFolderList, Map<File, File> dependencyJarList) throws ExecutionException, InterruptedException {
    List<Future> futures = new LinkedList<>();
    traceMethodFromSrc(srcFolderList, futures);     // 处理 class
    traceMethodFromJar(dependencyJarList, futures); // 处理 jar
    for (Future future : futures) {
        future.get();
    }
    futures.clear();
}

// 一个线程处理一个 class/dir 
private void traceMethodFromSrc(Map<File, File> srcMap, List<Future> futures) {
    if (null != srcMap) {
        for (Map.Entry<File, File> entry : srcMap.entrySet()) {
            futures.add(executor.submit(new Runnable() {
                public void run() { innerTraceMethodFromSrc(entry.getKey(), entry.getValue()); }
            }));
        }
    }
}

private void innerTraceMethodFromSrc(File input, File output) {
    // ... 依然是 ASM 里 classReader，classWriter 和 classVisitor 的经典用法
    is = new FileInputStream(classFile);
    ClassReader classReader = new ClassReader(is);
    ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
    ClassVisitor classVisitor = new TraceClassAdapter(Opcodes.ASM5, classWriter);   // 重要的类
    classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);
    // ...
}

public MethodVisitor TraceClassAdapter.visitMethod(int access, String name, String desc,
                                 String signature, String[] exceptions) {
    if (isABSClass) {
        return super.visitMethod(access, name, desc, signature, exceptions);
    } else {
        if (!hasWindowFocusMethod) {    // 匹配 Activity.onWindowFocusChanged，做页面打开速度统计
            hasWindowFocusMethod = MethodCollector.isWindowFocusChangeMethod(name, desc);
        }
        MethodVisitor methodVisitor = cv.visitMethod(access, name, desc, signature, exceptions);
        return new TraceMethodAdapter(api, methodVisitor, access, name, desc, this.className,
                hasWindowFocusMethod, isActivityOrSubClass, isNeedTrace);
    }
}

// 匹配 Activity.onWindowFocusChanged(boolean hasFocus)
public final static String MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD = "onWindowFocusChanged";
public final static String MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD_ARGS = "(Z)V";
public static boolean isWindowFocusChangeMethod(String name, String desc) {
    return null != name && null != desc && name.equals(TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD) 
        && desc.equals(TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD_ARGS);
}

public void TraceClassAdapter.visitEnd() {
    if (!hasWindowFocusMethod && isActivityOrSubClass && isNeedTrace) {
        insertWindowFocusChangeMethod(cv, className);
    }
    super.visitEnd();
}

// 如果 Activity 没有覆盖 onWindowFocusChanged 则覆盖之（需要在里面插桩统计页面打开速度）
private void insertWindowFocusChangeMethod(ClassVisitor cv, String classname) {
    MethodVisitor methodVisitor = cv.visitMethod(Opcodes.ACC_PUBLIC, TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD,
            TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD_ARGS, null, null);
    methodVisitor.visitCode();
    methodVisitor.visitVarInsn(Opcodes.ALOAD, 0);
    methodVisitor.visitVarInsn(Opcodes.ILOAD, 1);
    methodVisitor.visitMethodInsn(Opcodes.INVOKESPECIAL, TraceBuildConstants.MATRIX_TRACE_ACTIVITY_CLASS, 
        TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD, TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD_ARGS, false);
    traceWindowFocusChangeMethod(methodVisitor, classname);
    methodVisitor.visitInsn(Opcodes.RETURN);
    methodVisitor.visitMaxs(2, 2);
    methodVisitor.visitEnd();
}

private class TraceMethodAdapter extends AdviceAdapter {
    // 进入函数后先执行 MethodBeat.i
    protected void onMethodEnter() {
        TraceMethod traceMethod = collectedMethodMap.get(methodName);
        if (traceMethod != null) {
            traceMethodCount.incrementAndGet();
            mv.visitLdcInsn(traceMethod.id);
            mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "i", "(I)V", false);

            // 在 onWindowFocusChanged 插入 AppMethodBeat.at
            if (checkNeedTraceWindowFocusChangeMethod(traceMethod)) {
                traceWindowFocusChangeMethod(mv, className);
            }
        }
    }
    
    // 退出函数前，执行 MethodBeat.o
    protected void onMethodExit(int opcode) {
        TraceMethod traceMethod = collectedMethodMap.get(methodName);
        if (traceMethod != null) {
            traceMethodCount.incrementAndGet();
            mv.visitLdcInsn(traceMethod.id);
            mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "o", "(I)V", false);
        }
    }
}

// jar 包一样的，只不过用的 ZipFile API 那套 ...
```

## 页面启动耗时

注册 `ActivityLifecycleCallbacks`，在 `onActivityCreated` 记录 `Activity` 的创建时间

在上面的插桩阶段，`AppMethodBeat.at(activity, isFocus)` 被添加到 `Activity.onWindowFocusChanged(hasFocus)` 的第一行代码，此时认为 `Activity` 获得焦点启动完毕（用户可见可交互），与 `Activity` 创建时间的差即为页面启动耗时

```java
public class StartupTracer {

    // "{Activity全限定类名}@{activity.hashCode()}" -> uptimeMillis
    private HashMap<String, Long> createdTimeMap = new HashMap<>();

    @Override
    protected void onAlive() {
        super.onAlive();
        MatrixLog.i(TAG, "[onAlive] isStartupEnable:%s", isStartupEnable);
        if (isStartupEnable) {
            AppMethodBeat.getInstance().addListener(this);                              // 添加 onWindowFocusChanged(hasFocus) 监视器
            Matrix.with().getApplication().registerActivityLifecycleCallbacks(this);    // 注册 ActivityLifecycleCallbacks
        }
    }

    @Override
    public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        MatrixLog.i(TAG, "activeActivityCount:%d, coldCost:%d", activeActivityCount, coldCost);
        if (activeActivityCount == 0 && coldCost > 0) {
            lastCreateActivity = uptimeMillis();
            MatrixLog.i(TAG, "lastCreateActivity:%d, activity:%s", lastCreateActivity, activity.getClass().getName());
            isWarmStartUp = true;
        }
        activeActivityCount++;
        if (isShouldRecordCreateTime) {
            createdTimeMap.put(activity.getClass().getName() + "@" + activity.hashCode(), uptimeMillis());  // 记录 Activity 创建时间
        }
    }

    // 当 Activity.onWindowFocusChanged(hasFocus) hasFocus == true 时被调用
    @Override
    public void onActivityFocused(Activity activity) {
        if (ActivityThreadHacker.sApplicationCreateScene == Integer.MIN_VALUE) {
            Log.w(TAG, "start up from unknown scene");
            return;
        }

        String activityName = activity.getClass().getName();
        if (isColdStartup()) {
            boolean isCreatedByLaunchActivity = ActivityThreadHacker.isCreatedByLaunchActivity();
            MatrixLog.i(TAG, "#ColdStartup# activity:%s, splashActivities:%s, empty:%b, "
                            + "isCreatedByLaunchActivity:%b, hasShowSplashActivity:%b, "
                            + "firstScreenCost:%d, now:%d, application_create_begin_time:%d, app_cost:%d",
                    activityName, splashActivities, splashActivities.isEmpty(), isCreatedByLaunchActivity,
                    hasShowSplashActivity, firstScreenCost, uptimeMillis(),
                    ActivityThreadHacker.getEggBrokenTime(), ActivityThreadHacker.getApplicationCost());

            String key = activityName + "@" + activity.hashCode();
            Long createdTime = createdTimeMap.get(key);
            if (createdTime == null) {
                createdTime = 0L;
            }
            createdTimeMap.put(key, uptimeMillis() - createdTime);      // 页面启动耗时
            // ...
    }    
}

// 当进入 Activity.onWindowFocusChanged(hasFocus) 函数时首先会执行此函数
AppMethodBeat.at(Activity activity, boolean isFocus) {
    String activityName = activity.getClass().getName();
    if (isFocus) {
        if (sFocusActivitySet.add(activityName)) {
            synchronized (listeners) {
                for (IAppMethodBeatListener listener : listeners) {
                    listener.onActivityFocused(activity);
                }
            }
            MatrixLog.i(TAG, "[at] visibleScene[%s] has %s focus!", getVisibleScene(), "attach");
        }
    } else {
        if (sFocusActivitySet.remove(activityName)) {
            MatrixLog.i(TAG, "[at] visibleScene[%s] has %s focus!", getVisibleScene(), "detach");
        }
    }
}
```

## 首屏启动耗时

第一次执行 `AppMethodBeat.i` 时（应该是 `Application.attach`），将当前时间记录为 `Application` 创建时间（`ActivityThreadHacker.sApplicationCreateBeginTime`）

首屏启动耗时（`firstScreenCost`） = 第一个页面接收到焦点的时间 - 创建 `Application` 的时间

```java
public class AppMethodBeat {
    public static void i(int methodId) {
        // ... 第一次执行 AppMethodBeat.i 时
        if (status == STATUS_DEFAULT) {
            synchronized (statusLock) {
                if (status == STATUS_DEFAULT) {
                    realExecute();
                    status = STATUS_READY;
                }
            }
        }
        // ...
    }

    private static void realExecute() {
        // ...
        ActivityThreadHacker.hackSysHandlerCallback();
        /// ...
    }        
}

public class ActivityThreadHacker {

    private static long sApplicationCreateBeginTime = 0L;
    public static long getEggBrokenTime() {
        return ActivityThreadHacker.sApplicationCreateBeginTime;
    }    

    public static void hackSysHandlerCallback() {
        sApplicationCreateBeginTime = SystemClock.uptimeMillis();   // 记录 Application 的创建时间
        // ...
    }    
}

public class StartupTracer {    
    public void onActivityFocused(Activity activity) {
        // ... 记录首屏启动耗时
        if (firstScreenCost == 0) {
            this.firstScreenCost = uptimeMillis() - ActivityThreadHacker.getEggBrokenTime();
        }
        // ...
    }
}
```

## 冷启动和热启动耗时

当打开一个 `Activity` 时，如果 app process 不存在则需要通过 `zygote` 进程 `fork` 出 app process，执行 `Application.onCreate` 后再启动 `Activity`，这叫做 **冷启动**；APP 退出后，系统在内存充足的情况下并不会立刻销毁 app process，重新打开 APP 虽然会走 `Application.onCreate` 再打开 `Activity`，但这个 `Application` 实例并没有销毁（实际上是 JVM 没有被销毁），这叫 **热启动**

怎么判断是冷启动还是热启动呢？既然 JVM 没有销毁，那么类的静态成员变量就会一直存在于内存中，判断它有没初始化即可知道是冷启动还是热启动，实际上 `Matrix` 就是这么做的

`Matrix.sInstance` -> `HashSet<Plugin> plugins` -> `TracePlugin` -> `StartupTracer` -> `coldCost`

`StartupTracer.coldCost` 会在 `StartupTracer.onActivityFocused` 被初始化（> 0）

```java
```

## 参考

1. [ASM 入门](https://www.wanandroid.com/blog/show/2937)
2. [AOP 利器 ASM 基础入门](https://github.com/dengshiwei/asm-module/blob/master/doc/blog/AOP%20%E5%88%A9%E5%99%A8%20ASM%20%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8.md)
3. [字节码及ASM入门](https://su18.org/post/Nlwq9S-Ru/)
4. [Gradle Transform API 的基本使用](https://www.jianshu.com/p/031b62d02607)
5. [Matrix Android TraceCanary](https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary)