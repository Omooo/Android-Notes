---
Provider 的启动原理
---

1. 了解 ContentProvider 的生命周期
2. 熟悉 ContentProvider 的启动流程
3. 熟悉 Provider 启动过程中各方通信原理

```java
ContextResolver resolver = context.getContentResolver();
resolver.insert(uri, contentValues);
public ContextResolver getContextResolver(){
    return mContextResolver;
}
private ContextImpl(ConntextImpl container, ...) {
    mContextResolver = new ApplicationContextResolver(this, ...);
} 
public final Uri insert(Uri url, ContentValues values) {
    IContentProvider provider = acquireProvider(url);
    Uri createdRow = provider.insert(mPackageName, url, values);
    return createdRow;
}
protected IContentProvider acquireProvider(Context context, String auth) {
    return mMainThread.acquireProvider(context, ...);
}
IContentProvider acquireProvider(Context c, String auth, ...){
    IContentProvider provider = acquireExistingProvider(c, ...);
    if(provider != null){
        return provider;
    }
    IActivityManager.ContentProviderHolder holder = null;
    holder = ActivityManagerNative.getDefault().getContentProvider(getApplicationThread(), auth, userId, stable);
    holder = installProvider(c, holder, holder.info, ...);
    return holder.provider;
}
```

获取 ContentProvider 的过程小结：

本地找 IContentProvider 对象 -> 找到直接返回，不然请求 AMS  -> AMS 返回 holder，本地安装。

本地找：

```java
IContentProvider acquireExistingProvider (Context c, ...) {
	final ProviderKey key = new ProviderKey(auth, userId);
    final ProviderClientRecord pr = mProviderMap.get(key);
    if(pr == null){
        return null;
    }
    IContentProvider provider = pr.mProvider;
    IBinder jBinder = provider.asBinder();
    if(!jBinder.isBinderAlive()) {
        handleUnstableProvideDiedLocked(jBinder, true);
        return null;
    }
    return provider;
}
```

远程找：

```java
ContentProviderHolder getContentProvider(IApplicationThread caller, ...) {
	return getContentProviderImpl(caller, name, null, ...);
}
ContentProviderHolder getContentProviderImpl(IApplicationThread caller, ...) {
    // 如果 ContentProviderRecord 存在就直接返回
    // ContentProviderRecord 不存在，就创建一个
    // 如果 ContentProviderRecord 能跑在调用者进程，就直接返回，否则往下走
    // 如果 provider 所在进程没启动，就启动进程，然后等待发布，完成的时候返回
    // 如果 binder 对象还没发布就请求发布，然后等待，完成的时候返回
    
    if(providerRunning){
        // provider 存在
    }
    if(!providerRunning){
        // provider 不存在
    }
    synchronized(cpr){
        while(cpr.provider == null){
            cpr.wait();
        }
    }
    return cpr!=null?cpr.newHolder(conn):null;
}
```

```java
if(providerRunning){
	cpi = cpr.info;
	if(r!=null&&cpr.canRunHere(r)){
		ContentProviderHolder holder = cpr.newHolder(null);
		holder.provider = null;
		return holder;
	}
    conn = incProviderCountLocked(r, cpr, token, stable);
    return cpr.newHolder(conn);
}
```

```java
// 应用端安装 provider
IActivityManager.ContentProviderHolder installProvider(Context context, ...) {
	Context c = context.createPackageContext(ai.packageName, ...);
    ContentProvider localProvider = cl.loadClass(info.name).newInstance();
    IContentProvider provider = localProvider.getIContentProvider();
    // 调用 provider#onCreate
    localProvider.attachInfo(c, info);
    
    ProviderClientRecord pr = mLocalProvidersByName.get(cname);
    holder = new IActivityManager.ContentProviderHolder(info);
    holder.provider = provider;
    pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
    mLocalProviders.put(jBinder, pr);
    mLocalProvidersByName.put(cname, pr);
    return pr.mHolder;
}
```

provider 不存在的情况：

```java
if(!providerRunning) {
	cpr = new ContentProviderRecord(this, cpi, ai, comp, singleton);
    if(r!=null&&cpr.canRunHere(r)){
        return cpr.newHolder(null);
    }
    if(i>=N){
        ProcessRecord proc = getProcessRecordLocked(...);
        if(proc!=null&&proc.thread!=null){
            proc.thread.scheduleInstallProvider(cpi);
        }else{
            proc = startProcessLocked(cpi.processName, ...);
        }
        mLaunchingProviders.add(cpr);
    }
}
```

应用启动时主动发布 provider 的 Binder 对象：

```java
boolean attachApplicationLocked(IApplicationThread thread, ...){
    List<ProviderInfo> providers = generateApplicationProvidersLocked(app);
    if(providers != null && checkAppInLaunchingProvidersLocked(app)){
        Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
        msg.obj = app;
        mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
    }
    // 让应用端初始化 provider
    thread.bindApplication(processName, appInfo, providers, ...);
    return true;
}
// 应用主线程执行
private void handleBindApplication(AppBindData data){
    List<ProviderInfo> provider = data.providers;
    if(providers != null){
        installContentProviders(app, providers);
    }
}
void installContentProviders(Context context, List<ProviderInfo> providers){
    List<ContentProviderHolder> results = new ArrayList<ContentProviderHolder>();
    for(ProviderInfo cpi: providers){
        ContentProviderHolder cph = installProvider(context, ...);
        results.add(cph);
    }
    AMP.publishContentProviders(getApplicationThread(), results);
}
void publishContentProviders(IApplicationThread caller, providers){
    final ProcessRecord r = getRecordForAppLocked(caller);
    for(int i=0;i<N;i++){
        ContentProviderHolder src = providers.get(i);
        ContentProviderRecord dst = r.pubProviders.get(src.info.name);
        
        // 更新 provider 缓存
        // 更新 mLaunchingProviders
        // 删除超时计时的消息
        
        synchronized(dst){
            dst.provider = src.provider;
            dst.proc = r;
            dst.notifyAll();
        }
    }
}
```

AMS 主动向应用请求发布 provider：

```java
public void scheduleInstallProvider(ProviderInfo provider){
    sendMessage(H.INSTALL_PROVIDER, provider);
}
public void handleInstallProvider(ProviderInfo info){
    installContentProviders(mInitialApplication, Lists.newArrayList(info));
}
```

#### 总结

Provider 进程未启动：

![](https://i.loli.net/2020/03/25/8SMpsqbDTO5u7YL.png)

Provider 进程已经启动：

![](https://i.loli.net/2020/03/25/nYjFePh2gRbkQ1X.png)

