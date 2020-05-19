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

```java
public class ActivityStack {
    final int startActivityUncheckedLocked(ActivityRecord r, ...){
        startActivityLocked(ActivityRecord r, ...);
    }
    
    private final void startActivityLocked(ActivityRecord r, ...){
        mHistory.add(addPos, r);
        resumeTopActivityLocked(null);
    }
    
    final boolean resumeTopActivityLocked(ActivityRecord prev){
        // next 当前是 MainActivity
        ActivityRecord next = topRunningActivityLocked(null);
        // mResumedActivity 当前是 Launcher
        if(mResumedActivity != null){
            startPausingLocked();
            return true;
        }
    }
}
```

系统当前正在激活的 Activity 组件是 Launcher 组件，即 ActivityStack 类的成员变量 mResumedActivity 指向了 Launcher 组件，因此，接下来就会调用 startPausingLocked 来通知它进入 Paused 状态，它可以将焦点让给即将要启动的 MainActivity 组件。

```java
public class ActivityStack {
    private final void startPausingLocked() {
        ActivityRecord prev = mResumedActivity;
        if (prev.app != null && prev.app.thread != null) {
            prev.app.thread.schedulePauseActivity(...);
        }
    }
}
```

ActivtyRecord 类有一个成员变量 app，它的类型为 ProcessRecord，用来描述一个 Activity 组件所运行在的应用程序进程；而 ProcessRecord 类又有一个 thread 成员变量，它的类型为 ApplicationThreadProxy，用来描述一个 Binder 代理对象，引用的是一个类型为 ApplicationThread 的 Binder 本地对象。

```java
class ApplicationThreadProxy implements IApplicationThread {
    public final void schedulePauseActivity(IBinder token, ...){
        Parcel data = Parcel.obtain();
        data.writeStrongBinder(token);
        mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null, IBinder.FLAG_ONEWAY);
    }
}
```

通过 ApplicationThreadProxy 类内部的一个 Binder 代理对象 mRemote 向 Launcher 组件所在的应用程序进程发送一个类型为 SCHEDULE_PAUSE_ACTIVITY_TRANSACTION 的进程间通信请求。

这个进程间通信请求是异步的，因此，AMS 将它发送出去之后，就马上返回了。

以上都是在 AMS 中执行的，接下来就会在应用程序 Launcher 中去处理 AMS 的请求了。

```java
public final class ActivityThread {
    private final class ApplicationThread extends ApplicationThreadNative {
        public final void schedulePauseActivity(IBinder token, ...) {
            queueOrSendMessage(H.PAUSE_ACTIVITY, ...);
        }
    }
}
```

ApplicationThread 类的成员函数 schedulePauseActivity 用来处理类型为 SCHEDULE_PAUSE_ACTIVITY_TRANSACTION 的进程间通信请求。

最后调用外部类 ActivityThread 的 queueOrSendMessage 来向 Launcher 的主进程的消息队列发送一个类型为 PAUSE_ACTIVITY 的消息。

应用程序 Laucher 为什么不直接在当前线程中执行中止 Launcher 组件的操作呢？一方面是因为当前线程需要尽快返回到 Binder 线程池中去处理其他进程间通信请求；另一方面是因为在中止 Launcher 组件的过程中，可能会涉及用户页面相关的操作，因此就需要将它放在主线程中执行。

```java
public final class ActivityThread {
    private final class H extends Handler {
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case PAUSE_ACTIVITY:
                    handlePauseActivity(...);
            }
        }
    }
}

public final class ActivityThread {
    final HashMap<IBinder, ActivityClientRecord> mActivities
        = new HashMap<>();
    
    private final void handlePauseActivity(IBinder token, ...) {
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
            performPauseActivity();
            QueuedWork.waitToFinish();
            ActivityManagerNative.getDefault().activityPaused(token, state);
        }
    }
}
```

在应用程序进程中启动的每一个 Activity 组件都使用一个 ActivityClientRecord 对象来描述，这些 ActivtyClientRecord 对象对应于 ActivityManagerService 中的 ActivityRecord 对象，并且保存在 ActivityThread 类的成员变量 mActivities 中。

调用成员函数 performPauseActivity 向 Launcher 组件发送一个中止事件通知，即调用它的成员函数 onPause。然后调用 QueuedWork 的 waitToFinish 等待完成前面的一些数据写入操作，例如将数据写入磁盘的操作。由于现在 Launcher 组件即将要进入 Paused 状态了，因此就要保证它前面的所有数据写入操作都处理完成；否则，等到它重新进入 Resumed 状态时，就无法恢复之前所保存的一些状态数据。

