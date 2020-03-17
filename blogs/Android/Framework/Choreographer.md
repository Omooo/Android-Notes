---
Choreographer
---

#### 概述

Choreographer 是用来控制同步处理输入（Input）、动画（Animation）、绘制（Draw）三个操作（UI 显示的时候每一帧要完成的事情只有这三种）。其内部维护着一个 Queue，使用者可以通过 postXxx 来把一系列待运行的 UI 操作放到 Queue 中。这些事件会在 Choreographer 接收 VSYNC 信号后执行这些操作，比如 ViewRootImpl 对于 ViewTree 的更新事件：

```java
// ViewRootImpl
void scheduleTraversals() {
     ...
     mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
}
```

#### 对 Vsync 信号的监听



#### 参考

[Choreographer工作逻辑总结](https://github.com/SusionSuc/AdvancedAndroid/blob/master/Rabbit%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/Choreographer%E5%B7%A5%E4%BD%9C%E9%80%BB%E8%BE%91%E6%80%BB%E7%BB%93.md)