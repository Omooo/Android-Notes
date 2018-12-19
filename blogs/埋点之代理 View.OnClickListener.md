---
埋点之代理 View.OnClickListener
---

#### 目录

1. 思维导图
2. 原理
3. 实现
4. 优缺点
5. 参考

#### 思维导图

#### 原理

在注册的 Application.ActivityLifecycleCallbacks 的 onActivityResumed(Activity activity) 回调方法中，通过 activity.getWindow().getDecorView 获取 DecorView，然后遍历 DecorView，判断当前 View 是否设置了 onClickListener，如果已设置并且不是我们的 OnClickListenerWrapper，就用自定义的 OnClickListenerWrapper 替代当前 View.onClickListener。WrapperOnClickListener 的 onClick 里会先调用原有的 OnClickListener 处理逻辑，然后再调用埋点代码，从而达到自动埋点的效果。

之所以选择 DecorView 而不是 ContentView（Activity#setContentView），主要是考虑到了 MenuItem 的点击事件。

但是，这样并不能拦截到动态添加的 View，所以还需要引入 ViewTreeObserver.OnGlobalLayoutListener 来监听视图树变化，而且要在 Activity#onStop 中 removeOnGlobalLayoutListener。

#### 实现

1. 首先创建代理类

   ```java
   public class ClickListenerWrapper implements View.OnClickListener {
       private static final String TAG = "HookedClickListenerProx";
       private View.OnClickListener origin;
   
       public ClickListenerWrapper(View.OnClickListener origin) {
           this.origin = origin;
       }
   
       @Override
       public void onClick(View v) {
           if (origin != null) {
               Log.i(TAG, "onClick: 前");
               origin.onClick(v);
               Log.i(TAG, "onClick: 后");
           }
       }
   }
   ```

2. 代理原 OnClickListener 事件

   ```java
   public class HookOnClickHelper {
       private static final String TAG = "HookOnClickHelper";
   
       public static void hook(View view) throws Exception {
           //第一步：反射得到 ListenerInfo 对象
           Method getListenerInfo = View.class.getDeclaredMethod("getListenerInfo");
           getListenerInfo.setAccessible(true);
           Object listenerInfo = getListenerInfo.invoke(view);
           //第二步：得到原始的 OnClickListener 事件方法
           Class<?> listenerInfoClz = Class.forName("android.view.View$ListenerInfo");
           Field mOnClickListener = listenerInfoClz.getDeclaredField("mOnClickListener");
           mOnClickListener.setAccessible(true);
           View.OnClickListener originOnClickListener = (View.OnClickListener) mOnClickListener.get(listenerInfo);
           if (originOnClickListener == null || originOnClickListener instanceof ClickListenerWrapper) {
               //如果没有设置点击事件或者已经代理过了，则跳过
               return;
           }
           //第三步：替换
           ClickListenerWrapper proxy = new ClickListenerWrapper(originOnClickListener);
           mOnClickListener.set(listenerInfo, proxy);
       }
   }
   ```

3. 在 Activity 完全显示的时候遍历 View 动态代理 OnClickListener 

   ```java
   public class MyApplication extends Application {
   
       private static final String TAG = "MyApplication";
   
       @Override
       public void onCreate() {
           super.onCreate();
   
           this.registerActivityLifecycleCallbacks(new MyActivityLifecycleCallback());
       }
   
       static class MyActivityLifecycleCallback implements ActivityLifecycleCallbacks {
   
           private View mDecorView;
           private ViewTreeObserver.OnGlobalLayoutListener mOnGlobalLayoutListener=new ViewTreeObserver.OnGlobalLayoutListener() {
               @Override
               public void onGlobalLayout() {
                   setAllViewsProxy((ViewGroup) mDecorView);
               }
           };
   
           @Override
           public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
           }
   
           @Override
           public void onActivityStarted(Activity activity) {
           }
   
           @Override
           public void onActivityResumed(Activity activity) {
               mDecorView = activity.getWindow().getDecorView();
               setAllViewsProxy((ViewGroup) mDecorView);
           }
   
           @Override
           public void onActivityPaused(Activity activity) {
           }
   
           @Override
           public void onActivityStopped(Activity activity) {
               mDecorView.getViewTreeObserver().removeGlobalOnLayoutListener(mOnGlobalLayoutListener);
           }
   
           @Override
           public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
           }
   
           @Override
           public void onActivityDestroyed(Activity activity) {
   
           }
   
   
           private void setAllViewsProxy(ViewGroup viewGroup) {
               int childCount = viewGroup.getChildCount();
               for (int i = 0; i < childCount; i++) {
                   View view = viewGroup.getChildAt(i);
                   if (view instanceof ViewGroup) {
                       setAllViewsProxy((ViewGroup) view);
                   } else {
                       try {
                           if (view.hasOnClickListeners()) {
                               HookOnClickHelper.hook(view);
                           }
                       } catch (Exception e) {
                           e.printStackTrace();
                       }
                   }
               }
           }
       }
   }
   ```

#### 优缺点

优点：无

缺点：

1. 使用反射，对 APP 整体性能有影响，也可能会引入兼容性问题
2. 无法采集游离于 Activity 之上的 View 的点击，比如 Dialog、PopupWindow等等
3. OnGlobalLayoutListener API 16+

#### 参考

《神策数据 Android 全埋点技术白皮书》

[Android Hook 机制之简单实战](https://www.jianshu.com/p/c431ad21f071)