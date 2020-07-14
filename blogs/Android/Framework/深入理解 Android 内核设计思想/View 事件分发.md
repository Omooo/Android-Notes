---
View 事件分发
---

#### 目录

1. View 中 TouchEvent 的投递流程
2. ViewGroup 中 TouchEvent 的投递流程

#### View 中 TouchEvent 的投递流程

View 类对整个输入事件的处理流程是：

```xml
InputChannel -> InputEventReceiver -> ViewRootImpl#deliverInputEvent -> ViewPostImeInputStage#onProcess -> View#dispatchPointerEvent -> View#dispatchTouchEvent -> onTouch -> onTouchEvent
```

在最新的 Android 系统中，事件的处理者不再是由 InputEventReceiver 独自承担，而是通过多种形式的 InputStage 来分别处理它们都有一个回调接口 onProcess 函数，这些都声明在 ViewRootImpl 的内部类里面，并且在 setView 方法里面进行注册，我们重点关注与 View 事件分发相关的 ViewPostImeInputStage。在它的 onProcess 函数中，如果判断事件类型是 SOURCE_CLASS_POINTER 即触摸屏的 MotionEvent 事件，就会调用 mView 的 dispatchPointerEvent 方法处理。

```java
public final boolean dispatchPointerEvent(MotionEvent event) {
    if (event.isTouchEvent()) {
        return dispatchTouchEvent(event);
    } else {
        return dispatchGenericMotionEvent(event);
    }
}
```

如果判断是 TouchEvent，就会调用其 dispatchTouchEvent 方法处理。

```java

```

