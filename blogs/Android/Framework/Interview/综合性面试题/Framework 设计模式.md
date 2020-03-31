---
Framework 用到了哪些设计模式？
---

1. 是否熟悉常用的设计模式
2. 是否对 Framework 源码有所了解
3. 是否善于学习和总结

#### 单例模式

```java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>(){
    protected IActivityManager create(){
        IBinder b = ServiceManager.getService("activity");
        IActivityManager am = asInterface(b);
        return am;
    }
}
static public IActivityManager getDefault(){
    return gDefault.get();
}
```

```java
// 线程间单例
private static final ThreadLocal<Choreographer> sThreadInstance = new ThreadLocal<Choreographer>(){
    @Override
    protected Choreographer initialValue(){
        Looper looper = Looper.myLooper();
        return new Choreographer(looper);
    }
}
public static Choreographer getInstance(){
    return sThreadInstance.get();
}
```

```c++
// 进程间单例
// 单例：ServiceManager 中间人：Binder 驱动
int main(int argc, char **argv){
	struct binder_state *bs;
    bs = binder_open(128*1024);
    binder_become_context_manager(bs);
}
private static IServiceManager getIServiceManager(){
    if(sServiceManager != null){
        return sServiceManager;
    }
    sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
    return sServiceManager;
}
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>&){
    return getStrongProxyForHandle(0);
}
```

#### 观察者模式

```java
public class ContentObservable extends Observable<ContextObserver>{
	@OVerride
	public void registerObserver(ContentObserver observer){
		super.registerObserver(observer)l
	}
	public void notifyChange(boolean selfChange){
		synchronized(mObservers){
            for(ContentObserver observer: mObservers){
                observer.onChange(selfChange, null);
            }
        }
	}
}
void registerContentObserver(..., ContentObserver obsserver, ...){
    getContentService().registerContentObserver(..., observer.getContentObserver(), ...);
}
public IContentOnserver getContentObserver(){
    synchronized(mLock){
        if(mTransport == null){
            mTransport = new Transport(this);
        }
        return mTransport;
    }
}
private static final class Transport extends IContentObserver.Stub{
    private ContentObserver mContentObserver;
    public Transport(ContentObserver contentObserver){
        mContentObserver = contentObserver;
    }
    @Override
    public void onChange(boolean selfChange, Uri uri, int userId){
        ContentObserver contentObserver = mContentObserver;
        if(contentObserver!=null){
            contentObserver.dispatchChange(selfChange, uri, userId);
        }
    }
}
```

#### 代理模式

静态代理：

```java
class ActivityManagerProxy implements IActivityManager{
	private IBinder mRemote;
    public ActivityManagerProxy(IBinder remote){
        mRemote. = remote;
    }
}
class ActivityManagerNative extends Binder implements IActivityManager{
    static public IActivityManager asInterface(IBinder obj){
        if(obj == null){
            return null;
        }
        IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
        if(in != null){
            return in;
        }
        return new ActivityManagerProxy(obj);
    }
}

int startActivity(IApplicationThread caller, ...) throws RemoteException{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    
    mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
}
```

动态代理：

```java
public class Decorator<T> implements InvocationHandler{
    private final T mObject;
    private final DecoratorListener mListener;
    
    public static <T> T newInstance(T obj, DecoratorListener listener){
        return (T)Proxy.newProxyInstance(obj.getClass().getClassLoader(),
                                        obj.getClass().getInterfaces(),new Decorator<T>(obj, listener));
    }
    
    private Decorator(T obj, DecoratorListener listener){
        this.mObject = obj;
        this.mListener = listener;
    }
}
```

