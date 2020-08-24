---
View 体系相关口水话
---

#### 目录

1. View 绘制流程
2. View 事件分发
3. View 刷新机制
4. 项目总结

#### View 绘制流程

View 的显示是以 Activity 为载体的，Activity 是在 ActivityThread 的 performLaunchActivity 中进行创建的，在创建完成之后就会调用其 attach 方法，这个方法是优先于 onCreate、onStart 等生命周期函数的，attach 方法所做的就是 new 一个 PhoneWindow 并关联 WindowManager。接下来就是 onCreate 方法，这一步就是把我们的布局文件解析成 View 塞到 DecorView 的一个 id 为 R.id.content 的 mContentView 中。DecorView 本身是一个 FrameLayout，它还承载了 StatusBar 和 NavigationBar。然后在 handleResumeActivity 中，通过 WindowManagerImpl 的 addView 方法把 DecorView 添加进去，实际实现是 WindowManagerGlobal 的 addView 方法，它里面管理着所有的 DecorView 及其对应的 ViewRootImpl，ViewRootImpl 是 DecorView 的管理者，它负责 View 树的测量、布局、绘制，以及通过 Choreographer 来控制 View 的刷新。

WMS 是所有 Window 窗口的管理者，它负责 Window 的添加和删除、Surface 的管理和事件分发等等，因此每一个 Activity 中的 PhoneWindow 对象如果需要显示等操作，就需要要与 WMS 交互才能进行。这一步是在 ViewRootImpl 的 setView 方法中，会调用 requestLayout，并且通过 WindowSession 的 addToDisplay 与 WMS 进行交互，WMS 会为每一个 Window 关联一个 WindowState。除此之外，ViewRootImpl 的 setView 还做了一件重要的事就是注册 InputEventReceiver，这和 View 事件分发有关。

在 WindowState 中，会创建 SurfaceSession，它会在 Native 层构造一个 SurfaceComposerClient 对象，它是应用程序与 SurfaceFlinger 沟通的桥梁。至此，ViewRootImpl 与 WMS、SurfaceFlinger 都已经建立起连接，但是此时 View 还没显示出来，我们知道，所有的 UI 最终都要通过 Surface 来显示，那么 Surface 是什么时候创建的呢？

这就要回到前面所说的 ViewRootImpl 的 requestLayout 方法了，这个方法首先会 checkThread 检查是否是主线程，然后调用 scheduleTraversals 方法，这个方法首先会设置同步屏障，然后通过 Choreographer 在下一帧到来时去执行 doTraversal 方法。在 doTraversal 中调用 performTraversal 中真正进行 View 的绘制流程，即调用 performMeasure、performLayout、performDraw。不过在它们之前，会先调用 relayoutWindow 通过 WindowSession 与 WMS 进行交互，即把 Java 层创建的 Surface 与 Native 层的 Surface 关联起来。

接下来就是正式绘制 View 了，从 performTraversals 开始，Measure、Layout、Draw 三步走。

第一步是获取 DecorView 的宽高的 MeasureSpec 然后执行 performMeasure 流程。MeasureSpec 简单来说就是一个 int 值，高 2 位表示测量模式，低 30 位用来表示大小。测量模式有三种，EXACTLY、AT_MOST、UNSPECIFIED。EXACTLY 对应为 match_parent 和具体数值的情况，表示父容器已经确定 View 的大小；AT_MOST 对应 wrap_content，表示父容器规定 View 最大只能是 SpecSize；UNSPECIFIED 表示不限定测量模式，父容器不对 View 做任何限制，这种适用于系统内部。接着说，performMeasure 中会去调用 DecorView 的 measure 方法，这个是 View 里面的方法并且是 final 的，它里面会把参数透传给 onMeasure 方法，这个方法是可以重写的，也就是我们可以干预 View 的测量过程。在 onMeasure 中，会通过 getDefaultSize 获取到宽高的默认值，然后调用 setMeasureDimension 将获取的值进行设置。在 getDefaultSize 中，无论是 EXACTLY 还是 AT_MOST，都会返回 MeasureSpec 中的大小，这个 SpecSize 就是测量后的最终结果。至于 UNSPECIFIED 的情况，则会返回一个建议的最小值，这个值和子元素设置的最小值以及它的背景大小有关。从这个默认实现来看，如果我们自定义一个 View 不重写它的 onMeasure 方法，那么 warp_content 和 match_parent 一样。所以 DecorView 重写了 onMeasure 函数，它本身是一个 FrameLayout，所以最后也会调用到 FrameLayout 的 onMeasure 函数，作为一个 ViewGroup，都会遍历子 View 并调用子 View 的 measure 方法。这样便实现了层层递归调用到了每个子 View 的 onMeasure 方法进行测量。

