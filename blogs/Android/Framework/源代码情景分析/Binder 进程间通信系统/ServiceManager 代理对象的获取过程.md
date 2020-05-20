---
ServiceManager 代理对象的获取过程
---

Service 组件在启动时，需要将自己注册到 ServiceManager 中；而 Client 组件在使用 Service 组件提供的服务之前，也需要通过 ServiceManager 来获得 Servcie 组件的代理对象。由于 ServiceManager 本身也是一个 Service 组件，因此，其他的 Service 组件和 Client 组件在使用它提供的服务之前，也需要先获得它的代理对象。

ServiceManager 代理对象的类型为 BpServiceManager，它用来描述一个实现了 IServiceManager 接口的 Client 组件。IServiceManager 接口定义了四个成员函数 getService、checkService、addService 和 listService，其中，getService 和 checkService 用来获取 Service 组件的代理对象，addService 用来注册 Service 组件，listService 用来获取注册在 ServiceManager 中的 Service 组件名称列表。

对于一般的 Service 组件来说，Client 进程首先要通过 Binder 驱动程序来获得它的一个句柄值，然后才可以根据这个句柄值创建一个 Binder 代理对象，最后将这个 Binder 代理对象封装成一个实现了特定接口的代理对象。由于 ServiceManager 的句柄值恒为 0，因此，获取它的一个代理对象的过程就省去了与 Binder 驱动程序交互的过程。

Android 系统在应用程序框架层的 Binder 库中提供了一个函数 defaultServiceManager 来获得一个 ServiceManager代理对象。

```c++
sp<IServiceManager> defaultServiceManager()
{
	ProcessState::self()->getContextObject(null);
}
sp<IBinder> ProcessState::getContextObject()
{
    return getStrongProxyForHandle(0);
}
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    return new BpBinder(handle);
}
```

可以看到，获取 ServiceManager 的代理对象，其实就是 new BpBinder(0)，然后再转化为 BpServiceManager。

至此，一个 ServiceManager 代理对象的获取就分析完了。有了这个 ServiceManager 代理对象之后，Service 组件就可以在启动的过程中使用它的成员函数 addService 将自己注册到 ServiceManager 中，而 Client 组件就可以使用它的成员函数 getService 来获得一个指定名称的 Service 组件的代理对象了。