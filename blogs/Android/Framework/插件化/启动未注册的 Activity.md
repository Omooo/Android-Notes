---
启动未在 Manifest 中注册的 Activity
---

#### 依据

```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager {
	static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }

    static public IActivityManager getDefault() {
        return gDefault.get();
    }

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            IActivityManager am = asInterface(b);
            return am;
        }
    };
}
```

### 基本思路

通过欺骗 AMS，来启动未在 AndroidManifest 中注册的 Activity，基本思路是：

1. 发送要启动的 Activity 信息给 AMS 之前，把这个 Activity 替换为一个在 AndroidManifest 中已经声明的 SubActivity，这样就能绕过 AMS 的检查了。在替换的过程中，要把原来的 Activity 信息存放在 Bundle 中。
2. AMS 通知 App 启动 StubActivity 时，我们自然不会启动 StubActivity，而是在即将启动的时候，把 StubActivity 替换为原先的 Activity，原先的 Activity 信存放在 Bundle 中。

#### 实现

对 AMN 进行 hook 可以同时适用于 Activity 和 Context 的 startActivity 方法。

```java
public class AMSHookHelper {

    public static final String EXTRA_TARGET_INTENT = "extra_target_intent";

    /**
     * Hook AMS
     * 主要完成的操作是  "把真正要启动的Activity临时替换为在AndroidManifest.xml中声明的替身Activity",进而骗过AMS
     */
    public static void hookAMN() throws ClassNotFoundException {

        //获取AMN的gDefault单例gDefault，gDefault是final静态的
        Object gDefault = RefInvoke.getStaticFieldObject("android.app.ActivityManagerNative", "gDefault");

        // gDefault是一个 android.util.Singleton<T>对象; 我们取出这个单例里面的mInstance字段
        Object mInstance = RefInvoke.getFieldObject("android.util.Singleton", gDefault, "mInstance");

        // 创建一个这个对象的代理对象MockClass1, 然后替换这个字段, 让我们的代理对象帮忙干活
        Class<?> classB2Interface = Class.forName("android.app.IActivityManager");
        Object proxy = Proxy.newProxyInstance(
                Thread.currentThread().getContextClassLoader(),
                new Class<?>[]{classB2Interface},
                new MockClass1(mInstance));

        //把gDefault的mInstance字段，修改为proxy
        RefInvoke.setFieldObject("android.util.Singleton", gDefault, "mInstance", proxy);
    }

    /**
     * 由于之前我们用替身欺骗了AMS; 现在我们要换回我们真正需要启动的Activity
     * 不然就真的启动替身了, 狸猫换太子...
     * 到最终要启动Activity的时候,会交给ActivityThread 的一个内部类叫做 H 来完成
     * H 会完成这个消息转发; 最终调用它的callback
     */
    public static void hookActivityThread() throws Exception {

        // 先获取到当前的ActivityThread对象
        Object currentActivityThread = RefInvoke.getStaticFieldObject("android.app.ActivityThread", "sCurrentActivityThread");

        // 由于ActivityThread一个进程只有一个,我们获取这个对象的mH
        Handler mH = (Handler) RefInvoke.getFieldObject(currentActivityThread, "mH");


        //把Handler的mCallback字段，替换为new MockClass2(mH)
        RefInvoke.setFieldObject(Handler.class, mH, "mCallback", new MockClass2(mH));
    }
}
```

 第一步：

