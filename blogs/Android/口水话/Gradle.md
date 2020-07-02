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

后面呢，我还在 Plugin 里面添加了自动 TinyPng 资源压缩，考虑到我们的 minApi 19，又做了全量的 png 转 webp 这一步的压缩是包含第三方库里面的图片的，使用的是 Google 开源的 cwebp 工具。再来做包体积优化时，发现存在不少的重复资源，也就是遍历生成 MD5 值进行比较，减少了约 150 kb。

再之后，基于做了 MethodTracker，简单使用一个注解就可以查看方法耗时，还有基于 Choreographer 的 FPS 检测。这里当时是遇到一个问题的，我们知道插件里面的类，在外部模块是不能访问到的，然而我这里的注解以及 FPSDetector 都是需要在 app 模块使用的，这时候我就看了 JW 的 Hugo 的实现，它也是基于 AspectJ 做的方法耗时，因为它的注解也是可以在 app 模块使用的，看源码发现它不过是是把这个注解放到外部一个远程库的，然后在插件里面进行依赖的。Hugo 仅仅只有四个类，却有 7k 多的 Star，所以说有时候不能被表面吓着，以为会很麻烦其实很简单。

在我写 Plugin 的之后，滴滴开源了 Booster，然后我就按照它的架构方式，使用 SPI 去自动注册 Task，这就不需要每新增一个 Task 都要去 apply 方法进行注册了。当时是用 Java 写的，也都切成了 Kotlin，并且也用的 kts 构建。

#### Gradle 构建优化

#### App 构建流程

![](https://i.loli.net/2019/05/07/5cd14ad86262c.png)