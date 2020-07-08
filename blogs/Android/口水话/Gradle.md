---
Gradle 口水话
---

#### 目录

1. 简述
2. Gradle 构建流程
3. Gradle Plugin
4. Gradle 构建优化
5. App 构建流程

#### 简述

#### Gradle 构建流程

Gradle 的执行分为三大阶段：Initialization -> Configuration -> Execution。Initialization 阶段主要目的是初始化构建，它分为两个子过程，一个执行 Init Script，另一个是执行 Setting Script。Init Script 会读取全局脚本初始化一些通用的属性，比如 Gradle User Home 目录、Gradle version 等，一般情况下我们不会动这个文件。Setting Script 更常见，它初始化了一次构建所参与的所有模块。

Configuration 阶段会加载项目中所有模块的 build.gradle 文件，根据脚本创建对应的 Task，然后根据 Task 的依赖关系生成有向无环图。在每执行完一个 Project 的 build.gradle 文件后，都会回调每个 build.gradle 文件中配置的 project.afterEvaluate{} 函数。生成任务图的过程也可以通过 gradle.taskGraph.whenReady 来进行 hook。

Execution 阶段才真正进行任务的执行。Gradle 会按照 task graph 中的依赖关系执行每一个任务，对于任务的执行前后也可以添加 hook 函数。项目中我也通过对 assembleRelease 进行 hook，在它执行完把生成的 release Apk 复制到指定目录。

可以在项目中，使用 build --scan 来进行性能分析，查看这三个阶段做了哪些事情以及耗费了多少时间。

#### Gradle Plugin

起初呢，是因为有一个需求就是，我们的 App 因为权限问题被警告了，组长就希望我能找出 app 模块及其依赖的第三方库里面的权限信息。首先我的思路就是先找出项目中的 Manifest 文件，在正则匹配出权限就行了。对于本地工程，这个很好做，可以直接通过 Variant 直接拿到 Manifest 文件，写个脚本在根目录的 subprojects 跑一遍就可以了。但是对于第三方库就没办法了。这时候我就想到，在 Apk 打包的时候有一个 MergeManifest 的过程，这个时候 Gradle 是肯定知道所有 Manifest 的，然后就去看一下 Gradle 源码是怎么做的。其实呢是在 ProcessApplicationManifest 这个 Task 来做的，Manifest 是放在 VariantData 的 ArtifactCollection 里面的。然后我就把它封装成一个 Task 命令行调用就可以拿到所有权限信息了。

考虑到后面可能还会有一些类似的需求，于是我就写了一个插件。我觉得这是一个非常好的开端，之所以当初组长找到我做这件事是因为我之前在组内分享了 Gradle 的相关知识，包括 Gradle 的构建流程、Gradle 的核心概念 Task 以及利用 Transform API 结合 AspectJ、ASM 进行自动化埋点等。这对我来说呢是一个正向激励，后面我也越来越会做更多的分享。

后面呢，我还在 Plugin 里面添加了自动 TinyPng 资源压缩，考虑到我们的 minApi 19，又做了全量的 png 转 webp 这一步的压缩是包含第三方库里面的图片的，使用的是 Google 开源的 cwebp 工具。通过 png 转 webp，减少了 5.7M，再来做包体积优化时，发现存在不少的重复资源，也就是遍历生成 MD5 值进行比较，减少了 133 kb；通过配置 resConfigs 只保留 zh、en 减少了 1.1M。

再之后，基于做了 MethodTracker，简单使用一个注解就可以查看方法耗时，还有基于 Choreographer 的 FPS 检测。这里当时是遇到一个问题的，我们知道插件里面的类，在外部模块是不能访问到的，然而我这里的注解以及 FPSDetector 都是需要在 app 模块使用的，这时候我就看了 JW 的 Hugo 的实现，它也是基于 AspectJ 做的方法耗时，因为它的注解也是可以在 app 模块使用的，看源码发现它不过是是把这个注解放到外部一个远程库的，然后在插件里面进行依赖的。Hugo 仅仅只有四个类，却有 7k 多的 Star，所以说有时候不能被表面吓着，以为会很麻烦其实很简单。

在我写 Plugin 的之后，滴滴开源了 Booster，然后我就按照它的架构方式，使用 SPI 去自动注册 Task，这就不需要每新增一个 Task 都要去 apply 方法进行注册了。当时是用 Java 写的，也都切成了 Kotlin，并且也用的 kts 构建。

#### Gradle 构建优化

首先在测试优化效果时，不包含 Configuration 阶段的时间，所以实际上优化程度更高。但是一般来说并不会频繁更新依赖，所以去掉了 Configuration 阶段。结果就是：

|        | 全量编译 | 代码增量 | 资源增量 |
| ------ | -------- | -------- | -------- |
| 优化前 | 1m 59s   | 27s      | 8s       |
| 优化后 | 1m 42s   | 18s      | 6s       |

首先就是使用较高版本的 Gradle 和 Android Gradle Plugin，在项目中，我主导了 Gradle 从 4.10.1 到 5.4.1 的升级。前面说过，Gradle 的构建分为三个阶段，分别是 Initialzation、Configuration 和 Execution 阶段。下面我就从这三个阶段来着手优化。

在 Initialzation 阶段，可以在 gradle.properties 中开启构建缓存和并行构建，也可以适当增加内存分配；

在 Configuration 阶段，避免使用动态版本和快照版本，动态版本即 1.0.+ 这种方式，快照版本即 SNAPSHOT，这两种方式都会迫使 Gradle 链接到远程仓库检查是否有依赖更新，默认有效期是 24h。我在项目中就发现一个友盟的 analyze 库使用了动态版本，然后就改为固定版本号；然后就是调整 repo 顺序并过滤请求，也就是把内部的 maven 仓库放在最前面，因为 Gradle 在查找远程库时，是串行查询所有 repo 中的 maven 地址的，直到找到可用的依赖，所以应该把最快和最高命中率的仓库放在前面。同时，在 Gradle 5.1 开始，可以指定特定 repo 下载特定的包名的依赖，然后我就在我们内部的 maven 库过滤 com.ehi 的依赖。这一操作能有效的减少 Configuration 阶段的时间；然后就是减少不必要的 Plugin，可以通过 build-scan 扫描项目中用到了哪些 Plugin，我在项目中发现我们 app 模块使用了一个 Google 的 osdecetor 的插件，但是搞了好久也没发现是哪里引入的，我怀疑是无良第三方库里面的，但是可惜一直没解决这个问题。

在 Execution 阶段，可以通过 -x 跳过不需要的 Task。我是在自己的 AS 里面配置 -x test -x lint 过滤掉 Test 和 Lint 相关的 Task，并且配置了一个 -PdevBuild 参数，这样测试小伙伴在 Jenkins 打 Debug 时也能享受这种收益，除此之外，我还会根据这个参数来关闭 AAPT2 自带的 png 压缩和 png 的合法性检查、关闭多 abi 和多 density 的构建以及通过 resConfigs 来最小化使用资源文件。resConfigs 不仅可以减少包体积，在 debug 时可以选择只构建中文简体和 xxhdpi 的资源。

#### App 构建流程

![](https://i.loli.net/2019/05/07/5cd14ad86262c.png)