```java
class MockClass1 implements InvocationHandler {

    private static final String TAG = "MockClass1";

    Object mBase;

    public MockClass1(Object base) {
        mBase = base;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if ("startActivity".equals(method.getName())) {
            // 只拦截这个方法
            // 找到参数里面的第一个Intent 对象
            Intent raw;
            int index = 0;

            for (int i = 0; i < args.length; i++) {
                if (args[i] instanceof Intent) {
                    index = i;
                    break;
                }
            }
            raw = (Intent) args[index];

            Intent newIntent = new Intent();

            // 替身Activity的包名, 也就是我们自己的包名
            String stubPackage = raw.getComponent().getPackageName();

            // 这里我们把启动的Activity临时替换为 StubActivity
            ComponentName componentName = new ComponentName(stubPackage, StubActivity.class.getName());
            newIntent.setComponent(componentName);

            // 把我们原始要启动的TargetActivity先存起来
            newIntent.putExtra(AMSHookHelper.EXTRA_TARGET_INTENT, raw);

            // 替换掉Intent, 达到欺骗AMS的目的
            args[index] = newIntent;
            return method.invoke(mBase, args);

        }

        return method.invoke(mBase, args);
    }
}
```

第二步：

```java
class MockClass2 implements Handler.Callback {

    Handler mBase;

    public MockClass2(Handler base) {
        mBase = base;
    }

    @Override
    public boolean handleMessage(Message msg) {

        switch (msg.what) {
            // ActivityThread里面 "LAUNCH_ACTIVITY" 这个字段的值是100
            // 本来使用反射的方式获取最好, 这里为了简便直接使用硬编码
            case 100:
                handleLaunchActivity(msg);
                break;

        }

        mBase.handleMessage(msg);
        return true;
    }

    private void handleLaunchActivity(Message msg) {
        // 这里简单起见,直接取出TargetActivity;
        Object obj = msg.obj;

        // 把替身恢复成真身
        Intent intent = (Intent) RefInvoke.getFieldObject(obj, "intent");

        Intent targetIntent = intent.getParcelableExtra(AMSHookHelper.EXTRA_TARGET_INTENT);
        intent.setComponent(targetIntent.getComponent());
    }
}
```

#### 总结

1. Hook ActivityManagerNative 的 gDefault 字段，它是一个单例，mInstance 是 IActivityManager 接口，使用动态代理的方式，去代理 startActivity 方法，在 Intent 里的 Activity 临时替换成已经注册的占位 Activity。
2. Hook ActivityThread 的 mH，它是一个 Handler.Callback，同样使用动态代理的方式，代理 handleMessage 方法，处理 case 为 100 也就是 LAUNCH_ACTIVITY，在这里面去启动真正的 Activity。

采用这种思路，第二步还可以 hook ActivityThread 的 mInstrumention 字段，Instrumention 里面有两个方法可以拦截：

1. Activity newActivity(ClassLoader cl, String className, Intent intent)
2. Void callActivityOnCreate(Activity activity, ...)

callActivityOnCreate 没有 Intent 参数，所以无法读取之前存放的要启动的 Activity，所以选择拦截 newActivity 方法。

#### 弊端

1. AMS 会认为每次要打开的 Activity 都是 SubActivity，那就相当于那些没有在 AndroidManifest 文件里面注册的 Activity 的 LaunchMode 就只能是默认类型，即使设置了 singleTop 或 singleTask，也不会生效。
2. 兼容性问题，目前在 Android 6/7 上可行，但在 Android 7 以上就不行了，因为 AMN 的 gDefault 已经不存在了。

#### Android 8 适配

在 Android 8 中，ActivityManagerNative 中已经不存在 gDefault 字段了，转移到了 ActivityManager 类中，但此时这个字段改名为 IActivityManagerSingleton：

```java
    // ActivityManager
    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```

所以，需要的处理就变成了：

```java
        Object gDefault = null;

        if (android.os.Build.VERSION.SDK_INT <= 25) {
            // 获取 AMN 的 gDefault t单例 gDefault，gDefault是静态的
            gDefault = RefInvoke.getStaticFieldObject("android.app.ActivityManagerNative", "gDefault");
        } else {
            // 获取 ActivityManager 的单例 IActivityManagerSingleton，它其实就是之前的 gDefault
            gDefault = RefInvoke.getStaticFieldObject("android.app.ActivityManager", "IActivityManagerSingleton");
        }
```

#### Android 9 适配

