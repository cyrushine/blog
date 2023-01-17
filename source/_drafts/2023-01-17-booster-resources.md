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

[variant](https://developer.android.com/studio/build/build-variants) 指代构建的一个输出：apk or jar，它是 build type 和 product flavor 的组合，比如 demoDebug、productRelease 等等 

`VariantProcessor` 是 booster API，本模块入口 `CwebpCompressionVariantProcessor` 通过 [AutoService](#google-autoservice) 注册为它的一个实现，然后 booster-gradle-plugin 在运行时通过 [ServiceLoader](#spi) 发现并执行 cwebp 模块

```kotlin
// https://github.com/didi/booster/blob/master/booster-task-compression-cwebp/src/main/kotlin/com/didiglobal/booster/task/compression/cwebp/CwebpCompressionVariantProcessor.kt
@AutoService(VariantProcessor::class)
class CwebpCompressionVariantProcessor : VariantProcessor {

    override fun process(variant: BaseVariant) {  // 处理各种变体
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
    private val tool: CompressionTool,  /* CwebpCompressOpaqueFlatImages */
    private val compressor: (Boolean) -> KClass<out CompressImages<out CompressionOptions>>
) : CompressionTaskCreator {

    override fun createCompressionTask(
            variant: BaseVariant,
            results: CompressionResults,
            name: String,  /* resources */
            supplier: () -> Collection<File>,
            ignores: Set<Wildcard>,
            vararg deps: TaskProvider<out Task>
    ): TaskProvider<out CompressImages<out CompressionOptions>> {
        val project = variant.project
        val aapt2 = project.isAapt2Enabled
        val install = getCommandInstaller(variant)

        // 往 project 注册了一个名为 compressDemoReleaseResourcesWithCwebpCompressOpaqueFlatImages 的任务
        // 这个任务就是 CwebpCompressOpaqueFlatImages
        return project.tasks.register(
            "compress${variant.name.capitalize()}${name.capitalize()}With${tool.command.name.substringBefore('.').capitalize()}", 
            getCompressionTaskClass(aapt2).java
        ) { task ->  // 配置 CwebpCompressOpaqueFlatImages
            task.group = BOOSTER
            task.description = "Compress image resources by ${tool.command.name} for ${variant.name}"
            task.dependsOn(variant.preBuildTaskProvider.get())
            task.tool = tool
            task.variant = variant
            task.results = results
            task.filter = if (ignores.isNotEmpty()) excludes(ignores) else MATCH_ALL_RESOURCES
            task.images = lazy(supplier)::value
        }.apply {
            dependsOn(install)
            deps.forEach { dependsOn(it) }
            variant.processResTaskProvider?.dependsOn(this)
            variant.bundleResourcesTaskProvider?.dependsOn(this)
        }
    }

    override fun getCompressionTaskClass(aapt2: Boolean) = compressor(aapt2)
}
```