---
谈谈你对 Zygote 的理解？
---

#### 了解 Zygote 的作用

1. 启动 SystemServer
2. 孵化应用进程

#### 熟悉 Zygote 的启动流程

启动三段式：

进程启动 -> 准备工作 -> LOOP

Zygote 的启动可以分为两块：

1. 进程是怎么启动的？
2. 进程启动之后做了什么？

第一个问题：进程是怎么启动的？

Init 进程是 Linux 启动之后，用户空间的第一个进程。加载了一个 init.rc 配置文件，里面配置了很多系统服务。通过 fork+execev 系统调用。

```ini
// Service Name: zygote, 可执行路径 + 参数
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
class main
socket zygote stream 660 root system
onrestart write /sys/android_power/request_state wake
onrestart write /sys/power/state on
onrestart restart media
onrestart restart netd
writepid /dev/cpuset/foreground/tasks
```

启动进程有两种方式：

第一种是 fork + handle：

```c
pid_t pid = fork();
if(pid == 0){
		// child process
} else {
		// parent process
}
```

第二种是 fork + execve：

```c
pid_t pid = fork();
if(pid == 0){
		// child process
		execve(path, argv, env);
} else {
		// parent process
}
```

调用 fork 函数创建子进程，它会返回两次，子进程返回的 pid == 0，父进程返回的 pid 等于子进程的 pid。

默认情况下，子进程继承了父进程创建的所有资源，如果调用了系统调用 execve 去加载另一个二进程程序的话，那么继承父进程的资源就会被清掉。

信号处理，SIGCHLD 信号：

父进程 fork 出了子进程，如果子进程挂了，那么它的父进程就会收到 SIGCHLD 信号，然后就可以重启。

第二个问题：Zygote 进程启动之后做了什么？

* Zygote 的 Native 世界
* Zygote 的 Java 世界

Native 世界其实就是为了进入 Java 世界做准备，有以下三件事：

1. 启动虚拟机
2. 注册 Android 的 JNI 函数
3. 进入 Java 世界

```c
int main(int argc, char *argv[]) {
		JavaVM *jvm;
  	JNIEnv *env;
  	// 1
  	JNI_CreateJavaVM(&jvm, (void**)&env, &vm_args);
  	// 2
  	jclass clazz = env->FindClass("ZygoteInit");
  	jmethodID method = env->GetStaticMethodID(clazz, "Main", "(Ljava/lang/String;)V");
  	env->CallStaticVoidMethod(clazz, method, args);
  	jvm->DestoryJavaVM();
}
```

Java 虚拟机在 Zygote 都已经创建好了，应用程序都是由 Zygote 创建而来的，就直接继承了它的虚拟机。

再来看一下 Zygote 的世界，做了以下事情：

1. 通过 registerServerSockert 方法来创建一个 Server 端的 Socket，这个 name 为 zygote 的 socket 用于等待 AMS 请求 Zygote 来创建新的应用程序进程
2. 预加载资源，Preload Resources，在 Zygote 进程预加载系统资源后，然后通过它孵化出其他的虚拟机进程，进而共享虚拟机内存和框架层资源，这样大幅度提高应用程序的启动和运行速度
3. 启动 System Server
4. 进入 Loop 循环，执行 runSelectLoop 方法等待消息去创建应用进程

Zygote 处理请求的关键代码：

```java
boolean runOnce() {
  	// 1. 读取参数列表
		String[] args = readArgumentList();
  	// 2. 启动子进程
		int pid = Zygote.forkAndSpecialize();
		if(pid == 0){
      // 3. 在进程里面干活，其实就是执行了一个 java 类的 main 函数，java 类名就是上面读取的参数列表，参数列表是 AMS 跨进程发过来的，类名其实就是 ActivityThread
			// in child
			handleChildProc(args, ...);
			return true;
		}
}
```

注意细节：

1. Zygote 在 fork 的时候要保证是单线程的，为了避免造成死锁和状态不一致等问题
2. Zygote 的 IPC 没有采用 Binder，而是本地 Socket

问题：

1. 孵化应用进程这种事为什么不交给 SystemServer 来做，而专门设计一个 Zygote ？

> 我们知道，应用在启动的时候需要做很多准备工作，包括启动虚拟机，加载各类资源系统等等，这些都是非常耗时的，如果能在 zygote 里就给这些必要的初始化工作做好，子进程在 fork 的时候就能直接共享，那么这样的话效率就会非常高。这个就是 zygote 存在的价值，这一点 SystemServer 是替代不了的，主要是因为 SystemServer 里跑了一堆系统服务，这些是不能继承到应用程序的。而且我们应用程序在启动的时候，内存空间除了必要的资源外，最好是干干净净的，不要继承一些乱七八糟的东西，所以呢，不如给 SystemServer 和应用进程里都要用到的资源抽出来单独放到一个进程里，也就是这个 zygote 进程，然后 zygote 进程在分别孵化出 SystemServer 进程和应用进程。孵化出来之后，SystemServer 进程和应用程序进程就可以各干各事了。

1. Zygote 的 IPC 通信机制为什么不采用 Binder？如果采用 Binder 的话会有什么问题呢？

> https://www.zhihu.com/question/312480380
>
> https://blog.csdn.net/qq_39037047/article/details/88066589

#### 深刻理解 Zygote 的工作原理

