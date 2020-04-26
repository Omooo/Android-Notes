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



#### 参考

[https://www.cnblogs.com/Jax/p/9316422.html](https://www.cnblogs.com/Jax/p/9316422.html)

[https://github.com/BaoBaoJianqiang/AndroidSourceCode](https://github.com/BaoBaoJianqiang/AndroidSourceCode)