---
Android 系统架构
---

#### 五层架构

##### 概述

Android 底层内核空间以 Linux Kernal 作为基石，上层用户空间由 Native 系统库、虚拟机运行环境、框架层组成，通过系统调用（Syscall）连通系统的内核空间与用户空间。对于用户空间主要采用 C++ 和 Java 代码编写，通过 JNI 打通用户空间的 Java 层 和 Native 层，从而连通整个系统。

直接拿官方的图：

![](https://i.loli.net/2019/04/22/5cbd71433f4bf.png)

##### Linux 内核层

Android 平台的基础是 Linux 内核。例如，ART 依靠 Linux 内核来执行底层功能。Linux 内核的安全机制为 Android 提供了相应的保障，也允许设备制造商为内核开发硬件驱动程序。

##### 硬件抽象层 HAL

硬件抽象层提供标准界面，向更高级别的 Java Framework 层显示设备硬件功能。HAL 包含多个库模块，其中每个模块都为特定类型的硬件组件实现一个界面，例如相机和蓝牙模块。当框架 API 要求访问设备硬件时，Android 系统将为该硬件组件加载库模块。

##### Native C/C++ 库 && Android Runtime

每个应用都在其自己的进程中运行，都有自己的虚拟机实例。ART 通过执行 DEX 文件可在设备上运行多个虚拟机，DEX 文件是一种专为 Android 设计的字节码格式，经过优化，使用内存很少。ART 主要功能包括：AOT 和 JIT 编译，优化的 GC，以及调试相关的支持。

Native C/C++ 库主要包括 init 孵化来的用户空间的守护进程、HAL 层以及开机动画等。启动 init 进程，是 Linux 系统的用户进程，init 进程是所有用户进程的父进程。

##### Java Framework 层

Zygote 进程：

是由 init 进程通过解析 init.rc 文件后 fork 生成的，Zygote 进程主要包括：

- 加载 ZygoteInit 类，注册 Zygote Socket 服务端套接字
- 加载虚拟机
- 提前加载类 preloadClasses
- 提前加载资源 preloadResources

System Server 进程：

是由 Zygote 进程 fork 而来，System Server 是 Zygote 孵化的第一个进程，System Server 负责启动和管理整个 Java Framework，包括 ActivityManager、WindowManager、PackageManager、PowerManager 等服务。

Media Server 进程：

是由 init 进程 fork 而来，负责启动和管理整个 C++ Framework，包括 AudioFlinger、Camera Service 等服务。

##### System Apps 层

Zygote 进程孵化出第一个 App 进程是 Launcher，这是用户看到的桌面 App；

Zygote 进程还会创建 Browser、Phone、Email 等 App 进程，每个 App 至少运行在一个进程上。

所有的 App 进程都是由 Zygote 进程 fork 生成的。

系统内置的应用程序以及非系统级的应用程序都属于应用层，负责与用户进行直接交互。

#### 参考

[Android 平台架构](https://developer.android.com/guide/platform)

[袁辉辉 --- Android 系统架构开篇](<http://gityuan.com/android/>)