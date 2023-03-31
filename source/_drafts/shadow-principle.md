# Application



# Activity/View

plugin.apk 里的 Activity 并没有注册在 host.apk 的清单文件里，所以 plugin Activity 并不是真正的 Android component，所以：

1. plugin view 没法通过 Activity 显示出来，除非用 `WindowManager.addView()`，但这是需要悬浮窗权限的，很显然 app 一般不可能拿到这个权限
2. 即使通过 `WindowManager.addView()` 显示出来，也无法收到 Activity 生命周期回调：`onResume`、`onPause`、`onDestroy`...
3. 无法处理涉及 Activity API 的相关调用
4. ...

于是需要用一个 `桩` 作为所有 plugin activity 的壳，这里统一称为 `shell activity`（`PluginDefaultProxyActivity`）

shell activity 实例存在于 plugin process 并注册在 host.apk 的清单文件里，它是一个正常的 Android component，plugin activity 实例存在于 plugin process，它只是一个普通的 java 对象

## lifecycle event

shell activity 是个正常的 Android component 所以会有生命周期事件，它将其转发给 plugin activity，这样 plugin 业务就可以正常执行

```java
public class PluginDefaultProxyActivity extends PluginContainerActivity

public class PluginContainerActivity extends GeneratedPluginContainerActivity implements HostActivity, HostActivityDelegator

abstract class GeneratedPluginContainerActivity extends Activity implements GeneratedHostActivityDelegator {

  GeneratedHostActivityDelegate hostActivityDelegate;

  @Override
  public void onCreate(Bundle arg0, PersistableBundle arg1) {
    if (hostActivityDelegate != null) {
      hostActivityDelegate.onCreate(arg0, arg1);
    } else {
      super.onCreate(arg0, arg1);
    }
  }

  @Override
  protected void onStart() {
    if (hostActivityDelegate != null) {
      hostActivityDelegate.onStart();
    } else {
      super.onStart();
    }
  }

  // onResume()
  // onPause()
  // onStop()
  // onDestroy()
  // onNewIntent(Intent)
  // onConfigurationChanged(Configuration)
  // ...  
}
```

## ShadowActivity

plugin activity 不是真正的 Android component，所以它的一些方法如 `Activity.setContentView(layoutRes)` 是不生效的，需要被 shadow gradle plugin 改造为继承自 `ShadowActivity`

```java
public class ShadowActivity extends PluginActivity {

    // 将 layout resource 设置到 shell activity 上才能在屏幕上显示出来
    @Override
    public void setContentView(int layoutResID) {
        if ("merge".equals(XmlPullParserUtil.getLayoutStartTagName(getResources(), layoutResID))) {
            //如果传进来的xml文件的根tag是merge时，需要特殊处理
            View decorView = hostActivityDelegator.getWindow().getDecorView();
            ViewGroup viewGroup = decorView.findViewById(android.R.id.content);
            LayoutInflater.from(this).inflate(layoutResID, viewGroup);
        } else {
            View inflate = LayoutInflater.from(this).inflate(layoutResID, null);
            hostActivityDelegator.setContentView(inflate);
        }
    }

    // plugin process 实际上是 host process 的子进程，所以 Application 是 HostApplication
    // 这里得返回 PluginApplication 才能逻辑完备
    @Override
    public final ShadowApplication getApplication() {
        return mPluginApplication;
    }


    @Override
    public void startActivityForResult(Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }

    @Override
    public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
        final Intent pluginIntent = new Intent(intent);
        pluginIntent.setExtrasClassLoader(mPluginClassLoader);
        ComponentName callingActivity = new ComponentName(getPackageName(), getClass().getName());
        // true 表示该 Intent 是为了启动插件内 Activity 的，已经被正确消费了
        // false 表示该 Intent 不是插件内的 Activity
        final boolean success = mPluginComponentLauncher.startActivityForResult(hostActivityDelegator, pluginIntent, requestCode, options, callingActivity);
        if (!success) {
            hostActivityDelegator.startActivityForResult(intent, requestCode, options);
        }
    }
}
```

