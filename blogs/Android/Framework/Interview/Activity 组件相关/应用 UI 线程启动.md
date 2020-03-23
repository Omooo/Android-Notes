---
应用的 UI 线程是怎么启动的？
---

1. 什么是 UI 线程？
2. UI 线程的启动流程，消息循环是怎么创建的
3. 了解 Android 的 UI 显示原理，UI 线程和 UI 之间是怎么关联的？

#### 什么是 UI 线程？

1. UI 线程就是刷新 UI 所在的线程
2. UI 是单线程刷新的

UI 线程 == 主线程？

1. Activity.runOnUiThread(Runnable)
2. View.post(Runnable)

```java
final void runOnUiThread(Runnable action) {
	if(Thread.currentThread() != mUiThread) {
        // Handler mHandler = new Handler();
		mHandler.post(action);
	} else {
		action.run();
	}
}
final void attach(Context context, ...) {
    mUiThread = Thread.currentThread();
}
```

对 Activity 来说，UI 线程就是主线程！

```java
public boolean post(Runnable action) {
	final AttachInfo attachInfo = mAttachInfo;
	if(attachInfo != null) {
		return attachInfo.mHandler.post(action);
	}
	ViewRootImpl.getRunQueue().post(action);
	return true;
}
```

对 View 来说，它的 UI 线程就是 ViewRootImpl 创建的时候所在的线程！

```java
void checkThread() {
	if(mThread != Thread.currentThread()) {
		throw new CalledFromWrongThreadException("");
	}
}
```

Activity 的 DecorView 对应的 ViewRootImpl 是在主线程创建的！

**关于 UI 线程的三个结论**

1. activity.runOnUiThread()

   对 Activity 来说，UI 线程就是主线程！

2. View.post(Runnable r)

   对 View 来说，它的 UI 线程就是 ViewRootImpl 创建的时候所在的线程！

3. checkThread()

   Activity 的 DecorView 对应的 ViewRootImpl 是在主线程创建的！

#### UI 线程的启动流程

1. Zygote fork 进程
2. 启动 binder 线程
3. 执行入口函数

```java
// ActivityThread#main

```

