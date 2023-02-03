# SPI

SPI 全称 `Service Provider Interface`，是 Java/Android 内置于标准库里的一个 `服务发现` 机制 `ServiceLoader`

```kotlin
// 1）准备接口及相关实现

package work.dalvik.binder.myapplication

interface ISimpleInterface {
    val name: String
}

class SimplePlan: ISimpleInterface {
    override val name: String
        get() = "Plan"
}

class SimpleProject: ISimpleInterface {
    override val name: String
        get() = "Project"
}

class SimpleRobot: ISimpleInterface {
    override val name: String
        get() = "Robot"
}

// 2）在对应的文件里手动注册接口和实现
// 项目路径：app/src/main/resources/META-INF/services/work.dalvik.binder.myapplication.ISimpleInterface
// 打包后：/META-INF/services/work.dalvik.binder.myapplication.ISimpleInterface

work.dalvik.binder.myapplication.SimpleRobot
work.dalvik.binder.myapplication.SimpleProject
work.dalvik.binder.myapplication.SimplePlan

// 3）运行时发现服务

val services = ServiceLoader.load(ISimpleInterface::class.java)
for (service in services) {
    Log.d("cyrus", "${service.name} from ${service::class.java}")
}

// 11:23:39.785  5764-5764  cyrus D  Robot from class work.dalvik.binder.myapplication.SimpleRobot
// 11:23:39.785  5764-5764  cyrus D  Project from class work.dalvik.binder.myapplication.SimpleProject
// 11:23:39.786  5764-5764  cyrus D  Plan from class work.dalvik.binder.myapplication.SimplePlan
```

# Google AutoService

