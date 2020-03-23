---
Activity 的显示原理
---

1. Activity 的显示原理（Window/DecorView/ViewRoot）
2. Activity 的 UI 刷新机制（Vsync/Choreographer）
3. UI 的绘制原理（Measure/Layout/Draw）
4. Surface 原理（Surface/SurfaceFlinger）

#### 显示原理

1. setContentView 原理是什么？
2. Activity 在 onResume 之后才会显示的原因是什么？
3. ViewRoot 是干嘛的，是 View Tree 的 rootView 么？

```java
public void setContentView(int layoutResID) {
	getWindow().setContentView(layoutResID);
	//...
}
```

```java
final void attach(Context context, ){
	mWindow = new PhoneWindow(this);
}
```

```java
// PhoneWindow#setContentView
public void setContentView(int layoutResID) {
    if(mContentParent == null) {
        installDecor();
    }
    mLayoutInflater.inflate(layoutResID, mContentParent);
}

private void installDecor() {
    // FrameLayout
    mDecor = new DecorView(getContext());
    // 根据 Window Feature 选择一个系统布局
    View in = mLayoutInflater.inflate(layoutResource, null);
    mDecor.addView(in, ...);
    mContentParent = findViewById(ID_ANDROID_CONTENT);
}
```

```java
final void handleResumeActivity(IBinder token) {
	ActivityClientRecord r = performResumeActivity(token, );
    final Activity a = r.activity;
    
    if(r.window == null && !a.mFinished) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        
        ViewManager wm = a.getWindowManager();
        a.mDecor = decor;
        wm.addview(decor, l);
    }
    r.activity.makeVisible();
}
```

```java
// ViewManager#addView
void addView(View view, ViewGroup.LayoutParams params, ){
	ViewRootImpl root = new ViewRootImpl(view.getContext(), );
    root.setView(view, wparams, panelParentView);
}
```

```java
// ViewRootImpl#setView
public void setView(View view, ) {
    if(mView == null) {
        mView = view;
        requestLayout();
        //...
        mWindowSession.addToDisplay(mWindow, ...);
    }
}
```

```java
public void requestLayout() {
	scheduleTraversals();
}
void scheduleTraversals() {
    mChoreographer.postCallback(..., mTraversalRunnable, null);
}
class TraversalRunnable implements Runnable {
    public void run() {
        doTraversal();
    }
}
void doTraversal() {
    performTraversals();
}
private void performTraversals() {
    //...
    relayoutWindow(params, ...);
    performMeasure(childWidthMeasureSpec, ...);
    performLayout(lp, desiredWindowWidth, ...);
    performDraw();
}
// 向 WMS 申请 Surface
int relayoutWindow(WindowManager.LayoutParams params, ...) {
    mWindowSession.relayout(..., mSurface);
}
```

```java
// static class W extends IWindow.Stub {...}
mWindowSession.addToDisplay(mWindow, ...);
public int addToDisplay(IWindow window, int seq, ...) {
    // WMS 统一管理 Window，分配 Surface
    return mService.addWindow(this, window, seq, attrs, ...);
}

IWindowManager windowManager = getWindowManagerService();
sWindowSession = windowManager.openSession(...);

IWindowSession openSession(IWindowSessionCallback callback, ...) {
    Session session = new Session(this, callback, client, inputContext);
    return session;
}
```

WMS 的主要作用：

1. 分配 Surface
2. 掌管 Surface 显示顺序以及位置尺寸等
3. 控制窗口动画
4. 输入事件分发

#### 总结

![](https://i.loli.net/2020/03/23/X2pw9F6B5SmdVRj.png)

