---
Handler 消息机制
---

#### 目录

1. 思维导图
2. 概述
3. 基本使用
4. 源码分析
   - Handler
   - Message
   - Looper
   - MessageQueue
5. 常见问题汇总
6. 总结
7. 更新细节
   - 设置同步分割栏
8. 参考

#### 思维导图

![](https://i.loli.net/2019/02/13/5c63b4e092887.png)

#### 概述

Handler 的源码分析文章网上太多了，但是都是掌握一个大概流程，并没有深入细节。最开始我也是看一些文章掌握了一个大概流程，然后了解了 Handler、Message、Looper、MessageQueue、ThreadLocal 的作用以及联系，现在再自己不看任何资料从头再看一遍，把一些细节都理清楚。比如 Handler 的 sendXxx 和 postXxx 两者的区别、不同的 Handler 的构造方法的 handleMessage 的优先级等等，Message 的排序、回收等。

回到正文：

Android 应用是通过消息驱动运行的，在 Android 中一切皆消息，包括触摸事件，视图的绘制、显示和刷新等等都是消息。Handler 是消息机制的上层接口，平时开发中我们只会接触到 Handler 和 Message，内部还有 MessageQueue 和 Looper 两大助手共同实现消息循环系统。

#### 基本使用

基本使用不用多说，有以下两种，但是都存在内存泄漏的情况，如何避免呢？可以通过静态内部类 + 弱引用来避免，文末常见问题汇总会有示例。

```java
    private Handler mHandler = new Handler(new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            if (msg.arg1 == 0x01) {
                Toast.makeText(HandlerActivity.this, "处理消息", Toast.LENGTH_SHORT).show();
            }
            return false;
        }
    });

    @SuppressLint("HandlerLeak")
    private Handler mHandler1 = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };
```

#### Handler 源码分析

Handler 源码其实并不多，我们先来看一下构造方法：

```java
    //公开的四种构造函数
	public Handler() {
        this(null, false);
    }
    public Handler(Callback callback) {
        this(callback, false);
    }
    public Handler(Looper looper) {
        this(looper, null, false);
    }
    public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);
    }
    
	//实际上以上前两种都会调用以下构造方法
	//后两种构造方法其实会在使用 HandlerThread 的时候会用到，就不多阐述了
	public Handler(Callback callback, boolean async) {
		//初始化 Looper 并关联 MessageQueue 对象
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

这里可以看到，如果 mLooper 为 null，就是我们常见的运行时异常，提示我们是否忘记调用 Looper.prepare()。

那 Looper.myLooper 到底做了什么呢？

```java
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

    public static void prepare() {
        prepare(true);
    }
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            //一个线程只能有一个 Looper
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

可以看到 Looper.myLooper() 方法就是从 ThreadLocal 中取 Looper，而 Looper.prepare() 就是往 ThreadLocal 中存 Looper。ThreadLocal 是用于线程隔离的，它可以在不同的线程中互不干扰的存储数据，当某些数据是以线程为作用域并且不同线程具有不同的数据副本时就可以采用 ThreadLocal，下文中会分析 ThreadLocal 的源码。

回到上面，我们知道在主线程是可以直接使用 Handler 的，可是我们并没有手动调用 Looper.prepare() 方法呀，为什么没有报错呢？原因在于 ActivityThread.main 方法已经帮我们做了，所以主线程使用 Handler 时不需要手动调用 Looper.prepare() 方法，而在子线程使用 Handler 时就需要手动调用了。

ActivityThread.main 方法源码如下：

```java
    //ActivityThread 类中的 main 方法
	public static void main(String[] args) {
		//...
        Looper.prepareMainLooper();
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
	//Looper 类中的 prepareMainLooper 方法
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

综上，Handler 的构造方法就是初始化 Looper，并通过 Looper 关联 MessageQueue 对象。

熟悉了 Handler 的构造方法，然后就是 Handler 的发送消息的各种方法，发生消息可以分为两种：

1. post、postDelayed
2. sendMessage、sendMessageDelayed、sendEmptyMessage 等

```java
    //第一种方式
	public final boolean post(Runnable r){
    	return  sendMessageDelayed(getPostMessage(r), 0);
    }
    public final boolean postDelayed(Runnable r, long delayMillis){
    	return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
    public final boolean sendMessageDelayed(Message msg, long delayMillis){
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
	//第二种方式
    public final boolean sendMessage(Message msg){
        return sendMessageDelayed(msg, 0);
    }
    public final boolean sendEmptyMessage(int what){
        return sendEmptyMessageDelayed(what, 0);
    }
    public final boolean sendMessageDelayed(Message msg, long delayMillis){
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

可以看到，这两种方式其实最后都是调用 sendMessageAtTime 方法：

```java
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        //转到 MessageQueue 的 enqueueMessage 方法
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

这里需要注意的是，在调用 postDelayed 方法时传入的延迟时间需要再加上 SystemClock.uptimeMillis() 再往下传，而 SystemClock.uptimeMillis() 表示的是系统开机到当前的时间总数，单位是毫秒，并且当系统进入休眠时时间就会停止，但是不受时钟缩放或者其他节能机制的影响。这个时间总和是用来给 Message 排序的，下面会讲到。至于为什么是从开机到当前时间呢？这个也很好理解，毕竟只有开机才会有消息的分发；那为什么不用系统时间呢（System.currentTimeMilis），因为这个可能因为用户调整了系统时间而改变，该值并不可靠。

这里需要注意的是第一种方式，post 一个 Runnable，它不像第二种方式直接传 Message 对象，所以我们可以看 getPostMessage 到底干了啥？

```java
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```

OK，很清晰。只是把我们传入的 Runnable 赋值给了 Message 的 callback 成员变量。同时，这里实例化 Message 的时候是通过 Message.obtain 的方式，我们知道直接 new Message() 也是可行的，那么这两种有什么区别嘛？分析 Message 的时候再说～

这里，为什么我要单独提一下 Message 的 callback 成员变量呢？其实很有用的，下面会用到。

#### Message 源码分析

Message 作为消息的载体，它的源码也不是很多：

```java
public final class Message implements Parcelable {
    
   //消息的标示
    public int what;
    //系统自带的两个参数
    public int arg1;
    public int arg2;
    //处理消息的相对时间
    long when;
	
    Bundle data;
    Handler target;
    Runnable callback;
    
    Message next;	//消息池是以链表结构存储 Message
    private static Message sPool;	//消息池中的头节点
    
    //公有的构造方法，所以我们可以通过 new Message() 实例化一个消息了
    public Message() {
    }
    
    //推荐以这种方式实例化一个 Message，
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                //取出头节点返回
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    
    //回收消息
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

    void recycleUnchecked() {
        //清空消息的所有标志信息
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                //链表头插法
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
}
```

这里推荐使用 Message.obtain() 方法来实例化一个 Message，好处在于它会从消息池中取，而避免了重复创建的开销。虽然直接实例化一个 Message 其实并没有多大开销，但是我们知道 Android 是消息驱动的，这也就说明 Message 的使用量是很大的，所以当基数很大时，消息池就显得非常有必要了。

#### Looper 源码分析

前面已经分析过了 Looper.myLooper()、Looper.prepare() 、 Looper.prepareMainLooper() 方法了，剩下的源码为：

```java
public final class Looper {
    final MessageQueue mQueue;
    final Thread mThread;
    
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
    
    public void quit() {
        mQueue.quit(false);
    }

    public void quitSafely() {
        mQueue.quit(true);
    }
    
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        for (;;) {
            //从 MessageQueue 中取消息
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
			//通过 Handler 分发消息
            msg.target.dispatchMessage(msg);
            //回收消息
            msg.recycleUnchecked();
        }
    }
}
```

毫不夸张的说，消息机制的核心源码就在 Looper.loop 方法里，在这个方法里做了三件事：

1. 从 MessageQueue 中取出 Message
2. 通过 Handler 的 dispatchMessage 分发消息
3. 回收消息

它是一个死循环，不断地从 Message 里取消息，当消息为空时，根据注释可知，消息队列就停止了，也就是说明应用已经关闭了。

在死循环的第一步通过 MessageQueue.next 取消息，可能会阻塞，具体源码分析 MessageQueue 的时候再说。

第二步是通过 Handler 来分发消息，msg.target 即 Handler，看下 Handler.dispatchMessage 做了什么事：

```java
	public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            //通过 handler.postXxx 形式传入的 Runnable
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                //以 Handler(Handler.Callback) 写法
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            //以 Handler(){} 内存泄露写法
            handleMessage(msg);
        }
    }

	private static void handleCallback(Message message) {
        message.callback.run();
    }
```

这里，可以看到分发消息的优先级问题，Message 的 callback 优先级最高，它是一个 Runnable，处理消息时直接 run 就好了；然后就是通过 Handler.Callback 写法，它是由返回值的，如果返回 true，那么在通过 Handler(){} 重写的方法就不会执行到，这种内存泄露的写法的 handlerMessage 的优先级也是最低的。

#### MessageQueue 源码分析

接下来就是 MessageQueue 的源码了，不过 MessageQueue 中存在大量的 native 方法。这里只列举重要方法：

```java
public final class MessageQueue {
    //存消息，从 Handler.sendMessageAtTime 会跳到这
    boolean enqueueMessage(Message msg, long when) {
        
        synchronized (this) {
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            //当头节点为空或当前 Message 需要立即执行或当前 Message 的相对执行时间比头节点早
            //则把当前 Message 插入头节点
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                //根据 when 插入链表的合适位置
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; 
                prev.next = msg;
            }
        }
        return true;
    }
    
    //取消息
    Message next() {
        for (;;) {
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                //取出头节点 Message
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                //如果消息不为空，还要看 Message 是要立即取出还是延迟取出
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
            }
        }
    }
}
```

MessageQueue 有两个重要方法，一个是 enqueueMessage 用于存消息，存取消息需要根据它的 when 相对时间进行链表排序。另外一个是 next 方法取出消息，取消息的时候是通过 for 死循环不断取消息，也是取得头节点，不过需要注意该 Message 是否需要立即取出，如果不是，那就阻塞，等到时间到了在取出消息分发出去。

到这里，所有的源码都分析完了，撒花～

#### 常见问题汇总

1. Handler 的 sendXxx 和 postXxx 的区别？

   相信你心中已经有了答案。

2. Message 的插入以及回收是如何进行的，如何实例化一个 Message 呢？

   相信你心中也已经有了答案。

3. Looper.loop 死循环不会造成应用卡死嘛？

   如果按照 Message.next 方法的注释来解释的话，如果返回的 Message 为空，就说明消息队列已经退出了，这种情况下只能说明应用已经退出了。这也正符合我们开头所说的，Android 本身是消息驱动，所以没有消息几乎是不可能的事；如果按照源码分析，Message.next() 方法可能会阻塞是因为如果消息需要延迟处理（sendMessageDelayed等），那就需要阻塞等待时间到了才会把消息取出然后分发出去。然后这个 ANR 完全是两个概念，ANR 本质上是因为消息未得到及时处理而导致的。同时，从另外一方面来说，对于 CPU 来说，线程无非就是一段可执行的代码，执行完之后就结束了。而对于 Android 主线程来说，不可能运行一段时间之后就自己退出了，那就需要使用死循环，保证应用不会退出。这样一想，其实这样的安排还是很有道理的。

   这里，推荐一个说法：[https://www.zhihu.com/question/34652589/answer/90344494](https://www.zhihu.com/question/34652589/answer/90344494)

4. Handler 如何避免内存泄漏

   Handler 允许我们发送延时消息，如果在延时期间用户关闭了 Activity，那么该 Activity 泄漏。这是因为内部类默认持有外部类的引用。

   解决办法就是：将 Handler 定义为静态内部类的形式，在内部持有 Activity 的弱引用，并及时移除所有消息。

   ```java
   public class MainActivity extends AppCompatActivity {
   
       private MyHandler mMyHandler = new MyHandler(this);
   
       @Override
       protected void onCreate(Bundle savedInstanceState) {
           super.onCreate(savedInstanceState);
           setContentView(R.layout.activity_main);
       }
   
       private void handleMessage(Message msg) {
   
       }
   
       static class MyHandler extends Handler {
           private WeakReference<Activity> mReference;
   
           MyHandler(Activity reference) {
               mReference = new WeakReference<>(reference);
           }
   
           @Override
           public void handleMessage(Message msg) {
               MainActivity activity = (MainActivity) mReference.get();
               if (activity != null) {
                   activity.handleMessage(msg);
               }
           }
       }
   
       @Override
       protected void onDestroy() {
           mMyHandler.removeCallbacksAndMessages(null);
           super.onDestroy();
       }
   }
   ```


#### 总结

一张图说明一切：

![](https://i.loli.net/2019/02/14/5c64ece515ff8.jpg)

图片来源：

[Android消息机制1-Handler(Java层)](http://gityuan.com/2015/12/26/handler-message-framework/)

#### 更新细节

上面已经把 Java 层的说清楚了，但是 Native 细节并没有讲到。其次，还有一些 Java 层的细节没有讲到。

##### 设置同步分隔栏: MessageQueue.postSyncBarrier()

同步分割栏的原理其实很简单，本质上就是通过创建一个 target 成员为 null 的 Message 并插入到消息队列中，这样在这个特殊的 Message 之后的消息就不会被处理了，只有当这个 Message 被移除后才会继续执行之后的 Message。

最经典的实现就是 ViewRootImpl 调用 scheduleTraversals 方法进行视图更新时的使用：

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // 执行分割操作后会获取到分割令牌，使用它可以移除分割栏
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 发出一个有异步标志的Message，避免被分割
        // postCallback 里面会把 Message 设置为异步消息
        mChoreographer.postCallback(
            Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}
```

