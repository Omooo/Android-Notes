---
Binder 系列口水话
---

#### 面试题

谈谈你对 Binder 的理解？

#### 大纲

1. binder 是干嘛的？
2. binder 存在的意义是什么？
3. binder 的架构原理是怎样的？



#### Framework IPC 方式汇总

Android 是基于 Linux 内核构建的，Linux 已经提供了很多进程间通信机制，比如有管道、Socket、共享内存、信号等，在 Android Framework 中不仅用到了 Binder，这些 IPC 方式也都有使用到。

首先讲一下管道，管道是半双工的，管道的描述符只能读或写，想要既可以读也可以写就需要两个描述符，而且管道一般是用在父子进程之间的。Linux 提供了 pipe 函数创建一个管道，传入一个 fd[2] 数组，fd[0] 表示读端，fd[1] 表示写端。假如父进程创建了一对文件描述符，fork 出得子进程继承了这对文件描述符，这时候父进程想要往子进程写东西，就可以拿 fd[1] 写，然后子进程在 fd[0] 就可以读到了。在 Android 中，Native 层的 Looper 使用到了管道，它里面使用 epoll 监听读事件（epoll_wait），如果其他进程往里面写东西他就能收到通知。管道在哪写的呢，其实是在 wake 函数中，当别的线程向 Looper 线程发消息并且需要唤醒 Looper 线程的时候就会调用 wake 函数，wake 函数里面呢就是向管道写一个 "W" 字符。管道使用起来还是很方便的，主要是能配合 epoll 机制监听读写事件。这是 Android 19 才会使用到管道，更高版本使用的是 EventFd。

然后就是 Socket，Socket 是全双工的，也就是说既可以读也可以写，而且进程之间不需要亲缘关系，只需要公开一个地址即可。Framewok 中使用到 Socket 最经典的莫过于 Zygote 等待 AMS 请求 fork 应用程序进程了。在 Zygote 的 main 方法中 register 一个 ZygoteSocket，然后进入 runSelectLoop 循环去监听有没有新的连接，如果有的数据发过来就会去调用 runOnce 函数去根据参数 fork 出新的应用程序进程，其实就是去执行 ActivityThread 的 main 函数，然后也会通过这个 Socket 把新创建的应用进程 pid 返回给 Zygote。

接着是共享内存，共享内存它是不需要多次拷贝的，而且特别快。拿到文件描述符分别映射到进程的地址空间即可。在 Android 中提供了 MemoryFile 类，里面封装了 ashmem 机制，也就是 Android 的匿名共享内存。首先通过 ashmem_create_region 创建一块匿名共享内存，返回一个 fd，然后调用 mmap 函数给这个 fd 映射到当前进程地址空间中。

最后就是信号，信号是单向的，而且发出去之后不关心处理结果，知道进程的 pid 就能发信号了。在杀应用进程的时候会调用 Process 的 killProcess 函数发送一个 SIGNAL_KILL 信号。还有 Zygote 在 fork 完成一个新的子进程之后还会监听 SIGCHLD 信号，如果子进程退出之后就会回收相应的资源，避免子进程成为一个僵尸进程。