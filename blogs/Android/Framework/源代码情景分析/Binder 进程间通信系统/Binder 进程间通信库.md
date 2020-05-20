---
Binder 进程间通信库
---

Android 系统在应用程序框架层中将各种 Binder 驱动程序操作封装成了一个 Binder 库，这样进程就可以以方便的调用 Binder 库提供的接口来实现进程间通信。

在 Binder 库中，Service 组件和 Client 组件分别使用模版类 BnInterface 和 BpInterface 来描述，其中，前者称为 Binder 本地对象，后者称为 Binder 代理对象。Binder 库中的 Binder 本地对象和 Binder 代理对象分别对应于 Binder 驱动程序中的 Binder 实体对象和 Binder 引用对象。

模板类 BnInterface 继承了 BBinder 类，后者为 Binder 本地对象提供了抽象的进程间通信接口，它的定义如下：

```c++
class BBinder : public IBinder
{
	public:
		virtual status_t transact(...);
	protected:
		virtual status_t onTransact(...);
}
```

BBinder 类有两个重要的成员函数 transact 和 onTransact。当一个 Binder 代理对象通过 Binder 驱动程序向一个 Binder 本地对象发出一个进程间通信请求时，Binder 驱动程序就会调用该 Binder 本地对象的成员函数 transact 来处理该请求。成员函数 onTransact 是由 BBinder 的子类，即 Binder 本地对象类来实现的，它负责分发与业务相关的进程间通信请求。事实上，与业务相关的进程间通信请求是由 Binder 本地对象类的子类，即 Service 组件类来负责处理的。

模版类 BpInterface 继承了 BpRefBase 类，后者为 Binder 代理对象提供了抽象的进程间通信接口，它的定义如下所示：

```c++
class BpRefBase: public virtual RefBase
{
	inline IBinder* remote() { return mRemote; }
	
	private:
		IBinder* const mRemote;
}
```

BpRefBase 类有一个重要的成员变量 mRemote，它指向一个 BpBinder 对象，可以通过成员函数 remote 来获取。BpBinder 类实现了 BpRefBase 类的进程间通信接口，它的定义如下所示：

```c++
class BpBinder: public IBinder
{
	public:
		BpBinder(int32_t handle);
		transact(...);
	private:
		const int32_t mHandle;
}
```

BpBinder 类的成员变量 mHandle 是一个整数，它表示一个 Client 组件的句柄值，在前面介绍结构体 binder_ref 时提到，每一个 Client 组件在 Binder 驱动程序中都对应有一个 Binder 引用对象，而每一个 Binder 引用对象都有一个句柄值。其中，Client 组件就是通过这个句柄值来和 Binder 驱动程序中的 Binder 引用对象建立对应关系的。

BpBinder 类的成员函数 transact 用来向运行在 Server 进程中的 Service 组件发送进程间通信请求，这是通过 Binder 驱动程序间接实现的。BpBinder 类的成员函数 transact 会把 BpBinder 类的成员变量 mHandle，以及进程间通信数据发送给 Binder 驱动程序，这样 Binder 驱动程序就能够根据这个句柄值来找到对应的 Binder 引用对象，继而找到对应的 Binder 实体对象，最后就可以将进程间通信数据发送给对应的 Service 组件了。

无论是 BBinder 类，还是 BpBinder 类，它们都是通过 IPCThreadState 类来和 Binder 驱动程序交互的。IPCThreadState 类的定义如下所示：

```c++
class IPCThreadState
{
	public:
    	static	IPCThreadState* self();
    			status_t	transact(...);
    private:
    	status_t	talkWithDriver();
    	const	sp<ProcessState> mProcess;
}
```

在前面介绍结构体 binder_proc 时提到，每一个使用了 Binder 进程间通信机制的进程都有一个 Binder 线程池，用来处理进程间通信请求。对于每一个 Binder 线程来说，它的内部都有一个 IPCThreadState 对象，我们可以通过 IPCThreadState 类的静态成员函数 self 来获取，并且调用它的成员函数 transact 来和 Binder 驱动程序交互。在 transact 函数内部，与 Binder 驱动程序的交互操作又是通过调用成员函数 talkWithDriver 来实现的，它一方面负责向 Binder 驱动程序发送进程间通信请求，另一方面又负责接收来自 Binder 驱动程序的进程间通信请求。

IPCThreadState 类有一个成员变量 mProcess，它指向一个 ProcessState 对象。对于每一个使用了 Binder 进程间通信机制的进程来说，它的内部都有一个 ProcessState 对象，它负责初始化 Binder 设备，即打开设备文件 /dev/binder，以及将设备文件 /dev/binder 映射到进程的地址空间。由于这个 ProcessState 对象在进程范围内是唯一的，因此，Binder 线程池中的每一个线程都可以通过它来和 Binder 驱动程序建立连接。

ProcessState 类的定义如下所示：

```c++
class ProcessState: public virtual RefBase
{
	public:
		static sp<ProcessState> self();
    private:
    	int mDriverFD;
    	void* mVMStart;
}
```

进程中的 ProcessState 对象可以通过 ProcessState 类的静态成员函数 self 来获取。第一次调用 self 时，Binder 库就会为进程创建一个 ProcessState 对象，并且调用函数 open 来打开设备文件 /dev/binder，接着又调用函数 mmap 将它映射到进程的地址空间，即请求 Binder 驱动程序为进程分配内核缓冲区。设备文件 /dev/binder 映射到进程的地址空间后，得到的内核缓冲区的用户地址就保在其成员变量 mVMStart 中。

至此，Binder 库的基础知识就介绍完了。