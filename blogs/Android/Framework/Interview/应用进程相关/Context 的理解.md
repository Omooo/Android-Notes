---
谈谈你对 Context 的理解？
---

1. 了解 Context 的作用
2. 熟悉 Context 初始化流程
3. 深入理解不同应用组件之间 Context 的区别

#### 问题

1. 应用里面有多少个 Context？不同的 Context 之间有什么区别？
2. Activity 里的 this 和 getBaseContext 有什么区别？
3. getApplication 和 getApplicationContext 有什么区别？
4. 应用组件的构造，onCreate、attachBaseContext 调用顺序？



```java
class ContextImpl extends Context {
	final ActivityThread mMainThread;
	final LoadedApk mPackageInfo;
	
	private final ResourcesManager mResourcesManager;
    private final Resource mResources;
    
	private Resources.Theme mTheme = null;
    private PackageManager mPackageManager;
	
	final Object[] mServiceCache = SystemServiceRegistry.createServiceCache();
}
```

#### Context 是在哪创建的？

Application ：

1. 继承关系

   Application <- ContextWrapper <- Context

2. 调用顺序

   \<init> -> attachBaseContext -> onCreate

Activity：

```java
private Activity performLaunchActivity() {
	Activity activity = null;
	activity = mInstrumentation.newActivity();
	
	Application app = r.packageInfo.makeApplication();
	Context appContext = createBaseContextForActivity(r, activity);
	activity.attach(appContext, app, ...);
	
	activity.onCreate();
	return activity;
}
```

1. 继承关系

   Activity <- ContextThemeWrapper <- ContextWrapper

2. 调用顺序

   \<init> -> attachBaseContext -> onCreate

Service:

```java
private void handleCreateService(CreateServiceData data) {
	Service service = null;
    service = (Service) cl.loadClass(data.info.name).newInstance();
    
    ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
    context.setOuterContext(service);
    
    Application app = packageInfo.makeApplication();
    service.attach(context, app);
    service.onCreate();
}
```

