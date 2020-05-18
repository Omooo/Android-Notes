---
Binder 驱动程序之基础数据结构
---

#### 前言

Binder 驱动程序实现在内核空间中，它主要有 binder.h 和 binder.c 两个源文件组成。下面我们就开始介绍 Binder 驱动程序的基础知识，包括基础数据结构、初始化过程，以及设备文件 /dev/binder 的打开（open）、内存映射（mmap）和内核缓冲区管理等操作。

#### 基础数据结构

**struct binder_work**

结构体 binder_work 用来描述待处理的工作项。

**struct binder_node**

```c
struct binder_node {
    struct binder_proc *proc;
}
```

结构体 binder_node 用来描述一个 Binder 实体对象。每一个 Service 组件在 Binder 驱动程序中都对应有一个 Binder 实体对象，用来描述它在内核中的状态。

成员变量 proc 指向一个 Binder 实体对象的宿主进程。在 Binder 驱动程序中，这些宿主进程通过一个 binder_proc 结构体来描述。宿主进程使用一个红黑树来维护它内部所有的 Binder 实体对象。

由于一个 Binder 实体对象可能会同时被多个 Client 组件引用，因此，Binder 驱动程序就使用结构体 binder_ref 来描述这些引用关系，并且将引用了同一个 Binder 实体对象的所有引用都保存在一个hash 列表中。这个 hash 列表通过 Binder 实体对象的成员变量 refs 来描述。

**struct binder_ref_death**

结构体 binder_ref_death 用来描述一个 Service 组件的死亡接收通知。

**struct binder_ref**

```c
struct binder_ref {
    struct hlist_node node_entry;
	struct binder_proc *proc;
	struct binder_node *node;
    uint32_t desc;
}
```

结构体 binder_ref 用来描述一个 Binder 引用对象，每一个 Client 组件在 Binder 驱动程序中都对应有一个 Binder 引用对象，用来描述它在内核中的状态。

成员变量 node 用来描述一个 Binder 引用对象所引用的 Binder 实体对象。前面在介绍结构体 binder_node 时提到，每一个 Binder 实体对象都有一个 hash 列表，用来保存那些引用了它的 Binder 引用对象，而这些 Binder 引用对象的成员变量 node_entry 正好是这个 hash 列表的节点。

成员变量 desc 是一个句柄值，或者称为描述符，它是用来描述一个 Binder 引用对象的。在 Client 进程的用户空间中，一个 Binder 引用对象是使用一个句柄值来描述的。因此，当 Client 进程的用户空间通过 Binder 驱动程序来访问一个 Service 组件时，它只需要指定一个句柄值，Binder 驱动程序就可以通过该句柄值找到对应的 Binder 引用对象，然后再根据该 Binder 引用对象的成员变量 node 找到对应的 Binder 实体对象，最后就可以通过该 Binder 实体对象找到要访问的 Service 组件。

**struct binder_buffer**

```c
struct binder_buffer {
    unsigned free :1;
	unsigned allow_user_free : 1;
    unsigned async_transaction : 1;
    
    struct binder_transaction *transaction;
    struct binder_node *target_node;
}
```

结构体 binder_buffer 用来描述一个内核缓冲区，它是用来在进程间传输数据的。每一个使用 Binder 进程间通信机制的进程在 Binder 驱动程序中都有一个内核缓冲区列表，用来保存 Binder 驱动程序为它所分配的内核缓冲区。

成员变量 transaction 和 target_node 用来描述一个内核缓冲区正在交给哪一个事务以及哪一个 Binder 实体对象使用。Binder 驱动程序使用一个 binder_transaction 结构体来描述一个事务，每一个事务都关联有一个目标 Binder 实体对象。Binder 驱动程序将事务数据保存在一个内核缓冲区中，然后将它交给目标 Binder 实体对象处理；而目标 Binder 实体对象再将该内核缓冲区的内容交给相应的 Service 组件处理。Service 组件处理完成该事务之后，如果发现传递给它的内核缓冲区的成员变量 allow_user_free 的值为 1，那么该 Service 组件就会请求 Binder 驱动程序释放该内核缓冲区。

**struct binder_proc**

```c
struct binder_proc {
    struct rb_root threads;
    int pid;
    size_t buffer_size;
    struct list_head todo;
    wait_queue_head_t wait;
}
```