## 



# ContentProvider

显然 plugin ContentProvider 是没有注册进 host AndroidManifest.xml 里的，plugin.apk 里调用 `ContentResolver.query(uri, projection, selection, selectionArgs, sortOrder)` 会找不到 uri（authorities）对应的 ContentProvider，这里的应对方式依然是使用 `桩技术`（`Stub`）

1. 运用字节码工程 `Gradle Transform` + `javassist` 在编译期将传递给 ContentResolver 的 Uri 拦截，并交由 `PluginContentProviderManager` 进行转换操作

2. 运行期在加载 plugin.apk 时，通过 `PackageManager.getPackageArchiveInfo` 解析得到所有 plugin ContentProvider 和它们对应的 authorities

3. 针对每个 plugin 都需要在 host AndroidManifest.xml 里注册一个桩 `PluginContainerContentProvider`，这是真正的 Android component

4. `PluginContentProviderManager` 在运行期动态地查找 uri authorities 部分有没对应的 plugin ContentProvider 实例，有的话把 uri 转换为 `content://{hostStub}/{originUri}`，也即将请求转发给 stub ContentProvider 处理

5. stub ContentProvider 相当于所有 plugin ContentProvider 的已注册的代理

request 流转过程：

1. query/insert/... 在运行时被重定向至 host stub ContentProvider

2. stub 找到对应 plugin ContentProvider 实例并将 request 转发给它

## gradle transform

gradle transform 实际上就是接收指定类型的输入 -> 处理 -> 输出处理后的产物，多个 gradle transform 形成一条处理链，上个节点的输出成为下个节点的输入

