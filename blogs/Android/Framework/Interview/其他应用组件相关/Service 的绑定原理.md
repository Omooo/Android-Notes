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

