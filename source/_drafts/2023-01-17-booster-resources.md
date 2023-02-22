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

- 第一步，找到数据源：resource merger 处理后且 aapt2 编译后的资源文件集合

```kotlin
// gradle api
Project.objects.fileCollection().from(     // 创建一个空的文件集合，用以承载下面的文件
    Component.artifacts.get(               // Access to the variant's buildable artifacts for build customization.
        InternalArtifactType.MERGED_RES))  // 它是一个目录，包含了所有 resource merger 处理后的模块资源文件，且经过 aapt2 编译

// booster 为了兼容各个版本的 gradle：v3.6、v4.2、v7.3 等等，抽象出一个统一的接口：AGPInterface
// 针对不同的 gradle 版本，实现可能不一样，这里是针对 gradle 7.3 的实现
internal object V73 : AGPInterface {

    private val BaseVariant.component: ComponentImpl
        get() = BaseVariantImpl::class.java.getDeclaredField("component").apply {
            isAccessible = true
        }.get(this) as ComponentImpl

    override val BaseVariant.project: Project
        get() = component.variantDependencies.run {
            javaClass.getDeclaredField("project").apply {
                isAccessible = true
            }.get(this) as Project
        }
}                
```

- 第二步，过滤出 *.png.flat 图片（只考虑 aapt2 的情况）

`*.png.flat` 是经过 aapt2 compile 后的 png 格式图片资源

resources merger 将所有资源放至同一目录，并以资源限定符为前缀如 `app\build\intermediates\res\merged\release\drawable-ldpi-v4_exo_icon_next.png.flat`，`raw_` 开头的是 `res/raw` 目录下的资源文件，它们有资源 ID `R.raw.[id]` 但是不压缩，直接将原始数据包进 apk（Arbitrary files to save in their raw form），所以这里也要遵循规范排除这些 raw 资源

排除 nine patch 图片资源

```kotlin
fun isFlatPngExceptRaw(file: File) : Boolean = isFlatPng(file) && !file.name.startsWith("raw_")

fun isFlatPng(file: File): Boolean = file.name.endsWith(".png.flat", true)
        && (file.name.length < 11 || !file.name.regionMatches(file.name.length - 11, ".9", 0, 2, true))
```

- 第三步，cwebp 转换图片格式

