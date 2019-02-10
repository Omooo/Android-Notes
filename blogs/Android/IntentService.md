---
IntentService
---

#### 目录

1. 思维导图
2. 概述
3. 具体使用
4. 源码分析
5. 参考

#### 思维导图

#### 概述

IntentService 是继承于 Service 并处理异步请求的一个类，内部实现是 HandlerThread，处理完子线程的事后自动 stopService。

#### 具体使用

首先创建：

```java
public class MyIntentService extends IntentService {

    private static final String TAG = "MyIntentService";

    public MyIntentService() {
        //IntentService 工作线程的名字
        super("MyIntentService");
    }

    @Override
    public void onCreate() {
        super.onCreate();
        Log.i(TAG, "onCreate: ");
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        Log.i(TAG, "onStartCommand: ");
        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.i(TAG, "onDestroy: ");
    }

    @Override
    protected void onHandleIntent(@Nullable Intent intent) {
        if (intent != null) {
            String taskName = intent.getStringExtra("taskName");
            Log.i(TAG, "onHandleIntent: " + taskName);
            switch (taskName) {
                case "task1":
                    //任务一
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    break;
                case "task2":
                    //任务二
                    break;
                default:
                    break;
            }
        }
    }
}
```

Activity 中：

```java
    public void onClick(View v) {
        Intent intent = new Intent(this, MyIntentService.class);
        switch (v.getId()) {
            case R.id.btn1:
                intent.putExtra("taskName", "task1");
                startService(intent);
                break;
            case R.id.btn2:
                intent.putExtra("taskName", "task2");
                startService(intent);
                break;
            default:
                break;
        }
    }
```

这里如果点击按钮一后立马点击按钮二，日志打印如下：

```
onCreate、onStartCommand、onHandleIntent: task1、
onStartCommand、onHandleIntent: task2、onDestroy
```

如果是等三秒后再点击按钮二，就是：

```
onCreate、onStartCommand、onHandleIntent: task1、onDestroy、
onCreate、onStartCommand、onHandleIntent: task2、onDestroy
```

这下就清楚了吧。

#### 源码分析

```java
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            //重写的方法，子线程需要做的事情
            onHandleIntent((Intent)msg.obj);
            //做完事，自动停止
            stopSelf(msg.arg1);
        }
    }

    public IntentService(String name) {
        super();
        //IntentService 的线程名
        mName = name;
    }

    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();
		
        //构造子线程 Handler
        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        //在 Service 启动的时候发送消息，子线程开始工作
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        //调用上面的那个方法，促使子线程开始工作
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```

总结如下：

1. Service onCreate 的时候通过 HandlerThread 构建子线程的 Handler
2. Service onStartCommand 中通过子线程 Handler 发送消息
3. 子线程 handlerMessage 中调用我们重写的 onHandlerIntent 执行异步任务，执行完之后 Service 销毁

#### 参考