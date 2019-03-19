---
Java 线程生命周期
---

目录

1. 通用的线程生命周期
   - 初始状态
   - 可运行状态
   - 运行状态
   - 休眠状态
   - 终止状态
2. Java 中线程的生命周期
3. 摘自

#### 通用的线程生命周期

通用的线程生命周期基本是可以用下图 "五态模型" 来描述。这五态分别是：初始状态、可运行状态、运行状态、休眠状态和终止状态。

![](https://i.loli.net/2019/03/19/5c903a489f3c7.png)

这五种状态详细如下：

1. 初始状态

   指的是线程已经被创建，但是还不允许分配 CPU 执行。这个状态属于编程语言特有的，不过这里所谓的被创建，仅仅是在编程语言层面被创建，而在操作系统层面，真正的线程还没有创建。

2. 可运行状态

   指的是线程可以分配 CPU 执行。在这种状态下，真正的操作系统线程已经被成功创建了，所以可以分配 CPU 执行。

3. 当有空闲的 CPU 时，操作系统会将其分配一个处于可运行状态的线程，被分配到 CPU 的线程的状态就转换成了运行状态。

4. 运行状态的线程如果调用一个阻塞的 API（例如以阻塞方式读文件）或者等待某个事件（例如条件变量），那么线程的状态就会转换为休眠状态，同时释放 CPU 使用权，休眠状态的线程永远没有机会获得 CPU 使用权。当等待的事件出现了，线程就会从休眠状态转换为可运行状态。

5. 线程执行完或者出现异常就会进入终止状态，终止状态的线程不会切换到其它任何状态，进入终止状态也就意味着线程的生命周期结束了。

这五种状态在不同的编程语言里会有简化合并。Java 语言里把可运行状态和运行状态合并了，这两种状态在操作系统调度层面有用，而 JVM 层面不关心这两个状态，因为 JVM 把线程调度交给了操作系统处理了。

除了简化合并，这五种状态也有可能被细化，比如，Java 语言里就细化了休眠状态。

#### Java 中线程的生命周期

在 Java 语言里线程共有六种状态，分别是：

1. NEW 初始化状态
2. RUNNABLE 可运行 / 运行状态
3. BLOCKED 阻塞状态
4. WAITING 无时限等待
5. TIMED_WAITING 有时限等待
6. TERMINATED 终止状态

Java 中把操作系统中线程的休眠状态细分了阻塞状态、无时限等待和有时限等待状态，也就是说，只要 Java 线程处于这三种状态之一，那么这个线程就永远没有 CPU 的使用权。

![](https://i.loli.net/2019/03/19/5c907c24121ac.png)

那么有哪些情形会导致线程从 RUNNABLE 状态切换到这三种状态呢？

1. RUNNABLE 与 BLOCKED 的状态切换

   只有一种场景会触发这种切换，就是线程等待 synchronized 的隐式锁。synchronized 修饰的方法、代码块同一时刻只允许一个线程执行，其他线程只能等待，这种情况下，等待的线程就会从 RUNNABLE 转换为 BLOCKED 状态。而当等待的线程获得 synchronized 隐式锁时，就又会从 BLOCKED 转换为 RUNNABLE 状态。

   如果线程调用阻塞式 API 时，是否会转换为 BLOCKED 状态呢？在操作系统层面，线程是会转换到休眠状态的，但是在 JVM 层面，Java 线程的状态不会发生变化，也就是说 Java 线程的状态会依然保持 RUNNABLE 状态。JVM 层面并不关心操作系统调度相关的状态，因为在 JVM 看来，等待 CPU 使用权（操作系统层面此时处于可执行状态）与等待 I/O（操作系统层面此时处于休眠状态）没有区别，都在等待某个资源，所以都归入了 RUNNABLE 状态。

   而我们平时所谓的 Java 在调用阻塞式 API 时，线程会阻塞，指的是操作系统线程的状态，并不是 Java 线程的状态。

2. RUNNABLE 与 WAITING 的状态转换

   总的来说，有三种场景会触发这种转换。

   第一种场景，获得 synchronized 隐式锁的过程，调用无参数的 Object.wait() 方法。

   第二种场景，调用无参数的 Thread.join() 方法。其中的 join() 是一种线程同步方法。例如有一个线程对象 ThreadA，当调用 ThreadA.join() 的时候，执行这条语句的线程会等待 ThreadA 执行完，而等待中的这个线程，其状态会从 RUNNABLE 转换到 WAITING。当线程 ThreadA 执行完，原来等待它的线程又会从 WAITING 状态转换到 RUNNABLE。

   第三章场景，调用 LockSupport.park() 方法，当前线程会阻塞，线程的状态会从 RUNNABLE 转换到 WAITING。调用 LockSupport.unpark(Thread thread) 可唤醒目标线程，目标线程的状态又会从 WAITING 状态转换到 RUNNABLE。

3. RUNNABLE 与 TIMED_WAITING 的状态转换

   有五种场景会触发这种转换。

   第一种，调用带超时参数的 Thread.sleep(long millis) 方法；

   第二种，获得 synchronized 隐式锁的线程，调用带超时参数的 Object.wait(long millis) 方法；

   第三种，调用带超时参数的 Thread.join(long millis) 方法；

   第四种，调用带超时参数的 LockSupport.parkNanos(Object blocker, long deadline) 方法；

   第五种，调用带超时参数的 LockSupport.parkUntil(long deadline) 方法；

   这里 TIMED_WAITING 和 WAITING 状态的区别，仅仅是触发条件多了超时参数。

4. 从 NEW 到 RUNNABLE 状态

   Java 刚创建出来的 Thread 对象就是 NEW 状态，而创建 Thread 对象主要有两种方法。一种是继承 Thread 对象，另一种是实现 Runnable 接口。NEW 状态的线程，不会被操作系统调度，因此不会执行。Java 线程要执行，就必须转换为 RUNNABLE 状态。从 NEW 状态转换到 RUNNABLE 状态很简单，只要调用线程对象的 start() 方法即可。

5. 从 RUNNABLE 到 TERMINATED 状态

   线程执行完 run() 方法后，会自动转换到 TERMINATED 状态，当然如果执行 run() 方法的时候抛出异常，也会导致线程终止。有时候我们需要强制中断 run() 方法的执行，例如 run() 方法访问一个很慢的网络，我们等不下去了，想要终止怎么办？Java 的 Thread 类倒是提供了 stop() 方法，不过已经被标记为 @Deprecated，所以不建议使用了，正确的姿势其实是调用 interrupt() 方法。

那 stop() 和 interrupt() 方法的主要区别是什么呢？

stop() 方法会真的杀死线程，不给线程喘息的机会，如果线程持有 synchronized 隐式锁，也不会释放，那其他线程就再也没机会获得 synchronized 隐式锁，这实在是太危险了。所以该方法就不建议使用了，类似的方法还有 suspend() 和 resume() 方法。

而 interrupt() 方法就温柔多了，interrupt() 方法仅仅是通知线程，线程有机会执行一些后续操作，同时也可以无视这个通知。被 interrupt 的线程，是怎么收到通知的呢？一种是异常，另一种是主动检测。

当线程 A 处于 WAITING、TIMED_WAITING 状态时，如果其它线程调用 A 的 interrupt() 方法，会使线程 A 返回到 RUNNABLE 状态，同时线程 A 的代码会触发 InterruptedException 异常。上面我们提到转换到 WAITING、TIMED_WAITING 状态的触发条件，都是调用了类似 wait()、join()、sleep() 这样的方法，这些方法都会 throws InterruptedException 这个异常。这个异常的触发条件就是：其它线程调用了该线程的 interrupt() 方法。

当线程 A 处于 RUNNABLE 状态时，并且阻塞在 java.nio.channels.InterruptibleChannel 上时，如果其它线程调用线程 A 的 interrupt() 方法，线程 A 会触发 java.nio.channels.ClosedByInterruptException 这个异常；而阻塞在 java.nio.channels.Selector 上时，如果其它线程调用线程 A 的 interrupt() 方法，线程 A 的 java.nio.channels.Selector 会立即返回。

上面这两种情况属于被中断的线程通过异常的方式获得了通知。还有一种是主动监测，如果线程处于 RUNNABLE 状态，并且没有阻塞在某个 I/O 操作上，例如中断计算圆周率的线程 A，这时就得依赖线程 A 主动检测中断状态了。如果其它线程调用线程 A 的 interrupt() 方法，那么线程 A 可以通过 isInterrupted() 方法，检测是不是自己被中断了。