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

#### View 刷新机制

