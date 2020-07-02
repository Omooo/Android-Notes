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

#### Gradle 构建优化

#### App 构建流程

![](https://i.loli.net/2019/05/07/5cd14ad86262c.png)