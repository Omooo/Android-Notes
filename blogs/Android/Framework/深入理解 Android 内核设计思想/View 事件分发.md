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
// View
public boolean dispatchTouchEvent (MotionEvent event) {
   if (onFilterTouchEventForSecurity(event)) {
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
	return result;
}
```

可以看到，View 类中分发 TouchEvent 还是比较简单的，如果注册了 OnTouchListener 就调用其 onTouch 方法，如果返回 false，还会接着回调 onTouchEvent 方法。onTouchEvent 作为一种兜底方案，它在内部会根据 MotionEvent 的不同类型做相应处理，比如是 ACTION_UP，可能就需要执行 performClick 函数。

#### ViewGroup 中 TouchEvent 的投递流程

ViewGroup 因为涉及对子 View 的处理，其派发流程没有 View 那么简单直接，它重写了 dispatchTouchEvent 方法：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); 
            } else {
                intercepted = false;
            }
        } else {
            intercepted = true;
        }
        // 不拦截的情况
        if (actionMasked == MotionEvent.ACTION_DOWN
            || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                : TouchTarget.ALL_POINTER_IDS;
            final int childrenCount = mChildrenCount;
            if (newTouchTarget == null && childrenCount != 0) {
                final float x = ev.getX(actionIndex);
                final float y = ev.getY(actionIndex);
                final View[] children = mChildren;
                for (int i = childrenCount - 1; i >= 0; i--) {
                    if (!child.canReceivePointerEvents()
                        || !isTransformedTouchPointInView(x, y, child, null)) {
                        continue;
                        newTouchTarget = getTouchTarget(child);
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            break;
                        }
                    }
                }
            }
        }
    }
}
```

如果 ViewGroup 允许拦截，就调用其 onInterceptTouchEvent 来判断是否要真正执行拦截了，这样做了是方便扩展。如果不拦截，就从后遍历子 View 处理。它有两个函数可以过滤子 View，一个是判断这个子 View 能否接收 Pointer Events 事件，另一个是判断落点有没有落在子 View 范围内。如果都满足，则调用其 dispatchTouchEvent 处理。如果该子 View 是一个 ViewGroup 就继续调用其 dispatchTouchEvent，否则就是 View 的 dispatchTouchEvent。如此循环往复，直到事件真正被处理。