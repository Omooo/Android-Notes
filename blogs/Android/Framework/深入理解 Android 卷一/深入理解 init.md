---
深入理解 init
---

#### 目录

1. 思维导图
2. 概述
3. 引入 init
4. init 进程的入口函数
5. 参考

#### 思维导图

#### 前言

init 是 Linux 系统中用户空间的第一个进程，由于 Android 是基于 Linux 内核的，所以 init 也是 Android 系统中用户空间的第一个进程，它的进程号是 1，作为一号进程，init 被赋予了很多及其重要的工作职责，总的来说主要做了以下三件事：

1. 创建和挂载启动所需的文件目录
2. 初始化和启动属性服务
3. 解析 init.rc 配置文件并启动 Zygote 进程

#### 引入 init

为了讲解 init 进程，首先要了解 Android 系统启动流程的前几步，以引入 init 进程：

1. 启动电源以及系统启动

   当电源按下时引导芯片代码从预定义的地方（固化在 ROM）开始执行。加载引导程序 BootLoader 到 RAM 中，然后执行。

2. 引导程序 BootLoader

   引导程序 BootLoader 是在 Android 系统开始运行前的一个小程序，它的主要作用是把系统 OS 拉起来并运行。

3. Linux 内核启动

   当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。在内核完成系统设置后，它首先在系统文件中寻找 init.rc 文件，并启动 init 进程。

4. init 进程启动

   init 进程做的工作比较多，主要用来初始化和启动属性服务，也用来启动 Zygote 进程。

从上面的步骤可以看出，当我们按下启动电源时，系统启动后会加载引导程序，引导程序又启动 Linux 内核，在 Linux 内核加载完成后，第一件事就是启动 init 进程。

#### init 进程的入口函数

在 Linux 内核加载完成后，它首先在系统文件中寻找 init.rc 文件，并启动 init 进程，然后查看 init 进程的入口函数 main，代码如下：

```c
int SecondStageMain(int argc, char** argv) {
	
  	//初始化属性服务
    property_init();

  	//启动属性服务
    StartPropertyService(&epoll);
		//解析 init.rc 文件等
    LoadBootScripts();

    return 0;
}

static void LoadBootScripts() {
    Parser parser = CreateParser(action_manager, service_list);

    parser.ParseConfig("/init.rc");
  	//...
}
```