gradle 插件的实现类是 [ShadowPlugin](https://github.com/Tencent/Shadow/blob/master/projects/sdk/core/gradle-plugin/src/main/kotlin/com/tencent/shadow/core/gradle/ShadowPlugin.kt)，[它](https://github.com/Tencent/Shadow/tree/master/projects/sdk/core/gradle-plugin)是个标准的、使用 `java-gradle-plugin` 构建的 gradle plugin project

应用 shadow plugin 后会增加一个 flavor dimension `Shadow`：normal 表示正常构建，plugin 表示构建为插件版本

通过 `BaseExtension.registerTransform` 将 `ShadowTransform` 注册进去，它接收所有 class 文件作为输入：SCOPE_FULL_PROJECT + CONTENT_CLASS，`ShadowTransform` 包含许多方面的 transform：ApplicationTransform，ActivityTransform，ServiceTransform...

核心代码在 [ContentProviderTransform](https://github.com/Tencent/Shadow/blob/master/projects/sdk/core/transform/src/main/kotlin/com/tencent/shadow/core/transform/specific/ContentProviderTransform.kt)，基本流程为：

1. 如果是从项目源码编译出来的 class 文件，会是一个目录的结构，遍历此目录得到所有的 class 文件，如果是项目依赖则是一个 jar/aar 包 则遍历所有的 entry 找到 class 结尾的即是 class entry

2. `ClassPool.makeClass(InputStream)` 解析文件或 jar entry

3. `CodeConverter.redirectMethodCall` 将 `Uri.parse(String)` 替换为 `UriConverter.parse(String)`（它俩除了方法名不同外，签名必须一致）

4. `CodeConverter.redirectMethodCallToStatic` 将 `Uri.Builder.build()` 替换为静态方法调用 `UriConverter.build(Uri.Builder)`（第一个参数是被替换的实例对象）

5. `ContentResolver.call` 替换为静态方法调用 `UriConverter.call`

6. `ContentResolver.notifyChange` 替换为静态方法调用 `UriConverter.notifyChange`

7. `CtClass.toBytecode(DataOutputStream(outputStream))` 将修改后的 class 写回至 class 文件 or jar entry

拦截这些方法调用的目的都是将原始 Uri 替换为 `content://{hostStub}/{originUri}`

```kotlin
abstract class ClassTransform(val project: Project) : Transform() {

    val inputSet: MutableSet<TransformInput> = mutableSetOf()

    /**
     * 遍历目录 or jar，将其中的 class 记录至 inputSet 待处理
     */
    fun input(
        inputs: Collection<com.android.build.api.transform.TransformInput>,
        outputProvider: TransformOutputProvider
    ) {
        val logger = project.logger
        if (logger.isInfoEnabled) {
            val sb = StringBuilder()
            sb.appendln()
            inputs.forEach {
                it.directoryInputs.forEach {
                    sb.appendln(it.file.absolutePath)
                }
                it.jarInputs.forEach {
                    sb.appendln(it.file.absolutePath)
                }
            }
            logger.info("ClassTransform input paths:$sb")
        }

        // 目录，class 以文件形式存在
        inputs.forEach {
            it.directoryInputs.forEach {
                val inputDir = it.file
                val transformInput = TransformInput(it)
                inputSet.add(transformInput)
                val allFiles = FileUtils.getAllFiles(it.file)
                allFiles.filter {
                    it?.name?.endsWith(SdkConstants.DOT_CLASS) ?: false
                }.forEach {
                    val inputClass = DirInputClass()
                    inputClass.onInputClass(
                        it,
                        it.toOutputFile(inputDir, transformInput.toOutput(outputProvider))
                    )
                    transformInput.addInputClass(inputClass)
                }
            }

            // jar 包，class 以 jar entry 形式存在
            it.jarInputs.forEach {
                val transformInput = TransformInput(it)
                inputSet.add(transformInput)
                ZipInputStream(FileInputStream(it.file)).use { zis ->
                    var entry: ZipEntry?
                    while (true) {
                        entry = zis.nextEntry
                        if (entry == null) break

                        val name = entry.name

                        // 忽略一些实际上不会进入编译classpath的文件
                        if (entry.isDirectory) continue
                        if (!name.endsWith(SdkConstants.DOT_CLASS)) continue
                        if (name.startsWith("META-INF/", true)) continue
                        if (name.endsWith("module-info.class", true)) continue
                        if (name.endsWith("package-info.class", true)) continue

                        // 记录好entry和name的关系，添加再添加成transform的输入
                        val inputClass = JarInputClass()
                        inputClass.onInputClass(zis, name)
                        transformInput.addInputClass(inputClass)
                    }
                }
            }
        }
    }

    // 写操作，将处理后的 class 写入对应的文件 / jar 包
    fun output(outputProvider: TransformOutputProvider) {
        inputSet.forEach { input ->
            when (input.format) {
                Format.DIRECTORY -> {
                    input.getInputClass().forEach {
                        val dirInputClass = it as DirInputClass
                        dirInputClass.getOutput().forEach {
                            val className = it.first
                            val file = it.second
                            Files.createParentDirs(file)
                            FileOutputStream(file).use {
                                onOutputClass(null, className, it) // see JavassistTransform
                            }
                        }
                    }
                }
                Format.JAR -> {
                    val outputJar = input.toOutput(outputProvider)
                    Files.createParentDirs(outputJar)
                    ZipOutputStream(FileOutputStream(outputJar)).use { zos ->
                        input.getInputClass().forEach {
                            val jarInputClass = it as JarInputClass
                            jarInputClass.getOutput().forEach {
                                val className = it.first
                                val entryName = it.second
                                zos.putNextEntry(ZipEntry(entryName))
                                onOutputClass(entryName, className, zos) // see AbstractTransform
                            }
                        }
                    }
                }
            }
        }
    }

    // 接收所有的 class 文件作为输入，包括本项目和子项目的源码、以及依赖包

    // ImmutableSet.of(CLASSES);
    override fun getInputTypes(): MutableSet<ContentType> = TransformManager.CONTENT_CLASS

    // ImmutableSet.of(Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES);
    override fun getScopes(): MutableSet<in Scope> = TransformManager.SCOPE_FULL_PROJECT

    override fun isIncremental(): Boolean = false

    // 从输入里收集所有 class 文件 -> 处理 -> 写入输出目录
    final override fun transform(invocation: TransformInvocation) {
        beforeTransform(invocation)
        input(invocation.inputs, invocation.outputProvider)
        onTransform()
        output(invocation.outputProvider)
        afterTransform(invocation)
    }    
}

// 输出 class 文件
open class JavassistTransform(project: Project, val classPoolBuilder: ClassPoolBuilder) :
    ClassTransform(project) {
    val mCtClassInputMap = mutableMapOf<CtClass, InputClass>()
    lateinit var classPool: ClassPool

    override fun onOutputClass(entryName: String?, className: String, outputStream: OutputStream) {
        classPool[className].writeOut(outputStream)
    }

    fun CtClass.writeOut(output: OutputStream) {
        this.toBytecode(java.io.DataOutputStream(output))
    }
}

// 如果是 jar 包里的 class，需要重建整个 jar 包并将 class 逐个写入
abstract class AbstractTransform(
    project: Project,
    classPoolBuilder: ClassPoolBuilder
) : JavassistTransform(project, classPoolBuilder) {

    protected abstract val mTransformManager: AbstractTransformManager
    private val mOverrideCheck = OverrideCheck()
    private lateinit var mDebugClassJar: File
    private lateinit var mDebugClassJarZOS: ZipOutputStream

    override fun onOutputClass(entryName: String?, className: String, outputStream: OutputStream) {
        classPool[className].debugWriteJar(entryName, mDebugClassJarZOS)
        super.onOutputClass(entryName, className, outputStream)
    }

    private fun CtClass.debugWriteJar(outputEntryName: String?, outputStream: ZipOutputStream) {
        try {
            val entryName = outputEntryName ?: (name.replace('.', '/') + ".class")
            outputStream.putNextEntry(ZipEntry(entryName))
            val p = stopPruning(true)
            toBytecode(DataOutputStream(outputStream))
            defrost()
            stopPruning(p)
        } catch (e: Exception) {
            outputStream.close()
            throw RuntimeException(e)
        }
    }
}

class ContentProviderTransform : SpecificTransform() {

    companion object {
        const val ShadowUriClassname = "com.tencent.shadow.core.runtime.UriConverter"
        const val AndroidUriClassname = "android.net.Uri"
        const val uriBuilderName = "android.net.Uri\$Builder"
        const val resolverName = "android.content.ContentResolver"
    }

    // 将 Uri.parse(String) 替换为 UriConverter.parse(String)
    private fun prepareUriParseCodeConverter(classPool: ClassPool): CodeConverter {
        val uriMethod = mClassPool[AndroidUriClassname].methods!!
        val shadowUriMethod = mClassPool[ShadowUriClassname].methods!!

        val method_parse = uriMethod.filter { it.name == "parse" }
        val shadow_method_parse = shadowUriMethod.filter { it.name == "parse" }!!
        val codeConverter = CodeConverter()

        for (ctAndroidMethod in method_parse) {
            for (ctShadowMedthod in shadow_method_parse) {
                if (ctAndroidMethod.methodInfo.descriptor == ctShadowMedthod.methodInfo.descriptor) {
                    codeConverter.redirectMethodCall(ctAndroidMethod, ctShadowMedthod)
                }
            }
        }
        return codeConverter
    }

    // 将 Uri.Builder.build() 替换为 UriConverter.build(Uri.Builder)
    private fun prepareUriBuilderCodeConverter(classPool: ClassPool): CodeConverter {
        val uriClass = mClassPool[AndroidUriClassname]
        val uriBuilderClass = mClassPool[uriBuilderName]
        val buildMethod = uriBuilderClass.getMethod("build", Descriptor.ofMethod(uriClass, null))
        val newBuildMethod = mClassPool[ShadowUriClassname].getMethod(
            "build",
            Descriptor.ofMethod(uriClass, arrayOf(uriBuilderClass))
        )
        val codeConverter = CodeConverter()
        codeConverter.redirectMethodCallToStatic(buildMethod, newBuildMethod)
        return codeConverter
    }

    // 将 ContentResolver.call 替换为 UriConverter.call
    // 将 ContentResolver.notifyChange 替换为 UriConverter.notifyChange
    private fun prepareContentResolverCodeConverter(classPool: ClassPool): CodeConverter {
        val codeConverter = CodeConverter()
        val resolverClass = classPool[resolverName]
        val targetClass = classPool[ShadowUriClassname]
        val uriClass = classPool["android.net.Uri"]
        val stringClass = classPool["java.lang.String"]
        val bundleClass = classPool["android.os.Bundle"]
        val observerClass = classPool["android.database.ContentObserver"]

        val callMethod = resolverClass.getMethod(
            "call", Descriptor.ofMethod(
                bundleClass,
                arrayOf(uriClass, stringClass, stringClass, bundleClass)
            )
        )
        val newCallMethod = targetClass.getMethod(
            "call", Descriptor.ofMethod(
                bundleClass,
                arrayOf(resolverClass, uriClass, stringClass, stringClass, bundleClass)
            )
        )
        codeConverter.redirectMethodCallToStatic(callMethod, newCallMethod)

        val notifyMethod1 = resolverClass.getMethod(
            "notifyChange", Descriptor.ofMethod(
                CtClass.voidType,
                arrayOf(uriClass, observerClass)
            )
        )
        val newNotifyMethod1 = targetClass.getMethod(
            "notifyChange", Descriptor.ofMethod(
                CtClass.voidType,
                arrayOf(resolverClass, uriClass, observerClass)
            )
        )
        codeConverter.redirectMethodCallToStatic(notifyMethod1, newNotifyMethod1)

        val notifyMethod2 = resolverClass.getMethod(
            "notifyChange", Descriptor.ofMethod(
                CtClass.voidType,
                arrayOf(uriClass, observerClass, CtClass.booleanType)
            )
        )
        val newNotifyMethod2 = targetClass.getMethod(
            "notifyChange", Descriptor.ofMethod(
                CtClass.voidType,
                arrayOf(resolverClass, uriClass, observerClass, CtClass.booleanType)
            )
        )
        codeConverter.redirectMethodCallToStatic(notifyMethod2, newNotifyMethod2)

        val notifyMethod3 = resolverClass.getMethod(
            "notifyChange", Descriptor.ofMethod(
                CtClass.voidType,
                arrayOf(uriClass, observerClass, CtClass.intType)
            )
        )
        val newNotifyMethod3 = targetClass.getMethod(
            "notifyChange", Descriptor.ofMethod(
                CtClass.voidType,
                arrayOf(resolverClass, uriClass, observerClass, CtClass.intType)
            )
        )
        codeConverter.redirectMethodCallToStatic(notifyMethod3, newNotifyMethod3)

        return codeConverter
    }

    override fun setup(allInputClass: Set<CtClass>) {

        val uriParseCodeConverter = prepareUriParseCodeConverter(mClassPool)
        val uriBuilderCodeConverter = prepareUriBuilderCodeConverter(mClassPool)
        val contentResolverCodeConverter = prepareContentResolverCodeConverter(mClassPool)

        newStep(object : TransformStep {
            override fun filter(allInputClass: Set<CtClass>) =
                filterRefClasses(allInputClass, listOf(AndroidUriClassname))

            override fun transform(ctClass: CtClass) {
                try {
                    ctClass.instrument(uriParseCodeConverter)
                } catch (e: Exception) {
                    System.err.println("处理" + ctClass.name + "时出错")
                    throw e
                }
            }
        })

        newStep(object : TransformStep {
            override fun filter(allInputClass: Set<CtClass>) =
                filterRefClasses(allInputClass, listOf(uriBuilderName))

            override fun transform(ctClass: CtClass) {
                try {
                    ctClass.instrument(uriBuilderCodeConverter)
                } catch (e: Exception) {
                    System.err.println("处理" + ctClass.name + "时出错")
                    throw e
                }
            }
        })

        newStep(object : TransformStep {
            override fun filter(allInputClass: Set<CtClass>) =
                filterRefClasses(allInputClass, listOf(resolverName))

            override fun transform(ctClass: CtClass) {
                try {
                    ctClass.instrument(contentResolverCodeConverter)
                } catch (e: Exception) {
                    System.err.println("处理" + ctClass.name + "时出错")
                    throw e
                }
            }
        })
    }
}
```

## 初始化 plugin ContentProvider

`PluginContentProviderManager` 是管理 plugin ContentProvider 的核心类

通过 `PackageManager.getPackageArchiveInfo` 解析 plugin.apk 得到所有 plugin ContentProvider 的信息

`ContentProvider` 是在 `Application.attachBaseContext` 之后，`application.onCreate` 之前初始化的

`ClassLoader.loadClass(...)` 将 plugin provider 实例化出来，然后调用 `ContentProvider.attachInfo(Context, ProviderInfo)` 初始化（里面会调用 `ContentProvider.onCreate`），参数 Context 自然是 plugin application 而不是 host application，参数 ProviderInfo 填充从 plugin apk 解析得到的清单内容

用一个 Map 将 authorities 和 ContentProvider 映射起来

```kotlin
// com.tencent.shadow.core.loader.ShadowPluginLoader#callApplicationOnCreate
fun callApplicationOnCreate(partKey: String) {
    fun realAction() {
        val pluginParts = getPluginParts(partKey)
        pluginParts?.let {
            val application = pluginParts.application
            application.attachBaseContext(mHostAppContext)
            mPluginContentProviderManager.createContentProviderAndCallOnCreate(
                application, partKey, pluginParts
            )
            application.onCreate()
        }
    }
    if (isUiThread()) {
        realAction()
    } else {
        val waitUiLock = CountDownLatch(1)
        mUiHandler.post {
            realAction()
            waitUiLock.countDown()
        }
        waitUiLock.await();
    }
}

class PluginContentProviderManager() : UriConverter.UriParseDelegate {

    /**
     * key : pluginAuthority
     * value : plugin ContentProvider
     */
    private val providerMap = HashMap<String, ContentProvider>()

    fun createContentProviderAndCallOnCreate(
        context: Context,
        partKey: String,
        pluginParts: PluginParts?
    ) {
        pluginProviderInfoMap[partKey]?.forEach {
            try {
                val contentProvider = pluginParts!!.appComponentFactory
                    .instantiateProvider(pluginParts.classLoader, it.className)

                //convert PluginManifest.ProviderInfo to android.content.pm.ProviderInfo
                val providerInfo = ProviderInfo()
                providerInfo.packageName = context.packageName
                providerInfo.name = it.className
                providerInfo.authority = it.authorities
                providerInfo.grantUriPermissions = it.grantUriPermissions
                contentProvider?.attachInfo(context, providerInfo)
                providerMap[it.authorities] = contentProvider
            } catch (e: Exception) {
                throw RuntimeException(
                    "partKey==$partKey className==${it.className} authorities==${it.authorities}",
                    e
                )
            }
        }
    }
}
```

## stub in host

在 host AndroidManifest.xml 里需要给 plugin 配置一个 stub ContentProvider 如下

其中 `android:authorities` 部分需要与 plugin loader project 里的实现一致，这样 plugin uri 才能正确转换为 host uri

stub 必须与 plugin 在同一进程否则找不到 plugin ContentProvider 实例对象

```xml
<provider
    android:authorities="${applicationId}.contentprovider.authority.dynamic"
    android:name="com.tencent.shadow.core.runtime.container.PluginContainerContentProvider"
    android:grantUriPermissions="true"
    android:process=":plugin" />
```

```java
public class SampleComponentManager extends ComponentManager {
    /**
     * 配置对应宿主中预注册的壳子contentProvider的信息
     */
    @Override
    public ContainerProviderInfo onBindContainerContentProvider(ComponentName pluginContentProvider) {
        return new ContainerProviderInfo(
                "com.tencent.shadow.core.runtime.container.PluginContainerContentProvider",
                context.getPackageName() + ".contentprovider.authority.dynamic");
    }
}
```

## PluginContainerContentProvider

`PluginContainerContentProvider` 就是一个没有任何业务代码的壳，它将所有方法调用委托至 `ShadowContentProviderDelegate`

上面说过 `PluginContentProviderManager` 是管理核心，里面有 authorities -> ContentProvider 的映射，那么 `ShadowContentProviderDelegate` 就可以根据 plugin authorities 找到对应的 ContentProvider 实例对象

比如定义在 plugin AndroidManiest.xml 里的是 `${applicationId}.provider.test`，虽然 plugin package 是 `com.tencent.shadow.sample.plugin.app` 但会被 shadow plugin 在编译器改为 host package `com.tencent.shadow.sample.host`，所以打包出来的 plugin.apk AndroidManifest.xml 里其实是 `com.tencent.shadow.sample.host.provider.test`

某个 Uri 运行时会被转换为 `content://com.tencent.shadow.sample.host.contentprovider.authority.dynamic/com.tencent.shadow.sample.host.provider.test/test`，移除 host stub authorities 部分就是 `content://com.tencent.shadow.sample.host.provider.test/test` 从而在 `PluginContentProviderManager` 匹配到配置在 AndroidManifest.xml 里的对应的 ContentProvider 实例

```kotlin
class ShadowContentProviderDelegate(private val mProviderManager: PluginContentProviderManager) :
    ShadowDelegate(), HostContentProviderDelegate {

    override fun query(
        uri: Uri,
        projection: Array<String>?,
        selection: String?,
        selectionArgs: Array<String>?,
        sortOrder: String?
    ): Cursor? {
        // 移除 host stub authorities 部分就是定义在 plugin AndroidManifest.xml 里的真实 plugin ContentProvider authorities
        val pluginUri = mProviderManager.convert2PluginUri(uri)
        return mProviderManager.getPluginContentProvider(pluginUri.authority!!)!!
            .query(pluginUri, projection, selection, selectionArgs, sortOrder)
    }

    override fun onLowMemory() {
        mProviderManager.getAllContentProvider().forEach {
            it.onLowMemory()
        }
    }

    override fun onTrimMemory(level: Int) {
        mProviderManager.getAllContentProvider().forEach {
            it.onTrimMemory(level)
        }
    }    
}
```



# 解析 apk 获得清单内容

Via 浏览器，mark.via，4.5.1，20230222

```xml
<provider
    android:label="@ref/0x7f0f0007"
    android:name="mark.via.provider.BookmarksProvider"
    android:exported="true"
    android:multiprocess="false"
    android:authorities="mark.via.database" />

<provider
    android:name="androidx.core.content.FileProvider"
    android:exported="false"
    android:authorities="mark.via.provider"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@ref/0x7f120001" />
</provider>

<provider
    android:name="com.flurry.android.agent.FlurryContentProvider"
    android:exported="false"
    android:authorities="mark.via.FlurryContentProvider" />
```

```kotlin
val apkFile = File(getExternalFilesDir(null), "via.apk")
if (!apkFile.exists())
    apkFile.createNewFile()
apkFile.outputStream().use { out ->
    assets.open("via.apk").use { it.copyTo(out) }
}
packageManager.getPackageArchiveInfo(apkFile.absolutePath, PackageManager.GET_PROVIDERS)?.providers?.forEach {
    Log.d("cyrus", "${it.packageName} - ${it.name} - ${it.authority}")
}

// logcat:
// mark.via - mark.via.provider.BookmarksProvider - mark.via.database
// mark.via - androidx.core.content.FileProvider - mark.via.provider
// mark.via - com.flurry.android.agent.FlurryContentProvider - mark.via.FlurryContentProvider
```