---
View 工作原理
---

#### 目录

1. 思维导图
2. 概述
3. 
4. 参考

#### 思维导图

#### 概述

1. 初始化 PhoneWindow 和 WindowManager
2. 初始化 DecorView
3. ViewRootImpl 的创建和关联 DecorView
4. 建立 PhoneWindow 和 WMS 之间的连接
5. 建立与 SurfaceFlinger 之间的连接
6. 申请 Surface
7. 正式绘制 View 并显示

#### 步骤一：初始化 PhoneWindow 和 WindowManager

我们知道，Activity 是在 ActivityThread 的 performLaunchActivity 中进行创建的，在创建完成之后就会调用其 attach 方法，它是先于 onCreate、onStart、onResume 等生命周期函数的，因此将 attach 方法作为这篇文章主线的开头：

```java
   // Activity#attach()：
   
	final void attach(...) {
        attachBaseContext(context);
		//初始化 PhoneWindow
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        
		//初始化 WindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        mWindowManager = mWindow.getWindowManager();
    }

	// Window#setWindowManager()：

    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        //...
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
```

attach() 方法就是 new 一个 PhoneWindow 并且关联 WindowManager。

#### 步骤二：初始化 DecorView

接下来就到了 onCreate 方法：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
public void setContentView(int layoutResID) {
    installDecor();
    mLayoutInflater.inflate(layoutResID, mContentParent)
}
```

这一步就是把我们的布局文件解析成 View 塞到 DecorView 的一个 id 为 R.id.content 的 ContentView 中，DecorView 本身是一个 FrameLayout，它还承载了 StatusBar、NavigationBar 。

#### 步骤三：ViewRootImpl 的创建和关联 DecorView

然后在 handleResumeActivity 中，通过 WindowManager 的 addView 方法把 DecorView 添加进去，实际实现是 WindowManagerImpl 的 addView 方法，它里面再通过 WindowManagerGlobal 的实例去 addView 的，在它里面就会 new 一个 ViewRootImpl，也就是说最后是把 DecorView 传给了 ViewRootImpl 的 setView 方法。ViewRootImpl 是 DecorView 的管理者，它负责 View 树的测量、布局、绘制，以及通过 Choreographer 来控制 View 的刷新。

#### 步骤四：建立 PhoneWindow 和 WindowManagerService 之间的连接

WMS 是所有 Window 窗口的管理员，负责 Window 的添加和删除、Surface 的管理和事件派发等等，因此每一个 Activity 中的 PhoneWindow 对象如果需要显示等操作，就必须要与 WMS 交互才能进行。

在 ViewRootImpl 的 setView 方法中，会调用 requestLayout，并且通过 WindowSession 的 addToDisplay 与 WMS 进行交互。WMS 会为每一个 Window 关联一个 WindowStatus。

#### 步骤五：建立与 SurfaceFlinger 的连接

SurfaceFlinger 主要是进行 Layer 的合成和渲染。

在 WindowStatus 中，会创建 SurfaceSession，SurfaceSession 会在 Native 层构造一个 SurfaceComposerClient 对象，它是应用程序与 SurfaceFlinger 沟通的桥梁。

#### 步骤六：申请 Surface

经过步骤四和步骤五之后，ViewRootImpl 与 WMS、SurfaceFlinger 都已经建立起连接，但此时 View 还没显示出来，我们知道，所有的 UI 最终都要通过 Surface 来显示，那么 Surface 是什么时候创建的呢？

这就要回到前面所说的 ViewRootImpl 的 requestLayout 方法了，首先会 checkThread 检查是否是主线程，然后调用 scheduleTraversals 方法，scheduleTraversals 方法会先设置同步屏障，然后通过 Choreographer 类在下一帧到来时去执行 doTraversal 方法。简单来说，Choreographer 内部会接受来自 SurfaceFlinger 发出的 Vsync 垂直同步信号，这个信号周期一般是 16ms 左右。doTraversal 方法首先会先移除同步屏障，然后 performTraversals 真正进行 View 的绘制流程，即调用 performMeasure、performLayout、performDraw。不过在它们之前，会先调用 relayoutWindow 通过 WindowSession 与 WMS 进行交互，即把 Java 层创建的 Surface 与 Native 层的 Surface 关联起来。

#### 步骤七：正式绘制 View 并显示



#### 参考

[https://juejin.im/post/5c67c1e16fb9a04a05403549](https://juejin.im/post/5c67c1e16fb9a04a05403549)

[https://juejin.im/post/5bf16ff5f265da6141712acc](https://juejin.im/post/5bf16ff5f265da6141712acc)

