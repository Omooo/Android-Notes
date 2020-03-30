---
主线程为什么没有 ANR？
---

1. 了解 ANR 的触发的原理
2. 了解应用大致启动流程
3. 了解线程的消息循环机制
4. 了解应用和系统服务通信过程

#### ANR 是什么？

```java
final void appNotResponding(ProcessRecord app, ...){
    Message msg = Message.obtain();
    msg.what = SHOW_NOT_RESPONDING_MSG;
    mUiHandler.sendMessage(msg);
}
// SystemService 进程
Dialog d = new AppNotRespondingDialog(...);
d.show();
```

ANR 场景有哪些？

1. Service Timeout
2. BroadcastQueue Timeout
3. ContentProvider Timeout
4. InputDispatching Timeout

以 Service 为例，看一下 ANR 是怎么触发的？

```java
void realStartServiceLocked(ServiceRecord r, ProcessRecord app, boolean execInfg){
	bumpServiceExecutingLocked(r, execInFg, "create");
    app.thread.scheduleCreateService(r, r.serviceInfo, ...);
}
void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why){
    scheduleServiceTimeoutLocked(r.app);
}
void scheduleServiceTimeoutLocked(ProcessRecord proc){
    long now = SystemClock.uptimeMillis();
    Message msg = mAm.mHandler.obtainMessage(SERVICE_TIMEOUT_MSG);
    mAm.mHandler.sendMessageAtTime(msg, now+SERVICE_TIMEOUT);
}
void serviceTimeout(ProcessRecord proc){
    mAm.appNotResponding(proc, ...);
}
// 应用端处理 Service 请求
private void handleCreateService(CreateServiceData data) {
    service = (Service)cl.loadClass(data.info.name).newInstance();
    ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
    Application app = packageInfo.makeApplication(false, mInstrumentation);
    service.attach(context, this, data.info.name, data.token, ...);
    service.onCreate();
    ActivityManagerNative.getDefault().serviceDoneExecuting(...);
}
private void serviceDoneExecutingLocked(ServiceRecord r, ...){
    mAm.mHandler.removeMessage(SERVICE_TIMEOUT_MSG, r.app);
}
```

往主线程发消息：

```java
@Override
void scheduleLaunchActivity(Intent intent, IBinder token, ...){
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
void scheduleCreateService(IBinder token, ...){
    sendMessage(H.CREATE_SERVICE, s);
}
void scheduleReceiver(Intent intent, ActivityInfo info, ...){
    sendMessage(H.RECEIVER, r);
}
```

#### 总结

1. ANR 是应用没有在规定的时间内完成 AMS 指定的任务导致的
2. AMS 请求调用应用端 binder 线程，再丢消息去唤醒主线程来处理
3. ANR 不是因为主线程 loop 循环，而是因为主线程有耗时任务