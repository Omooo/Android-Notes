---
ServiceManager 的启动过程
---

ServiceManager 是 Binder 进程间通信机制的核心组件之一，它扮演着 Binder 进程间通信机制上下文管理者的角色，同时负责管理系统中的 Service 组件，并且向 Client 组件提供获取 Service 代理对象的服务。

ServiceManager 运行在一个独立的进程中，因此，Service 组件和 Client 组件也需要通过进程间通信机制来和它交互，而采用的进程间通信机制正好也是 Binder 进程间通信机制。

ServiceManager 是由 init 进程负责启动的，而 init 进程是在系统启动时启动的，因此，ServiceManager 也是在系统启动时启动。ServiceManager 的入口函数 main 实现如下：

```c++
int main() {
    struct binder_state *bs;
    void *svcmgr = BINDER_SERVICE_MANAGER;
	bs = binder_open(128*1024);
	if(binder_become_context_manager(bs)) {
		return -1;
	}
    svcmgr_handle = svcmgr;
	binder_loop(bs, svcmgr_handler);
}
```

ServiceManager 的启动过程由三个步骤组成：第一步是调用函数 binder_open 打开设备文件 /dev/binder，以及将它映射到本进程的地址空间；第二步是调用函数 binder_become_context_manager 将自己注册为 Binder 进程间通信机制的上下文管理者；第三步是调用函数 binder_loop 来循环等待和处理 Client 进程的通信请求。

ServiceManager 是一个特殊的 Service 组件，它的特殊之处就在于与它对应的 Binder 本地对象是一个虚拟的对象。这个虚拟的 Binder 本地对象的地址值等于 0，并且在 Binder 驱动程序中引用了它的 Binder 引用对象的句柄值也等于 0。

接下来，我们就分别分析函数 binder_open、binder_become_context_manager 和 binder_loop 的实现。

#### 打开和映射 Binder 设备文件

函数 binder_open 用来打开设备文件 /dev/binder，并且将它映射到进程的地址空间，它的实现如下所示：

```c++
struct binder_state *binder_open(unsigned mapsize)
{
	struct binder_state *bs;
	bs->fd = open("/dev/binder", O_RDWR);
	bs->mapped = mmap(...);
	return bs;
}
```

当进程调用 open 函数打开设备文件时，Binder 驱动程序中的函数 binder_open 就会被调用，它会为当前进程创建一个 binder_proc 结构体，用来描述当前进程的 Binder 进程间通信状态。然后调用函数 mmap 将设备文件 /dev/binder 映射到进程的地址空间，请求 Binder 驱动程序为进程分配 128K 大小的内核缓冲区。

打开了设备文件 /dev/binder，以及将它映射到进程的地址空间之后，ServiceManager 接下来就会将自己注册为 Binder 进程间通信机制的上下文管理者。

#### 注册为 Binder 上下文管理者

ServiceManager 要成为 Binder 进程间通信机制的上下文管理者，就必须要通过 IO 控制命令 BINDER_SET_CONTEXT_MGR 将自己注册到 Binder 驱动程序中，这是通过调用函数 binder_become_context_manager 来实现的，它的定义如下：

```
int binder_become_context_manager(struct binder_state *bs)
{
	return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```

一个进程的所有 Binder 线程都保存在一个 binder_proc 结构体的成员变量 threads 所描述的一个红黑树中。

一个进程的所有 Binder 实体对象都保存在它的成员变量 nodes 所描述的一个红黑树中。

ServiceManager 返回到进程的用户空间之后，接着继续调用函数 binder_loop 来循环等待和处理 Client 进程的通信请求，即等待和处理 Service 组件的注册请求以及其代理对象的获取请求。

#### 循环等待 Client 进程请求

由于 ServiceManager 需要在系统运行期间为 Service 组件和 Client 组件提供服务，因此，它就需要通过一个无限循环来等待和处理 Service 组件和 Client 组件的进程间通信请求，这是通过调用函数 binder_loop 来实现的，如下所示：

```c++
void binder_loop(...)
{
	struct binder_write_read bwr;
	for(;;) {
		res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
		res = binder_parse(bs, ...);
	}
}
```

在函数 binder_loop 中，循环不断使用 IO 控制命令 BINDER_WRITE_READ 来检查 Binder 驱动程序是否有新的进程间通信请求需要它来处理。如果有，就将它们交给函数 binder_parse 来处理，否则，当前线程就会在 Binder 驱动程序中睡眠等待，直到有新的进程间通信请求到来为止。由于该 binder_write_read 结构体中的输入缓冲区的长度为 0，因此，Binder 驱动程序的函数 binder_ioctl 在处理这些 IO 控制命令 BINDER_WRITE_READ 时，只会调用函数 binder_thread_read 来检查 ServiceManager 进程是否有新的进程间通信请求需要处理。

至此，ServiceManager 的启动过程就分析完了。为了方便接下来内容的描述，我们假设 ServiceManager 进程的主线程没有待处理的工作项，因此，它就睡眠在 Binder 驱动程序的函数 binder_thread_read 中，等待其他进程的 Service 组件或者 Client 组件向它发送进程间通信请求。

此外，从 ServiceManager 的启动过程可以知道，当一个进程使用 IO 控制命令 BINDER_WRITE_READ 与 Binder 驱动程序交互时：

1. 如果传递给 Binder 驱动程序的 binder_write_read 结构体的输入缓冲区的长度大于 0，那么 Binder 驱动程序就会调用函数 binder_thread_write 来处理输出缓冲区中的命令协议；
2. 如果传递给 Binder 驱动程序的 binder_write_read 结构体的输出缓冲区长度大于 0，那么 Binder 驱动程序就会调用函数 binder_write_read 将进程需要处理的工作项写入到该输出缓冲区中，即将相应的返回协议写入到该缓冲区中，以便进程可以在用户空间处理。