简化 [SPI](#spi) 的使用：使用注解 `@AutoService` 替换手动的 SPI 配置文件管理

```kotlin
// 1）加入依赖

dependencies {
    kapt 'com.google.auto.service:auto-service:1.0.1'
    implementation 'com.google.auto.service:auto-service-annotations:1.0.1'
}

// 2）加上注解 @AutoService

package work.dalvik.binder.myapplication

import com.google.auto.service.AutoService

interface ISimpleInterface {
    val name: String
}

@AutoService(ISimpleInterface::class)
class SimplePlan: ISimpleInterface {
    override val name: String
        get() = "Plan"
}

@AutoService(ISimpleInterface::class)
class SimpleProject: ISimpleInterface {
    override val name: String
        get() = "Project"
}

@AutoService(ISimpleInterface::class)
class SimpleRobot: ISimpleInterface {
    override val name: String
        get() = "Robot"
}

// 3）不再需要手动增加/删除 SPI 配置文件里的内容

// 4）运行时发现服务

val services = ServiceLoader.load(ISimpleInterface::class.java)
for (service in services) {
    Log.d("cyrus", "${service.name} from ${service::class.java}")
}

// 11:34:01.747  6141-6141  cyrus D  Plan from class work.dalvik.binder.myapplication.SimplePlan
// 11:34:01.748  6141-6141  cyrus D  Project from class work.dalvik.binder.myapplication.SimpleProject
// 11:34:01.749  6141-6141  cyrus D  Robot from class work.dalvik.binder.myapplication.SimpleRobot
```

# 引入 booster 框架

```groovy
buildscript {

    dependencies {

        classpath "com.didiglobal.booster:booster-gradle-plugin:$booster_version"
        
        // booster 是模块化的，每个功能 / 特性都是一个单独的依赖项，需要哪项特性把对应的依赖项加入进来即可
        // booster-gradle-plugin 通过 SPI / AutoService 发现并执行这些服务
        classpath "com.didiglobal.booster:booster-task-compression-cwebp:$booster_version"
        classpath "com.didiglobal.booster:booster-transform-shared-preferences:$booster_version"
        classpath "com.didiglobal.booster:booster-task-analyser:$booster_version"
        // ...
    }
}

apply plugin: 'com.didiglobal.booster'
```

# webp 格式化

## 注册 cwebp gradle task

[variant](https://developer.android.com/studio/build/build-variants) 指代构建的一个输出：apk or jar，它是 build type 和 product flavor 的组合，比如 demoDebug、productRelease 等等 

`VariantProcessor` 是 booster API，本模块入口 `CwebpCompressionVariantProcessor` 通过 [AutoService](#google-autoservice) 注册为它的一个实现，然后 booster-gradle-plugin 在运行时通过 [ServiceLoader](#spi) 发现并执行 cwebp 模块

实现类是 `CwebpCompressOpaqueFlatImages`、`CwebpCompressFlatImages`、`CwebpCompressOpaqueImages` or `CwebpCompressImages`（根据 aapt2 和 alpha 的支持情况四选一）

依赖图：

- 在这些 task 后执行
    - `pre-build tasks`
    - `merge[XXX]Resources task`
    - `install cwebp executable`

- 在这些 task 前执行
    - `process[XXX]Resources tasks`
    - `bundle[XXX]Resources tasks`

```kotlin
// https://github.com/didi/booster/blob/master/booster-task-compression-cwebp/src/main/kotlin/com/didiglobal/booster/task/compression/cwebp/CwebpCompressionVariantProcessor.kt
@AutoService(VariantProcessor::class)
class CwebpCompressionVariantProcessor : VariantProcessor {

    override fun process(variant: BaseVariant) {  // 处理各种变体（构建任务）
        val project = variant.project
        val results = CompressionResults()
        val ignores = project.findProperty(PROPERTY_IGNORES)?.toString()?.trim()?.split(',')?.map {
            Wildcard(it)
        }?.toSet() ?: emptySet()

        Cwebp.get(variant)?.newCompressionTaskCreator()?.createCompressionTask(variant, results, "resources", {
            variant.mergedRes.search(if (project.isAapt2Enabled) ::isFlatPngExceptRaw else ::isPngExceptRaw)
        }, ignores, variant.mergeResourcesTaskProvider)?.configure {
            it.doLast {
                results.generateReport(variant, Build.ARTIFACT)
            }
        }
    }
}

class SimpleCompressionTaskCreator(
    private val tool: CompressionTool,  /* Cwebp */
    private val compressor: (Boolean) -> KClass<out CompressImages<out CompressionOptions>>
) : CompressionTaskCreator {

    override fun createCompressionTask(
            variant: BaseVariant,
            results: CompressionResults,
            name: String,                       // resources
            supplier: () -> Collection<File>,   // variant.mergedRes.search(if (project.isAapt2Enabled) ::isFlatPngExceptRaw else ::isPngExceptRaw)
            ignores: Set<Wildcard>,             // emptySet()
            vararg deps: TaskProvider<out Task> // variant.mergeResourcesTaskProvider
    ): TaskProvider<out CompressImages<out CompressionOptions>> {

        val project = variant.project
        val aapt2 = project.isAapt2Enabled
        val install = getCommandInstaller(variant)

        // TaskContainer.register(String name, Class<T> type, Action<? super T> configurationAction)
        // 往 project 注册了一个 cwebp task
        // 名称是 compress[DemoRelease]ResourcesWith[CwebpCompressOpaqueFlatImages]（显示在 Android Studio 的 Build 窗口里）
        // 实现类是 CwebpCompressOpaqueFlatImages
        return project.tasks.register(
            "compress${variant.name.capitalize()}${name.capitalize()}With${tool.command.name.substringBefore('.').capitalize()}", 
            getCompressionTaskClass(aapt2).java

        // 配置 cwebp task 的依赖图，确定它的执行次序
        ) { task ->
            task.group = BOOSTER
            task.description = "Compress image resources by ${tool.command.name} for ${variant.name}"
            task.dependsOn(variant.preBuildTaskProvider.get())    // 在所有的 pre-build tasks 之后执行
            task.tool = tool
            task.variant = variant
            task.results = results
            task.filter = if (ignores.isNotEmpty()) excludes(ignores) else MATCH_ALL_RESOURCES
            task.images = lazy(supplier)::value
        }.apply {
            dependsOn(install)                                      // 在安装 cwebp 可执行文件之后执行
            deps.forEach { dependsOn(it) }                          // 在所有 merge[XXX]Resources task 之后执行
            variant.processResTaskProvider?.dependsOn(this)         // 在所有 process[XXX]Resources task 前执行
            variant.bundleResourcesTaskProvider?.dependsOn(this)    // 在所有 bundle[XXX]Resources task 前执行
        }
    }

    // 注册一个 install cwebp task
    // 名称是 install[DemoRelease]Cwebp
    // 实现类是 CommandInstaller
    private fun getCommandInstaller(variant: BaseVariant): TaskProvider<out Task> {
        return variant.project.tasks.register(getInstallTaskName(variant.name)) {
            it.group = BOOSTER
            it.description = "Install ${tool.command.name} for ${variant.name}"
        }.apply {
            dependsOn(getCommandInstaller(variant.project))
            dependsOn(variant.mergeResourcesTaskProvider)    // 在所有 merge[XXX]Resources task 之后执行
        }
    }

    // cwebp 可执行文件每次构建只释放一次即可
    private fun getCommandInstaller(project: Project): TaskProvider<out Task> {
        val name = getInstallTaskName()
        return try {
            project.tasks.named(name)
        } catch (e: UnknownTaskException) {
            null
        } ?: project.tasks.register(name, CommandInstaller::class.java) {
            it.group = BOOSTER
            it.description = "Install ${tool.command.name}"
            it.command = tool.command
        }
    }

    private fun getInstallTaskName(variant: String = ""): String {
        @Suppress("DEPRECATION")
        return "install${variant.capitalize()}${tool.command.name.substringBefore('.').capitalize()}"
    }    
}

// 根据是否支持 aapt2、是否支持 alpha 等情况有四种核心实现，这里选取支持 aapt2 和 alpha 的 CwebpCompressOpaqueFlatImages 为例
class Cwebp internal constructor(val supportAlpha: Boolean) : CompressionTool(CommandService.get(CWEBP)) {
    override fun newCompressionTaskCreator() = SimpleCompressionTaskCreator(this) { aapt2 ->
        when (aapt2) {
            true -> when (supportAlpha) {
                true -> CwebpCompressOpaqueFlatImages::class
                else -> CwebpCompressFlatImages::class
            }
            else -> when (supportAlpha) {
                true -> CwebpCompressOpaqueImages::class
                else -> CwebpCompressImages::class
            }
        }
    }
}
```

## install cwebp executable task

cwebp 可执行文件被打包进 aar，然后在 gradle task 期间被释放到 build dir 使用

[cwebp_executable.png](/image/2023-01-17-booster-resources/cwebp_executable.png)

```kotlin
// 此 task 在上一章节的 getCommandInstaller 方法里被注册进去
@CacheableTask
open class CommandInstaller : DefaultTask() {

    @get:Input
    lateinit var command: Command

    @get:OutputFile
    val location: File
        get() = project.buildDir.file("bin", command.name)

    @TaskAction
    fun install() {
        logger.info("Installing $command => $location")

        this.command.location.openStream().buffered().use { input ->
            FileUtils.copyInputStreamToFile(input, location)  // 将 cwebp 拷贝一份到 build dir
            project.exec {                                    // 然后给 cwebp 增加 executable 权限
                it.commandLine = when {
                    OS.isLinux() || OS.isMac() -> listOf("chmod", "+x", location.canonicalPath)
                    OS.isWindows() -> listOf("cmd", "/c echo Y|cacls ${location.canonicalPath} /t /p everyone:f")
                    else -> TODO("Unsupported OS ${OS.name}")
                }
            }
        }
    }
}

open class Command(
    val name: String,  // 可执行文件的名称：cwebp or cwebp.exe(windows)
    val location: URL  // 可执行文件的相对位置（如上图）：ClassLoader.getResource("bin/linux/x64 | bin/macosx/[10.1x] | windows/x64")
) : Serializable {

    override fun equals(other: Any?) = when {
        this === other -> true
        other is Command -> name == other.name && location == other.location
        else -> false
    }

    @Throws(IOException::class)
    open fun execute(vararg args: String) {
        Runtime.getRuntime().exec(arrayOf(location.file.let(::File).canonicalPath) + args).let { p ->
            p.waitFor()
            if (p.exitValue() != 0) {
                throw IOException(p.stderr)
            }
        }
    }

    override fun hashCode(): Int {
        return arrayOf(name, location).contentHashCode()
    }

    override fun toString() = "$name:$location"

}
```

## cwebp task

- 找到数据源：resource 文件集

在 merge resource 之后

- 过滤出待处理的图片资源集

- cwebp 转换图片格式

- aapt2 编译

```kotlin
// 继承关系
CwebpCompressOpaqueFlatImages
  - CwebpCompressFlatImages
    - AbstractCwebpCompressImages
      - CompressImages<CompressionOptions>
        - org.gradle.api.DefaultTask

@CacheableTask
internal abstract class CwebpCompressOpaqueFlatImages {

    // 格式化后的 webp 文件的输出目录：
    // build/intermediates/compressed_res_cwebp/release/demo/compressDemoReleaseResourcesWithCwebpCompressOpaqueFlatImages
    @get:OutputDirectory
    val compressedRes: File
        get() = variant.project.buildDir.file(FD_INTERMEDIATES).file("compressed_${FD_RES}_cwebp", variant.dirName, this.name)

    @get:Internal
    val compressor: File
        get() = project.tasks.withType(CommandInstaller::class.java).find {
            it.command == tool.command
        }!!.location        

    @TaskAction
    fun run() {
        this.options = CompressionOptions(project.getProperty(PROPERTY_OPTION_QUALITY, 80))
        compress(File::hasNotAlpha)
    }

    override fun compress(filter: (File) -> Boolean /* always true for CwebpCompressOpaqueFlatImages */) {
        val cwebp = this.compressor.canonicalPath  // cwebp 可执行文件的位置
        val aapt2 = variant.buildTools.getPath(BuildToolInfo.PathId.AAPT2)
        val parser = SAXParserFactory.newInstance().newSAXParser()
        val icons = variant.mergedManifests.search {
            it.name == SdkConstants.ANDROID_MANIFEST_XML
        }.parallelStream().map { manifest ->
            LauncherIconHandler().let {
                parser.parse(manifest, it)
                it.icons
            }
        }.flatMap {
            it.parallelStream()
        }.collect(Collectors.toSet())

        // Google Play only accept APK with PNG format launcher icon
        // https://developer.android.com/topic/performance/reduce-apk-size#use-webp
        val isNotLauncherIcon: (Pair<File, Aapt2Container.Metadata>) -> Boolean = { (input, metadata) ->
            if (!icons.contains(metadata.resourceName)) true else false.also {
                ignore(metadata.resourceName, input, File(metadata.sourcePath))
            }
        }

        // images() == variant.mergedRes.search(if (project.isAapt2Enabled) ::isFlatPngExceptRaw else ::isPngExceptRaw)
        // variant.mergedRes 是数据源
        // if (project.isAapt2Enabled) ::isFlatPngExceptRaw else ::isPngExceptRaw 是主要的过滤器
        images().parallelStream().map {
            it to it.metadata
        }.filter(this::includes).filter(isNotLauncherIcon).filter {
            filter(File(it.second.sourcePath))
        }.map {
            // 放到输出目录，文件名不变，后缀改为 webp
            val output = compressedRes.file("${it.second.resourcePath.substringBeforeLast('.')}.webp")
            Aapt2ActionData(it.first, it.second, output,
                    listOf(cwebp, "-mt", "-quiet", "-q", options.quality.toString(), it.second.sourcePath, "-o", output.canonicalPath),
                    listOf(aapt2, "compile", "-o", it.first.parent, output.canonicalPath))
        }.forEach {
            it.output.parentFile.mkdirs()  // 为目标文件创建必须的目录
            val s0 = File(it.metadata.sourcePath).length()

            // cwebp 进行格式转换
            // Executes an external command.
            // The given action configures a {@link org.gradle.process.ExecSpec}, which is used to launch the process.
            // This method blocks until the process terminates, with its result being returned.
            // Params:
            //   action – The action for configuring the execution.
            // Returns:
            //   the result of the execution
            // ExecResult Project.exec(Action<? super ExecSpec> action);
            val rc = project.exec { spec ->
                spec.isIgnoreExitValue = true
                spec.commandLine = it.cmdline  // build/bin/cwebp -mt -quiet -q 80 [src] -o [output]
            }
            when (rc.exitValue) {
                0 -> {  // webp 格式转换成功
                    val s1 = it.output.length()
                    if (s1 > s0) {
                        results.add(CompressionResult(it.input, s0, s0, File(it.metadata.sourcePath)))
                        it.output.delete()
                    } else {

                        // aapt2 编译
                        val rcAapt2 = project.exec { spec ->
                            spec.isIgnoreExitValue = true
                            spec.commandLine = it.aapt2
                        }
                        if (0 == rcAapt2.exitValue) {
                            results.add(CompressionResult(it.input, s0, s1, File(it.metadata.sourcePath)))
                            it.input.delete()
                        } else {
                            logger.error("${CSI_RED}Command `${it.aapt2.joinToString(" ")}` exited with non-zero value ${rc.exitValue}$CSI_RESET")
                            results.add(CompressionResult(it.input, s0, s0, File(it.metadata.sourcePath)))
                            rcAapt2.assertNormalExitValue()
                        }
                    }
                }
                else -> {  // 格式转换失败
                    logger.error("${CSI_RED}Command `${it.cmdline.joinToString(" ")}` exited with non-zero value ${rc.exitValue}$CSI_RESET")
                    results.add(CompressionResult(it.input, s0, s0, File(it.metadata.sourcePath)))
                    it.output.delete()
                }
            }
        }
    }

}
```

## 用到的 gradle 注解

```kotlin

/**
 * <p>Attached to a task type to indicate that task output caching should be enabled by default for tasks of this type.</p>
 *
 * <p>Only tasks that produce reproducible and relocatable output should be marked with {@code CacheableTask}.</p>
 *
 * <p>Caching for individual task instances can be enabled and disabled via {@link TaskOutputs#cacheIf(String, Spec)} or disabled via {@link TaskOutputs#doNotCacheIf(String, Spec)}.
 * Using these APIs takes precedence over the presence (or absence) of {@code @CacheableTask}.</p>
 *
 * @see DisableCachingByDefault
 *
 * @since 3.0
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface CacheableTask {
}

/**
 * <p>Attached to a task property to indicate that the property specifies some input value for the task.</p>
 *
 * <p>This annotation should be attached to the getter method in Java or the property in Groovy.
 * Annotations on setters or just the field in Java are ignored.</p>
 *
 * <p>This will cause the task to be considered out-of-date when the property has changed.
 * This annotation cannot be used on a {@link java.io.File} object. If you want to refer to the file path,
 * independently of its contents, return a {@link java.lang.String String} instead which returns the absolute
 * path.
 * If, instead, you want to refer to the contents and path of a file or a directory, use
 * {@link org.gradle.api.tasks.InputFile} or {@link org.gradle.api.tasks.InputDirectory} respectively.
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.FIELD})
public @interface Input {
}

/**
 * <p>Marks a property as specifying an output file for a task.</p>
 *
 * <p>This annotation should be attached to the getter method in Java or the property in Groovy.
 * Annotations on setters or just the field in Java are ignored.</p>
 *
 * <p>This will cause the task to be considered out-of-date when the file path or contents
 * are different to when the task was last run.</p>
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.FIELD})
public @interface OutputFile {
}

/**
 * Marks a method as the action to run when the task is executed.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public @interface TaskAction {
}

/**
 * <p>Attached to a task property to indicate that the property is not to be taken into account for up-to-date checking.</p>
 *
 * <p>This annotation should be attached to the getter method in Java or the property in Groovy.
 * Annotations on setters or just the field in Java are ignored.</p>
 *
 * <p>This will cause the task <em>not</em> to be considered out-of-date when the property has changed.</p>
 *
 * @since 3.0
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.FIELD})
public @interface Internal {
    /**
     * The reason for ignoring this element.
     */
    String value() default "";
}

/**
 * <p>Marks a property as specifying an output directory for a task.</p>
 *
 * <p>This annotation should be attached to the getter method in Java or the property in Groovy.
 * Annotations on setters or just the field in Java are ignored.</p>
 *
 * <p>This will cause the task to be considered out-of-date when the directory path or task
 * output to that directory has been modified since the task was last run.</p>
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.FIELD})
public @interface OutputDirectory {
}
```