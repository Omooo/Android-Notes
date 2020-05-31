---
Binder 系列口水话
---

#### 面试题

1. 谈谈你对 Binder 的理解？
2. 一次完整的 IPC 通信流程是怎样的？
3. Binder 对象跨进程传递的原理是怎么样的？
4. 说一说 Binder 的 oneway 机制
5. Framework 中其他的 IPC 通信方式 

#### 谈谈你对 Binder 的理解？

Binder 是 Android 中一种高效、方便、安全的进程间通信方式。高效是指 Binder 只需要一次数据拷贝，把一块物理内存同时映射到内核和目标进程的用户空间。方便是指用起来简单直接，Client 端使用 Service 端的提供的服务只需要传 Service 的一个描述符即可，就可以获取到 Service 提供的服务接口。安全是指 Binder 验证调用方可靠的身份信息，这个身份信息不能是调用方自己填写的，显然不可靠的，而可靠的身份信息应该是 IPC 机制本身在内核态中添加。

Binder 通信模型由四方参与，分别是 Binder 驱动层、Client 端、Service 端和 ServiceManager。

Client 端表示应用程序进程，Service 端表示系统服务，它可能运行在 SystemService 进程，比如 AMS、PKMS等，也可能是运行在一个单独的进程中，比如 SurfaceFlinger。ServiceManager 是 Binder 进程间通信方式的上下文管理者，它提供 Service 端的服务注册和 Client 端的服务获取功能。它们之间是不能直接通信的，需要借助于 Binder 驱动层进行交互。

这就需要它们首先通过 binder_open 打开 binder 驱动，然后根据返回的 fd 进行内存映射，分配缓冲区，最后启动 binder 线程，启动 binder 线程一方面是把这个这些线程注册到 binder 驱动，另一方面是这个线程要进入 binder_loop 循环，不断的去跟 binder 驱动进程交互。

接下来就可以开始 binder 通信了，我们从 ServiceManager 说起。

ServiceManger 的 main 函数首先调用 binder_open 打开 binder 驱动，然后调用 binder_become_context_manager 注册为 binder 的大管家，也就是告诉 Binder 驱动 Service 的注册和获取都是通过我来做的，最后进入 binder_loop 循环。binder_loop 首先通过 BC_ENTER_LOOPER 命令协议把当前线程注册为 binder 线程，也就是 ServiceManager 的主线程，然后在一个 for 死循环中不断去读 binder 驱动发送来的请求去处理，也就调用 ioctl。

有了 ServiceManager 之后，Service 系统服务就可以向 ServiceManager 进行注册了。也 ServiceFlinger 为例，在它的入口函数 main 函数中，首先也需要启动 binder 机制，也就是上所说的那三步，然后就是初始化 ServiceFlinger，最后就是注册服务。注册服务首先需要拿到 ServiceManager 的 Binder 代理对象，也就是通过 defaultServiceManager 方法，真正获取 ServiceManager 代理对象的是通过 getStrongProxyForHandle(0)，也就是查的是句柄值为 0 的 binder 引用，也就是 ServiceManager。如果没查到就说明可能 ServiceManager 还没来得及注册，这个时候 sleep(1) 等等就行了。然后就是调用 addService 来进行注册了。addService 就会把 name 和 binder 对象都写到 Parcel 中，然后就是调用 transact 发送一个 ADD_SERVICE_TRANSACTION 的请求。实际上是调用 IPCThreadState 的 transact 函数，第一个参数是 mHandle 值，也就是说底层在和 binder 驱动进行交互的时候是不区分 BpBinder 还是 BBinder，它只认一个 handle 值。 

Binder 驱动就会把这个请求交给 Binder 实体对象去处理，也就是是在 ServiceManager 的 onTransact 函数中处理 ADD_SERVICE_TRANSACTION 请求，也就是根据 handle 值封装一个 BinderProxy 对象，至此，Service 的注册就完成了。

至于 Client 获取服务，其实和这个差不多，也就是拿到服务的 BinderProxy 对象即可。

