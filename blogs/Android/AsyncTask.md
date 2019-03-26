---
AsyncTask
---

#### 目录

1. 前言
2. 源码分析

#### 前言

AsyncTask 是一个轻量级的异步任务类、抽象泛型类。现在基本上没人会用到它了，但是学习一下源码还是很有必要的。很早之前就阅读 AsyncTask 源码了，当时看的迷迷糊糊的，很大一部分原因是我对线程池以及 Future、Callable 不熟悉，所以在看源码之前，一定要熟悉这一块的知识，不熟悉的可以先看看 [线程、线程池](https://github.com/Omooo/Android-Notes/blob/master/blogs/Java/%E5%B9%B6%E5%8F%91/%E7%BA%BF%E7%A8%8B%E3%80%81%E7%BA%BF%E7%A8%8B%E6%B1%A0.md)。

在熟悉了 Callable、FutureTask、ArrayDeque 之后，在看 AsyncTask，简直不要太简单。

#### 源码分析

首先看一下 AsyncTask 的成员变量：

```java
    //串行的任务执行器
		public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;

    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    private static InternalHandler sHandler;

		//任务队列，是 Callable 类型
    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;
```

那 SerialExecutor 到底做了什么呢？

```java
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
          	//把一个任务插入到队列尾部
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                      	//当这个任务执行完毕后，再取任务队列的头部执行
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
          	//取任务队列的头部，然后删除
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```

所以这里就明白了，默认任务是串行执行的，越早放入的任务越早执行，同时当这个任务执行完毕之后，就会再从头部去取任务，也就达到了循环执行任务的目的了。

然后就是 AsyncTask 的构造方法：

```java
    public AsyncTask() {
        this((Looper) null);
    }
    public AsyncTask(@Nullable Looper callbackLooper) {
    		//获取主线程 Handler
    		//用于在任务执行完和任务执行过程回调函数更新进度
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //重写的后台操作方法
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```

接下来就是 AsyncTask#execute() 方法：

```java
    //调用这个方法，默认是 sDefaultExecutor，也就是上面的 SerialExecutor，即串行执行
		public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
		//当然，AsyncTask 也支持并行执行，传入 AsyncTask.THREAD_POOL_EXECUTOR
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;
				//重写的第一个方法
        onPreExecute();

        mWorker.mParams = params;
      	//mFuture 是对任务队列（mWorker）的包装，可以通过 mFuture.get() 获得执行结果
        exec.execute(mFuture);

        return this;
    }
```

我们知道，AsyncTask 的 execute 必须要在主线程中去执行，所以 onPreExecute() 也是在主线程中运行的。

再回头看看 AsyncTask 在执行完任务之后调用的 postResult 方法：

```java
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
		
		//任务执行过程中和执行完毕的包装类
		//mData 即执行的中间数据或执行结果
		//mTask 即当前 AsyncTask，用于调用 finsh 和 publishProgress 方法
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }

    private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // 任务执行完毕回调
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                		// 任务执行过程中产生中间结果回调
                		// 重写的更新进度方法
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
          	//重写的回调结果方法
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }


```

##### 总结

首先说默认实现，即单线程线程池（SerialExecutor），它是系统全局的，也就是说每次实例化一个 AsyncTask 去执行任务的时候，都是先把任务添加到 ArrayDeque 队列尾部，然后等待上一个任务执行完毕之后，在去队列头部拿一个任务继续执行。
任务都被封装成一个 Callable<Params,Result>，即需要获取任务执行结果，执行的时候即是调用 doInBackground 方法，关于 onPostExecute 和 onProgressUpdate 即是在执行完毕和执行过程中通过 Handler 发消息来调用。
AsyncTask 支持多线程并发，需要使用 executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR)，传入 AsyncTask 中静态的线程池而已。

