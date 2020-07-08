---
Handler 消息机制口水话
---

#### 大纲

1. 基本使用
2. 源码分析
3. 常见问题
4. 实际应用

Android 应用是通过消息驱动运行的，在 Android 中一切皆消息，包括触摸事件、视图的绘制、显示和刷新等。Handler 是消息机制的上层接口，平时开发中我们只会接触到 Handler 和 Message，内部还有 MessageQueue 和 Looper 两大助手共同实现消息循环系统。

Handler 的基本使用我就不多说了，不过需要注意的是内存泄露，最好使用静态内部类 + 弱引用，并且在 Activity 退出时移除消息。

Handler 消息机制比较简单：

首先通过 Handler 的 sendMessage 或 post 两种方式去发送消息，post 内部也是 sendMessage 形式，只不过把传入的 Runnable 参数包装成 Message 的 callback。然后把 Message 传给 MessageQueue 的 enqueueMessage 入队，其实就是维护一个 Message 链表，它会根据 Message 的 when 时间排序，延迟消息不过是延时时间再加上当前时间。

然后就是核心逻辑了，在 Looper 的 loop 方法，它是一个死循环，里面做了三件事，调用 MessageQueue 的 next 方法取消息，然后通过 Message 的 target 也就是 handler 去 dispatchMessage 分发消息，最后回收消息。MessageQueue 的 next 也是一个死循环，首先会调用 nativePollOnce 函数，如果没有消息或者处理时间未到，就会阻塞在 Native 层的 pollOnce 函数，睡眠时间就是从 Java 层传过来的。不过其核心实现是在 pollInner 中，如果监听的文件描述符没有发生 IO 读写事件，那么当前线程就会在 epoll_wait 中进入休眠。如果当前线程有新的消息需要处理，就会被唤醒，然后沿着之前的调用路径返回到 Java 层，最后就是对新消息进行处理。

看完源码，有两点可以参考，第一是获取 Message 最好通过 Message 的 obtain 获取，不要直接 new，因为 Message.obtain 会从缓存里面去取。第二是可以使用 IdleHandler 在消息队列空闲时提前做一些操作。这个我在项目中也有用到，再点击消息中心图标时，会跳到 h5 页面，并且在 url 后面加密拼接 uid 和 phone number，这一加密操作是同步的，所以就可以通过 IdleHandler 提前做这一操作，并且返回 false，表示只做一次。不过这一 API 需要至少 23。