---
Service 的启动原理
---

1. Service 启动有哪几种方式？
2. Service 启动过程中主要流程有哪些？
3. Service 启动过程涉及哪些参与者，通信过程是怎样的？

```java
@Override
public ComponentName startService(Intent service) {
	return startServiceCommon(service, mUser);
}
```

```java
ComponentName startServiceCommon(Intent service, ...) {
	ComponentName cn = ActivityManagerNative.getDefault().startService(mMainThread.getApplicationThread(), service, ...);
	return cn;
}
```

```java
// ASM#startService
ComponentName startService(IApplicationThread caller, Intent service, ...) {
    ComponentName res = mService.startServiceLocked(caller, service, ...);
    return res;
}
ComponentName startServiceLocked(Intent service, ...){
    ServiceLookupResult res = retrieveServiceLocked(service, ...);
    ServiceRecord r = res.record;
    //...
    r.pendingStarts.add(new ServiceRecord.StartItem(r, ...));
    return startServiceInnerLocked(smap, service, r, ...);
}
ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ...){
    bringUpServiceLocked(r, service.getFlags(), callerFg, false);
}
final String bringUpServiceLocked(ServiceRecord r, ...) {
    if(r.app != null && r.app.thread != null){
        // 如果 Service 已经启动，就执行 Service#onStartCommand
        sendServiceArgsLocked(r, ...);
        return null;
    }
    ProcessRecord app = mAm.getProcessRecordLocked(procName, ...);
    if(app != null && app.thread != null) {
        realStartServiceLocked(r, app, execlnFg);
        return null;
    }
    if(app == null){
        app = mAm.startProcessLocked(procName, ...);
    }
    if(!mPendingService.contains(r)){
        mPendingService.add(r);
    }
}

```

```java
//ASM#attachApplicationLocked
boolean attachApplicationLocked(IApplicationThread thread, ...) {
	// 处理 Pending 的 Service
    mService.attachApplicationLocked(app, processName);
    return true;
}
boolean attachApplicationLocked(ProcessRecord proc, ...) {
    for(int i=0;i<mPendingService.size();i++) {
        sr = mPendingService.get(i);
        //...
        mPendingServices.remove(i--);
        realStartServiceLocked(sr, proc, ...);
    }
}
void realStartServiceLocked(ServiceRecord r, ProcessRecord app, ...) {
    r.app = app;
    // scheduleCreateService: 向应用端发起 IPC 调用
    // 应用端收到之后就会去创建 Service，并执行 onCreate 回调
    // r: class ServiceRecord extends Binder{}
    app.thread.scheduleCreateService(r, r.serviceInfo, ...);
    // 触发 Service 的 onStartCommand 回调
    sendServiceArgsLocked(r, ...);
}

// 应用端处理 CreateService 请求，运行在主线程
private void handleCreateService(CreateServiceData data) {
    LoadedApk = packageInfo = getPackageInfoNoCheck(...);
    Service service = (Service)cl.loadClass(data.info.name).newInstance();
    
    ContextImpl context = ContextImpl.createAppContext(this, ...);
    
    Application app = packageInfo.makeApplication(false, ...);
    service.attach(context, this, ...);
    service.onCreate();
    // mService: ArrayMap<IBinder, Service>,key: ServiceRecord
    mService.put(data.token, service);
}

private final void sendServiceArgsLocked(ServiceRecord r, ){
    while(r.pendingStarts.size() > 0){
        StartItem si = r.pendingStarts.remove(0);
        // 调到应用端
        r.app.thread.scheduleServiceArgs(r, ...);
    }
}
public final void scheduleServiceArgs(IBinder token, ...) {
    ServiceArgsData s = new ServiceArgsData();
    //...
    sendMessage(H.SERVICE_ARGS, s);
}
private void handleServiceArgs(ServiceArgsData data) {
    Service s = mService.get(data, token);
    if(s != null){
        s.onStartCommand(data.args, data.flags, data.startId);
    }
}
```

**总结**

![](https://i.loli.net/2020/03/24/A6pSmv2zoZMGPnq.png)

![](https://i.loli.net/2020/03/24/SGLOXC9IHlzBcop.png)

绑定服务：

```java
int bindServiceLocked(IApplicationThread caller, ...) {
	//...
	if((flags & Context.BIND_AUTO_CREATE)!=0){
		bringUpServiceLocked(s, ...);
	}
}
```

BindService 并不会触发 onStartCommand，因为 bind service 它没有给 ServiceRecord 加到 pendingStarts 队列里面。