---
谈谈你对 Application 的理解
---

1. 了解 Application 的作用
2. 熟悉 Application 的类继承关系以及生命周期
3. 深入理解 Application 的初始化原理

#### Application 有什么用？

1. 应用进程初始化操作
2. 提供上下文

#### Application 怎么初始化？

```java
// ActivityThread#attach
private void attach() {
    final IActivityManager mgr = ActivityManagerNative.getDefault();
    mgr.attachAppliation(mAppThread);
}
```

```java
// AMS
public final void attachApplication(IApplicationThread thread) {
    synchronized(this) {
        attachApplicationLocked(thread, callingPid);
    }
}
```

```java
boolean attachApplicationLocked(IApplicationThread thread, ...) {
	//...
    // IPC 调用，回到应用进程
	thread.bindApplication(...);
}
```

```java
public final void bindApplication(...) {
	AppBindData data = new AppBindData();
	//...
	sendMessage(H.BIND_APPLICATION, data);
}
```

```java
private void handleBindApplication(AppBindData data) {
    // LoadedApk
	data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    // LoadedApk#makeApplication
	Application app = data.info.makeApplication(...);
    // Application#onCreate
	mInstrumentation.callApplicationOnCreate(app);
}
```

```java
public Application makeApplication(...) {
	if(mApplication != null) {
		return mApplication;
	}
	ContextImpl appContext = ContextImpl.createAppContext(...);
	app = mActivityThread.mInstrumentation.newApplication(...);
	return app;
}
```

```java
Application new Application(ClassLoader cl, String className, Context context) {
	return newApplication(cl.loadClass(className), context);
}
```

```java
static Application newApplication(Class<?> clazz, Context context) {
	Application app = (Application)clazz.newInstance();
	app.attach(context);
	return app;
}
```

```java
final void attach(Context context) {
	attachBaseContext(context);
}
```

总结一下 Application 的初始化：

1. new Application()
2. Application.attachBaseContext()
3. Application.onCreate()

**不要执行耗时操作**

```java
boolean attachApplicationLocked(IApplicationThread thread, ...) {
	//...
    // IPC 调用，回到应用进程
	thread.bindApplication(...);
    //...
    // 启动待启动的组件
    mStackSupervisor.attachApplicationLocked(...);
    mServices.attachApplicationLocked(app, processName);
    sendPendingBroadcastsLocked(app);
}
```