[cwebp](https://developers.google.com/speed/webp/docs/cwebp) 可以将 PNG, JPEG, TIFF, WebP or raw Y'CbCr 转换为 webp 格式（flat ?）

```shell
cwebp -mt -quiet -q [quality] [*.png.flat] -o [*.webp]
```

- 第四步，将格式转换后的 webp 图片交给 aapt2 编译成 flat 格式：`aapt2 compile`

整体流程如下：

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
            // 指定输出文件：文件名不变，后缀改为 webp，放到输出目录
            val output = compressedRes.file("${it.second.resourcePath.substringBeforeLast('.')}.webp")
            // 先用 cwebp 转格式，再用 aapt2 compile 编译为 flat 格式，因为这些 png 资源文件之前是 flat 格式的
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

# aapt2

[aapt2](https://developer.android.com/studio/command-line/aapt2) 全称 Android Asset Packaging Tool，是一个专门处理资源文件的命令行工具，主要有以下功能：

1. compile 

输入原始的资源文件，输出编译后的中间产物（intermediate file）：`aapt2 compile path-to-input-files [options] -o output-directory/`

    1. `res/values` 目录下的 xml 文件，比如字符串 strings.xml、颜色 colors.xml、数组 arrays.xml 等，被 aapt2 编译为 `*.arsc.flat` 中间产物，最后被链接为一个单独的 `resources.arsc` 打包进 apk，也就是说项目源码里会存在 `app/src/main/res/values` 目录但 apk 里是没有 `res/values` 目录的

    2. 其余的 xml 文件被编译为紧凑的、二进制的 `*.flat` 中间产物，最后被链接打包进 apk

    3. png 图片被压缩为 `*.png.flat` 中间产物（不是 png 格式的图片了），size 会因为压缩变小很多

    4. 其余资源文件直接链接打包进 apk

`*.flat` 文件是 aapt2 编译的产物，也叫做 aapt2 容器，由文件头（header）和资源项（entry）两大部分组成

| Category | Size in bytes | Field Name | Description |
|----------|---------------|------------|-------------|
| header | 4            | magic        | AAPT2 容器文件标识：AAPT 或者 0x54504141 |
|        | 4            | version      | AAPT2 容器版本 |
|        | 4            | entry_count  | 容器中包含的条目数量（一个 flat 文件可以包含多个资源项） |
| entry  | 4            | entry_type   | 资源类型，目前仅支持两种：RES_TABLE(0x00000000) 和 RES_FILE(0x00000001) |
|        | 8            | entry_length | 资源的长度 |
|        | entry_length | data         | 资源内容 |

RES_FILE 类型的结构如下：

| Size in bytes | Field | Description |
|---------------|-------|-------------|
| 4           | header_size    | header 的长度 |
| 8           | data_size      | data 的长度 |
| header_size | header         | protobuf 序列化的 CompiledFile 结构，保存了文件名、文件路径、文件配置和文件类型等信息 |
| x           | header_padding | 0 - 3 个填充字节，用以 data 32 位对齐 |
| data_size   | data           | 资源文件的内容：png、二进制的 XML 或者 protobuf 序列化的 XmlNode 结构 |
| x           | data_padding   | 0 - 3 个填充字节，用以 data 32 位对齐 |

```java
// RES_TABLE 是 protobuf 格式的 ResourceTable 结构

message ResourceTable {
    // 字符串常量池是为了把资源文件中的 string 复用起来，从而减少体积，资源文件中对应的字符串会被替换为字符串池中的索引
    StringPool source_pool = 1;
    repeated Package package = 2;                   // 用来生成资源 ID
    repeated Overlayable overlayable = 3;           // 资源叠加层相关
    repeated ToolFingerprint tool_fingerprint = 4;  // 工具版本
}

// 资源 ID 的命名方式遵循 0xPPTTEEEE 的规则
// 1. [PP] 对应 PackageId，一般应用使用的资源为 0x7f
// 2. [TT] 对应的是资源类型（文件夹的名称）
// 3. [EEEE] 为资源的 id，从 0 开始
message Package {
  PackageId package_id = 1;  // 包 ID
  string package_name = 2;   // 包名
  repeated Type type = 3;    // 资源类型：string, layout, xml 等，其对应的资源 ID 区间为 [0x01, 0xff]
}

// 资源 ID 的包 ID 部分，在 [0x00, 0xff] 范围内
// 1. [0x02, 0x7f) 由系统使用
// 2. 0x7f 应用使用
// 3. (0x7f, 0xff] 预留Id
message PackageId {
  uint32 id = 1;
}
```

2. link

将编译阶段产生的中间产物（intermediate files）如资源表文件（resources.arsc）、二进制的 xml 文件、压缩后的 png 图片等，打包进一个单独的 apk 文件，此 apk 是不包含字节码文件 dex 的，也没有经过签名

```shell
 aapt2 link -o output.apk
 -I android_sdk/platforms/android_version/android.jar
    compiled/res/values_values.arsc.flat
    compiled/res/drawable_Image.flat --manifest /path/to/AndroidManifest.xml -v
```

3. dump

打印 apk 文件里资源相关信息如：清单文件（badging，被编译为二进制了）、用到的资源限定符（configurations）、字符串池（strings）、整个资源表（resources）等等

```shell
aapt2 dump [sub-command] filename.apk [options]

sub-command: apc, badging, configurations, overlayable, packagename, permissions, strings, styleparents, resources, xmlstrings, xmltree
```

4. diff

找出两个 apk 文件间的差异

```shell
aapt2 diff first.apk second.apk
```

# 远程调试 gradle

这里分享一种较为简单的远程调试 gradle build script 的方法：

[debug 模式开启 gradle build](https://docs.gradle.org/6.5/userguide/troubleshooting.html#sec:troubleshooting_build_logic)，注意 Windows 下要这么写：`.\gradlew :app:assembleRelease '-Dorg.gradle.debug=true'`

git clone booster & checkout version，用 Android Studio 打开 booster project，运行以下远程调试任务

[remote_debug.png](/image/2023-01-17-booster-resources/remote_debug.png)

# 一些 gradle 名词和概念

[variant](https://developer.android.com/studio/build/build-variants) 指代构建的一个输出：apk or jar，它是 build type 和 product flavor 的组合，比如 demoDebug、productRelease 等等 

`artifacts` 指代构建过程用到的、产生的各种文件、目录，比如：

```kotlin

/**
 * Public [Artifact] for Android Gradle plugin.
 */
sealed class SingleArtifact<T : FileSystemLocation>(
    kind: ArtifactKind<T>,
    category: Category = Category.INTERMEDIATES,
    private val fileName: String? = null
)
    : Artifact.Single<T>(kind, category) {

    override fun getFileSystemLocationName(): String {
        return fileName ?: ""
    }

    /**
     * 存放 apk 的目录
     * Directory where APK files will be located. Some builds can be optimized for testing when
     * invoked from Android Studio. In such cases, the APKs are not suitable for deployment to
     * Play Store.
     */
    object APK:
        SingleArtifact<Directory>(DIRECTORY),
        Transformable,
        Replaceable,
        ContainsMany

    /**
     * 合并后的清单文件
     * Merged manifest file that will be used in the APK, Bundle and InstantApp packages.
     * This will only be available on modules applying one of the following plugins :
     *      com.android.application
     *      com.android.dynamic-feature
     *      com.android.library
     *      com.android.test
     *
     * For each module, unit test and android test variants will not have a manifest file
     * available.
     */
    object MERGED_MANIFEST:
        SingleArtifact<RegularFile>(FILE, Category.INTERMEDIATES, "AndroidManifest.xml"),
        Replaceable,
        Transformable

    object OBFUSCATION_MAPPING_FILE:
        SingleArtifact<RegularFile>(FILE, Category.OUTPUTS, "mapping.txt") {
            override fun getFolderName(): String = "mapping"
        }

    /**
     * 
     * The final Bundle ready for consumption at Play Store.
     * This is only valid for the base module.
     */
    object BUNDLE:
        SingleArtifact<RegularFile>(FILE, Category.OUTPUTS),
        Transformable

    /**
     * The final AAR file as it would be published.
     */
    object AAR:
        SingleArtifact<RegularFile>(FILE, Category.OUTPUTS),
        Transformable

    /**
     * 存放资源的目录
     * Assets that will be packaged in the resulting APK or Bundle.
     *
     * When used as an input, the content will be the merged assets.
     * For the APK, the assets will be compressed before packaging.
     *
     * To add new folders to [ASSETS], you must use [com.android.build.api.variant.Sources.assets]
     */
    @Incubating
    object ASSETS:
        SingleArtifact<Directory>(DIRECTORY),
        Transformable,
        Replaceable
}

/**
 *  A type of output generated by a task.
 */
sealed class
InternalArtifactType<T : FileSystemLocation>(
    kind: ArtifactKind<T>,
    category: Category = Category.INTERMEDIATES,
    private val folderName: String? = null,
    val finalizingArtifact: List<Artifact<*>> = listOf(),
) : Artifact.Single<T>(kind, category) {

    // --- classes ---
    // Javac task output.
    object JAVAC: InternalArtifactType<Directory>(DIRECTORY), Replaceable

    // --- Published classes ---
    // Class-type task output for tasks that generate published classes.

    // Packaged classes for library intermediate publishing
    // This is for external usage. For usage inside a module use ALL_CLASSES
    object RUNTIME_LIBRARY_CLASSES_JAR: InternalArtifactType<RegularFile>(FILE), Replaceable
    /**
     * A directory containing runtime classes. NOTE: It may contain either class files only
     * (preferred) or a single jar only, see {@link AndroidArtifacts.ClassesDirFormat}.
     */
    object RUNTIME_LIBRARY_CLASSES_DIR: InternalArtifactType<Directory>(DIRECTORY), Replaceable

    // Packaged library classes published only to compile configuration. This is to allow other
    // projects to compile again classes that were not additionally processed e.g. classes with
    // Jacoco instrumentation (b/109771903).
    object COMPILE_LIBRARY_CLASSES_JAR: InternalArtifactType<RegularFile>(FILE), Replaceable

    // Jar generated by the code shrinker
    object SHRUNK_CLASSES: InternalArtifactType<RegularFile>(FILE), Replaceable

    // Dex archive artifacts for project.
    object PROJECT_DEX_ARCHIVE: InternalArtifactType<Directory>(DIRECTORY), Replaceable
    // Dex archive artifacts for sub projects.
    object SUB_PROJECT_DEX_ARCHIVE: InternalArtifactType<Directory>(DIRECTORY), Replaceable
    // Dex archive artifacts for external (Maven) libraries.
    object EXTERNAL_LIBS_DEX_ARCHIVE: InternalArtifactType<Directory>(DIRECTORY), Replaceable
    // Dex archive artifacts for external (Maven) libraries processed using aritfact transforms.
    object EXTERNAL_LIBS_DEX_ARCHIVE_WITH_ARTIFACT_TRANSFORMS: InternalArtifactType<Directory>(DIRECTORY), Replaceable

    // --- android res ---
    // generated res
    object GENERATED_RES: InternalArtifactType<Directory>(DIRECTORY, Category.GENERATED,), Replaceable

    // output of the resource merger ready for aapt.
    object MERGED_RES: InternalArtifactType<Directory>(DIRECTORY), Replaceable

    // output of the resource merger for unit tests and the resource shrinker.
    object MERGED_NOT_COMPILED_RES: InternalArtifactType<Directory>(DIRECTORY), Replaceable

    // compiled resources (output of aapt)
    object PROCESSED_RES: InternalArtifactType<Directory>(DIRECTORY), Replaceable, ContainsMany

    // Processed res after an AAPT2 optimize operation
    object OPTIMIZED_PROCESSED_RES: InternalArtifactType<Directory>(DIRECTORY), Replaceable, ContainsMany

    // package resources for aar publishing.
    object PACKAGED_RES: InternalArtifactType<Directory>(DIRECTORY), Replaceable

    // ...
}
```

gradle 相关注解

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