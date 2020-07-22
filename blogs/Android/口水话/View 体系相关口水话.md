---
View 体系相关口水话
---

#### 目录

1. View 绘制流程
2. View 事件分发
3. View 刷新机制

#### View 绘制流程

View 的显示是以 Activity 为载体的，Activity 是在 ActivityThread 的 performLaunchActivity 中进行创建的，在创建完成之后就会调用其 attach 方法，这个方法是优先于 onCreate、onStart 等生命周期函数的，attach 方法所做的就是 new 一个 PhoneWindow 并关联 WindowManager。接下来就是 onCreate 方法，这一步就是把我们的布局文件解析成 View 塞到 DecorView 的一个 id 为 R.id.content 的 mContentView 中。DecorView 本身是一个 FrameLayout，它还承载了 StatusBar 和 NavigationBar。然后在 handleResumeActivity 中，通过 WindowManagerImpl 的 addView 方法把 DecorView 添加进去，实际实现是 WindowManagerGlobal 的 addView 方法，它里面管理着所有的 DecorView 及其对应的 ViewRootImpl，ViewRootImpl 是 DecorView 的管理者，它负责 View 树的测量、布局、绘制，以及通过 Choreographer 来控制 View 的刷新。

WMS 是所有 Window 窗口的管理者，它负责 Window 的添加和删除、Surface 的管理和事件分发等等，因此每一个 Activity 中的 PhoneWindow 对象如果需要显示等操作，就需要要与 WMS 交互才能进行。这一步是在 ViewRootImpl 的 setView 方法中，会调用 requestLayout，并且通过 WindowSession 的 addToDisplay 与 WMS 进行交互，WMS 会为每一个 Window 关联一个 WindowState。除此之外，ViewRootImpl 的 setView 还做了一件重要的事就是注册 InputEventReceiver，这和 View 事件分发有关。

在 WindowState 中，会创建 SurfaceSession，它会在 Native 层构造一个 SurfaceComposerClient 对象，它是应用程序与 SurfaceFlinger 沟通的桥梁。至此，ViewRootImpl 与 WMS、SurfaceFlinger 都已经建立起连接，但是此时 View 还没显示出来，我们知道，所有的 UI 最终都要通过 Surface 来显示，那么 Surface 是什么时候创建的呢？

这就要回到前面所说的 ViewRootImpl 的 requestLayout 方法了，这个方法首先会 checkThread 检查是否是主线程，然后调用 scheduleTraversals 方法，这个方法首先会设置同步屏障，然后通过 Choreographer 在下一帧到来时去执行 doTraversal 方法。在 doTraversal 中调用 performTraversal 中真正进行 View 的绘制流程，即调用 performMeasure、performLayout、performDraw。不过在它们之前，会先调用 relayoutWindow 通过 WindowSession 与 WMS 进行交互，即把 Java 层创建的 Surface 与 Native 层的 Surface 关联起来。



#### View 事件分发

在最新的 Android 系统中，事件的处理者不再由 InputEventReceiver 独自承担，而是通过多种形式的 InputStage 来分别处理，它们都有一个回调接口 onProcess 函数，这些都声明在 ViewRootImpl 内部类里面，并且在 setView 里面进行注册，比如有 ViewPreImeInputStage 用于分发 KeyEvent，这里我们重点关注与 MotionEvent 事件分发相关的 ViewPostImeInputStage。在它的 onProcess 函数中，如果判断事件类型是 SOURCE_CLASS_POINTER，即触摸屏的 MotionEvent 事件，就会调用 mView 的 dispatchPointerEvent 方法处理。也就是说事件分发最开始是传递给 DecorView 的，DecorView 的 dispatchTouchEvent 是传给 Window Callback 接口方法 dispatchTouchEvent，而 Activity 实现了 Window Callback 接口，在 Activity 的 dispatchTouchEvent 方法里，是调到 Window 的 dispatchTouchEvent，Window 的唯一实现类 PhoneWindow 又会把这个事件回传给 DecorView，DecorView 在它的 superDispatchTouchEvent 把事件转交给了 ViewGroup。

所以，事件分发的流程是：

```xml
DecorView -> Activity -> PhoneWindow -> DecorView -> ViewGroup -> View
```

这里面涉及了三个元素，Activity、ViewGroup 和 View。Activity 的 dispatchTouchEvent 前面说过，它的 onTouchEvent 一般都是返回 false 不消费往下传；在说 View 的 dispatchTouchEvent，如果注册了 OnTouchListener 就调用其 onTouch 方法，如果 onTouch 返回 false 还会接着调用 onTouchEvent 函数，onTouchEvent 作为一种兜底方案，它在内部会根据 MotionEvent 的不同类型做相应处理，比如是 ACTION_UP 就需要执行 performClick 函数。ViewGroup 因为涉及对子 View 的处理，其派发流程没有 View 那么简单直接，它重写了 dispatchTouchEvent 方法，如果 ViewGroup 允许拦截，就调用其 onInterceptTouchEvent 来判断是否要真正执行拦截了，如果拦截了就交由自己的 onTouchEvent 处理，如果不拦截，就从后遍历子 View 处理，它有两个函数可以过滤子 View，一个是判断这个子 View 是否接受 Pointer Events 事件，另一个是判断落点有没有落在子 View 范围内。如果都满足，则调用其 dispatchTouchEvent 处理。如果该子 View 是一个 ViewGroup 就继续调用其 dispatchTouchEvent，否则就是 View 的 dispatchTouchEvent 方法，如此循环往复，直到事件真正被处理。

伪代码表示为：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean consume = false;
    if (onInterceptTouchEvent(event)) {
        consume = onTouchEvent(event);
    } else {
        consume = child.dispatchTouchEvent(event);
    }
    return consume;
}
```

最后可以画一下这个图：

![](https://i.loli.net/2020/07/22/qgVSpUYRJP7ycrO.jpg)

#### View 刷新机制

当我们调用 View 的 invalidate 时刷新视图时，它会调到 ViewRootImp 的 invalidateChildInParent，这个方法首先会 checkThread 检查是否是主线程，然后调用其 scheduleTraversals 方法。这个方法就是视图绘制的开始，但是它并不是立即去执行 View 的三大流程，而是先往消息队列里面添加一个同步屏障，然后在往 Choreographer 里面注册一个 TRAVERSAL 的回调。在下一次 Vsync 信号到来时，会去执行 doTraversals 方法。

Choreographer 主要是用来接收 Vsync 信号，并且在信号到来时去处理一些回调事件。事件类型有四种，分别是 Input、Animation、Traversal、Commit。在 Vsync 信号到来时，会依次处理这些事件，前三种比较好理解，第四种 Commit 是用来执行组件的 onTrimMemory 函数的。Choreographer 是通过 FrameDisplayEventReceiver 来监听底层发出的 Vsync 信号的，然后在它的回调函数 onVsync 中去处理，首先会计算掉帧，然后就是 doCallbacks 处理上面所说的回调事件。

Vsync 信号可以理解为底层硬件的一个消息脉冲，它每 16ms 发出一次，它有两种方式发出，一种是 HWComposer 硬件产生，一种是用软件模拟，即 VsyncThread。不管使用哪种方式，都统一由 DispSyncThread 进行分发。