---
Binder 对象引用计数技术
---

#### 前言

在 Client 进程和 Server 进程的一次通信过程中，涉及了四种类型的对象，它们分别是位于 Binder 驱动程序中的 Binder 实体对象（binder_node）和 Binder 引用对象（binder_ref），以及位于 Binder 库中的 Binder 本地对象（BBinder）和 Binder 代理对象（BpBinder），它们的交互过程如下图所示：

![](https://i.loli.net/2020/05/18/53KXVjQmGT2EuCg.png)

它们的交互过程可以划分为五个步骤：

1. 运行在 Client 进程中的 Binder 代理对象通过 Binder 驱动程序向运行在 Server 进程中的 Binder 本地对象发出一个进程间通信请求，Binder 驱动程序接着就根据 Client 进程传递过来的 Binder 代理对象的句柄值来找到对应的 Binder 引用对象。
2. Binder 驱动程序根据前面找到的 Binder 引用对象找到对应的 Binder 实体对象，并且创建一个事务（binder_transaction）来描述该次进程间通信过程。
3. Binder 驱动程序根据前面找到的 Binder 实体对象来找到运行在 Server 进程中的 Binder 本地对象，并且将 Client 进程传递过来的通信数据发送给它处理。
4. Binder 本地对象处理完成 Client 进程的通信请求之后，就将通信结果返回给 Binder 驱动程序，Binder 驱动程序接着就找到前面所创建的一个事务。
5. Binder 驱动程序根据前面找到的事务的相关属性来找到发出通信请求的 Client 进程，并且通知 Client 进程将通信结果返回给对应的 Binder 代理对象处理。

从这个过程就可以看出，Binder 代理对象依赖于 Binder 引用对象，而 Binder 引用对象又依赖于 Binder 实体对象，最后，Binder 实体对象又依赖于 Binder 本地对象。这样，Binder 进程间通信机制就必须采用一种技术措施来保证，不能销毁一个还被其他对象依赖着的对象。为了维护这些 Binder 对象的依赖关系，Binder 进程间通信机制采用了引用计数计数来维护每一个 Binder 对象的生命周期。

接下来，我们就分析 Binder 驱动程序和 Binder 库是如何维护 Binder 本地对象、Binder 实体对象、Binder 引用对象和 Binder 代理对象的生命周期。

#### Binder 本地对象的生命周期

Binder 本地对象是一个类型为 BBinder 的对象，它是在用户空间中创建的，并且运行在 Server 进程中。Binder 本地对象一方面会被运行在 Server 进程中的其他对象引用，另一方面也会被 Binder 驱动程序中的 Binder 实体对象引用。由于 BBinder 类继承了 RefBase 类，因此，Server 进程中的其他对象可以简单的通过智能指针来引用这些 Binder 本地对象，以便可以控制它们的生命周期。由于 Binder 驱动程序中的 Binder 实体对象是运行在内核空间的，它不能够通过智能指针来引用运行在用户空间的 Binder 本地对象，因此，Binder 驱动程序就需要和 Server 进程约定一套规则来维护它们的引用计数，避免它们在还被 Binder 实体对象引用的情况下销毁。

Server 进程将一个 Binder 本地对象注册到 ServerManager 时，Binder 驱动程序就会为它创建一个 Binder 实体对象。接下来，当 Client 进程通过 ServerManager 来查询一个 Binder 本地对象的代理对象接口时，Binder 驱动程序就会为它所对应的 Binder 实体对象创建一个 Binder 引用对象，接着在使用 BR_INCREFS 和 BR_ACQUIRE 协议来通知对应的 Server 进程增加对应的 Binder 本地对象的弱引用技术和强引用技术。这样就能保证 Client 进程中的 Binder 代理对象在引用一个 Binder 本地对象期间，该 Binder 本地对象不会被销毁。当没有任何 Binder 代理对象引用一个 Binder 本地对象时，Binder 驱动程序就会使用 BR_DECREFS 和 BR_RELEASE 协议来通知对应的 Server 进程减少对应的 Binder 本地对象的弱引用技术和强引用技术。

总结来说，Binder 驱动程序就是通过 BR_INCREFS、BR_ACQUIRE、BR_DECREFS 和 BR_RELEASE 协议来引用运行在 Server 进程中的 Binder 本地对象的，相关的代码实现在函数 binder_thread_read 中，这里就不在贴代码了。

至此，我们就分析完成 Binder 本地对象的生命周期了。

#### Binder 实体对象的生命周期

Binder 实体对象是一个类型为 binder_node 的对象，它是在 Binder 驱动程序中创建的，并且被 Binder 驱动程序中的 Binder 引用对象所引用。

当 Client 进程第一次引用一个 Binder 实体对象时，Binder 驱动程序就会在内部为它创建一个 Binder 引用对象。例如，当 Client 进程通过 ServerManager 来获得一个 Service 组件的代理对象接口时，Binder 驱动程序就会找到与该 Service 组件对应的 Binder 实体对象，接着再创建一个 Binder 引用对象来引用它。这时候就需要增加被引用的 Binder 实体对象的引用计数。相应地，当 Client 进程不再引用一个 Service 组件时，它也会请求 Binder 驱动程序释放之前为它所创建的一个 Binder 引用对象。这时候就需要减少该 Binder 引用对象所引用的 Binder 实体对象的引用计数。

#### Binder 引用对象的生命周期

Binder 引用对象是一个类型为 binder_ref 的对象，它是在 Binder 驱动程序中创建的，并且被用户空间中的 Binder 代理对象所引用。

当 Client 进程引用了 Server 进程中的一个 Binder 本地对象时，Binder 驱动程序就会在内部为它创建一个 Binder 引用对象。由于 Binder 引用对象是运行在内核空间的，而引用了它的 Binder 代理对象是运行在用户空间的，因此，Client 进程和 Binder 驱动程序就需要约定一套规则来维护 Binder 引用对象的引用计数，避免它们在还被 Binder 代理对象引用的情况下被销毁。

这套规则可以划分为 BC_ACQUIRE、BC_INCREFS、BC_RELEASE 和 BC_DECREFS 四个协议，分别用来增加和减少一个 Binder 引用对象的强引用技术和弱引用技术。相关的代码实现在 Binder 驱动程序的函数 binder_thread_write 中。

#### Binder 代理对象的生命周期

Binder 代理对象是一个类型为 BpBinder 的对象，它是在用户空间中创建的，并且运行在 Client 进程中。与 Binder 本地对象类似，Binder 代理对象一方面会被运行在 Client 进程中的其他对象引用，另一方面它也会引用 Binder 驱动程序中的 Binder 引用对象。由于 BpBinder 类继承了 RefBase 类，因此，Client 进程中的其他对象可以简单地通过智能指针来引用这些 Binder 代理对象，以便可以控制它们的生命周期。由于 Binder 驱动程序中的 Binder 引用对象是运行在内核空间的，Binder 代理对象就不能通过智能指针来引用它们，因此，Client 进程就需要通过 BC_ACQUIRE、BC_INCREFS、BC_RELEASE 和 BC_DECREFS 四个协议来引用 Binder 驱动程序中的 Binder 引用对象。

前面提到，每一个 Binder 代理对象都是通过一个句柄值来和一个 Binder 引用对象关联的，而 Client 进程就是通过这个句柄值来维护运行在它里面的 Binder 代理对象的。具体来说，就是 Client 进程会在内部创建一个 handle_entry 类型的 Binder 代理对象列表，它以句柄值作为关键字来维护它内部所有的 Binder 代理对象。

