---
Framework 解决实际问题
---

1. 你有没有解决复杂问题的实践经验
2. 你有没有深入研究底层原理的习惯
3. 你的知识体系是否有一定深度

#### 应用组件

1. 想了解一下为什么 Activity 在 onResume 之后才会显示出来
2. bindService 的时候 onRebind 总是调不到，研究原理
3. 广播 onReceiver 的 context 可否启动 Activity，显示 Dialog？
4. 发现 provider 的 onCreate 比 Application 还早，研究一下

#### 消息通信

1. intent 带的数据量大了会异常，研究原因
2. 需要跨进程传大图，研究 Bitmap 传输原理，Ashmem 机制
3. 想知道 Handler 消息延时的精度怎么样，去了解原理
4. 为什么有时候 IdleHandler 调不到，去了解原理

#### 性能优化

1. ANR 了，看主线程有没有耗时任务
2. 卡断掉帧，了解屏幕刷新机制
3. 启动速度优化，了解应用启动原理
4. 内存优化，清理不必要的资源

```java
public class Resources{
	private static final LongSparseArray<ConstantState>[] sPreloadedDrawables;
    private static final LongSparseArray<ConstantState> sPreloadedColorDrawableds = new LongSparseArray<>();
    private static final LongSparseArray<ConstantState<ColorStateList>> sPreloadedColorStateLists = new LongSparseArray<>();
}
 public static void main(String argvp[]){
     registerZygoteSocket(socketName);
     preload();
     if(startSystemServer){
         startSystemServer(abiList, socketName);
     }
     runSelectLoop(abiList);
     closeServerSocket();
 }
static void preload(){
    preloadClasses();
    preloadResources();
    preloadOpenGL();
    preloadSharedLibraries();
    preloadTextResources();
    WebViewFactory.prepareWebViewInZygote();
}
private static void preloadResources(){
    mResources.startPreloading();
    if(PRELOAD_RESOURCES){
        preloadDrawables(runtime, ar);
        preloadColorStateLists(runtime, ar);
    }
    mResources.finishPreloading();
}
private static int preloadDrawables(VMRuntime runtime, TypedArray ar){
    int N = ar.length();
    for(int i=0;i<N;i++){
        int id = ar.getResourceId(i, 0);
        mResources.getDrawable(id, null);
    }
    return N;
}
Drawable getDrawable(){
    final Drawable res = loadDrawable(value, id, theme);
}
Drawable loadDrawable(TypeValue value, int id, Theme theme){
    Drawable dr;
    if(isColorDrawable){
        dr = new ColorDrawable(value.data);
    }else{
        dr = loadDrawableForCookie(value, id, null);
    }
    cacheDrawable(value, ... caches, theme, ..., dr);
    return dr;
}
private void cacheDrawable(..., long key, Drawable dr){
    if(isColorDrawable){
        sPreloadedColorDrawables.put(key, cs);
    }else{
        if((changingConfigs&LAYOUT_DIR_CONFIG)==0){
            sPreloadedDrawables[0].put(key, cs);
            sPreloadedDrawables[1].put(key, cs);
        }else{
            sPreloadedDrawables[mConfiguration.getLayoutDirection()].put(key, cs);
        }
    }
}

Drawable loadDrawable(TypeValue value, int fd, Theme theme){
    final Drawable cachedDrawable = caches.getInsstance(key, theme);
    if(cachedDrawable!=null){
        return cachedDrawable;
    }
    cs = sPreloadedDrawables[mConfiguration.getLayoutDirection()].get(key);
    if(cs!=null){
        dr = cs.newDrawable(this);
    }else{
        dr = loadDrawableForCookie(value, id, null);
    }
    cacheDrawable(value, isColorDrawable, caches, theme, ...);
    return dr;
}
void cacheDrawable(..., DrawableCache caches, ..., long key, Drawable dr){
    final ConstantState cs = dr.getConstantState();
    synchronized(mAccessLock){
        caches.put(key, cs,);
    }
}
```

总结

1. zygote 启动的时候会预加载系统资源
2. zygote 启动应用进程，应用进程继承了这些系统资源
3. 应用 getDrawable 先检查自己的 cache，在检查预加载资源
4. 加载完 drawable 后缓存到自己的 cache

解决方案：

1. 可以考虑清理这些预加载资源
2. 可以通过反射获取这些资源缓存，然后清理掉
3. 适用于不需要 UI 的单独进程的后台 Service
4. 带 UI 的进程最好还是别随便清