在回答的时候，最后可以画一下图：

![](https://i.loli.net/2020/03/27/REqCWzQSnokHKFw.png)

![](https://i.loli.net/2020/03/28/1qUCWh5B7vSVzAe.png)

上面已经说清楚了 Binder 通信模型的大致流程，下面可以再说一下一次完整的 IPC 通信流程是怎么样的。

#### 一次完整的 IPC 通信流程是怎样的？

首先是从应用层的 Proxy 的 transact 函数开始，传递到 Java 层的 BinderProxy，最后到 Native 层的 BpBinder 的 transact。在 BpBinder 的 transact 实际上是调用 IPCThreadState 的 transact 函数，在它的第一个参数是 handle 值，Binder 驱动就会根据这个 handle 找到 Binder 引用对象，继而找到 Binder 实体对象。在这个函数中，做了两件事件，一件是调用 writeTransactionData 向 Binder 驱动发出一个 BC_TRANSACTION 的命令协议，把所需参数写到 mOut 中，第二件是 waitForResponse 等待回复，在它里面才会真正的和 Binder 驱动进行交互，也就是调用 talkWithDriver，然后接收到的响应执行相应的处理。这时候 Client 接收到的是 BR_TRANSACTION_COMPLETE，表示 Binder 驱动已经接收到了 Client 的请求了。在这里面还有一个 cmd 为 BR_REPLY 的返回协议，表示 Binder 驱动已经把响应返回给 Client 端了。在 talkWithDriver 中，是通过系统调用 ioctl 来和 Binder 驱动进行交互的，传递一个 BINDER_WRITE_READ 的命令并且携带一个 binder_write_read 数据结构体。在 Binder 驱动层就会根据 write_size/read_size 处理该 BINDER_WRITE_READ 命令。

到这里，已经讲完了 Client 端如何和 Binder 驱动进行交互的了，下面就讲 Service 端是如何和 Binder 驱动进行交互的。

Service 端首先会开启一个 Binder 线程来处理进程间通信消息，也就是通过 new Thread 然后把该线程 joinThreadPool 注册到 Binder 驱动。注册呢也就是通过 BC_ECTER_LOOPER 命令协议来做的，接下来就是在 do while 死循环中调用 getAndExecuteCommand。它里面做的就是不断从驱动读取请求，也就是 talkWithDriver，然后再处理请求 executeCommand。在 executeCommand 中，就会根据 BR_TRANSACTION 来调用 BBinder Binder 实体对象的 onTransact 函数来进行处理，然后在发送一个 BC_REPLY 把响应结构返回给 Binder 驱动。Binder 驱动在接收到 BC_REPLY 之后就会向 Service 发送一个 BR_TRANSACTION_COMPLETE 协议表示 Binder 驱动已经收到了，在此同时呢，也会向 Client 端发送一个 BR_REPLY把响应回写给 Client 端。

需要注意的是，上面的 onTransact 函数就是 Service 端 AIDL 生成的 Stub 类的 onTransact 函数，这时一次完整的 IPC 通信流程就完成了。

最后可画一张图即可：

![](https://i.loli.net/2020/03/28/1ZbMj2fUiX8BGc7.png)

#### Binder 对象跨进程传递的原理是怎么样的？

有以下五点：

1. Parcel 的 writeStrongBinder 和 readStrongBinder
2. Binder 在 Parcel 中存储原理，flat_binder_object
3. 说清楚 binder_node，binder_ref
4. 目标进程根据 binder_ref 的 handle 创建 BpBinder
5. 由 BpBinder 再往上到 BinderProxy 到业务层的 Proxy

在 Native 层，Binder 对象是存在 Parcel 中的，通过 readStrongBinder/writeStrongBinder 来进行读或写，在其内部是通过一个 flat_binder_object 数据结构进行存储的，它的 type 字段是 BINDER_TYPE_BINDER，表示 Binder 实体对象，它的 cookie 指向自己。

Parcel 到了驱动层是如何处理的呢？其实就是根据 flat_binder_object 创建用于在驱动层表示的 binder_node Binder 实体对象和 binder_ref Binder 引用对象。

读 Binder 对象就是调用 unflatten_binder 把 flat_binder_object 解析出来，如果是 Binder 实体对象，返回的就是 cookie，如果是 Binder 引用对象，就是返回 getStrongProxyForHandle(handle)，其实也就是根据 handle 值 new BpBinder 出来。

#### Binder OneWay 机制

OneWay 就是异步 binder 调用，带 ONEWAY 的 waitForResponse 参数为 null，也就是不需要等待返回结果，而不带 ONEWAY 的，就是普通的 AIDL 接口，它是需要等待对方回复的。

对于系统服务来说，一般都是 oneway 的，比如在启动 Activity 时，它是异步的，不会阻塞系统服务，但是在 Service 端，它是串行化的，都是放在进程的 todo 队列里面一个一个的进行分发处理。

![](https://i.loli.net/2020/03/28/8ENCcGDdYVlUKQm.png)

#### Framework IPC 方式汇总

Android 是基于 Linux 内核构建的，Linux 已经提供了很多进程间通信机制，比如有管道、Socket、共享内存、信号等，在 Android Framework 中不仅用到了 Binder，这些 IPC 方式也都有使用到。

首先讲一下管道，管道是半双工的，管道的描述符只能读或写，想要既可以读也可以写就需要两个描述符，而且管道一般是用在父子进程之间的。Linux 提供了 pipe 函数创建一个管道，传入一个 fd[2] 数组，fd[0] 表示读端，fd[1] 表示写端。假如父进程创建了一对文件描述符，fork 出得子进程继承了这对文件描述符，这时候父进程想要往子进程写东西，就可以拿 fd[1] 写，然后子进程在 fd[0] 就可以读到了。在 Android 中，Native 层的 Looper 使用到了管道，它里面使用 epoll 监听读事件（epoll_wait），如果其他进程往里面写东西他就能收到通知。管道在哪写的呢，其实是在 wake 函数中，当别的线程向 Looper 线程发消息并且需要唤醒 Looper 线程的时候就会调用 wake 函数，wake 函数里面呢就是向管道写一个 "W" 字符。管道使用起来还是很方便的，主要是能配合 epoll 机制监听读写事件。这是 Android 19 才会使用到管道，更高版本使用的是 EventFd。

然后就是 Socket，Socket 是全双工的，也就是说既可以读也可以写，而且进程之间不需要亲缘关系，只需要公开一个地址即可。Framewok 中使用到 Socket 最经典的莫过于 Zygote 等待 AMS 请求 fork 应用程序进程了。在 Zygote 的 main 方法中 register 一个 ZygoteSocket，然后进入 runSelectLoop 循环去监听有没有新的连接，如果有的数据发过来就会去调用 runOnce 函数去根据参数 fork 出新的应用程序进程，其实就是去执行 ActivityThread 的 main 函数，然后也会通过这个 Socket 把新创建的应用进程 pid 返回给 Zygote。

接着是共享内存，共享内存它是不需要多次拷贝的，而且特别快。拿到文件描述符分别映射到进程的地址空间即可。在 Android 中提供了 MemoryFile 类，里面封装了 ashmem 机制，也就是 Android 的匿名共享内存。首先通过 ashmem_create_region 创建一块匿名共享内存，返回一个 fd，然后调用 mmap 函数给这个 fd 映射到当前进程地址空间中。

最后就是信号，信号是单向的，而且发出去之后不关心处理结果，知道进程的 pid 就能发信号了。在杀应用进程的时候会调用 Process 的 killProcess 函数发送一个 SIGNAL_KILL 信号。还有 Zygote 在 fork 完成一个新的子进程之后还会监听 SIGCHLD 信号，如果子进程退出之后就会回收相应的资源，避免子进程成为一个僵尸进程。