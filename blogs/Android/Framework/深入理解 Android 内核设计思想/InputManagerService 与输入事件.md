---
InputManagerService 与输入事件
---

#### 目录

1. 事件的分类
2. 事件的投递流程
3. 事件注入

#### 事件的分类

常见的时间分为两种：

1. 按键事件（KeyEvent）

   由物理按键产生的事件，比如 Home 键、Back 键等等。

2. 触摸事件（MotionEvent）

   触摸屏的点击、滑动事件。

#### 事件的投递流程

```
源信息采集 -> 前期处理 -> WMS 分配 -> 应用程序处理
```

WindowManagerService 是窗口的大主管，同时也是 InputEvent 的派发者。这样的设计是自然而然的，因为 WMS 记录了当前系统中所有窗口的完整状态信息，所以也只有它才能判断出应该把事件投递给哪一个具体的应用程序进程处理。

InputManagerService 的创建过程和 WMS 类似，都是由 SystemServer 统一启动：

```java
// SystemServer
public void run() {
    inputManager = new InputManagerService(context, wmHandler);
    wm = WindowManagerService.main(context, inputManager, ...);
    ServiceManager.addService(Context.WINDOW_SERVICE, wm);
    ServiceManager.addService(Context.Input_SERVICE, inputManager);
    inputManager.setWindowManagerCallbacks(wm.getInputMonitor);
    inputManager.start();
}
```

IMS 在 Native 层创建了两个新的线程，InputReaderThread 和 InputDispatcherThread。前者负责从驱动节点中读取 Event，后者专责于分发事件。

InputReaderThread 中的实现核心是 InputReader 类，但在 InputReader 实际上并不直接去访问设备节点，而是通过 EventHub 来完成这一工作。EventHub 通过读取 /dev/input 下的相关文件来判断是否有新事件，并通知 InputReader。

InputDispatcherThread 和 InputReaderThread 都是独立的线程，并且都运行在系统进程中。InputDispatcherThread 在创建之初，就把自己的实例传给了 InputReaderThread，这样便可以源源不断的获取事件进行分发了。

分发时是如何找到投递目标呢？也就是 findFocusedWindowTargetsLocked 方法的实现。也就是通过 InputMonitor 来找到 “最前端” 的窗口即可。这个 InputMonitor 是 WMS 提供的，而 IMS 实现了 WindowManagerCallbacks 接口，并把 InputMonitor 作为参数传递进去。

在获知 InputTarget 之后，InputDispatcher 就需要和窗口建立连接，是通过 InputChannel，这也是一个跨进程通信，但是并不是采用 Binder，而是 Unix Domain Socket 实现。

在 Java 层，InputEventReceiver 对 InputChannel 进行包装，它是一个抽象类，它唯一的实现 WindowInputEventReceiver 就是在 ViewRootImpl 中，这样 ViewRootImpl 就可以获取到事件了。

在 ViewRootImpl 中，一旦获知 InputEvent，就会进行入队操作，如果是紧急事件就直接调用 doProcessInputEvent 处理，如果不是紧急事件，就会把这个 InputEvent 推送到消息队列，然后按顺序处理，此时需要注意 Message 为异步消息。

#### 事件注入

正常情况下，Android 系统中的各种 Event 都是由用户主动发起而产生的，但是在某些特殊的情况下，比如自动化测试的场景中，我们希望可以通过程序来模拟用户的上述行为，此时就需要事件注入技术了。

事件注入有几下几种：

1. 通过 InputManager 提供的内部 API

   ```java
   InputManager.getInstance().injectInputEvent(...);
   ```

2. 利用 Instrumentation 提供的辅助事件注入接口

3. 通过 system/bin 目录下的 input 程序

   ```shell
   adb shell
   // 发送一个 HOME 按键事件
   input keyevent 3
   ```

4. 直接写入数据到 /dev/input/eventX 节点中