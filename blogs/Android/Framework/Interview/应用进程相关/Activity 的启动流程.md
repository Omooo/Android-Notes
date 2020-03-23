---
说说 Activity 的启动流程
---

1. 启动 Activity 会经历哪些生命周期回调
2. 冷启动大致流程，涉及哪些组件，通信过程是怎么样的？
3. Activity 启动过程中，生命周期回调的原理？

#### 启动流程

```java
startActivity ->
ActivityManagerNative.getDefault().startActivity ->
mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);

@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) {
	switch(code) {
		case START_ACTIVITY_TRANSACTION: {
			startActivity(app, callingPackage, intent, ...);
		}
	}
}
```

```java
// AMS
void startSpecialActivityLocked(ActivityRecord r, ...) {
    // 查询组件对应的进程是否已经启动
	ProcessRecord app = mService.getProcessRecordLocked(r.processName, ...);
	if(app!=null&&app.thread!=null){
		realStartActivity(r, app, ...);
		return;
	}
    // 启动进程
	mService.startProcessLocked(r.processName, ...);
}
```

```java
private final void startProcessLocked(ProcessRecord app) {
    // 进程启动完成后，执行的入口函数的 Java 类
	if(entryPoint == null) entryPoint = "android.app.ActivityThread";
	Process.ProcessStartResult startResult = Process.start(entryPoint, );
	
	synchronized(mPidsSelfLocked) {
		mPidsSelfLocked.put(startResult.pid, app);
		Message msg = mHandle.obtainMessage(PROC_START_TIMEOUT_MSG);
		msg.obj = app;
		mHandle.sendMessageDelayed(msg, PROC_START_TIMEOUT);
	}
}
```

```java
public static final ProcessStartResult start(final String processClass, ){
	return startViaZygote(processClass,);
}
```

```java
ProcessStartResult startViaZygote(final String processClass, ){
	return zygoteSendArgsAndGetResult(
			openZygoteSocketIfNeeded(abi), argsForZygote);
}
```

```java
boolean runOnce() {
    // 读取 Zygote 发送过来的参数列表
	String[] args = readArgumentList();
    // 创建应用进程
	int pid = Zygote.forkAndSpecialize(...);
	if(pid == 0) {
        // 子进程，执行 ActivityThread#main 函数
		handleChildProc(...);
		return true;
	} else {
        // 父进程把 pid 通过 Socket 在写回去
		return handlePArentProc(pid, ...);
	}
}
```

```java
// ActivityThread
public static void main(String[] args) {
    Looper.prepareMainLooper();
    
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    
    Looper.loop();

    throw new RuntimeException("");
}
```

```java
private void attach(boolean system) {
	IActivityManager mgr = ActivityManagerNative.getDefault();
	mgr.attachApplication(mAppThread);
}
```

```java
boolean attachApplicationLocked(IApplicationThread thread, ...) {
	ProcessRecord app = mPidsSelfLocked.get(pid);
    // IPC 调用，回到应用进程
	thread.bindApplication(...);
    // 以下处理挂起的应用组件
    mStackSupervisor.attachApplicationLocked(app);
    mServices.attachApplicationLocked(app, processName);
    sendPendingBroadcastsLocked(app);
}
```

```java
boolean attachApplicationLocked(ProcessRecord app) {
	// stack: mFocusedStack，
	ActivityRecord hr = stack.topRunningActivityLocked(null);
	realStartActivityLocked(hr, app, true, true);
}
```

```java
final boolean realStartActivityLocked(ActivityRecord r, ) {
    // ProcessRecord.ApplicationThread.scheduleLaunchActivity
	app.thread.scheduleLaunchActivity(new Intent(r.intent), ...);
}
```

```java
@Override 
public final void scheduleLaunchActivity(Intent intent, IBinder token, ){
	ActivityClientRecord r = new ActivityClientRecord();
    // mH,sendMessage，丢到主线程的 Handler
	sendMessage(H.LAUNCH_ACTIVITY, r);
}
```

```java
// 主线程处理
final ActivityClientRecord r = (ActivityClientRecord)msg.obj;
// 生成 LoadedApk 对象
r.packageInfo = getPackageInfoNoCheck(...);
handleLaunchActivity(r, null);
```

```java
private void handleLaunchActivity(ActivityClientRecord r, ...) {
	Activity a = performLaunchActivity(r, customIntent);
	if(a!=null) {
		handleResumeActivity(r.token, false, ...);
	}
}
```

```java
private Activity performLaunchActivity(ActivityClientRecord r, ...) {
	Activity activity = mInstrumentation.newActivity(...);
	Application app = r.packageInfo.makeApplication(false, mInstrumentation);
	Context appContext = createBaseContextForActivity(r, activity);
    // activity.attach
	activity.attach(appContext, ...);
    // activity.onCreate
	mInstrumentation.callActivityOnCreate(activity, r.state);
    // activity.onStart
	activity.performStart();
	return activity;
}
```

```java
final void handleResumeActivity(IBinder token) {
	ActivityClientRecord r = performResumeActivity(...);
	//...
}
```

**总结**

1. 创建 Activity 对象

   首先通过 ClassLoder 加载 LoadedApk 里的类生成 Activity 对象

2. 准备好 Application

   Application 在应用进程启动时就已经创建好了，这一步直接拿即可

3. 创建 ContextImpl

4. attach 上下文

5. 生命周期回调

![Activity 启动流程.png](https://i.loli.net/2020/03/23/tcsnLmOjI7DagKl.png)

#### 说说 Activity 的启动流程

1. 启动 Activity 要向 AMS 发起 binder 调用
2. Activity 所在进程是怎么启动的？
3. 应用 Activity 的生命周期回调原理