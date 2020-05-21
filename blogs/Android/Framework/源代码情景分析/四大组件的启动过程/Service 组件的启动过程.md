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

// ActivityThread
private final void handleCreateService(CreateServiceData data) {
    LoadedApk packageInfo = getPackageInfoNoCheck(data.info.applicationInfo);
    Service service = null;
    ClassLoader cl = packageInfo.getClassLoader();
    service = (Service) cl.loadClass(data.info.name).newInstance();
    ContextImpl context = new ContextImpl();
    context.init(packageInfo, null, this);
    
    Application app = packageInfo.makeApplication(false, mInstrumentation);
    context.setOuterContext(context);
    service.attach(context, this, data.info.name, ...);
    service.onCreate();
    mServices.put(data.token, service);
}
```

首先调用 getPakcageInfoNoCheck 来获得一个用来描述即将要启动的 Service 组件所在的应用程序的 LoadedApk 对象，并且将它保存在变量 packageInfo 中。在进程中加载的每一个应用程序都使用一个 LoadedApk 对象来描述，通过它就可以访问到它所描述的应用程序的资源。

然后通过 LoadedApk 对象的  getClassLoader 来获得一个类加载器来加载 Service 组件，在调用其 onCreate 方法。

至此，Service 组件的启动过程就分析完了，它是在一个新的应用程序进程中启动的。

#### 绑定服务的启动过程

```java
// ContextImpl
private LoadedApk mPackageInfo;
private ActivityThread mMainThread;
public boolean bindService(Intent service, ServiceConnection conn, int flags) {
    IServiceConnection sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), mMainThread.getHandler(), flags);
    int res = ActivityManagerNative.getDefault().bindService(
    	mMainThread.getApplicationThread(), getActivityToken(),
    	service, ...);
    return res;
    
}
```

ContextImpl 类的成员变量 mPackageInfo 的类型为 LoadedApk，然后调用它的 getServiceDispatcher 将 ServiceConnection 对象封装成一个实现了 IServiceConnection 接口的 Binder 本地对象，然后再调用 AMS 代理对象的 bindService 函数。

再将 ServiceConnection 对象封装成一个实现了 IServiceConnection 接口的 Binder 本地对象的过程中，还使用到了另外两个参数：

1. 第一个参数是 mMainThread 的 mH，当 AMS 成功的将 Service 组件启动起来，并且获得它内部的一个 Binder 本地对象之后，AMS 便会将这个 Binder 本地对象传递给 Binder 本地对象 sd，接着 Binder 本地对象 sd 再将这个 Binder 本地对象封装成一个消息，发送到 Activity 组件所运行的应用程序的主线程消息队列中，最后在分发给 Activity 组件内部的成员变量 ServiceConnection 的 onServiceConnected 来处理。
2. 第二个参数是通过 getOuterContext 函数来获得的，它指向的是一个 Activity 组件，这就是 bindService 是的 Activity 组件，这样，前面所获得的 Binder 本地对象 sd 就知道它所封装的 ServiceConnection 对象 conn 是与该 Activity 组件关联在一起的。

接下来，我们继续分析 ContextImpl 类的成员变量 mPackageInfo 的成员变量 getServiceDispatcher 是如何将 ServiceConnection 对象 conn 封装成一个实现了 IServiceConnection 接口的 Binder 本地对象 sd 的：

```java
// LoakdedApk
public final IServiceConnection getServiceDispatcher(ServiceConnection c, Context context, Handler handler, int flags) {
    
}
```