在执行 doTraversal 方法后，才会移除分割栏：

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        performTraversals();

        ...
    }
}
```

这样做的原因是，doTraversal 的操作是通过 Handler 进行处理的，然而这个消息队列却是整个主线程公用的，比如说四大组件的各个生命周期的调用，然后 doTraversal 的内容是更新 UI，这个任务无疑是最高优先级的，所以在这之前，需要确保队列中其它同步消息不会影响到它的执行。

看一下 MessageQueue.postSyncBarrier() 的实现：

```java
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;
        // 注意这里，并没有为target成员进行初始化

        Message prev = null;
        Message p = mMessages;
        // 插入到队列中
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

可以看到，设置分割栏和普通的 post Message 是一样的，不同的是 target 为空。

分割栏真正起作用的地方是在：

```java
Message next() {
    ...
    for (;;) {
        ...
        // 进行队列遍历
           Message msg = mMessages;
        if (msg != null && msg.target == null) {
            do {
                prevMsg = msg;
                msg = msg.next;
            // 如果target为NULL，将会陷入这个循环，除非是有异步标志的消息才会跳出循环
            } while (msg != null && !msg.isAsynchronous());
        }
        ...
    }
}
```



#### 参考

Android SDK 26 源码。

[Handler 都没搞懂，拿什么去跳槽啊？](<https://juejin.im/post/5c74b64a6fb9a049be5e22fc>)

[Android消息机制，你真的了解Handler吗？](https://mp.weixin.qq.com/s/JSrMjvBVBYeq6iBOWTGUng)