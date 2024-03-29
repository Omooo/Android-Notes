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
   - Native 层实现
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
            //以 Handler(){} 写法
            handleMessage(msg);
        }
    }

	private static void handleCallback(Message message) {
        message.callback.run();
    }
```

这里，可以看到分发消息的优先级问题，Message 的 callback 优先级最高，它是一个 Runnable，处理消息时直接 run 就好了；然后就是通过 Handler.Callback 写法，它是由返回值的，如果返回 true，那么在通过 Handler(){} 重写的方法就不会执行到，这种写法的 handlerMessage 的优先级也是最低的。

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

   其实这两种都会调用到 sendMessageAtTime 方法，只不过对于 postXxx 方法，它会把传入的 Runnable 参数赋值给 Message 的 callback 成员变量。当 Handler 进行分发消息时，msg.callback 会优先执行。

1. 子线程中怎么使用 Handler？

   子线程中使用 Handler 需要先执行两个操作：Looper.prepare 和 Looper.loop。前者会往 ThreadLocal 里面创建一个 Looper 对象，这样就避免了在 Handler 中通过 Looper.myLooper 获取 Looper 为空导致了抛异常，Looper.loop 就是开始消息循环机制了。这里一般会引申到一个问题，就是主线程中为什么不用手动调用这两个方法呢？原因是在 ActivityThread.main 方法中已经进行了调用。

2. Message 的插入以及回收是如何进行的，如何实例化一个 Message 呢？

   Message 往 MessageQueue 插入消息时，会根据 when 字段来判断插入的顺序，在消息执行完成之后，会进行回收消息，回收消息不过是把 Message 的成员变量都赋零值。实例化 Message 的时候，尽量使用 Message.obtain 方法，因为该方法是从缓存的消息池里面取得消息，可以避免 Message 的重复创建。

4. MessageQueue 中如何等待消息？

   这一步的实现是通过 MessageQueue 的 nativePollOnce 函数实现的，该方法的第二个参数就是休眠时间。在 native 侧，最终是使用了 epoll_wait 来进行等待的。

   这里为啥不使用 Java 中的 wait/notify 来实现阻塞等待呢？其实在 Android 2.2 及其以前，也确实是这样做的，后面需要处理 native 侧的事件，所以只使用 Java 的 wait/notify 就不够用了。

5. 线程、Handler、Looper、MessageQueue 的关系？

   这里的关系是，一个线程对应一个 Looper 对应一个 MessageQueue 对应多个 Handler。

6. 多个线程给 MessageQueue 发消息，如何保证线程安全？

   既然一个线程对应一个 MessageQueue，那么直接加锁就可以了：

   ```java
   // MessageQueue.java
   boolean enqueueMessage(Message msg, long when) {
       synchronized (this) {
           // ...
       }
   }
   ```

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

5. 你知道延时消息的原理嘛？

   发送的消息 Message 有一个属性 when 用来记录这个消息需要处理的时间，when 的值：普通消息的处理时间是当前时间；而延时消息的处理时间是当前时间 + delay 的时间。Message 会按 when 递增插入到 MessageQueue，也就是越早时间的排在越前面。

   在取消息处理时，如果时间还没到，就休眠到指定时间；如果当前时间已经到了，就返回这个消息交给 Handler 去分发，这样就实现处理延时消息了。休眠具体时间是在 MessageQueue 的 nativePollOnce 函数中处理，该函数的第二个参数就是指定需要休眠多少时间。

6. 主线程的 Looper#loop() 在死循环，会很消耗资源嘛？

   如果没有消息时，就会调用 MessageQueue 的 nativePollOnce 方法让线程进入休眠，当消息队列没有消息时，无限休眠；当队列的第一个消息还没到需要处理的时间时，则休眠时间为 Message.when - 当前时间。这样在空闲的时候主线程也不会消耗额外的资源了。而当有新消息入队时，enqueueMessage 里会判读是否需要通过 nativeWake 方法唤醒主线程来处理新消息。唤醒最终是通过往 EventFd 发起一个写操作，这样主线程就会收到一个可读事件进而从休眠状态被唤醒。

7. 你知道 IdleHandler 嘛？

   IdleHandler 是通过 MessageQueue.addIdleHandler 来添加到 MessageQueue 的，前面提到当 MessageQueue.next 当前没有需要处理的消息时就会进入休眠，而在进入休眠之前呢，就会调用 IdleHandler 接口里的 boolean queueIdle 方法。这个方法的返回 true 则调用后保留，下次队列空闲时还会继续调用；而如果返回 false 调用完就被 remove 了。可以用到做延时加载，而且是在空闲时加载。
   
7. View.post 和 Handler.post 的区别？

   ```java
   // View.java
   public boolean post(Runnable action) {
       final AttachInfo attachInfo = mAttachInfo;
       if (attachInfo != null) {
           return attachInfo.mHandler.post(action);
       }
   
       getRunQueue().post(action);
       return true;
   }
   ```
   
   区别就是：
   
   1. 如果在 performTraversals 前调用 View.post，则会将消息进行保存，之后在 dispatchAttachedToWindow 的时候通过 ViewRootImpl 中的 ViewRootHandler 进行调用。
   2. 如果在 performTraversals 以后调用 View.post，则直接通过 ViewRootImpl 的 Handler 进行调用。
   
   这里就可以回答一个问题了，为什么 View.post 里就可以拿到 View 的宽高信息了呢？因为 View.post 的 Runnable 执行的时候，已经执行过 performTraversals 了，也就是 View 的三大流程都执行过了，自然就可以获取到 View 的宽高信息了。
   
13. 非 UI 线程真的不能操作 View 嘛？

    ```java
    // ViewRootImpl.java
    public ViewRootImpl(Context context, Display display) {
        mThread = Thread.currentThread();
    }
    
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
    
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
    ```

    可以看到，这里的检查并不是检查的主线程，而是检查当前线程是不是 ViewRootImpl 创建的线程，因为 ViewRootImpl 是在主线程创建的，所以在非主线程操作 UI 在这里检查会失败。但是我们可以在该检查之前在非主线程里面操作 UI，比如在 Activity 的 onCreate、onResume 里面新建子线程去操作 UI，因为这时候还不会触发 requestLayout 检查是否是主线程。

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

#### Native 层实现

首先需要清楚，在 Java 层，在 Looper 的构造方法里面初始化了一个 MessageQueue：

```java
public final class MessageQueue {
	private long mPtr;
	
	MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
}
```

通过 nativeInit 关联了 Native 层的 MessageQueue，在 Native 层的 MessageQueue 中创建了 Native 层的 Looper。

```c++
Looper::Looper(bool allowNonCallbacks) {
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    rebuildEpollLocked();
}
void Looper::rebuildEpollLocked() {
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeEventFd;
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);
}
```

首先调用 epoll_create 来创建一个 epoll 实例，并且将它保存在 mEpollFd 中，然后将前面所创建的管道的读端描述符添加到这个 epoll 实例中，以便可以对它所描述的管道的写操作进行监听。

Linux 系统的 epoll 机制是为了同时监听多个文件描述符的 IO 读写事件而设计的，它是一个多路复用 IO 接口，类似于 Linux 系统的 select 机制，但是它是 select 机制的增强版。如果一个 epoll 实例监听了大量的文件描述符的 IO 读写事件，但是只有少量的文件描述符是活跃的，那么这个 epoll 实例可以显著减少 CPU 的使用率，从而提高系统的并发处理能力。

可是前面所创建的 epoll 实例只监听了一个文件描述符的 IO 写事件，这值得使用 epoll 机制来实现嘛？其实，以后我们还可以调用 C++ 层的 Looper 类的成员函数 addFd 向这个 epoll 实例中注册更多的文件描述符，以便可以监听它们的 IO 读写事件，这样就可以充分利用一个线程的消息循环来做其他事情了。在后面分析 Android 应用程序的键盘消息处理机制时，我们就会看到 C++ 层的 Looper 类的成员函数 addFd 的使用场景。

在看 MessageQueue 的 next 方法：

```java
public class MessageQueue {
	final Message next() {
		for(;;){
			nativePollOnce(mPtr, nextPollTimeoutMillis);
		}
	}
}
```

nativePollOnce 是用来检查当前线程的消息队列中是否有新的消息需要处理，nextPollTimeoutMills 用来描述当消息队列没有新的消息需要处理时，当前线程需要进去睡眠等待状态的时间。

```c++
// Native 层的 Looper
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
	int result = 0;
	for(;;){
		if(result != 0){
			return result;
		}
		result = pollInner(timeoutMillis);
	}
}

