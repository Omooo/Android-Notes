---
Service 组件的启动过程
---

#### Service 组件在新进程中的启动过程

在我们调用 Context 的 startService 时，其实是调用到 ContextImpl 的 startService：

```java
class ContextImpl extends Context {
    
    @Override
    public ComponentName startService(Intent service) {
        ComponentName cn = ActivityManagerNative.getDefault().startService(
        mMainThread.getApplicationThread(), service,
        service.resolveTypeIfNeeded(getContextResolver()));
        return cn;
    }
}
```

获取 AMS 的代理对象去处理 startService 的请求：

```java
// ActivityManagerProxy
public ComponentName startService(IApplicationThread caller, Intent service, String resolvedType) {
    Parcel data = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller);
    mRemote.transact(START_SERVICE_TRANSACTION, data, ...);
}
```

接着再通过 ActivityManagerProxy 类内部的一个 Binder 代理对象 mRemote 向 AMS 发送一个类型为 START_SERVICE_TRANSACTION 的进程间通信请求。

接下来就是在 AMS 中去处理 Client 组件发送来的类型为 START_SERVICE_TRANSACTION 的进程间通信请求。

```java
// AMS
public ComponentName startService(IApplicationThread caller, Intent service, String resolvedType) {
    ComponentName res = startServiceLocked(caller, service, ...);
    return res;
}

ComponentName startServiceLocked(IApplicationThread caller, Intent service, ...) {
    ServiceLookupResult res = retrieveServiceLocked(service, ...);
    ServiceRecord r = res.record;
    bringUpServiceLocked(r, service.getFlags());
    return r.name;
}
```

在 AMS 中，每一个 Service 组件都使用一个 ServiceRecord 对象来描述，就像每一个 Activity 组件都使用一个 ActivityRecord 对象来描述一样。

首先调用 retrieveServiceLocked 在 AMS 中查找是否存在与参数 service 对应的一个 ServiceRecord 对象。如果不存在，AMS 就会到 PKMS 中去获取与参数 service 对应的一个 Service 组件的信息，然后再将这些信息封装成一个 ServiceRecord 对象，最后调用 bringUpServiceLocked 来启东 ServiceRecord 对象 r 所描述的一个 Service 组件。

```java
// AMS
private final boolean bringUpServiceLocked(ServiceRecord r, ...) {
    final String appName = r.processName;
    ProcessRecord app = getProcessRecordLocked(appName, r.appInfo.uid);
    if (app != null && app.thread != null) {
        realStartServiceLocked(r, app);
        return true;
    }
    if (startProcessLocked(appName, r.appInfom, "service", ...)) {
        return false;
    }
    if (!mPendingServices.contains(r)) {
        mPendingServices.add(r);
    }
    return true;
}
```

首先判断该 Service 组件所在的进程是否已经创建了，如果没有创建，就会去执行 startProcessLocked 去创建一个应用程序进程，并且把该 Service 添加到 mPendingService 列表中。

startProcessLocked 就是调用 Process 类的 start 来创建一个新的应用程序进程，创建完成之后，新的应用程序进程会把 ApplicationThread 传递给 AMS，以便 AMS 可以和这个新创建的应用程序进程通信。新创建的应用程序进程会向 AMS 发送一个 ATTACH_APPLICATION_TRANSACTION 的进程间通信请求。

```java
// AMS
private final boolean attachApplicationLocked(IApplicationThread thread, int pid) {
    ProcessRecord app = mPidsSlefLocked.get(pid);
    app.thread = thread;
    if (mPendingServices.size() > 0) {
        ServiceRecord sr = null;
        for (int i=0;i<mPengdingServices.size();i++) {
            sr = mPengdingServices.get(i);
            if (app.info.uid != sr.appInfo.uid
                || !processName.equals(sr.processName)) {
                continue;
            }
            mPendingServices.remove(i);
            i--;
            realStartServiceLocked(sr, app);
        }
    }
    
}
```

在 mPendingServices 中找到对应的 Service 组件，然后调用 realStartServiceLocked 将它启动起来。

```java
// AMS
private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app) {
    r.app = app;
    app.thread.scheduleCreateService(r, r.serviceInfo);
}
```

也就是通过 ApplicationThreadProxy 去执行：

```java
// ApplicationThreadProxy
public final void scheduleCreateService(IBinder token, ServiceInfo info) {
    Parcel data = Parcel.obtain();
    data.writeInterfaceToken(IApplicationThread.descriptor);
    data.writeStrongBinder(token);
    mRemote.transact(SCHEDULE_CREATE_SERVICE_TRANSACTION, data, null, IBinder.FLAG_ONEWAY);
}
```

也就是通过 ApplicationThreadProxy 类内部的一个 Binder 代理对象 mRemote 向新创建的应用程序进程发送一个类型为 SCHEDULE_CREATE_SERVICE_TRANSACTION 的进程间通信请求。

接下来就会到应用程序进程去处理这个请求：

```java
// ApplicationThread
public final void scheduleCreateService(IBinder token, ServiceInfo info) {
    CreateServiceData s = new CreateServiceData();
    s.token = token;
    s.info = info;
    queueOrSendMessage(H.CREATE_SERVICE, s);
}
```

ApplicationThread 类的成员函数 scheduleCreateService 用来处理类型为 SCHEDULE_CREATE_SERVICE_TRANSACTION 的进程间通信请求。

首先封装成一个 CreateServiceData 对象，然后再往主线程发送一个 CREATE_SERVICE 消息。

```java
// H
public void handleMessage(Message msg) {
    switch (msg.what) {
        case CREATE_SERVICE:
            handleCreateService((CreateServiceData)msg.obj);
            break;
    }
}
```

