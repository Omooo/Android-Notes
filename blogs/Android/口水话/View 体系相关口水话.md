---
View 体系相关口水话
---

#### 目录

1. View 绘制流程
2. View 事件分发
3. View 刷新机制

#### View 绘制流程

#### View 事件分发

#### View 刷新机制

当我们调用 View 的 invalidate 时刷新视图时，它会调到 ViewRootImp 的 invalidateChildInParent，这个方法首先会 checkThread 检查是否是主线程，然后调用其 scheduleTraversals 方法。这个方法就是视图绘制的开始，但是它并不是立即去执行 View 的三大流程，而是先往消息队列里面添加一个同步屏障，然后在往 Choreographer 里面注册一个 TRAVERSAL 的回调。在下一次 Vsync 信号到来时，会去执行 doTraversals 方法。

Choreographer 主要是用来接收 Vsync 信号，并且在信号到来时去处理一些回调事件。事件类型有四种，分别是 Input、Animation、Traversal、Commit。在 Vsync 信号到来时，会依次处理这些事件，前三种比较好理解，第四种 Commit 是用来执行组件的 onTrimMemory 函数的。Choreographer 是通过 FrameDisplayEventReceiver 来监听底层发出的 Vsync 信号的，然后在它的回调函数 onVsync 中去处理，首先会计算掉帧，然后就是 doCallbacks 处理上面所说的回调事件。

Vsync 信号可以理解为底层硬件的一个消息脉冲，它每 16ms 发出一次，它有两种方式发出，一种是 HWComposer 硬件产生，一种是用软件模拟，即 VsyncThread。不管使用哪种方式，都统一由 DispSyncThread 进行分发。