---
LeakCanary
---

#### 目录

1. 实现原理
2. 简单实现
3. 参考

#### 实现原理



#### 简单实现

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
        // 在 Activity#onDestory 时，把 Activity 添加到观察列表里
        // ！！注意这里的 queue，如果 activity 被回收了，activity 会被添加到 queue 里面
        val reference =
          MyKeyedWeakReference(activity, UUID.randomUUID().toString(), "activity", queue)
        watchedObjects[reference.key] = reference
        Log.i(TAG, "onActivityDestroyed: activity 即将被回收")
        // 主线程延迟 5s，之所以延迟 5s，是因为两次 gc 最小间隔时间是 5s
        Handler(Looper.getMainLooper()).postDelayed({
          // 看看对象有没有被回收
          removeRefIfCollect()
          // 可能泄漏了，手动 GC 一下再试试
          Log.i(TAG, "gc: ")
          Runtime.getRuntime().gc()
          try {
            // 直接粗暴的睡 100ms，让对象能够有足够的时间添加到弱引用队列里面
            Thread.sleep(100)
          } catch (e: InterruptedException) {
            throw AssertionError()
          }
          System.runFinalization()
          // 再次看看对象有没有被回收
          removeRefIfCollect()
          val leakRef = watchedObjects[reference.key]
          // 如果不为 null，也就是没有从观察对象列表里面移除
          // 说明对象被其他对象引用着，也就是存在泄漏
          if (leakRef != null) {
            Log.i(TAG, "对象泄露了: ${leakRef.desc}")
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
}

class MyKeyedWeakReference(ref: Any, val key: String, val desc: String, queue: ReferenceQueue<Any>) :
  WeakReference<Any>(ref, queue) {
}

private const val TAG = "MyAppWatcher"
```



#### 参考

[https://github.com/square/leakcanary](https://github.com/square/leakcanary)