---
Gradle 构建速度优化
---

#### 前言

Gradle 构建分为三个过程，Initialization 、Configuration 和 Execution 阶段。通常来说，Initialization 都很少，下面主要是针对 Configuration 和 Execution 这两个阶段的优化。

#### Configuration

##### 避免使用动态（Dynamic）或者快照（SNAPSHOT）版本

当我们使用某个版本的依赖时，推荐写死一个固定的版本，例如 1.0.0。这种方式下 Gradle 会从相关的 repo 下载依赖并缓存，后续再有引用该依赖的地方都可以从缓存里面读取，避免缓慢的网络下载。

除了上述固定版本以外，Gradle 还支持两种版本格式：动态版本和快照版本。

动态版本是类似于 1.0.+ 这种，用 + 替代具体版本的声明方式，快照版本是类似 1.0.0-SNAPSHOT 这种。这两种版本引用都会迫使 Gradle 链接远程仓库检查是否有更新的依赖可用，如果有则下载后缓存到本地，默认情况下，这种缓存有效期是 24 小时，当然也可以调整有效期。

##### 调整 repo 顺序并过滤 aar 请求

Gradle 在查找远程依赖的时候，会串行查询所有 repo 中的 maven 地址，直到找到可用的 aar 后下载，因此把最快和最高命中率的仓库放在前面，会有效减少 configuration 阶段所需的时间。

除了顺序以外，并不是所有的仓库都提供所有的依赖，尤其是有些公司会将业务 aar 放在内部搭建的仓库上，这种情况下如果盲目增加 repository 会让 Configuration 时间变得难以接受。通常我们需要将内部仓库放在前面，同时明确指定哪些依赖可以去这里下载。

```java
repositories {
    maven {
        url = uri("http://repo.mycompany.com/maven2")
        content {
            includeGroup("com.google")
        }
    }

    jcenter()
    ...
}
```

##### 减少不必要的 Plugin

应用到项目中的每个插件和脚本都会增加 Configuration 阶段的执行时间。

##### 使用 implementation 方法依赖

implementation 阻止依赖传递，避免每次改动一处就要全量编译。

#### Execution 优化

##### 使用 build cache

类似增量编译，Gradle Cache 可以把之前构建过得 task 结果缓存起来，一旦后面需要执行该 task 的时候直接使用缓存结果，与增量编译不同的是，cache 是全局的，对所有构建都生效。此外，cache 既可以保存在本地，又可以使用网络路径。

##### 配置缓存服务器

##### 跳过不需要执行的 Task

命令行中使用 -x 跳过不需要的 task，例如：

```terminal
// --console=verbose 可以输出所有执行的 task
./gradlew build --console=verbose -x lint -x test
```

#### Android 构建优化

除了上述的 Gradle 的通用优化之外，下面开始介绍针对 Android 的优化。

##### 使用最新的 Android Gradle Plugin

新版本的 AGP 每次更新都会修复大量 Bug 及提升性能等，因此保持最新的 AGP 版本有很大必要。

##### 关闭 APK split

对于日常开发调试而言，我们通常不需要构建多架构或者多尺寸的 apk，因此可以使用某个标记来区别调试构建和正式构建。

首先在 AS 增加一个标记 -PdevBuild，然后：

```kotlin
android {
    ....
    if (project.hasProperty("devBuild")) {
        splits.abi.isEnable = false;
        splits.density.isEnable = false;
    }
}
```

##### 关闭 aapt 自带的 png 压缩

在 Android 编译过程中，aapt 默认会对 png 图片进行压缩。可以去掉该功能来提高编译速度：

```kotlin
android {
    ....
    if (project.hasProperty("devBuild")) {
        ....
        aaptOptions.cruncherEnabled = false
    }
}
```

##### 避免 Android Manifest 改动

避免 AndroidManifest 里面的字段进行动态化设置。

##### 最小化使用资源文件

```groovy
android {
    defaultConfig {
        minSdkVersion 21
        resConfigs("zh", "en")
        if (project.hasProperty("devBuild")) {
            resConfigs("zh-rCN", "xxhdpi")
        }
    }
}
```

当你的应用包含大量本地化资源或为不同像素密度加入了特别的资源时，你可能需要应用这个小技巧来提高构件速度 --- 最小化开发阶段打包进应用的资源数量。

构建系统默认会将声明过或者使用过的资源全部打包进 APK，但在开发阶段我们可能只用到了其中一套而已，针对这种情况，我们需要使用 resConfig() 来指定构建开发版本时所需要用到的资源，如语言版本和屏幕像素密度。

##### 开启构建缓存

```xml
// gradle.properties
org.gradle.caching=true
```

或者在命令行里加入 --build-cache 参数。