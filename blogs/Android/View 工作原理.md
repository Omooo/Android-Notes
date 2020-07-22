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

接下来就是正式绘制 View 了，从 performTraversals 开始，Measure、Layout、Draw 三步走。

第一步是获取 DecorView 的宽高的 MeasureSpec 然后执行 performMeasure 流程。MeasureSpec 简单来说就是一个 int 值，高 2 位表示测量模式，低 30 位用来表示大小。策略模式有三种，EXACTLY、AT_MOST、UNSPECIFIED。EXACTLY 对应为 match_parent 和具体数值的情况，表示父容器已经确定 View 的大小；AT_MOST 对应 wrap_content，表示父容器规定 View 最大只能是 SpecSize；UNSPECIFIED 表示不限定测量模式，父容器不对 View 做任何限制，这种适用于系统内部。接着说，performMeasure 中会去调用 DecorView 的 measure 方法，这个是 View 里面的方法并且是 final 的，它里面会把参数透传给 onMeasure 方法，这个方法是可以重写的，也就是我们可以干预 View 的测量过程。在 onMeasure 中，会通过 getDefaultSize 获取到宽高的默认值，然后调用 setMeasureDimension 将获取的值进行设置。在 getDefaultSize 中，无论是 EXACTLY 还是 AT_MOST，都会返回 MeasureSpec 中的大小，这个 SpecSize 就是测量后的最终结果。至于 UNSPECIFIED 的情况，则会返回一个建议的最小值，这个值和子元素设置的最小值以及它的背景大小有关。从这个默认实现来看，如果我们自定义一个 View 不重写它的 onMeasure 方法，那么 warp_content 和 match_parent 一样。所以 DecorView 重写了 onMeasure 函数，它本身是一个 FrameLayout，所以最后也会调用到 FrameLayout 的 onMeasure 函数，作为一个 ViewGroup，都会遍历子 View 并调用子 View 的 measure 方法。这样便实现了层层递归调用到了每个子 View 的 onMeasure 方法进行测量。

第二步是执行 performLayout 的流程，也就是调用到 DecorView 的 layout 方法，也就是 View 里面的方法，如果 View 大小发生变化，则会回调 onSizeChanged 方法，如果 View 状态发生变化，则会回调 onLayout 方法，这个方法在 View 中是空实现，因此需要看 DecorView 的父容器 FrameLayout 的 onLayout 方法，这个方法就是遍历子 View 调用其 layout 方法进行布局，子 View 的 layout 方法被调用的时候，它的 onLayout 方法又会被调用，这样就布局完了所有的 View。

第三步就是 performDraw 方法了，里面会调用 drawSoftware 方法，这个方法需要先通过 mSurface lockCanvas 获取一个 Canvas 对象，作为参数传给 DecorView 的 draw 方法。这个方法调用的是 View 的 draw 方法，先绘制 View 背景，然后绘制 View 的内容，如果有子 View 则会调用子 View 的 draw 方法，层层递归调用，最终完成绘制。

完成这三步之后，会在 ActivityThread 的 handleResumeActivity 最后调用 Activity 的 makeVisible，这个方法就是将 DecorView 设置为可见状态。

#### 参考

[https://juejin.im/post/5c67c1e16fb9a04a05403549](https://juejin.im/post/5c67c1e16fb9a04a05403549)

[https://juejin.im/post/5bf16ff5f265da6141712acc](https://juejin.im/post/5bf16ff5f265da6141712acc)

