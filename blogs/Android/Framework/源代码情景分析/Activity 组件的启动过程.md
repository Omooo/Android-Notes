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
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{
	static public IActivityManager asInterface(IBinder obj)
    {
    	IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
        	return in;
        }
        return new ActivityManagerProxy(obj);
    }
    
    static public IActivityManager getDefault()
    {
    	if (gDefault != null) {
    		return gDefault;
    	}
    	IBinder b = ServiceManager.getService("activity");
    	gDefault = asInterface(b);
    	return gDefault;
    }
    private static IActivityManager gDefault;
}
```

第一次调用 ActivityManagerNative 类的静态成员函数 getDefault 时，它便会通过 ServiceManager 来获得一个名称为 "activity" 的 Java 服务代理对象，即获得一个引用了 ActivityManagerService 的代理对象。然后调用静态函数 asInterface 将这个代理对象封装成一个类型为 ActivityManagerProxy 的代理对象，最后将它保存在静态成员变量 gDefault 中。这样，以后再调用 ActivityManagerNative 类的静态成员函数 getDefault 时，就可以直接获得 ActivityManagerService 的一个代理对象了。

回到 Instrumentation 类的成员函数 execStartActivity 中，接下来就调用 ActivityManagerProxy 类的成员函数 startActivity 通知 AMS 启动一个 Activity 组件。

```java
class ActivityManagerProxy implements IActivityManager
{
	public int startActivity(IApplicationThread caller, Intent intent, IBinder resultTo, ...) {
		Parcel data = Parcel.obtain();
		Parcel reply = Parcel.obtain();
		data.writeInterfaceToken(IActivityManager.descriptor);
		data.writeStrongBinder(caller != null ? caller.asBinder() : null);
		intent.writeToParcel(data, 0);
		//...
		mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
		return result;
	}
}
```

首先将前面传进来的参数写入到 Parcel 对象 data 中，接着再通过 ActivityManagerProxy 类内部的一个 Binder 代理对象 mRemote 向 AMS 发送一个类型为 START_ACTIVITY_TRANSACTION 的进程间通信请求。

传递给 AMS 的参数比较多，不过我们需要重点关注的参数有三个，它们分别是 caller、intent 和 resultTo。其中，参数 caller 指向 Launcher 组件所运行在的应用程序进程的 ApplicationThread 对象；参数 intent 包含了即将要启动的 MainActivity 组件的信息；参数 resultTo 指向 ActivityManagerService 内部的一个 ActivityRecord 对象，它里面保存了 Launcher 组件的详细信息。

以上都是在应用程序 Launcher 中执行的，接下来就是在 AMS 中去处理 Launcher 组件发出的类型为 START_ACTIVITY_TRANSACTION 的进程间通信请求。

```java
public final class ActivityManagerService extends ActivityManagerNative
{
	public final int startActivity(IApplicationThread caller, Intent intent, IBinder resultTo, ...){
		return mMainStack.startActivity(...);
	}
}
```

AMS 类的成员函数 startActivity 用来处理类型为 START_ACTIVITY_TRANSACTION 的进程间通信请求。

AMS 类有一个类型为 ActivityStack 的成员变量 mMainStack，用来描述一个 Activity 组件堆栈，然后调用它的 startActivityMayWait 来进一步处理。

```java
public class ActivityStack {
    final int startActivityMayWait(IApplicationThread caller, Intent intent, IBinder resultTo, ...) {
    	ActivityInfo aInfo;
        ResolveInfo rInfo = AppGlobals.getPackageManager().resolveIntent(intent, ...);
        aInfo = rInfo != null ? rInfo.activityInfo : null;
        int res = startActivityLocked(caller, intent, aInfo, ...);
        return res;
    }
}
```

首先会到 PKMS 中去解析参数 intent 的内容，以便可以获得即将启动的 Activity 组件的更多信息；接着调用 startActivityLocked 来继续执行启动 Activity 组件的工作。

```java
public class ActivityStack {
    final int startActivityLocked(IApplicationThread caller, Intent intent, ...) {
        ProcessRecord callerApp = null;
        if (caller != null) {
            callerApp = mService.getRecordForAppLocked(caller);
            if (callerApp != null) {
                callingPid = callerApp.pid;
                callingUid = callerApp.info.uid;
            }
        }
        ActivityRecord r = new ActivityRecord(mService, ...);
        return startActivityUncheckedLocked(r, ...);
    }
}
```

在 AMS 中，每一个应用程序进程都使用一个 ProcessRecord 对象来描述，并且保存在 AMS 内部。ActivityStack 类的成员变量 mService 指向了 ASM，通过调用它的成员函数 getRecordForAppLocked 来获得与参数 caller 对应的一个 ProcessRecord 对象 callerApp。前面提到，参数 caller 指向的是 Launcher 组件所运行在的应用程序进程的一个 ApplicationThread 对象，因此，得到的 ProcessRecord 对象 callerApp 实际上就指向了 Launcher 组件所运行在的应用程序进程。

ActivityStack 类的成员变量 mHistory 用来描述系统的 Activity 组件堆栈。在这个堆栈中，每一个已经启动的 Activity 组件都使用一个 ActivityRecord 对象来描述。然后创建一个 ActivityRecord 对象 r 来描述即将启动的 Activity 组件，即 MainActivity 组件。

现在，ActivityStack 类就得到了请求 AMS 执行启动 Activity 组件操作的源 Activity 组件，以及要启动的目标 Activity 组件的信息了，它们分别保存在 ActivityRecord 对象 sourceRecord 和 r 中。最后调用 startActivityUncheckedLocked 进一步执行启动目标 Activity 组件的操作。

