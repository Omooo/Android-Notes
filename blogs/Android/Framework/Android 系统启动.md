---
Android 系统启动
---

#### 目录

1. 流程图
2. init 进程启动过程
3. Zygote 进程启动过程
4. SystemServer 处理过程
5. Launcher 启动过程
6. 启动流程总结

#### 流程图

![](https://i.loli.net/2019/04/23/5cbe5a6cbb114.png)

#### init 进程启动过程

init 进程是 Android 系统中用户空间的第一个进程，进程号为 1，是 Android 系统启动流程中一个关键的步骤，作为第一个进程，它被赋予了很多及其重要的工作职责，比如创建 Zygote 孵化器和属性服务等。

但在这之前，我们先要了解 Android 系统启动流程的前几步，以引入 init 进程：

1. 启动电源以及系统启动
2. 引导程序 BootLoader
3. Linux 内核启动
4. init 进程启动

当我们按下电源时，系统启动后会加载引导程序，引导程序又启动 Linux 内核，在 Linux 内核加载完成后，第一件事就是要启动 init 进程。

init 进程启动后会 fork 出 Zygote 进程，为了防止 init 进程的子进程成为僵尸进程，系统会在子进程暂停和终止的时候发出 SIGCHLD 信号。

僵尸进程与危害：

在 Linux 中，父进程使用 fork 创建子进程，在子进程终止之后，如果父进程并不知道子进程已经终止了，这时子进程虽然已经退出了，但是在系统进程表中还为它保留了一定的信息（比如进程号、退出状态、运行时间等），这个子进程就被称为僵尸进程。系统进程表是一项有限资源，如果系统进程表被僵尸进程耗尽的话，系统就可能无法创建新的进程了。

**总结：**

init 进程主要做了以下三件事：

1. 创建和挂载启动所需的文件目录
2. 初始化和启动属性服务
3. 解析 init.rc 配置文件并启动 Zygote 进程

#### Zygote 进程启动过程

在 Android 系统中，DVM 和 ART、应用程序进程以及运行系统的关键服务的 SystemServer 进程都是由 Zygote 进程来创建的，我们也将它称为孵化器。它通过 fork 的形式来创建应用程序进程和 SystemServer 进程。

**总结：**

Zygote 进程主要做了以下几件事：

1. 创建 AppRuntime 并调用其 start 方法，启动 Zygote 进程
2. 创建 Java 虚拟机并为 Java 虚拟机注册 JNI 方法
3. 通过 JNI 调用 ZygoteInit 的 main 方法进入 Zygote 的 Java 框架层
4. 通过 registerZygoteSocket 方法创建服务端 Socket，并通过 runSelectLoop 方法等待 AMS 的请求来创建新的应用程序进程
5. 启动 SystemServer 进程

#### SystemServer 处理过程

SystemServer 进程主要用于创建系统服务，比如 AMS、WMS、PMS 等都是由它来创建的。

**总结：**

SystemServer 进程被创建后，主要做了以下工作：

1. 启动 Binder 线程池，这样就可以与其他进程进行通信
2. 启动 SystemServiceManager，其用于对系统的服务进程创建、启动和生命周期管理
3. 启动各种系统服务

#### Launcher 启动过程

系统启动的最后一步是启动一个应用程序用来显示系统中已经被安装的应用程序，这个应用程序就叫 Launcher。Launcher 在启动过程中会请求 PackageManagerService 返回系统中已经安装的应用程序信息，并将这些信息封装成一个快捷图标列表展示在系统屏幕上，这样用户就可以通过点击这些快捷图标来启动相应的应用程序。

通俗的讲，Launcher 就是 Android 系统的桌面。

#### 启动流程总结

结合以上过程，就可以总结出 Android 系统启动的过程，大致有以下几部分：

1. 启动电源以及系统启动

   当电源按下时引导芯片代码从预定义的地方开始执行。加载引导程序 BootLoader 到 RAM，然后执行。

2. 引导程序 BootLoader

   引导程序 BootLoader 是在 Android 操作系统开始运行前的一个小程序，它的主要作用是把系统 OS 拉起来运行。

3. Linux 内核启动

   当内核启动时，设置缓存、存储器、计划列表、加载驱动。当内核完成系统设置时，它首先在系统文件中寻找 init.rc 文件，并启动 init 进程。

4. init 进程启动

   初始化和启动属性服务，并且启动 Zygote 进程。

5. Zygote 进程启动

   创建 Java 虚拟机并为 Java 虚拟机注册 JNI 方法，创建服务端 Socket，启动 SystemServer 进程。

6. SystemServer 进程启动

   启动 Binder 线程池和 SystemServiceManager，并且启动各种系统服务。

7. Launcher 启动

   被 SystemServer 进程启动的 AMS 会启动 Launcher，Launcher 启动后会将已安装的应用的快捷图标显示在界面上。