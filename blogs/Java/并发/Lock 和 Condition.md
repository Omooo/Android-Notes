---
Lock 和 Condition: 隐藏在并发包中的管程
---

#### 前言

Java SDK 并发包内容很丰富，包罗万象，但是我觉得最核心的还是其对管程的实现。因为理论上利用管程，你几乎可以实现并发包里所有的工具类。前面提到过，在并发编程领域，有两大核心问题：一个是互斥，即同一时刻只允许一个线程访问共享资源；另一个是同步，即线程之间如何进行通信、协作。这两大问题，管程都是能够解决的。Java SDK 并发包通过 Lock 和 Condition 两个接口来实现管程，其中 Lock 用于解决互斥问题，Condition 用于解决同步问题。

在介绍 Lock 之前，我们知道，synchronized 也是管程的一种实现，既然 Java 从语言层面已经实现了管程，那为什么还要在 SDK 里提供另外一种实现呢？

#### 再造管程的理由

在 Java 1.5 版本中，synchronized 性能不如 SDK 里面的 Lock，但 1.6 版本之后，synchronized 做了很多优化，将性能追了上来，所以 1.6 之后的版本又有人推荐使用 synchronized 了。至此，为什么还需要重复造轮子呢？

在介绍死锁的时候，提出了一个**破坏不可抢占条件**的方案，但是这个方案 synchronized 没有办法解决。原因是 synchronized 申请资源的时候，如果申请不到，线程直接进入阻塞状态了，而线程进入阻塞状态，啥都干不了，也释放不了线程已经占用的资源。

但是我们希望的是：对于 “不可抢占” 这个条件，占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占用的资源，这样不可抢占这个条件就破坏了。

那该如何去设计呢？我觉得有三种方案：

1. 能够响应中断

   synchronized 的问题是，持有锁 A 后，如果尝试获取锁 B 失败，那么线程就进入阻塞状态，一旦发生死锁，就没有任何机会来唤醒阻塞的线程。但如果阻塞状态的线程能够响应中断信息，也就是说当我们给阻塞的线程发生中断信号的时候，能够唤醒它，那它就有机会释放曾经持有的锁 A。这样就破坏了不可抢占的条件了。

2. 支持超时

   如果线程在一段时间之内没有获取到锁，不是进入阻塞状态，而是返回一个错误，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件。

3. 非阻塞的获取锁

   如果尝试获取锁失败，并不进入阻塞状态，而是直接返回，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件。

这三种方案可以全面弥补 synchronized 的问题。到这里相信你应该也能理解了，这三个方案就是重复造轮子的主要原因，体现在 API 上，就是 Lock 接口的三个方法：

```java
public interface Lock {
  	void lock();
  	//支持中断
  	void lockInterruptibly() throws InterruptedException;
  	//支持非阻塞获取锁
  	boolean tryLock();
  	//支持超时
  	boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
  	void unlock();
  	Condition newCondition();
}
```

#### 可见性的保证

Java SDK 里面 Lock 的使用，有一个经典的范例，就是 try{} finally{}，需要重点关注的是在 finally 里面释放锁：

```java
    private int value;
    private Lock lock = new ReentrantLock();

    private void add() {
        lock.lock();
        try {
            value++;
        } finally {
            lock.unlock();
        }
    }
```

但是它是如何实现可见性的呢？它是利用了 volatile 变量的 Happens-Before 规则。ReentrantLock 用到了 AQS，AQS 里面有一个 volatile 类型的 state 的值，在加解锁的时候都会去读 state 的值。简化代码如下：

```java
class SampleLock {
  volatile int state;
  // 加锁
  lock() {
    state = 1;
  }
  // 解锁
  unlock() {
    state = 0;
  }
}
```

根据相关的 Happens-Before 规则：

1. 顺序性规则

   对于线程 T1，value++ Happens-Before 释放锁的操作 unlock()；

2. volatile 变量规则

   由于 state = 1 会先读取 state，所以线程 T1 的 unlock() 操作 Happens-Before 线程 T2 的 lock() 操作；

3. 传递性规则

   线程 T1 的 value++ Happens-Before 线程 T2 的 lock() 操作；

