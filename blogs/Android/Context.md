---
Context
---

#### 目录

1. 思维导图
2. Context 类层次
3. Context 作用以及作用域
4. ContextWrapper、ContextImpl、ContextThemeWrapper
5. 常见问题汇总
6. 参考

#### 思维导图

![](https://i.loli.net/2019/01/05/5c3025010fcfa.png)

#### Context 类层次

![](https://i.loli.net/2019/01/04/5c2ecc5c23ca3.png)

#### Context 作用以及作用域

##### 作用：

看接口就能看出来。

```java
public abstract class Context {
    public abstract AssetManager getAssets();
    public abstract Resources getResources();
    public abstract PackageManager getPackageManager();
    public abstract ContentResolver getContentResolver();
    public abstract Looper getMainLooper();
    public Executor getMainExecutor() {
        return new HandlerExecutor(new Handler(getMainLooper()));
    }
    public abstract Context getApplicationContext();
    //...
}
```

作用如下：

1. 四大组件的交互，包括启动 Activity、Broadcast、Service，获取 ContentResolver 等
2. 获取系统 / 应用资源，包括 AssetManager、PackageManager、Resources、SystemService 以及 color、string、drawable 等
3. 文件、SharedPreference、数据库相关
4. 其他辅助功能，比如设置 ComponentCallbacks，即监听配置信息改变、内存不足等事件的发生

##### 作用域：

虽然 Context 神通广大，但是并不是随便拿到一个 Context 实例就可以为所欲为，还是有一些限制的。在绝大多数场景下，Activity、Service 和 Application 这三种类型的 Context 都是通用的，不过也有几种场景比较特殊，比如启动 Activity、弹出 Dialog。Android 是不允许 Activity 或 Dialog 凭空出现的，一个 Activity 的启动必须建立在另外一个 Activity 的基础之上，也就以此形成任务栈。而 Dialog 则必须在一个 Activity 的上面弹出（除非是 System Alert 类型的 Dialog），因此在这种场景下，我们只能使用 Activity 类的 Context。

所以，最常见的错误场景就是：

1. 使用 ApplicationContext 去启动一个 LaunchMode 为 standard 的 Activity

   会报错，因为非 Activity 类型的 Context 没呀所谓的任务栈。

#### ContextWarpper、ContextImpl、ContextThemeWrapper

ContextWrapper 是 Context 的包装类，ContextImpl 才是 Context 的真正实现类，ContextWrapper 的核心工作都是交给其成员变量 mBase 来完成，mBase 是通过 attachBaseContext() 方法来设置的，本质上是 ContextImpl 对象。

ContextThemeWrapper 包含与主题相关的资源，所以 Activity 会继承它，而 Application、Service 不需要主题则没有继承它。

#### 常见问题汇总

1. getApplication、getApplicationContext 区别

   在绝大多数场景下，getApplication 和 getApplicationContext 这两个方法完全一致，返回值也相同。区别在于 getApplication 只存在于 Activity 和 Service 对象，对于 BroadcastReceiver 和 ContentProvider 只能使用 getApplicationContext。

   但是对于 ContextProvider 使用 getApplicationContext 可能会出现空指针问题。当同一个进程有多个 apk 的情况下，对于第二个 apk 是由 provide 方式拉起的，provide 创建过程并不会初始化所在 application，此时返回的结果是 null。

#### 参考

[深入理解 Android 中的各种 Context](https://juejin.im/post/5c1fab7d5188254eb05fbe48)

[Context都没弄明白，还怎么做Android开发？](https://www.jianshu.com/p/94e0f9ab3f1d)

[理解 Android Context](http://gityuan.com/2017/04/09/android_context/)