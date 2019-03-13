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
2. 

#### 步骤一：初始化 PhoneWindow 和 WindowManager

在 Activity 的 onCreate、onStart、onResume 等生命周期被调用之前，attach 方法将先会被调用，因此将 attach 方法作为这篇文章主线的开头：

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



#### 参考