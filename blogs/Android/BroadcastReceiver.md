---
BroadcastReceiver
---

#### 目录

1. 思维导图
2. 概述
3. 注册方式
   - 静态注册
   - 动态注册
4. 广播分类
   - 普通广播
   - 系统广播
   - 有序广播
   - 粘性广播
   - 应用内广播
5. 参考

#### 思维导图

![](https://i.loli.net/2019/02/01/5c53fda691b83.png)

#### 概述

在 Android 系统中，广播是在组件之间传递数据的一种机制，这些组件可以位于不同的进程中，起到进程间通信的作用。

BroadcastReceiver 是对发送出来的广播进行过滤、接收和响应的组件，首先将要发送的消息和用于过滤的信息（action、category）装入一个 Intent 对象，然后通过调用 Context.sendBroadcast() 或 Context.sendOrderBoradcast() 方法把 Intent 对象以广播形式发送出去。广播发送出去后，所有已注册的 BroadcastReceiver 会检查注册时的 IntnetFilter 是否与发送的 Intent 相匹配，若匹配则会调用 BroadcastReceiver 的 onReceiver 方法。

BroadcastReceiver 的生命周期很短，一旦 onReceiver 执行完毕，生命周期就结束了。

#### 注册方式

广播的注册分为静态注册和动态注册。

不管静态注册还是动态注册，首先得有广播呀：

```java
public class MyBroadcastReceiver extends BroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {
        if (!TextUtils.isEmpty(intent.getStringExtra("name"))){
            Toast.makeText(context, intent.getStringExtra("name"), Toast.LENGTH_SHORT).show();
        }
    }
}
```

##### 静态注册

静态注册即指在 AndroidManifest 里面注册。应用一旦启动，它就会常驻进程，不受任何组件的生命周期影响，即使应用程序关闭后，仍能接收广播，耗电同时占内存。

```xml
        <receiver android:name=".broadcast.MyBroadcastReceiver"
            android:exported="false">
            <intent-filter>
                <action android:name="demo"/>
            </intent-filter>
        </receiver>
```

```java
      mButton.setOnClickListener(new ClickListenerWrapper(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent("demo");
                intent.putExtra("name", "Omooo");
                sendBroadcast(intent);
            }
        }));
```

**注意：**

静态注册在 Android 8 上已经完全失效了，所以不建议在使用静态注册。

##### 动态注册

动态注册即在代码中通过调用 Context.registerReceiver(BroadcastReceiver,IntentFilter) 来注册，它有几个重载方法，但是都会有这两个参数。一般建议在 Activity 的 onResume 注册，在 onPause 中注销，避免内存泄露。同时也不允许重复注册、重复注销。

```java
    private MyBroadcastReceiver mBroadcastReceiver;

    @Override
    protected void onResume() {
        super.onResume();
        IntentFilter filter = new IntentFilter();
        filter.addAction("demo");
        mBroadcastReceiver = new MyBroadcastReceiver();
        registerReceiver(mBroadcastReceiver, filter);
    }

    @Override
    protected void onStop() {
        super.onStop();
        unregisterReceiver(mBroadcastReceiver);
    }
```

#### 广播分类

广播可以分为五类：普通广播、系统广播、有序广播、粘性广播和应用内广播。

##### 普通广播

即开发者自定义的广播，也是最常用的广播，上面的实例就是普通广播。

##### 系统广播

Android 中内置了多个系统广播：只要涉及到手机的基本操作（如开关机、网络状态变化、拍照等待），都会发出相应的广播。常用的有：

| 系统操作     | action                               |
| ------------ | ------------------------------------ |
| 监听网络变化 | android.net.conn.CONNECTIVITY_CHANGE |
| 电池电量低   | Intent.ACTION_BATTERY_LOW            |
| 成功安装APK  | Intent.ACTION_PACKAGE_               |
| ...          | ...                                  |

上面我们自己随便写的 action="demo"，所以每次发送广播需要自己手动调用 sendBroadcast，这时候可以携带数据过去，而系统广播并不能自己传递数据。

##### 有序广播

有序是针对广播接收者来说的，前面的实例，广播发出去之后，所有满足条件的广播接收者都能通知接收广播。有序广播则是按照广播接收者的 priority 属性值从大到小排序依次接收，同 priority 的动态广播优先于静态广播。同时，前一个广播接收器也可以通过调用 abortBroadcast() 方法截断广播，这样优先级低的广播接收者就无法接收到广播了，也可以修改数据传递，这就带来了安全性问题。

前面说过，强烈不建议使用静态注册，而 priority 属性就是在 xml 文件里面配置的。

发送有序广播使用：

```java
sendOrderedBroadcast(Intent intent, String receiverPermission)
```

##### 粘性广播

在 Android 5.0 上已经失效，不建议使用。

##### 应用内广播

之前发送和接收到的广播全都是属于系统全局广播，即发出的广播可以被其他应用接收到，而且也可以接收到其他应用发送出的广播，这样可能会有不安全因素。

当不需要通过 sendBroadcast 来完成跨应用的通信，那么建议使用 LocalBroadcastManager，将会是更加高效、安全的方式，并且对系统带来的影响也更小。

BroadcastReceiver 采用的 Binder 方式实现跨进程通信，而 LocalBroadcastManager 使用 Handler 通信机制。

| 函数                                                         | 作用          |
| ------------------------------------------------------------ | ------------- |
| LocalBroadcastManager.getInstance(this).registerReceiver(BroadcastReceiver,IntentFilter) | 注册 Receiver |
| LocalBroadcastManager.getInstance(this).unregisterReceiver(BroadcastReceiver,IntentFilter) | 注销 Receiver |
| LocalBroadcastManager.getInstance(this).sendBroadcast(Intent) | 发送异步广播  |
| LocalBroadcastManager.getInstance(this).sendBroadcastSync(Intent) | 发送同步广播  |

注意，应用内广播只能是动态注册。

#### 参考

[Android四大组件：BroadcastReceiver史上最全面解析](https://www.jianshu.com/p/ca3d87a4cdf3)

[Android BroadcastReceiver使用详解](https://juejin.im/post/5a9266185188257a5a4ccfc5)

[LocalBroadcastManager原理分析](http://gityuan.com/2017/04/23/local_broadcast_manager/)