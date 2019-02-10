---
HandlerThread
---

#### 目录

1. 思维导图
2. 概述
3. 具体使用
4. 源码分析
5. 参考

#### 思维导图

#### 概述

HandlerThread 继承于 Thread，所以它本质就是个 Thread。与普通 Thread 的区别在于，它不仅建立了一个线程，并且创建了消息队列，有自己的 Looper，可以让我们在自己的线程中分发和处理消息，并对外提供自己的 Looper 的 get 方法。

HandlerThread 自带 Looper 使它可以通过消息队列来重复使用当前线程，节省系统资源开销。这是它的优点也是缺点，每个任务都将以队列的方式逐个被执行到，一旦队列中有某个任务执行时间过长，那么就会导致后续的任务都会被延迟处理。

#### 具体使用

```java
public class HandlerThreadActivity extends AppCompatActivity {

    private Button mButton;
    private HandlerThread mHandlerThread;
    private Handler mUiHandler;
    private Handler mChildHandler;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        initView();

        mHandlerThread = new HandlerThread("HandlerThread");
        mHandlerThread.start();
        mUiHandler = new Handler(new Handler.Callback() {
            @Override
            public boolean handleMessage(Message msg) {
                if (msg.what == 2) {
                    mButton.setText("子线程更新");
                }
                return false;
            }
        });
        mChildHandler = new Handler(mHandlerThread.getLooper(), new Handler.Callback() {
            @Override
            public boolean handleMessage(Message msg) {
                if (msg.what == 1) {
                    try {
                        //子线程模拟延迟处理
                        Thread.sleep(2000);
                        mUiHandler.sendEmptyMessage(2);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                return false;
            }
        });

    }

    public void initView() {
        mButton = findViewById(R.id.btn_show);
        mButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mChildHandler.sendEmptyMessage(1);
            }
        });
    }
}
```

#### 源码分析

```java
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    private @Nullable Handler mHandler;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    protected void onLooperPrepared() {
    }

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    @NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }

    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    public int getThreadId() {
        return mTid;
    }
}
```

源码很简单，就是在 run 方法中执行 Looper.prepare()、Looper.loop() 构造消息循环系统。外界可以通过 getLooper() 这个方法拿到这个 Looper。

总结为下：

1. HandlerThread 是一个自带 Looper 的线程，因此只能作为子线程使用
2. HandlerThread 必须配合 Handler 使用，HandlerThread 线程中具体做什么事，需要在 Handler 的 callback 中进行，因为它自己的 run 方法被写死了
3. 子线程的 Handler 与 HandlerThread 关系建立是通过构造子线程的Handler 传入 HandlerThread 的 Looper 。所以在此之前，必须先调用 mHandlerThread.start 让 run 方法跑起来 Looper 才能创建。

#### 参考

[Android中的线程形态（二）(HandlerThread/IntentService)](https://www.jianshu.com/p/4ca760e5040b)