结构体 binder_proc 用来描述一个正在使用 Binder 进程间通信机制的进程。当一个进程调用函数 open 来打开设备文件 /dev/binder 时，Binder 驱动程序就会为它创建一个 binder_proc 结构体，并且将它保存在一个全局的 hash 列表中。

进程打开了设备文件 /dev/binder 之后，还必须调用函数 mmap 将它映射到进程的地址空间来，实际上是请求 Binder 驱动程序为它分配一块内核缓冲区，以便可以用来在进程间传输数据。Binder 驱动程序为进程分配的内核缓冲区的大小保存在成员变量 buffer_size 中。这些内核缓冲区有两个地址，其中一个是内核空间地址，另一个是用户空间地址。

前面提到，每一个使用了 Binder 进程间通信机制的进程都有一个 Binder 线程池，用来处理进程间通信请求，这个 Binder 线程池是由 Binder 驱动程序来维护的。结构体 binder_proc 的成员变量 threads 是一个红黑树的根节点，它以线程 ID 作为关键字来组织一个进程的 Binder 线程池。进程可以调用 ioctl 将一个线程注册到 Binder 驱动程序中，同时当进程没有足够的空闲线程来处理进程间通信请求时，Binder 驱动程序也可以主动要求进程注册更多的线程到 Binder 线程池中。

当进程接收到一个进程间通信请求时，Binder 驱动程序就将该请求封装成一个工具项，并且加入到进程的待处理工作项队列中，这个队列使用成员变量 todo 来描述。Binder 线程池中的空闲 Binder 线程会睡眠在由成员变量 wait 所描述的一个等待队列中，当它们的宿主进程的待处理工作项队列增加了新的工作项之后，Binder 驱动程序就会唤醒这些线程，以便它们可以去处理新的工作项。

**struct binder_thread**

```c
struct binder_thread {
	struct binder_proc *proc;
    int pid;
    int looper;
}
```

结构体 binder_thread 用来描述 Binder 线程池中的一个线程，其中，成员变量 proc 指向其宿主进程。

一个 Binder 线程的 ID 和状态是通过成员变量 pid 和 looper 来描述的。

一个线程注册到 Binder 驱动程序时，Binder 驱动程序就会为它创建一个 binder_thread 结构体，并且将它的状态初始化为 BINDER_LOOPER_STATE_NEED_RETURN，表示该线程需要马上返回到用户空间。由于一个线程在注册为 Binder 线程时可能还没有准备好去处理进程间通信请求，因此，最好返回到用户空间去做准备工作。

一个线程注册到 Binder 驱动程序之后，它接着就会通过 BC_REGISTER_LOOPER（Binder 驱动程序请求创建） 或者 BC_ENTER_LOOPER （应用程序主动注册）协议来通知 Binder 驱动程序，它可以处理进程间通信请求了。

**struct binder_transaction**

```c
struct binder_transaction {
	unsigned need_reply : 1;
}
```

结构体 binder_transaction 用来描述进程间通信过程，这个过程又称为一个事务。成员变量 need_reply 用来区分一个事务是同步的还是异步的。同步事务需要等待对方回复，这时候它的成员变量 need_reply 的值就会设置为 1；否则就设置为 0，表示这是一个异步事务，不需要等待回复。



以上介绍的结构体都是在 Binder 驱动程序内部使用的。前面提到，应用程序进程在打开了设备文件 /dev/binder 之后，需要通过 IO 控制函数 ioctl 来进一步与 Binder 驱动程序进行交互，因此，Binder 驱动程序就提供了一系列的 IO 控制命令来和应用程序进程通信。在这些 IO 控制命令中，最重要的便是 BINDER_WRITE_READ 命令了。

IO 控制命令 BINDER_WRITE_READ 后面所跟的参数是一个 binder_write_read 结构体，它的定义如下：

```c
struct binder_write_read {
	signed long write_size;
    signed long write_consumed;
    unsigned long write_buffer;
    signed long read_size;
    signed long read_consumed;
    unsigned long read_buffer;
}
```

结构体 binder_write_read 用来描述进程间通信过程中所传输的数据。这些数据包括输入数据和输出数据，其中，成员变量 write_size、write_consumed 和 write_buffer 用来描述输入数据，即从用户空间传输到 Binder 驱动程序的数据；而成员变量 read_size、read_consumend 和 read_buffer 用来描述输出数据，即从 Binder 驱动程序返回给用户空间的数据，它也是进程间通信结果数据。