在 Android 9 之前，我们是 Hook 掉 H 类的 mCallback 对象，拦截 handleMessage 方法，在里面它会根据 msg.what 来判断到底是哪种消息。在 H 类中，定义了几十种消息，比如说 LAUNCH_ACTIVITY 的值是 100，PAUSE_ACTIVITY 的值是 101。从 100-109 都是给 Activity 的生命周期函数准备的。从 110 开始，才是给 Application、Service、ContentProvider、BroadcastReceiver 准备的。

但是在 Android 9，它重构了 H 类，把 100 到 109 这十个用于 Activity 的消息，都合并为 159 这个消息，消息名为 EXECUTE_TRANSACTION。

为什么要这么改呢？我们知道 Activity 的生命周期图，这其实是一个由 Create、Pause、Resume、Stop、Destory、Restart 组成的状态机。按照设计模式中状态模式的定义，可以把每个状态都定义成一个类，于是便有了如下的类图：

![](https://i.loli.net/2020/04/27/NgncLAk9jmGlDKY.png)

就拿 LaunchActivity 来说，在 Android P 之前，是在 H 类的 handleMessage 方法的 switch 分支语句中，编写启动一个 Activity 的逻辑：

```java
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                } break;
            }
        }
```

在 Android P 中，启动 Activity 的这部分逻辑，被转移到了 LaunchActivityItem 类的 execute 方法中：

```java
public class LaunchActivityItem extends ClientTransactionItem {

    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    }
}
```



这就导致了我们之前的插件化解决方法，在 Android 9 上是不生效的，因为找不到 100 这个消息。为此我们需要拦截 159 这个消息。拦截后还需要判断这个消息到底是 Launch 还是 Pause 或者是 Resume。

关键在于 H 类的 handleMessage 方法的 Message 参数，这个 Message 的 obj 字段，在 Message 是 159 的时候，返回的是 ClientTransaction 类型对象，它内部有一个 mActivityCallbacks 集合：

```java
public class ClientTransaction implements Parcelable, ObjectPoolItem {

      private List<ClientTransactionItem> mActivityCallbacks;

}
```

这个 mActivityCallbacks 集合中，存放的是 ClientTransactionItem 的各种子类对象，比如 LaunchActivityItem、DestoryActivityItem。拿到 LaunchActivityItem 类的对象，它内部有一个 mIntent 字段，里面存放的就是要启动的 Activity 名称，在这里进行替换。

代码如下：

```java
class MockClass2 implements Handler.Callback {

    Handler mBase;

    public MockClass2(Handler base) {
        mBase = base;
    }

    @Override
    public boolean handleMessage(Message msg) {

        switch (msg.what) {
            // ActivityThread里面 "LAUNCH_ACTIVITY" 这个字段的值是100
            // 本来使用反射的方式获取最好, 这里为了简便直接使用硬编码
            case 100:   //for API 28以下
                handleLaunchActivity(msg);
                break;
            case 159:   //for API 28
                handleActivity(msg);
                break;
        }

        mBase.handleMessage(msg);
        return true;
    }

    private void handleActivity(Message msg) {
        // 这里简单起见,直接取出TargetActivity;
        Object obj = msg.obj;

        List<Object> mActivityCallbacks = (List<Object>) RefInvoke.getFieldObject(obj, "mActivityCallbacks");
        if(mActivityCallbacks.size() > 0) {
            String className = "android.app.servertransaction.LaunchActivityItem";
            if(mActivityCallbacks.get(0).getClass().getCanonicalName().equals(className)) {
                Object object = mActivityCallbacks.get(0);
                Intent intent = (Intent) RefInvoke.getFieldObject(object, "mIntent");
                Intent target = intent.getParcelableExtra(AMSHookHelper.EXTRA_TARGET_INTENT);
                intent.setComponent(target.getComponent());
            }
        }
    }
}
```

#### 参考

[https://www.cnblogs.com/Jax/p/9316422.html](https://www.cnblogs.com/Jax/p/9316422.html)

[https://github.com/BaoBaoJianqiang/AndroidSourceCode](https://github.com/BaoBaoJianqiang/AndroidSourceCode)