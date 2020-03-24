---
Service 的绑定原理
---

1. 知道 bindService 的用法
2. 了解 bindService 的大致流程
3. bindService 涉及哪些参与者，通信过程是怎样的？

```java
IRemoteCaller mCaller;

ServiceConnection mServiceConnection = new ServiceConnection(){
	@Override
	public void onServiceConnected(ComponentName name, IBinder service) {
	mCaller = IRemoteCaller.Stub.asInterface(service);
	mCaller.call();
	}
}

Intent service = new Intent(this, MyService.class);
bindService(service, mServiceConnection, BIND_AUTO_CREATE);
```

Bind Service 总体流程：

![](https://i.loli.net/2020/03/24/F4HYsiel7phDGBw.png)

```java
@Override
boolean bindService(Intent service, ServiceConnection conn, int flags) {
	return bindServiceCommon(service, conn, flags, ...);
}
boolean bindServiceCommon(Intent service, ServiceConnection conn, ...) {
    IServiceConnection sd;
    // 为 ServiceConnection 接口生成一个 binder 对象，可用于跨进程传递给 AMS
    sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), mMainThread.getHandler(), flags);
    ActivityManagerNative.getDefult().bindService(mMainThread.getApplicationThread(), service, sd, ...);
}
IServiceConnection getServiceDispatcher(ServiceConnection c, ...) {
    ServiceDispatcher sd = null;
    Map<ServiceConnection, ServiceDispatcher> map = mServices.get(context);
    if(map != null){
        sd = map.get(c);
    }
    if(sd == null){
        sd = new ServiceDispatcher(c, context, handler,flags);
        if(map == null){
            map = new ArrayMap<ServiceConnection, ServiceDispatcher>();
            mService.put(context, map);
        }
        map.put(c, sd);
    }
    return sd.getIServiceConnection();
}
ServiceDispatcher(ServiceConnection conn, ...){
    mIServiceConnection = new InnerConnection(this);
    mConnection = conn;
}

static class InnerConnection extends IServiceConnection.Stub {
    final WeakReference<ServiceDispatcher> mDispatcher;
    public void connected(ComponentName name, IBinder service) {
        LoadedApk.ServiceDispatcher sd = mDispatcher.get();
        if(sd != null){
            sd.connected(name, service);
        }
    }
}
void connected(){
    mActivityThread.post(new RunConnection(name, service, 0));
}

public void doConnected(ComponentName name, IBinder service) {
    old = mActiveConnections.get(name);
    if(old!=null && old.binder == service){
        return;
    }
    if(service != null){
        info = new ConnectionInfo(service);
        mActiveConnections.put(name, info);
    }else{
        mActiveConnections.remove(name);
    }
    if(old != null){
        mConnection.onServiceDisconnected(name);
    }
    if(service != null){
        mConnection.onServiceConnected(name, service);
    }
}
```

```java
// AMS#bindServiceLocked
bindServiceLocked(IApplicationThread caller, ...){
	//...
	if((flag&Context.BIND_AUTO_CREATE)!=0){
		bringUpServiceLocked(s, );
	}
	if(s.app != null && b.intent.received){
        c.conn.connected(s.name, b.intent.binder);
    }else if(!b.intent.requested){
        requestServiceBindingLocked(s, b.intent, false);
    }
}
```

```java
final String bringUpServiceLocked(ServiceRecord r, ...) {
    if(r.app != null && r.app.thread != null){
        // 如果 Service 已经启动，就执行 Service#onStartCommand
        // 在 bind service 中没啥用
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
    return null;
}
```

```java
void realStartServiceLocked(ServiceRecord r, ProcessRecord app, ...) {
    r.app = app;
    // scheduleCreateService: 向应用端发起 IPC 调用
    // 应用端收到之后就会去创建 Service，并执行 onCreate 回调
    // r: class ServiceRecord extends Binder{}
    app.thread.scheduleCreateService(r, r.serviceInfo, ...);
    // 请求 Service 发布它的 binder 句柄给 AMS
    requestServiceBindingLocked(r, ...);
    // 触发 Service 的 onStartCommand 回调
    sendServiceArgsLocked(r, ...);
}
boolean requestServiceBindingLocked(ServiceRecord r, ...) {
    if(r.app == null || r.app.thread == null){
        return false;
    }
    if((!i.requested||rebind)&&i.apps.size()>0){
        r.app.thread.scheduleBindService(r, rebind, ...);
        if(!rebind){
            i.requested = true;
        }
        i.doRebind = false;
    }
    return true;
}
// scheduleBindService 的处理过程
private void handleBindService(BindServiceData data) {
    Service s = mService.get(data.token);
    if(!data.rebind){
        IBinder binder = s.onBind(data.intent);
        ActivityManagerNative.getDefault().publishService(data.token, data.intent, binder);
    }else {
        s.onRebind(data.intent);
    }
}
```

```java
// AMS#publishServiceLocked
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    Intent.FilterComparison filter = new Intent.FilterComparison(intent);
    IntentBindRecord b = r.bindings.get(filter);
    if(b!=null && !b.received){
        b.binder = service;
        b.requested = true;
        b.received = true;
        for(int conni=r.connections.size()-1;conni>=0;conni--) {
            ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
            for(int i=0;i<clist.size();i++){
                ConnectionRecord c = clist.get(i);
                if(!filter.equals(c.binding.intent.intent)){
                    continue;
                }
                c.conn.connected(r.name, service);
            }
        }
    }
}
```

onRebind 什么时候调用？

```java
int bindServiceLocked(IApplicationThread caller, IBinder token, ) {
    if(s.app!=null&&b.intent.received){
        if(b.intent.apps.size()==1&&b.intent.doRebind){
            requestServiceBindingLocked(s, b.intent, callerFg, true);
        }
    }
}
boolean unbindServiceLocked(IServiceConnection connection) {
    IBinder binder = connection.asBinder();
    ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
    while(clist.size()>0){
        ConnectionRecord r = clist.get(0);
        removeConnectionLocked(r, null, null);
        if(clist.size()>0&&clist.get(0)==r){
            clist.remove(0);
        }
    }
    return true;
}
void removeConnectionLocked(ConnectionRecord c, ...){
    if(s.app!=null&&s.app.thread!=null&&b.intent.apps.size()==0&&b.intent.hasBound){
        b.intent.hasBound = false;
        b.intent.doRebind = false;
        s.app.thread.scheduleUnbindService(s, b.intent.intent.getIntent());
    }
}
private void handleUnbindService(BindServiceData data) {
    Service s = mService.get(data.token);
    boolean doRebind = s.onUnbind(data.intent);
    if(doRebind){
        ActivityManagerNative.getDefault().unbindFinished(doRebind);
    }
}
// AMS#unbindFinishedLocked
void unbindFinishedLocked(ServiceRecord r, Intent intent, boolean doRebind) {
    Intent.FilterComparison filter = new Intent.FilterComparison(intent);
    IntentBindRecord b = r.bindings.get(filter);
    if(b.apps.size()>0&&!inDestorying) {
        
    }else {
        b.doRebind = true;
    }
}
```