完成这些事之后，ActivityThread 类的成员函数 handlePauseActivity 就处理完 AMS 给它发送的中止 Launcher 组件的进程间通信请求了。

最后调用 ActivityManagerNative 类的静态成员函数 getDefault 来获得 AMS 的一个代理对象，然后再调用这个代理对象的成员函数 activityPaused 来通知 AMS Launcher 组件已经进入 Paused 状态了，因此，它就可以将 MainActivity 组件启动起来了。

```java
class ActivityManagerProxy implements IActivityManager {
    public void activityPaused(IBinder token, ...) {
        Parcel data = Parcel.obtain();
        data.writeStrongBinder(token);
        mRemote.transact(ACTIVITY_PAUSED_TRANSACTION, data, ...);
    }
}
```

将前面传进来的参数写入到 Parcel 对象 data 中，然后再通过 ActivityManagerProxy 类内部的一个 Binder 代理对象 mRemote 向 AMS 发送一个类型为 ACTIVITY_PAUSED_TRANSACTION 的进程间通信请求。

以上都是在应用程序 Launcher 中执行的，接下来是在 AMS 中处理 Launcher 组件发出的进程间通信请求。

```java
public final int class ActivityManagerService extends ActivityManagerNative {
    public final void activityPaused(IBinder token, ...) {
        mMainStack.activityPaused(token, ...);
    }
}
```

ActivityManagerService 类的成员函数 activityPaused 用来处理类型为 ACTIVITY_PAUSED_TRANSACTION 的进程间通信请求。

```java
public class ActivityStack {
    final void activityPaused(IBinder token, ...) {
        ActivityRecord r = null;
        r = (ActivityRecord)mHistory.get(index);
        r.state = ActivityState.PAUSED;
       completePauseLocked();
    }
}
```

首先把 Launcher 组件对应的 ActivityRecord 对象的 state 设置为 PAUSED 状态，表示 Launcher 组件已经进入 Paused 状态了。然后调用 completePauseLocked 来执行启动 MainActivity 组件的操作。

```java
public class ActivityStack {
    private final void completePauseLocked() {
        ActivityRecord prev = mPausingActivity;
        resumeTopActivityLocked(prev);
    }
    
    final boolean resumeTopActivityLocked(ActivityRecord prev) {
        startSpecificActivityLocked(next, true, true);
    }
    
    private final void startSpecificActivityLocked(ActivityRecord r, ...) {
        ProcessRecord app = mService.getProcessRecordLocked(r.processName, ...);
        if (app != null && app.thread != null) {
            realStartActivityLocked(r, ...);
            return;
        }
        mService.startProcessLocked(r.processName, r.info.application, true, 0, "activity", r.intent.getComponent(), false);
    }
}
```

在 AMS 中，每一个 Activity 组件都有一个用户 ID 和一个进程名称；其中，用户 ID 是在安装该 Activity 组件时由 PKMS 分配的，而进程名称则是由该 Activity 组件的 android:process 属性来决定的。AMS 在启动一个 Activity 组件时，首先会以它的用户 ID 和进程名称来检查系统中是否存在一个对应的应用程序进程。如果存在，就会直接通知这个应用程序进程将该 Activity 组件启动起来；否则，就会先以这个用户 ID 和进程名称来创建一个应用程序进程，然后再通知这个应用程序进程将该 Activity 组件启动起来。

如果 ActivityRecord 对象 r 对应的 Activity 组件所需要的应用程序进程已经存在，就直接调用 realStartActivityLocked 来启动该 Activity 组件，否则，就先调用 startProcessLocked 为该 Activity 组件创建一个应用程序进程，然后再将它启动起来。

由于 MainActivity 组件是第一次被启动，所以 AMS 会先创建应用程序进程。

```java
// AMS
private final void startProcessLocked(...) {
    int pid = Process.start("android.app.ActivityThread", ...);
}
```

调用 Process 类的 start 来启动一个新的应用程序进程，指定了该进程的入口函数为 ActivityThread 的 main 函数。

```java
public final class ActivityThread {
    final ApplicationThread mAppThread = new ApplicationThread();
    
    private final void attach(boolean system) {
        IActivityManager mgr = ActivityManagerNative.getDefault();
        mgr.attachApplication(mAppThread);
    }
    
    public static final void main(String[] args) {
        Looper.prepareMainLooper();
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
        Looper.loop();
    }
}
```

新的应用程序进程在启动时，主要做了两件事情：

1. 在进程中创建一个 ActivityThread 对象，并且调用 attach 函数向 AMS 发送一个启动完成通知
2. 创建一个消息循环

