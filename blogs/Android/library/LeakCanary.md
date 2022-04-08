---
手写 LeakCanary 简单实现
---

#### 前言

这篇文章写作于 2022/04/08，截止至此，[leakcanary](https://github.com/square/leakcanary) 的版本是 v2.8.1，应该没我更新的了。

这篇文章不是源码分析，而是根据 LeakCanary 的实现原理，手写一个简单的内存泄漏实现。

#### 前置理论知识

内存泄露是指，不再使用的对象本应该要被回收，但却没有被回收。

在 Android 中，哪些是不再使用的对象呢？

这个问题比较好回答，比如已经回调 onDestory 的 Activity 对象、已经回调 onFragmentDestroyed 的 Fragment、已经回调 onViewDetachedFromWindow 的 Dialog 等等。

那怎么检测这些对象有没有被回收呢？

一般我们为了避免强引用某些对象而导致内存泄露，而选择使用 WeakReference 包一层，使用的是如下构造函数：

```java
public WeakReference(T referent) {
  	super(referent);
}
```

而 WeakReference 还有另一个构造函数：

```java
public WeakReference(T referent, ReferenceQueue<? super T> queue) {
  super(referent, queue);
}
```

**注意这里的第二个参数，这个参数是为其指定一个引用队列，当 referent 被垃圾回收时，会把 referent 放进关联的引用队列 queue 里面。**

那我们岂不是可以把 onDestory 的 Activity 对象使用 WeakReference 承载，并且指定一个 ReferenceQueue，然后看看 ReferenceQueue 里面有没有该 Activity 对象，就知道该 Activity 有没有被回收了？

OK，知道原理之后就可以动手了。

#### 简单实现

注释写的非常清楚：

```kotlin
object MyAppWatcher {

  /** 引用队列 */
  private val queue = ReferenceQueue<Any>()

  /** 待观察的对象 */
  private val watchedObjects = mutableMapOf<String, MyKeyedWeakReference>()

  /**
   * 注册方法，可以在 Application#onCreate 里调用或者直接使用 ContentProvider 注册
   */
  fun install(app: Application) {
    // 这里以检测 Activity 泄露为例，注册 Activity#onDestory 回调
    app.registerActivityLifecycleCallbacks(object :
      Application.ActivityLifecycleCallbacks by noOpDelegateMy() {

      override fun onActivityDestroyed(activity: Activity) {
        // 1. 在 Activity#onDestory 时，把 Activity 添加到观察列表里
        // ！！注意这里的 queue，如果 activity 被回收了，activity 会被添加到 queue 里面
        val reference =
          MyKeyedWeakReference(activity, UUID.randomUUID().toString(), "activity", queue)
        watchedObjects[reference.key] = reference
        Log.i(TAG, "onActivityDestroyed: activity 即将被回收")
        // 2. 主线程延迟 5s，之所以延迟 5s，是因为两次 gc 最小间隔时间是 5s
        Handler(Looper.getMainLooper()).postDelayed({
          // 3. 看看对象有没有被回收
          removeRefIfCollect()
          // 4. 可能泄漏了，手动 GC 一下再试试
          Log.i(TAG, "gc: ")
          Runtime.getRuntime().gc()
          try {
            // 直接粗暴的睡 100ms，让对象能够有足够的时间添加到弱引用队列里面
            Thread.sleep(100)
          } catch (e: InterruptedException) {
            throw AssertionError()
          }
          System.runFinalization()
          // 5. 再次看看对象有没有被回收
          removeRefIfCollect()
          val leakRef = watchedObjects[reference.key]
          // 6. 如果不为 null，也就是没有从观察对象列表里面移除
          // 说明对象被其他对象引用着，也就是存在泄漏
          if (leakRef != null) {
            Log.i(TAG, "对象泄露了: ${leakRef.desc}")
            // 7. dump 堆转储文件并分析
            dumpHeap(activity.applicationContext)
          }
        }, 5000)
      }
    })
  }

  /**
   * 如果对象被回收了，则把它从观察对象列表里面移除
   * 核心原理：当对象被回收时，会把它添加到其引用队列里面
   */
  private fun removeRefIfCollect() {
    var ref: MyKeyedWeakReference?
    do {
      ref = queue.poll() as MyKeyedWeakReference?
      if (ref != null) {
        Log.i(TAG, "removeRefIfCollect: ${ref.desc}")
        watchedObjects.remove(ref.key)
      }
    } while (ref != null)
  }

  /**
   * dump hprof 文件并使用 shark 库分析，输出结果
   */
  private fun dumpHeap(context: Context) {
    val handlerThread = HandlerThread("Dump-Heap-Thread")
    handlerThread.start()
    Handler(handlerThread.looper).post {
      val storageDirectory = File(context.cacheDir, "leakcanary")
      if (!storageDirectory.exists()) {
        storageDirectory.mkdir()
      }
      val fileName = SimpleDateFormat("yyyy-MM-dd_HH-mm-ss_SSS'.hprof'", Locale.US).format(Date())
      val file = File(storageDirectory, fileName)
      // dump 出堆转储文件
      Debug.dumpHprofData(file.absolutePath)
      Log.i(TAG, "dumpHeap: ${file.absolutePath}")
      // 使用 Shark 库里的 HeapAnalyzer 来分析
      val heapAnalyzer = HeapAnalyzer(OnAnalysisProgressListener { step ->
        Log.i(TAG, "Analysis in progress, working on: ${step.name}")
      })
      val heapAnalysis = heapAnalyzer.analyze(
        heapDumpFile = file,
        leakingObjectFinder = FilteringLeakingObjectFinder(
          AndroidObjectInspectors.appLeakingObjectFilters
        ),
        referenceMatchers = AndroidReferenceMatchers.appDefaults,
        computeRetainedHeapSize = true,
        objectInspectors = AndroidObjectInspectors.appDefaults.toMutableList(),
        proguardMapping = null,
        metadataExtractor = AndroidMetadataExtractor
      )
      Log.i(TAG, "dumpHeap: \n$heapAnalysis")
    }
  }
}

private const val TAG = "MyAppWatcher"
```

整个代码除了 dumpHeap 方法需要依赖 leakcanary-shark 库之外，是没有其他依赖的，dumpHeap 方法适用于当发生了内存泄露，直接把 hprof 文件 dump 并解析泄漏的引用链的。

#### 简单测试

首先需要注册一下，leakcanary 是使用 ContentProvider 注册，我们这里使用 Application 注册一下，主要是可以很方便的模拟内存泄露：

```kotlin
open class ExampleApplication : Application() {
  // 泄漏的 View
  val leakedViews = mutableListOf<View>()

  override fun onCreate() {
    super.onCreate()
    MyAppWatcher.install(this)
  }
}
```

搞一个 SecondActivity 泄漏一下：

```kotlin
class SecondActivity : Activity() {
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_second)

    val app = application as ExampleApplication
    app.leakedViews.add(findViewById(R.id.tv_text))
  }
}
```

当 SecondActivity onDestory 时，因为 application 持有了 view 的强引用导致 SecondActivity 不会被回收掉，也就是发生了内存泄露，我们来看看输出了啥日志。

#### 日志输出

该日志，也就是上面 MyAppWatcher 打出来的 log：

```log
2022-04-08 14:25:00.030 25500-25500/com.example.leakcanary I/MyAppWatcher: onActivityDestroyed: activity 即将被回收
2022-04-08 14:25:05.036 25500-25500/com.example.leakcanary I/MyAppWatcher: gc: 
2022-04-08 14:25:05.212 25500-25500/com.example.leakcanary I/MyAppWatcher: 对象泄露了: activity
2022-04-08 14:25:07.004 25500-25569/com.example.leakcanary I/MyAppWatcher: dumpHeap: /data/user/0/com.example.leakcanary/cache/leakcanary/2022-04-08_14-25-05_220.hprof
2022-04-08 14:25:07.127 25500-25569/com.example.leakcanary I/MyAppWatcher: Analysis in progress, working on: PARSING_HEAP_DUMP
2022-04-08 14:25:08.916 25500-25569/com.example.leakcanary I/MyAppWatcher: Analysis in progress, working on: EXTRACTING_METADATA
2022-04-08 14:25:09.026 25500-25569/com.example.leakcanary I/MyAppWatcher: Analysis in progress, working on: FINDING_RETAINED_OBJECTS
2022-04-08 14:25:12.094 25500-25569/com.example.leakcanary I/MyAppWatcher: Analysis in progress, working on: FINDING_PATHS_TO_RETAINED_OBJECTS
2022-04-08 14:25:16.039 25500-25569/com.example.leakcanary I/MyAppWatcher: Analysis in progress, working on: INSPECTING_OBJECTS
2022-04-08 14:25:16.073 25500-25569/com.example.leakcanary I/MyAppWatcher: Analysis in progress, working on: COMPUTING_NATIVE_RETAINED_SIZE
2022-04-08 14:25:16.289 25500-25569/com.example.leakcanary I/MyAppWatcher: Analysis in progress, working on: COMPUTING_RETAINED_SIZE
2022-04-08 14:25:16.431 25500-25569/com.example.leakcanary I/MyAppWatcher: Analysis in progress, working on: BUILDING_LEAK_TRACES
2022-04-08 14:25:16.443 25500-25569/com.example.leakcanary I/MyAppWatcher: dumpHeap: 
    ====================================
    HEAP ANALYSIS RESULT
    ====================================
    1 APPLICATION LEAKS
    
    References underlined with "~~~" are likely causes.
    Learn more at https://squ.re/leaks.
    
    118361 bytes retained by leaking objects
    Signature: 63f8e699b71efe361be3e4edf804ad0ff76866a1
    ┬───
    │ GC Root: System class
    │
    ├─ android.provider.FontsContract class
    │    Leaking: NO (ExampleApplication↓ is not leaking and a class is never leaking)
    │    ↓ static FontsContract.sContext
    ├─ com.example.leakcanary.ExampleApplication instance
    │    Leaking: NO (Application is a singleton)
    │    mBase instance of android.app.ContextImpl
    │    ↓ ExampleApplication.leakedViews
    │                         ~~~~~~~~~~~
    ├─ java.util.ArrayList instance
    │    Leaking: UNKNOWN
    │    Retaining 118.4 kB in 1069 objects
    │    ↓ ArrayList[0]
    │               ~~~
    ├─ android.widget.TextView instance
    │    Leaking: YES (View.mContext references a destroyed activity)
    │    Retaining 118.4 kB in 1067 objects
    │    View is part of a window view hierarchy
    │    View.mAttachInfo is null (view detached)
    │    View.mWindowAttachCount = 1
    │    mContext instance of com.example.SecondActivity with mDestroyed = true
    │    ↓ View.mContext
    ╰→ com.example.SecondActivity instance
    ​     Leaking: YES (Activity#mDestroyed is true)
    ​     Retaining 24.4 kB in 74 objects
    ​     mApplication instance of com.example.leakcanary.ExampleApplication
    ​     mBase instance of android.app.ContextImpl
    ====================================
    0 LIBRARY LEAKS
    
    A Library Leak is a leak caused by a known bug in 3rd party code that you do not have control over.
    See https://square.github.io/leakcanary/fundamentals-how-leakcanary-works/#4-categorizing-leaks
    ====================================
    1 UNREACHABLE OBJECTS
    
    An unreachable object is still in memory but LeakCanary could not find a strong reference path
    from GC roots.
    
    android.view.ViewRootImpl instance
    ​  Leaking: YES (ViewRootImpl#mView is null)
    ​  mContext instance of com.android.internal.policy.DecorContext, wrapping activity com.example.SecondActivity with mDestroyed = true
    ​  mWindowAttributes.mTitle = "com.example.leakcanary/com.example.SecondActivity"
    ​  mWindowAttributes.type = 1
    ====================================
    METADATA
    
    Please include this in bug reports and Stack Overflow questions.
    
    Build.VERSION.SDK_INT: 30
    Build.MANUFACTURER: OPPO
    LeakCanary version: Unknown
    App process name: com.example.leakcanary
    Stats: LruCache[maxSize=3000,hits=37016,misses=78089,hitRate=32%] RandomAccess[bytes=3892247,reads=78089,travel=30739517820,range=20956324,size=26877108]
    Analysis duration: 7820 ms
    Heap dump file path: /data/user/0/com.example.leakcanary/cache/leakcanary/2022-04-08_14-25-05_220.hprof
    Heap dump timestamp: 1649399116436
    Heap dump duration: Unknown
    ====================================

```

这样我们就完成了一个简单的内存泄露检测了。

强烈大家把 leakcanary 下载下来，跑一下 sample 工程，调试一下，对理解 leakcanary 非常有用，甚至你不用看任何源码分析的文章就可以直接理解了。