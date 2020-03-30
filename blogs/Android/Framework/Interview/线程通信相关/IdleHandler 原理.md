---
IdleHandler 原理
---

1. 了解 IdleHandler 的作用以及调用方式
2. 了解 IdleHandler 有哪些使用场景
3. 熟悉 IdleHandler 的实现原理

```java
public static interface IdleHandler{
	boolean queueIdle();
}
```

```java
Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler()){
	@Override
    public boolean queueIdle(){
        return true;
    }
}

public void addIdleHandler(IdleHandler handler){
    synchronized(this){
        mIdleHandler.add(handler);
    }
}
Message next(){
    for(;;){
        nativePollOnce(ptr, nextPollTimeoutMillis);
        // 看消息列表是否有消息可以分发的，如果有，就返回该消息
        // 走到这说明没有消息可以分发，下一个 for 循环就要进入休眠了
        if(pendingIdleHandlerCount<0){
            pendingIdleHandlercount = mIdleHandlers.size();
        }
        if(pendingIdleHandlerCount<0){
            continue;
        }
        mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        for(int i=0;i<pendingIdleHandlerCount;i++){
            final IdleHandler idler = mPendingIdleHandlers[i];
            if(!(idler.queueIdle())){
                mIdleHandlers.remove(idler);
            }
        }
    }
}
```

Framework 哪些地方用到了 IdleHandler 呢？

```java
// ActivityThread#scheduleGcIdler
void scheduleGcIdler(){
	if(!mGcIdlerScheduled){
        mGcIdlerScheduled = true;
        Looper.myQueue().addIdleHandler(mGcIdler);
    }
    mH.removeMessages(H.GC_WHEN_IDLE);
}
final class GcIdler implements MessageQueue.IdleHandler{
    @Override
    public final boolean queueIdle(){
        // BinderInternal.forceGc("bg");
        doGcIfNeeded();
        return false;
    }
}
// ActivityThread#waitForIdle
public void waitForIdle(Runnable recipient){
    mMessageQueue.addIdleHandler(new Idler(recipient));
    mThread.getHandler().post(new EmptyRunnable());
}
private static final class Idler implements MessageQueue.IdleHandler{
    public final boolean queueIdle(){
        if(mCallback!=null){
            mCallback.run();
        }
        return false;
    }
}
```

#### 适用场景

1. 延时处理
2. 批量任务