int Looper::pollInner(int timeoutMillis) {
	int result = POLL_WAKE;
	struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
        for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            }
        }
    }
}
```

如果监听的文件描述符没有发生 IO 读写事件，那么当前线程就会在 epoll_wait 中进入睡眠等待状态，等待的时间由最后一个参数 timeoutMillis 来指定。

从函数 epoll_wait 返回来之后，接下来第 11 行到第 21 行的 for 循环就检查是哪一个文件描述符发生了 IO 读写事件，如果是 mWakeReadPipeFd 并且发生的 IO 读写事件的类型是 EPOLLIN，就说明其他线程向当前线程所关联的一个管道写入了新的数据。

当其他线程向当前线程的消息队列发送一个消息之后，它们就会向与当前线程所关联的一个管道写入一个新的数据，目的就是将当前线程唤醒，以便它可以及时的去处理刚刚发送到它的消息队列的消息。

在分析 Java 层的 MessageQueue#enqueueMessage 时，我们知道，一个线程讲一个消息插入到一个消息队列之后，可能需要将目标线程唤醒，这需要分两种情况来讨论：

1. 插入的消息在目标队列中间
2. 插入的消息在目标队列头部

第一种情况下，由于保存在目标消息队列头部的消息没有发生变化，因此当前线程无论如何都不需要对目标线程执行唤醒操作。

第二种情况，由于保存在目标消息队列头部的消息发生了变化，因此，当前线程就需要将目标线程唤醒，以便它可以对保存在目标消息队列头部的新消息进行处理。但是如果这时目标线程不是正处于睡眠等待状态，那么当前线程就不需要对它进行唤醒，当前线程是否处于睡眠等待状态由 mBlocked 来记录。

```c++
// Native 层 MessageQueue#wake
void NativeMessageQueue::wake() {
	mLooper->wake();
}

void Looper::wake() {
    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
}
```

也就是调用 write 函数写一个 1，这时候目标线程就会因为这个管道发生了一个 IO 写事件而被唤醒。

消息的处理过程就简单了：当一个线程没有新的消息需要处理时，它就会在 C++ 层的 Looper 类的成员函数 pollInner 中进入睡眠等待状态，因此，当这个线程有新的消息需要处理时，它首先会在 C++ 层的 Looper 类的成员函数 pollInnter 中被唤醒，然后沿着之前的调用路径一直返回到 Java 层的 Looper 类的静态成员函数 loop 中，最后就可以对新的消息进行处理了。

Native 层分析完毕，总的来说就是利用 Linux 的 epoll 机制。

#### 参考

Android SDK 26 源码。

[Handler 都没搞懂，拿什么去跳槽啊？](<https://juejin.im/post/5c74b64a6fb9a049be5e22fc>)

[Android消息机制，你真的了解Handler吗？](https://mp.weixin.qq.com/s/JSrMjvBVBYeq6iBOWTGUng)

[美团面试官带你学 Android - Handler 这些知识点你都知道吗](https://github.com/5A59/android-training/blob/master/interview/%E9%9D%A2%E8%AF%95%E5%AE%98%E5%B8%A6%E4%BD%A0%E5%AD%A6%E5%AE%89%E5%8D%93-Handler%E8%BF%99%E4%BA%9B%E7%9F%A5%E8%AF%86%E7%82%B9%E4%BD%A0%E9%83%BD%E7%9F%A5%E9%81%93%E5%90%97.md)