---
Activity 组件的启动过程
---

#### 根 Activity 组件的启动过程

当我们在应用程序启动器 Launcher 界面上点击一应用程序的快捷图标时，Launcher 组件的成员函数 startActivitySafely 就会被调用来启动这个应用程序根 Activity，其中，要启动的根 Activity 的信息包含在参数 intent 中。

```java
public final class Launcher extends Activity {
	void startActivitySafely(Intent intent, Object tag) {
		intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
	}
}
```

Launcher 组件是如何获得这些信息的呢？系统在启动时，会启动一个 PackageManagerService，并且通过它来安装系统中的应用程序。PackageManagerService 在安装一个应用程序的过程中，会对它的配置文件 AndroidManifest.xml 进行解析，从而得到它里面的组件信息。系统在启动完成之后，就会将 Launcher 组件启动起来。Launcher 组件在启动过程中，会向 PackageManagerService 查询所有 Action 名称等于 "Intent.ACTION_MAIN"，并且 Category 名称等于 “Intent.CATEGORY_LAYNCHER” 的 Activity 组件，最后为每一个 Activity 组件创建一个快捷图标，并且将它们的信息与各自的快捷图标关联起来，以便用户点击它们时可以将对应的 Activity 组件启动起来。

回到 Launcher 组件的成员函数 startActivitySafely 中，接着会调用父类 Activity 的成员函数 startActivity：

```java
public class Activity extends ContextThemeWrapper {
	public void startActivity(Intent intent) {
		startActivityForResult(intent, -1);
	}
    
    public void startActivityForResult(Intent intent, int requestCode) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar = 
                mInstrumentation.execStartActivity(
            	this, mMainThread.getApplicationThread(), mToken,
            	this, intent, requestCode);
        } else {
            
        }
    }
}
```

Activity 类的成员变量 mInstrumentation 的类型为 Instrumentation，它用来监控应用程序和系统之间的交互操作。

Activity 类的成员变量 mMainThread 的类型为 ActivityThread，用来描述一个应用程序进程。系统每当启动一个应用程序进程时，都会在它里面夹在一个 ActivityThread 类实例，并且会将这个 ActivityThread 类实例保存在每一个在该进程中启动的 Activity 组件的父类 Activity 的成员变量 mMainThread 中。ActivityThread 类的成员函数 getApplicationThread 用来获取它内部的一个类型为 ApplicationThread 的 Binder 本地对象。在 Instrumentation 的 execStartActivity 时会作为参数传递给 ActivityManagerService，这样 AMS 接下来就可以通过它来通知 Launcher 组件进入 Paused 状态了。

Activity 类的成员变量 mToken 的类型为 Binder，它是一个 Binder 代理对象，指向了 AMS 中一个类型为 ActivityRecord 的 Binder 本地对象。每一个已经启动的 Activity 组件在 AMS 中都有一个对应的 ActivityRecord 对象，用来维护对应的 Activity 组件的运行状态以及信息。同样也作为参数传递给了 AMS，这样 AMS 就可以获得 Launcher 组件的详细信息了。

接下来继续看 Instrumentation#execStartActivity 的实现：

```java
public class Instrumentation {
	public ActivityResult execStartActivity(...) {
		int result = ActivityManagerNative.getDefault()
			.startActivity(...);
	}
}
```

首先调用 ActivityManagerNative 类的静态成员函数 getDefault 来获得 ActivityManagerService 的一个代理对象，接着再调用它的成员函数 startActivity 来通知 AMS 将一个 Activity 组件启动起来。

ActivityMAnagerNative#getDefault 实现如下：

```java

```

