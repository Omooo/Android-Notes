---
Android 系统的启动流程
---

#### Android 有哪些主要的系统进程？

在 init.rc 启动配置文件中定义了很多 Service，这些 Service 就是要单独启动的系统服务进程。

```ini
service zygote /system/bin/app_process ...
service servicemanager /system/bin/servicemanager ...
service surfaceflinger /system/bin/surfaceflinger ...
service media /system/bin/mediaserver ...
...
```

#### 这些系统进程是怎么启动的？以及做了什么事？

Zygote 启动：

1. init 进程 fork 出 zygote 进程
2. 启动虚拟机，注册 jni 函数，为进入 Java 世界做准备
3. 预加载系统资源
4. 启动 SystemServer
5. 进入 Socket Loop

Zygote 工作流程就是执行 runOnce 函数的过程。

SystemServer 启动：

```java
private static boolean startSystemServer(...) {
		String args[] = {
      	...
        "com.android.server.SystemServer",
    }
  	int pid = Zygote.forkSystemServer(...);
  	if(pid == 0){
      	handlerSystemServerProcess(parsedArgs);
    }
  	return true;
}
```

```java
void handlerSystemServerProcess(Arguments parsedArgs) {
		RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, ...);
}
```

```java
void zygoteInit(String[] argv, ...) {
		commonInit();
		nativeZygoteInit();
		applicationInit(argv, ...);
}
```

```java
// 启动 Binder 机制
void nativeZygoteInit() {
		sp<ProcessState> proc = ProcessState::self();
  	proc->startThreadPool();
}
```

```java
// 执行 SystemServer 的 main 函数
void applicationInit() {
		invokeStaticMain(args, ...);
}
```

```java
public static void main() {
		new SystemServer().run();
}
```

```java
// SystemServer#run

```

```java
private void run() {
		Looper.prepareMainLooper();
		
		System.loadLibrary("android_servers");
		createSystemContext();
		
		startBootstrapServcies();
		startCoreServices();
		startOtherServices();
		
		Looper.loop();
}
```



#### 两个问题：

1. 系统服务是怎么启动的？
2. 怎么解决系统服务之间的相互依赖？

第一个问题，系统服务是怎么启动的？

我们只需要关注两个问题：

1. 系统服务怎么发布，让应用程序可见？

   ```java
   void publishBinderService(String name, IBiner service, ...) {
   		ServiceManager.addService(name, service, allowlsolated);
   }
   ```

   也就是把系统服务的 Binder 注册到 ServiceManager 中。

2. 系统服务跑在什么线程？

3. 

课堂作业：

1. 为什么系统服务不都跑在 binder 线程里呢？
2. 为什么系统服务不都跑在自己私有的工作线程里呢？
3. 跑在 binder 线程和跑在工作线程，如何取舍？

第二个问题，怎么解决系统服务之间的相互依赖？

1. 分批启动（AMS、PMS、PKMS）
2. 分阶段启动（阶段1、阶段2...）

#### 桌面启动

在 AMS 服务就绪的时候，会调用以下函数：

```java
public void systemReady(final Runnable goingCallback) {
		//...
  	startHomeActivityLocked(mCurrentUserId, "systemReady");
}
```

在 Launcher2 Activity 的 onCreate 里面会启动一个 LoaderTask：

```java
mLoaderTask = new LoaderTask(mApp.getContext(), loadFlags);
```

这个 LoaderTask 会向 PMS 去查询已经安装的应用：

```
mPm.queryIntentActivitiesAsUser
```

#### 说说 Android 系统的启动流程？

1. Zygote 是怎么启动的？
2. systemServer 是怎么启动的？
3. 系统服务是怎么启动的？