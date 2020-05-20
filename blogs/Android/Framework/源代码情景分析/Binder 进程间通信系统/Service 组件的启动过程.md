---
Service 组件的启动过程
---

Service 组件是在 Service 进程中运行。Service 进程在启动时，会首先将它里面的 Service 组件注册到 ServiceManager 中，接着在启动一个 Binder 线程池来等待和处理 Client 进程的通信请求。

#### 注册 Service 组件

在将 Service 组件注册到 ServiceManager 之前，我们首先要获得一个 ServiceManager 代理对象，然后才可以通过它将该 Service 组件注册到 ServiceManager 中。

函数 defaultServiceManager 返回的 ServiceManager 代理对象的类型为 BpServiceManager，然后调用它的 addService 函数来注册自己。

Client 进程和 Service 进程的一次进程间通信过程可以划分为如下五个步骤：

1. Client 进程将进程间通信数据封装成一个 Parcel 对象，以便可以将进程间通信数据传递给 Binder 驱动程序。
2. Client 进程向 Binder 驱动程序发送一个 BC_TRANSACTION 命令协议。Binder 驱动程序根据协议内容找到目标 Service 进程之后，就会向 Client 进程发送一个 BR_TRANSACTION_COMPLETE 返回协议，表示它的进程间通信请求已经被接受。Client 进程接收到 Binder 驱动程序发送给它的 BR_TRANSACTION_COMPLETE 返回协议，并且对它进程处理之后，就会再次进入 Binder 驱动程序中去等待目标 Service 进程返回进程间通信结果。
3. Binder 驱动程序再向 Client 进程发送 BR_TRANSACTION_COMPLETE 返回协议的同时，也会向目标 Server 进程发送一个 BR_TRANSACTION 返回协议，请求目标 Service 进程处理该进程间通信请求。
4. Service 进程接收到 Binder 驱动程序发来的 BR_TRANSACTION 返回协议，并且对它进行处理之后，就会向 Binder 驱动程序发送一个 BC_REPLY 命令协议。Binder 驱动程序根据协议内容找到目标 Client 进程之后，就会向 Service 进程发送一个 BC_REPLY 命令协议。Binder 驱动程序根据协议内容找到目标 Client 进程之后，就会向 Service 进程发送一个 BR_TRANSACTION_COMPLETE 返回协议，表示它返回的进程间通信结果已经收到了。Service 进程接收到 Binder 驱动程序发送给它的 BR_TRANSACTION_COMPLETE 返回协议，并且对它进行处理之后，一次进程间通信过程就结束了。接着它会再次进入到 Binder 驱动程序中去等待下一次进程间通信请求。
5. Binder 驱动程序向 Service 进程发送 BR_TRANSACTION_COMPLETE 返回协议的同时，也会向目标 Client 进程发送一个 BR_REPLY 返回协议，表示 Service 进程已经处理完它的进程间通信请求了，并且将进程间通信结果返回给它。

为什么 Binder 驱动程序要发送一个 BR_TRANSACTION_COMPLETE 返回协议来响应 Client 进程和 Service 进程的 BC_TRANSACTION 与 BC_REPLY 命令协议呢？Client 进程和 Service 进程其实并不需要对 BR_TRANSACTION_COMPLETE 返回协议做特殊处理。这样做其实是为了让 Client 进程和 Service 进程在执行进程间通信的过程中，有机会返回到用户空间去做其他事情，从而增加它们的并发处理能力。

在接下来的内容中，我们将根据上述五个步骤来分析 BpServiceManager 类的成员函数 addService 的实现。

**封装进程间通信数据**