第二步是执行 performLayout 的流程，也就是调用到 DecorView 的 layout 方法，也就是 View 里面的方法，如果 View 大小发生变化，则会回调 onSizeChanged 方法，如果 View 状态发生变化，则会回调 onLayout 方法，这个方法在 View 中是空实现，因此需要看 DecorView 的父容器 FrameLayout 的 onLayout 方法，这个方法就是遍历子 View 调用其 layout 方法进行布局，子 View 的 layout 方法被调用的时候，它的 onLayout 方法又会被调用，这样就布局完了所有的 View。

第三步就是 performDraw 方法了，里面会调用 drawSoftware 方法，这个方法需要先通过 mSurface lockCanvas 获取一个 Canvas 对象，作为参数传给 DecorView 的 draw 方法。这个方法调用的是 View 的 draw 方法，先绘制 View 背景，然后绘制 View 的内容，如果有子 View 则会调用子 View 的 draw 方法，层层递归调用，最终完成绘制。

完成这三步之后，会在 ActivityThread 的 handleResumeActivity 最后调用 Activity 的 makeVisible，这个方法就是将 DecorView 设置为可见状态。

#### View 事件分发

在最新的 Android 系统中，事件的处理者不再由 InputEventReceiver 独自承担，而是通过多种形式的 InputStage 来分别处理，它们都有一个回调接口 onProcess 函数，这些都声明在 ViewRootImpl 内部类里面，并且在 setView 里面进行注册，比如有 ViewPreImeInputStage 用于分发 KeyEvent，这里我们重点关注与 MotionEvent 事件分发相关的 ViewPostImeInputStage。在它的 onProcess 函数中，如果判断事件类型是 SOURCE_CLASS_POINTER，即触摸屏的 MotionEvent 事件，就会调用 mView 的 dispatchPointerEvent 方法处理。也就是说事件分发最开始是传递给 DecorView 的，DecorView 的 dispatchTouchEvent 是传给 Window Callback 接口方法 dispatchTouchEvent，而 Activity 实现了 Window Callback 接口，在 Activity 的 dispatchTouchEvent 方法里，是调到 Window 的 dispatchTouchEvent，Window 的唯一实现类 PhoneWindow 又会把这个事件回传给 DecorView，DecorView 在它的 superDispatchTouchEvent 把事件转交给了 ViewGroup。

所以，事件分发的流程是：

```xml
DecorView -> Activity -> PhoneWindow -> DecorView -> ViewGroup -> View
```

这里面涉及了三个元素，Activity、ViewGroup 和 View。Activity 的 dispatchTouchEvent 前面说过，它的 dispatchTouchEvent 一般都是返回 false 不消费往下传；在说 View 的 dispatchTouchEvent，如果注册了 OnTouchListener 就调用其 onTouch 方法，如果 onTouch 返回 false 还会接着调用 onTouchEvent 函数，onTouchEvent 作为一种兜底方案，它在内部会根据 MotionEvent 的不同类型做相应处理，比如是 ACTION_UP 就需要执行 performClick 函数。ViewGroup 因为涉及对子 View 的处理，其派发流程没有 View 那么简单直接，它重写了 dispatchTouchEvent 方法，如果 ViewGroup 允许拦截，就调用其 onInterceptTouchEvent 来判断是否要真正执行拦截了，如果拦截了就交由自己的 onTouchEvent 处理，如果不拦截，就从后遍历子 View 处理，它有两个函数可以过滤子 View，一个是判断这个子 View 是否接受 Pointer Events 事件，另一个是判断落点有没有落在子 View 范围内。如果都满足，则调用其 dispatchTouchEvent 处理。如果该子 View 是一个 ViewGroup 就继续调用其 dispatchTouchEvent，否则就是 View 的 dispatchTouchEvent 方法，如此循环往复，直到事件真正被处理。

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

到这里基本上事件分发就讲完了，但是还可以扩展一下，事件到底是从哪里来的？

这就要涉及 ImputManagerService 相关的知识了。IMS 的创建过程和 WMS 类似，都是由 SystemServer 统一启动，在创建时 IMS 会把自己的实例传给 WMS，也就是表明 WMS 是 InputEvent 的派发者，这样的设计是自然而然的，因为 WMS 记录了当前系统中所有窗口的完整状态信息，所以也只有它才能判断应该把事件投递给哪一个具体的应用程序处理。