成员变量 write_buffer 指向一个用户空间缓冲区的地址，里面保存的内容即为要传输到 Binder 驱动程序的数据。缓冲区 write_buffer 的大小由成员变量 write_size 来制定，单位是字节。成员变量 write_consumed 用来描述 Binder 驱动程序从缓冲区 write_buffer 中处理了多少个字节的数据。

成员变量 read_buffer 也是指向一个用户空间缓冲区的地址，里面保存的内容即为 Binder 驱动程序返回给用户空间的进程间通信结果数据。缓冲区 read_buffer 的大小由成员变量 read_size 来指定，单位是字节。成员变量 read_consumed 用来描述用户空间应用程序从缓冲区 read_buffer 中处理了多少个字节的数据。

缓冲区 write_buffer 和 read_buffer 都是一个数组，数组的每一个元素都由一个通信协议代码及其通信数据组成。协议代码又分为两种类型，其中一种是在输入缓冲区 write_buffer 中使用的，称为命令协议代码；另一种是在输出缓冲区 read_buffer 中使用的，称为返回协议代码。命令协议代码通过 BinderDriverCommandProtocol 枚举值来定义，而返回协议代码通过 BinderDriverReturnProtocol 枚举值来定义。

命令协议码：

命令协议代码 BC_TRANSACTION 和 BC_REPLY 后面跟的通信数据使用一个结构体 binder_transaction_data 来描述。当一个进程请求另外一个进程执行某一个操作时，源进程就使用命令协议代码 BC_TRANSACTION 来请求 Binder 驱动程序将通信数据传递到目标进程；当目标进程处理完成源进程所请求的操作之后，它就使用命令协议代码 BC_REPLY 来请求 Binder 驱动程序将结果数据传递给源进程。

返回协议码：

返回协议代码 BR_TRANSACTION 和 BR_REPLY 后面跟的通信数据使用一个结构体 binder_transaction_data 来描述。当一个 Client 进程向一个 Servce 进程发出进程间通信请求时，Binder 驱动程序就会使用返回协议码 BR_TRANSACTION 通知该 Servcer 进程来处理该进程间通信请求；当 Server 进程处理完成该进程间通信请求之后，Binder 驱动程序就会使用返回协议码 BR_REPLY 将进程间通信请求结果数据返回给 Client。

返回协议代码 BR_TRANSACTION_COMPLETE 后面不需要指定通信数据。当 Binder 驱动程序接收到应用程序进程给它发送的一个命令协议代码 BC_TRANSACTION 或者 BC_REPLY 时，它就会使用返回协议码 BR_TRANSACTION_COMPLET来通知应用程序进程，该命令协议码已经被接收，正在分发给目标进程或者目标线程处理。

介绍完 Binder 驱动程序提供的命令协议码和返回协议码之后，接下来我们继续分析这些协议所使用的一个结构体 binder_transaction_data 的定义。

**struct binder_transaction_data**

```c
struct binder_transaction_data {
	union {
        size_t handle;
        void *ptr;
    } target;
    unsigned int code;
    pit_t sender_pid;
    uid_t sender_euid;
}
```

结构体 binder_transaction_data 用来描述进程间通信过程中所传输的数据。

成员变量 target 是一个联合体，用来描述一个目标 Binder 实体对象或者目标 Binder 引用对象。如果它描述的是一个目标 Binder 实体对象，那么它的成员变量 ptr 就指向与该 Binder 实体对象对应的一个 Service 组件内部的一个弱引用计数对象（weakref_impl）的地址；如果它描述的是一个目标 Binder 引用对象，那么它的成员变量 handle 就指向该 Binder 引用对象的句柄值。

成员变量 flags 是一个标志值，用来描述进程间通信行为特征，它的取值有 TF_ONE_WAY 和 TF_ACCEPT_FDS。如果成员变量 flags 的 TF_ONE_WAY 位被设置为 1，就表示这是一个异步的进程间通信过程；如果成员变量 flags 的 TF_ACCEPT_FDS 位被设置为 0，就表示源进程不允许目标进程返回的结果数据中包含有文件描述符；

成员变量 sender_pid 和 sender_euid 表示发起进程间通信请求的进程的 PID 和 UID。这两个成员变量的值是由 Binder 驱动程序来填写的，因此，目标进程通过这两个成员变量就可以识别出源进程的身份，以便进行安全检查。