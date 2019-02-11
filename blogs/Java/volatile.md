---
volatile
---

#### 目录

1. 概述
   - volatile 的原理
   - 可见性
   - 有序性
   - 原子性
2. 参考

#### 概述

volatile 通常被比喻成轻量级的 synchronized，它保证了变量的可见性和有序性。但是使用 volatile 需要注意两点：

1. 运行结果不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值
2. 变量不需要与其他的状态变量共同参与不变约束

##### volatile 原理

对于 volatile 变量，当对 volatile 变量进行写操作的时候，JVM 会向处理器发生一个 lock 前缀的指令，将这个工作内存的值回写到主存中。其他线程在发现该变量的引用发生变化时，就会重新从主存中拉取更新。

##### 可见性

volatile 保证了可见性，可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看到修改后的值。

内部实现就是可参考内存模型。

##### 有序性

volatile 保证了有序性，禁止指令重排。

##### 原子性

volatile 并不保证原子性。在学 synchronized 的时候，我们知道，为了保证原子性，需要通过字节码指令 monitorenter 和 monitorexit 指令 或 ACCESS_SYNCHRONIZED 标记位来实现，而 volatile 和它们并没有关系，所以，volatile 并不保证原子性。

#### 参考

[漫画：什么是 volatile 关键字？](https://mp.weixin.qq.com/s/RQv8u-IhPmoGoDR1k1b6iw)

[再有人问你volatile是什么，就把这篇文章发给他](https://mp.weixin.qq.com/s/jSDAHKHWogeNU41ZS-fUwA)