IMS 在 Native 层创建了两个线程，InputReaderThread 和 InputDispatcherThread，前者负责从驱动节点中读取 Event，后者专责与分发事件。InputReaderThread 中的实现核心是 InputReader 类，但在 InputReader 实际上并不直接去访问设备节点，而是通过 EventHub 来完成这一工作。EventHub 通过读取 /dev/input 下的相关文件来判断是否有新事件，并通知 InputReader。InputDispatcherThread 在创建之初，就把自己的实例传给了 InputReaderThread，这样便可以源源不断的获取事件进行分发了。

分发时是如何找到投递目标呢？也就是 findFocusedWindowTargetsLocked 方法的实现。也就是通过 InputMonitor 来找到 “最前端” 的窗口即可。这个 InputMonitor 是 WMS 提供的，而 IMS 实现了 WindowManagerCallbacks 接口，并把 InputMonitor 作为参数传递进去。在获知 InputTarget 之后，InputDispatcher 就需要和窗口建立连接，是通过 InputChannel，这也是一个跨进程通信，但是并不是采用 Binder，而是 Unix Domain Socket 实现。

在 Java 层，InputEventReceiver 对 InputChannel 进行包装，它是一个抽象类，它唯一的实现 WindowInputEventReceiver 就是在 ViewRootImpl 中，这样 ViewRootImpl 就可以获取到事件了。在 ViewRootImpl 中，一旦获知 InputEvent，就会进行入队操作，如果是紧急事件就直接调用 doProcessInputEvent 处理，如果不是紧急事件，就会把这个 InputEvent 推送到消息队列，然后按顺序处理，此时需要注意 Message 为异步消息。

至此，事件就流向了 ViewRootImpl 了。

#### View 刷新机制

当我们调用 View 的 invalidate 时刷新视图时，它会调到 ViewRootImp 的 invalidateChildInParent，这个方法首先会 checkThread 检查是否是主线程，然后调用其 scheduleTraversals 方法。这个方法就是视图绘制的开始，但是它并不是立即去执行 View 的三大流程，而是先往消息队列里面添加一个同步屏障，然后在往 Choreographer 里面注册一个 TRAVERSAL 的回调。在下一次 Vsync 信号到来时，会去执行 doTraversals 方法。

Choreographer 主要是用来接收 Vsync 信号，并且在信号到来时去处理一些回调事件。事件类型有四种，分别是 Input、Animation、Traversal、Commit。在 Vsync 信号到来时，会依次处理这些事件，前三种比较好理解，第四种 Commit 是用来执行组件的 onTrimMemory 函数的。Choreographer 是通过 FrameDisplayEventReceiver 来监听底层发出的 Vsync 信号的，然后在它的回调函数 onVsync 中去处理，首先会计算掉帧，然后就是 doCallbacks 处理上面所说的回调事件。

Vsync 信号可以理解为底层硬件的一个消息脉冲，它每 16ms 发出一次，它有两种方式发出，一种是 HWComposer 硬件产生，一种是用软件模拟，即 VsyncThread。不管使用哪种方式，都统一由 DispSyncThread 进行分发。

#### 项目总结

##### 自定义 View 

待定。

##### 事件分发

熟悉事件分发是处理嵌套滑动的基础。

在项目中，我们遇到了一个 ViewPager2 嵌套 RecyclerView 滑动过于灵敏的问题，即稍微滑动一点 RecyclerView 就会导致 ViewPager2 切换，这在使用 ViewPager 是没啥问题，但是 ViewPager2 内部使用的也是 RecyclerView + SnapHelper 做横向滑动，导致了滑动灵敏问题。又因为 ViewPager2 是 final 的，所以只能使用内部拦截法。

解决办法是：重写 ViewPager2 的根布局 FrameLayout 的 dispatchTouchEvent，判断如果 dx>dy+30，即表明是横向滑动，requestDisallowInterceptTouchEvent(false) 允许 ViewPager2 去拦截处理。详细代码如下：

```java
    int dx = 0;
    int dy = 0;

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {

        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                dx = (int) ev.getRawX();
                dy = (int) ev.getRawY();
                getParent().requestDisallowInterceptTouchEvent(true);
                break;
            case MotionEvent.ACTION_MOVE:
                if (Math.abs(dx - ev.getRawX()) > Math.abs(dy - ev.getRawY()) + 30) {
                    getParent().requestDisallowInterceptTouchEvent(false);
                } else {
                    getParent().requestDisallowInterceptTouchEvent(true);
                }
                break;
            case MotionEvent.ACTION_UP:
                dx = 0;
                dy = 0;
                getParent().requestDisallowInterceptTouchEvent(false);
                break;
            default:
                break;
        }
        return super.dispatchTouchEvent(ev);
    }
```

